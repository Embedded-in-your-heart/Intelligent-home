# Phase 3e-2 — BLE backend 平台選用 + 自動重連 + device_status Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 依平台選用 `BluepyManager`/`MockBLEManager`、為 `BleRuntime` 加上斷線自動重連（指數退避）與 `device_status` 即時推播 + 前端徽章；全程以 mock 在 macOS 單元測試，真 STM32 的 bluepy 冷煙留為 RPi 手動步驟。

**Architecture:** 新增純函式 `select_ble_manager(backend, platform)`（bluepy 延遲匯入）；擴充既有 `BleRuntime`（已擁有連線/訂閱生命週期）加入輪詢式重連監看（可同步測試的 `_monitor_tick(now)` + 背景 `monitor_start/stop`）與 `on_status` 回呼；`create_app` 把 `on_status` 接到 `socketio.emit("device_status", …)`，前端徽章由 `dashboard.js` 即時更新。副作用（執行緒、連線）只在 `__main__` 觸發。

**Tech Stack:** Python 3.11 / Flask / Flask-SocketIO（threading）/ SQLite / Chart.js+htmx+socket.io（vendored）/ pytest / ruff / mypy(strict)。

> 對應規格：`docs/superpowers/specs/2026-05-31-phase-3e-2-platform-reconnect-design.md`
> 工作目錄：`Intelligent-home-RPi-server/`（以下相對路徑皆以此為根；指令在此目錄執行）。
> 每個 commit 前驗證：`uv run ruff check && uv run mypy src && uv run pytest`
> 重要：mypy 只檢查 `src`（不檢查 `tests`）；ruff 檢查全部。`dashboard.js` 為純 JS，不納入 ruff/mypy。起點：`main` 分支，148→（本階段結束）約 162 passed。

---

## Task 1: 平台選用 — `select_ble_manager` + `Config.ble_backend`

**Files:**
- Create: `src/home_server/ble/selection.py`
- Modify: `src/home_server/config.py`
- Test: `tests/test_ble_selection.py`

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_ble_selection.py`：

```python
import sys

import pytest

from home_server.ble.mock_manager import MockBLEManager
from home_server.ble.selection import select_ble_manager
from home_server.config import Config

_ON_LINUX = sys.platform.startswith("linux")


def test_mock_backend_returns_mock() -> None:
    assert isinstance(select_ble_manager("mock", "linux"), MockBLEManager)


def test_auto_non_linux_returns_mock() -> None:
    assert isinstance(select_ble_manager("auto", "darwin"), MockBLEManager)


def test_unknown_backend_raises() -> None:
    with pytest.raises(ValueError):
        select_ble_manager("nonsense", "linux")


@pytest.mark.skipif(_ON_LINUX, reason="bluepy may import on Linux; fallback only on non-Linux")
def test_auto_linux_falls_back_to_mock_when_bluepy_unavailable() -> None:
    assert isinstance(select_ble_manager("auto", "linux"), MockBLEManager)


@pytest.mark.skipif(_ON_LINUX, reason="bluepy import only fails on non-Linux")
def test_bluepy_backend_raises_on_non_linux() -> None:
    with pytest.raises(ImportError):
        select_ble_manager("bluepy", "linux")


def test_config_ble_backend_default_and_override(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("HOME_SERVER_DEBUG", "1")
    monkeypatch.delenv("HOME_SERVER_BLE_BACKEND", raising=False)
    assert Config.from_env().ble_backend == "auto"
    monkeypatch.setenv("HOME_SERVER_BLE_BACKEND", "mock")
    assert Config.from_env().ble_backend == "mock"
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_ble_selection.py -v`
Expected: FAIL（`home_server.ble.selection` 不存在；`Config` 無 `ble_backend`）。

- [ ] **Step 3: 建立 `selection.py`**

Create `src/home_server/ble/selection.py`：

```python
"""Select the BLE backend at startup.

bluepy only imports on Linux (the module raises ImportError elsewhere), so the
import is deferred into _load_bluepy() to keep this module importable anywhere.
"""

from __future__ import annotations

import logging

from home_server.ble.interface import BLEManager
from home_server.ble.mock_manager import MockBLEManager

log = logging.getLogger(__name__)


def select_ble_manager(backend: str, platform: str) -> BLEManager:
    if backend == "mock":
        return MockBLEManager()
    if backend == "bluepy":
        return _load_bluepy()
    if backend == "auto":
        if platform.startswith("linux"):
            try:
                return _load_bluepy()
            except ImportError:
                log.warning("bluepy unavailable; falling back to MockBLEManager")
                return MockBLEManager()
        return MockBLEManager()
    raise ValueError(f"unknown BLE backend: {backend!r}")


def _load_bluepy() -> BLEManager:
    from home_server.ble.bluepy_manager import BluepyManager

    return BluepyManager()
```

- [ ] **Step 4: 加 `Config.ble_backend`**

In `src/home_server/config.py`:

加欄位（放在 `debug: bool` 之後，作為最後一個欄位；給預設值以免既有 `Config(...)` 呼叫端需改）：

```python
    admin_password: str | None
    debug: bool
    ble_backend: str = "auto"
```

在 `from_env` 的 `return cls(...)` 內、`debug=debug,` 之後新增一行：

```python
            debug=debug,
            ble_backend=_env_str("HOME_SERVER_BLE_BACKEND", "auto"),
        )
```

- [ ] **Step 5: 跑測試確認通過**

Run: `uv run pytest tests/test_ble_selection.py -v`
Expected: PASS（在 macOS 上，兩個 skipif 測試會實際執行並通過）。

- [ ] **Step 6: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/ble/selection.py src/home_server/config.py tests/test_ble_selection.py
git commit -m "feat(ble): add platform-based backend selection + HOME_SERVER_BLE_BACKEND"
```

---

## Task 2: `DeviceService.is_connected`

**Files:**
- Modify: `src/home_server/services/device_service.py`
- Test: `tests/test_device_service.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_device_service.py` 末端追加（檔案已 import `MockBLEManager`、`DeviceService`、定義 `ADDR`）：

```python
def test_is_connected_reflects_ble() -> None:
    mock = MockBLEManager()
    svc = DeviceService(mock)
    assert svc.is_connected(ADDR) is False
    mock.connect(ADDR)
    assert svc.is_connected(ADDR) is True
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_device_service.py::test_is_connected_reflects_ble -v`
Expected: FAIL（`DeviceService` 無 `is_connected`）。

- [ ] **Step 3: 實作 `is_connected`**

In `src/home_server/services/device_service.py`，在 `list_devices` 方法之後追加：

```python
    def is_connected(self, address: str) -> bool:
        return self._ble.is_connected(address)
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_device_service.py -v`
Expected: PASS。

- [ ] **Step 5: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/services/device_service.py tests/test_device_service.py
git commit -m "feat(services): add DeviceService.is_connected"
```

---

## Task 3: `BleRuntime` 自動重連監看（輪詢 + 退避 + 狀態回呼）

**Files:**
- Modify (full rewrite): `src/home_server/services/ble_runtime.py`
- Test: `tests/test_ble_runtime_reconnect.py`

> 既有 `tests/test_ble_runtime.py` **不需改動**：新增的 `on_status` 參數有預設 no-op，且 `activate()` 重構為呼叫 `_bring_up_device` 後行為不變。

- [ ] **Step 1: 寫失敗測試**

Create `tests/test_ble_runtime_reconnect.py`：

```python
"""Tests for BleRuntime auto-reconnect monitor and device status."""

import time
from pathlib import Path

import pytest

from home_server.ble.mock_manager import MockBLEManager
from home_server.ble.rate_limiter import RateLimiter
from home_server.db import channels, connection, devices, users
from home_server.services.ble_runtime import BleRuntime
from home_server.services.channel_service import ChannelService

ADDR = "AA:BB:CC:DD:EE:FF"
DISP_UUID = "uuid-disp"
CTRL_UUID = "uuid-ctrl"


def _seed(path: Path, *, with_display: bool = True, with_controller: bool = False) -> None:
    connection.initialize(path)
    conn = connection.connect(path)
    try:
        uid = users.create(conn, username="u", password_hash="x")
        did = devices.create(conn, address=ADDR, name="dev", owner_user_id=uid)
        if with_display:
            channels.create(
                conn, device_id=did, name="temp", type="display",
                char_uuid=DISP_UUID, data_format="uint8", unit=None,
            )
        if with_controller:
            channels.create(
                conn, device_id=did, name="led", type="controller",
                char_uuid=CTRL_UUID, data_format="uint8", unit=None,
            )
    finally:
        conn.close()


def _runtime(
    path: Path, ble: MockBLEManager
) -> tuple[BleRuntime, list[tuple[int, str]]]:
    statuses: list[tuple[int, str]] = []
    svc = ChannelService(ble, RateLimiter(0.0), lambda *_: None)
    rt = BleRuntime(
        ble, svc,
        conn_factory=lambda: connection.connect(path),
        scan_duration=1.0,
        on_status=lambda did, st: statuses.append((did, st)),
    )
    return rt, statuses


def test_tick_emits_connected_for_live_device(tmp_path: Path) -> None:
    path = tmp_path / "rt.db"
    _seed(path)
    ble = MockBLEManager()
    rt, statuses = _runtime(path, ble)
    rt.activate()
    rt._monitor_tick(0.0)
    assert statuses[-1][1] == "connected"


def test_disconnect_then_reconnect_cycle(tmp_path: Path) -> None:
    path = tmp_path / "rt.db"
    _seed(path)
    ble = MockBLEManager()
    rt, statuses = _runtime(path, ble)
    rt.activate()
    rt._monitor_tick(0.0)                 # connected
    ble.simulate_disconnect(ADDR)
    rt._monitor_tick(1.0)                 # drop detected -> disconnected, retry at 2.0
    assert statuses[-1][1] == "disconnected"
    rt._monitor_tick(1.5)                 # too early, no change
    assert statuses[-1][1] == "disconnected"
    rt._monitor_tick(2.0)                 # retry -> reconnecting then connected
    seen = [s for _, s in statuses]
    assert "reconnecting" in seen
    assert statuses[-1][1] == "connected"
    ble.simulate_notify(ADDR, DISP_UUID, b"\x2a")  # re-subscribed: no MockBLEError


def test_backoff_doubles_and_caps_at_60(tmp_path: Path) -> None:
    path = tmp_path / "rt.db"
    _seed(path)
    ble = MockBLEManager()
    rt, _ = _runtime(path, ble)
    rt.activate()
    rt._monitor_tick(0.0)                 # connected
    ble.fail_connect_for.add(ADDR)        # reconnects now fail
    ble.simulate_disconnect(ADDR)
    rt._monitor_tick(10.0)                # disconnected, retry scheduled
    for t in range(100, 900, 100):        # each tick = one failed retry (now >> next_retry)
        rt._monitor_tick(float(t))
    assert rt._monitor[ADDR].backoff_s == 60.0


def test_status_emitted_once_per_transition(tmp_path: Path) -> None:
    path = tmp_path / "rt.db"
    _seed(path)
    ble = MockBLEManager()
    rt, statuses = _runtime(path, ble)
    rt.activate()
    rt._monitor_tick(0.0)
    rt._monitor_tick(1.0)
    rt._monitor_tick(2.0)
    assert [s for _, s in statuses].count("connected") == 1


def test_controller_only_device_reconnects(tmp_path: Path) -> None:
    path = tmp_path / "rt.db"
    _seed(path, with_display=False, with_controller=True)
    ble = MockBLEManager()
    rt, statuses = _runtime(path, ble)
    rt.activate()
    rt._monitor_tick(0.0)                 # connected
    ble.simulate_disconnect(ADDR)
    rt._monitor_tick(1.0)                 # disconnected
    rt._monitor_tick(2.0)                 # reconnecting -> connected
    assert ble.is_connected(ADDR)
    assert statuses[-1][1] == "connected"


def test_monitor_start_stop_lifecycle(tmp_path: Path) -> None:
    path = tmp_path / "rt.db"
    _seed(path)
    ble = MockBLEManager()
    rt, _ = _runtime(path, ble)
    rt.activate()
    rt.monitor_start(interval_s=0.01)
    time.sleep(0.05)
    thread = rt._monitor_thread
    rt.monitor_stop()
    assert rt._monitor_thread is None
    assert thread is not None and not thread.is_alive()
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_ble_runtime_reconnect.py -v`
Expected: FAIL（`_monitor_tick` / `monitor_start` / `on_status` 不存在）。

- [ ] **Step 3: 全檔重寫 `ble_runtime.py`**

把 `src/home_server/services/ble_runtime.py` 全檔替換為：

```python
"""Wire BLE notify subscriptions to the channel service, feed the mock, and
keep device connections alive (auto-reconnect with status reporting).

Constructed (inert) by the application factory and stored in app.extensions.
`activate()` connects known devices and subscribes their display channels;
`monitor_start()` runs a background loop that reconnects dropped devices with
exponential backoff and reports status via the injected `on_status` callback.
Side effects (connections, threads) run only from `__main__`; tests drive
`_monitor_tick(now)` directly and stay thread-free.
"""

from __future__ import annotations

import logging
import math
import sqlite3
import threading
import time
from collections.abc import Callable
from dataclasses import dataclass

from home_server.ble import parser
from home_server.ble.interface import BLEManager
from home_server.db import channels, devices
from home_server.db.channels import Channel
from home_server.db.devices import Device
from home_server.services.channel_service import ChannelService

log = logging.getLogger(__name__)

# Reconnect backoff: 1s, 2s, 4s, ... capped at 60s (RPi-Server doc §4.1.3).
_RECONNECT_BASE_S = 1.0
_RECONNECT_FACTOR = 2.0
_RECONNECT_CAP_S = 60.0

STATUS_CONNECTED = "connected"
STATUS_RECONNECTING = "reconnecting"
STATUS_DISCONNECTED = "disconnected"

StatusCallback = Callable[[int, str], None]


def _noop_status(device_id: int, status: str) -> None:
    """Default on_status: no-op (used until create_app wires SocketIO)."""


@dataclass
class _DeviceMonitorState:
    last_status: str | None = None
    backoff_s: float = _RECONNECT_BASE_S
    next_retry_at: float = 0.0


class BleRuntime:
    def __init__(
        self,
        ble: BLEManager,
        channel_service: ChannelService,
        *,
        conn_factory: Callable[[], sqlite3.Connection],
        scan_duration: float,
        on_status: StatusCallback = _noop_status,
    ) -> None:
        self._ble = ble
        self._channel_service = channel_service
        self._conn_factory = conn_factory
        self._scan_duration = scan_duration
        self._on_status = on_status
        # (address, char_uuid) -> Channel, so make_feed() knows each data_format.
        self._subscribed: dict[tuple[str, str], Channel] = {}
        # address -> reconnect bookkeeping.
        self._monitor: dict[str, _DeviceMonitorState] = {}
        self._monitor_thread: threading.Thread | None = None
        self._monitor_stop: threading.Event | None = None

    # ---- Initial bring-up ----

    def activate(self) -> None:
        """Connect every known device and subscribe its display channels."""
        conn = self._conn_factory()
        try:
            for device in devices.list_all(conn):
                self._bring_up_device(conn, device)
        finally:
            conn.close()

    def _bring_up_device(self, conn: sqlite3.Connection, device: Device) -> bool:
        """Connect one device and subscribe its display channels. Returns success."""
        try:
            self._ble.connect(device.address)
        except Exception:
            log.warning("connect to %s failed", device.address, exc_info=True)
            return False
        for channel in channels.list_by_device(conn, device.id):
            if channel.type == "display":
                self.subscribe_channel(device.address, channel)
        return True

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

    # ---- Auto-reconnect monitor ----

    def monitor_start(self, interval_s: float = 1.0) -> None:
        """Run the reconnect monitor in a background daemon thread."""
        if self._monitor_thread is not None:
            return
        stop = threading.Event()

        def _run() -> None:
            while not stop.wait(interval_s):
                try:
                    self._monitor_tick(time.monotonic())
                except Exception:
                    log.exception("BLE monitor tick failed")

        thread = threading.Thread(target=_run, name="ble-monitor", daemon=True)
        self._monitor_stop = stop
        self._monitor_thread = thread
        thread.start()

    def monitor_stop(self) -> None:
        if self._monitor_stop is not None:
            self._monitor_stop.set()
        if self._monitor_thread is not None:
            self._monitor_thread.join(timeout=2.0)
        self._monitor_thread = None
        self._monitor_stop = None

    def _monitor_tick(self, now: float) -> None:
        """One reconnect pass. Pure w.r.t. time (caller passes `now`)."""
        conn = self._conn_factory()
        try:
            device_list = devices.list_all(conn)
            live = {d.address for d in device_list}
            for addr in list(self._monitor):
                if addr not in live:
                    del self._monitor[addr]
            for device in device_list:
                state = self._monitor.setdefault(device.address, _DeviceMonitorState())
                if self._ble.is_connected(device.address):
                    state.backoff_s = _RECONNECT_BASE_S
                    state.next_retry_at = 0.0
                    self._set_status(device.id, state, STATUS_CONNECTED)
                    continue
                if state.last_status in (None, STATUS_CONNECTED):
                    # Just observed a drop: announce it, schedule first retry.
                    state.backoff_s = _RECONNECT_BASE_S
                    state.next_retry_at = now + state.backoff_s
                    self._set_status(device.id, state, STATUS_DISCONNECTED)
                    continue
                if now >= state.next_retry_at:
                    self._set_status(device.id, state, STATUS_RECONNECTING)
                    if self._bring_up_device(conn, device):
                        state.backoff_s = _RECONNECT_BASE_S
                        state.next_retry_at = 0.0
                        self._set_status(device.id, state, STATUS_CONNECTED)
                    else:
                        state.backoff_s = min(
                            state.backoff_s * _RECONNECT_FACTOR, _RECONNECT_CAP_S
                        )
                        state.next_retry_at = now + state.backoff_s
        finally:
            conn.close()

    def _set_status(
        self, device_id: int, state: _DeviceMonitorState, status: str
    ) -> None:
        if state.last_status != status:
            state.last_status = status
            self._on_status(device_id, status)

    # ---- Mock synthetic feed ----

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

- [ ] **Step 4: 跑測試確認通過（含既有 ble_runtime 測試與全套）**

Run: `uv run pytest tests/test_ble_runtime_reconnect.py tests/test_ble_runtime.py -v && uv run pytest`
Expected: PASS（新重連測試全綠；既有 `test_ble_runtime.py` 仍綠；全套綠）。

- [ ] **Step 5: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/services/ble_runtime.py tests/test_ble_runtime_reconnect.py
git commit -m "feat(ble): auto-reconnect monitor with exponential backoff + status callback"
```

---

## Task 4: `device_status` 推播接線（create_app）

**Files:**
- Modify: `src/home_server/web/__init__.py`
- Test: `tests/test_realtime.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_realtime.py`：把第 8 行 import 改為（加入 `get_ble_runtime`）：

```python
from home_server.web.services import get_ble_runtime, get_channel_service
```

在檔案末端追加：

```python
def test_device_status_pushed_to_client(app: Flask, logged_in_client: FlaskClient) -> None:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        uid = users.create(conn, username="u2", password_hash="x")
        did = devices.create(
            conn, address="C0:DE:00:00:00:01", name="d2", owner_user_id=uid
        )
    finally:
        conn.close()
    ws = socketio.test_client(app, flask_test_client=logged_in_client)
    with app.app_context():
        get_ble_runtime()._monitor_tick(0.0)  # device not connected -> "disconnected"
    events = [e for e in ws.get_received() if e["name"] == "device_status"]
    assert events, "expected a device_status event"
    assert events[0]["args"][0] == {"device_id": did, "status": "disconnected"}
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_realtime.py::test_device_status_pushed_to_client -v`
Expected: FAIL（`create_app` 仍用預設 no-op `on_status`，故無 `device_status` 事件）。

- [ ] **Step 3: 接線 `_emit_device_status`**

In `src/home_server/web/__init__.py`：

在 `_emit_reading` 函式之後新增：

```python
def _emit_device_status(device_id: int, status: str) -> None:
    """UI push: broadcast a device connection-status change to all clients."""
    socketio.emit("device_status", {"device_id": device_id, "status": status})
```

把 `create_app` 內建立 `BleRuntime` 的呼叫改為傳入 `on_status`：

```python
    app.extensions[BLE_RUNTIME_KEY] = BleRuntime(
        ble,
        channel_service,
        conn_factory=lambda: connection.connect(config.db_path),
        scan_duration=config.ble_scan_duration,
        on_status=_emit_device_status,
    )
```

- [ ] **Step 4: 跑測試確認通過**

Run: `uv run pytest tests/test_realtime.py -v`
Expected: PASS（4 個 realtime 測試全綠）。

- [ ] **Step 5: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/__init__.py tests/test_realtime.py
git commit -m "feat(web): emit device_status over SocketIO from BleRuntime"
```

---

## Task 5: 前端裝置狀態徽章（路由 + 模板 + dashboard.js）

**Files:**
- Modify: `src/home_server/web/__init__.py`（index 路由帶狀態）
- Modify: `src/home_server/web/devices.py`（detail 路由帶狀態）
- Modify: `src/home_server/web/templates/devices/detail.html`
- Modify: `src/home_server/web/templates/index.html`
- Modify: `src/home_server/web/static/js/dashboard.js`
- Test: `tests/test_frontend.py`

- [ ] **Step 1: 寫失敗測試**

在 `tests/test_frontend.py` 末端追加：

```python
def test_detail_shows_device_status_badge(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:51", "Sensor3")
    body = logged_in_client.get(f"/devices/{device_id}").get_data(as_text=True)
    assert f'data-device-id="{device_id}"' in body


def test_index_shows_device_status_badge(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:52", "Sensor4")
    body = logged_in_client.get("/").get_data(as_text=True)
    assert f'data-device-id="{device_id}"' in body
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `uv run pytest tests/test_frontend.py::test_detail_shows_device_status_badge tests/test_frontend.py::test_index_shows_device_status_badge -v`
Expected: FAIL（模板尚無 `data-device-id` 徽章）。

- [ ] **Step 3: index 路由帶狀態（`web/__init__.py`）**

在 `create_app` 內，把建立 `DeviceService` 那行改為具名變數（供 index 閉包使用）：

把：

```python
    app.extensions[DEVICE_SERVICE_KEY] = DeviceService(ble)
```

改為：

```python
    device_service = DeviceService(ble)
    app.extensions[DEVICE_SERVICE_KEY] = device_service
```

把 index 路由改為帶每裝置狀態的三元組：

```python
    @app.get("/")
    @login_required
    def index() -> str:
        conn = get_conn()
        overview = [
            (
                d,
                "connected" if device_service.is_connected(d.address) else "disconnected",
                db_channels.list_by_device(conn, d.id),
            )
            for d in db_devices.list_all(conn)
        ]
        return render_template("index.html", overview=overview)
```

- [ ] **Step 4: detail 路由帶狀態（`web/devices.py`）**

把 `detail` 函式改為（新增 `status` 並傳入模板；`get_device_service` 已 import）：

```python
@bp.get("/devices/<int:device_id>")
@login_required
def detail(device_id: int) -> str:
    conn = get_conn()
    device = devices.get_by_id(conn, device_id)
    if device is None:
        abort(404)
    device_channels = get_channel_service().list_by_device(conn, device_id)
    status = (
        "connected" if get_device_service().is_connected(device.address) else "disconnected"
    )
    return render_template(
        "devices/detail.html",
        device=device,
        channels=device_channels,
        formats=parser.supported_formats(),
        device_status=status,
    )
```

- [ ] **Step 5: detail.html 加徽章（全檔重寫）**

把 `src/home_server/web/templates/devices/detail.html` 全檔替換為：

```html
{% extends "base.html" %}
{% block title %}{{ device.name }}{% endblock %}
{% block content %}
<p><a href="{{ url_for('devices.list_devices') }}">&larr; Devices</a></p>

<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h1 class="card-title h4">{{ device.name }}
      <span class="badge {{ {'connected':'bg-success','reconnecting':'bg-warning','disconnected':'bg-secondary'}.get(device_status, 'bg-secondary') }}"
            data-device-id="{{ device.id }}">{{ device_status }}</span>
    </h1>
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

- [ ] **Step 6: index.html 三元組 + 徽章（全檔重寫）**

把 `src/home_server/web/templates/index.html` 全檔替換為：

```html
{% extends "base.html" %}
{% block title %}Dashboard{% endblock %}
{% block content %}
<h1 class="h4 mb-3">Dashboard</h1>

{% if overview %}
  {% for device, status, device_channels in overview %}
  <div class="card shadow-sm mb-4">
    <div class="card-body">
      <h2 class="card-title h5">
        <a href="{{ url_for('devices.detail', device_id=device.id) }}">{{ device.name }}</a>
        <span class="badge {{ {'connected':'bg-success','reconnecting':'bg-warning','disconnected':'bg-secondary'}.get(status, 'bg-secondary') }}"
              data-device-id="{{ device.id }}">{{ status }}</span>
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

- [ ] **Step 7: dashboard.js 加 device_status 處理（全檔重寫）**

把 `src/home_server/web/static/js/dashboard.js` 全檔替換為（socket 一律連線，圖表才依 canvas 有無建立）：

```javascript
(function () {
  function init() {
    var sock = io();

    sock.on("device_status", function (d) {
      var cls = { connected: "bg-success", reconnecting: "bg-warning", disconnected: "bg-secondary" }[d.status] || "bg-secondary";
      document.querySelectorAll('[data-device-id="' + d.device_id + '"]').forEach(function (b) {
        b.textContent = d.status;
        b.className = "badge " + cls;
      });
    });

    var canvases = document.querySelectorAll("canvas[data-channel-id]");
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
        })
        .catch(function (err) {
          console.warn("history fetch failed for channel " + channelId, err);
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
    document.addEventListener("DOMContentLoaded", init);
  } else {
    init();
  }
})();
```

- [ ] **Step 8: 跑測試確認通過（含既有 frontend 測試）**

Run: `uv run pytest tests/test_frontend.py -v`
Expected: PASS（含既有 `test_index_dashboard_lists_channels` 仍綠，因新模板照樣渲染頻道）。

- [ ] **Step 9: 全套驗證 + Commit**

```bash
uv run ruff check && uv run mypy src && uv run pytest
git add src/home_server/web/__init__.py src/home_server/web/devices.py src/home_server/web/templates/devices/detail.html src/home_server/web/templates/index.html src/home_server/web/static/js/dashboard.js tests/test_frontend.py
git commit -m "feat(web): per-device connection-status badges (detail + dashboard)"
```

---

## Task 6: `__main__` 平台選用 + 啟動重連監看

**Files:**
- Modify: `src/home_server/__main__.py`

無自動化測試（牽涉真實伺服器與背景執行緒）；以靜態檢查 + import 檢查驗證，執行端冷煙由 orchestrator 另行手動執行。

- [ ] **Step 1: 改寫 `__main__.py`**

匯入區頂部加入 `import sys` 與 selection（最終匯入區如下）：

```python
"""Entry point: `python -m home_server`."""

from __future__ import annotations

import logging
import sys

from home_server.ble.interface import DiscoveredDevice
from home_server.ble.mock_manager import MockBLEManager
from home_server.ble.selection import select_ble_manager
from home_server.config import Config
from home_server.core.logging import setup_logging
from home_server.db import connection
from home_server.services import user_service
from home_server.web import create_app, socketio
from home_server.web.services import get_ble_runtime
```

把 `main()` 內從建立 `ble` 到 `socketio.run` 的段落改為：

```python
    ble = select_ble_manager(config.ble_backend, sys.platform)
    if isinstance(ble, MockBLEManager):
        # Dev demo: discoverable devices so the Scan button shows output.
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
        runtime.monitor_start(interval_s=1.0)

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

- [ ] **Step 2: 靜態驗證 + import 檢查**

Run:
```bash
uv run ruff check && uv run mypy src && uv run pytest
uv run python -c "import home_server.__main__; print('import OK')"
```
Expected: 全綠；印出 `import OK`。

- [ ] **Step 3: Commit**

```bash
git add src/home_server/__main__.py
git commit -m "feat: select BLE backend by platform and start reconnect monitor"
```

---

## Task 7: 更新 RPi-Server 開發文件

**Files:**
- Modify: `docs/RPi-Server 開發文件.md`（注意：此檔在**父 repo** `/Users/eric/ESLAB/Intelligent-home`，不在 submodule）

- [ ] **Step 1: 量測最終測試數**

Run（在 submodule 目錄）：`uv run pytest -q | tail -2`
記下 `N passed`，取代下方 `<N>`。

- [ ] **Step 2: 更新 §11.1 進度表、測試現況與環境變數**

在 `docs/RPi-Server 開發文件.md`：

把 3e-2 那列狀態改為完成：

```markdown
| 3e-2 | 真實 `BluepyManager` 依平台選用、斷線自動重連、`device_status` 事件 | ✅ 完成（mock 單元測試；真硬體待 RPi 冷煙） |
```

把「測試現況」行數字改為實際值：

```markdown
測試現況：<N> unit tests passing、`ruff check` 與 `mypy src`（strict）全綠。
```

在 §11.1 取捨備註區新增一條：

```markdown
> 3e-2 的設計取捨：BLE backend 由 `HOME_SERVER_BLE_BACKEND=auto|mock|bluepy` 選用（`ble/selection.py` 純函式 + bluepy 延遲匯入；auto 在非 Linux 或 bluepy 不可用時 fallback mock）。重連擴充 `BleRuntime`（輪詢 `is_connected` + 指數退避 1→60s + `_bring_up_device` 重新訂閱 display 頻道），偵測機制對 mock/bluepy 一致、`_monitor_tick(now)` 可同步測試；副作用（執行緒、連線）只在 `__main__`。`device_status` 經 SocketIO 廣播、僅狀態轉變才 emit，前端每裝置徽章（server-render 初值 + 即時更新）。真 STM32 + bluepy 整合為 RPi 手動冷煙，不在自動化範圍。
```

在 §9 環境變數表（若存在）補一列：

```markdown
| `HOME_SERVER_BLE_BACKEND` | `auto` | BLE backend 選用：`auto`/`mock`/`bluepy` |
```

- [ ] **Step 3: Commit（父 repo）**

```bash
cd /Users/eric/ESLAB/Intelligent-home
git add "docs/RPi-Server 開發文件.md"
git commit -m "docs: mark RPi-server phase 3e-2 complete"
```

---

## Self-Review（規劃者已執行）

**1. Spec coverage：**
- 平台選用（`select_ble_manager` + `Config.ble_backend` + `__main__`）→ Task 1、Task 6 ✓
- 自動重連（`BleRuntime` 輪詢 + 退避 + 重新訂閱 + monitor 執行緒）→ Task 3、Task 6 ✓
- `device_status` 事件（emit 接線）→ Task 4 ✓
- `DeviceService.is_connected`（server-render 初值）→ Task 2 ✓
- 前端徽章（detail + dashboard + dashboard.js 即時更新）→ Task 5 ✓
- 測試（selection / reconnect / device_status / badge）→ 各 Task ✓
- 文件進度 → Task 7 ✓
- 範圍外（真 bluepy 整合）→ 明列為 RPi 手動冷煙，未自動化 ✓

**2. Placeholder scan：** 無 TBD/TODO；每個程式步驟皆含完整程式碼。`<N>`（Task 7）為實測後填入的測試數，非程式碼佔位。

**3. Type/名稱一致性：**
- `select_ble_manager(backend: str, platform: str) -> BLEManager`、`_load_bluepy()` 一致（Task 1 定義、Task 6 使用）✓
- `BleRuntime.__init__(..., on_status: StatusCallback = _noop_status)`（Task 3）；create_app 傳 `on_status=_emit_device_status`（Task 4）；reconnect 測試傳 `on_status=lambda did, st: ...`（Task 3）—簽章一致 ✓
- `_monitor_tick(now: float)`、`monitor_start/stop`、`_DeviceMonitorState`、`_bring_up_device`、`_set_status` 在 Task 3 定義；Task 4/6 使用一致 ✓
- `DeviceService.is_connected(address: str) -> bool`（Task 2）；detail/index 路由使用（Task 5）✓
- `device_status` payload `{device_id, status}` 與 dashboard.js `d.device_id`/`d.status`、徽章 `data-device-id` 一致 ✓
- 既有 `on_reading`/`subscribe_channel`/`make_feed` 行為不變；`activate` 改用 `_bring_up_device` 後對既有測試透明 ✓
- index 路由三元組 `(device, status, channels)` 與 index.html `{% for device, status, device_channels in overview %}` 一致 ✓

---

*計畫結束。*
