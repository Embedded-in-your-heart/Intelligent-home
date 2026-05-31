# Phase 3e-1 — SocketIO 即時推播 + HTMX/Chart.js 前端 設計文件

> 對應主文件：[RPi-Server 開發文件](../../RPi-Server%20開發文件.md) §11.1（子階段 3e）、§4.1.3–4.1.4、§4.3.4
> 對應原始碼目錄：`Intelligent-home-RPi-server/src/home_server/`
> 日期：2026-05-31

---

## 1. 目標與範圍

接線「BLE Notify → SocketIO 即時推播 → 前端 Chart.js 趨勢圖」全鏈路，並補上 3e 範圍的三個 Web 端點（`/devices/scan`、`/channels/<id>/write`、`/channels/<id>/history`），整合 HTMX 與 Chart.js 前端。

**驗收標準**：在開發機（macOS）以 `MockBLEManager` 啟動後，瀏覽器可完成「掃描 → 加入裝置 → 新增頻道 → 對 display 頻道**看到趨勢圖即時跳動** → 對 controller 頻道下達控制 → 檢視歷史趨勢」全流程，且 `uv run pytest` / `ruff check` / `mypy src`（strict）全綠。

### 1.1 範圍內（3e-1，Mac + Mock 可驗證）

- SocketIO 即時推播接線：`on_reading` callback → `socketio.emit`（取代 `_noop_reading`）。
- Notify 訂閱 wiring：對所有 display 頻道 `subscribe`，notify 進 `channel_service.handle_notify`（已實作）。
- `MockBLEManager` 背景產數：定時對已訂閱頻道送出合成讀數，使 Mac 上可端到端看到圖跳動。
- 三個 Web 端點：`GET /devices/scan`、`POST /channels/<id>/write`、`GET /channels/<id>/history`。
- 前端：vendored `socket.io` / `htmx` / `chart.js`；裝置詳情頁 per-channel 即時圖與控制；首頁總覽 Dashboard；裝置列表頁掃描 UI。
- 對應單元測試；維持 `ruff check`、`mypy src`（strict）全綠。

### 1.2 範圍外（留待 3e-2，需 RPi 硬體）

- 真實 `BluepyManager` 啟用與依平台選用（`create_app` 注入 `BluepyManager`）。
- 斷線自動重連背景迴圈（文件 §4.1.3 指數退避）。
- `device_status` 連線狀態 SocketIO 事件（依賴真實連線生命週期）。
- 多節點佈署、systemd（Phase 4）。

---

## 2. 現況盤點（為何 3e-1 多為「接線」而非「實作」）

讀過程式碼後確認下列**已就緒**，本階段不重做：

- `ChannelService.write_command(conn, *, channel_id, raw_value)`：驗證 controller 型別 → `parser.encode(raw_value, channel.data_format)` → `ble.write`。**已實作**。
- `ChannelService.handle_notify(conn, *, channel_id, raw_bytes)`：`parser.decode` → 呼叫 `on_reading`（不限頻、try/except 包覆）→ `RateLimiter.should_emit` 為真才 `readings.insert`。**已實作**。
- `ChannelService.get_history(conn, channel_id, *, since, until, limit)`：`readings.list_by_channel`（oldest→newest）。**已實作**。
- `DeviceService.scan(duration_s)` → `ble.start_scan`。**已實作**。
- `db/readings.py`、`db/channels.py`（含 `data_format`、`unit` 欄位）、`ble/parser.py`（uint8/int*/float32 等）、`ble/rate_limiter.py`。**已就緒**。
- `pyproject.toml` 已含 `flask-socketio>=5.3`、`simple-websocket>=1.0`。**無須新增 Python 相依**。

**缺口（本階段要補）**：① `create_app` 仍注入 `_noop_reading`、未初始化 SocketIO；② 無人呼叫 `subscribe` / `handle_notify`（notify 從未被觸發）；③ Web 端點 `scan` / `write` / `history` 尚未實作；④ Mock 不會自行產數；⑤ 前端無即時圖、控制、掃描 UI。

> **更正先前假設**：規劃對話中曾假設「`channels` 表無編碼欄位、控制寫入須寫死單一 byte」。實際 schema **已有 `data_format`**，且 `write_command` 已依此編碼。本設計沿用既有編碼（uint8 自然為 1 byte、float32_le 為 4 bytes），UI 仍為單一數值輸入欄位；零改動 service 層。

---

## 3. 關鍵設計決策

### 3.1 SocketIO 採模組級單例 + `init_app`（`create_app` 簽章不變）

```python
# web/__init__.py
from flask_socketio import SocketIO

socketio = SocketIO()  # 模組級單例（Flask-SocketIO 慣例）

def create_app(config: Config, ble: BLEManager | None = None) -> Flask:
    ...
    socketio.init_app(app, async_mode="threading")
    ...
    return app  # 回傳型別維持 Flask 不變
```

- **理由**：改成 `create_app -> tuple[Flask, SocketIO]`（如文件 §4.3.1 示意）會牽動 `conftest.py` 的 `app` fixture 與所有既有測試。採模組級單例可讓 `create_app` 維持回傳 `Flask`，是 surgical change。
- `threading` 模式底層用已在相依的 `simple-websocket` + Werkzeug，與 `bluepy` 的同步呼叫共存。
- **測試隔離**：每個測試由 fixture 建立獨立 app 後立即使用，`socketio.test_client(app)` 綁定當前 app；模組級單例的 server 每次 `init_app` 重綁，HTTP-only 測試不受影響（用 `app.test_client()`）。此取捨於文件記錄。

被否決替代：回傳 tuple（牽動全測試）、每請求建 SocketIO（不合理）。

### 3.2 即時推播：`on_reading` → `emit`，per-channel room

`create_app` 內以閉包建立 `_emit_reading` 取代 `_noop_reading`，注入 `ChannelService`：

```python
def _emit_reading(channel_id: int, value: float, timestamp: str) -> None:
    socketio.emit(
        "reading",
        {"channel_id": channel_id, "value": value, "timestamp": timestamp},
        room=f"channel:{channel_id}",
    )
```

- 依文件 §4.3.4：前端 emit `subscribe_channel {channel_id}` → server `join_room(f"channel:{channel_id}")`；`unsubscribe_channel` → `leave_room`。
- `handle_notify` 已用 try/except 包覆 `on_reading`，故死掉的 SocketIO client 不會中斷限頻 DB 寫入。

### 3.3 Notify 訂閱 wiring + 啟用：新增 `services/ble_runtime.py`

新增輕量協調器 `BleRuntime`，把「連線所有裝置、訂閱所有 display 頻道、notify→handle_notify」收斂在一處：

```python
class BleRuntime:
    def __init__(
        self,
        ble: BLEManager,
        channel_service: ChannelService,
        conn_factory: Callable[[], sqlite3.Connection],
        *,
        scan_duration: float,
    ) -> None: ...

    def activate(self) -> None:
        """連線所有已知裝置，訂閱所有 display 頻道。供 __main__ 呼叫。"""

    def subscribe_channel(self, address: str, channel: Channel) -> None:
        """對單一 display 頻道 subscribe；notify callback 開短連線呼叫 handle_notify。"""

    def make_feed(self) -> Callable[[str, str], bytes | None]:
        """給 MockBLEManager 背景執行緒用的 format-aware 合成讀數產生器。"""

    def stop(self) -> None: ...
```

- **連線管理 = worker 執行緒短生命週期連線**：notify callback 在 BLE worker 執行緒被呼叫，`sqlite3` 連線須執行緒專屬，故 callback 內 `conn = conn_factory(); try: handle_notify(...); finally: conn.close()`（符合文件 §3.2、§4.2.1）。
- **副作用隔離**：`create_app` 只**建構** `BleRuntime` 並存入 `app.extensions`（inert，不連線、不開執行緒）；`activate()` 與 `mock.start()` 只在 `__main__` 呼叫。測試不會起背景執行緒，但可手動呼叫 `activate()` / `subscribe_channel()` 驗證 wiring。
- 存取器：`web/services.py` 新增 `get_ble_runtime()`（仿既有 `get_device_service` 型別化存取）。

被否決替代：把 wiring 塞進 `DeviceService`（職責膨脹、且需 conn_factory 與 ble 一起）；放 `web/`（非 HTTP 概念）。

### 3.4 Mock 背景產數：`start(feed, interval)` / `stop()`，格式知識外置

`MockBLEManager` 新增：

```python
def start(self, feed: Callable[[str, str], bytes | None], interval_s: float = 1.0) -> None: ...
def stop(self) -> None: ...
def _tick(self, feed) -> None:  # 單次掃描所有 subscription 並觸發 callback（供測試同步呼叫）
```

- daemon 執行緒每 `interval_s` 呼叫 `_tick`：對每個 active subscription `(addr, uuid)`，`data = feed(addr, uuid)`；非 `None` 則觸發該訂閱 callback（重用既有 notify 路徑）。
- **格式知識不進 mock**：`feed` 由 `BleRuntime.make_feed()` 提供，內部依 `(addr, uuid)→channel.data_format` 用 `parser.encode(合成值, fmt)` 產生正確 bytes。合成值用簡單正弦或隨機 walk（依 `data_format` 取合理範圍）。
- 既有 `simulate_notify()` / `set_read_value()` / `queue_read_values()` **保留**給單元測試手動驅動。
- `stop()` 以 `threading.Event` 通知並 `join`，確保乾淨退出、無執行緒洩漏。

### 3.5 控制寫入：單一數值欄位 + 既有 `data_format` 編碼

- UI：controller 頻道顯示一個數值輸入 + 送出鈕。
- 後端：沿用 `write_command(conn, channel_id=..., raw_value=float(value))`；`parser.encode` 依該頻道 `data_format` 編碼（不寫死單一 byte）。零改動 service。

### 3.6 前端 vendoring（沿用 Bootstrap 既有作法，非 CDN）

專案既有把 Bootstrap vendored 於 `static/vendor/`。為與之一致並支援 RPi 離線：把 `socket.io` client、`htmx`、`chart.js` 同樣 vendored 於 `static/vendor/`，由 `base.html` 以 `<script>` 載入。

被否決替代：CDN（與既有 vendoring 不一致、RPi 離線環境不可靠）。

---

## 4. Web 端點設計

皆 `@login_required`，沿用 flat 權限模型（不做 owner 檢查）。

### 4.1 Device blueprint（`web/devices.py`，擴充）

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/devices/scan` | `get_device_service().scan(config.ble_scan_duration)` → 渲染 partial `devices/_scan_results.html`。每筆發現裝置一個預填 `address`/`name` 的「加入」表單，POST 至既有 `POST /devices`。 |

- 回傳僅片段 HTML（供 HTMX `hx-get` 置換目標容器）。
- mock 預設提供數筆 `scan_results` 以利展示（於 `__main__` 用 mock 時設定，或 mock 預設值）。

### 4.2 Channel blueprint（`web/channels.py`，擴充）

| Method | Path | 說明 |
| --- | --- | --- |
| POST | `/channels/<int:channel_id>/write` | 表單 `value`（數值）→ `write_command(conn, channel_id=id, raw_value=float(value))`。 |
| GET | `/channels/<int:channel_id>/history` | `get_history(conn, id, limit=limit)` → `jsonify`。`limit` 由 query param 提供，預設 200、上限 1000（避免一次回傳過多點）。 |

錯誤處理：

| 操作 | 例外 / 條件 | 行為 |
| --- | --- | --- |
| write | `value` 無法轉 float / 缺欄位 | flash 錯誤 + 導回詳情（或 HTMX 回 400） |
| write | `WrongChannelTypeError`（非 controller） | 400 |
| write | `ChannelNotFoundError` | 404 |
| write | 成功 | 導回裝置詳情（HTMX 則回 204 No Content） |
| history | 頻道不存在 | 404 |
| history | 無資料 | 200 + `{"channel_id": id, "readings": []}` |

history 回傳格式：

```json
{"channel_id": 3, "readings": [{"value": 24.5, "recorded_at": "2026-05-31 10:00:00"}, ...]}
```

`readings` 由舊到新（`get_history` 已 ASC 排序），供 Chart.js 直接畫線。

### 4.3 SocketIO 事件（`web/__init__.py` 內註冊）

| 方向 | Event | Payload | 行為 |
| --- | --- | --- | --- |
| C→S | `subscribe_channel` | `{channel_id}` | `join_room(f"channel:{id}")` |
| C→S | `unsubscribe_channel` | `{channel_id}` | `leave_room(f"channel:{id}")` |
| S→C | `reading` | `{channel_id, value, timestamp}` | `handle_notify` 經 `on_reading` emit 至該 room |

> `device_status` 事件留待 3e-2（需真實連線生命週期）。

---

## 5. 前端設計

### 5.1 `base.html`（小幅修改）

`<head>`/`<body>` 末端加入 vendored `<script>`：`chart.js`、`socket.io`、`htmx`。navbar 既有 Devices 連結；首頁連結即 Dashboard。

### 5.2 `static/js/dashboard.js`（新增）

單一腳本，供詳情頁與首頁共用：

1. 建立 `const sock = io();`
2. 對每個 `<canvas data-channel-id="N" data-unit="...">`：
   - `fetch('/channels/N/history')` → 以回傳 readings 初始化 Chart.js line chart。
   - `sock.emit('subscribe_channel', {channel_id: N})`。
   - `sock.on('reading', d => { if (d.channel_id === N) { 追加 (d.timestamp, d.value)，裁切視窗長度，chart.update() } })`。
3. controller 控制表單以原生 POST（漸進增強：可選 `hx-post` + `hx-swap="none"`）。

### 5.3 `devices/detail.html`（修改）

頻道表格每列依 `type`：
- `display`：加一個 `<canvas data-channel-id>` 即時趨勢圖區塊。
- `controller`：加「數值輸入 + 送出」表單，POST `/channels/<id>/write`（含 `csrf_token`）。

### 5.4 `devices/list.html`（修改）+ `devices/_scan_results.html`（新增）

- 列表頁加「Scan」按鈕：`hx-get="/devices/scan"`、`hx-target="#scan-results"`、`hx-swap="innerHTML"`，下方 `<div id="scan-results">`。
- partial 列出發現裝置，每筆一個預填 `address`/`name` 的「加入」表單 POST `/devices`。

### 5.5 `index.html`（改為 Dashboard）

`index` route 改為帶入所有裝置與其頻道；模板列出各頻道：display 顯示迷你即時圖、controller 顯示控制，重用 `dashboard.js`。

---

## 6. `__main__.py` 與啟動流程（修改）

```python
ble = MockBLEManager()                # 3e-2 改為依平台選 BluepyManager
app = create_app(config, ble=ble)     # 顯式注入，__main__ 才握有 ble handle
with app.app_context():
    runtime = get_ble_runtime()       # 從 app.extensions 取（需 app context）
    runtime.activate()                # 連線 + 訂閱所有 display 頻道
    if isinstance(ble, MockBLEManager):   # 開發機用 mock：啟動背景產數
        ble.start(runtime.make_feed(), interval_s=1.0)
socketio.run(app, host=config.host, port=config.port, use_reloader=False)
```

- 顯式建立 `ble` 並 `create_app(config, ble=ble)`，使 `__main__` 握有 `ble` handle（`create_app` 仍可預設自建 mock 供測試）。
- `app.run(...)` → `socketio.run(app, ...)`。
- macOS 開發以 `HOME_SERVER_PORT=5001`（AirPlay 佔 5000）；正式於 RPi 用 `config.port`。
- 3e-2 才把 `ble` 換成 `BluepyManager` 並加自動重連；屆時 `mock.start` 分支自然略過。

---

## 7. 測試策略

維持「非 BLE 層完整測試、不在測試起背景執行緒」原則。新增/擴充：

**`tests/test_web_channels.py`（擴充）**
- write：controller 成功 → 斷言 `mock.writes` 記到正確 `(address, char_uuid, bytes)`。
- write：display 頻道 → 400。
- write：`value` 缺/非數值 → 400 或 flash。
- write：頻道不存在 → 404。
- history：回傳 JSON 結構與排序（舊→新）。
- history：無資料 → `{"readings": []}`。
- history：頻道不存在 → 404。
- 未登入 → 302。

**`tests/test_web_devices.py`（擴充）**
- scan：回傳 partial 含 mock 預設發現裝置；需登入。

**`tests/test_ble_runtime.py`（新增）**
- `activate()`：對既有 device + display 頻道呼叫 `connect` 與 `subscribe`（用 mock 斷言 `_subscriptions`）。
- `subscribe_channel` 後 `mock.simulate_notify(addr, uuid, encoded)` → `readings` 寫入一筆且 `on_reading` 被呼叫（用注入的 spy callback）。
- `make_feed()` 對已知 `(addr, uuid)` 回傳可被 `parser.decode` 正確解析的 bytes；未知回 `None`。
- controller 頻道**不**被 `activate` 訂閱。

**`tests/test_realtime.py`（新增）**
- `socketio.test_client(app)`：emit `subscribe_channel {channel_id}`；**同步**呼叫 `channel_service.handle_notify(conn, channel_id, raw)`（避免背景執行緒不確定性）→ `client.get_received()` 含一筆 `reading` 且 payload 正確。
- 未訂閱該 room 的 client 不應收到該頻道 reading。

**`tests/test_mock_manager.py`（擴充）**
- `_tick(feed)`：對 active subscription 觸發 callback；feed 回 `None` 則不觸發。
- `start/stop` 生命週期：`stop()` 後執行緒結束（`thread.is_alive()` 為 False），無洩漏。

驗證指令（沿用）：`uv run ruff check`、`uv run mypy src`、`uv run pytest`（現有 125 tests 仍須全綠）。

> mypy：`flask_socketio` 無型別 stub，於 `pyproject.toml` 的 `[[tool.mypy.overrides]]` 加 `flask_socketio.*` 至 `ignore_missing_imports`；SocketIO 事件處理器位於 `home_server.web.*`，已由既有 `disallow_untyped_decorators = false` 覆蓋。

---

## 8. 影響的檔案

**新增：**
- `src/home_server/services/ble_runtime.py`
- `src/home_server/web/templates/devices/_scan_results.html`
- `src/home_server/web/static/js/dashboard.js`
- `src/home_server/web/static/vendor/socketio/socket.io.min.js`
- `src/home_server/web/static/vendor/htmx/htmx.min.js`
- `src/home_server/web/static/vendor/chartjs/chart.umd.min.js`
- `tests/test_ble_runtime.py`
- `tests/test_realtime.py`

**修改：**
- `src/home_server/web/__init__.py`（SocketIO `init_app` + `_emit_reading` + 事件處理器 + 建構並存 `BleRuntime` + index route 帶資料）
- `src/home_server/web/services.py`（新增 `get_ble_runtime` + `BLE_RUNTIME_KEY`）
- `src/home_server/web/devices.py`（`GET /devices/scan`）
- `src/home_server/web/channels.py`（`POST /channels/<id>/write`、`GET /channels/<id>/history`）
- `src/home_server/ble/mock_manager.py`（`start` / `stop` / `_tick`）
- `src/home_server/__main__.py`（`socketio.run` + `activate` + mock `start`）
- `src/home_server/web/templates/base.html`（vendored scripts）
- `src/home_server/web/templates/index.html`（Dashboard）
- `src/home_server/web/templates/devices/list.html`（Scan UI）
- `src/home_server/web/templates/devices/detail.html`（per-channel 圖 / 控制）
- `pyproject.toml`（mypy override 加 `flask_socketio.*`）
- `tests/test_web_channels.py`、`tests/test_web_devices.py`、`tests/test_mock_manager.py`

**不修改：** `services/channel_service.py`、`services/device_service.py`、`db/*`、`ble/parser.py`、`ble/rate_limiter.py`、`ble/interface.py`、`ble/bluepy_manager.py`、`db/schema.sql`、`config.py`（如實作中發現必要，於 plan 提出）。

---

## 9. 風險與待決

| 項目 | 風險 | 處理 |
| --- | --- | --- |
| 模組級 socketio 單例跨測試殘留狀態 | 測試交互影響 | 每測試獨立 app fixture；WS 測試用 `socketio.test_client(app)`；必要時測試後 `socketio` 不持久連線 |
| worker 執行緒短連線寫入 SQLite | 並發寫 | schema 已啟用 WAL；每 notify 短連線、限頻寫入；序列化由 mock/bluepy worker 處理 |
| vendored 前端資產取得 | 需下載三個 JS 檔 | 實作時下載固定版本置於 `static/vendor/`，記錄版本 |
| `socketio.run` 與 Flask debug reloader | reloader 重啟造成雙執行緒 | `use_reloader=False`（沿用既有 `__main__`） |
| 真實 bluepy 行為差異 | mock 通過不代表硬體通過 | 明列 3e-2 在 RPi 上整合驗證；wiring 走介面，硬體只換注入 |

---

## 10. 與文件路由表的偏離說明

文件 §4.3.3 將刪除列為 `DELETE` method；3d 已決議用純 POST（`POST .../delete`）。本階段新增的 `write`（POST）、`history`（GET）、`scan`（GET）與文件一致。SocketIO 事件與 §4.3.4 一致（`device_status` 留 3e-2）。

---

*文件結束。實作前若需調整任何決策，請於此討論。*
