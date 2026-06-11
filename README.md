# 智能家庭控制系統

> 基於 Raspberry Pi 3 與 STM32 之 BLE 通訊應用  
> **課程**：Embedded Systems Lab｜**團隊**：趙子佾、詹詠翔、詹育晟  
> **GitHub**：[Embedded-in-your-heart/Intelligent-home](https://github.com/Embedded-in-your-heart/Intelligent-home)

---

## 目錄

1. [動機](#一動機)
2. [作法](#二作法)
3. [成果](#三成果)
4. [參考資料](#四參考資料)

---

## 一、動機

本學期課程中常用的 **STM32 開發板**擁有豐富的板載感測器（溫濕度、加速度、陀螺儀、麥克風），卻在「網路連線」與「建立伺服器」方面能力有限。

現有 IoT 方案大多依賴雲端中繼（MQTT broker、Firebase 等），帶來三個主要問題：
- **延遲**：資料必須繞過遠端伺服器才能回到同一區域網路的顯示端
- **隱私疑慮**：家庭感測資料（溫度、聲音、動作）上傳至第三方雲端
- **連網依賴**：斷網即失效

本專案的目標是打造一個**完全本地端運作**的智慧家庭系統：

- 以 **Raspberry Pi 3** 作為中央伺服器，提供 Web 管理介面與資料持久化
- 以 **STM32 B-L475E-IOT01A** 作為感測節點，透過 **BLE（低功耗藍牙）** 傳送感測資料
- 感測器原始數值經由 **DSP 處理**，輸出有意義的分析結果（dB(A) 音量、警報音偵測、震動分析、地震帶偵測）
- 支援**一對多**架構：一台 RPi 可同時管理多個 STM32 節點

---

## 二、作法

### 2.1 系統架構

```
┌──────────────────┐                      ┌──────────────────────┐
│  STM32 節點 A    │◄────── BLE ─────────►│                      │
│  HOME-XXXX       │                      │  Raspberry Pi 3      │
└──────────────────┘                      │  Flask + SQLite      │
                                          │  port 5000           │
┌──────────────────┐                      │                      │
│  STM32 節點 B    │◄────── BLE ─────────►│                      │
│  HOME-YYYY       │                      └──────────┬───────────┘
└──────────────────┘                                 │ HTTP / WebSocket
                                          ┌──────────▼───────────┐
                                          │     使用者瀏覽器      │
                                          │  HTMX + Chart.js     │
                                          └──────────────────────┘
```

通訊採 **BLE GATT** 協定，三個階段：
1. **廣播與探索**：STM32 廣播 `HOME-XXXX`（後綴為 BD_ADDR 末 2 bytes），RPi 掃描後由使用者在 Web UI 選擇新增
2. **GATT 連線**：RPi 主動連上 STM32，透過 Service / Characteristic 進行雙向讀寫
3. **Notify 推播**：感測數值更新後 STM32 主動推播給 RPi，RPi 再透過 WebSocket 推播至瀏覽器

### 2.2 STM32 韌體（`Intelligent-home-STM32-client`）

**硬體：** STM32 B-L475E-IOT01A（Cortex-M4F, 80 MHz）+ BlueNRG-MS BLE 模組

**FreeRTOS 三任務架構：**

| Task | 優先級 | 工作內容 |
|---|---|---|
| `BleTask` | High | BLE stack 主迴圈；GATT write callback（LED 控制）；從 NotifyQueue 讀資料後呼叫 `aci_gatt_update_char_value()` |
| `SensorTask` | Normal | LSM6DSL FIFO 104 Hz 批量讀取；每秒讀 HTS221；計算 MotionAlert；IMU DSP 計算震動 RMS / 地震帶 RMS |
| `AudioTask` | Normal | DFSDM + DMA 取 PDM；Audio DSP 計算 dB(A) 與 FFT 警報音偵測 |

所有 BLE ACI 呼叫序列化在 BleTask，其他 Task 透過 `NotifyQueue`（`osMessageQueue`）傳資料，確保 SPI3 無競爭。

**DSP 模組：**

*Audio DSP（`audio_dsp.c`）— 每 200 ms 處理 1600 個樣本（8 kHz）*
```
raw samples → A-weighting IIR biquad（3-stage DF1，IEC 61672-1）
            → weighted RMS → dB(A)

            → Hanning window → arm_rfft_fast_f32（1024-point FFT）
            → 偵測 2800–3600 Hz 峰值比例 → 警報音偵測
```

*IMU DSP（`imu_dsp.c`）— 每樣本輸入，每 104 樣本（1 s）輸出*
```
accel @ 104 Hz → 高通 Butterworth（fc=0.4 Hz）移除重力
              → 線性加速度 magnitude
              → Vibration path：RMS → 震動強度（mg）
              → Quake path：1–10 Hz 帶通 Butterworth → RMS → 地震警報
```

**GATT 表（對外契約，燒錄後固定）：**

Base UUID：`xxxxxxxx-8E22-4541-9D4C-21EDAE82ED19`

| Characteristic | UUID 短碼 | 格式 | 說明 |
|---|---|---|---|
| Temperature | `1A220002` | float32_le | 溫度 °C |
| Humidity | `1A220003` | float32_le | 濕度 % RH |
| AccelMagnitude | `1A220004` | float32_le | 加速度量值 g |
| GyroMagnitude | `1A220005` | float32_le | 陀螺儀量值 dps |
| MotionAlert | `1A220006` | uint8 | 異常晃動警示 0/1 |
| MicLevel | `1A220007` | uint16_le | 麥克風音量 0–1023 |
| LoudAlert | `1A220008` | uint8 | 大聲警示 0/1 |
| AlarmDetected | `1A22000A` | uint8 | 警報聲偵測 0/1 |
| MicDBA | `1A22000B` | float32_le | A-加權音量 dBA |
| VibrationRMS | `1A22000C` | float32_le | 震動強度 mg |
| VibrationAlert | `1A22000D` | uint8 | 家電運轉偵測 0/1 |
| QuakeAlert | `1A22000E` | uint8 | 地震警報 0/1 |
| Led1State | `1A22F002` | uint8 | LED 控制（Write）|
| ControlFlag | `1A22F003` | uint8 | 通用旗標（Write）|

### 2.3 Raspberry Pi 伺服器（`Intelligent-home-RPi-server`）

**技術選型：** Python 3.11、Flask、Flask-SocketIO（threading mode）、bluepy、SQLite、Jinja2 + HTMX + Bootstrap 5 + Chart.js

**分層架構：**

```
Web Layer     (Flask Blueprint)   — HTTP 路由、SocketIO 事件、Jinja2 模板
Service Layer (services/)         — 業務邏輯：device/channel/user service
BLE Layer     (ble/)              — BluepyManager（每個 peripheral 一個 worker thread）
                                    MockBLEManager（開發 / 測試用）
DB Layer      (db/)               — SQLite repository（users / devices / channels / readings）
```

**關鍵設計決策：**

- `bluepy` 是同步阻塞函式庫，每個 peripheral 獨立 worker thread + command queue + `Future`，Flask 主執行緒透過 `Future.result()` 等待，避免阻塞
- Notify → SocketIO 即時推播（不限頻）；DB 寫入限頻 1 Hz，控制儲存量
- 斷線後指數退避重連（1 s → 2 s → … → 60 s 上限），重連後自動重新訂閱所有頻道
- BLE 位址型別自動推斷（`infer_addr_type()`），處理 STM32 靜態隨機位址
- 頻道 Widget 依資料類型自動切換顯示模式（折線圖 / flag LED 燈號 / enum badge）

---

## 三、成果

### 3.1 感測與 DSP 功能

| 功能 | 顯示方式 | 更新頻率 |
|---|---|---|
| 溫度、濕度 | 即時折線圖 | 1 s |
| 加速度量值、陀螺儀量值 | 即時折線圖 | 250 ms |
| 麥克風音量（0–1023）| 即時折線圖 | 200 ms |
| **A-加權音量 dB(A)**（IEC 61672-1 A-weighting IIR）| 即時折線圖 | 200 ms |
| **震動強度 RMS（mg）**（104 Hz FIFO + 高通 biquad）| 即時折線圖 | 1 s |

### 3.2 警報偵測

觸發時右上角彈出紅色 Bootstrap Toast 通知（5 秒），並在 Dashboard 顯示 LED 燈號與上次觸發時間。

| 警報 | 觸發條件 |
|---|---|
| 異常晃動 | 加速度 > 1.8 g 或陀螺儀 > 250 dps，持續 ≥ 100 ms，1 s lockout |
| 大聲警示 | 麥克風音量 > 400，持續 ≥ 200 ms，1 s lockout |
| **警報聲偵測** | 1024-point FFT peak ratio（2800–3600 Hz）> 30% |
| **家電運轉** | 震動 RMS > 30 mg（遲滯門檻 15 mg）|
| **地震警報** | 1–10 Hz 帶通 RMS > 20 mg，2 s lockout |

### 3.3 遠端控制

- 瀏覽器 Toggle 開關 → BLE Write → STM32 LED1（PA5）即時亮滅
- ControlFlag 數值寫入（保留擴充點）

### 3.4 Web 管理系統

- 多使用者帳號（bcrypt 雜湊，CSRF 保護）
- BLE 掃描新增裝置；14 個頻道預設清單，下拉選單快速套用
- 裝置連線狀態即時更新；斷線自動重連
- Dashboard 時間窗切換（1m / 10m / 1h / 1d / 1w）
- 支援多節點同時管理

### 3.5 工程品質

- **190+ unit tests**，全部通過
- `ruff`（lint + format）與 `mypy strict`（型別檢查）全綠
- BLE 後端抽象化（`BLEManager` Protocol），非 BLE 層在 Windows 上即可執行完整測試

### 3.6 克服的技術挑戰

| 挑戰 | 解法 |
|---|---|
| bluepy 非執行緒安全 | 每個 peripheral 獨立 worker thread + command queue + Future |
| 開發板 DMA1 硬體故障（DMA 不搬資料）| 逐層 dump 暫存器確認根因，換板後改 DMA + interrupt 驅動，CPU 佔用從 ~5% 降至 ~0% |
| 多執行緒 SQLite | Connection injection 模式；BLE worker 執行緒獨立短連線 |
| STM32 靜態隨機 BLE 位址 | `infer_addr_type()` 依位址高 2 bits 自動推斷 |
| A-weighting 在 8 kHz 下的精度限制 | 已記錄於程式碼；3 kHz 以上有 ~2 dB 偏差，標示「calibration pending」|
| LSM6DSL 104 Hz 高速取樣 vs. 250 ms task 週期 | FIFO 連續模式 + 每次批量排水約 26 samples，等效 104 Hz 連續 DSP 輸入 |

---

## 四、參考資料

### 硬體文件
- STMicroelectronics, *STM32L475VG Datasheet*, DocID028698 Rev 4
- STMicroelectronics, *STM32L4 Series Reference Manual (RM0351)*, Rev 9
- STMicroelectronics, *B-L475E-IOT01A User Manual (UM2153)*, Rev 5
- STMicroelectronics, *HTS221 Datasheet* — Capacitive digital sensor for relative humidity and temperature
- STMicroelectronics, *LSM6DSL Datasheet* — iNEMO inertial module, 6 DoF IMU
- STMicroelectronics, *MP34DT01-M Datasheet* — MEMS audio sensor, omnidirectional digital microphone

### 軟體與中介層
- STMicroelectronics, *BlueNRG-MS Middleware* — HCI transport layer and GATT/GAP ACI
- STMicroelectronics, *X-CUBE-MEMS1* — MEMS component driver pack（HTS221、LSM6DSL）
- ARM, *CMSIS-DSP Software Library V1.6.0* — `arm_rfft_fast_f32`、`arm_biquad_cascade_df1_f32`
- FreeRTOS, *FreeRTOS Kernel V10* + CMSIS-RTOS2 wrapper

### 標準與演算法
- IEC 61672-1:2013, *Electroacoustics — Sound level meters — Part 1: Specifications* — A-weighting 濾波器極點頻率
- A. V. Oppenheim & R. W. Schafer, *Discrete-Time Signal Processing*, 3rd ed. — bilinear transform、Butterworth filter design

### 開放原始碼函式庫（RPi 端）
- Pallets, *Flask* — [https://flask.palletsprojects.com](https://flask.palletsprojects.com)
- *Flask-SocketIO* — [https://flask-socketio.readthedocs.io](https://flask-socketio.readthedocs.io)
- *bluepy* — BLE peripheral interface for Python on Linux — [https://github.com/IanHarvey/bluepy](https://github.com/IanHarvey/bluepy)
- *Chart.js* — [https://www.chartjs.org](https://www.chartjs.org)
- *HTMX* — [https://htmx.org](https://htmx.org)
- *Bootstrap 5* — [https://getbootstrap.com](https://getbootstrap.com)
- *uv* — Python package manager — [https://github.com/astral-sh/uv](https://github.com/astral-sh/uv)

### 開發工具
- STMicroelectronics, *STM32CubeIDE 1.19.0* — [https://www.st.com/stm32cubeide](https://www.st.com/stm32cubeide)
- *nRF Connect for Mobile* — BLE scanner / GATT browser（Nordic Semiconductor）
