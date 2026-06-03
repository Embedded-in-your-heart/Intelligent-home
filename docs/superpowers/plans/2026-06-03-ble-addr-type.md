# BLE 位址型別（public/random）傳遞 實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓 BLE 連線使用正確的位址型別（public/random），取代 `bluepy_manager` 寫死的 `ADDR_TYPE_PUBLIC`；型別自 scan 權威帶出並隨 `Device` 持久化，無 scan 來源時由位址 bits 推斷。

**Architecture:** 新增純函式 `ble/address.py`（`infer_addr_type`，不依賴 bluepy）為單一真相來源；`DiscoveredDevice`/`Device`/`devices.create`/`add_device`/`connect()` 逐層接線；`connection.initialize()` 加一次性守衛式 migration 補欄並推斷 backfill 既有資料。

**Tech Stack:** Python 3.12、Flask、SQLite（sqlite3）、bluepy（僅 Linux）、pytest、ruff、mypy --strict。

---

## 執行環境

- **所有檔案路徑與指令皆相對於 submodule 根目錄 `Intelligent-home-RPi-server/`**；請 `cd Intelligent-home-RPi-server` 後執行。
- 測試/檢查指令：`uv run pytest`、`uv run ruff check src tests`、`uv run mypy src`。
- **Task 7（bluepy 接線）在 macOS 無法單元測試**（`bluepy_manager.py` 頂部 `if sys.platform != "linux": raise ImportError`）；僅檢視 + RPi 上機冷煙驗證。
- commit 都在 submodule 內。完成全部後再 push submodule、RPi `git pull` 並重啟 `task run`（無 auto-reload）。

---

## 檔案結構

| 檔案 | 責任 | 動作 |
|------|------|------|
| `src/home_server/ble/address.py` | 位址型別常數 + 推斷（單一真相） | 新增 |
| `tests/test_ble_address.py` | `infer_addr_type` 單元測試 | 新增 |
| `src/home_server/db/schema.sql` | `devices` 加 `addr_type` 欄（新 DB） | 修改 |
| `src/home_server/db/devices.py` | `Device.addr_type` + `from_row` + `create` | 修改 |
| `tests/test_db_devices.py` | `create`/round-trip addr_type | 修改 |
| `src/home_server/db/connection.py` | `apply_migrations()` + `initialize()` 呼叫 | 修改 |
| `tests/test_db_migration.py` | 舊 DB migration + 推斷 backfill + 冪等 | 新增 |
| `src/home_server/ble/interface.py` | `DiscoveredDevice.addr_type` + `connect` 簽名 | 修改 |
| `src/home_server/ble/mock_manager.py` | `connect(addr_type)` + `connect_calls` 記錄 | 修改 |
| `tests/test_mock_manager.py` | mock connect 記錄 addr_type | 修改 |
| `src/home_server/services/device_service.py` | `add_device(addr_type=None)` 推斷 + 穿透 | 修改 |
| `tests/test_device_service.py` | 推斷 / 顯式 / 穿透至 connect | 修改 |
| `src/home_server/services/ble_runtime.py` | `_bring_up_device` 傳 `device.addr_type` | 修改 |
| `tests/test_ble_runtime.py` | activate 用 device.addr_type 連線 | 修改 |
| `src/home_server/ble/bluepy_manager.py` | start_scan 帶 addrType；connect + worker | 修改（檢視驗證） |
| `src/home_server/web/devices.py` | `AddDeviceForm` hidden `addr_type` + handler | 修改 |
| `src/home_server/web/templates/devices/_scan_results.html` | 每列 hidden `addr_type` | 修改 |
| `tests/test_web_devices.py` | 表單帶/不帶 addr_type；scan 模板含 hidden | 修改 |

---

### Task 1: `ble/address.py` — 位址型別推斷（純函式）

**Files:**
- Create: `src/home_server/ble/address.py`
- Test: `tests/test_ble_address.py`

- [ ] **Step 1: 寫 failing 測試**

`tests/test_ble_address.py`：
```python
import pytest

from home_server.ble.address import (
    ADDR_TYPE_PUBLIC,
    ADDR_TYPE_RANDOM,
    infer_addr_type,
)


@pytest.mark.parametrize(
    "address,expected",
    [
        ("f6:8c:f2:d3:ea:e7", ADDR_TYPE_RANDOM),  # 0xf6 = 0b11110110 -> top2=11
        ("c0:00:00:00:00:00", ADDR_TYPE_RANDOM),  # 0xc0 = 0b11000000 -> top2=11
        ("ff:ff:ff:ff:ff:ff", ADDR_TYPE_RANDOM),  # 0xff -> top2=11
        ("bf:00:00:00:00:00", ADDR_TYPE_PUBLIC),  # 0xbf = 0b10111111 -> top2=10
        ("aa:bb:cc:dd:ee:ff", ADDR_TYPE_PUBLIC),  # 0xaa = 0b10101010 -> top2=10
        ("00:11:22:33:44:55", ADDR_TYPE_PUBLIC),  # 0x00 -> top2=00
    ],
)
def test_infer_addr_type(address: str, expected: str) -> None:
    assert infer_addr_type(address) == expected
```

- [ ] **Step 2: 跑測試確認 fail**

Run: `uv run pytest tests/test_ble_address.py -v`
Expected: FAIL（`ModuleNotFoundError: home_server.ble.address`）

- [ ] **Step 3: 實作**

`src/home_server/ble/address.py`：
```python
"""BLE address-type helpers (backend-agnostic, importable anywhere).

Kept free of bluepy imports so it works on macOS/dev and in tests. The string
values match bluepy's ``btle.ADDR_TYPE_*`` constants, so a stored value can be
passed straight to ``btle.Peripheral(addr, addrType=...)``.
"""

from __future__ import annotations

ADDR_TYPE_PUBLIC = "public"
ADDR_TYPE_RANDOM = "random"


def infer_addr_type(address: str) -> str:
    """Infer the BLE address type from the most-significant octet.

    BLE static random addresses have the top two bits of the most-significant
    octet set to 0b11. Everything else (public IEEE addresses, and the less
    common resolvable/non-resolvable private forms) is reported as public; the
    scan path supplies the authoritative type when it is available.
    """
    msb = int(address.split(":")[0], 16)
    return ADDR_TYPE_RANDOM if (msb >> 6) == 0b11 else ADDR_TYPE_PUBLIC
```

- [ ] **Step 4: 跑測試確認 pass**

Run: `uv run pytest tests/test_ble_address.py -v`
Expected: PASS（6 cases）

- [ ] **Step 5: Commit**

```bash
git add src/home_server/ble/address.py tests/test_ble_address.py
git commit -m "feat(ble): add infer_addr_type address-type helper"
```

---

### Task 2: `devices` 資料表 — `addr_type` 欄 + `Device` + `create`

**Files:**
- Modify: `src/home_server/db/schema.sql`（`devices` 表）
- Modify: `src/home_server/db/devices.py`（`Device`、`from_row`、`create`）
- Test: `tests/test_db_devices.py`

- [ ] **Step 1: 寫 failing 測試**

於 `tests/test_db_devices.py` 末端新增：
```python
def test_create_stores_addr_type(db_conn, alice_id) -> None:
    did = devices.create(
        db_conn,
        address="f6:8c:f2:d3:ea:e7",
        name="stm",
        owner_user_id=alice_id,
        addr_type="random",
    )
    d = devices.get_by_id(db_conn, did)
    assert d is not None
    assert d.addr_type == "random"


def test_create_defaults_addr_type_public(db_conn, alice_id) -> None:
    did = devices.create(
        db_conn, address="aa:bb", name="d", owner_user_id=alice_id
    )
    d = devices.get_by_id(db_conn, did)
    assert d is not None
    assert d.addr_type == "public"
```

- [ ] **Step 2: 跑測試確認 fail**

Run: `uv run pytest tests/test_db_devices.py -v`
Expected: FAIL（`create()` 無 `addr_type` 參數 / `Device` 無 `addr_type`）

- [ ] **Step 3: 改 schema.sql**

`src/home_server/db/schema.sql` 的 `devices` 表改為（在 `owner_user_id` 與 `created_at` 之間插入 `addr_type`）：
```sql
CREATE TABLE IF NOT EXISTS devices (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    address        TEXT NOT NULL UNIQUE,
    name           TEXT NOT NULL,
    owner_user_id  INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    addr_type      TEXT NOT NULL DEFAULT 'public' CHECK (addr_type IN ('public', 'random')),
    created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

- [ ] **Step 4: 改 `Device` 與 `create`**

`src/home_server/db/devices.py`：`Device` 在 `address` 後加欄位、`from_row` 讀取、`create` 加參數。

`Device` dataclass：
```python
@dataclass(frozen=True)
class Device:
    id: int
    address: str
    addr_type: str
    name: str
    owner_user_id: int
    created_at: str

    @classmethod
    def from_row(cls, row: sqlite3.Row) -> Device:
        return cls(
            id=row["id"],
            address=row["address"],
            addr_type=row["addr_type"],
            name=row["name"],
            owner_user_id=row["owner_user_id"],
            created_at=row["created_at"],
        )
```

`create`：
```python
def create(
    conn: sqlite3.Connection,
    *,
    address: str,
    name: str,
    owner_user_id: int,
    addr_type: str = "public",
) -> int:
    try:
        cur = conn.execute(
            "INSERT INTO devices (address, name, owner_user_id, addr_type) "
            "VALUES (?, ?, ?, ?)",
            (address, name, owner_user_id, addr_type),
        )
    except sqlite3.IntegrityError as e:
        msg = str(e).upper()
        if "UNIQUE" in msg:
            raise DuplicateAddressError(f"address already exists: {address}") from e
        raise
    device_id = cur.lastrowid
    assert device_id is not None
    return device_id
```

- [ ] **Step 5: 跑測試確認 pass**

Run: `uv run pytest tests/test_db_devices.py -v`
Expected: PASS（含既有測試；`from_row` 仍涵蓋所有欄位）

- [ ] **Step 6: Commit**

```bash
git add src/home_server/db/schema.sql src/home_server/db/devices.py tests/test_db_devices.py
git commit -m "feat(db): persist addr_type on devices"
```

---

### Task 3: `connection.initialize()` 一次性守衛式 migration

**Files:**
- Modify: `src/home_server/db/connection.py`
- Test: `tests/test_db_migration.py`（新增）

- [ ] **Step 1: 寫 failing 測試**

`tests/test_db_migration.py`：
```python
import sqlite3

from home_server.db import connection

# Pre-addr_type devices schema (before this migration existed).
_OLD_SCHEMA = """
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE devices (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    address TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    owner_user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
"""


def _old_db() -> sqlite3.Connection:
    conn = sqlite3.connect(":memory:", isolation_level=None)
    conn.row_factory = sqlite3.Row
    conn.executescript(_OLD_SCHEMA)
    conn.execute(
        "INSERT INTO users (id, username, password_hash) VALUES (1, 'u', 'h')"
    )
    return conn


def test_migration_adds_column_and_backfills_by_inference() -> None:
    conn = _old_db()
    conn.execute(
        "INSERT INTO devices (address, name, owner_user_id) "
        "VALUES ('f6:8c:f2:d3:ea:e7', 'stm', 1)"
    )
    conn.execute(
        "INSERT INTO devices (address, name, owner_user_id) "
        "VALUES ('aa:bb:cc:dd:ee:ff', 'pub', 1)"
    )
    connection.apply_migrations(conn)
    rows = {
        r["address"]: r["addr_type"]
        for r in conn.execute("SELECT address, addr_type FROM devices")
    }
    assert rows["f6:8c:f2:d3:ea:e7"] == "random"
    assert rows["aa:bb:cc:dd:ee:ff"] == "public"
    conn.close()


def test_migration_idempotent_preserves_existing_values() -> None:
    conn = _old_db()
    conn.execute(
        "INSERT INTO devices (address, name, owner_user_id) "
        "VALUES ('f6:8c:f2:d3:ea:e7', 'stm', 1)"
    )
    connection.apply_migrations(conn)
    # An authoritative correction (e.g. scan reported public) is later stored.
    conn.execute(
        "UPDATE devices SET addr_type = 'public' "
        "WHERE address = 'f6:8c:f2:d3:ea:e7'"
    )
    connection.apply_migrations(conn)  # second run must not touch the column
    row = conn.execute(
        "SELECT addr_type FROM devices WHERE address = 'f6:8c:f2:d3:ea:e7'"
    ).fetchone()
    assert row["addr_type"] == "public"
    conn.close()
```

- [ ] **Step 2: 跑測試確認 fail**

Run: `uv run pytest tests/test_db_migration.py -v`
Expected: FAIL（`connection.apply_migrations` 不存在）

- [ ] **Step 3: 實作 `apply_migrations` 並於 `initialize()` 呼叫**

`src/home_server/db/connection.py`：頂部 import，新增函式，並在 `initialize()` 內 `executescript` 後呼叫。
```python
from home_server.ble.address import infer_addr_type


def apply_migrations(conn: sqlite3.Connection) -> None:
    """Idempotent in-place migrations for databases created before a column
    existed. Each block runs only when its target column is missing, so values
    written later (by scan or the user) are never overwritten.
    """
    cols = {r["name"] for r in conn.execute("PRAGMA table_info(devices)")}
    if "addr_type" not in cols:
        conn.execute(
            "ALTER TABLE devices ADD COLUMN addr_type TEXT NOT NULL "
            "DEFAULT 'public' CHECK (addr_type IN ('public', 'random'))"
        )
        for row in conn.execute("SELECT id, address FROM devices").fetchall():
            conn.execute(
                "UPDATE devices SET addr_type = ? WHERE id = ?",
                (infer_addr_type(row["address"]), row["id"]),
            )
```

`initialize()` 改為：
```python
def initialize(db_path: Path) -> None:
    """Create parent dir, run schema.sql, apply migrations. Safe to repeat."""
    db_path.parent.mkdir(parents=True, exist_ok=True)
    conn = connect(db_path)
    try:
        conn.executescript(_SCHEMA_PATH.read_text(encoding="utf-8"))
        apply_migrations(conn)
    finally:
        conn.close()
```

- [ ] **Step 4: 跑測試確認 pass（含既有全測試不回歸）**

Run: `uv run pytest tests/test_db_migration.py -v`
Expected: PASS（2 cases）
Run: `uv run pytest -q`
Expected: 全綠（`initialize()` 對新 schema DB 中 `apply_migrations` 為 no-op）

- [ ] **Step 5: Commit**

```bash
git add src/home_server/db/connection.py tests/test_db_migration.py
git commit -m "feat(db): one-shot addr_type migration with inferred backfill"
```

---

### Task 4: `interface` 簽名 + `MockBLEManager` 記錄 addr_type

**Files:**
- Modify: `src/home_server/ble/interface.py`（`DiscoveredDevice`、`BLEManager.connect`）
- Modify: `src/home_server/ble/mock_manager.py`（`connect` + `connect_calls`）
- Test: `tests/test_mock_manager.py`

- [ ] **Step 1: 寫 failing 測試**

於 `tests/test_mock_manager.py` 末端新增：
```python
def test_connect_records_address_and_addr_type() -> None:
    mock = MockBLEManager()
    mock.connect("f6:8c:f2:d3:ea:e7", "random")
    assert mock.connect_calls == [("f6:8c:f2:d3:ea:e7", "random")]
    assert mock.is_connected("f6:8c:f2:d3:ea:e7")


def test_connect_defaults_addr_type_public() -> None:
    mock = MockBLEManager()
    mock.connect("aa:bb:cc:dd:ee:ff")
    assert mock.connect_calls == [("aa:bb:cc:dd:ee:ff", "public")]
```

（`tests/test_mock_manager.py` 已 import `MockBLEManager`；若無請補 `from home_server.ble.mock_manager import MockBLEManager`。）

- [ ] **Step 2: 跑測試確認 fail**

Run: `uv run pytest tests/test_mock_manager.py -v`
Expected: FAIL（`connect()` 不收第二參數 / 無 `connect_calls`）

- [ ] **Step 3: 改 interface**

`src/home_server/ble/interface.py`：
`DiscoveredDevice` 加欄位：
```python
@dataclass(frozen=True)
class DiscoveredDevice:
    address: str
    name: str | None
    rssi: int
    addr_type: str = "public"
```
`BLEManager.connect` 簽名：
```python
    def connect(self, address: str, addr_type: str = "public") -> ConnectionHandle:
        """Establish a GATT connection. Raises if the peer is unreachable."""
        ...
```

- [ ] **Step 4: 改 mock_manager**

`src/home_server/ble/mock_manager.py`：在 dataclass 欄位加（與 `scan_calls` 同段）：
```python
    connect_calls: list[tuple[str, str]] = field(default_factory=list)
```
`connect` 改為：
```python
    def connect(
        self, address: str, addr_type: str = "public"
    ) -> ConnectionHandle:
        self.connect_calls.append((address, addr_type))
        if address in self.fail_connect_for:
            raise MockBLEError(f"Mocked connect failure for {address}")
        self._connected.add(address)
        return address
```

- [ ] **Step 5: 跑測試確認 pass**

Run: `uv run pytest tests/test_mock_manager.py tests/test_device_service.py tests/test_ble_runtime.py -v`
Expected: PASS（既有測試不受影響；`connect_calls` 為新欄位）

- [ ] **Step 6: Commit**

```bash
git add src/home_server/ble/interface.py src/home_server/ble/mock_manager.py tests/test_mock_manager.py
git commit -m "feat(ble): thread addr_type through BLEManager.connect and mock"
```

---

### Task 5: `device_service.add_device` 推斷 + 穿透

**Files:**
- Modify: `src/home_server/services/device_service.py`
- Test: `tests/test_device_service.py`

- [ ] **Step 1: 寫 failing 測試**

於 `tests/test_device_service.py` 末端新增：
```python
RANDOM_ADDR = "f6:8c:f2:d3:ea:e7"


def test_add_device_infers_random_addr_type(db_conn, owner) -> None:
    mock = MockBLEManager()
    svc = DeviceService(mock)
    d = svc.add_device(
        db_conn, owner_user_id=owner, address=RANDOM_ADDR, name="stm"
    )
    assert d.addr_type == "random"
    assert mock.connect_calls == [(RANDOM_ADDR, "random")]


def test_add_device_uses_explicit_addr_type(db_conn, owner) -> None:
    mock = MockBLEManager()
    svc = DeviceService(mock)
    d = svc.add_device(
        db_conn,
        owner_user_id=owner,
        address=RANDOM_ADDR,
        name="stm",
        addr_type="public",
    )
    assert d.addr_type == "public"
    assert mock.connect_calls == [(RANDOM_ADDR, "public")]
```

- [ ] **Step 2: 跑測試確認 fail**

Run: `uv run pytest tests/test_device_service.py -v`
Expected: FAIL（`add_device` 無 `addr_type` 參數；連線未帶型別）

- [ ] **Step 3: 改 add_device**

`src/home_server/services/device_service.py`：頂部 import `from home_server.ble.address import infer_addr_type`，`add_device` 改為：
```python
    def add_device(
        self,
        conn: sqlite3.Connection,
        *,
        owner_user_id: int,
        address: str,
        name: str,
        addr_type: str | None = None,
    ) -> Device:
        if not _MAC_RE.match(address):
            raise InvalidAddressError(f"invalid BLE address: {address!r}")
        resolved_addr_type = addr_type or infer_addr_type(address)
        device_id = devices.create(
            conn,
            address=address,
            name=name,
            owner_user_id=owner_user_id,
            addr_type=resolved_addr_type,
        )
        # Best-effort initial connect; keep the device on failure for later retry.
        try:
            self._ble.connect(address, resolved_addr_type)
        except Exception:
            log.warning(
                "Initial connect to %s failed; device kept for later retry",
                address,
                exc_info=True,
            )
        device = devices.get_by_id(conn, device_id)
        assert device is not None
        return device
```

- [ ] **Step 4: 跑測試確認 pass**

Run: `uv run pytest tests/test_device_service.py -v`
Expected: PASS（新 2 案 + 既有案；既有 `ADDR="AA:BB:..."` 推斷為 public，不影響 `is_connected` 斷言）

- [ ] **Step 5: Commit**

```bash
git add src/home_server/services/device_service.py tests/test_device_service.py
git commit -m "feat(device): infer addr_type on add and pass it to connect"
```

---

### Task 6: `ble_runtime._bring_up_device` 傳 `device.addr_type`

**Files:**
- Modify: `src/home_server/services/ble_runtime.py`（`_bring_up_device`）
- Test: `tests/test_ble_runtime.py`

- [ ] **Step 1: 寫 failing 測試**

於 `tests/test_ble_runtime.py` 末端新增：
```python
def test_activate_connects_with_device_addr_type(db_path: Path) -> None:
    ble = MockBLEManager()
    rt, _ = _runtime(db_path, ble)
    rt.activate()
    # db_path fixture created the device without addr_type -> defaults public.
    assert ble.connect_calls == [(ADDR, "public")]
```

- [ ] **Step 2: 跑測試確認 fail**

Run: `uv run pytest tests/test_ble_runtime.py::test_activate_connects_with_device_addr_type -v`
Expected: FAIL（`connect_calls` 內為 `(ADDR,)` 形狀不符 / 目前只傳 address → 實際記錄為 `(ADDR, "public")`？）

> 註：Task 4 後 `_bring_up_device` 仍呼叫 `connect(device.address)`，mock 以預設 `addr_type="public"` 記錄 `(ADDR, "public")` —— 此測試可能已 PASS。若已 PASS，仍保留此測試作為迴歸防護，直接進 Step 3 確認程式碼顯式傳遞、再跑一次。**目的是讓連線顯式採用 `device.addr_type` 而非倚賴預設值。**

- [ ] **Step 3: 改 _bring_up_device**

`src/home_server/services/ble_runtime.py`：
```python
    def _bring_up_device(self, conn: sqlite3.Connection, device: Device) -> bool:
        """Connect one device and subscribe its display channels. Returns success."""
        try:
            self._ble.connect(device.address, device.addr_type)
        except Exception:
            log.warning("connect to %s failed", device.address, exc_info=True)
            return False
        for channel in channels.list_by_device(conn, device.id):
            if channel.type == "display":
                self.subscribe_channel(device.address, channel)
        return True
```

- [ ] **Step 4: 跑測試確認 pass**

Run: `uv run pytest tests/test_ble_runtime.py tests/test_ble_runtime_reconnect.py -v`
Expected: PASS（`activate` 與重連 monitor 皆經 `_bring_up_device`，連線帶 `device.addr_type`）

- [ ] **Step 5: Commit**

```bash
git add src/home_server/services/ble_runtime.py tests/test_ble_runtime.py
git commit -m "feat(runtime): connect using each device's stored addr_type"
```

---

### Task 7: `bluepy_manager` 接線（檢視驗證，macOS 不測）

**Files:**
- Modify: `src/home_server/ble/bluepy_manager.py`（`start_scan`、`connect`、`_PeripheralWorker`）

> 本檔在 macOS 匯入即 `ImportError`，無單元測試；以檢視 + RPi 上機冷煙驗證。改動須與 interface 一致。

- [ ] **Step 1: import 推斷工具**

頂部既有 import 區（`from .interface import ...` 附近）加：
```python
from .address import infer_addr_type
```

- [ ] **Step 2: `start_scan` 帶 addrType**

`BluepyManager.start_scan` 的迴圈內 `out.append(...)` 改為：
```python
            out.append(
                DiscoveredDevice(
                    address=e.addr,
                    name=name,
                    rssi=e.rssi,
                    addr_type=e.addrType or infer_addr_type(e.addr),
                )
            )
```
（bluepy `ScanEntry.addrType` 回傳 `"public"`/`"random"`；缺值則推斷。）

- [ ] **Step 3: `connect` 簽名 + 傳入 worker**

`BluepyManager.connect`：
```python
    def connect(self, address: str, addr_type: str = "public") -> ConnectionHandle:
        with self._lock:
            existing = self._workers.get(address)
            if existing is not None and existing.is_alive():
                return address
            worker = _PeripheralWorker(address, addr_type)
            worker.start()
            self._workers[address] = worker
        worker.wait_until_connected(timeout=self._op_timeout_s)
        return address
```

- [ ] **Step 4: `_PeripheralWorker` 存型別並用於連線**

`_PeripheralWorker.__init__` 簽名與欄位：
```python
    def __init__(self, address: str, addr_type: str = "public") -> None:
        super().__init__(name=f"ble-worker-{address}", daemon=True)
        self.address = address
        self._addr_type = addr_type
        self._queue: queue.Queue[_Cmd] = queue.Queue()
        self._peripheral: btle.Peripheral | None = None
        self._delegate = _NotifyDelegate()
        self._char_cache: dict[str, btle.Characteristic] = {}
        self._connected = threading.Event()
        # NOTE: must NOT be named ``_stop`` — that shadows threading.Thread._stop(),
        # which CPython calls internally from is_alive()/join() once the thread has
        # terminated, raising "TypeError: 'Event' object is not callable".
        self._stop_event = threading.Event()
        self._connect_future: Future[None] = Future()
```
`run()` 內建立 Peripheral 改為：
```python
            self._peripheral = btle.Peripheral(
                self.address, addrType=self._addr_type
            )
```

- [ ] **Step 5: 靜態檢查（macOS 可跑的部分）**

Run: `uv run ruff check src` 與 `uv run mypy src`
Expected: PASS（mypy 不執行 bluepy，但會型別檢查簽名一致性）

- [ ] **Step 6: Commit**

```bash
git add src/home_server/ble/bluepy_manager.py
git commit -m "feat(ble): connect bluepy peripherals with the resolved addr_type"
```

---

### Task 8: Web 層 — scan 模板 hidden 欄位 + 表單 + handler

**Files:**
- Modify: `src/home_server/web/devices.py`（`AddDeviceForm`、`list_devices`）
- Modify: `src/home_server/web/templates/devices/_scan_results.html`
- Test: `tests/test_web_devices.py`

- [ ] **Step 1: 寫 failing 測試**

於 `tests/test_web_devices.py` 末端新增：
```python
def test_add_device_stores_addr_type_from_form(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    logged_in_client.post(
        "/devices",
        data={
            "address": "f6:8c:f2:d3:ea:e7",
            "name": "STM",
            "addr_type": "random",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    conn = connection.connect(app.config["DB_PATH"])
    try:
        device = devices.get_by_address(conn, "f6:8c:f2:d3:ea:e7")
    finally:
        conn.close()
    assert device is not None
    assert device.addr_type == "random"


def test_add_device_without_addr_type_infers(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    logged_in_client.post(
        "/devices",
        data={
            "address": "f6:8c:f2:d3:ea:e7",
            "name": "STM",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    conn = connection.connect(app.config["DB_PATH"])
    try:
        device = devices.get_by_address(conn, "f6:8c:f2:d3:ea:e7")
    finally:
        conn.close()
    assert device is not None
    assert device.addr_type == "random"


def test_scan_results_include_addr_type_hidden_input(
    logged_in_client: FlaskClient, mock_ble
) -> None:
    mock_ble.scan_results = [
        DiscoveredDevice(
            address="f6:8c:f2:d3:ea:e7", name="N", rssi=-40, addr_type="random"
        )
    ]
    body = logged_in_client.get("/devices/scan").get_data(as_text=True)
    assert 'name="addr_type"' in body
    assert 'value="random"' in body
```

- [ ] **Step 2: 跑測試確認 fail**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: FAIL（表單無 `addr_type`；模板無 hidden input）

- [ ] **Step 3: 改 `web/devices.py`**

import 改為包含 `HiddenField`：
```python
from wtforms import HiddenField, StringField
```
`AddDeviceForm` 加欄位：
```python
class AddDeviceForm(FlaskForm):
    address = StringField("Address", validators=[DataRequired()])
    name = StringField("Name", validators=[DataRequired()])
    addr_type = HiddenField("Addr type")
```
`list_devices` 內呼叫 `add_device` 改為傳入 `addr_type`：
```python
            get_device_service().add_device(
                get_conn(),
                owner_user_id=int(current_user.get_id()),
                address=form.address.data,
                name=form.name.data,
                addr_type=form.addr_type.data or None,
            )
```

- [ ] **Step 4: 改 scan 結果模板**

`src/home_server/web/templates/devices/_scan_results.html` 的每列 `<form>` 內，於 `address` hidden input 後加一行：
```html
      <input type="hidden" name="address" value="{{ d.address }}">
      <input type="hidden" name="addr_type" value="{{ d.addr_type }}">
```

- [ ] **Step 5: 跑測試確認 pass**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: PASS（含新 3 案；既有手動「Add device」表單無 addr_type → handler 傳 `None` → service 推斷）

- [ ] **Step 6: Commit**

```bash
git add src/home_server/web/devices.py src/home_server/web/templates/devices/_scan_results.html tests/test_web_devices.py
git commit -m "feat(web): carry scanned addr_type through the add-device form"
```

---

### Task 9: 全量驗證（lint + typecheck + test）

**Files:** 無（驗證）

- [ ] **Step 1: 全測試**

Run: `uv run pytest -q`
Expected: 全綠

- [ ] **Step 2: Lint**

Run: `uv run ruff check src tests`
Expected: All checks passed

- [ ] **Step 3: 型別檢查**

Run: `uv run mypy src`
Expected: Success, no issues

- [ ] **Step 4: （等同 CI）**

Run: `uv run ruff check src tests && uv run mypy src && uv run pytest -q`
Expected: 全綠

---

## 部署（全部 Task 完成後）

- [ ] push submodule：`git push origin main`
- [ ] RPi：`cd ~/Intelligent-home-RPi-server && git pull --ff-only origin main`
- [ ] RPi：重啟 `task run`（Ctrl+C 後重跑，無 auto-reload）
- [ ] RPi 冷煙：既有 `f6:8c:f2:d3:ea:e7` 經 migration 為 `addr_type=random`；server 以 random 連線，連得上（或 log 顯示真實 BTLE 原因而非寫死 public 失敗）。
- [ ]（可選）父專案 bump submodule 指標並 commit。

---

## Self-Review 紀錄

- **Spec coverage**：address.py(T1)、DiscoveredDevice/connect(T4)、schema+Device+create(T2)、migration(T3)、add_device 推斷(T5)、ble_runtime(T6)、bluepy worker(T7)、web 模板/表單(T8)、測試與全綠(T9) —— 對應 spec §3–§5 全覆蓋。
- **Placeholder**：無 TBD/TODO；每個 code step 皆含完整程式碼。
- **Type 一致**：`addr_type: str` 全程字串 `"public"`/`"random"`；`connect(address, addr_type="public")` 於 interface/mock/bluepy/呼叫端一致；`Device.addr_type`/`DiscoveredDevice.addr_type`/`devices.create(addr_type=)`/`add_device(addr_type=)`/`apply_migrations` 名稱一致；`connect_calls: list[tuple[str, str]]` 一致。
