# STM32 Client 開發文件

> 對應主文件：[智能家庭控制系統開發文件](./智能家庭控制系統開發文件%20(System%20Development%20Document).md)
> 對應 RPi 端文件：[RPi Server 開發文件](./RPi-Server%20開發文件.md)
> 對應原始碼目錄：`Intelligent-home-STM32-client/`
> 文件版本：v0.1（2026-05-30）

---

## 1. 文件目的與範圍

### 1.1 目的

本文件描述「智能家庭控制系統」中 **STM32 BLE 周邊節點** 的硬體配置、韌體架構、BLE GATT 介面、FreeRTOS 任務模型與開發流程，作為實作前的對齊基礎，並提供 RPi 端在做頻道對應（device → channel → characteristic UUID）時所需的契約。

### 1.2 範圍

**包含：**
- 韌體所有層（HAL/BSP 包裝、BlueNRG-MS 應用層、感測器讀取、FreeRTOS 任務）
- STM32 對外發布的 BLE GATT 表（Service、Characteristic、UUID、封包格式）
- 與 RPi 端的資料契約（資料格式、單位、推播頻率、命令編碼）
- CubeMX 設定範圍與規則（見 [`.claude/rules/file-modification-scope.md`](../Intelligent-home-STM32-client/.claude/rules/file-modification-scope.md)）

**不包含：**
- RPi 端 BLE 掃描、連線管理、Web UI（屬 RPi 文件範疇）
- 手機端 App（本專案不開發手機 App，原 ST `BLESensor-App` 介面已停用）

---

## 2. 硬體配置

### 2.1 開發板

- **STM32 B-L475E-IOT01A**（IoT Discovery kit）
  - MCU：STM32L475VG（Cortex-M4F, 80 MHz, 1 MB Flash, 128 KB SRAM）
  - 板載感測器：
    - HTS221（溫濕度，I²C2）
    - LSM6DSL（6-axis 加速度+陀螺儀，I²C2）
    - LIS3MDL（3-axis 磁力計，I²C2，v1 不使用）
    - LPS22HB（氣壓計，I²C2，v1 不使用）
    - MP34DT01-M（PDM 數位麥克風，DFSDM）
  - 致動器：板載 LED1 (PA5)、LED2 (PB14)
  - 按鍵：User button (PC13)
- **X-NUCLEO-IDB05A2**（BlueNRG-MS BLE 模組，SPI）
  - 連接埠：**SPI3**（B-L475E-IOT01A 內接，與板載 ISM43362 WiFi 共享匯流排，目前未啟用 WiFi 不會衝突）；CS=PD13、IRQ=PE6（EXTI9_5）、RST=PA8

### 2.2 已啟用週邊（由 `.ioc` 設定，本文件僅描述需求）

| 週邊 | 用途 |
| --- | --- |
| SPI3 | BlueNRG-MS HCI 傳輸 |
| I²C2 | HTS221 + LSM6DSL |
| DFSDM1 | MP34DT01 PDM 麥克風 + DMA |
| TIM | FreeRTOS time base（避免佔用 SysTick）|
| GPIO | LED1/LED2、BLE IRQ/CS/RESET、User button |
| USART | Debug log（ST-LINK VCP）|

> ⚠️ 任何週邊新增/移除請走 `.ioc` 流程（見規則 §5），勿手改 `MX_*_Init()`。

---

## 3. 技術選型總覽

| 類別 | 選擇 | 理由 |
| --- | --- | --- |
| IDE | STM32CubeIDE 1.19.0 | 與 `.ioc` 工具鏈一致 |
| HAL/BSP | ST HAL + B-L475E-IOT01A BSP | CubeMX 原生支援，省去手刻 driver |
| RTOS | FreeRTOS（CMSIS-RTOS2 v2 wrapper） | 已啟用；多任務分離可讀性高 |
| BLE Stack | BlueNRG-MS middleware（中介韌體） | 模組為 BlueNRG-MS，配套 driver 完整 |
| 起點程式碼 | 沿用 ST `SensorDemo_BLESensor-App` | 已含 HCI、廣播、GATT 樣板，改造成本最低 |
| 編譯/燒錄 | STM32CubeIDE 內建（或 `cmake` preset） | 倉庫已含 CMakeLists.txt 與 Preset |
| Debug log | `printf` → USART → ST-LINK VCP | 學生專題夠用 |

> 不引入 ThreadX、Zephyr 或其他 OS／BLE 替代方案；不引入 mbed TLS 等加密層（v1 BLE open，安全議題見 §13）。

---

## 4. 系統架構

### 4.1 軟體分層

```
┌──────────────────────────────────────────────────────────┐
│  Application Layer (BlueNRG_MS/App/)                     │
│  ┌────────────────────┐  ┌────────────────────────────┐  │
│  │ gatt_db.c          │  │ app_bluenrg_ms.c           │  │
│  │ Service/Char 註冊  │  │ Adv、HCI event 派送        │  │
│  └────────────────────┘  └────────────────────────────┘  │
│  ┌────────────────────┐  ┌────────────────────────────┐  │
│  │ sensor_task.c      │  │ actuator.c                 │  │
│  │ HTS221/LSM6 採集   │  │ LED / ControlFlag 解碼     │  │
│  └────────────────────┘  └────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │ audio_task.c   PDM → RMS → MicLevel / LoudAlert    │  │
│  └────────────────────────────────────────────────────┘  │
└────────────────────┬─────────────────────────────────────┘
                     │ CMSIS-RTOS2 queue / mutex
┌────────────────────▼─────────────────────────────────────┐
│  RTOS Layer (Core/Src/freertos.c)                        │
│  BleTask │ SensorTask │ AudioTask                        │
└────────────────────┬─────────────────────────────────────┘
                     │ HAL / BSP / BlueNRG driver
┌────────────────────▼─────────────────────────────────────┐
│  HAL + BSP (Drivers/, Middlewares/)  *CubeMX 生成*       │
│  SPI / I²C / DFSDM / DMA / GPIO / Tick                   │
└──────────────────────────────────────────────────────────┘
```

### 4.2 模組職責

| 模組 | 職責 | 不負責 |
| --- | --- | --- |
| `app_bluenrg_ms.c` | BLE stack 初始化、廣播、HCI event loop、連線狀態 | 感測器讀取、業務邏輯 |
| `gatt_db.c` | Service/Characteristic 註冊、Read/Write callback 派送 | 解碼資料、實際 IO |
| `sensor_task.c` | I²C 讀取 HTS221、LSM6DSL，計算 magnitude / 事件，送 BLE queue | BLE 直接呼叫 |
| `audio_task.c` | DFSDM+DMA 取 PDM、RMS、門檻判斷，送 BLE queue | BLE 直接呼叫 |
| `actuator.c` | 解析寫入命令（LED1State / ControlFlag），操作 GPIO | BLE 協定 |
| `Core/Src/freertos.c` | 建立 task / queue / mutex（由 CubeMX 生成框架） | 業務邏輯 |

---

## 5. BLE GATT 設計（核心契約）

### 5.1 設計原則

- **固定 GATT 表**：所有 Characteristic UUID 在韌體燒錄時即決定，RPi 端在 Web UI 上「新增頻道」時只是選擇一個現有 UUID 並指定它的 `data_format`，**STM32 不做動態註冊**。
- **每條 Characteristic 一個語意**：避免複合封包；對齊 RPi 端「一頻道 ↔ 一 characteristic UUID」的模型。
- **資料型別限定於 RPi parser 支援的集合**（見 RPi 文件 §4.1.5）：
  - `uint8` — 開關、事件旗標
  - `uint16_le` — 0..65535 範圍的數值
  - `float32_le` — 浮點實數
- **單位寫進文件、不寫進 BLE 封包**（節省頻寬；RPi 頻道 `unit` 欄位另記）。

### 5.2 廣播設定

| 項目 | 值 | 備註 |
| --- | --- | --- |
| Local name | `HOME-XXXX` | 後綴為 BD_ADDR 末 2 byte（hex），避免多裝置撞名 |
| 廣告 interval | 100 ms – 200 ms | Phase 1 預設；省電議題見 §13 |
| 廣告 type | `ADV_IND`（可連線、可掃描）| RPi 主動連線 |
| 包含 service UUID | Home Sensor Service（見 §5.4）| 讓 RPi 掃描時可早期過濾 |
| TX power | 預設 -8 dBm | BlueNRG 預設；之後可調 |

### 5.3 連線參數（建議值）

| 項目 | 值 | 備註 |
| --- | --- | --- |
| Min connection interval | 30 ms | |
| Max connection interval | 50 ms | |
| Slave latency | 0 | Phase 1 簡單模型 |
| Supervision timeout | 5000 ms | |

### 5.4 GATT 表（v1 草案，UUID 待最終定）

**Base UUID（128-bit）**：`xxxxxxxx-8E22-4541-9D4C-21EDAE82ED19`
（前 4 byte 替換為下表的 short ID；其餘維持本專案專屬 base，避免與既有產品撞號。實作時請固定到 `gatt_db.c` 中的常數，**燒錄後不再變動**。）

#### Home Sensor Service — `1A220001-…`

| Name | UUID 短碼 | 屬性 | 資料格式 | 單位 / 範圍 | 推播時機 |
| --- | --- | --- | --- | --- | --- |
| Temperature | `1A220002` | Read + Notify | `float32_le` | °C | 每 1 s |
| Humidity | `1A220003` | Read + Notify | `float32_le` | % RH | 每 1 s |
| AccelMagnitude | `1A220004` | Read + Notify | `float32_le` | g（含重力）| 每 250 ms |
| GyroMagnitude | `1A220005` | Read + Notify | `float32_le` | dps | 每 250 ms |
| MotionAlert | `1A220006` | Read + Notify | `uint8` | `0`=normal / `1`=abnormal | 事件觸發即推 |
| MicLevel | `1A220007` | Read + Notify | `uint16_le` | 0..1023 正規化能量 | 每 200 ms |
| LoudAlert | `1A220008` | Read + Notify | `uint8` | `0`=quiet / `1`=loud | 事件觸發即推 |

#### Home Control Service — `1A22F001-…`

| Name | UUID 短碼 | 屬性 | 資料格式 | 編碼 |
| --- | --- | --- | --- | --- |
| Led1State | `1A22F002` | Read + Write | `uint8` | `0`=off / `1`=on |
| ControlFlag | `1A22F003` | Read + Write | `uint8` | 預留：低 4 bit 為通用旗標 |

> **與 RPi 端 channel 對應方式**：使用者在 RPi Web UI 新增頻道時填入 char UUID（例如 `1A220002-8E22-4541-9D4C-21EDAE82ED19`）、`type=display`、`data_format=float32_le`、`unit=°C`。STM32 端不需知道對應關係。

### 5.5 事件判斷規則（韌體內部）

| 事件 | 觸發條件 | 反跳 |
| --- | --- | --- |
| MotionAlert=1 | `AccelMagnitude > 1.8 g` 或 `GyroMagnitude > 250 dps` | 連續 ≥ 100 ms 才觸發；事件結束後鎖 1 s 不重複觸發 |
| LoudAlert=1 | `MicLevel > 800`（正規化）| 連續 ≥ 200 ms 才觸發；事件結束後鎖 1 s |

> 門檻為 v1 預估值；正式調校在 Phase 1 驗證時進行。門檻定義在韌體常數，不開放動態調整（簡化 GATT 表；之後可加 `MotionThreshold` characteristic）。

### 5.6 寫入命令處理

`gatt_db.c` 註冊 `aci_gatt_attribute_modified_event` callback：

```
on_attribute_modified(handle, length, data):
  if handle == Led1State_handle and length == 1:
      ActuatorQueue.put({cmd: LED1, value: data[0]})
  elif handle == ControlFlag_handle and length == 1:
      ActuatorQueue.put({cmd: FLAG, value: data[0]})
```

`actuator.c` 從 queue 取出後在 **BleTask context** 同步執行（LED toggle 是 GPIO write，O(µs)，不需要單獨 task）。

---

## 6. FreeRTOS Task 設計

### 6.1 Task 列表

| Task | 優先級（CMSIS-RTOS2）| Stack | 週期 | 工作內容 |
| --- | --- | --- | --- | --- |
| `BleTask` | `osPriorityHigh` | 1 KB | event-driven | `hci_user_evt_proc()` loop；處理 GATT write；從 NotifyQueue 取資料呼叫 `aci_gatt_update_char_value()` |
| `SensorTask` | `osPriorityNormal` | 768 B | 250 ms | 讀 LSM6DSL（4 Hz）；每 4 次（1 s）讀一次 HTS221；計算 magnitude/alert；送 NotifyQueue |
| `AudioTask` | `osPriorityNormal` | 1 KB | 200 ms 視窗 | DFSDM+DMA 取 PDM；計算 RMS；判斷 LoudAlert；送 NotifyQueue |

### 6.2 通訊原語

| 名稱 | 種類 | 元素型別 | 容量 | 寫入者 | 讀出者 |
| --- | --- | --- | --- | --- | --- |
| `NotifyQueue` | `osMessageQueue` | `{char_id: uint8, len: uint8, payload: uint8[4]}` | 16 | SensorTask、AudioTask | BleTask |
| `BleMutex` | `osMutex` | — | — | 保護 BlueNRG stack 呼叫 | 任何呼叫 ACI 的 task（目前只有 BleTask） |
| `ActuatorQueue` | （不需要）| | | | 寫入命令在 BleTask context 直接執行 |

> 設計取捨：所有 ACI / HCI 呼叫**只在 BleTask 內進行**，避免 mutex 競爭。其他 task 透過 queue 把要 notify 的資料丟過去，由 BleTask 序列化發送。

### 6.3 啟動順序（`main()` → `MX_FREERTOS_Init()`）

```
1. CubeMX 預設初始化（HAL、Clock、GPIO、Peripherals）
2. 建立 BleMutex、NotifyQueue
3. 建立 BleTask、SensorTask、AudioTask
4. osKernelStart()
5. BleTask 啟動後：BlueNRG reset → HCI init → GATT/GAP init →
   呼叫 add_sensor_service() / add_control_service()（gatt_db.c）→
   設定廣告 → 進入 hci_user_evt_proc loop
6. SensorTask、AudioTask 等 BleTask 完成 BLE init 後（用 osEventFlags 通知）
   開始週期工作
```

---

## 7. 感測器與致動器整合

### 7.1 HTS221（溫濕度）

- 透過 B-L475E-IOT01A BSP（`BSP_HSENSOR_*` / `BSP_TSENSOR_*`）讀取
- 採樣率：1 Hz（SensorTask 每 4 次週期讀一次）
- 失敗處理：BSP 回傳錯誤時略過該次推播；連續 5 次失敗則 `printf` 記錄

### 7.2 LSM6DSL（加速度 + 陀螺儀）

- 透過 BSP（`BSP_ACCELERO_*` / `BSP_GYRO_*`）讀取 raw 3-axis
- 採樣率：4 Hz
- `AccelMagnitude = sqrt(ax² + ay² + az²) / 1000.0` (mg → g)
- `GyroMagnitude = sqrt(gx² + gy² + gz²) / 1000.0` (mdps → dps)
- MotionAlert：見 §5.5

### 7.3 MP34DT01（麥克風）

- DFSDM + DMA 串流，每 200 ms 視窗（取決於 DFSDM 設定）
- DMA half/full callback 計算 RMS（int32 累加，避免溢位）
- 正規化：`MicLevel = clamp(round(rms / scale), 0, 1023)`（`scale` 在校正時決定）
- LoudAlert：見 §5.5

> v1 不送原始 PCM 也不做 FFT。

### 7.4 LED（致動器）

- LED1 = `BSP_LED_On(LED1) / LED_Off(LED1)`，由 `Led1State` write 觸發
- `ControlFlag` 寫入後僅儲存到全域變數並 `printf`，無實際動作（保留擴充點）

---

## 8. 專案目錄結構

> 以現有結構為主，**沿用 SensorDemo 起點**改造。
> CubeMX 控制的檔案（`Core/`、`Drivers/`、`Middlewares/`、`.ioc`）結構不能手改命名。

```
Intelligent-home-STM32-client/
├── Intelligent-home-STM32-client.ioc   # CubeMX 設定（由我修改）
├── README.md                           # 可修改
├── CMakeLists.txt / CMakePresets.json  # CubeMX/IDE 維護
├── STM32L475XX_FLASH.ld                # Linker script（不動）
├── startup_stm32l475xx.s               # Startup（不動）
├── Core/
│   ├── Inc/
│   │   ├── main.h                      # CubeMX 生成；USER 區塊可寫
│   │   ├── FreeRTOSConfig.h            # CubeMX 生成；除非調整不動
│   │   └── …
│   └── Src/
│       ├── main.c                      # 只在 USER 區塊新增
│       ├── freertos.c                  # task / queue 建立（USER 區塊）
│       ├── stm32l4xx_it.c              # ISR（USER 區塊；DMA/EXTI callback）
│       └── …
├── BlueNRG_MS/
│   ├── App/
│   │   ├── app_bluenrg_ms.c            # BLE init、廣告、event loop（改造）
│   │   ├── gatt_db.c                   # 新 Service/Char 表（重寫）
│   │   ├── gatt_db.h                   # UUID 常數、handle 宣告
│   │   ├── sensor.c                    # 舊 SensorDemo 邏輯 → 改名/分拆為：
│   │   ├── sensor_task.c               #   感測器採集（新）
│   │   ├── audio_task.c                #   麥克風（新）
│   │   ├── actuator.c                  #   LED 寫入（新）
│   │   └── (對應 .h)
│   └── Target/                         # hci_tl_interface 等（不動）
├── Drivers/                            # HAL + BSP（不動）
└── Middlewares/                        # FreeRTOS、BlueNRG-MS 中介層（不動）
```

> `.claude/rules/file-modification-scope.md` 已規範允許修改範圍：`Core/`、`BlueNRG_MS/`、`README.md`。其他須先告知。

---

## 9. 開發環境

### 9.1 工具

- **STM32CubeIDE 1.19.0**（編譯、燒錄、debug）
- **STM32CubeMX**（內建於 IDE；以 `.ioc` 管理腳位與週邊）
- **ST-LINK driver**（VCP for printf log）
- 序列埠監看：PuTTY / Tera Term（115200 8N1，VCP）

### 9.2 開發流程（每次需求進來的判斷）

```
需求屬於：
├─ 純應用邏輯（gatt_db.c 邏輯、sensor 處理、actuator 行為）
│   └─ 直接改 BlueNRG_MS/App 與 Core/USER 區塊
├─ 新增/移除 peripheral、調整 pin、調 RTOS 設定
│   └─ 走 .ioc 流程：先告知 → 我改 .ioc → 重 generate → 再寫應用層
└─ 改 HAL/BSP/Middleware 內部
    └─ 先告知，原則上不允許
```

詳見 [`Intelligent-home-STM32-client/.claude/rules/file-modification-scope.md`](../Intelligent-home-STM32-client/.claude/rules/file-modification-scope.md)。

### 9.3 燒錄與驗證

1. STM32CubeIDE → Run → 燒錄到 B-L475E-IOT01A
2. 開 VCP serial monitor，確認 boot log 與 `BleTask started` / `Adv started`
3. 用手機 BLE scanner（nRF Connect）確認可看到 `HOME-XXXX`
4. RPi 端執行 `bluetoothctl scan on` 或自家 server 的掃描 endpoint，確認可發現
5. nRF Connect 連線後檢查 GATT 表是否與 §5.4 一致

---

## 10. 測試策略

> STM32 端難以完全脫離硬體做單元測試。本專案採用「分層驗證」而非追求覆蓋率。

| 層 | 策略 | 工具 |
| --- | --- | --- |
| 感測器讀取 | 燒錄後 `printf` 顯示原始與計算後數值，肉眼比對 | VCP |
| GATT 表 | nRF Connect 直接 Read / 訂閱 Notify / Write | 手機 App |
| 事件判斷（Motion/Loud Alert）| 物理刺激（搖晃、拍手）+ printf trace | VCP |
| 整合（與 RPi）| RPi 端 `MockBLEManager` 換成 `BluepyManager`，跑端到端流程 | RPi side |
| 連線穩定性 | 連續運作 ≥ 1 小時、刻意斷電重啟 | 觀察 RPi 重連 |

> 不導入 Unity / Ceedling 等 on-target unit test，理由：純邏輯函式（如 magnitude 計算、RMS、門檻判斷）量少，且皆需要實機資料才有意義。如後續純邏輯增多，再考慮抽出至 host-side test。

---

## 11. 對應主文件的開發階段

| 主文件 Phase | STM32 工作項目 | 驗證標準 |
| --- | --- | --- |
| Phase 1 | `.ioc` 設定齊備、SensorDemo 改造起手、Sensor/AudioTask 框架、GATT 表 §5.4 完整註冊、廣告可被掃到 | nRF Connect 看得到 `HOME-XXXX`、可連線、Service/Char 與文件一致 |
| Phase 2 | Notify 推播全部上線、Write 觸發 LED1 動作、事件 Alert 可觸發 | nRF Connect 訂閱 Notify 看到資料、Write `01` → LED 亮、搖晃觸發 MotionAlert |
| Phase 3 | 與 RPi 端整合：Web UI 新增頻道後可即時看到趨勢、可下指令控制 LED | 從瀏覽器看到溫度趨勢圖、按 UI 按鈕 LED 亮滅 |
| Phase 4 | 連線穩定性調整、（選用）low power、多節點同時上線測試 | 兩台 STM32 同時連 RPi ≥ 1 小時不掉線 |

### 11.1 目前實作進度（截至 2026-05-30）

| 項目 | 狀態 |
| --- | --- |
| 板子可開機、BlueNRG_MS 起點程式碼匯入 | ✅ |
| `.ioc` 內 FreeRTOS / SPI / I²C / DFSDM 啟用 | 🔲 需確認（在 Phase 1 開頭驗證並補上） |
| GATT 表 §5.4 註冊 | ⬜ 未開始 |
| Sensor/Audio Task 實作 | ⬜ 未開始 |
| 與 RPi 整合 | ⬜ 未開始 |

---

## 12. 與 RPi 端的契約對照表

> 這是 RPi 端在 Web UI「新增頻道」時要填入的內容。STM32 燒錄後一旦變動，需同步更新本表並通知 RPi 端。

| Characteristic（STM32 提供）| RPi `type` | RPi `data_format` | RPi `unit` | 建議頻道命名 |
| --- | --- | --- | --- | --- |
| Temperature | display | float32_le | °C | `Temp` |
| Humidity | display | float32_le | % | `Humidity` |
| AccelMagnitude | display | float32_le | g | `Accel` |
| GyroMagnitude | display | float32_le | dps | `Gyro` |
| MotionAlert | display | uint8 | — | `MotionAlert` |
| MicLevel | display | uint16_le | — | `MicLevel` |
| LoudAlert | display | uint8 | — | `LoudAlert` |
| Led1State | controller | uint8 | — | `LED1` |
| ControlFlag | controller | uint8 | — | `ControlFlag` |

> RPi parser 若尚未支援上述某格式（目前列舉 `uint8` / `uint16_le` / `float32_le`，全部涵蓋）需另行擴充。

---

## 13. 已知風險與待決議事項

| 項目 | 風險 / 待決 | 暫定處理 |
| --- | --- | --- |
| BlueNRG-MS HCI 偶發 timeout | 高 ISR 負載下可能 stack 重置 | BleTask 高優先級 + mutex 序列化；Phase 1 驗證 |
| float32 magnitude 失精度 | 抖動小時數值無感 | v1 接受；後續可換成 mg/mdps 的 `uint16_le` |
| 麥克風校正常數 `scale` 待定 | LoudAlert 門檻可能偏掉 | Phase 1 跑實機抓 baseline |
| BLE pairing/encryption | v1 完全 open，誰都能 connect+write | 學生專題範圍接受；之後可加 LE Secure Connections（需 RPi 端配合） |
| 多裝置同時廣告撞名 | RPi UI 可能難辨識 | Local name 後綴 BD_ADDR 末 2 byte |
| 低功耗模式 | v1 沒做，FreeRTOS idle 不進 sleep | 主文件 §2.2 提到省電是核心目標；Phase 4 再優化 |
| 燒錄後 `data_format` 與 RPi 不一致 | RPi 解錯誤 | 本文件 §12 為唯一真實來源；變更須同步通知 RPi |
| Magnitude 計算用 `sqrt()` | M4F 有 FPU 不致命，但麥克風 RMS 量大時要小心 | RMS 用 int 累加 → 最後一次 sqrt |

---

## 14. 下一步行動

按本文件對齊後，下一步進入 Phase 1：

1. ~~確認 `.ioc` 狀態~~ ✅ 已完成（2026-05-30）：SPI3、I²C2 polling、DFSDM1（OutputClock divider=39、Ch1 SPI Falling）、DMA1_Ch4 Circular、FreeRTOS Heap 24 KB、HAL Timebase=TIM1、PA5=`LED1_PIN`。
2. **改造 `BlueNRG_MS/App/gatt_db.c`**：依 §5.4 註冊 Home Sensor / Home Control Service，固定 UUID。
3. **重整 `app_bluenrg_ms.c`**：移除 ST BLE Sensor App 專屬邏輯，留 BLE init + adv + event loop；廣播名稱改為 `HOME-XXXX`。
4. **建立 task 骨架**：在 `freertos.c` USER 區塊建立 `BleTask` / `SensorTask` / `AudioTask` + queue。
5. **整合 BSP 感測器讀取**：HTS221 + LSM6DSL + DFSDM。
6. **驗證**：nRF Connect 確認 §5.4 GATT 表全數可見、Read 有資料、Notify 有持續推送、Write LED 有反應。

文件結束。如需修改技術選型、GATT 表或任務模型，請在實作前提出討論。
