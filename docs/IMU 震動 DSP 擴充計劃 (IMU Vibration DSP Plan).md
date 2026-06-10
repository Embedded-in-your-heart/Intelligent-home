# IMU 震動 DSP 擴充計劃 (IMU Vibration DSP Plan)

> 對應主文件：[智能家庭控制系統開發文件](./智能家庭控制系統開發文件%20(System%20Development%20Document).md)
> 前置：[音訊 DSP 擴充計劃](./音訊%20DSP%20擴充計劃%20(Audio%20DSP%20Expansion%20Plan).md)（已完成，CMSIS-DSP 基礎設施可直接重用）
> 狀態：**已實作，待實機校正**（2026-06-11 核准並完成；STM32 `feat/imu-dsp` 已併入 main。
> 實機必驗：CTRL1_XL ODR 讀回、FIFO_CTRL3/5 設定讀回、首次 FIFO dump 的 X/Y/Z 槽序、
> 門檻常數 VIB_ON 30 mg / VIB_OFF 15 mg / QUAKE 20 mg 校正；開機時注意 `[imu] FIFO empty` 退回訊息）

---

## 1. 目標

把 LSM6DSL 從「4 Hz 讀 magnitude」升級為有頻域理解的震動感測：

1. **線性加速度**：每軸 high-pass 去除重力分量，讓偵測不受板子擺放姿勢影響。
2. **震動強度**：貼附家電（洗衣機、壓縮機）運轉與否的偵測。
3. **地震偵測**：1–10 Hz 低頻持續振盪的辨識。

既有 `AccelMagnitude` / `GyroMagnitude` / `MotionAlert` 契約**完全不變**。

## 2. BLE 契約草案（核准後即釘死）

Home Sensor Service 新增三條 characteristic（接續音訊 DSP 的編號）：

| Name | 短碼 | 格式 | 屬性 | 推播時機 | 語意 |
| --- | --- | --- | --- | --- | --- |
| VibrationRMS | `1A22000C` | `float32_le` | Read+Notify | 每 1 s | 高通後線性加速度的 1 s 視窗 RMS（mg） |
| VibrationAlert | `1A22000D` | `uint8` | Read+Notify | 事件觸發 | 持續震動（家電運轉）：VibrationRMS 高於門檻連續 ≥5 s → 1；低於門檻連續 ≥10 s → 0 |
| QuakeAlert | `1A22000E` | `uint8` | Read+Notify | 事件觸發 | 1–10 Hz 帶通能量持續 ≥3 s 高於門檻 → 1；hold/lockout 比照 LoudAlert 模式 |

RPi 端：presets +3 筆（震動強度 / 家電運轉 / 地震警報），後兩條沿用 `0/1` unit，自動獲得 LED + toast 通知 UI；無需新前端機制。

## 3. STM32 實作架構

- **採樣升頻**：LSM6DSL accel ODR 104 Hz + 內建 FIFO；SensorTask 維持 250 ms 週期，但每 tick 從 FIFO 批次讀 ~26 筆樣本（一次 I²C burst，避免中斷風暴）。HTS221 1 Hz 與 gyro 4 Hz 路徑不變。
- **新模組 `BlueNRG_MS/App/imu_dsp.{c,h}`**（比照 `audio_dsp` 模式）：
  - 每軸 high-pass biquad（fc ≈ 0.4 Hz @ 104 Hz）→ 線性加速度 → magnitude。
  - **不用 FFT**：地震特徵以 1–10 Hz band-pass biquad + 持續能量規則判斷——RAM 已用 75%，避免再吃大緩衝區（FFT 版留作 future work；若必要可把 audio 緩衝區移 RAM2 32 KB 騰空間）。
  - 1 s 滑動視窗 RMS（mg）→ VibrationRMS；兩個 alert 狀態機沿用 LoudAlert 的 hold/lockout 樣式。
- **記憶體預算**：biquad 狀態 + 1 s 視窗累加器 < 1 KB，遠小於音訊 DSP；SensorTask stack 768 B 需重新評估（FIFO 批次 + 濾波，預估 bump 至 1 KB）。

## 4. 驗證

1. **Serial 階段**：印 `[imu] lin_mag / vib_rms / band_e`；測試：靜置（≈0）、改變擺放姿勢（高通後不應觸發）、放在運轉中的洗衣機/風扇上、手動模擬地震（低頻晃桌 vs 高頻敲擊應可區分）。
2. **BLE 階段**：nRF Connect 訂閱三條新 char；RPi mock feed 加對應合成資料。
3. CI：STM32 clean build；RPi ruff + mypy + pytest。

## 5. 風險與待決

| 項目 | 處理 |
| --- | --- |
| FIFO 讀取與 HTS221 共用 I²C2 | 同一 task 序列化存取（現行模式），實測 burst 讀耗時 |
| 104 Hz 下 0.4 Hz HPF 的安定時間（~數秒） | 開機後前 5 s 抑制 alert（warm-up 旗標） |
| 門檻純屬預估 | 比照音訊 DSP：常數集中、標示 calibration-pending，實機校正 |
| `.ioc` 是否需改 | 預期不用（I²C2 polling 已啟用，ODR/FIFO 是暫存器層設定）；若需改走 .ioc 流程 |
