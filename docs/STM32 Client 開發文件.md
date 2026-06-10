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
| SoundClass | `1A220009` | Read + Notify | `uint8` | `0`=quiet / `1`=speech / `2`=clap / `3`=alarm / `4`=other | 變化時 |
| AlarmDetected | `1A22000A` | Read + Notify | `uint8` | `1`=alarm tone detected / `0`=normal | 事件觸發即推 |
| MicDBA | `1A22000B` | Read + Notify | `float32_le` | dB(A) | 每 200 ms |
| VibrationRMS | `1A22000C` | Read + Notify | `float32_le` | mg（高通濾波加速度 RMS，1 s 視窗）| 每 1 s |
| VibrationAlert | `1A22000D` | Read + Notify | `uint8` | `1`=vibration detected / `0`=normal | 事件觸發即推 |
| QuakeAlert | `1A22000E` | Read + Notify | `uint8` | `1`=earthquake-like motion detected / `0`=normal | 事件觸發即推 |

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
| LoudAlert=1 | `MicLevel > 400`（正規化，2026-05-31 實機校正）| 連續 ≥ 200 ms 才觸發；事件結束後鎖 1 s |
| AlarmDetected=1 | SoundClass 連續 ≥ 3 個 200 ms 視窗判定為 alarm (SoundClass=3) | 連續 ≥ 5 個非警報視窗 → 回到 0；狀態轉換間鎖 2 s |
| VibrationAlert=1 | VibrationRMS ≥ `VIB_ON` 門檻的連續 ≥ 5 個 1 s 視窗 → 觸發；≥ 10 個視窗低於 `VIB_OFF` 門檻 → 清除 | 鎖機制內建於狀態機 |
| QuakeAlert=1 | 1–10 Hz 帶通能量（RMS）≥ `QUAKE` 門檻的連續 ≥ 3 個 1 s 視窗 → 觸發；≥ 5 個視窗低於門檻 → 清除；狀態轉換間鎖 2 s | 開機後前 5 s 的 warm-up 期間抑制所有振動/地震告警 |

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
- **加速度採樣率**：104 Hz（FIFO 每 250 ms 批次讀取；高通濾波（0.4 Hz）去重力後用於 VibrationRMS 計算）；**陀螺儀採樣率**：4 Hz（MotionAlert 用，保持原狀）
- **FIFO 使用**：每 250 ms 從 FIFO 讀出一批加速度樣本（≈26 個），並應用 1–10 Hz 帶通濾波用於地震檢測
- `AccelMagnitude = sqrt(ax² + ay² + az²) / 1000.0` (mg → g)
- `GyroMagnitude = sqrt(gx² + gy² + gz²) / 1000.0` (mdps → dps)
- **VibrationRMS** = 高通濾波（0.4 Hz）加速度大小的 RMS，每 1 s 視窗計算
- **QuakeAlert** = 1–10 Hz 帶通濾波能量判斷（見 §5.5）
- MotionAlert / AccelMagnitude / GyroMagnitude：見 §5.5，行為無變

### 7.3 MP34DT01（麥克風）

- DFSDM + DMA 串流，每 200 ms 視窗（取決於 DFSDM 設定）
- DMA half/full callback 計算 RMS（int32 累加，避免溢位）
- 正規化：`MicLevel = clamp(round(rms / scale), 0, 1023)`（`scale` 在校正時決定）
- LoudAlert：見 §5.5

> v2 支援音頻 DSP：每 200 ms 視窗對前 1024 樣本做 Hanning window + RFFT(1024, CMSIS-DSP)、依頻帶能量規則分類（quiet/speech/clap/alarm/other）至 SoundClass，並應用 A-weighting 3 級 biquad filter 計算 dB(A) 至 MicDBA。常數（分類門檻、A-weighting 係數）待實機校正。

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
│   │   ├── app_bluenrg_ms.c            # BLE init、main-loop callback（改造）
│   │   ├── sensor.c                    # BLE GAP / 廣告（HOME-XXXX）/ HCI event router
│   │   ├── gatt_db.c                   # GATT Service/Char 表 + Write callback（LED1 GPIO）
│   │   ├── notify_queue.c              # sensor/audio task → BleTask 推播佇列（新）
│   │   ├── sensor_task.c               #   HTS221 + LSM6DSL 採集（新）
│   │   ├── audio_task.c                #   MP34DT01 麥克風 DFSDM+DMA（新）
│   │   ├── audio_dsp.c                 #   CMSIS-DSP 音頻分類、A-weighting（新）
│   ├── imu_dsp.c                      #   LSM6DSL 104 Hz FIFO 讀取、高通與帶通濾波、VibrationRMS/QuakeAlert 計算（新）
│   │   ├── imu_dsp.h
│   │   └── (對應 .h)
│   └── Target/                         # hci_tl_interface 等（不動）
├── X-CUBE-MEMS1/Target/                # MEMS 設定（hook 到 BSP_I2C2，不動）
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

### 10.1 開發工作流程：先 serial 再 BLE

每個新增的感測器 / 致動器都遵守兩步流程：

1. **Serial 驗證階段**：先把原始讀值、解碼後數值、事件觸發都打到 USART1 VCP；肉眼確認 I²C / DMA / 時間軸 / 門檻全部正確。**此階段完全不動 BlueNRG stack**。
2. **BLE 串接階段**：上一步全部 pass 才把資料推到 `NotifyQueue` 給 BleTask 呼叫 `aci_gatt_update_char_value()`；或在 Write callback 中驅動 GPIO，完成端到端。

為什麼這樣分：BlueNRG-MS 偶發 HCI timeout、SPI 卡死、`aci_gatt_update_char_value()` 回非零 status 等錯誤，與「感測器讀值錯」症狀類似但解法完全不同。先讓 serial 看到正確值，就能把這兩類問題切開。Milestone 編號裡的「a / b」後綴對應這兩個階段（見 §14）。

### 10.2 分層驗證表

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

### 11.1 目前實作進度（截至 2026-05-31）

| Milestone | 子項目 | 狀態 |
| --- | --- | --- |
| M0 | 板子可開機、BlueNRG_MS template 匯入 | ✅ |
| M0 | `.ioc` 設定齊備（SPI3、I²C2 polling、DFSDM1 + DMA、FreeRTOS Heap 24 KB、HAL Timebase=TIM1、`LED1_PIN`=PA5） | ✅ |
| **M1** | `gatt_db.{c,h}` 改寫為 §5.4 兩個 Service / 9 條 char（固定 UUID） | ✅ |
| **M1** | `sensor.c` 廣告 local name 改成執行時組 `HOME-XXXX` | ✅ |
| **M1** | `app_bluenrg_ms.c` 砍掉 SensorDemo 邏輯、無 pairing | ✅ |
| **M1** | `freertos.c` 新增 `BleTask`（osPriorityHigh, 1 KB stack） | ✅ |
| **M1** | `BLE1_DEBUG=1` + `BUS_UART1_BAUDRATE=115200` | ✅ |
| **M1** | nRF Connect 看得到 `HOME-XXXX`、GATT browser 與 §5.4 一致、Write Control char 進入 `Attribute_Modified_CB` | ✅（2026-05-30 驗證） |
| **M2a** | X-CUBE-MEMS1 + X-CUBE-ALGOBUILD generate（含 HTS221/LSM6DSL component driver + CMSIS-DSP） | ✅（2026-05-31） |
| **M2a** | `sensor_task.c` + SensorTask 4 Hz 採集（走 BSP_I2C2 + component driver），USART1 印解碼值 | ✅（2026-05-31 驗證：室溫合理、握住升溫、搖晃 accel 跳、靜止 gyro 近零） |
| **M2b** | `notify_queue.{c,h}` + SensorTask 推 NotifyQueue + BleTask `NotifyQueue_Pump()` + MotionAlert 100 ms hold / 1 s lockout | ✅（2026-05-31 驗證：nRF Connect 訂閱所有 5 條感測 char 看到資料流） |
| **M3a** | `audio_task.c` 麥克風採集算 RMS，印 mic 能量 | ✅（2026-05-31 先以 polling 驗證 mic 硬體健康：安靜 rms≈0、大聲說話 rms 100–130；§15.2.6）|
| **M3a-DMA** | 改 DMA circular + HT/TC 中斷驅動（callback `osThreadFlagsSet` 喚醒 AudioTask） | ✅（分支 `feat/dma-interrupt`；原「DMA 不搬」確認為開發板硬體故障，換板解決，§15.1）|
| **M3b** | 推 NotifyQueue → `MicLevel` (`rms × 8`, clamp 0..1023) + LoudAlert 200 ms hold / 1 s lockout（門檻 400）| ✅（2026-05-31 驗證：nRF Connect 訂閱 MicLevel 跟拍手 / 說話變動、LoudAlert 觸發/清除正常）|
| **M4** | `Attribute_Modified_CB` 內驅動 PA5 GPIO；ControlFlag 持久化 | ✅（2026-05-31 驗證：nRF Connect 寫 0x01 LED 亮、0x00 熄滅、ControlFlag echo） |
| **M5** | CMSIS-DSP 整合 + SoundClass/AlarmDetected/MicDBA 三條 char | ✅（2026-06-11，待實機校正門檻） |
| **M6** | LSM6DSL 104 Hz FIFO + VibrationRMS/VibrationAlert/QuakeAlert | ✅（2026-06-11，待實機校正門檻與 FIFO 驗證） |
| Phase 3 | 與 RPi 端整合測試 | ⬜ |
| Phase 4 | 多節點、連線穩定性、（選用）low power | ⬜ |

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
| SoundClass | display | uint8 | `enum:0=安靜,1=語音,2=拍手,3=警報,4=其他` | `SoundClass` |
| AlarmDetected | display | uint8 | — | `AlarmDetected` |
| MicDBA | display | float32_le | dB(A) | `MicDBA` |
| VibrationRMS | display | float32_le | mg | `VibrationRMS` |
| VibrationAlert | display | uint8 | — | `VibrationAlert` |
| QuakeAlert | display | uint8 | — | `QuakeAlert` |
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
| ~~DFSDM1_FLT0 DMA 不搬資料~~ ✅ 已解決 | 曾以為是 CubeMX DMA request 映射 bug，實為**開發板硬體故障**（DMA1 控制器忽略周邊 request）。設定/HAL 一直正確：`CSELR.C4S=0` 即 DFSDM1_FLT0（RM0351 Rev 9 **Table 44**，非 Table 41）| 換板後 DMA 正常；麥克風改 DMA + interrupt 驅動（分支 `feat/dma-interrupt`）。完整紀錄見 §15 |

---

## 14. 下一步行動

Milestone 1 已完成（2026-05-30）。後續按 §10.1 「先 serial 再 BLE」流程逐 milestone 推進。

### Milestone 2 — 環境 / 動作感測器

**2a：Serial 驗證**
- 新增 `BlueNRG_MS/App/i2c_dev.{c,h}`：I²C2 read / write byte 與 multi-byte register block helper（B-L475E-IOT01A1 BSP 沒附 MEMS driver）
- 新增 `BlueNRG_MS/App/hts221.{c,h}`：WHO_AM_I 檢查、讀 H0/H1/T0/T1 校正、解碼 `HUM_OUT` / `TEMP_OUT`
- 新增 `BlueNRG_MS/App/lsm6dsl.{c,h}`：WHO_AM_I、CTRL1_XL（accel 416 Hz / ±2 g）/ CTRL2_G（gyro 416 Hz / 250 dps）、讀 `OUTX_L/H_A` × 3 / `OUTX_L/H_G` × 3
- 新增 `BlueNRG_MS/App/sensor_task.{c,h}`：FreeRTOS task（osPriorityNormal, 768 B stack），250 ms tick：4 Hz LSM6DSL、每 4 tick 一次 HTS221
- `freertos.c` USER 區塊建 SensorTask
- USART1 印 `[env] T=25.3 H=42.1` 每秒、`[imu] mag_a=1.01g mag_g=2dps` 每 250 ms
- **驗證**：室溫、握住板子體溫上升、搖晃 → 加速度爆增、靜止 → 陀螺儀近零

**2b：BLE 串接**
- 在 `freertos.c` 加 `NotifyQueue`（`osMessageQueue`, 16 元素 × 8 byte）：`{char_id, len, payload[4]}`
- 在 `gatt_db.c` 新增內部 `enum HomeCharId`（temp / humidity / accel_mag / gyro_mag / motion_alert / mic_level / loud_alert / led1 / control_flag），讓 BleTask 一個 switch 派送
- SensorTask：計算 `AccelMagnitude` / `GyroMagnitude`、MotionAlert 門檻 + 反跳（§5.5）→ 入 queue
- BleTask 主迴圈追加 `osMessageQueueGet(NotifyQueue, ..., 0)` → 依 char_id 呼 `Home_*_Update`
- **驗證**：nRF Connect 訂閱 Temperature 看到秒級更新；搖晃看到 AccelMag 跳動、MotionAlert 從 0 → 1

### Milestone 3 — 麥克風

> ✅ **狀態：DMA + interrupt 路徑完成（分支 `feat/dma-interrupt`）**。
> 歷程：先以 polling 完成（2026-05-31），因當時 DMA 不搬資料；後查明（issue #1）是**開發板硬體故障**，非韌體 — DMA 設定與 HAL 一直正確（`CSELR.C4S=0` 即 DFSDM1_FLT0，RM0351 Table 44）。換板後 DMA 正常，遂把 audio_task 改回 DMA circular + HT/TC 中斷驅動：callback 只 `osThreadFlagsSet` 喚醒 AudioTask，task 對每個 200 ms 半 buffer 算 RMS → MicLevel / LoudAlert。CPU 從 ~5% 降到接近 0%、取樣覆蓋率 5% → 100%。校正沿用 polling（`raw>>16 == sample24>>8`，故 `MIC_SCALE_NUM=8` / `LOUD_THRESHOLD=400` 不變）。完整紀錄見 §15。

**3a：Serial 驗證**
- 新增 `BlueNRG_MS/App/audio_task.{c,h}`：FreeRTOS task（osPriorityNormal, 1 KB stack）
- AudioTask init 時呼 `HAL_DFSDM_FilterRegularStart_DMA(&hdfsdm1_filter0, buf, BUF_LEN)`，CIRCULAR 模式
- 實作 `HAL_DFSDM_FilterRegConvHalfCpltCallback` / `HAL_DFSDM_FilterRegConvCpltCallback`（在 `Core/Src/stm32l4xx_it.c` 的 USER 區塊或新檔），把半 / 全 buffer 內樣本平方累加，透過 task notification 喚醒 AudioTask
- AudioTask 取累加值 → `sqrt` 一次 → 印 `[mic] rms=42`
- **驗證**：安靜 baseline 約幾十、拍手 / 說話飆到上百

**3b：BLE 串接**
- 正規化：`MicLevel = clamp(round(rms / scale), 0, 1023)`，`scale` 用 3a 抓的 baseline 校
- LoudAlert：`MicLevel > 400` 連續 ≥ 200 ms（同 §5.5；門檻已實機校正）
- 推 NotifyQueue → BleTask
- **驗證**：nRF Connect 訂閱 MicLevel 看到 200 ms 級數值流；拍手觸發 LoudAlert

### Milestone 4 — 致動器

> 範圍小，不切 a/b：寫 callback 即可同時用 nRF Connect 立刻驗。
- `gatt_db.c::Attribute_Modified_CB` 內 `if (handle == led1_state_char_handle + 1)` 那段：呼 `HAL_GPIO_WritePin(LED1_PIN_GPIO_Port, LED1_PIN_Pin, data[0] ? GPIO_PIN_SET : GPIO_PIN_RESET)`
- ControlFlag：先存在 `static volatile uint8_t g_control_flag` 全域，PRINTF 印出新值即可（無實際動作，保留擴充點）
- **驗證**：nRF Connect 寫 `0x01` 到 `1A22F002` → PA5 LED 亮、`0x00` → 熄滅；寫 `0xAB` 到 `1A22F003` → serial 印 `Write ControlFlag = 0xAB`

### Phase 2 完成標準
- nRF Connect 訂閱 7 條 Display char 全部都有數據流（**若 M3 仍 blocked，MicLevel / LoudAlert 暫不計入**；5 條感測 char + 2 條 control char 即視為 Phase 2 主功能完成）
- 寫 2 條 Control char 都有反應
- 連續 30 分鐘不掉線、不需要復位

### Phase 3 / Phase 4

主文件對應的後續：
- **Phase 3**：與 RPi server 端整合（RPi 接 §3e SocketIO + Web Dashboard 後直連本韌體）
- **Phase 4**：多 STM32 節點同時佈署、連線穩定性、（選用）low power

---

## 15. Blocker 紀錄

### 15.1 M3a — DFSDM1 DMA 不搬資料（✅ 已解決：開發板硬體故障）

> **結論（2026-06-04，GitHub issue #1 結案）**：根因是**開發板 DMA1 控制器硬體故障** —
> 它能執行軟體觸發（MEM2MEM）搬移，卻忽略所有周邊觸發的 DMA request（DFSDM ch4 與 ADC ch1
> 兩通道都不搬、且無任何錯誤旗標）。**韌體 / 設定 / HAL 一直是對的**：
> - `CSELR.C4S=0` 即 DFSDM1_FLT0（RM0351 Rev 9 **Table 44**，非當初誤查的 Table 41）。
> - 當初「CubeMX 把 request 設成 ADC2、應強制 C4S=7」的假設**是錯的**；在 ch4 上 `0111` 並非 DFSDM，
>   強制 C4S=7 反而把 mux 指離 DFSDM1_FLT0。
> - polling 能讀到 mic → mic / DFSDM / 時脈 / PDM 解調全部正常。
>
> **換一片新板子後 DMA 立即正常**，audio_task 已改回 DMA + interrupt 驅動（分支 `feat/dma-interrupt`）。
>
> 以下保留原始除錯歷程（含已被推翻的「request 映射」假設）作為過程紀錄。

**症狀（原始觀察）**
- `HAL_DFSDM_FilterRegularStart_DMA(&hdfsdm1_filter0, ...)` 回傳 `HAL_OK`
- 但 ISR (`HAL_DFSDM_FilterRegConvHalfCpltCallback` / `...CpltCallback`) 完全沒被呼叫
- AudioTask 持續印 `[mic] no samples (DMA stalled?)`

**診斷（執行 `dump_dfsdm_state()` 在 audio_task.c）**

```
[diag:after-start]
  CH0 CHCFGR1=80260000  DFSDMEN=1 CKOUTDIV=38         ← DFSDM 全域開、CKOUT 2.05 MHz ✓
  CH1 CHCFGR1=00000185  CHEN=1                        ← Channel 1 ✓
  CH2 CHCFGR1=00000084  CHEN=1                        ← Channel 2 ✓
  FLT0 CR1=21240001  DFEN=1 RDMAEN=1 RCONT=1 RCH=1    ← Filter 啟用、綁 ch1、DMA enable、continuous ✓
  FLT0 CR2=00000000  ISR=00FF400A                     ← ISR 的 ROVRF (bit 3) = 1：Filter 產出 data 但沒人讀
  DMA1 CCR=00000AAF EN=1 HTIE=1 TCIE=1 CIRC=1  CNDTR=800  CSELR.C4S=0
```

關鍵：
1. **`CNDTR=800`**（兩次 dump 完全相同 → DMA 完全沒搬資料）
2. **`FLTISR.ROVRF=1`** → Filter 在產 data，DMA 沒接走，filter 內 overrun
3. **`CSELR.C4S=0`** → DMA1_Channel4 的 request selector 是 0

**根因**
- CubeMX 在 `MX_DFSDM1_Init` 的 `HAL_DFSDM_FilterMspInit` 內寫死 `hdma_dfsdm1_flt0.Init.Request = DMA_REQUEST_0`
- 但 **STM32L475 RM0351 Rev 9 Table 41** 對 DMA1_Channel4 的 8 個 request 映射：

  | C4S | Peripheral |
  | --- | --- |
  | 0 | ADC2 |
  | 1 | SPI2_RX |
  | 2 | USART1_TX |
  | 3 | I2C2_RX |
  | 4 | TIM1_CH4 / TIM1_TRIG / TIM1_COM |
  | 5 | TIM7_UP / DAC_CH2 |
  | 6 | SAI1_A |
  | **7** | **DFSDM1_FLT0** ← 我們要的 |

- 所以 `CSELR.C4S=0` 意味著 DMA 在等 ADC2 的 request，filter 產 data 它根本看不到

**為什麼 CubeMX 會錯**
猜測 X-CUBE-BLE1 / 純 DFSDM template 的 MSP 預設值是 hard-code 的 `DMA_REQUEST_0`，沒有依 chip 的 DMA mapping 表 query 正確值。X-CUBE-MEMS / X-CUBE-AUDIO 的官方範例會主動 patch 掉，但純 DFSDM template 漏了這一步。CubeMX 內也沒有 user-configurable 的 DMA Request 欄位 — 重 generate 沒用。

**已試過但無效**
1. `MODIFY_REG(DMA1_CSELR->CSELR, (0xFU<<12), (7U<<12))` 在 `HAL_DFSDM_FilterRegularStart_DMA` 之前直接寫 CSELR → dump 仍顯示 `C4S=0`
2. `hdma_dfsdm1_flt0.Init.Request = 7U; HAL_DMA_Init(&hdma_dfsdm1_flt0);` 再 PRINTF 立即驗證 → 仍 `C4S=0`
3. 額外呼 `HAL_DFSDM_FilterMspInit(&hdfsdm1_filter0)`（追加嘗試）→ 反向把 Request 沖回 0（MspInit 內部寫 `Init.Request = DMA_REQUEST_0`）

**還沒試的方向**
- 移除 audio_task.c 內 `HAL_DFSDM_FilterMspInit` 呼叫，只留 (2) 的兩行，再驗
- ST-LINK debugger step 進 `HAL_DMA_Init`，確認 line 251 (`DMA1_CSELR->CSELR |= ...`) 是否真的執行、寫入後立即讀回是否仍是 7
- 試其他 C4S 值（1–6）— 排除 RM 解讀有誤的可能
- 改用 DFSDM **polling mode**（`HAL_DFSDM_FilterRegularStart` + 在 AudioTask loop 內 `HAL_DFSDM_FilterPollForRegConversion`）—— 慢但完全繞過 CSELR
- 直接 drop DFSDM mic — `MicLevel` / `LoudAlert` 兩條 char 仍註冊但永遠 0；對主功能無影響

**目前處置（2026-05-31 更新）**
- M3 已**改用 polling 路徑**完成（`audio_task.c` 用 `HAL_DFSDM_FilterPollForRegConversion`），MicLevel 與 LoudAlert 兩條 BLE char 都正常推播
- Mic 硬體在 polling 模式下確認健康（安靜 rms≈0、loud speech rms ~100–130；§15.2.6）
- DMA bug 仍然存在但 **降級為 Phase 4 效能優化項目**（不再 blocker）：
  - polling 成本：CPU ~5%、取樣覆蓋率 ~5%（每 200 ms 取 10 ms）
  - DMA 修好可降到 CPU 接近 0%、覆蓋率 100%
- 修 DMA 的工作底稿仍在 §15.2.8，將來 Phase 4 啟動時直接照那個順序排除

### 15.2 MP34DT01 / DFSDM / DMA 通訊鏈技術參考

> 將來恢復 M3 時的工作底稿：協定、接線、暫存器層已確認 vs 待確認、測試順序。

#### 15.2.1 板子釐清

我們手上是 **B-L475E-IOT01A（STM32L475VG）**。**B-L4S5I-IOT01A（STM32L4S5VI / L4+）** 是新版，用 **DMAMUX**（非 CSELR）— §15.1 的 `DMA_REQUEST_0` 對 ADC2 的 bug 在 L4+ 上不存在（因為 `DMA_REQUEST_DFSDM1_FLT0` 是 named constant）。若改用 L4S5I 板，本節絕大部分技術描述仍適用，僅 DMA request 映射段落不適用。

#### 15.2.2 MP34DT01 通訊協定（PDM）

- **輸出格式**：PDM (Pulse-Density Modulation) — 單線、單 bit、過取樣
- 0/1 密度比 ≈ 瞬時振幅；**必須經 decimation filter 還原為 PCM**
- 時脈範圍：**1 – 3.25 MHz**（典型 2.4 MHz，我們用 2.05 MHz）
- 取樣率公式：`fs = CKOUT / (FOSR × IOSR)` → `2.05M / 256 ≈ 8 kHz`

| Pin（mic 側）| 方向 | 說明 |
| --- | --- | --- |
| CLK | MCU → mic | PDM 時脈 |
| DOUT | mic → MCU | PDM 1-bit 資料，與 CLK 同步 |
| L/R | 板上固定 | B-L475E-IOT01A 上接 GND → 左聲道、上升緣輸出 |

#### 15.2.3 GPIO 接線（B-L475E-IOT01A 板載 routing）

| Pin | 信號 | AF | mode / speed / pull |
| --- | --- | --- | --- |
| **PE7** | `DFSDM1_DATIN2`（mic DOUT）| AF6 | `AF_PP`, LOW, `NOPULL` |
| **PE9** | `DFSDM1_CKOUT`（mic CLK）| AF6 | `AF_PP`, LOW, `NOPULL` |

> 注意：PE7 是 **Channel 2** 的 data input pin，不是 Channel 1。Channel 1 透過 `Pins=FOLLOWING_CHANNEL_PINS` 從 Channel 2 的 pin 取資料 — PDM 在 DFSDM 上的標準對位。

#### 15.2.4 DFSDM channel routing

```
            ┌────────────────────────────────────────────────┐
            │                  DFSDM1                         │
            │                                                 │
PE9  ◀──────┤ CKOUT  (divider 39 from 80 MHz APB → 2.05 MHz)  │
            │                                                 │
PE7  ──────▶┤ DATIN2 ──→  Channel 2  (rising edge)  ─┐         │
            │                                        │         │
            │              Channel 1                 ├──→ Filter 0 ──→ FLTRDATAR
            │              (falling edge, takes      │                  (24-bit signed
            │               data from Ch2's pin)     │                   in bits [31:8])
            │                                                 │
            └─────────────────────────────────────────────────┘
```

關鍵設定（取自 `Core/Src/dfsdm.c`）：

| 項目 | 值 | 作用 |
| --- | --- | --- |
| `Ch1.Input.Multiplexer` | `EXTERNAL_INPUTS` | 從外部 pin 取 |
| `Ch1.Input.Pins` | `FOLLOWING_CHANNEL_PINS` | 從 Ch2 的 pin (PE7) 拿 |
| `Ch1.SerialInterface.Type` | `SPI_FALLING` | CKOUT 下降緣 sample |
| `Ch1.SerialInterface.SpiClock` | `INTERNAL` | 用 CKOUT 當 SPI clock |
| `Ch2.Input.Pins` | `SAME_CHANNEL_PINS` | 用自己的 pin (PE7) |
| `Ch2.SerialInterface.Type` | `SPI_RISING` | CKOUT 上升緣 sample（PDM 主路徑）|
| `Filter0.RegularParam.Trigger` | `SW_TRIGGER` | 啟動時靠 RSWSTART |
| `Filter0.RegularParam.DmaMode` | `ENABLE` | 設 FLTCR1.RDMAEN |
| `Filter0.FilterParam.SincOrder` | `SINC3_ORDER` | |
| `Filter0.FilterParam.Oversampling` | 16 | FOSR |
| `Filter0.FilterParam.IntOversampling` | 16 | IOSR |
| `Filter0 ConfigRegChannel` | `(filter0, CHANNEL_1, CONTINUOUS_CONV_ON)` | RCH=1、RCONT=1 |

> Ch1 + Ch2 **都要 enable**（CHEN bit）。Ch2 即使資料不被 Filter 直接讀，**它的 pin (PE7) 才是 mic 真正接到的點**。

#### 15.2.5 DMA 路徑

```
Filter 0 完成一次 conversion
        │
        ▼
FLTRDATAR (24-bit signed in bits [31:8])
        │
        ▼
DFSDM raise "FLT0 conversion complete" → DMA request 訊號
        │                              ▲
        │  ┌───────────────────────────┘
        │  │
        │  │ 等的是「CSELR.C4S 指定的那個 peripheral」的 request
        ▼  │
   DMA1_Channel4  (★ §15.1 BUG: CSELR.C4S=0 → 在等 ADC2 而非 DFSDM1_FLT0)
        │
        │ (若 C4S 對 → transfer 觸發)
        ▼
   讀 FLTRDATAR → 寫 dma_buf[idx];  CNDTR--;  idx++
        │
        ▼
   到半 buffer (400 word) → fire HTIE → DMA1_Channel4_IRQHandler
   到全 buffer (800 word) → fire TCIE → DMA1_Channel4_IRQHandler
        │
        ▼
   HAL_DMA_IRQHandler → XferHalfCpltCallback / XferCpltCallback
        │
        ▼
   HAL_DFSDM_FilterRegConv{Half}CpltCallback
        │ (audio_task.c override)
        ▼
   accumulate_half() → ISR 平方累加 acc_sumsq / acc_count
```

#### 15.2.6 已確認 OK 部位（暫存器值，2026-05-31 dump）

| 層 | 暫存器 / 位元 | 觀測值 | 評估 |
| --- | --- | --- | --- |
| DFSDM 全域 | `CH0.DFSDMEN` (bit 31) | 1 | ✅ Peripheral 通電 |
| CKOUT 時脈分頻 | `CH0.CKOUTDIV` (bits [23:16]) | 38 | ✅ 80 MHz / 39 ≈ 2.05 MHz |
| Channel 1 啟用 | `CH1.CHEN` (bit 7) | 1 | ✅ |
| Channel 2 啟用 | `CH2.CHEN` (bit 7) | 1 | ✅ |
| Filter 啟用 | `FLT0.DFEN` (bit 0) | 1 | ✅ |
| Filter → DMA | `FLT0.RDMAEN` (bit 21) | 1 | ✅ |
| Filter 連續轉換 | `FLT0.RCONT` (bit 18) | 1 | ✅ |
| Filter regular channel | `FLT0.RCH` (bits [26:24]) | 1 | ✅ 綁 Ch1 |
| Filter fast mode | `FLT0.FAST` (bit 29) | 1 | ✅ |
| **Filter 真的在產 data** | `FLTISR.REOCF` (bit 1) | 1 | ✅ 有 sample ready |
| DMA channel 啟用 | `DMA1_Ch4.CCR.EN` (bit 0) | 1 | ✅ |
| DMA 半完成中斷 | `DMA1_Ch4.CCR.HTIE` (bit 2) | 1 | ✅ |
| DMA 完成中斷 | `DMA1_Ch4.CCR.TCIE` (bit 1) | 1 | ✅ |
| DMA Circular | `DMA1_Ch4.CCR.CIRC` (bit 5) | 1 | ✅ |
| DMA 資料寬度 | `CCR.PSIZE/MSIZE` | 32 / 32 | ✅ |
| DMA mem auto-inc | `CCR.MINC` (bit 7) | 1 | ✅ |
| GPIO PE7/PE9 AF6 | (MspInit 已執行；filter init 成功推論)| — | ✅ |
| DMA1 clock | `MX_DMA_Init` 內 `__HAL_RCC_DMA1_CLK_ENABLE` | — | ✅ |
| NVIC DMA1_Ch4 | `MX_DMA_Init` priority 6 | — | ✅ |

#### 15.2.7 可疑 / 未實機確認部位（註：此表為當時假設，最終根因為硬體故障，見 §15.1）

| 部位 | 觀察 | 可疑原因 |
| --- | --- | --- |
| ~~`CSELR.C4S = 0`（應為 7）~~ | §15.1 | ❌ 假設錯誤 — `C4S=0` 才正確（=DFSDM1_FLT0, Table 44）|
| `FLTISR.ROVRF=1` (bit 3) | regular overrun | Filter 產 data 但 DMA 沒讀（板子 DMA1 不回應周邊 request 的後果）|
| `FLTISR` 高 byte = 0xFF（bits [23:16] = CKABF[7:0]）| 多 channel clock-absence flag | sticky bit / 啟動 transient / 真有 clock 異常 — 任一可能，**未實機驗** |
| **PE9 CKOUT 實際波形** | 未量 | 軟體邏輯正確，但**無 scope 量過**；可能 CKOUT 根本沒輸出 |
| **PE7 DOUT 實際波形** | 未量 | 若 CKOUT 不出，mic 不吐 data；需 scope 看 PE7 是否有 1/0 切換 |
| GPIO speed `LOW` | CubeMX 預設 | 對 2 MHz CKOUT 邊緣略弱（不致命，可改 MEDIUM 改善穩定度）|
| Mic 供電 | always-on（板上設計）| 未實測；若板子焊接有問題、VCC rail 異常都會無聲 |
| L/R select pin | 應接 GND | 未實測腳位電位 |

#### 15.2.8 將來恢復 M3 的測試順序

照「先軟體再硬體」的逐步排除：

**Step 1 — 修 CSELR.C4S（純軟體）**

按 GitHub Issue #1 第一條：拿掉 `audio_task.c` 內額外的 `HAL_DFSDM_FilterMspInit` 呼叫，只留：

```c
hdma_dfsdm1_flt0.Init.Request = 7U;
HAL_DMA_Init(&hdma_dfsdm1_flt0);
```

驗 `[patch]` PRINTF 顯示 `C4S=7`。

**Step 2 — Step 1 修好但仍無資料 → 量 PE9 CKOUT**

```
scope / logic analyzer on PE9:
  期望：穩定 2.05 MHz 方波，duty ~50%
  若無      → CKOUT 沒輸出，查 CH0.CKOUTSRC / Activation / DFSDMEN
  若頻率怪  → CKOUTDIV 寫錯
```

**Step 3 — CKOUT OK 但 mic 不吐 data → 量 PE7 DOUT**

```
scope on PE7:
  安靜環境：密集 1/0 切換，接近 50/50（零振幅）
  拍手    ：1/0 密度明顯偏移
  卡固定值：mic 死掉 / VCC 沒接 / L/R 腳問題
  與 PE9 無關：mic 沒收到 CKOUT
```

**Step 4 — 硬體 OK 但 DFSDM 解不出 data → 改 polling 排除 DMA**

```c
HAL_DFSDM_FilterRegularStart(&hdfsdm1_filter0);
for (;;) {
    if (HAL_DFSDM_FilterPollForRegConversion(&hdfsdm1_filter0, 100) == HAL_OK) {
        uint32_t ch;
        int32_t v = HAL_DFSDM_FilterGetRegularValue(&hdfsdm1_filter0, &ch);
        PRINTF("mic = %ld\n", (long)(v >> 8));
    }
}
```

若 polling 拿到合理值 → 100% 鎖定問題只在 DMA / CSELR。

**Step 5 — Polling OK 再切回 DMA + Step 1 patch**

驗 CNDTR 真的會掉、ISR 真的進來。

#### 15.2.9 ~~快速 demo drop 方案~~ — 已不適用（2026-05-31）

> 原規劃在 M3 卡死時把 `MicLevel`/`LoudAlert` 都推 0。實際我們找到了 **polling 路徑**直接取得真實資料，不必 drop。此節保留紀錄用，本專案不採用。
- 文件標 future work

---

文件結束。如需修改技術選型、GATT 表或任務模型，請在實作前提出討論。
