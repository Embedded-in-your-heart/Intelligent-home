# 智能家庭控制系統

> 基於 Raspberry Pi 3 與 STM32 之 BLE 通訊應用  
> **課程**：Embedded Systems Lab｜**團隊**：趙子佾、詹詠翔、詹育晟

完全本地端運作的智慧家庭系統——STM32 感測節點透過 BLE 傳送即時感測與 DSP 分析結果，Raspberry Pi 作為中央伺服器提供 Web 管理介面，所有資料保留在本地，不依賴任何雲端服務。

---

## 系統架構

```
┌──────────────────┐                      ┌──────────────────────┐
│  STM32 節點 A    │◄────── BLE ─────────►│                      │
│  HOME-XXXX       │                      │  Raspberry Pi 3      │
└──────────────────┘                      │  Flask + SQLite      │
                                          │                      │
┌──────────────────┐                      │  port 5000           │
│  STM32 節點 B    │◄────── BLE ─────────►│                      │
│  HOME-YYYY       │                      └──────────┬───────────┘
└──────────────────┘                                 │ HTTP / WebSocket
                                          ┌──────────▼───────────┐
                                          │     使用者瀏覽器      │
                                          │  HTMX + Chart.js     │
                                          └──────────────────────┘
```

---

## Repository 結構

```
Intelligent-home/                       ← 本 repo（文件 + submodule 管理）
├── docs/
│   ├── 智能家庭控制系統開發文件.md      ← 系統總設計
│   ├── RPi-Server 開發文件.md          ← 伺服器軟體設計
│   ├── STM32 Client 開發文件.md        ← 韌體架構與 GATT 規格
│   └── project_summary_hackmd_v2.md   ← 期末報告（最新版）
├── slides/
│   └── progress_report.pdf
├── Intelligent-home-RPi-server/        ← submodule（Python / Flask 伺服器）
└── Intelligent-home-STM32-client/      ← submodule（STM32 C 韌體）
```

---

## 功能總覽

### 感測器監控（即時折線圖）

| 頻道 | 來源 | 格式 | 更新頻率 |
|---|---|---|---|
| 溫度（°C）| HTS221 | float32 | 1 s |
| 濕度（% RH）| HTS221 | float32 | 1 s |
| 加速度量值（g）| LSM6DSL 104 Hz FIFO | float32 | 250 ms |
| 陀螺儀量值（dps）| LSM6DSL | float32 | 250 ms |
| 麥克風音量（0–1023）| MP34DT01 DFSDM+DMA | uint16 | 200 ms |
| 音量 dB(A) | Audio DSP（A-weighting IIR）| float32 | 200 ms |
| 震動強度 RMS（mg）| IMU DSP（HP biquad）| float32 | 1 s |

### 警報偵測（即時 Toast 通知）

| 警報 | 觸發條件 | Demo 方式 |
|---|---|---|
| 異常晃動 | 加速度 > 1.8 g 或陀螺儀 > 250 dps，持續 100 ms | 搖晃板子 |
| 大聲警示 | 麥克風音量 > 400，持續 200 ms | 拍手 / 大喊 |
| 警報聲偵測 | FFT peak ratio（2800–3600 Hz）> 30% | 播放煙霧警報聲 |
| 家電運轉 | 震動 RMS > 30 mg | 放在運轉中的馬達上 |
| 地震警報 | 1–10 Hz 帶通 RMS > 20 mg | 規律前後搖晃板子 |

### 遠端控制

| 功能 | 操作 | 結果 |
|---|---|---|
| LED1 開關 | 瀏覽器 Toggle | 板子 PA5 LED 亮 / 滅 |
| ControlFlag | 數值輸入送出 | Serial log 印出收到值 |

### Web 系統

- 使用者註冊 / 登入 / 登出（bcrypt 雜湊，CSRF 保護）
- BLE 掃描並新增裝置
- 頻道預設清單（14 個預設，下拉選單快速套用）
- 裝置連線狀態即時徽章
- 斷線自動重連（指數退避 1 s → 60 s）
- 時間窗切換（1m / 10m / 1h / 1d / 1w）
- 支援多裝置同時管理

---

## 快速啟動

### 1. Clone（含 submodule）

```bash
git clone --recurse-submodules https://github.com/Embedded-in-your-heart/Intelligent-home.git
cd Intelligent-home
```

### 2. STM32 韌體燒錄

硬體需求：**STM32 B-L475E-IOT01A** + USB 連電腦

```
1. 用 STM32CubeIDE 開啟 Intelligent-home-STM32-client/
   File → Open Projects from File System → 選資料夾

2. 編譯：Project → Build All（Ctrl+B）

3. 燒錄：Run → Run（F11）

4. 開 Serial Monitor（115200 8N1），確認看到：
   Advertising as HOME-XXXX
   SensorTask started (HTS221=OK, LSM6DSL=OK).
   AudioTask started ...
```

詳細說明見 [STM32 Client 開發文件](docs/STM32%20Client%20開發文件.md)。

### 3. Raspberry Pi 伺服器部署

```bash
# 安裝系統套件
sudo apt update
sudo apt install -y libglib2.0-dev libbluetooth-dev pkg-config

# 安裝 uv
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# 進入伺服器目錄
cd Intelligent-home-RPi-server

# 安裝相依套件
uv sync

# 設定 BLE 權限（避免每次 sudo）
sudo setcap cap_net_raw,cap_net_admin+eip \
  $(uv run python -c "import bluepy, os; print(os.path.join(os.path.dirname(bluepy.__file__), 'bluepy-helper'))")

# 設定環境變數
cp .env.example .env
# 編輯 .env，填入 HOME_SERVER_SECRET_KEY

# 啟動伺服器
uv run python -m home_server
```

瀏覽器開啟 `http://<RPi的IP>:5000`

### 4. 新增裝置與頻道

1. 註冊帳號並登入
2. Devices → **Scan** → 選擇 `HOME-XXXX` → 填名稱 → Add Device
3. 進入裝置頁 → Add Channel → 從下拉選單選預設頻道（例如「溫度 (°C)」）
4. 回 Dashboard，即時折線圖開始更新

---

## GATT 表（BLE 對外契約）

Base UUID：`xxxxxxxx-8E22-4541-9D4C-21EDAE82ED19`

| Characteristic | UUID 短碼 | 屬性 | 格式 |
|---|---|---|---|
| Temperature | `1A220002` | Read + Notify | float32_le |
| Humidity | `1A220003` | Read + Notify | float32_le |
| AccelMagnitude | `1A220004` | Read + Notify | float32_le |
| GyroMagnitude | `1A220005` | Read + Notify | float32_le |
| MotionAlert | `1A220006` | Read + Notify | uint8 |
| MicLevel | `1A220007` | Read + Notify | uint16_le |
| LoudAlert | `1A220008` | Read + Notify | uint8 |
| AlarmDetected | `1A22000A` | Read + Notify | uint8 |
| MicDBA | `1A22000B` | Read + Notify | float32_le |
| VibrationRMS | `1A22000C` | Read + Notify | float32_le |
| VibrationAlert | `1A22000D` | Read + Notify | uint8 |
| QuakeAlert | `1A22000E` | Read + Notify | uint8 |
| Led1State | `1A22F002` | Read + Write | uint8 |
| ControlFlag | `1A22F003` | Read + Write | uint8 |

---

## 技術棧

### STM32 韌體
- **MCU**：STM32L475VG（Cortex-M4F, 80 MHz），**BLE**：BlueNRG-MS
- **OS**：FreeRTOS（CMSIS-RTOS2）— BleTask / SensorTask / AudioTask
- **DSP**：CMSIS-DSP V1.6.0（`arm_rfft_fast_f32`、`arm_biquad_cascade_df1_f32`）
- **build**：STM32CubeIDE 1.19.0 / CMake preset

### RPi 伺服器
- **語言**：Python 3.11，**套件管理**：uv
- **Web**：Flask + Flask-SocketIO（threading mode）+ Flask-Login + Flask-WTF
- **BLE**：bluepy（Linux only；開發環境自動 fallback MockBLEManager）
- **DB**：SQLite（WAL 模式）
- **前端**：Jinja2 + HTMX + Bootstrap 5 + Chart.js（全部 vendor，無 CDN 依賴）
- **測試**：pytest（190+ tests）；ruff + mypy strict 全綠

---

## 專案狀態

| 項目 | 狀態 |
|---|---|
| STM32 韌體（BLE + 感測器 + DSP）| ✅ 完成 |
| RPi Web 伺服器（Phase 1–3e）| ✅ 完成 |
| 即時 Dashboard（折線圖 + 警報 Toast）| ✅ 完成 |
| 190+ unit tests，ruff / mypy 全綠 | ✅ 完成 |
| Phase 4：多節點長時間穩定性 + systemd 部署 | 🚧 進行中 |

---

## 相關文件

| 文件 | 說明 |
|---|---|
| [系統總設計](docs/智能家庭控制系統開發文件%20(System%20Development%20Document).md) | 架構概覽、BLE 通訊模型 |
| [RPi Server 開發文件](docs/RPi-Server%20開發文件.md) | 伺服器分層設計、API、部署 |
| [STM32 Client 開發文件](docs/STM32%20Client%20開發文件.md) | 韌體架構、GATT 規格、FreeRTOS 任務模型 |
| [期末報告（v2）](docs/project_summary_hackmd_v2.md) | 完整系統說明，含 DSP 技術細節 |
| [RPi Server README](Intelligent-home-RPi-server/README.md) | 伺服器快速啟動指令 |
| [STM32 Client README](Intelligent-home-STM32-client/README.md) | 韌體編譯、燒錄、驗證 |
