# BLE 掃描／維護鏈重構設計

- 日期：2026-06-03
- 範圍：`Intelligent-home-RPi-server`（submodule）
- 類型：重構（behavior-preserving 為主）
- 基線：`ruff check` 乾淨、`mypy src`（strict）無誤、`pytest` 190 passed

## 1. 動機與原則

使用者要求重構 server codebase，重點放在 **BLE 連線的「掃描」與「維護」** 兩條鏈。

經完整閱讀，這個 codebase **已相當乾淨**（分層清楚、190 測試全綠、ruff + mypy strict 全綠、依賴注入良好），**不需要大翻修**。因此本次重構嚴守專案 `CLAUDE.md`（Simplicity First / Surgical Changes /「不修沒壞的東西」）與全域重構規範（「一個 code smell 一個 PR」、「沒有測試不重構」、小步提交）：

- 只做「明確有價值且風險可控」的項目。
- 行為盡量不變；唯二的例外是 demo 假資料（#1，無測試覆蓋）與新增可設定項（#2，預設值維持現行為）。
- 每一步先確保測試覆蓋，再改；改完即跑 `ruff + mypy + pytest`，必須維持全綠。

## 2. 範圍（In Scope）

| # | 項目 | 類型 | 風險 |
|---|------|------|------|
| 1 | 修正 demo 掃描裝置名稱與 `HOME-` 過濾器衝突 | bug fix | 低 |
| 2 | 掃描名稱前綴改為可設定（Config） | feature-ish（預設不變行為） | 低 |
| 3 | 抽出 `PeriodicWorker` 消除背景輪詢執行緒重複 | DRY 重構 | 低 |
| 4 | `bluepy_manager.py` 拆成 worker / facade 兩模組 | 結構搬移（行為不變） | 低（Linux-only，無法在 Mac 實機驗） |
| 5 | connect() 契約整頓（**B1：零行為改變**） | 文件＋具名化重構 | 低 |
| 6 | `README.md` 專案狀態過時更新 | 文件 | 無 |
| 7 | `channel_service.py` docstring 過時更新 | 文件 | 無 |

## 3. 範圍外（Out of Scope）

- **B2 connect() 拆分契約**（`connect()` 同步誠實＋新增 `ensure_connecting()`＋訂閱綁到 connected transition）：架構更乾淨但有真實行為改變，只能 mock 測試、需在 RPi 實機驗證，**本輪不做**，留待願意實機測試時單獨一輪。
- `db/` `web/` `services/` 其餘各層的更深掃查：風格一致、無明顯 smell，不動。
- 前端／templates、部署腳本、整合測試（Phase 4）。

## 4. 各項詳細設計

### #1 Demo 掃描裝置與 `HOME-` 過濾器衝突（真 bug）

**現況**：`__main__.py` 為 mock 後端種了 demo `scan_results`，名稱為 `"Demo Sensor"` / `"Demo Lamp"`；但 `DeviceService.scan` 只放行名稱以 `HOME-` 開頭者（`device_service.py:_REQUIRED_NAME_PREFIX`）。結果 dev「Scan」按鈕**掃不出任何東西**，與該行「so the Scan button shows output」的註解直接矛盾。

**修法**：把 demo 名稱改成通過前綴的形式（例如 `"HOME-Demo Sensor"` / `"HOME-Demo Lamp"`）。`DiscoveredDevice.addr_type` 維持預設 `"public"` 即可。

### #2 掃描名稱前綴可設定

**現況**：`_REQUIRED_NAME_PREFIX = "HOME-"` 寫死在 `device_service` 模組。

**設計**：
- `Config` 新增欄位 `scan_name_prefix: str`，由環境變數 `HOME_SERVER_SCAN_NAME_PREFIX` 讀取，預設 `"HOME-"`。
- `DeviceService.__init__` 新增參數 `scan_name_prefix: str = "HOME-"`（保留預設以維持相容）；`scan()` 使用注入值。
- `create_app`（`web/__init__.py`）建構 `DeviceService(ble, scan_name_prefix=config.scan_name_prefix)`。
- 語意：空字串 `""` ⇒ 不依前綴過濾（仍保留 `name is not None` 條件，即「所有具名裝置」）。
- `.env.example` 補上該變數說明。

### #3 抽出 `PeriodicWorker`（DRY）

**現況**：以下兩處是同一套「daemon thread + stop Event + `while not stop.wait(interval): tick()` + `join(timeout=2)`」樣板：
- `BleRuntime.monitor_start/monitor_stop`（`services/ble_runtime.py`）
- `MockBLEManager.start/stop`（`ble/mock_manager.py`）

**設計**：新增 `core/periodic.py`：

```python
class PeriodicWorker:
    """Run `tick` on a daemon thread every `interval_s` until stopped.

    Exceptions from `tick` are logged and the loop continues, so one bad
    tick never kills the worker. start()/stop() are idempotent.
    """
    def __init__(self, tick: Callable[[], None], *, interval_s: float, name: str) -> None: ...
    def start(self) -> None: ...      # no-op if already running
    def stop(self) -> None: ...       # set event, join(timeout=2), reset
    def is_running(self) -> bool: ...
```

- `BleRuntime`：以 `PeriodicWorker(lambda: self._monitor_tick(time.monotonic()), interval_s=..., name="ble-monitor")` 取代手寫執行緒；`_monitor_tick` 維持可注入 `now` 不變（測試仍直接呼叫它，不碰執行緒）。
- `MockBLEManager`：保留**對外 API 不變**（`start(feed, interval_s)` / `stop()`），內部改用 `PeriodicWorker` 包裝 `lambda: self._tick(feed)`。
- **行為微調（更穩健）**：`PeriodicWorker` 會 catch + log tick 例外後續跑。`BleRuntime` 原本就這麼做（保持一致）；`MockBLEManager._run` 原本不 catch，改後 feed 例外不再殺死執行緒——屬安全的健壯化，於 spec 記錄。
- **測試影響**：`test_ble_runtime_reconnect.test_monitor_start_stop_lifecycle` 與 mock 相關測試若直接戳 `_monitor_thread` / `_producer_thread` 內部屬性，改用新結構（例如 `is_running()`）斷言生命週期。實作時逐一檢查並更新。

### #4 `bluepy_manager.py` 拆 worker / facade

**現況**：單檔 310 行含兩個責任——per-peripheral worker（`_Cmd` / `_NotifyDelegate` / `_PeripheralWorker`）與對外 facade（`BluepyManager`）。

**設計**（純結構搬移，行為不變，維持扁平佈局）：
- 新增 `ble/bluepy_worker.py`：放 `_Cmd`、`_NotifyDelegate`、`_PeripheralWorker`、`_NOTIFY_POLL_INTERVAL_S`，以及 `btle` import 與 `if sys.platform != "linux": raise ImportError` 平台守門。
- `ble/bluepy_manager.py`：保留 `BluepyManager`，`from .bluepy_worker import _PeripheralWorker`（此 import 會連帶觸發平台守門，維持 `selection._load_bluepy()` 所依賴的「非 Linux 匯入即 `ImportError`」契約）。
- **不可破壞的契約**：`selection.py` 以 `try/except ImportError` 偵測 bluepy 可用性；拆檔後在非 Linux 仍須擲 `ImportError`。
- **驗證限制**：Linux-only，無法在本機（macOS）實機跑，但 `mypy src` 在 Mac 上仍會靜態分析此檔（目前 30 檔全綠），拆檔後須維持 mypy 全綠；其餘以人工審閱保證為等價搬移。

### #5 connect() 契約整頓 —— B1（零行為改變）

**背景**：`BluepyManager.connect()` 對「新 worker」會 `wait_until_connected()` 阻塞並於失敗時拋例外；對「既有 alive worker」則直接回傳、不保證 `is_connected()`。此不對稱**是 load-bearing 且被測試鎖定的**：
- 新 worker 的阻塞讓 `activate()`/`add_device` 連上後才能 `subscribe` display 頻道。
- 既有 worker 直接回傳是 powered-off 反 flap 修正（commit `3a0985f`），由 `test_powered_off_device_does_not_flap_connected` 等以 `_LinkNeverUpBLE`（connect 假成功、`is_connected()` 永遠 False）鎖定。

因此「讓 connect() 一律拋例外」會回退 flap 修正並弄壞 2 個 regression 測試；「讓 connect() 一律 fire-and-forget」會弄壞 `activate()` 初始訂閱。兩者皆不可取。

**B1 設計（只把契約講清楚，不改行為）**：
- `BLEManager.connect`（`ble/interface.py`）docstring 改為誠實契約：
  > Begin/ensure a GATT connection and return the handle. A newly created connection is awaited up to the manager's timeout (raising on failure); a connection already being managed returns immediately, so the link may not be up yet — `is_connected()` is the authoritative check.
- `BluepyManager.connect` 抽出具名輔助 `_ensure_worker(address, addr_type) -> tuple[_PeripheralWorker, bool]`（回傳 `(worker, created)`，於鎖內檢查既有 alive worker、否則建立並啟動）：
  ```python
  def connect(self, address, addr_type="public"):
      worker, created = self._ensure_worker(address, addr_type)
      if created:
          worker.wait_until_connected(timeout=self._op_timeout_s)  # initial subscribe relies on this
      return address  # reused worker: caller polls is_connected()
  ```
  邏輯與現行**完全等價**，只是把兩條路講清楚。
- `BleRuntime._bring_up_device` 內那段「workaround」長註解改寫為引用契約：`is_connected()` 是 `BLEManager.connect` 契約下的權威來源；檢查保留。
- 結果：行為不變，190 測試維持全綠。

### #6 `README.md` 專案狀態過時

「🚧 進行中／未完成」段把 Phase 3d（device/channel CRUD blueprint）與 Phase 3e（SocketIO + 前端）依 git 紀錄移至「✅ 已完成」，避免誤導。

### #7 `channel_service.py` docstring 過時

檔頭 docstring「the BLE worker-thread wiring that calls it lives in a later phase」已不成立（`services/ble_runtime.py` 已接好 notify wiring），更新為現況描述。

## 5. 測試策略

- **基線**：`pytest` 190 passed、`ruff check` 乾淨、`mypy src` strict 無誤。
- **行為保持項（#4、#5、#6、#7）**：不新增行為，靠既有測試全綠證明等價。
- **新增程式碼補測試**：
  - `PeriodicWorker`（#3）：start/stop 生命週期、start 冪等、stop 冪等、tick 例外後續跑、stop 後執行緒結束。
  - `DeviceService.scan` 前綴可設定（#2）：預設 `HOME-` 行為不變、自訂前綴、空字串=不過濾、`name is None` 仍排除。
- **需更新的既有測試**：#3 觸及執行緒內部屬性的生命週期測試改用新 API 斷言。
- **流程**：每完成一項即跑 `task ci`（lint + typecheck + test），維持全綠後才進下一項；小步提交（`refactor:` / `fix:` / `docs:` 分型）。

## 6. 風險與緩解

| 風險 | 緩解 |
|------|------|
| #4 bluepy 拆檔在 macOS 無法實機驗 | 純等價搬移＋維持 `mypy src` 全綠＋人工審閱＋保留非 Linux `ImportError` 契約 |
| #3 測試耦合內部執行緒屬性 | 將受影響測試一併更新為以 `PeriodicWorker` 對外行為斷言 |
| 重構擴散（scope creep） | 嚴守上表範圍；發現其他 smell 只記錄不順手改 |

## 7. 提交與分支

- 程式碼變更於 submodule `Intelligent-home-RPi-server`，開 feature 分支、小步提交。
- 本設計文件提交於 parent repo `docs/superpowers/specs/`。
