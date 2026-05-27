# Phase 3b / 3c Implementation Plan — Service 層與認證 Blueprint

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 實作 RPi server 的 service 層（user / device / channel）與認證 blueprint（register / login / logout），銜接既有 DB repository（3a）與 BLE 層（2）。

**Architecture:** device/channel service 為 class，建構注入長生命週期協作者（`BLEManager`、`RateLimiter`、`on_reading` callback），每個方法第一參數收該執行緒專屬的 `sqlite3.Connection`；user service 為純函式。web 用 application factory，per-request conn 以 `flask.g` + `teardown_appcontext` 管理。所有 BLE 操作同步呼叫 `BLEManager` 介面（`BluepyManager` 內部已序列化到 worker thread）。SocketIO、notify subscribe wiring、重連迴圈留待 3e。

**Tech Stack:** Python 3.11+、Flask、Flask-Login、Flask-WTF、bcrypt、SQLite、pytest。

> **Working directory:** 所有路徑相對於 `Intelligent-home-RPi-server/`（git submodule）。所有指令在該目錄下執行。
>
> **測試/lint/type 指令：** `uv run pytest`、`uv run ruff check`、`uv run mypy src`。

---

## Task 0: 建立工作分支

- [ ] **Step 1: 在 submodule 內開分支**

```bash
cd Intelligent-home-RPi-server
git checkout -b phase-3b-3c-service-auth
```

- [ ] **Step 2: 確認基準測試全綠**

Run: `uv run pytest`
Expected: PASS（既有 68 個測試全過）

---

## Task 1: user_service（註冊 / 認證）

**Files:**
- Create: `src/home_server/services/user_service.py`
- Test: `tests/test_user_service.py`

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_user_service.py`:

```python
import pytest

from home_server.db.users import DuplicateUsernameError
from home_server.services import user_service
from home_server.services.user_service import WeakPasswordError


def test_register_and_authenticate(db_conn) -> None:
    uid = user_service.register(db_conn, username="alice", password="password1", cost=4)
    assert uid > 0
    user = user_service.authenticate(db_conn, username="alice", password="password1")
    assert user is not None
    assert user.id == uid
    assert user.username == "alice"


def test_register_rejects_weak_password(db_conn) -> None:
    with pytest.raises(WeakPasswordError):
        user_service.register(db_conn, username="bob", password="short", cost=4)


def test_register_rejects_duplicate_username(db_conn) -> None:
    user_service.register(db_conn, username="alice", password="password1", cost=4)
    with pytest.raises(DuplicateUsernameError):
        user_service.register(db_conn, username="alice", password="password2", cost=4)


def test_authenticate_wrong_password_returns_none(db_conn) -> None:
    user_service.register(db_conn, username="alice", password="password1", cost=4)
    assert user_service.authenticate(db_conn, username="alice", password="wrong-pass") is None


def test_authenticate_unknown_user_returns_none(db_conn) -> None:
    assert user_service.authenticate(db_conn, username="ghost", password="password1") is None


def test_password_hash_is_not_plaintext(db_conn) -> None:
    user_service.register(db_conn, username="alice", password="password1", cost=4)
    user = user_service.authenticate(db_conn, username="alice", password="password1")
    assert user is not None
    assert user.password_hash != "password1"
    assert user.password_hash.startswith("$2")  # bcrypt prefix
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_user_service.py -v`
Expected: FAIL（`ModuleNotFoundError: home_server.services.user_service`）

- [ ] **Step 3: 實作**

Create `src/home_server/services/user_service.py`:

```python
"""User registration and authentication (bcrypt password hashing)."""

from __future__ import annotations

import sqlite3

import bcrypt

from home_server.db import users
from home_server.db.users import User


class WeakPasswordError(ValueError):
    pass


_MIN_PASSWORD_LEN = 8


def hash_password(password: str, *, cost: int = 12) -> str:
    salt = bcrypt.gensalt(rounds=cost)
    return bcrypt.hashpw(password.encode("utf-8"), salt).decode("utf-8")


def register(
    conn: sqlite3.Connection,
    *,
    username: str,
    password: str,
    cost: int = 12,
) -> int:
    if len(password) < _MIN_PASSWORD_LEN:
        raise WeakPasswordError(
            f"password must be at least {_MIN_PASSWORD_LEN} characters"
        )
    password_hash = hash_password(password, cost=cost)
    return users.create(conn, username=username, password_hash=password_hash)


def authenticate(
    conn: sqlite3.Connection,
    *,
    username: str,
    password: str,
) -> User | None:
    user = users.get_by_username(conn, username)
    if user is None:
        return None
    if not bcrypt.checkpw(password.encode("utf-8"), user.password_hash.encode("utf-8")):
        return None
    return user
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_user_service.py -v`
Expected: PASS（6 passed）

- [ ] **Step 5: lint + type check**

Run: `uv run ruff check && uv run mypy src`
Expected: 全綠

- [ ] **Step 6: Commit**

```bash
git add src/home_server/services/user_service.py tests/test_user_service.py
git commit -m "feat(services): add user_service for registration and authentication"
```

---

## Task 2: device_service（裝置管理 + BLE 連線）

**Files:**
- Create: `src/home_server/services/device_service.py`
- Test: `tests/test_device_service.py`

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_device_service.py`:

```python
import pytest

from home_server.ble.interface import DiscoveredDevice
from home_server.ble.mock_manager import MockBLEManager
from home_server.db import devices, users
from home_server.db.devices import DeviceNotFoundError, DuplicateAddressError
from home_server.services.device_service import DeviceService, InvalidAddressError

ADDR = "AA:BB:CC:DD:EE:FF"


@pytest.fixture
def owner(db_conn) -> int:
    return users.create(db_conn, username="owner", password_hash="h")


def test_scan_returns_devices(db_conn) -> None:
    mock = MockBLEManager(scan_results=[DiscoveredDevice(ADDR, "STM32", -50)])
    svc = DeviceService(mock)
    found = svc.scan(5.0)
    assert found == [DiscoveredDevice(ADDR, "STM32", -50)]
    assert mock.scan_calls == [5.0]


def test_add_device_persists_and_connects(db_conn, owner) -> None:
    mock = MockBLEManager()
    svc = DeviceService(mock)
    device = svc.add_device(db_conn, owner_user_id=owner, address=ADDR, name="Living Room")
    assert device.address == ADDR
    assert device.name == "Living Room"
    assert mock.is_connected(ADDR)


def test_add_device_kept_when_connect_fails(db_conn, owner) -> None:
    mock = MockBLEManager(fail_connect_for={ADDR})
    svc = DeviceService(mock)
    device = svc.add_device(db_conn, owner_user_id=owner, address=ADDR, name="x")
    assert devices.get_by_id(db_conn, device.id) is not None
    assert not mock.is_connected(ADDR)


def test_add_device_rejects_invalid_address(db_conn, owner) -> None:
    svc = DeviceService(MockBLEManager())
    with pytest.raises(InvalidAddressError):
        svc.add_device(db_conn, owner_user_id=owner, address="not-a-mac", name="x")


def test_add_device_rejects_duplicate(db_conn, owner) -> None:
    svc = DeviceService(MockBLEManager())
    svc.add_device(db_conn, owner_user_id=owner, address=ADDR, name="x")
    with pytest.raises(DuplicateAddressError):
        svc.add_device(db_conn, owner_user_id=owner, address=ADDR, name="y")


def test_remove_device_disconnects_and_deletes(db_conn, owner) -> None:
    mock = MockBLEManager()
    svc = DeviceService(mock)
    device = svc.add_device(db_conn, owner_user_id=owner, address=ADDR, name="x")
    svc.remove_device(db_conn, device.id)
    assert devices.get_by_id(db_conn, device.id) is None
    assert not mock.is_connected(ADDR)


def test_remove_missing_device_raises(db_conn) -> None:
    svc = DeviceService(MockBLEManager())
    with pytest.raises(DeviceNotFoundError):
        svc.remove_device(db_conn, 999)


def test_list_devices(db_conn, owner) -> None:
    svc = DeviceService(MockBLEManager())
    svc.add_device(db_conn, owner_user_id=owner, address=ADDR, name="x")
    assert len(svc.list_devices(db_conn)) == 1
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_device_service.py -v`
Expected: FAIL（`ModuleNotFoundError: home_server.services.device_service`）

- [ ] **Step 3: 實作**

Create `src/home_server/services/device_service.py`:

```python
"""Device management: validate input, persist, drive BLE connection.

BLE operations call the BLEManager interface synchronously; the production
BluepyManager serializes each operation onto its per-peripheral worker thread,
so the service itself never touches threads.
"""

from __future__ import annotations

import logging
import re
import sqlite3

from home_server.ble.interface import BLEManager, DiscoveredDevice
from home_server.db import devices
from home_server.db.devices import Device, DeviceNotFoundError

log = logging.getLogger(__name__)


class InvalidAddressError(ValueError):
    pass


_MAC_RE = re.compile(r"^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$")


class DeviceService:
    def __init__(self, ble: BLEManager) -> None:
        self._ble = ble

    def scan(self, duration_s: float) -> list[DiscoveredDevice]:
        return self._ble.start_scan(duration_s)

    def add_device(
        self,
        conn: sqlite3.Connection,
        *,
        owner_user_id: int,
        address: str,
        name: str,
    ) -> Device:
        if not _MAC_RE.match(address):
            raise InvalidAddressError(f"invalid BLE address: {address!r}")
        device_id = devices.create(
            conn, address=address, name=name, owner_user_id=owner_user_id
        )
        # Best-effort initial connect; keep the device on failure for later retry.
        try:
            self._ble.connect(address)
        except Exception:
            log.warning(
                "Initial connect to %s failed; device kept for later retry",
                address,
                exc_info=True,
            )
        device = devices.get_by_id(conn, device_id)
        assert device is not None
        return device

    def remove_device(self, conn: sqlite3.Connection, device_id: int) -> None:
        device = devices.get_by_id(conn, device_id)
        if device is None:
            raise DeviceNotFoundError(f"device not found: id={device_id}")
        if self._ble.is_connected(device.address):
            self._ble.disconnect(device.address)
        devices.delete(conn, device_id)

    def list_devices(self, conn: sqlite3.Connection) -> list[Device]:
        return devices.list_all(conn)
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_device_service.py -v`
Expected: PASS（8 passed）

- [ ] **Step 5: lint + type check**

Run: `uv run ruff check && uv run mypy src`
Expected: 全綠

- [ ] **Step 6: Commit**

```bash
git add src/home_server/services/device_service.py tests/test_device_service.py
git commit -m "feat(services): add device_service with BLE connect on add"
```

---

## Task 3: channel_service（CRUD / 控制寫入 / notify 處理）

**Files:**
- Create: `src/home_server/services/channel_service.py`
- Test: `tests/test_channel_service.py`

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_channel_service.py`:

```python
from collections.abc import Callable

import pytest

from home_server.ble import parser
from home_server.ble.mock_manager import MockBLEManager
from home_server.ble.rate_limiter import RateLimiter
from home_server.db import channels, devices, readings, users
from home_server.db.channels import ChannelNotFoundError
from home_server.services.channel_service import ChannelService, WrongChannelTypeError

ADDR = "AA:BB:CC:DD:EE:FF"
CTRL_UUID = "0000aaaa-0000-1000-8000-00805f9b34fb"
DISP_UUID = "0000bbbb-0000-1000-8000-00805f9b34fb"


@pytest.fixture
def device_id(db_conn) -> int:
    uid = users.create(db_conn, username="u", password_hash="h")
    return devices.create(db_conn, address=ADDR, name="dev", owner_user_id=uid)


def _make_service(
    *,
    on_reading: Callable[[int, float, str], None] | None = None,
    min_interval: float = 1.0,
    clock: Callable[[], float] | None = None,
) -> tuple[ChannelService, MockBLEManager]:
    mock = MockBLEManager()
    mock.connect(ADDR)  # so write() is permitted
    limiter = (
        RateLimiter(min_interval)
        if clock is None
        else RateLimiter(min_interval, clock=clock)
    )
    svc = ChannelService(mock, limiter, on_reading or (lambda cid, value, ts: None))
    return svc, mock


def test_add_channel_rejects_unknown_format(db_conn, device_id) -> None:
    svc, _ = _make_service()
    with pytest.raises(parser.UnknownFormatError):
        svc.add_channel(
            db_conn,
            device_id=device_id,
            name="bad",
            type="display",
            char_uuid=DISP_UUID,
            data_format="nonsense",
        )


def test_write_command_encodes_and_writes(db_conn, device_id) -> None:
    svc, mock = _make_service()
    channel = svc.add_channel(
        db_conn,
        device_id=device_id,
        name="led",
        type="controller",
        char_uuid=CTRL_UUID,
        data_format="uint8",
    )
    svc.write_command(db_conn, channel_id=channel.id, raw_value=1)
    assert mock.writes_for(ADDR, CTRL_UUID) == [parser.encode(1, "uint8")]


def test_write_command_rejects_display_channel(db_conn, device_id) -> None:
    svc, _ = _make_service()
    channel = svc.add_channel(
        db_conn,
        device_id=device_id,
        name="temp",
        type="display",
        char_uuid=DISP_UUID,
        data_format="float32_le",
    )
    with pytest.raises(WrongChannelTypeError):
        svc.write_command(db_conn, channel_id=channel.id, raw_value=1)


def test_write_command_missing_channel(db_conn, device_id) -> None:
    svc, _ = _make_service()
    with pytest.raises(ChannelNotFoundError):
        svc.write_command(db_conn, channel_id=999, raw_value=1)


def test_handle_notify_decodes_emits_and_persists(db_conn, device_id) -> None:
    received: list[tuple[int, float, str]] = []
    svc, _ = _make_service(on_reading=lambda cid, value, ts: received.append((cid, value, ts)))
    channel = svc.add_channel(
        db_conn,
        device_id=device_id,
        name="temp",
        type="display",
        char_uuid=DISP_UUID,
        data_format="uint8",
    )
    value = svc.handle_notify(db_conn, channel_id=channel.id, raw_bytes=b"\x2a")
    assert value == 42.0
    assert received[0][0] == channel.id
    assert received[0][1] == 42.0
    assert readings.count_by_channel(db_conn, channel.id) == 1


def test_handle_notify_rate_limits_persistence_but_always_emits(db_conn, device_id) -> None:
    now = [0.0]
    received: list[tuple[int, float, str]] = []
    svc, _ = _make_service(
        on_reading=lambda cid, value, ts: received.append((cid, value, ts)),
        min_interval=10.0,
        clock=lambda: now[0],
    )
    channel = svc.add_channel(
        db_conn,
        device_id=device_id,
        name="temp",
        type="display",
        char_uuid=DISP_UUID,
        data_format="uint8",
    )
    svc.handle_notify(db_conn, channel_id=channel.id, raw_bytes=b"\x01")
    svc.handle_notify(db_conn, channel_id=channel.id, raw_bytes=b"\x02")  # within interval
    assert readings.count_by_channel(db_conn, channel.id) == 1  # 2nd not persisted
    assert len(received) == 2  # both pushed to UI


def test_get_history_returns_readings(db_conn, device_id) -> None:
    svc, _ = _make_service()
    channel = svc.add_channel(
        db_conn,
        device_id=device_id,
        name="temp",
        type="display",
        char_uuid=DISP_UUID,
        data_format="uint8",
    )
    svc.handle_notify(db_conn, channel_id=channel.id, raw_bytes=b"\x05")
    history = svc.get_history(db_conn, channel.id)
    assert len(history) == 1
    assert history[0].value == 5.0


def test_list_by_device(db_conn, device_id) -> None:
    svc, _ = _make_service()
    svc.add_channel(
        db_conn,
        device_id=device_id,
        name="temp",
        type="display",
        char_uuid=DISP_UUID,
        data_format="uint8",
    )
    assert len(svc.list_by_device(db_conn, device_id)) == 1
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_channel_service.py -v`
Expected: FAIL（`ModuleNotFoundError: home_server.services.channel_service`）

- [ ] **Step 3: 實作**

Create `src/home_server/services/channel_service.py`:

```python
"""Channel service: CRUD, control writes, and notify handling.

Notify handling is a plain method (`handle_notify`) taking the caller's
connection — the BLE worker-thread wiring that calls it lives in a later phase.
UI push (on_reading) is unthrottled; DB persistence is rate-limited per channel.
"""

from __future__ import annotations

import sqlite3
from collections.abc import Callable
from datetime import UTC, datetime

from home_server.ble import parser
from home_server.ble.interface import BLEManager
from home_server.ble.rate_limiter import RateLimiter
from home_server.db import channels, devices, readings
from home_server.db.channels import Channel, ChannelNotFoundError, ChannelType
from home_server.db.readings import Reading

# (channel_id, value, iso_utc_timestamp) pushed to the UI on every notify.
ReadingCallback = Callable[[int, float, str], None]

_TS_FMT = "%Y-%m-%d %H:%M:%S"


class WrongChannelTypeError(ValueError):
    pass


class ChannelService:
    def __init__(
        self,
        ble: BLEManager,
        limiter: RateLimiter,
        on_reading: ReadingCallback,
    ) -> None:
        self._ble = ble
        self._limiter = limiter
        self._on_reading = on_reading

    def add_channel(
        self,
        conn: sqlite3.Connection,
        *,
        device_id: int,
        name: str,
        type: ChannelType,
        char_uuid: str,
        data_format: str,
        unit: str | None = None,
    ) -> Channel:
        if data_format not in parser.supported_formats():
            raise parser.UnknownFormatError(f"unknown data_format: {data_format!r}")
        channel_id = channels.create(
            conn,
            device_id=device_id,
            name=name,
            type=type,
            char_uuid=char_uuid,
            data_format=data_format,
            unit=unit,
        )
        channel = channels.get_by_id(conn, channel_id)
        assert channel is not None
        return channel

    def write_command(
        self,
        conn: sqlite3.Connection,
        *,
        channel_id: int,
        raw_value: float,
    ) -> None:
        channel = channels.get_by_id(conn, channel_id)
        if channel is None:
            raise ChannelNotFoundError(f"channel not found: id={channel_id}")
        if channel.type != "controller":
            raise WrongChannelTypeError(f"channel {channel_id} is not a controller")
        data = parser.encode(raw_value, channel.data_format)
        device = devices.get_by_id(conn, channel.device_id)
        assert device is not None
        self._ble.write(device.address, channel.char_uuid, data)

    def handle_notify(
        self,
        conn: sqlite3.Connection,
        *,
        channel_id: int,
        raw_bytes: bytes,
    ) -> float:
        channel = channels.get_by_id(conn, channel_id)
        if channel is None:
            raise ChannelNotFoundError(f"channel not found: id={channel_id}")
        value = parser.decode(raw_bytes, channel.data_format)
        timestamp = datetime.now(UTC).strftime(_TS_FMT)
        self._on_reading(channel_id, value, timestamp)  # UI push, unthrottled
        if self._limiter.should_emit(str(channel_id)):
            readings.insert(conn, channel_id=channel_id, value=value)
        return value

    def get_history(
        self,
        conn: sqlite3.Connection,
        channel_id: int,
        *,
        since: datetime | None = None,
        until: datetime | None = None,
        limit: int | None = None,
    ) -> list[Reading]:
        return readings.list_by_channel(
            conn, channel_id, since=since, until=until, limit=limit
        )

    def list_by_device(
        self, conn: sqlite3.Connection, device_id: int
    ) -> list[Channel]:
        return channels.list_by_device(conn, device_id)
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_channel_service.py -v`
Expected: PASS（8 passed）

- [ ] **Step 5: lint + type check**

Run: `uv run ruff check && uv run mypy src`
Expected: 全綠

- [ ] **Step 6: Commit**

```bash
git add src/home_server/services/channel_service.py tests/test_channel_service.py
git commit -m "feat(services): add channel_service for control writes and notify handling"
```

---

## Task 4: 認證 Blueprint 與 application factory

**Files:**
- Create: `src/home_server/web/auth.py`
- Create: `src/home_server/web/templates/base.html`
- Create: `src/home_server/web/templates/index.html`
- Create: `src/home_server/web/templates/auth/login.html`
- Create: `src/home_server/web/templates/auth/register.html`
- Modify (overwrite): `src/home_server/web/__init__.py`
- Modify: `src/home_server/__main__.py`
- Modify: `tests/conftest.py`
- Modify: `pyproject.toml`（mypy overrides）
- Test: `tests/test_web_auth.py`

- [ ] **Step 1: 為第三方套件加 mypy overrides**

`flask_login` / `flask_wtf` 缺完整型別資訊，strict mypy 會報 missing stubs。Append to `pyproject.toml`:

```toml
[[tool.mypy.overrides]]
module = ["flask_login.*", "flask_wtf.*", "wtforms.*"]
ignore_missing_imports = true
```

- [ ] **Step 2: 寫 application factory**

Overwrite `src/home_server/web/__init__.py`:

```python
"""Flask application factory and per-request DB connection helper."""

from __future__ import annotations

import sqlite3

from flask import Flask, current_app, g, render_template
from flask_login import LoginManager, current_user, login_required
from flask_wtf import CSRFProtect

from home_server.config import Config
from home_server.db import connection, users

login_manager = LoginManager()
csrf = CSRFProtect()


def get_conn() -> sqlite3.Connection:
    """Return the per-request connection, opening one on first use."""
    if "conn" not in g:
        g.conn = connection.connect(current_app.config["DB_PATH"])
    return g.conn


def create_app(config: Config) -> Flask:
    app = Flask(__name__)
    app.config["SECRET_KEY"] = config.secret_key
    app.config["DB_PATH"] = config.db_path

    login_manager.init_app(app)
    login_manager.login_view = "auth.login"
    csrf.init_app(app)

    from home_server.web.auth import LoginUser
    from home_server.web.auth import bp as auth_bp

    @login_manager.user_loader
    def load_user(user_id: str) -> LoginUser | None:
        user = users.get_by_id(get_conn(), int(user_id))
        return LoginUser(user) if user is not None else None

    app.register_blueprint(auth_bp)

    @app.get("/")
    @login_required
    def index() -> str:
        return render_template("index.html")

    @app.get("/health")
    def health() -> dict[str, str]:
        return {"status": "ok"}

    @app.teardown_appcontext
    def close_conn(exc: BaseException | None) -> None:
        conn = g.pop("conn", None)
        if conn is not None:
            conn.close()

    return app
```

- [ ] **Step 3: 寫 auth blueprint**

Create `src/home_server/web/auth.py`:

```python
"""Authentication blueprint: register, login, logout."""

from __future__ import annotations

from flask import (
    Blueprint,
    flash,
    redirect,
    render_template,
    request,
    url_for,
)
from flask_login import UserMixin, login_required, login_user, logout_user
from flask_wtf import FlaskForm
from werkzeug.wrappers import Response
from wtforms import PasswordField, StringField
from wtforms.validators import DataRequired, EqualTo, Length

from home_server.db import users
from home_server.db.users import DuplicateUsernameError, User
from home_server.services import user_service

bp = Blueprint("auth", __name__, url_prefix="/auth")


class LoginUser(UserMixin):
    def __init__(self, user: User) -> None:
        self.user = user

    def get_id(self) -> str:
        return str(self.user.id)

    @property
    def username(self) -> str:
        return self.user.username


class LoginForm(FlaskForm):
    username = StringField("Username", validators=[DataRequired()])
    password = PasswordField("Password", validators=[DataRequired()])


class RegisterForm(FlaskForm):
    username = StringField("Username", validators=[DataRequired()])
    password = PasswordField("Password", validators=[DataRequired(), Length(min=8)])
    confirm = PasswordField(
        "Confirm", validators=[DataRequired(), EqualTo("password")]
    )


def _safe_next(target: str | None) -> str:
    # Only allow relative redirects to avoid open-redirect.
    if target and target.startswith("/") and not target.startswith("//"):
        return target
    return url_for("index")


@bp.route("/register", methods=["GET", "POST"])
def register() -> Response | str:
    from home_server.web import get_conn

    form = RegisterForm()
    if form.validate_on_submit():
        try:
            uid = user_service.register(
                get_conn(), username=form.username.data, password=form.password.data
            )
        except DuplicateUsernameError:
            flash("Username already taken")
            return render_template("auth/register.html", form=form)
        except user_service.WeakPasswordError:
            flash("Password too weak")
            return render_template("auth/register.html", form=form)
        user = users.get_by_id(get_conn(), uid)
        assert user is not None
        login_user(LoginUser(user))
        return redirect(url_for("index"))
    return render_template("auth/register.html", form=form)


@bp.route("/login", methods=["GET", "POST"])
def login() -> Response | str:
    from home_server.web import get_conn

    form = LoginForm()
    if form.validate_on_submit():
        user = user_service.authenticate(
            get_conn(), username=form.username.data, password=form.password.data
        )
        if user is None:
            flash("Invalid username or password")
            return render_template("auth/login.html", form=form)
        login_user(LoginUser(user))
        return redirect(_safe_next(request.args.get("next")))
    return render_template("auth/login.html", form=form)


@bp.route("/logout", methods=["POST"])
@login_required
def logout() -> Response:
    logout_user()
    return redirect(url_for("auth.login"))
```

- [ ] **Step 4: 寫模板**

Create `src/home_server/web/templates/base.html`:

```html
<!doctype html>
<html lang="zh-Hant">
<head>
  <meta charset="utf-8">
  <title>{% block title %}Home Server{% endblock %}</title>
</head>
<body>
  {% with messages = get_flashed_messages() %}
    {% if messages %}
      <ul class="flash">
        {% for message in messages %}<li>{{ message }}</li>{% endfor %}
      </ul>
    {% endif %}
  {% endwith %}
  {% block content %}{% endblock %}
</body>
</html>
```

Create `src/home_server/web/templates/index.html`:

```html
{% extends "base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<p>Logged in as {{ current_user.username }}</p>
<form method="post" action="{{ url_for('auth.logout') }}">
  <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
  <button type="submit">Logout</button>
</form>
{% endblock %}
```

Create `src/home_server/web/templates/auth/login.html`:

```html
{% extends "base.html" %}
{% block title %}Login{% endblock %}
{% block content %}
<h1>Login</h1>
<form method="post">
  {{ form.csrf_token }}
  <p>{{ form.username.label }} {{ form.username() }}</p>
  <p>{{ form.password.label }} {{ form.password() }}</p>
  <button type="submit">Login</button>
</form>
<a href="{{ url_for('auth.register') }}">Register</a>
{% endblock %}
```

Create `src/home_server/web/templates/auth/register.html`:

```html
{% extends "base.html" %}
{% block title %}Register{% endblock %}
{% block content %}
<h1>Register</h1>
<form method="post">
  {{ form.csrf_token }}
  <p>{{ form.username.label }} {{ form.username() }}</p>
  <p>{{ form.password.label }} {{ form.password() }}</p>
  <p>{{ form.confirm.label }} {{ form.confirm() }}</p>
  <button type="submit">Register</button>
</form>
<a href="{{ url_for('auth.login') }}">Login</a>
{% endblock %}
```

- [ ] **Step 5: 改寫入口點**

Overwrite `src/home_server/__main__.py`:

```python
"""Entry point: `python -m home_server`."""

from __future__ import annotations

import logging

from home_server.config import Config
from home_server.core.logging import setup_logging
from home_server.db import connection
from home_server.web import create_app


def main() -> None:
    config = Config.from_env()
    setup_logging(config.log_level)
    log = logging.getLogger(__name__)

    log.info("Initializing database at %s", config.db_path)
    connection.initialize(config.db_path)

    app = create_app(config)
    log.info(
        "Starting Flask app on %s:%d (debug=%s)", config.host, config.port, config.debug
    )
    app.run(host=config.host, port=config.port, debug=config.debug, use_reloader=False)


if __name__ == "__main__":
    main()
```

- [ ] **Step 6: 加 conftest fixtures**

Append to `tests/conftest.py`（保留既有 `db_conn`；新增 imports 置於檔首既有 import 區）:

```python
import re

from flask import Flask
from flask.testing import FlaskClient

from home_server.ble.mock_manager import MockBLEManager
from home_server.config import Config
from home_server.db import connection
from home_server.web import create_app


@pytest.fixture
def mock_ble() -> MockBLEManager:
    return MockBLEManager()


@pytest.fixture
def app(tmp_path) -> Flask:
    # File-backed DB (not :memory:) so each per-request connection sees the
    # same database — distinct :memory: connections would each be empty.
    db_path = tmp_path / "test.db"
    connection.initialize(db_path)
    config = Config(
        db_path=db_path,
        secret_key="test-secret",
        host="127.0.0.1",
        port=5000,
        log_level="INFO",
        ble_scan_duration=1.0,
        reading_min_interval=1.0,
        debug=True,
    )
    flask_app = create_app(config)
    flask_app.config["TESTING"] = True
    return flask_app


@pytest.fixture
def client(app: Flask) -> FlaskClient:
    return app.test_client()


def _csrf_token(client: FlaskClient, path: str) -> str:
    html = client.get(path).get_data(as_text=True)
    match = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    assert match, f"csrf_token not found at {path}"
    return match.group(1)


@pytest.fixture
def logged_in_client(client: FlaskClient) -> FlaskClient:
    token = _csrf_token(client, "/auth/register")
    client.post(
        "/auth/register",
        data={
            "username": "tester",
            "password": "password1",
            "confirm": "password1",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    return client
```

- [ ] **Step 7: 寫測試**

Create `tests/test_web_auth.py`:

```python
import re

from flask.testing import FlaskClient


def _csrf_token(client: FlaskClient, path: str) -> str:
    html = client.get(path).get_data(as_text=True)
    match = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    assert match, f"csrf_token not found at {path}"
    return match.group(1)


def test_health_ok(client) -> None:
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.get_json() == {"status": "ok"}


def test_login_page_renders_with_csrf(client) -> None:
    resp = client.get("/auth/login")
    assert resp.status_code == 200
    assert b'name="csrf_token"' in resp.data


def test_unauthenticated_index_redirects_to_login(client) -> None:
    resp = client.get("/")
    assert resp.status_code == 302
    assert "/auth/login" in resp.headers["Location"]


def test_register_then_access_index(client) -> None:
    token = _csrf_token(client, "/auth/register")
    resp = client.post(
        "/auth/register",
        data={
            "username": "alice",
            "password": "password1",
            "confirm": "password1",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert b"Logged in as alice" in resp.data


def test_register_duplicate_shows_error(client) -> None:
    token = _csrf_token(client, "/auth/register")
    client.post(
        "/auth/register",
        data={
            "username": "alice",
            "password": "password1",
            "confirm": "password1",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    # Re-register same username in a fresh request.
    token = _csrf_token(client, "/auth/register")
    resp = client.post(
        "/auth/register",
        data={
            "username": "alice",
            "password": "password2",
            "confirm": "password2",
            "csrf_token": token,
        },
    )
    assert resp.status_code == 200
    assert b"Username already taken" in resp.data


def test_login_logout_flow(client) -> None:
    # Register (auto-logs in), then log out, then log back in.
    token = _csrf_token(client, "/auth/register")
    client.post(
        "/auth/register",
        data={
            "username": "bob",
            "password": "password1",
            "confirm": "password1",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    logout_token = _csrf_token(client, "/")
    client.post("/auth/logout", data={"csrf_token": logout_token}, follow_redirects=True)
    assert client.get("/").status_code == 302  # logged out

    token = _csrf_token(client, "/auth/login")
    resp = client.post(
        "/auth/login",
        data={"username": "bob", "password": "password1", "csrf_token": token},
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert b"Logged in as bob" in resp.data


def test_login_wrong_password_shows_error(client) -> None:
    token = _csrf_token(client, "/auth/register")
    client.post(
        "/auth/register",
        data={
            "username": "carol",
            "password": "password1",
            "confirm": "password1",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    token = _csrf_token(client, "/auth/login")
    resp = client.post(
        "/auth/login",
        data={"username": "carol", "password": "wrong-pass", "csrf_token": token},
    )
    assert resp.status_code == 200
    assert b"Invalid username or password" in resp.data
```

- [ ] **Step 8: 跑全部測試**

Run: `uv run pytest -v`
Expected: PASS（既有 68 + Task 1–4 全部新測試）

- [ ] **Step 9: lint + type check**

Run: `uv run ruff check && uv run mypy src`
Expected: 全綠（若 mypy 仍報某套件 missing stubs，將其 module 加入 Step 1 的 overrides 清單後重跑）

- [ ] **Step 10: 手動冒煙測試（可選，非 Linux 亦可）**

```bash
HOME_SERVER_DEBUG=1 uv run python -m home_server
```
瀏覽器開 `http://127.0.0.1:5000/auth/register`，註冊 → 自動登入看到 "Logged in as ..." → Logout。Ctrl-C 結束。

- [ ] **Step 11: Commit**

```bash
git add src/home_server/web/ src/home_server/__main__.py tests/conftest.py tests/test_web_auth.py pyproject.toml
git commit -m "feat(web): add auth blueprint, app factory, and login/register/logout flow"
```

---

## 完成後

- [ ] 確認 `uv run pytest`、`uv run ruff check`、`uv run mypy src` 三者全綠
- [ ] 在 submodule push 分支並開 PR（待使用者指示）
- [ ] 在主 repo 更新 submodule pointer 與 README 進度（待使用者指示）

---

## Self-review notes（撰寫時自查）

- **Spec coverage:** user/device/channel service（§3）→ Task 1–3；create_app + auth + 模板（§4）→ Task 4；conftest fixtures（§5）→ Task 4 Step 6；驗收（§6）→ 各 Task lint/type/test 步驟 + Task 4 Step 8–10。
- **明確排除項**（SocketIO、subscribe wiring、重連迴圈、3d CRUD、change_password）皆未出現在任何 task。
- **型別一致性:** `ReadingCallback` 簽章 `(int, float, str)` 在 service 與測試一致；`LoginUser`/`get_conn`/`create_app` 跨 Task 4 各檔一致；`InvalidAddressError`/`WrongChannelTypeError`/`WeakPasswordError` 定義與測試 import 路徑一致。
