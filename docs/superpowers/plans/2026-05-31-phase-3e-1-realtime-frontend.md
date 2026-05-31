# Phase 3e-1 — SocketIO 即時推播 + HTMX/Chart.js 前端 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 接線「BLE Notify → SocketIO 即時推播 → 前端 Chart.js 趨勢圖」，並補上 `/devices/scan`、`/channels/<id>/write`、`/channels/<id>/history` 三個 Web 端點與 HTMX/Chart.js 前端，全程可在 macOS + `MockBLEManager` 端到端驗證。

**Architecture:** Flask application factory 既有結構不變；新增模組級 `SocketIO` 單例與 `services/ble_runtime.py` 協調器（連線裝置、訂閱 display 頻道、notify→`handle_notify`）。`create_app` 仍回傳 `Flask`；副作用（背景執行緒、連線）只在 `__main__` 觸發，測試不起執行緒。控制寫入沿用既有 `write_command` 依頻道 `data_format` 編碼。

**Tech Stack:** Python 3.11 / Flask / Flask-SocketIO（threading，已在相依）/ SQLite / HTMX + Chart.js + socket.io client（vendored）/ pytest / ruff / mypy(strict)。

> 對應規格：`docs/superpowers/specs/2026-05-31-phase-3e-realtime-frontend-design.md`
> 工作目錄：`Intelligent-home-RPi-server/`（以下相對路徑皆以此為根；指令請在此目錄執行）。
> 驗證指令（每個 commit 前）：`uv run ruff check && uv run mypy src && uv run pytest`
> 重要：mypy 只檢查 `src`，不檢查 `tests`；ruff 檢查全部。測試碼須通過 ruff（import 排序、行長 100）。

---

## Task 1: 測試 harness — 讓測試能取得 app 內的 MockBLEManager

**Files:**
- Modify: `tests/conftest.py`

目前 `app` fixture 內部自建 mock，測試無法斷言 `mock.writes` 或設定 `scan_results`。新增共用 `mock_ble` fixture 並讓 `app` 注入它。既有測試不受影響（它們不請求 `mock_ble`）。

- [ ] **Step 1: 修改 conftest 匯入**

在 `tests/conftest.py` 既有匯入區，將 home_server 匯入改為（新增 `MockBLEManager`，排序在最前）：

```python
from home_server.ble.mock_manager import MockBLEManager
from home_server.config import Config
from home_server.db import connection
from home_server.web import create_app
```

- [ ] **Step 2: 新增 `mock_ble` fixture 並修改 `app` fixture**

把現有 `app` fixture 換成下列（新增 `mock_ble` fixture，`app` 多一個參數並以 `ble=mock_ble` 注入）：

```python
@pytest.fixture
def mock_ble() -> MockBLEManager:
    """Shared MockBLEManager so tests can set scan_results and assert on writes."""
    return MockBLEManager()


@pytest.fixture
def app(tmp_path: Path, mock_ble: MockBLEManager) -> Flask:
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
        admin_username="admin",
        admin_password=None,
        debug=True,
    )
    flask_app = create_app(config, ble=mock_ble)
    flask_app.config["TESTING"] = True
    return flask_app
```

- [ ] **Step 3: 跑既有測試確認無回歸**

Run: `uv run ruff check && uv run mypy src && uv run pytest`
Expected: PASS（仍為 125 passed；mypy/ruff 全綠）。

- [ ] **Step 4: Commit**

```bash
git add tests/conftest.py
git commit -m "test: expose app's MockBLEManager via mock_ble fixture"
```

---

## Task 2: `POST /channels/<id>/write` 控制寫入端點

**Files:**
- Modify: `src/home_server/web/channels.py`
- Test: `tests/test_web_channels.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_web_channels.py` 末端追加（沿用既有 `_csrf_token` / `_make_device`）：

```python
def test_write_controller_sends_encoded_bytes(
    app: Flask, logged_in_client: FlaskClient, mock_ble
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:21", "Lamp")
    mock_ble.connect("AA:BB:CC:DD:EE:21")  # write() requires a connected handle
    conn = connection.connect(app.config["DB_PATH"])
    try:
        channel_id = channels.create(
            conn, device_id=device_id, name="LED", type="controller",
            char_uuid="uuid-led", data_format="uint8", unit=None,
        )
    finally:
        conn.close()
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/channels/{channel_id}/write",
        data={"value": "1", "csrf_token": token},
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert mock_ble.writes_for("AA:BB:CC:DD:EE:21", "uuid-led") == [b"\x01"]


def test_write_display_channel_rejected(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:22", "Sensor")
    conn = connection.connect(app.config["DB_PATH"])
    try:
        channel_id = channels.create(
            conn, device_id=device_id, name="Temp", type="display",
            char_uuid="uuid-t", data_format="uint8", unit=None,
        )
    finally:
        conn.close()
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/channels/{channel_id}/write",
        data={"value": "1", "csrf_token": token},
    )
    assert resp.status_code == 400


def test_write_non_numeric_value_flashes(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:23", "Lamp2")
    conn = connection.connect(app.config["DB_PATH"])
    try:
        channel_id = channels.create(
            conn, device_id=device_id, name="LED", type="controller",
            char_uuid="uuid-led", data_format="uint8", unit=None,
        )
    finally:
        conn.close()
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/channels/{channel_id}/write",
        data={"value": "abc", "csrf_token": token},
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert b"Value must be a number" in resp.data


def test_write_missing_channel_404(logged_in_client: FlaskClient) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    resp = logged_in_client.post(
        "/channels/999/write", data={"value": "1", "csrf_token": token}
    )
    assert resp.status_code == 404
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_web_channels.py::test_write_controller_sends_encoded_bytes -v`
Expected: FAIL（404，route 不存在）。

- [ ] **Step 3: 實作端點**

在 `src/home_server/web/channels.py`：

匯入區改為（新增 `jsonify, request` 並於 channel_service 匯入 `WrongChannelTypeError`）：

```python
from flask import (
    Blueprint,
    abort,
    flash,
    jsonify,
    redirect,
    render_template,
    request,
    url_for,
)
from flask_login import login_required
from flask_wtf import FlaskForm
from werkzeug.wrappers import Response
from wtforms import SelectField, StringField
from wtforms.validators import DataRequired

from home_server.ble import parser
from home_server.db import channels, devices
from home_server.db.channels import DuplicateChannelNameError
from home_server.services.channel_service import WrongChannelTypeError
from home_server.web.db import get_conn
from home_server.web.services import get_channel_service
```

在檔案末端（`delete_channel` 之後）追加：

```python
@bp.post("/channels/<int:channel_id>/write")
@login_required
def write_channel(channel_id: int) -> Response:
    conn = get_conn()
    channel = channels.get_by_id(conn, channel_id)
    if channel is None:
        abort(404)
    raw = request.form.get("value", "").strip()
    try:
        value = float(raw)
    except ValueError:
        flash("Value must be a number")
        return redirect(url_for("devices.detail", device_id=channel.device_id))
    try:
        get_channel_service().write_command(conn, channel_id=channel_id, raw_value=value)
    except WrongChannelTypeError:
        abort(400)
    return redirect(url_for("devices.detail", device_id=channel.device_id))
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_web_channels.py -v`
Expected: PASS（新增 4 個測試全綠）。

- [ ] **Step 5: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/channels.py tests/test_web_channels.py
git commit -m "feat(web): add POST /channels/<id>/write control endpoint"
```

---

## Task 3: `GET /channels/<id>/history` 歷史資料 JSON 端點

**Files:**
- Modify: `src/home_server/web/channels.py`
- Test: `tests/test_web_channels.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_web_channels.py` 末端追加（新增 `readings` 匯入；把第 6 行匯入改為 `from home_server.db import channels, connection, devices, readings`）：

```python
def test_history_returns_readings_oldest_first(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:31", "Sensor")
    conn = connection.connect(app.config["DB_PATH"])
    try:
        channel_id = channels.create(
            conn, device_id=device_id, name="Temp", type="display",
            char_uuid="uuid-t", data_format="uint8", unit="C",
        )
        readings.insert(conn, channel_id=channel_id, value=24.5)
        readings.insert(conn, channel_id=channel_id, value=25.0)
    finally:
        conn.close()
    resp = logged_in_client.get(f"/channels/{channel_id}/history")
    assert resp.status_code == 200
    body = resp.get_json()
    assert body["channel_id"] == channel_id
    assert [r["value"] for r in body["readings"]] == [24.5, 25.0]


def test_history_empty(app: Flask, logged_in_client: FlaskClient) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:32", "Sensor2")
    conn = connection.connect(app.config["DB_PATH"])
    try:
        channel_id = channels.create(
            conn, device_id=device_id, name="Temp", type="display",
            char_uuid="uuid-t", data_format="uint8", unit=None,
        )
    finally:
        conn.close()
    resp = logged_in_client.get(f"/channels/{channel_id}/history")
    assert resp.status_code == 200
    assert resp.get_json()["readings"] == []


def test_history_missing_channel_404(logged_in_client: FlaskClient) -> None:
    resp = logged_in_client.get("/channels/999/history")
    assert resp.status_code == 404
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_web_channels.py::test_history_returns_readings_oldest_first -v`
Expected: FAIL（404）。

- [ ] **Step 3: 實作端點**

在 `src/home_server/web/channels.py` 末端追加：

```python
@bp.get("/channels/<int:channel_id>/history")
@login_required
def channel_history(channel_id: int) -> Response:
    conn = get_conn()
    channel = channels.get_by_id(conn, channel_id)
    if channel is None:
        abort(404)
    limit_raw = request.args.get("limit", type=int)
    limit = 200 if limit_raw is None else max(1, min(limit_raw, 1000))
    history = get_channel_service().get_history(conn, channel_id, limit=limit)
    return jsonify(
        {
            "channel_id": channel_id,
            "readings": [
                {"value": r.value, "recorded_at": r.recorded_at} for r in history
            ],
        }
    )
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_web_channels.py -v`
Expected: PASS。

- [ ] **Step 5: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/channels.py tests/test_web_channels.py
git commit -m "feat(web): add GET /channels/<id>/history JSON endpoint"
```

---

## Task 4: `GET /devices/scan` 掃描端點 + HTMX partial

**Files:**
- Modify: `src/home_server/web/devices.py`
- Modify: `src/home_server/web/__init__.py`（在 app.config 加入 `BLE_SCAN_DURATION`）
- Create: `src/home_server/web/templates/devices/_scan_results.html`
- Test: `tests/test_web_devices.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_web_devices.py` 末端追加（新增匯入 `from home_server.ble.interface import DiscoveredDevice`，置於 home_server 匯入區最前）：

```python
def test_scan_lists_discovered_devices(
    logged_in_client: FlaskClient, mock_ble
) -> None:
    mock_ble.scan_results = [
        DiscoveredDevice(address="11:22:33:44:55:66", name="Node-A", rssi=-50)
    ]
    resp = logged_in_client.get("/devices/scan")
    assert resp.status_code == 200
    body = resp.get_data(as_text=True)
    assert "11:22:33:44:55:66" in body
    assert "Node-A" in body


def test_scan_requires_login(client: FlaskClient) -> None:
    resp = client.get("/devices/scan")
    assert resp.status_code == 302
    assert "/auth/login" in resp.headers["Location"]
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_web_devices.py::test_scan_lists_discovered_devices -v`
Expected: FAIL（404）。

- [ ] **Step 3: 在 create_app 暴露掃描秒數**

在 `src/home_server/web/__init__.py` 的 `create_app` 內，緊接 `app.config["DB_PATH"] = config.db_path` 之後加一行：

```python
    app.config["BLE_SCAN_DURATION"] = config.ble_scan_duration
```

- [ ] **Step 4: 實作掃描端點**

在 `src/home_server/web/devices.py` 匯入區的 flask 匯入加入 `current_app`：

```python
from flask import (
    Blueprint,
    abort,
    current_app,
    flash,
    redirect,
    render_template,
    url_for,
)
```

在檔案末端（`delete` 之後）追加：

```python
@bp.get("/devices/scan")
@login_required
def scan() -> str:
    found = get_device_service().scan(current_app.config["BLE_SCAN_DURATION"])
    return render_template("devices/_scan_results.html", found=found)
```

- [ ] **Step 5: 建立 partial 模板**

Create `src/home_server/web/templates/devices/_scan_results.html`：

```html
{% if found %}
<ul class="list-group">
  {% for d in found %}
  <li class="list-group-item d-flex justify-content-between align-items-center flex-wrap gap-2">
    <span>
      <code>{{ d.address }}</code>
      {{ d.name or "(unknown)" }}
      <small class="text-muted">{{ d.rssi }} dBm</small>
    </span>
    <form method="post" action="{{ url_for('devices.list_devices') }}" class="d-flex gap-2">
      <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
      <input type="hidden" name="address" value="{{ d.address }}">
      <input class="form-control form-control-sm" name="name" value="{{ d.name or d.address }}">
      <button class="btn btn-sm btn-primary" type="submit">Add</button>
    </form>
  </li>
  {% endfor %}
</ul>
{% else %}
<p class="text-muted mb-0">No devices found.</p>
{% endif %}
```

- [ ] **Step 6: 跑測試確認通過**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: PASS。

- [ ] **Step 7: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/devices.py src/home_server/web/__init__.py src/home_server/web/templates/devices/_scan_results.html tests/test_web_devices.py
git commit -m "feat(web): add GET /devices/scan with HTMX results partial"
```

---

## Task 5: `MockBLEManager` 背景產數（start / stop / _tick）

**Files:**
- Modify: `src/home_server/ble/mock_manager.py`
- Test: `tests/test_mock_manager.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_mock_manager.py` 末端追加（在第 1 行 `import pytest` 之上加 `import time`，使匯入成為 `import time` 然後空行 `import pytest`）：

```python
def test_tick_emits_to_subscriber() -> None:
    mgr = MockBLEManager()
    mgr.connect("aa:bb")
    received: list[bytes] = []
    mgr.subscribe("aa:bb", "uuid-x", received.append)
    mgr._tick(lambda address, char_uuid: b"\x07")
    assert received == [b"\x07"]


def test_tick_skips_when_feed_returns_none() -> None:
    mgr = MockBLEManager()
    mgr.connect("aa:bb")
    received: list[bytes] = []
    mgr.subscribe("aa:bb", "uuid-x", received.append)
    mgr._tick(lambda address, char_uuid: None)
    assert received == []


def test_start_then_stop_emits_and_terminates() -> None:
    mgr = MockBLEManager()
    mgr.connect("aa:bb")
    received: list[bytes] = []
    mgr.subscribe("aa:bb", "uuid-x", received.append)
    mgr.start(lambda address, char_uuid: b"\x01", interval_s=0.01)
    time.sleep(0.05)
    mgr.stop()
    assert len(received) >= 1
    assert mgr._producer_thread is None
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_mock_manager.py::test_tick_emits_to_subscriber -v`
Expected: FAIL（`_tick` 不存在）。

- [ ] **Step 3: 實作 start / stop / _tick**

在 `src/home_server/ble/mock_manager.py`：

匯入區加入 `threading` 與 `Callable`：

```python
from __future__ import annotations

import threading
from collections.abc import Callable
from dataclasses import dataclass, field

from .interface import ConnectionHandle, DiscoveredDevice, NotifyCallback
```

在 dataclass 欄位區末端（`scan_calls` 之後）新增兩個非比較欄位：

```python
    scan_calls: list[float] = field(default_factory=list)

    # Dev-only synthetic producer (see start()/stop()). Not part of equality.
    _producer_thread: threading.Thread | None = field(
        default=None, compare=False, repr=False
    )
    _producer_stop: threading.Event | None = field(
        default=None, compare=False, repr=False
    )
```

在 `# ---- Test helpers ----` 區塊之前（或 `_require_connected` 之前）新增方法：

```python
    # ---- Dev-only synthetic producer ----

    def start(
        self, feed: Callable[[str, str], bytes | None], interval_s: float = 1.0
    ) -> None:
        """Begin emitting synthetic notifications to active subscribers.

        ``feed(address, char_uuid)`` returns the bytes to deliver, or None to
        skip that subscription this tick. Dev/demo use only — tests drive
        notifications synchronously via simulate_notify().
        """
        if self._producer_thread is not None:
            return
        stop = threading.Event()

        def _run() -> None:
            while not stop.wait(interval_s):
                self._tick(feed)

        thread = threading.Thread(target=_run, name="mock-ble-producer", daemon=True)
        self._producer_stop = stop
        self._producer_thread = thread
        thread.start()

    def stop(self) -> None:
        if self._producer_stop is not None:
            self._producer_stop.set()
        if self._producer_thread is not None:
            self._producer_thread.join(timeout=2.0)
        self._producer_thread = None
        self._producer_stop = None

    def _tick(self, feed: Callable[[str, str], bytes | None]) -> None:
        for (address, char_uuid), callback in list(self._subscriptions.items()):
            data = feed(address, char_uuid)
            if data is not None:
                callback(data)
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_mock_manager.py -v`
Expected: PASS。

- [ ] **Step 5: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/ble/mock_manager.py tests/test_mock_manager.py
git commit -m "feat(ble): add MockBLEManager synthetic producer (start/stop/_tick)"
```

---

## Task 6: `BleRuntime` 協調器 + `get_ble_runtime` 存取器

**Files:**
- Create: `src/home_server/services/ble_runtime.py`
- Modify: `src/home_server/web/services.py`
- Test: `tests/test_ble_runtime.py`

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_ble_runtime.py`：

```python
"""Tests for BleRuntime notify wiring and synthetic feed."""

from pathlib import Path

import pytest

from home_server.ble import parser
from home_server.ble.mock_manager import MockBLEError, MockBLEManager
from home_server.ble.rate_limiter import RateLimiter
from home_server.db import channels, connection, devices, readings, users
from home_server.services.ble_runtime import BleRuntime
from home_server.services.channel_service import ChannelService

ADDR = "AA:BB:CC:DD:EE:FF"
DISP_UUID = "uuid-disp"
CTRL_UUID = "uuid-ctrl"


@pytest.fixture
def db_path(tmp_path: Path) -> Path:
    path = tmp_path / "rt.db"
    connection.initialize(path)
    conn = connection.connect(path)
    try:
        uid = users.create(conn, username="u", password_hash="x")
        did = devices.create(conn, address=ADDR, name="dev", owner_user_id=uid)
        channels.create(
            conn, device_id=did, name="temp", type="display",
            char_uuid=DISP_UUID, data_format="uint8", unit=None,
        )
        channels.create(
            conn, device_id=did, name="led", type="controller",
            char_uuid=CTRL_UUID, data_format="uint8", unit=None,
        )
    finally:
        conn.close()
    return path


def _runtime(
    db_path: Path, ble: MockBLEManager
) -> tuple[BleRuntime, list[tuple[int, float, str]]]:
    seen: list[tuple[int, float, str]] = []
    svc = ChannelService(
        ble, RateLimiter(0.0), lambda cid, value, ts: seen.append((cid, value, ts))
    )
    rt = BleRuntime(
        ble, svc,
        conn_factory=lambda: connection.connect(db_path),
        scan_duration=1.0,
    )
    return rt, seen


def test_activate_connects_and_subscribes_only_display(db_path: Path) -> None:
    ble = MockBLEManager()
    rt, _ = _runtime(db_path, ble)
    rt.activate()
    assert ble.is_connected(ADDR)
    # display channel is subscribed; controller channel is not
    ble.simulate_notify(ADDR, DISP_UUID, b"\x2a")  # no error
    with pytest.raises(MockBLEError):
        ble.simulate_notify(ADDR, CTRL_UUID, b"\x01")


def test_notify_persists_reading_and_calls_on_reading(db_path: Path) -> None:
    ble = MockBLEManager()
    rt, seen = _runtime(db_path, ble)
    rt.activate()
    ble.simulate_notify(ADDR, DISP_UUID, b"\x2a")
    assert seen and seen[0][1] == 42.0
    conn = connection.connect(db_path)
    try:
        rows = readings.list_by_channel(conn, 1)  # display channel id == 1
    finally:
        conn.close()
    assert [r.value for r in rows] == [42.0]


def test_make_feed_encodes_known_channel_and_skips_unknown(db_path: Path) -> None:
    ble = MockBLEManager()
    rt, _ = _runtime(db_path, ble)
    rt.activate()
    feed = rt.make_feed()
    data = feed(ADDR, DISP_UUID)
    assert data is not None
    assert 0 <= parser.decode(data, "uint8") <= 255
    assert feed("ZZ:ZZ", "nope") is None
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_ble_runtime.py -v`
Expected: FAIL（`home_server.services.ble_runtime` 不存在）。

- [ ] **Step 3: 實作 `BleRuntime`**

Create `src/home_server/services/ble_runtime.py`：

```python
"""Wire BLE notify subscriptions to the channel service and feed the mock.

Constructed (inert) by the application factory and stored in app.extensions.
`activate()` connects known devices and subscribes their display channels —
it has side effects (connections, callbacks) and is invoked only from
`__main__`, never at app-construction time, so tests stay thread-free.
"""

from __future__ import annotations

import logging
import math
import sqlite3
from collections.abc import Callable

from home_server.ble import parser
from home_server.ble.interface import BLEManager
from home_server.db import channels, devices
from home_server.db.channels import Channel
from home_server.services.channel_service import ChannelService

log = logging.getLogger(__name__)


class BleRuntime:
    def __init__(
        self,
        ble: BLEManager,
        channel_service: ChannelService,
        *,
        conn_factory: Callable[[], sqlite3.Connection],
        scan_duration: float,
    ) -> None:
        self._ble = ble
        self._channel_service = channel_service
        self._conn_factory = conn_factory
        self._scan_duration = scan_duration
        # (address, char_uuid) -> Channel, so make_feed() knows each data_format.
        self._subscribed: dict[tuple[str, str], Channel] = {}

    def activate(self) -> None:
        """Connect every known device and subscribe its display channels."""
        conn = self._conn_factory()
        try:
            for device in devices.list_all(conn):
                try:
                    self._ble.connect(device.address)
                except Exception:
                    log.warning(
                        "connect to %s failed; skipping", device.address, exc_info=True
                    )
                    continue
                for channel in channels.list_by_device(conn, device.id):
                    if channel.type == "display":
                        self.subscribe_channel(device.address, channel)
        finally:
            conn.close()

    def subscribe_channel(self, address: str, channel: Channel) -> None:
        """Subscribe one display channel; each notify opens a short-lived conn."""
        channel_id = channel.id

        def _on_notify(raw: bytes) -> None:
            conn = self._conn_factory()
            try:
                self._channel_service.handle_notify(
                    conn, channel_id=channel_id, raw_bytes=raw
                )
            finally:
                conn.close()

        self._ble.subscribe(address, channel.char_uuid, _on_notify)
        self._subscribed[(address, channel.char_uuid)] = channel

    def make_feed(self) -> Callable[[str, str], bytes | None]:
        """Return a format-aware synthetic reading generator for the mock."""
        state = {"n": 0}

        def feed(address: str, char_uuid: str) -> bytes | None:
            channel = self._subscribed.get((address, char_uuid))
            if channel is None:
                return None
            state["n"] += 1
            value = 25.0 + 5.0 * math.sin(state["n"] / 10.0)
            try:
                return parser.encode(value, channel.data_format)
            except parser.ParseError:
                return None

        return feed
```

- [ ] **Step 4: 新增 `get_ble_runtime` 存取器**

在 `src/home_server/web/services.py`：

匯入區加入 `BleRuntime`：

```python
from home_server.services.ble_runtime import BleRuntime
from home_server.services.channel_service import ChannelService
from home_server.services.device_service import DeviceService

DEVICE_SERVICE_KEY = "home_device_service"
CHANNEL_SERVICE_KEY = "home_channel_service"
BLE_RUNTIME_KEY = "home_ble_runtime"
```

檔案末端追加存取器：

```python
def get_ble_runtime() -> BleRuntime:
    rt = current_app.extensions[BLE_RUNTIME_KEY]
    if not isinstance(rt, BleRuntime):
        raise TypeError(
            f"Expected BleRuntime in extensions[{BLE_RUNTIME_KEY!r}], "
            f"got {type(rt).__name__}"
        )
    return rt
```

- [ ] **Step 5: 跑測試確認通過**

Run: `uv run pytest tests/test_ble_runtime.py -v`
Expected: PASS。

- [ ] **Step 6: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/services/ble_runtime.py src/home_server/web/services.py tests/test_ble_runtime.py
git commit -m "feat(ble): add BleRuntime notify wiring + get_ble_runtime accessor"
```

---

## Task 7: SocketIO 即時推播接線（create_app）+ index 帶資料 + mypy override

**Files:**
- Modify: `src/home_server/web/__init__.py`
- Modify: `pyproject.toml`
- Test: `tests/test_realtime.py`

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_realtime.py`：

```python
"""Tests for SocketIO realtime reading push."""

from pathlib import Path

from flask import Flask

from home_server.db import channels, connection, devices, users
from home_server.web import socketio
from home_server.web.services import get_channel_service

ADDR = "AA:BB:CC:DD:EE:FF"
DISP_UUID = "uuid-disp"


def _seed_display_channel(app: Flask) -> int:
    db_path = app.config["DB_PATH"]
    conn = connection.connect(db_path)
    try:
        uid = users.create(conn, username="u", password_hash="x")
        did = devices.create(conn, address=ADDR, name="d", owner_user_id=uid)
        return channels.create(
            conn, device_id=did, name="temp", type="display",
            char_uuid=DISP_UUID, data_format="uint8", unit=None,
        )
    finally:
        conn.close()


def _notify(app: Flask, channel_id: int, raw: bytes) -> None:
    with app.app_context():
        conn = connection.connect(app.config["DB_PATH"])
        try:
            get_channel_service().handle_notify(
                conn, channel_id=channel_id, raw_bytes=raw
            )
        finally:
            conn.close()


def test_reading_pushed_to_subscribed_client(app: Flask) -> None:
    channel_id = _seed_display_channel(app)
    ws = socketio.test_client(app)
    ws.emit("subscribe_channel", {"channel_id": channel_id})
    _notify(app, channel_id, b"\x2a")
    events = [e for e in ws.get_received() if e["name"] == "reading"]
    assert events, "expected a reading event"
    payload = events[0]["args"][0]
    assert payload["channel_id"] == channel_id
    assert payload["value"] == 42.0


def test_reading_not_pushed_without_subscription(app: Flask) -> None:
    channel_id = _seed_display_channel(app)
    ws = socketio.test_client(app)  # never subscribes to the room
    _notify(app, channel_id, b"\x2a")
    assert [e for e in ws.get_received() if e["name"] == "reading"] == []
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_realtime.py -v`
Expected: FAIL（`cannot import name 'socketio' from home_server.web`）。

- [ ] **Step 3: 改寫 `web/__init__.py`**

把 `src/home_server/web/__init__.py` 全檔替換為：

```python
"""Flask application factory."""

from __future__ import annotations

from typing import Any

from flask import Flask, g, render_template
from flask_login import LoginManager, login_required
from flask_socketio import SocketIO, join_room, leave_room
from flask_wtf import CSRFProtect

from home_server.ble.interface import BLEManager
from home_server.ble.mock_manager import MockBLEManager
from home_server.ble.rate_limiter import RateLimiter
from home_server.config import Config
from home_server.db import channels, connection, devices, users
from home_server.services.ble_runtime import BleRuntime
from home_server.services.channel_service import ChannelService
from home_server.services.device_service import DeviceService
from home_server.web.db import get_conn
from home_server.web.services import (
    BLE_RUNTIME_KEY,
    CHANNEL_SERVICE_KEY,
    DEVICE_SERVICE_KEY,
)

socketio = SocketIO()


def _emit_reading(channel_id: int, value: float, timestamp: str) -> None:
    """UI push: send each notify to the per-channel SocketIO room."""
    socketio.emit(
        "reading",
        {"channel_id": channel_id, "value": value, "timestamp": timestamp},
        room=f"channel:{channel_id}",
    )


@socketio.on("subscribe_channel")
def _on_subscribe_channel(data: dict[str, Any]) -> None:
    channel_id = data.get("channel_id")
    if channel_id is not None:
        join_room(f"channel:{channel_id}")


@socketio.on("unsubscribe_channel")
def _on_unsubscribe_channel(data: dict[str, Any]) -> None:
    channel_id = data.get("channel_id")
    if channel_id is not None:
        leave_room(f"channel:{channel_id}")


def create_app(config: Config, ble: BLEManager | None = None) -> Flask:
    app = Flask(__name__)
    app.config["SECRET_KEY"] = config.secret_key
    app.config["DB_PATH"] = config.db_path
    app.config["BLE_SCAN_DURATION"] = config.ble_scan_duration

    socketio.init_app(app, async_mode="threading")

    if ble is None:
        ble = MockBLEManager()
    limiter = RateLimiter(config.reading_min_interval)
    channel_service = ChannelService(ble, limiter, _emit_reading)
    app.extensions[DEVICE_SERVICE_KEY] = DeviceService(ble)
    app.extensions[CHANNEL_SERVICE_KEY] = channel_service
    app.extensions[BLE_RUNTIME_KEY] = BleRuntime(
        ble,
        channel_service,
        conn_factory=lambda: connection.connect(config.db_path),
        scan_duration=config.ble_scan_duration,
    )

    login_manager = LoginManager()
    login_manager.init_app(app)
    login_manager.login_view = "auth.login"
    CSRFProtect(app)

    from home_server.web.auth import LoginUser
    from home_server.web.auth import bp as auth_bp

    @login_manager.user_loader
    def load_user(user_id: str) -> LoginUser | None:
        user = users.get_by_id(get_conn(), int(user_id))
        return LoginUser(user) if user is not None else None

    app.register_blueprint(auth_bp)

    from home_server.web.devices import bp as devices_bp

    app.register_blueprint(devices_bp)

    from home_server.web.channels import bp as channels_bp

    app.register_blueprint(channels_bp)

    @app.get("/")
    @login_required
    def index() -> str:
        conn = get_conn()
        overview = [
            (d, channels.list_by_device(conn, d.id)) for d in devices.list_all(conn)
        ]
        return render_template("index.html", overview=overview)

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

- [ ] **Step 4: 在 pyproject 加 mypy override**

在 `pyproject.toml` 把第一條 mypy overrides 的 module 清單加入 `flask_socketio.*`：

```toml
[[tool.mypy.overrides]]
module = ["flask_login.*", "flask_wtf.*", "wtforms.*", "bluepy.*", "flask_socketio.*"]
ignore_missing_imports = true
```

- [ ] **Step 5: 跑測試確認通過（含既有全套，確認 index 改動無回歸）**

Run: `uv run pytest tests/test_realtime.py -v && uv run pytest`
Expected: PASS（test_realtime 2 個綠；既有測試仍綠）。

- [ ] **Step 6: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/__init__.py pyproject.toml tests/test_realtime.py
git commit -m "feat(web): wire SocketIO realtime reading push + dashboard data"
```

---

## Task 8: Vendored 前端資產 + base.html 載入

**Files:**
- Create: `src/home_server/web/static/vendor/chartjs/chart.umd.min.js`
- Create: `src/home_server/web/static/vendor/socketio/socket.io.min.js`
- Create: `src/home_server/web/static/vendor/htmx/htmx.min.js`
- Modify: `src/home_server/web/templates/base.html`
- Test: `tests/test_frontend.py`

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_frontend.py`：

```python
"""Rendering checks for the realtime/control frontend."""

from flask import Flask
from flask.testing import FlaskClient

from home_server.db import channels, connection, devices


def _make_device(app: Flask, address: str, name: str) -> int:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        return devices.create(conn, address=address, name=name, owner_user_id=1)
    finally:
        conn.close()


def _add_channel(
    app: Flask, device_id: int, *, name: str, type_: str, char_uuid: str
) -> int:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        return channels.create(
            conn, device_id=device_id, name=name, type=type_,
            char_uuid=char_uuid, data_format="uint8", unit=None,
        )
    finally:
        conn.close()


def test_base_includes_frontend_assets(logged_in_client: FlaskClient) -> None:
    body = logged_in_client.get("/").get_data(as_text=True)
    assert "vendor/chartjs/chart.umd.min.js" in body
    assert "vendor/socketio/socket.io.min.js" in body
    assert "vendor/htmx/htmx.min.js" in body
    assert "js/dashboard.js" in body
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_frontend.py::test_base_includes_frontend_assets -v`
Expected: FAIL（找不到 asset 字串）。

- [ ] **Step 3: 下載 vendored 資產（固定版本）**

Run:

```bash
mkdir -p src/home_server/web/static/vendor/chartjs \
         src/home_server/web/static/vendor/socketio \
         src/home_server/web/static/vendor/htmx
curl -fsSL https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js \
  -o src/home_server/web/static/vendor/chartjs/chart.umd.min.js
curl -fsSL https://cdn.jsdelivr.net/npm/socket.io-client@4.7.5/dist/socket.io.min.js \
  -o src/home_server/web/static/vendor/socketio/socket.io.min.js
curl -fsSL https://cdn.jsdelivr.net/npm/htmx.org@1.9.12/dist/htmx.min.js \
  -o src/home_server/web/static/vendor/htmx/htmx.min.js
ls -l src/home_server/web/static/vendor/chartjs src/home_server/web/static/vendor/socketio src/home_server/web/static/vendor/htmx
```

Expected: 三個檔案各 >10 KB（chart.umd.min.js ~200 KB、socket.io.min.js ~40 KB、htmx.min.js ~45 KB）。若 curl 失敗（離線），改由可上網的機器下載後放入相同路徑。

> 相容性：Flask-SocketIO 5.x 對應 socket.io client 4.x（本處 4.7.5）。

- [ ] **Step 4: 修改 base.html 載入腳本**

在 `src/home_server/web/templates/base.html` 把既有 bootstrap bundle script 行替換為下列四行 + 原本那行（即在其後新增三個 vendor 腳本與 dashboard.js）：

把：

```html
  <script src="{{ url_for('static', filename='vendor/bootstrap/bootstrap.bundle.min.js') }}"></script>
</body>
```

改為：

```html
  <script src="{{ url_for('static', filename='vendor/bootstrap/bootstrap.bundle.min.js') }}"></script>
  <script src="{{ url_for('static', filename='vendor/chartjs/chart.umd.min.js') }}"></script>
  <script src="{{ url_for('static', filename='vendor/socketio/socket.io.min.js') }}"></script>
  <script src="{{ url_for('static', filename='vendor/htmx/htmx.min.js') }}"></script>
  <script src="{{ url_for('static', filename='js/dashboard.js') }}"></script>
</body>
```

- [ ] **Step 5: 跑測試確認通過**

Run: `uv run pytest tests/test_frontend.py -v`
Expected: PASS。

- [ ] **Step 6: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/static/vendor src/home_server/web/templates/base.html tests/test_frontend.py
git commit -m "feat(web): vendor chart.js/socket.io/htmx and load in base layout"
```

> 註：`dashboard.js` 於 Task 9 建立；本 task 後該 `<script>` 暫時 404 但不影響頁面渲染與此測試。

---

## Task 9: 前端模板與 dashboard.js（圖表 / 控制 / 掃描 / Dashboard）

**Files:**
- Create: `src/home_server/web/static/js/dashboard.js`
- Modify: `src/home_server/web/templates/devices/detail.html`
- Modify: `src/home_server/web/templates/devices/list.html`
- Modify: `src/home_server/web/templates/index.html`
- Test: `tests/test_frontend.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_frontend.py` 末端追加：

```python
def test_detail_shows_chart_for_display_channel(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:41", "Sensor")
    channel_id = _add_channel(
        app, device_id, name="Temp", type_="display", char_uuid="uuid-t"
    )
    body = logged_in_client.get(f"/devices/{device_id}").get_data(as_text=True)
    assert f'data-channel-id="{channel_id}"' in body


def test_detail_shows_control_form_for_controller(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:42", "Lamp")
    channel_id = _add_channel(
        app, device_id, name="LED", type_="controller", char_uuid="uuid-led"
    )
    body = logged_in_client.get(f"/devices/{device_id}").get_data(as_text=True)
    assert f"/channels/{channel_id}/write" in body
    assert 'name="value"' in body


def test_list_has_scan_button(logged_in_client: FlaskClient) -> None:
    body = logged_in_client.get("/devices").get_data(as_text=True)
    assert 'hx-get="/devices/scan"' in body
    assert 'id="scan-results"' in body


def test_index_dashboard_lists_channels(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:43", "Sensor2")
    channel_id = _add_channel(
        app, device_id, name="Humidity", type_="display", char_uuid="uuid-h"
    )
    body = logged_in_client.get("/").get_data(as_text=True)
    assert "Humidity" in body
    assert f'data-channel-id="{channel_id}"' in body
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_frontend.py -v`
Expected: FAIL（detail/list/index 尚無對應標記）。

- [ ] **Step 3: 建立 dashboard.js**

Create `src/home_server/web/static/js/dashboard.js`：

```javascript
(function () {
  function initCharts() {
    var canvases = document.querySelectorAll("canvas[data-channel-id]");
    if (canvases.length === 0) return;
    var sock = io();
    var charts = {};

    canvases.forEach(function (canvas) {
      var channelId = parseInt(canvas.getAttribute("data-channel-id"), 10);
      var unit = canvas.getAttribute("data-unit") || "value";
      var chart = new Chart(canvas, {
        type: "line",
        data: { labels: [], datasets: [{ label: unit, data: [], tension: 0.3 }] },
        options: { animation: false, scales: { x: { display: false } } },
      });
      charts[channelId] = chart;

      fetch("/channels/" + channelId + "/history")
        .then(function (r) { return r.json(); })
        .then(function (j) {
          j.readings.forEach(function (pt) {
            chart.data.labels.push(pt.recorded_at);
            chart.data.datasets[0].data.push(pt.value);
          });
          chart.update();
        });

      sock.emit("subscribe_channel", { channel_id: channelId });
    });

    sock.on("reading", function (d) {
      var chart = charts[d.channel_id];
      if (!chart) return;
      chart.data.labels.push(d.timestamp);
      chart.data.datasets[0].data.push(d.value);
      if (chart.data.labels.length > 60) {
        chart.data.labels.shift();
        chart.data.datasets[0].data.shift();
      }
      chart.update();
    });
  }

  if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", initCharts);
  } else {
    initCharts();
  }
})();
```

- [ ] **Step 4: 改寫 detail.html（在頻道表格與「Add channel」卡片之間插入 Live & control 卡片）**

把 `src/home_server/web/templates/devices/detail.html` 全檔替換為：

```html
{% extends "base.html" %}
{% block title %}{{ device.name }}{% endblock %}
{% block content %}
<p><a href="{{ url_for('devices.list_devices') }}">&larr; Devices</a></p>

<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h1 class="card-title h4">{{ device.name }}</h1>
    <p class="card-text mb-0"><code>{{ device.address }}</code></p>
  </div>
</div>

<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Channels</h2>
    {% if channels %}
    <table class="table table-sm align-middle">
      <thead>
        <tr><th>Name</th><th>Type</th><th>UUID</th><th>Format</th><th>Unit</th><th></th></tr>
      </thead>
      <tbody>
        {% for channel in channels %}
        <tr>
          <td>{{ channel.name }}</td>
          <td>{{ channel.type }}</td>
          <td><code>{{ channel.char_uuid }}</code></td>
          <td>{{ channel.data_format }}</td>
          <td>{{ channel.unit or "" }}</td>
          <td class="text-end">
            <form method="post"
                  action="{{ url_for('channels.delete_channel', channel_id=channel.id) }}"
                  onsubmit="return confirm('Delete this channel?');">
              <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
              <button class="btn btn-outline-danger btn-sm" type="submit">Delete</button>
            </form>
          </td>
        </tr>
        {% endfor %}
      </tbody>
    </table>
    {% else %}
    <p class="text-muted mb-0">No channels yet.</p>
    {% endif %}
  </div>
</div>

{% if channels %}
<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Live &amp; control</h2>
    {% for channel in channels %}
    <div class="mb-4">
      <h3 class="h6">{{ channel.name }} <small class="text-muted">({{ channel.type }})</small></h3>
      {% if channel.type == "display" %}
      <canvas data-channel-id="{{ channel.id }}" data-unit="{{ channel.unit or '' }}" height="80"></canvas>
      {% else %}
      <form method="post"
            action="{{ url_for('channels.write_channel', channel_id=channel.id) }}"
            class="row g-2">
        <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
        <div class="col-auto">
          <input class="form-control form-control-sm" name="value" placeholder="value" required>
        </div>
        <div class="col-auto">
          <button class="btn btn-sm btn-primary" type="submit">Send</button>
        </div>
      </form>
      {% endif %}
    </div>
    {% endfor %}
  </div>
</div>
{% endif %}

<div class="card shadow-sm">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Add channel</h2>
    <form method="post"
          action="{{ url_for('channels.add_channel', device_id=device.id) }}"
          class="row g-2">
      <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
      <div class="col-sm-3">
        <input class="form-control" name="name" placeholder="Temperature" required>
      </div>
      <div class="col-sm-2">
        <select class="form-select" name="type">
          <option value="display">display</option>
          <option value="controller">controller</option>
        </select>
      </div>
      <div class="col-sm-3">
        <input class="form-control" name="char_uuid" placeholder="Characteristic UUID" required>
      </div>
      <div class="col-sm-2">
        <select class="form-select" name="data_format">
          {% for fmt in formats %}
          <option value="{{ fmt }}">{{ fmt }}</option>
          {% endfor %}
        </select>
      </div>
      <div class="col-sm-1">
        <input class="form-control" name="unit" placeholder="Unit">
      </div>
      <div class="col-sm-1 d-grid">
        <button class="btn btn-primary" type="submit">Add</button>
      </div>
    </form>
  </div>
</div>
{% endblock %}
```

- [ ] **Step 5: 在 list.html 插入 Scan 卡片**

在 `src/home_server/web/templates/devices/list.html`，把：

```html
<div class="card shadow-sm">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Add device</h2>
```

替換為（在 Add device 卡片前插入 Scan 卡片）：

```html
<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Scan for devices</h2>
    <button class="btn btn-outline-primary mb-3"
            hx-get="/devices/scan"
            hx-target="#scan-results"
            hx-swap="innerHTML">Scan</button>
    <div id="scan-results"></div>
  </div>
</div>

<div class="card shadow-sm">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Add device</h2>
```

> 測試斷言 `hx-get="/devices/scan"` 字面字串，故此處直接寫死路徑（與 `url_for('devices.scan')` 結果相同）。

- [ ] **Step 6: 改寫 index.html 為 Dashboard**

把 `src/home_server/web/templates/index.html` 全檔替換為：

```html
{% extends "base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<h1 class="h4 mb-3">Dashboard</h1>

{% if overview %}
  {% for device, device_channels in overview %}
  <div class="card shadow-sm mb-4">
    <div class="card-body">
      <h2 class="card-title h5">
        <a href="{{ url_for('devices.detail', device_id=device.id) }}">{{ device.name }}</a>
        <small class="text-muted"><code>{{ device.address }}</code></small>
      </h2>
      {% if device_channels %}
        {% for channel in device_channels %}
        <div class="mb-3">
          <h3 class="h6">{{ channel.name }} <small class="text-muted">({{ channel.type }})</small></h3>
          {% if channel.type == "display" %}
          <canvas data-channel-id="{{ channel.id }}" data-unit="{{ channel.unit or '' }}" height="70"></canvas>
          {% else %}
          <form method="post"
                action="{{ url_for('channels.write_channel', channel_id=channel.id) }}"
                class="row g-2">
            <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
            <div class="col-auto">
              <input class="form-control form-control-sm" name="value" placeholder="value" required>
            </div>
            <div class="col-auto">
              <button class="btn btn-sm btn-primary" type="submit">Send</button>
            </div>
          </form>
          {% endif %}
        </div>
        {% endfor %}
      {% else %}
        <p class="text-muted mb-0">No channels.</p>
      {% endif %}
    </div>
  </div>
  {% endfor %}
{% else %}
  <p class="text-muted">No devices yet. <a href="{{ url_for('devices.list_devices') }}">Add one</a>.</p>
{% endif %}
{% endblock %}
```

- [ ] **Step 7: 跑測試確認通過**

Run: `uv run pytest tests/test_frontend.py -v`
Expected: PASS（5 個前端測試全綠）。

- [ ] **Step 8: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/static/js/dashboard.js src/home_server/web/templates/devices/detail.html src/home_server/web/templates/devices/list.html src/home_server/web/templates/index.html tests/test_frontend.py
git commit -m "feat(web): live charts, control forms, scan UI, dashboard"
```

---

## Task 10: `__main__` 啟動接線（socketio.run + activate + mock 產數）+ 手動冒煙驗證

**Files:**
- Modify: `src/home_server/__main__.py`

此 task 接上即時鏈路的啟動端，無法用單元測試覆蓋（牽涉真的伺服器與背景執行緒），改以手動冒煙驗證。

- [ ] **Step 1: 改寫 `__main__.py` 的 main()**

把 `src/home_server/__main__.py` 的匯入區與 `main()` 改為：

匯入區（檔案頂部）改為：

```python
"""Entry point: `python -m home_server`."""

from __future__ import annotations

import logging

from home_server.ble.interface import DiscoveredDevice
from home_server.ble.mock_manager import MockBLEManager
from home_server.config import Config
from home_server.core.logging import setup_logging
from home_server.db import connection
from home_server.services import user_service
from home_server.web import create_app, socketio
from home_server.web.services import get_ble_runtime
```

`main()` 改為：

```python
def main() -> None:
    config = Config.from_env()
    setup_logging(config.log_level)
    log = logging.getLogger(__name__)

    log.info("Initializing database at %s", config.db_path)
    connection.initialize(config.db_path)
    _seed_admin(config, log)

    ble = MockBLEManager()
    # Dev demo: a couple of discoverable devices so the Scan button shows output.
    ble.scan_results = [
        DiscoveredDevice(address="C0:FF:EE:00:00:01", name="Demo Sensor", rssi=-55),
        DiscoveredDevice(address="C0:FF:EE:00:00:02", name="Demo Lamp", rssi=-61),
    ]
    app = create_app(config, ble=ble)

    with app.app_context():
        runtime = get_ble_runtime()
        runtime.activate()
        if isinstance(ble, MockBLEManager):
            ble.start(runtime.make_feed(), interval_s=1.0)

    log.info(
        "Starting server on %s:%d (debug=%s)", config.host, config.port, config.debug
    )
    socketio.run(
        app,
        host=config.host,
        port=config.port,
        debug=config.debug,
        use_reloader=False,
        allow_unsafe_werkzeug=True,
    )
```

> `allow_unsafe_werkzeug=True`：Flask-SocketIO 在 Werkzeug dev server 上需此旗標才允許 `socketio.run`（本專案以 Werkzeug 跑於 RPi/開發機，非高負載生產）。

- [ ] **Step 2: 靜態驗證**

Run: `uv run ruff check && uv run mypy src && uv run pytest`
Expected: PASS（全套仍綠）。

- [ ] **Step 3: 冒煙驗證 — 啟動伺服器並打 /health**

Run（背景啟動，5001 避開 macOS AirPlay 佔用的 5000）：

```bash
HOME_SERVER_DEBUG=1 HOME_SERVER_PORT=5001 HOME_SERVER_DB_PATH=./data/dev.db \
  HOME_SERVER_ADMIN_USERNAME=admin HOME_SERVER_ADMIN_PASSWORD=admin123 \
  uv run python -m home_server &
sleep 3
curl -s http://127.0.0.1:5001/health
```

Expected: `{"status":"ok"}`。

- [ ] **Step 4: 手動瀏覽器驗證（人工）**

在瀏覽器開 `http://127.0.0.1:5001`：
1. 以 `admin` / `admin123` 登入。
2. 進 Devices → 按「Scan」→ 應看到 Demo Sensor / Demo Lamp → 按「Add」加入其一。
3. 進該裝置詳情 → 新增一個 `display` 頻道（char_uuid 任意、data_format `uint8`）。
4. 回詳情頁，Live & control 區的圖應**每秒自動長出新點**（mock 背景產數 + SocketIO 推播）。
5. 新增一個 `controller` 頻道 → 在其控制表單輸入 0–255 數值送出 → 不報錯（mock 記錄寫入）。
6. 回首頁 Dashboard → 應看到所有頻道的迷你即時圖 / 控制表單。

驗證後關閉背景伺服器：

```bash
kill %1 2>/dev/null || pkill -f "python -m home_server"
rm -f ./data/dev.db ./data/dev.db-wal ./data/dev.db-shm
```

- [ ] **Step 5: Commit**

```bash
git add src/home_server/__main__.py
git commit -m "feat: serve with SocketIO, activate BLE runtime, mock data feed"
```

---

## Task 11: 更新 RPi-Server 開發文件進度

**Files:**
- Modify: `docs/RPi-Server 開發文件.md`

- [ ] **Step 1: 量測最終測試數**

Run: `uv run pytest -q | tail -3`
Expected: 顯示新的 `N passed`（記下 N，取代下方的 `<N>`）。

- [ ] **Step 2: 更新 §11.1 進度表與測試現況**

在 `docs/RPi-Server 開發文件.md` §11.1：

把 3e 那列拆成 3e-1（完成）與 3e-2（未開始）：

```markdown
| 3e-1 | SocketIO 即時推播接線、`/devices/scan`・`/channels/<id>/write`・`/channels/<id>/history`、Mock 背景產數、HTMX/Chart.js 前端 | ✅ 完成 |
| 3e-2 | 真實 `BluepyManager` 依平台選用、斷線自動重連、`device_status` 事件（需 RPi 硬體） | ⬜ 未開始 |
| 4 | 多節點整合測試、systemd 部署 | ⬜ 未開始 |
```

把「測試現況」行的數字改為實際值：

```markdown
測試現況：<N> unit tests passing、`ruff check` 與 `mypy src`（strict）全綠。
```

在 §11.1 取捨備註區新增一條：

```markdown
> 3e-1 的設計取捨：SocketIO 採模組級單例 + `init_app`，`create_app` 維持回傳 `Flask`（不動既有測試）；notify→DB→emit 主流程在 3b 已實作，3e-1 以 `services/ble_runtime.py` 接線（連線、訂閱 display 頻道、worker 執行緒短連線寫入）。副作用（背景執行緒、連線）只在 `__main__` 觸發，測試不起執行緒。控制寫入沿用既有 `write_command` 依頻道 `data_format` 編碼（非單一 byte）。前端資產與 Bootstrap 一致採 vendoring。真實 bluepy 與自動重連留待 3e-2 於 RPi 驗證。
```

- [ ] **Step 3: Commit**

```bash
git add "docs/RPi-Server 開發文件.md"
git commit -m "docs: mark RPi-server phase 3e-1 complete; split 3e into 3e-1/3e-2"
```

---

## Self-Review（規劃者已執行）

**1. Spec coverage：**
- SocketIO 即時推播 → Task 7（`_emit_reading` + 事件處理）✓
- Notify 訂閱 wiring → Task 6（`BleRuntime`）✓
- Mock 背景產數 → Task 5 ✓
- `/devices/scan` → Task 4；`/channels/<id>/write` → Task 2；`/channels/<id>/history` → Task 3 ✓
- 前端 vendoring + 圖/控制/掃描/Dashboard → Task 8、9 ✓
- `__main__` socketio.run + activate + mock.start → Task 10 ✓
- 測試（web/runtime/realtime/mock/frontend）→ 各 Task ✓
- mypy override（flask_socketio）→ Task 7 ✓
- 文件進度 → Task 11 ✓
- 範圍外（真實 bluepy、自動重連、device_status）→ 明列 3e-2，未排入本計畫 ✓

**2. Placeholder scan：** 無 TBD/TODO；每個程式步驟皆含完整程式碼。

**3. Type/名稱一致性：**
- 端點函式名 `write_channel` / `channel_history` / `scan` 與模板 `url_for('channels.write_channel')`、測試斷言一致 ✓
- `BleRuntime(ble, channel_service, *, conn_factory, scan_duration)` 在 Task 6 定義、Task 7 create_app 與測試以相同簽章呼叫 ✓
- `get_ble_runtime()`（無參數，靠 `current_app`）於 Task 6 定義、Task 10 在 `app_context` 內呼叫 ✓
- `MockBLEManager.start(feed, interval_s)` / `stop()` / `_tick(feed)` 在 Task 5 定義、Task 10 使用 ✓
- `ChannelService.handle_notify(conn, *, channel_id, raw_bytes)`、`write_command(conn, *, channel_id, raw_value)`、`get_history(conn, channel_id, *, limit=...)` 皆與既有實作簽章一致 ✓
- `_emit_reading(channel_id, value, timestamp)` 與 `ReadingCallback = Callable[[int, float, str], None]` 一致 ✓
- SocketIO event `reading` payload `{channel_id, value, timestamp}` 與 dashboard.js 取用欄位一致 ✓

---

*計畫結束。*
