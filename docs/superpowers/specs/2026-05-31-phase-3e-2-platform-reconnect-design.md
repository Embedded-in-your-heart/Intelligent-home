# Phase 3e-2 — BLE backend 平台選用 + 自動重連 + device_status 設計文件

> 對應主文件：[RPi-Server 開發文件](../../RPi-Server%20開發文件.md) §4.1.3（連線管理/重連）、§4.3.4（`device_status` 事件）、§11.1（子階段 3e-2）
> 對應原始碼目錄：`Intelligent-home-RPi-server/src/home_server/`
> 日期：2026-05-31
> 前置：Phase 3e-1（SocketIO 即時推播 + HTMX/Chart.js 前端）已完成並合併。

---

## 1. 目標與範圍

補齊 RPi 端「真實硬體」相關的連線韌性：依平台選用 BLE backend、斷線自動重連（指數退避）、並把連線狀態即時推播到前端。

**驗收標準**：
- 開發機（macOS）：`uv run pytest / ruff check / mypy src`（strict）全綠；以 `MockBLEManager` + `simulate_disconnect` 在單元測試中完整驗證平台選用、重連狀態機與 `device_status` 推播；瀏覽器每裝置卡片顯示連線狀態徽章。
- RPi（手動冷煙，本階段不自動化）：`HOME_SERVER_BLE_BACKEND=bluepy` 接真 STM32，斷電重開後徽章 `disconnected → reconnecting → connected` 且讀數恢復。

### 1.1 範圍內（3e-2）

- BLE backend 平台選用：`MockBLEManager`（非 Linux / 開發）vs `BluepyManager`（Linux / RPi），由 `HOME_SERVER_BLE_BACKEND=auto|mock|bluepy` 控制（預設 `auto`）。
- 自動重連：背景輪詢偵測斷線 → 指數退避（1s→2s→4s→…→上限 60s）重連 → 重新訂閱該裝置 display 頻道。
- `device_status` SocketIO 事件（`connected` / `reconnecting` / `disconnected`）+ 每裝置前端狀態徽章（詳情頁 + Dashboard）。
- 對應單元測試；維持 `ruff` / `mypy --strict` 全綠。

### 1.2 範圍外

- 真實 STM32 + bluepy 的整合驗證（需 RPi 硬體）——列為手動冷煙步驟，不在本階段自動化。
- 多節點分散式佈署測試、`systemd` 部署（Phase 4）。
- 連線品質指標（RSSI 持續監看、訊號統計）等增強功能（YAGNI）。

---

## 2. 現況盤點（為何 3e-2 多為「接線 + 一個狀態機」）

讀過程式碼後確認：

- **`ble/bluepy_manager.py` 已完整實作**：`BluepyManager` facade（`start_scan` / `connect` / `disconnect` / `is_connected` / `read` / `write` / `subscribe` / `unsubscribe`）+ per-peripheral worker thread + `_NotifyDelegate`。模組頂部 `if sys.platform != "linux": raise ImportError(...)`（line 25），故**在非 Linux 匯入此模組會直接拋 ImportError**。worker 偵測到 `BTLEDisconnectError` 會結束並使 `is_connected` 回 False，但**自己不重連**（docstring：「Reconnect logic lives in the service layer」）。
- **無任何 reconnect / device_status / 平台選用**邏輯：`__main__` 永遠 `MockBLEManager()`；`Config` 無 backend 欄位。
- **`MockBLEManager.simulate_disconnect(handle)`**（line 154）標記斷線但**保留訂閱**，正是為了測試重連；`is_connected` 兩個 backend 皆有。
- `BleRuntime` 目前：`activate()`（連線所有裝置 + 訂閱 display 頻道）、`subscribe_channel()`、`make_feed()`；持有 `_ble` / `_channel_service` / `_conn_factory` / `_subscribed`。
- `DeviceService` 持有 `_ble`，有 `scan` / `add_device` / `remove_device` / `list_devices`。

**缺口（本階段補）**：① 平台選用函式 + `Config.ble_backend`；② `BleRuntime` 重連 supervisor（輪詢 + 退避 + 重新訂閱 + 狀態回呼）；③ `device_status` emit + 前端徽章；④ `DeviceService.is_connected` 供 server-render 初始徽章。

---

## 3. 關鍵設計決策

### 3.1 平台選用：純函式 + 延遲匯入（採方案：auto + env 覆寫）

新增 `src/home_server/ble/selection.py`：

```python
def select_ble_manager(backend: str, platform: str) -> BLEManager:
    if backend == "mock":
        return MockBLEManager()
    if backend == "bluepy":
        return _load_bluepy()  # 非 Linux → ImportError（fail fast）
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
    from home_server.ble.bluepy_manager import BluepyManager  # 延遲匯入
    return BluepyManager()
```

- **延遲匯入**：`bluepy_manager` 只在需要時匯入，故本模組可在 Mac 安全 import 與測試。
- `Config.ble_backend`：由 `HOME_SERVER_BLE_BACKEND` 讀取，預設 `"auto"`（沿用 `config.py` 既有 `_env_str` 模式）。
- `__main__`：`ble = select_ble_manager(config.ble_backend, sys.platform)`；demo `scan_results` 與 `mock.start(feed)` 維持 `isinstance(ble, MockBLEManager)` 守衛。

**可在 Mac 測試的路徑**：`mock`→Mock；`auto`+`"darwin"`→Mock；`auto`+`"linux"`→（Mac 上 `_load_bluepy` 因模組守衛拋 ImportError）→ fallback Mock；`bluepy`→ `pytest.raises(ImportError)`；未知→ `pytest.raises(ValueError)`。

被否決替代：只看 `sys.platform`（無法在 RPi 強制 mock 開發）；強制顯式 env（部署摩擦大）。

### 3.2 自動重連：擴充 `BleRuntime`，輪詢偵測（採方案 A）

**架構取捨（方案 A vs B）**：
- **方案 A（採用）**：擴充 `BleRuntime`。重連需 `connect` + 重新訂閱 display 頻道，這套生命週期與所需狀態（`_subscribed`、`conn_factory`、`_ble`）皆已在 `BleRuntime`。內聚於單一職責「BLE 連線生命週期（連線、訂閱、重連、回報狀態）」。
- **方案 B（否決）**：另開 `ConnectionSupervisor`。需反向引用 `BleRuntime` 的 bring-up/subscribe 與重複注入相依，wiring 較多、耦合未減。

**偵測機制：輪詢（而非 callback）**。每 tick 以 `ble.is_connected(address)` 比對狀態，對 mock 與 bluepy **行為一致**（mock 由 `simulate_disconnect` 翻轉、bluepy 由 worker 結束翻轉），且 `_monitor_tick` 可同步單元測試，無需改 `BLEManager` 介面。

`BleRuntime` 變更：
- 建構式新增 `on_status: Callable[[int, str], None]`（device_id, status）。
- 抽出 `_bring_up_device(conn, device) -> bool`：`connect` + 訂閱該裝置所有 display 頻道；回傳成功與否。`activate()` 改用之（初始連線失敗的裝置也會被 monitor 後續重試）。
- 退避常數（模組級）：`_RECONNECT_BASE_S = 1.0`、`_RECONNECT_FACTOR = 2.0`、`_RECONNECT_CAP_S = 60.0`。
- 每裝置狀態：`_DeviceMonitorState`（`last_status: str | None`、`next_retry_at: float`、`backoff_s: float`）以 `dict[address, _DeviceMonitorState]` 維護。
- `monitor_start(interval_s=1.0)` / `monitor_stop()`：背景 daemon 執行緒，仿 `MockBLEManager.start/stop`（`threading.Event` + `join(timeout)`）。
- `_monitor_tick(now: float) -> None`（可同步測試）：
  1. 重查 `devices.list_all(conn)`（自動納入新增、移除已刪裝置的狀態）。
  2. 對每裝置：
     - `is_connected` 為 True：`backoff_s` 重設為 base、`next_retry_at` 清零；若 `last_status != "connected"` → `_set_status(device_id, "connected")`。
     - 為 False：
       - **轉入斷線**（`last_status` 為 `None`/`"connected"`）→ `_set_status(device_id, "disconnected")`、`backoff_s = base`、`next_retry_at = now + backoff_s`（下次 tick 才重試，避免同一 tick 內 `disconnected`+`reconnecting` 雙重 emit）。
       - **否則**（已在斷線/重連中）若 `now >= next_retry_at` → `_set_status(device_id, "reconnecting")`；嘗試 `_bring_up_device`：成功 → `backoff_s = base`、`_set_status(connected)`；失敗 → `backoff_s = min(backoff_s * factor, cap)`、`next_retry_at = now + backoff_s`（維持 `reconnecting`）。
- `_set_status(device_id, status)`：僅在與 `last_status` 不同才更新並呼叫 `on_status`（去重，不灌爆）。

> 注意：`now` 用單調時鐘。生產用 `time.monotonic`；測試以注入的 `clock`/直接傳 `now` 驅動 `_monitor_tick`，不依賴真實時間。`monitor_start` 內部迴圈使用 `time.monotonic`。

重連涵蓋**所有裝置**（含僅 controller 的裝置，使其 write 在重連後可用）；無 display 頻道者重連後僅恢復連線、不訂閱。

### 3.3 device_status 事件 + 前端徽章

- `create_app` 新增模組級 `_emit_device_status(device_id, status)` → `socketio.emit("device_status", {"device_id": device_id, "status": status})`（廣播；socket 已於 3e-1 要求登入認證），傳入 `BleRuntime(on_status=_emit_device_status, ...)`。
- 初始狀態 server-render：新增 `DeviceService.is_connected(address) -> bool`（包 `self._ble.is_connected`）。詳情頁與 Dashboard 路由為每裝置算出 `"connected"`/`"disconnected"` 初值傳入模板。
- 模板每裝置卡片放一個徽章元素：`<span class="badge" data-device-id="{{ device.id }}">`，初值 class 依狀態（connected→`bg-success`、reconnecting→`bg-warning`、disconnected→`bg-secondary`）。
- `dashboard.js` 監聽 `socket.on("device_status", ...)`：依 `device_id` 找對應徽章，更新文字與 class。

事件契約（與文件 §4.3.4 一致）：

| 方向 | Event | Payload |
| --- | --- | --- |
| S→C | `device_status` | `{device_id: int, status: "connected" \| "reconnecting" \| "disconnected"}` |

---

## 4. 設定與啟動

### 4.1 `config.py`
新增 `ble_backend: str` 欄位，`from_env` 以 `_env_str("HOME_SERVER_BLE_BACKEND", "auto")` 載入。

### 4.2 `__main__.py`
- `import sys`；`from home_server.ble.selection import select_ble_manager`。
- `ble = select_ble_manager(config.ble_backend, sys.platform)`（取代 `MockBLEManager()`）。
- demo `scan_results` 與 `mock.start(runtime.make_feed())` 包在 `if isinstance(ble, MockBLEManager):`。
- `runtime.activate()` 後呼叫 `runtime.monitor_start(interval_s=1.0)`（daemon thread；程序結束自然回收）。

---

## 5. 影響的檔案

**新增：**
- `src/home_server/ble/selection.py`
- `tests/test_ble_selection.py`
- `tests/test_ble_runtime_reconnect.py`

**修改：**
- `src/home_server/config.py`（`ble_backend` 欄位 + `from_env`）
- `src/home_server/services/ble_runtime.py`（`on_status`、`_bring_up_device`、`activate` 重構、monitor、`_monitor_tick`、退避狀態）
- `src/home_server/services/device_service.py`（`is_connected`）
- `src/home_server/web/__init__.py`（`_emit_device_status`、`BleRuntime(on_status=...)`、index 路由帶每裝置狀態）
- `src/home_server/web/devices.py`（detail 路由帶裝置連線狀態）
- `src/home_server/web/templates/devices/detail.html`（裝置卡片狀態徽章）
- `src/home_server/web/templates/index.html`（每裝置狀態徽章）
- `src/home_server/web/static/js/dashboard.js`（`device_status` 處理）
- `src/home_server/__main__.py`（select + monitor_start）
- `tests/test_realtime.py`（device_status 推播測試）、`tests/test_frontend.py`（徽章 render 測試）、`tests/test_ble_runtime.py`（`activate` 重構後仍綠 + `on_status` 注入）
- `docs/RPi-Server 開發文件.md`（§11.1 標 3e-2 完成、測試數、取捨備註）

**不修改：** `ble/bluepy_manager.py`（已完整）、`ble/mock_manager.py`（`simulate_disconnect` 已備）、`ble/interface.py`、`services/channel_service.py`、`db/*`、`db/schema.sql`、`rate_limiter.py`、`parser.py`。

---

## 6. 測試策略

維持「非 BLE 層完整測試、測試不起背景執行緒（除生命週期測試短暫啟停）」原則。

**`tests/test_ble_selection.py`（新增）**
- `select_ble_manager("mock", "darwin")` → `MockBLEManager`。
- `select_ble_manager("auto", "darwin")` → `MockBLEManager`。
- `select_ble_manager("auto", "linux")`（在 Mac 跑）→ fallback `MockBLEManager`（bluepy 模組守衛拋 ImportError）。
- `select_ble_manager("bluepy", "linux")` → `pytest.raises(ImportError)`（Mac 上）。
- `select_ble_manager("nonsense", "linux")` → `pytest.raises(ValueError)`。

**`tests/test_ble_runtime_reconnect.py`（新增，mock 驅動 + 注入 clock）**
- 連線後 `simulate_disconnect` → `_monitor_tick` → `on_status` 收到 `disconnected`。
- 退避到期 → tick → `reconnecting`，`_bring_up_device` 成功（mock）→ `connected`，且該裝置 display 頻道恢復訂閱（`simulate_notify` 不再拋、或 `_subscribed` 復原）。
- 連線持續失敗（`mock.fail_connect_for`）→ `backoff_s` 1→2→4…並夾在 60s 上限；以注入 `now` 驗證 `next_retry_at` 推進。
- 僅狀態轉變才 emit（連續 connected 不重複 emit）。
- `monitor_start` / `monitor_stop`：啟動後執行緒 alive、`monitor_stop` 後 `not thread.is_alive()`，無洩漏。
- 重連涵蓋僅 controller 的裝置（無 display 頻道也會被重連、不訂閱、狀態正確）。

**`tests/test_realtime.py`（擴充）**
- 已認證 `socketio.test_client`：直接呼叫 `runtime._monitor_tick`（或 `_set_status`）驅動一次轉變 → client `get_received()` 含 `device_status`，payload 正確；未認證 client 仍被拒。

**`tests/test_frontend.py`（擴充）**
- 詳情頁 / Dashboard 每裝置含 `data-device-id` 徽章元素，初值 class 反映 `device_service.is_connected`（mock 未連 → disconnected class）。

**`tests/test_ble_runtime.py`（擴充）**
- `activate` 重構後既有測試仍綠；建構式 `on_status` 注入（既有測試以 no-op lambda 帶入）。

**`tests/test_config.py`（若存在則擴充，否則於 selection 測試覆蓋）**：`HOME_SERVER_BLE_BACKEND` 預設 `auto`、可覆寫。

驗證：`uv run ruff check`、`uv run mypy src`、`uv run pytest`（既有測試全綠）。`dashboard.js` 為純 JS，不納入 ruff/mypy。

---

## 7. 風險與待決

| 項目 | 風險 | 處理 |
| --- | --- | --- |
| 真實 bluepy 行為與 mock 差異 | mock 全綠不代表硬體通過 | 明列 RPi 手動冷煙；偵測走統一的 `is_connected` 輪詢，硬體只換注入 backend |
| monitor 每 tick 重查 DB | 高頻 DB 讀 | tick 間隔 1s、裝置數少、走短連線唯讀，負載可忽略 |
| `device_status` 廣播範圍 | 資訊外洩 | socket 已要求登入（3e-1）；沿用 flat 模型不做 owner 過濾 |
| 重連風暴（裝置永遠連不上） | 不斷重試 | 指數退避至 60s 上限；僅轉變才 emit |
| `_monitor_tick` 與 notify worker 並行寫 DB | 並發 | 沿用 WAL + 每次短連線；tick 僅讀 `devices` |

---

## 8. 與文件的偏離說明

文件 §4.1.3 將重連描述在 BLE 層；本設計依 `bluepy_manager` docstring 的既定分工，把重連放在 service 層（`BleRuntime`），BLE 層只回報 `is_connected`。`device_status` 事件與 §4.3.4 一致（`connected`/`reconnecting`/`disconnected`）。平台選用新增 `HOME_SERVER_BLE_BACKEND`（文件 §9 環境變數表可於實作時補列）。

---

*文件結束。實作前若需調整任何決策，請於此討論。*
