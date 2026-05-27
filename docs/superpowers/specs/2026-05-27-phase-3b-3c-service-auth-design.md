# Phase 3b / 3c 設計：Service 層與認證 Blueprint

> 日期：2026-05-27
> 對應 repo：`Intelligent-home-RPi-server/`（git submodule）
> 上游設計：[`RPi-Server 開發文件.md`](../../RPi-Server%20開發文件.md) §4.3、§4.4、§8

---

## 1. 目標與範圍

完成 RPi server 的 **Service 層（Phase 3b）** 與 **認證 Blueprint（Phase 3c）**，銜接既有的 DB repository 層（3a）與 BLE 層（2）。

### 包含
- `services/user_service.py`、`services/device_service.py`、`services/channel_service.py`
- `web/__init__.py` 的 application factory `create_app()`
- `web/auth.py`（register / login / logout）+ 最小 HTML 模板
- 對應單元測試與 `conftest.py` 共用 fixtures

### 不包含（留待後續 phase）
- **SocketIO**（3e）
- **BLE notify subscribe 的執行緒 wiring**（3e）：「worker thread 收到 notify → 開該執行緒 conn → 呼叫 `handle_notify`」這段
- **BLE 自動重連背景迴圈**（3e）：需長生命週期執行緒 + 狀態推播，本質是整合層
- **device / channel 的 CRUD blueprint**（3d）
- **前端美化、Chart.js**（3e）
- `change_password`（README 3c 未列，YAGNI）

---

## 2. 關鍵設計決策

| 決策 | 選擇 | 理由 |
|---|---|---|
| Service 外部效果 | 純業務邏輯，SocketIO/notify 用注入的 callback 抽象 | 全部可用 `MockBLEManager` + 記憶體 DB 測試，零執行緒 |
| Service API 形狀 | device/channel 為 class（建構注入協作者），方法第一參數收 `conn`；user 為純函式 | 延續 repository 的 conn 注入風格；長生命週期協作者（`ble`/`limiter`/`on_reading`）放建構子 |
| BLE 操作執行緒 | service 直接同步呼叫 `BLEManager` 介面 | `BluepyManager` 內部已把每個操作丟 per-peripheral worker + `Future` 阻塞，呼叫端執行緒安全 |
| web conn 管理 | per-request 用 `flask.g` + `teardown_appcontext` 開關 | 每個 request 在主執行緒一條專屬 conn |
| 密碼最低長度 | 8 碼 | — |
| bcrypt cost | 參數預設 12，測試傳 4 | cost 12 太慢拖累測試 |

---

## 3. Service 層詳細契約

### 3.1 `services/user_service.py`（純函式）

```python
class WeakPasswordError(ValueError): ...

_MIN_PASSWORD_LEN = 8

def hash_password(password: str, *, cost: int = 12) -> str
def register(conn, *, username: str, password: str, cost: int = 12) -> int
def authenticate(conn, *, username: str, password: str) -> User | None
```

- `register`：密碼長度 < 8 → `WeakPasswordError`；hash 後 `users.create`；重複帳號讓 `DuplicateUsernameError`（db 層）上拋。
- `authenticate`：`users.get_by_username`；找不到或 `bcrypt.checkpw` 失敗回 `None`；成功回 `User`。
- 不引入新例外給「帳密錯誤」——回 `None`，由 web 層決定訊息。

### 3.2 `services/device_service.py`（class）

```python
class InvalidAddressError(ValueError): ...

_MAC_RE = re.compile(r"^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$")

class DeviceService:
    def __init__(self, ble: BLEManager) -> None
    def scan(self, duration_s: float) -> list[DiscoveredDevice]
    def add_device(self, conn, *, owner_user_id: int, address: str, name: str) -> Device
    def remove_device(self, conn, device_id: int) -> None
    def list_devices(self, conn) -> list[Device]
```

- `add_device`：MAC 格式不符 → `InvalidAddressError`；`devices.create`（重複 → `DuplicateAddressError` 上拋）；接著 **best-effort** `ble.connect(address)`，失敗只 `log.warning`、**不 rollback**（裝置保留，留待 3e 重連）；回傳建立後的 `Device`。
- `remove_device`：`devices.get_by_id`（無 → `DeviceNotFoundError`）；若 `ble.is_connected` 則 `ble.disconnect`；`devices.delete`。
- `list_devices`：flat 權限模型，`devices.list_all`。

### 3.3 `services/channel_service.py`（class）

```python
ReadingCallback = Callable[[int, float, str], None]  # (channel_id, value, iso_utc)

class WrongChannelTypeError(ValueError): ...

class ChannelService:
    def __init__(self, ble: BLEManager, limiter: RateLimiter, on_reading: ReadingCallback) -> None
    def add_channel(self, conn, *, device_id, name, type, char_uuid, data_format, unit=None) -> Channel
    def write_command(self, conn, *, channel_id: int, raw_value: float) -> None
    def handle_notify(self, conn, *, channel_id: int, raw_bytes: bytes) -> float
    def get_history(self, conn, channel_id, *, since=None, until=None, limit=None) -> list[Reading]
    def list_by_device(self, conn, device_id: int) -> list[Channel]
```

- `add_channel`：`data_format` ∉ `parser.supported_formats()` → `UnknownFormatError`（parser 的）；type 非法 → `InvalidChannelTypeError`（db 的）；`channels.create`（重複名 → `DuplicateChannelNameError` 上拋）。
- `write_command`：`channels.get_by_id`（無 → `ChannelNotFoundError`）；`type != "controller"` → `WrongChannelTypeError`；`parser.encode(raw_value, data_format)`；查 `devices.get_by_id` 取 `address`；`ble.write(address, char_uuid, data)`。
- `handle_notify`：`parser.decode(raw_bytes, data_format)`；呼叫 `on_reading(channel_id, value, <now UTC iso>)`（**即時推播，不限頻**）；`if limiter.should_emit(str(channel_id)): readings.insert(...)`（限頻寫 DB）；回傳 decoded value。
- `get_history` / `list_by_device`：直接轉呼 `readings` / `channels` repository。

---

## 4. 認證（3c）

### 4.1 `web/__init__.py`

```python
def create_app(config: Config) -> Flask
def get_conn() -> sqlite3.Connection   # 取 g 上的 per-request conn，無則開啟
```

- 建立 `Flask`，設 `SECRET_KEY`；把 `config` 存入 `app.config` 供 `get_conn` 取 `db_path`。
- `LoginManager`：`login_view = "auth.login"`；`user_loader` 用 `users.get_by_id` → 包成 `LoginUser`。
- `CSRFProtect(app)`。
- 註冊 `auth` blueprint；保留 `/health`。
- `teardown_appcontext`：關閉 `g` 上的 conn（若有）。
- `__main__.py` 改為 `from home_server.web import create_app` 並呼叫之；移除原本內嵌的 `create_app`。

### 4.2 `web/auth.py`

```python
class LoginUser(UserMixin):  # 包 db.users.User，id 為 str(user.id)
class RegisterForm(FlaskForm): username, password, confirm
class LoginForm(FlaskForm): username, password

bp = Blueprint("auth", __name__, url_prefix="/auth")
GET/POST  /auth/register   # 成功 → 自動登入 → redirect 首頁
GET/POST  /auth/login      # 成功 → login_user → redirect next/首頁
POST      /auth/logout     # logout_user → redirect login
```

- 表單驗證失敗、帳密錯誤、弱密碼、重複帳號 → 重新渲染表單並帶 flash 訊息。
- register 呼叫 `user_service.register(get_conn(), ...)`；login 呼叫 `user_service.authenticate(...)`。

### 4.3 模板（陽春，含 CSRF）

- `templates/base.html`：HTML 骨架 + flash 訊息區塊。
- `templates/auth/login.html`、`templates/auth/register.html`：`{{ form.csrf_token }}` + 欄位 + submit。

---

## 5. 測試

| 檔案 | 平台 | 重點 |
|---|---|---|
| `tests/test_user_service.py` | all | register 成功 / 弱密碼 / 重複帳號；authenticate 成功 / 錯密碼 / 不存在（cost=4） |
| `tests/test_device_service.py` | all | scan；add（含觸發 connect、connect 失敗仍保留裝置）；無效 MAC；重複 address；remove（含 disconnect）；list |
| `tests/test_channel_service.py` | all | add（format/type 驗證）；write_command（controller OK、非 controller 拒絕、encode 正確寫入 mock.writes）；handle_notify（decode + on_reading 被呼叫 + 限頻只寫一筆）；history |
| `tests/test_web_auth.py` | all | test client：註冊→登入→存取受保護頁→登出；未登入導向 login；CSRF token 存在 |

### `conftest.py` 新增 fixtures（文件 §8.2）

```python
@pytest.fixture
def mock_ble() -> MockBLEManager
@pytest.fixture
def app(tmp_path) -> Flask          # 用 tmp_path 檔案 DB（非 :memory:，否則 per-request 每條 conn 是不同空庫），initialize 後 create_app
@pytest.fixture
def client(app)                     # app.test_client()
@pytest.fixture
def logged_in_client(client, app)   # 先註冊+登入一個使用者
```

既有 `db_conn`（`:memory:`）保留給 service / repository 測試直接傳 conn。

---

## 6. 驗收標準

- `uv run pytest` 全綠（既有 68 + 新增測試）。
- `uv run ruff check` 全綠。
- `uv run mypy src` 通過（strict）。
- `__main__.py` 仍可 `python -m home_server` 啟動，`/auth/login` 可從瀏覽器註冊 / 登入 / 登出。
