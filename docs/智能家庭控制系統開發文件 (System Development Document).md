# 智能家庭控制系統開發文件

本項目旨在利用低功耗無線技術（BLE），建立一個**高度模組化**且具備**本地端安全性**的智慧家庭生態系統 。  

## 1. 專案基本資訊

- 

  **專案名稱：** 智能家庭控制系統（基於 Raspberry Pi 3 與 STM32 之 BLE 通訊應用）   

- 

  **開發團隊：** 趙子佾、詹詠翔、詹育晟   

- 

  **開源代碼庫：** [GitHub Repository](https://github.com/Embedded-in-your-heart/Intelligent-home)   

## 2. 技術與功能總覽（給組員）

> 新組員 5 分鐘速覽：系統在做什麼、用了哪些技術、目前支援哪些感測功能、要看什麼文件。

### 2.1 系統架構

系統以一台 Raspberry Pi 3 作為中央 Hub，透過 BLE 同時管理多個 STM32 感測節點，並以 HTTP/WebSocket 對瀏覽器提供即時儀表板與歷史圖表。

```
  STM32 B-L475E-IOT01A          Raspberry Pi 3             瀏覽器
  ┌──────────────────┐          ┌────────────────┐          ┌─────────┐
  │ FreeRTOS tasks   │  BLE     │ Flask server   │  HTTP /  │ htmx +  │
  │ sensor / audio / │ ◄──────► │ bluepy BLE mgr │ WebSocket│ Chart.js│
  │ imu / ble        │  GATT    │ SQLite DB      │ ◄──────► │ (任意   │
  └──────────────────┘  Notify  └────────────────┘          │  裝置)  │
  （可多台同時連線）    Write                               └─────────┘
```

STM32 廣播 `HOME-XXXX`，RPi 掃描後建立連線，訂閱 Notify 接收感測資料，再透過 SocketIO 即時推送給瀏覽器；控制指令反向由 Web UI 送出 BLE Write。

### 2.2 技術棧

#### STM32 端（韌體）

| 類別 | 選擇 |
| --- | --- |
| 開發板 | STM32 B-L475E-IOT01A（STM32L475VG, Cortex-M4F, 80 MHz） |
| RTOS | FreeRTOS（CMSIS-RTOS2 v2 wrapper） |
| BLE Stack | BlueNRG-MS middleware（X-NUCLEO-IDB05A2 模組，SPI3） |
| 感測器驅動 | X-CUBE-MEMS1（HTS221、LSM6DSL BSP component driver） |
| DSP 函式庫 | CMSIS-DSP V1.6.0（`arm_rfft_fast_f32`、biquad cascade filter） |
| 開發工具 | STM32CubeIDE 1.19.0 / STM32CubeMX / CMake / arm-none-eabi-gcc |

#### RPi 端（伺服器）

| 類別 | 選擇 |
| --- | --- |
| 語言 | Python 3.11 |
| Web 框架 | Flask + Flask-SocketIO（threading 模式） |
| BLE 函式庫 | bluepy（Linux BlueZ） |
| 資料庫 | SQLite（標準函式庫 `sqlite3`） |
| 前端 | htmx + Chart.js + Bootstrap（無前端 build pipeline） |
| 套件管理 / 品質 | uv / ruff / mypy（strict） / pytest |

### 2.3 功能清單（現況）

#### 感測頻道（12 條 BLE Characteristic，監控型）

| Characteristic | UUID 短碼 | 資料 | 說明 |
| --- | --- | --- | --- |
| Temperature | `1A220002` | float32, °C | 溫度（每 1 s） |
| Humidity | `1A220003` | float32, % RH | 濕度（每 1 s） |
| AccelMagnitude | `1A220004` | float32, g | 加速度量值（每 250 ms） |
| GyroMagnitude | `1A220005` | float32, dps | 陀螺儀量值（每 250 ms） |
| MotionAlert | `1A220006` | uint8, 0/1 | 異常晃動事件 |
| MicLevel | `1A220007` | uint16, 0–1023 | 麥克風正規化能量（每 200 ms） |
| LoudAlert | `1A220008` | uint8, 0/1 | 大聲警示事件 |
| AlarmDetected | `1A22000A` | uint8, 0/1 | 警報音偵測（RFFT 頻帶分析，連續 ≥3 個 200 ms 視窗） |
| MicDBA | `1A22000B` | float32, dB(A) | A-weighting 音量（每 200 ms） |
| VibrationRMS | `1A22000C` | float32, mg | 高通後加速度 RMS，1 s 視窗（家電運轉強度） |
| VibrationAlert | `1A22000D` | uint8, 0/1 | 持續震動警示（家電運轉偵測） |
| QuakeAlert | `1A22000E` | uint8, 0/1 | 地震偵測（1–10 Hz 帶通能量） |

> 注意：SoundClass（UUID `1A220009`，音頻分類枚舉）已從系統移除；上表僅列現行有效頻道。

#### 控制頻道（2 條 BLE Characteristic，控制型）

| Characteristic | UUID 短碼 | 說明 |
| --- | --- | --- |
| Led1State | `1A22F002` | LED1（PA5）開關，`0`=off / `1`=on |
| ControlFlag | `1A22F003` | 通用控制旗標，低 4 bit（預留擴充） |

#### DSP 能力（韌體內建，門檻待實機校正）

- **A-weighting dB(A)**：IEC 61672 3 級 biquad 濾波（fs = 8 kHz），對應人耳感受音量。
- **RFFT 警報音偵測**：每 200 ms 對前 1024 樣本做 Hanning window + `arm_rfft_fast_f32`，在 2.8–3.6 kHz 偵測住宅煙霧警報器單音。
- **IMU 高通 + 帶通濾波**：LSM6DSL 以 104 Hz FIFO 批次採樣（每 250 ms 讀 ≈26 筆）；高通（fc ≈ 0.4 Hz）去重力後算 VibrationRMS；1–10 Hz 帶通能量用於 QuakeAlert。

#### Web 功能

- 即時儀表板：所有裝置頻道當前值，SocketIO 推播。
- 折線圖 + 統計（平均 / 最大 / 最小），可選觀測時間窗。
- `0/1` 旗標頻道：LED 開關 + 瀏覽器 toast 通知。
- `enum:` 徽章：通用機制，`unit` 欄位以 `enum:0=標籤,...` 格式定義，dashboard 即時更新狀態標籤（SoundClass 移除後目前無 preset 使用，但機制保留）。
- 控制型頻道 0/1 開關按鈕。
- 頻道 preset 目錄（14 筆，對應全部現行 Characteristic）。
- 裝置掃描 / 自訂名稱 / 卡片摺疊。
- 跨裝置 analytics 頁。
- mock 模式：免硬體即可跑完整 demo（`MockBLEManager`）。
- 開機自動啟動：`task set-start` 寫入 root crontab `@reboot`，搭配 `boot.log` 監控。

### 2.4 文件地圖

| 文件 | 看什麼 |
| --- | --- |
| **本文件**（主文件） | 系統架構總覽、BLE 頻道模型、開發階段規劃 |
| [STM32 Client 開發文件](./STM32%20Client%20開發文件.md) | 韌體架構、GATT 表（契約）、FreeRTOS task 設計、感測器整合、進度狀態 |
| [RPi-Server 開發文件](./RPi-Server%20開發文件.md) | Flask 分層架構、BLE Manager、DB schema、Web 路由、SocketIO 事件、測試策略 |
| [音訊 DSP 擴充計劃](./音訊%20DSP%20擴充計劃%20(Audio%20DSP%20Expansion%20Plan).md) | AlarmDetected / MicDBA 的 RFFT + A-weighting 演算法細節與校正說明 |
| [IMU 震動 DSP 擴充計劃](./IMU%20震動%20DSP%20擴充計劃%20(IMU%20Vibration%20DSP%20Plan).md) | VibrationRMS / VibrationAlert / QuakeAlert 的 FIFO + 帶通濾波演算法細節 |
| `Intelligent-home-STM32-client/README.md` | STM32 端快速上手：GATT 表速查、nRF Connect 驗證步驟、Python bluepy 範例 |
| `Intelligent-home-RPi-server/README.md` | RPi 端快速上手：環境安裝、Taskfile 指令、開機自動啟動設定 |

---

## 3. 核心概念與挑戰

### 3.1 異質設備整合

本學期常用的 **STM32 開發板**具備豐富的嵌入式感測器，功能多元，但在網路連線與建立伺服器（Server）方面的能力較為薄弱 。為了克服此限制，本項目將建立一個**通用的中央伺服器**，用以動態管理多個感測節點 。  

### 3.2 通訊效能與省電優化

採用 **「中央伺服器 + 藍牙 BLE」** 的架構 ：  

- 使用者可透過伺服器進行遠端控制 。  
- 家中的周邊設備（感測節點）不需各自連接網際網路，而是統一透過低功耗藍牙（BLE）與中央伺服器通訊，達到顯著的省電目的 。  

## 4. 系統硬體架構與部署

系統採用「一對多」的分散式節點佈署架構，允許一台 Raspberry Pi 主機同時管理部署在不同空間（如：客廳、廚房）的多個 STM32 節點 。  

### 4.1 伺服器核心：Raspberry Pi 3

作為系統的中央調度中心，負責以下核心功能 ：  

- 

  **使用者管理介面：** 提供使用者網頁 UI，用以新增設備、定義控制頻道與監控頻道 。  

- 

  **資料持久化：** 本地端儲存裝置清單、頻道相關設定以及歷史監控數據 。  

- 

  **中央調度：** 負責主動掃描、連接並維護與所有 STM32 BLE 周邊設備的通訊鏈結 。  

### 4.2 感測節點：STM32 B-L475E-IOT01A

作為被控制的周邊設備（Peripheral），利用其豐富的硬體資源 ：  

- 

  **內建感測器：** 包含溫濕度感測器、陀螺儀、麥克風及 NFC 。  

- 

  **BLE 模組：** 負責定時發送廣告封包，並提供穩定的數據廣播與連線能力 。  

- 

  **指令執行：** 接收來自 RPi 的控制命令，執行切換 LED 燈、設定溫度或更新狀態等動作 。  

## 5. 藍牙 BLE 通訊架構與頻道模型

系統通訊完全基於 BLE GATT 規範進行資料讀寫與狀態推播 。  

### 5.1 通訊流程三階段

1. 

   **廣播與探索 (Advertising & Scanning)：** STM32 定期發送廣告（Advertising）封包；RPi 開啟掃描模式（Scanning）以識別並發現這些智慧家庭節點 。  

2. 

   **GATT 連接 (GATT Connection)：** 成功建立連線後，透過 Service 與 Characteristic 進行數據讀寫，實現雙向通訊 。  

3. 

   **通知機制 (Notification)：** 針對監控型頻道，當 STM32 偵測到數值變化時，會利用 **Notify** 機制主動推播數據至 RPi，無需 RPi 持續輪詢 。  

### 5.2 頻道管理模型 (Channel Model)

裝置加入後可動態建立兩種類型的頻道 ：  

| **頻道類型**            | **藍牙底層行為**                  | **應用情境與實例**                                           |
| ----------------------- | --------------------------------- | ------------------------------------------------------------ |
| **控制型 (Controller)** | 伺服器發送 **Write** 指令         | • 燈光開關切換    • 溫度高低設定                             |
| **監控型 (Display)**    | 伺服器接收 **Read** 或 **Notify** | • 室內溫濕度偵測    • 異常晃動偵測 (陀螺儀)    • 大小聲偵測 (麥克風) |

註：未來預留複合型（Hybrid）頻道的擴充空間，可同時支援控制與顯示 。  

## 6. 伺服器端軟體邏輯

1. 

   **動態新增裝置：** 伺服器掃描周遭 UUID，使用者從網頁 UI 選擇特定的 STM32 UUID 後，將其記錄至本地資料庫 。  

2. 

   **動態新增頻道：** 裝置建立後，使用者可自由為該裝置配置「控制」或「監控」頻道 。  

3. 

   **資料解析與轉換：** 伺服器負責將 BLE 傳來的原始二進位數據（Raw Data），轉換為人類可讀的溫濕度數值或開關狀態 。  

4. 

   **歷史記錄儲存：** 監控數據自動寫入資料庫，以供前端繪製連續性的歷史趨勢圖表（例如溫濕度變化） 。  

## 7. 開發環境與工具鏈 (預定作法)

### 7.1 韌體開發 (STM32)

- 

  **開發工具：** STM32CubeIDE   

- 

  **軟體庫：** ST HAL 庫   

- 

  **核心任務：** 開發並設定 BLE GATT Service、Characteristic，以及處理感測器數據採集與週邊控制邏輯 。  

### 7.2 通訊協議與伺服器 (Raspberry Pi)

- 

  **底層協議棧：** Linux BlueZ 協定棧   

- 

  **開發套件：** Python `bluepy` 庫（用於 RPi 的 BLE 控制與連線管理）   

### 7.3 前端與使用者介面

- 

  **網頁框架：** Python Flask   

- 

  **功能：** 提供簡易的 Web Dashboard，動態顯示即時環境數據狀態、繪製趨勢圖，並供使用者下達控制指令 。  

## 8. 專案開發進度表 (Development Phases)

> **RPi Server 端目前進度（2026-05-27）：** Phase 1–2，以及 Phase 3 的伺服器骨架、BLE 層、DB repository、Service 層與認證系統（註冊／登入／登出）已完成並通過測試（103 unit tests）；尚待裝置／頻道管理頁面、SocketIO 即時推播與前端趨勢圖。細項見 [RPi-Server 開發文件 §11.1](./RPi-Server%20開發文件.md)。

本項目預計分為四個階段逐步實作 ：  

- 

  **Phase 1: 硬體環境搭建與基礎測試**   

  - 搭建開發環境。
  - 完成 STM32 基礎感測器驅動與 BLE 韌體廣播/通訊功能測試 。  

- 

  **Phase 2: BLE 通訊鏈結實作**   

  - 實作 Raspberry Pi 的 BLE 掃描與自動連線邏輯 。  
  - 成功完成 GATT 特徵值（Characteristic）的讀（Read）、寫（Write）與通知（Notify）通訊 。  

- 

  **Phase 3: 伺服器與網頁端整合**   

  - 開發 Flask 伺服器端的頻道管理模型 。  
  - 整合本地資料庫與前端 UI，完成資料持久化與趨勢圖表展示 。  

- 

  **Phase 4: 全系統整合測試與優化**   

  - 進行多節點分散式佈署測試 。  
  - 系統穩定性優化，並準備專案成果展示 。  