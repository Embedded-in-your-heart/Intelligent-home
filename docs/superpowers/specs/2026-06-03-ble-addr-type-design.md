# BLE 位址型別（public/random）傳遞 設計文件

> 對應原始碼目錄：`Intelligent-home-RPi-server/src/home_server/`
> 日期：2026-06-03
> 前置：Phase 3e-2（BLE backend 平台選用 + 自動重連 + device_status）已完成並合併。
> 緣由：實機 `task run` 連線 STM32（位址 `f6:8c:f2:d3:ea:e7`）持續逾時，bluepy 回報 `Failed to connect to peripheral ..., addr type: public`。該位址最高位元組 `0xf6 = 0b1111_0110`，最高兩 bit `11` → 屬 **static random** 位址，但 `bluepy_manager.py` 連線時寫死 `addrType=ADDR_TYPE_PUBLIC`，故連不上。

---

## 1. 目標與範圍

讓 BLE 連線使用**正確的位址型別**（public/random），取代目前寫死的 `ADDR_TYPE_PUBLIC`。位址型別自 BLE scan 權威帶出並隨 `Device` 持久化；無 scan 來源時（既有 DB 舊裝置、手動輸入 MAC）由位址 bits 推斷。

**驗收標準**：
- 開發機（macOS）：`uv run pytest / ruff check / mypy src`（strict）全綠。`infer_addr_type`、`add_device`、`devices.create/from_row`、migration 皆有單元測試；`MockBLEManager` 記錄 `connect` 收到的 `addr_type` 供斷言。
- RPi（手動冷煙，不自動化）：既有 `f6:…` 裝置經 migration 後 `addr_type=random`；重啟 `task run` 後該裝置可連上（或 log 顯示真實 BTLE 原因而非寫死 public 的失敗）。

### 1.1 範圍內

- 新增純函式單元 `ble/address.py`：`infer_addr_type(address) -> "public" | "random"` + 常數。
- `DiscoveredDevice` 新增 `addr_type`，`BluepyManager.start_scan` 自 `ScanEntry.addrType` 帶出。
- `devices` 資料表新增 `addr_type` 欄；`connection.initialize()` 一次性守衛式 migration（補欄 + 用推斷 backfill 既有列）。
- `Device` / `devices.create` / `device_service.add_device` / `BLEManager.connect` 簽名 / `BluepyManager` worker / `ble_runtime` 連線呼叫點 / scan 模板 + 表單，逐一接線。
- **Scan 結果過濾**（追加需求）：`DeviceService.scan()` 只回傳廣播名稱以 `HOME-` 開頭的裝置。

### 1.2 範圍外（YAGNI）

- migration 框架（僅此一段 inline 守衛式 migration）。
- 手動「Add device」表單的型別選單、逐裝置編輯 `addr_type` 的 UI。
- resolvable / non-resolvable private 位址（MSB 兩 bit `01` / `00`）的細分推斷——一律落到 public，由 scan 權威值覆蓋。
- `bluepy_manager` 的 macOS 單元測試（模組頂部 `if sys.platform != "linux": raise ImportError`，無法匯入）。

---

## 2. 現況盤點

讀過程式碼後確認資料流：

`bluepy_manager.start_scan`（`DiscoveredDevice(address, name, rssi)`，**丟棄 `e.addrType`**）→ `_scan_results.html` 每列隱藏表單 POST `address`+`name` → `web/devices.py` POST `/devices` → `device_service.add_device(conn, owner_user_id, address, name)` → `devices.create(address, name, owner)` + `self._ble.connect(address)` → `BluepyManager.connect` → `_PeripheralWorker(address)` → `run()`：`btle.Peripheral(self.address, addrType=btle.ADDR_TYPE_PUBLIC)`（**寫死**）。`ble_runtime`（`activate` 初次 bring-up 與重連 monitor）亦以 `connect(device.address)` 呼叫，僅有位址、無型別。

`devices` 資料表（`schema.sql`）：`id / address(UNIQUE) / name / owner_user_id / created_at`，以 `CREATE TABLE IF NOT EXISTS` 建立，**無 migration 機制**（既有 DB 不會自動長出新欄）。RPi 上 `data/home.db` 已存在 `f6:…` 裝置。

**缺口**：scan 未帶位址型別、DB 未存、`connect` 寫死 public。

---

## 3. 關鍵設計決策

### 3.1 位址型別表示法：正規化字串 `"public"` / `"random"`

刻意對齊 bluepy 的 `ScanEntry.addrType` 與 `btle.ADDR_TYPE_PUBLIC/RANDOM`，故 DB 取出的字串可直接餵 `btle.Peripheral(addr, addrType=...)`，無需轉換層。

### 3.2 推斷規則：`ble/address.py`（不依賴 bluepy，可在 macOS 測試）

```python
ADDR_TYPE_PUBLIC = "public"
ADDR_TYPE_RANDOM = "random"

def infer_addr_type(address: str) -> str:
    """Infer BLE address type from the most-significant octet.
    Static random addresses have the top two bits of the MSB set to 0b11.
    """
    msb = int(address.split(":")[0], 16)
    return ADDR_TYPE_RANDOM if (msb >> 6) == 0b11 else ADDR_TYPE_PUBLIC
```

可靠辨識 static random（如 `f6:…`）；private 隨機（`01`/`00`）落到 public，由 scan 權威值覆蓋。

### 3.3 一次性守衛式 migration（`connection.initialize()`）

`executescript(schema.sql)` 後：

```python
cols = {r["name"] for r in conn.execute("PRAGMA table_info(devices)")}
if "addr_type" not in cols:
    conn.execute(
        "ALTER TABLE devices ADD COLUMN addr_type TEXT NOT NULL "
        "DEFAULT 'public' CHECK (addr_type IN ('public','random'))"
    )
    for row in conn.execute("SELECT id, address FROM devices").fetchall():
        conn.execute(
            "UPDATE devices SET addr_type = ? WHERE id = ?",
            (infer_addr_type(row["address"]), row["id"]),
        )
```

只在「欄位不存在」分支執行一次（升級後首次啟動），之後永不再動，不覆蓋 scan/使用者設定值。既有 `f6:…` → `random`，自我修復。

> 註：SQLite 的 `ADD COLUMN` 支援僅參照該欄自身的 `CHECK`，故 migrated 欄位與 `schema.sql` 新建欄位帶**相同** `CHECK (addr_type IN ('public','random'))`，兩路徑一致。

### 3.4 `connect` 簽名：`connect(address, addr_type="public")`

預設 `"public"` 保持不關心型別的呼叫端/測試免改。`BluepyManager` 將其帶入 `_PeripheralWorker(address, addr_type)` → `run()` 用 `addrType=self._addr_type`。`MockBLEManager.connect` 接受並**記錄**（供測試斷言）。

### 3.5 無 scan 來源時推斷的落點：`device_service.add_device`

`add_device(..., addr_type: str | None = None)`；`None` → `infer_addr_type(address)`。手動表單（無型別）走推斷；scan 路徑帶權威值穿透。

### 3.6 Scan 名稱過濾（追加需求）：`DeviceService.scan()`

只回傳廣播名稱以 `HOME-` 開頭的裝置：

```python
_REQUIRED_NAME_PREFIX = "HOME-"

def scan(self, duration_s: float) -> list[DiscoveredDevice]:
    found = self._ble.start_scan(duration_s)
    return [
        d for d in found
        if d.name is not None and d.name.startswith(_REQUIRED_NAME_PREFIX)
    ]
```

落在服務層（非 BLE manager），故 mock 可測；無名稱（`None`）排除；**大小寫敏感**（`home-` 不符）。前綴以模組常數表示，不做成設定（YAGNI）。

---

## 4. 改動清單（逐檔）

| 檔案 | 改動 |
|------|------|
| `ble/address.py`（新） | 常數 + `infer_addr_type` |
| `ble/interface.py` | `DiscoveredDevice` 加 `addr_type: str = "public"`；`BLEManager.connect` 簽名加 `addr_type="public"` |
| `ble/bluepy_manager.py` | `start_scan` 帶 `e.addrType`（缺值推斷）；`connect(address, addr_type)`；`_PeripheralWorker.__init__(address, addr_type)` 存型別；`run()` 用 `addrType=self._addr_type` |
| `ble/mock_manager.py` | `connect(address, addr_type="public")` 接受並記錄（如 `connect_calls`） |
| `db/schema.sql` | `devices` 加 `addr_type TEXT NOT NULL DEFAULT 'public' CHECK(...)` |
| `db/connection.py` | `initialize()` 守衛式 migration + 推斷 backfill |
| `db/devices.py` | `Device` 加 `addr_type`；`from_row` 讀；`create(..., addr_type)` 寫 |
| `services/device_service.py` | `add_device(..., addr_type=None)` 推斷 + 傳給 `create` 與 `connect`；`scan()` 過濾 `HOME-` 名稱前綴（§3.6） |
| `services/ble_runtime.py` | 所有 `connect(device.address)` → `connect(device.address, device.addr_type)` |
| `web/devices.py` | `AddDeviceForm` 加 hidden `addr_type`；POST handler 傳 `addr_type=form.addr_type.data or None` |
| `web/templates/devices/_scan_results.html` | 每列加 `<input type="hidden" name="addr_type" value="{{ d.addr_type }}">` |

---

## 5. 測試計畫

- `infer_addr_type`：table-driven（public OUI、`f6:…` random、MSB 邊界 `c0`/`bf`/`00`/`ff`）。
- `db.devices`：`create` + `from_row` round-trip `addr_type`。
- migration：以舊 schema（無 `addr_type` 欄）建 DB 並插入 `f6:…` 列 → `initialize()` 後欄位存在且該列 `addr_type=random`；重複呼叫 `initialize()` 不覆寫。
- `device_service.add_device`：未給 → 推斷；給定 → 穿透；斷言 `MockBLEManager` 記錄的 `connect` `addr_type` 正確。
- `device_service.scan`：只回傳 `HOME-` 前綴裝置；排除非 `HOME-`、無名稱、大小寫不符；既有 scan 測試（`test_device_service` / `test_web_devices`）更新為 `HOME-` 名稱。
- `ruff` / `mypy --strict` 全綠。
- **不在本機測**：`_PeripheralWorker` 的 `addrType` 接線（bluepy 僅 Linux）——靠檢視 + RPi 上機冷煙。

---

## 6. 部署備註

改動全在 submodule `Intelligent-home-RPi-server`。完成後 commit/push submodule，RPi `git pull` 並**重啟 `task run`**（無 auto-reload）。migration 於首次啟動自動執行；既有 `f6:…` 裝置自我修復為 `random`。
