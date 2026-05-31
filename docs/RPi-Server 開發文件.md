# RPi Server 開發文件

> 對應主文件：[智能家庭控制系統開發文件](./智能家庭控制系統開發文件%20(System%20Development%20Document).md)
> 對應原始碼目錄：`Intelligent-home-RPi-server/`
> 文件版本：v0.1（2026-05-27）

---

## 1. 文件目的與範圍

### 1.1 目的

本文件描述「智能家庭控制系統」中 **Raspberry Pi 中央伺服器** 的軟體設計、模組結構、資料模型、開發流程與測試策略，作為實作前的對齊基礎。

### 1.2 範圍

**包含：**
- RPi 端 Python 伺服器的所有軟體層（BLE 管理、Web 應用、資料持久化）
- 與 STM32 之間的 BLE 通訊邏輯
- 本地 Web UI（含使用者帳號系統）

**不包含：**
- STM32 韌體實作細節（另見 STM32 規格文件）
- 具體 GATT Service / Characteristic UUID 對照表（屬於 STM32 規格文件範疇；本文件中 UUID 視為可由 Web UI 動態設定的字串）

---

## 2. 技術選型總覽

| 類別 | 選擇 | 理由 |
| --- | --- | --- |
| 語言 | Python 3.11+ | RPi OS Bookworm 內建；型別註解與標準函式庫充足 |
| 套件管理 | `uv` | 快速、鎖檔可靠、單一工具完成 venv + 安裝 |
| BLE 函式庫 | `bluepy` | 依原始開發文件指定（Linux only） |
| Web 框架 | Flask | 輕量、適合本地端服務 |
| 即時通訊 | Flask-SocketIO（threading 模式） | 主動推播感測器數據；threading 模式不需 eventlet/gevent，可與 `bluepy` 的同步呼叫共存 |
| 認證 | Flask-Login + `bcrypt` | 多使用者帳號系統需求；session-based |
| 模板 | Jinja2 + HTMX + 少量 vanilla JS | 簡單、無前端 build pipeline |
| 圖表 | Chart.js（CDN） | 歷史趨勢圖；零安裝 |
| 資料庫 | SQLite（標準函式庫 `sqlite3`） | 單檔、零設定、學生專題規模足夠 |
| 測試 | `pytest` + `pytest-mock` | 非 BLE 層完整測試；BLE 層用 mock |
| 程式碼品質 | `ruff`（lint + format）、`mypy`（型別） | uv 可直接管理 |
| 部署 | `systemd` service | RPi 開機自動啟動 |

---

## 3. 系統架構

### 3.1 分層

```
┌─────────────────────────────────────────────────┐
│  Web Browser (HTMX + Chart.js + Socket.IO JS)   │
└──────────────┬──────────────────────────────────┘
               │ HTTP / WebSocket
┌──────────────▼──────────────────────────────────┐
│  Flask Application (web/)                       │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ HTTP routes  │  │ SocketIO event handlers  │ │
│  │ (auth, CRUD) │  │ (live readings, status)  │ │
│  └──────┬───────┘  └──────────┬───────────────┘ │
└─────────┼──────────────────────┼─────────────────┘
          │                      │
          │     ┌────────────────┘
          ▼     ▼
┌─────────────────────────────────────────────────┐
│  Service Layer (services/)                      │
│  device_service, channel_service, user_service  │
└────────┬────────────────────────┬───────────────┘
         │                        │
         ▼                        ▼
┌────────────────────┐   ┌────────────────────────┐
│  Repository (db/)  │   │  BLE Manager (ble/)    │
│  - users           │   │  - Scanner             │
│  - devices         │   │  - Connection pool     │
│  - channels        │   │  - Notify handler      │
│  - readings        │   │  - Rate limiter (1Hz)  │
└────────┬───────────┘   └──────────┬─────────────┘
         │                          │
         ▼                          ▼
   ┌─────────┐                ┌──────────┐
   │ SQLite  │                │  bluepy  │
   └─────────┘                └──────────┘
```

### 3.2 執行緒模型

```
Main thread        ── Flask + SocketIO (threading mode)
BLE worker thread  ── bluepy 同步 API；負責掃描、連線、Notify
                      事件透過 thread-safe queue → SocketIO emit
Rate-limit timer   ── 對每個監控頻道，每秒最多寫入 1 筆 reading
```

關鍵設計：
- **`bluepy` 是同步且阻塞的**，必須跑在獨立執行緒，避免阻塞 Flask 主迴圈。
- BLE 執行緒透過 `queue.Queue` 或 callback 將事件交給 Flask-SocketIO，後者用 `socketio.emit(..., namespace=..., to=...)` 推播給瀏覽器。
- 寫入資料庫的呼叫從 BLE 執行緒發出時，`sqlite3` 連線必須是該執行緒專屬（`check_same_thread=True` 預設），所以 Repository 層需提供 thread-local connection 或每次寫入建立短連線。

### 3.3 模組職責

| 模組 | 職責 | 不負責 |
| --- | --- | --- |
| `ble/` | BLE 掃描、連線、Read/Write/Notify、限頻 | 業務邏輯、HTTP |
| `db/` | SQLite schema、CRUD、遷移 | 業務規則 |
| `services/` | 業務邏輯（綁定 BLE 與 DB）、權限檢查 | HTTP、BLE 低階呼叫 |
| `web/` | HTTP 路由、模板渲染、SocketIO event | DB 操作（透過 service） |
| `core/` | 設定載入、logging、共用型別 | — |

---

## 4. 模組詳細設計

### 4.1 BLE 層 (`ble/`)

#### 4.1.1 介面抽象

為了讓非 BLE 層在 Windows 上可用 pytest 測試，BLE 層提供抽象介面：

```python
# ble/interface.py
class BLEManager(Protocol):
    def start_scan(self, duration_s: float) -> list[DiscoveredDevice]: ...
    def connect(self, address: str) -> ConnectionHandle: ...
    def disconnect(self, handle: ConnectionHandle) -> None: ...
    def read(self, handle: ConnectionHandle, char_uuid: str) -> bytes: ...
    def write(self, handle: ConnectionHandle, char_uuid: str, data: bytes) -> None: ...
    def subscribe(
        self,
        handle: ConnectionHandle,
        char_uuid: str,
        callback: Callable[[bytes], None],
    ) -> None: ...
```

實作：
- `BluepyManager`（生產）：用 `bluepy.btle.Scanner` / `Peripheral`
- `MockBLEManager`（測試）：可預設掃描結果、模擬 Notify 流量

#### 4.1.2 掃描

- 由 Web UI 觸發「新增裝置」時，呼叫 `start_scan(duration_s=5.0)`
- 回傳發現的 `(address, name, rssi)` 清單給前端
- 使用者選擇後，address 寫入 `devices` 表

#### 4.1.3 連線管理

- 啟動時自動連線資料庫中所有已知裝置
- 維護 `address → Peripheral` 對照表
- 偵測斷線後指數退避重試（1s → 2s → 4s → … → 上限 60s）
- 連線狀態變化透過 SocketIO 通知前端

#### 4.1.4 Notify 限頻

每個監控頻道對應一個 `RateLimiter`：
```python
class RateLimiter:
    def __init__(self, min_interval_s: float = 1.0):
        self._last_emit: dict[str, float] = {}

    def should_emit(self, key: str) -> bool:
        now = time.monotonic()
        if now - self._last_emit.get(key, 0) >= self._min_interval_s:
            self._last_emit[key] = now
            return True
        return False
```

收到 Notify 時：
1. 解析資料（依該頻道的 `data_format` 設定）
2. 透過 SocketIO 推播給所有訂閱該頻道的瀏覽器（即時 UI，不受限頻影響）
3. 若 `should_emit(channel_id)` 為 True，寫入 `readings` 資料表

> **設計選擇**：UI 推播不限頻、DB 寫入限頻。這樣 UI 看到的是即時數據，但歷史資料量受控。

#### 4.1.5 資料解析

每個頻道在 `channels` 表中記錄 `data_format`，例如：
- `"uint8"`：開關狀態（0 / 1）
- `"float32_le"`：小端 32-bit float（溫度）
- `"uint16_le"`：小端 unsigned 16-bit（濕度 ×100）

解析器：`ble/parser.py`，輸入 `bytes` 與 `data_format`，輸出 `float | int | bool`。

### 4.2 資料持久化層 (`db/`)

#### 4.2.1 連線管理

- 單一 SQLite 檔案：`data/home.db`
- 啟動時自動執行 `schema.sql` 建表（IF NOT EXISTS）
- 每個執行緒自己的 connection（thread-local）
- 啟用 `PRAGMA foreign_keys = ON` 與 `journal_mode = WAL`（讀寫並行）

#### 4.2.2 Repository 介面

每個資料表一個 repository 模組（純函式 + connection 注入）：
```python
# db/devices.py
def create(conn, *, address: str, name: str, owner_user_id: int) -> int: ...
def get_by_address(conn, address: str) -> Device | None: ...
def list_all(conn) -> list[Device]: ...
def delete(conn, device_id: int) -> None: ...
```

不使用 ORM；用 `sqlite3.Row` + dataclass 對應。

### 4.3 Web 層 (`web/`)

#### 4.3.1 應用結構

```python
# web/__init__.py
def create_app(config) -> tuple[Flask, SocketIO]:
    app = Flask(__name__)
    app.config.update(config)
    LoginManager().init_app(app)
    socketio = SocketIO(app, async_mode="threading")
    register_blueprints(app)
    register_socketio_handlers(socketio)
    return app, socketio
```

使用 **application factory** 模式，方便測試時注入不同設定（例如記憶體資料庫）。

#### 4.3.2 認證

- `auth/` blueprint：register、login、logout、change_password
- 密碼用 `bcrypt` hash（cost factor 12）
- session cookie + CSRF token（Flask-WTF）
- 所有業務頁面與 API 都需 `@login_required`

> **權限模型**：本版採用 flat 模型——所有登入使用者皆可管理所有裝置。未來若需 role-based，在 `users.role` 加欄位即可。

#### 4.3.3 路由設計（HTTP）

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/` | Dashboard：所有裝置與頻道概覽 |
| GET / POST | `/auth/login` | 登入 |
| GET / POST | `/auth/register` | 註冊 |
| POST | `/auth/logout` | 登出 |
| GET | `/devices` | 裝置列表頁 |
| GET | `/devices/scan` | 觸發掃描，回傳 HTMX partial |
| POST | `/devices` | 新增裝置（form：address, name） |
| GET | `/devices/<id>` | 單一裝置頁，含其頻道 |
| DELETE | `/devices/<id>` | 刪除裝置 |
| POST | `/devices/<id>/channels` | 新增頻道（form：name, type, char_uuid, data_format） |
| DELETE | `/channels/<id>` | 刪除頻道 |
| POST | `/channels/<id>/write` | 控制型頻道下達指令 |
| GET | `/channels/<id>/history` | 歷史資料（JSON for Chart.js） |

> 所有 GET 同時支援完整 HTML 頁面（無 `HX-Request` header）與 partial 片段（有 `HX-Request`）。

#### 4.3.4 WebSocket 事件

| 方向 | Event 名稱 | Payload |
| --- | --- | --- |
| Server → Client | `reading` | `{channel_id, value, timestamp}` |
| Server → Client | `device_status` | `{device_id, status: "connected" \| "disconnected" \| "reconnecting"}` |
| Client → Server | `subscribe_channel` | `{channel_id}` |
| Client → Server | `unsubscribe_channel` | `{channel_id}` |

使用 SocketIO **room**：每個頻道一個 room，前端 subscribe 時 `join_room(f"channel:{id}")`，BLE 層收到 Notify 時 `emit("reading", ..., room=f"channel:{id}")`。

### 4.4 Service 層 (`services/`)

業務邏輯，例如：
- `device_service.add_device(user, address, name)`：驗證 address 格式 → 寫 DB → 觸發 BLE 連線
- `channel_service.write_command(user, channel_id, raw_value)`：權限檢查 → 編碼 → BLE write
- `channel_service.handle_notify(channel_id, raw_bytes)`：解析 → emit WS → 限頻寫 DB

Service 不直接呼叫 `bluepy`；透過 `BLEManager` 介面，方便測試。

---

## 5. 資料模型 (SQLite Schema)

```sql
-- users
CREATE TABLE users (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    username      TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- devices
CREATE TABLE devices (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    address        TEXT NOT NULL UNIQUE,         -- BLE MAC
    name           TEXT NOT NULL,
    owner_user_id  INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- channels
CREATE TABLE channels (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    device_id    INTEGER NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    name         TEXT NOT NULL,
    type         TEXT NOT NULL CHECK (type IN ('controller', 'display')),
    char_uuid    TEXT NOT NULL,                  -- BLE Characteristic UUID
    data_format  TEXT NOT NULL,                  -- e.g., 'float32_le', 'uint8'
    unit         TEXT,                           -- e.g., '°C', '%'
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(device_id, name)
);

-- readings (歷史數據，僅 display 頻道)
CREATE TABLE readings (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    channel_id  INTEGER NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    value       REAL NOT NULL,
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_readings_channel_time ON readings(channel_id, recorded_at);
```

**保留策略**：本版不自動清理舊資料；若 `readings` 表過大，可由使用者手動觸發或之後加排程清理。

---

## 6. 專案目錄結構

```
Intelligent-home-RPi-server/
├── pyproject.toml            # uv 管理
├── uv.lock
├── README.md
├── .python-version           # 3.11
├── .gitignore
├── data/                     # SQLite 檔案放這裡（不入 git）
│   └── .gitkeep
├── scripts/
│   ├── run_dev.sh            # 開發啟動
│   └── install_systemd.sh    # 部署
├── src/
│   └── home_server/
│       ├── __init__.py
│       ├── __main__.py       # python -m home_server
│       ├── config.py
│       ├── core/
│       │   ├── logging.py
│       │   └── types.py
│       ├── ble/
│       │   ├── __init__.py
│       │   ├── interface.py        # BLEManager Protocol
│       │   ├── bluepy_manager.py   # 生產實作
│       │   ├── mock_manager.py     # 測試實作
│       │   ├── parser.py           # bytes ↔ value
│       │   └── rate_limiter.py
│       ├── db/
│       │   ├── __init__.py
│       │   ├── schema.sql
│       │   ├── connection.py
│       │   ├── users.py
│       │   ├── devices.py
│       │   ├── channels.py
│       │   └── readings.py
│       ├── services/
│       │   ├── __init__.py
│       │   ├── device_service.py
│       │   ├── channel_service.py
│       │   └── user_service.py
│       └── web/
│           ├── __init__.py         # create_app()
│           ├── auth.py             # auth blueprint
│           ├── devices.py          # device routes
│           ├── channels.py         # channel routes
│           ├── socketio_events.py
│           ├── templates/
│           │   ├── base.html
│           │   ├── auth/
│           │   ├── devices/
│           │   └── channels/
│           └── static/
│               ├── css/
│               └── js/
└── tests/
    ├── conftest.py
    ├── test_ble_parser.py
    ├── test_rate_limiter.py
    ├── test_db_repositories.py
    ├── test_services.py
    └── test_web_routes.py
```

---

## 7. 開發環境設定

### 7.1 在 RPi 上的初次設定

```bash
# 1. 安裝系統相依（bluepy 需要）
sudo apt update
sudo apt install -y libglib2.0-dev libbluetooth-dev pkg-config

# 2. 安裝 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 3. 取得程式碼
git clone <repo> && cd Intelligent-home/Intelligent-home-RPi-server

# 4. 同步依賴
uv sync

# 5. 給予 BLE 權限（避免每次 sudo）
sudo setcap cap_net_raw+e $(uv run python -c "import bluepy; import os; print(os.path.join(os.path.dirname(bluepy.__file__), 'bluepy-helper'))")
sudo setcap cap_net_admin+e $(uv run python -c "import bluepy; import os; print(os.path.join(os.path.dirname(bluepy.__file__), 'bluepy-helper'))")

# 6. 啟動
uv run python -m home_server
```

### 7.2 在 Windows 上開發（非 BLE 部分）

```powershell
# 1. 安裝 uv
winget install --id=astral-sh.uv -e

# 2. 同步依賴（bluepy 列為 platform-specific，Windows 上會跳過）
uv sync

# 3. 跑測試
uv run pytest
```

`pyproject.toml` 中將 `bluepy` 標記為 Linux only：
```toml
[project]
dependencies = [
    "flask",
    "flask-socketio",
    "flask-login",
    "flask-wtf",
    "bcrypt",
    "bluepy ; sys_platform == 'linux'",
]
```

---

## 8. 測試策略

### 8.1 分層測試

| 層 | 策略 | 平台 |
| --- | --- | --- |
| `ble/parser.py`, `rate_limiter.py` | 純單元測試 | Windows + RPi |
| `db/` | 用 `:memory:` SQLite，每個測試一個 fixture connection | Windows + RPi |
| `services/` | 注入 `MockBLEManager` + 記憶體 DB | Windows + RPi |
| `web/` | Flask test client + 注入記憶體 DB + MockBLEManager | Windows + RPi |
| `ble/bluepy_manager.py` | 手動整合測試（搭配實體 STM32） | 僅 RPi |

### 8.2 conftest.py 提供的 fixtures

```python
@pytest.fixture
def db_conn(): ...                 # 記憶體 SQLite，已建表
@pytest.fixture
def mock_ble(): ...                # MockBLEManager 實例
@pytest.fixture
def app(db_conn, mock_ble): ...    # Flask app
@pytest.fixture
def client(app): ...               # Flask test client
@pytest.fixture
def logged_in_client(client): ...  # 已登入的 client
```

### 8.3 覆蓋率目標

- 非 BLE 層（parser、limiter、db、services、web）：≥ 80%
- `bluepy_manager.py`：不要求單元覆蓋，靠整合測試
- CI（若之後加上）：在 ubuntu-latest 跑 pytest

---

## 9. 設定與環境變數

`config.py` 由環境變數讀取，提供 dev / prod 兩種 profile：

| 變數 | 預設 | 說明 |
| --- | --- | --- |
| `HOME_SERVER_DB_PATH` | `./data/home.db` | SQLite 檔案路徑 |
| `HOME_SERVER_SECRET_KEY` | （prod 必須設） | Flask session secret |
| `HOME_SERVER_HOST` | `0.0.0.0` | Web bind |
| `HOME_SERVER_PORT` | `5000` | Web port |
| `HOME_SERVER_LOG_LEVEL` | `INFO` | logging level |
| `HOME_SERVER_BLE_SCAN_DURATION` | `5.0` | 預設掃描秒數 |
| `HOME_SERVER_READING_MIN_INTERVAL` | `1.0` | DB 寫入限頻（秒） |
| `HOME_SERVER_BLE_BACKEND` | `auto` | BLE backend 選用：`auto`/`mock`/`bluepy` |

---

## 10. 部署

### 10.1 systemd unit

`/etc/systemd/system/home-server.service`：
```ini
[Unit]
Description=Intelligent Home RPi Server
After=network.target bluetooth.service
Requires=bluetooth.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/Intelligent-home/Intelligent-home-RPi-server
Environment=HOME_SERVER_SECRET_KEY=<change-me>
ExecStart=/home/pi/.local/bin/uv run python -m home_server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 10.2 啟用

```bash
sudo systemctl daemon-reload
sudo systemctl enable home-server
sudo systemctl start home-server
sudo systemctl status home-server
```

---

## 11. 對應主文件的開發階段

| 主文件 Phase | RPi Server 工作項目 | 驗證標準 |
| --- | --- | --- |
| Phase 1 | 建立 uv 專案、`pyproject.toml`、目錄結構、`schema.sql`、core logging | `uv run python -m home_server` 可啟動空殼，DB 自動建立 |
| Phase 2 | `ble/bluepy_manager.py`、`MockBLEManager`、parser、rate_limiter | pytest 全綠；RPi 上能掃描並 print 出 STM32 廣播 |
| Phase 3 | 完整 web 層（auth、devices、channels、history）、SocketIO event、Chart.js 整合 | 從瀏覽器完成「新增裝置 → 新增頻道 → 即時看到數據 → 歷史趨勢圖」全流程 |
| Phase 4 | 多節點佈署測試、systemd 部署、效能與穩定性調整 | RPi 同時連 ≥2 個 STM32 連續運作 ≥1 小時不掉線 |

### 11.1 目前實作進度（截至 2026-05-31）

Phase 3 依子階段拆分實作，目前進度：

| 子階段 | 內容 | 狀態 |
| --- | --- | --- |
| 1 | 骨架、`config`、`schema.sql`、logging | ✅ 完成 |
| 2 | BLE 層（interface / parser / rate_limiter / mock / bluepy manager） | ✅ 完成 |
| 3a | DB repository 層（users / devices / channels / readings） | ✅ 完成 |
| 3b | Service 層（`user_service` / `device_service` / `channel_service`） | ✅ 完成 |
| 3c | 認證 blueprint（application factory、Flask-Login、CSRF、register/login/logout、陽春模板） | ✅ 完成 |
| 3d | Device / Channel CRUD blueprint（裝置/頻道 列表・新增・詳情・刪除，純 POST 表單） | ✅ 完成 |
| 3e-1 | SocketIO 即時推播接線、`/devices/scan`・`/channels/<id>/write`・`/channels/<id>/history`、Mock 背景產數、HTMX/Chart.js 前端 | ✅ 完成 |
| 3e-2 | 真實 `BluepyManager` 依平台選用、斷線自動重連、`device_status` 事件 | ✅ 完成（mock 單元測試；真硬體待 RPi 冷煙） |
| 4 | 多節點整合測試、systemd 部署 | ⬜ 未開始 |

測試現況：167 unit tests passing、`ruff check` 與 `mypy src`（strict）全綠。

> 3e-1 的設計取捨：SocketIO 採模組級單例 + `init_app`，`create_app` 維持回傳 `Flask`（不動既有測試）；notify→DB→emit 主流程在 3b 已實作，3e-1 以 `services/ble_runtime.py` 接線（連線、訂閱 display 頻道、worker 執行緒短連線寫入）。副作用（背景執行緒、連線）只在 `__main__` 觸發，測試不起執行緒。控制寫入沿用既有 `write_command` 依頻道 `data_format` 編碼（非單一 byte）。前端資產與 Bootstrap 一致採 vendoring。SocketIO 連線要求已登入 session（`connect` 拒絕匿名）；沿用 flat 權限模型，不做 per-user owner 檢查。真實 bluepy 與自動重連留待 3e-2 於 RPi 驗證。

> 3e-2 的設計取捨：BLE backend 由 `HOME_SERVER_BLE_BACKEND=auto|mock|bluepy` 選用（`ble/selection.py` 純函式 + bluepy 延遲匯入；auto 在非 Linux 或 bluepy 不可用時 fallback mock）。重連擴充 `BleRuntime`（輪詢 `is_connected` + 指數退避 1→60s + `_bring_up_device` 重新訂閱 display 頻道），偵測機制對 mock/bluepy 一致、`_monitor_tick(now)` 可同步測試；副作用（執行緒、連線）只在 `__main__`。`device_status` 經 SocketIO 廣播、僅狀態轉變才 emit，前端每裝置徽章（server-render 初值 + 即時更新）。真 STM32 + bluepy 整合為 RPi 手動冷煙，不在自動化範圍。

> 3d 的設計取捨：服務接線採 application factory + `app.extensions` 注入（`web/services.py` 提供型別化存取器），開發機注入 `MockBLEManager`；真實 `BluepyManager` 依平台選用、自動重連與 notify subscribe 仍留待 3e。新增/刪除採純 POST 表單（`POST .../delete`）以換取零 JS；HTMX 漸進增強留待 3e。權限沿用 flat 模型（`owner_user_id = current_user`、列表用 `list_all`）。

> 3b 的設計取捨：service 層維持純業務邏輯，BLE 操作同步呼叫 `BLEManager` 介面（序列化由 `BluepyManager` 內部的 per-peripheral worker thread 處理）；§4.1.3 的自動重連背景迴圈與 §4.1.4 notify subscribe 的執行緒 wiring 留待 3e 與 SocketIO 一併接線。

---

## 12. 已知風險與待決議事項

| 項目 | 風險 / 待決 | 暫定處理 |
| --- | --- | --- |
| `bluepy` 在新版 BlueZ 偶有相容問題 | 連線不穩 | Phase 2 早期驗證；若不穩可改用 `bleak` |
| `bluepy` 同步呼叫長時間佔用執行緒 | 阻塞其他 BLE 操作 | 序列化所有 BLE 操作至單一 worker thread |
| SQLite 寫入並發 | 多執行緒同時寫 | 啟用 WAL；BLE 寫入序列化由 worker thread 統一處理 |
| 多使用者權限 | 目前所有人可控制所有裝置 | 本版 flat 模型；之後需要時加 `users.role` |
| 密碼遺失 | 無找回機制 | 本版提供 CLI `home-server reset-password <user>` 指令（之後實作） |
| HTTPS | 本地網路明文傳輸 | 本版 HTTP only；之後可加 self-signed cert 或反向代理 |

---

## 13. 下一步行動

Phase 1–3d 已完成（見 §11.1）。後續：

1. **Phase 3e**：Flask-SocketIO 即時推播，接線 notify subscribe 與自動重連，整合 Jinja2/HTMX 前端與 Chart.js；補上 3e 範圍的 `/devices/scan`、`/channels/<id>/write`、`/channels/<id>/history`
2. **Phase 4**：多節點整合測試與 `systemd` 部署

---

*文件結束。如需修改任何技術選型或設計決策，請在開始實作前提出討論。*
