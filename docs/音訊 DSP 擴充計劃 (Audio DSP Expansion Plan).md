# 音訊 DSP 擴充計劃 (Audio DSP Expansion Plan)

> 對應主文件：[智能家庭控制系統開發文件](./智能家庭控制系統開發文件%20(System%20Development%20Document).md)
> 狀態：**實作進行中**（2026-06-11，平行 multi-agent workflow）
> 分支：STM32 `feat/audio-dsp`、RPi-server `feat/sound-dsp-ui`（獨立 worktree，避開另一條進行中的 dashboard 摺疊功能）

---

## 1. 目標

把 STM32 端「DMA 已搬進記憶體、但只算一個 RMS」的 8 kHz 音訊串流升級為有頻譜理解的聲音感測：

1. **A-weighting → dB(A) 音量**：MicLevel 之外提供貼近人耳感受的音量估計值。
2. **RFFT 頻帶分析 → 聲音分類**：每 200 ms 視窗區分 安靜/語音/拍手/警報/其他。
3. **警報音偵測**：住宅煙霧警報器單音（約 3.1–3.5 kHz）持續出現時主動推播事件。
4. **RPi 端配套**：新頻道 preset、enum 類別徽章 UI、mock feed demo 資料、文件同步。

**不包含（下一個 milestone）**：IMU 升頻 + 震動 DSP（家電運轉/地震偵測）。理由：需重構 sensor_task 採樣架構（LSM6DSL FIFO 批次讀），與本輪關注點獨立；本輪完成後 CMSIS-DSP 基礎設施可直接重用。

## 2. BLE 契約（已釘死，agents 依此實作）

Home Sensor Service：

| Name | 短碼 | 格式 | 屬性 | 推播時機 | 狀態 | 語意 |
| --- | --- | --- | --- | --- | --- | --- |
| AlarmDetected | `1A22000A` | `uint8` | Read+Notify | 事件觸發 | ✅ 保留 | 連續 ≥3 個 200 ms 視窗判定警報音 → 1；連續 ≥5 個非警報視窗 → 0；轉換間鎖 2 s |
| MicDBA | `1A22000B` | `float32_le` | Read+Notify | 每 200 ms | ✅ 保留 | dB(A) 估計值：`20·log10(max(rms_w,1)) + OFFSET`（OFFSET 預設 30，**待實機校正**） |
| SoundClass | `1A220009` | `uint8` | Read+Notify | 變化時 | ⛔ 已移除（2026-06-11，分類器簡化為警報音偵測） | ~~0=安靜 1=語音 2=拍手 3=警報 4=其他~~ |

既有 MicLevel / LoudAlert 契約**完全不變**（校正常數沿用）。

RPi 端新慣例：`unit` 以 `enum:` 開頭（如 `enum:0=安靜,1=語音,2=拍手,3=警報,4=其他`）的 display 頻道以**狀態徽章**呈現，初始值取自新端點 `GET /channels/<id>/latest`，即時更新走既有 SocketIO `reading` 事件。

## 3. STM32 實作架構

- **建置**（已完成並通過編譯）：vendor CMSIS-DSP V1.6.0（取自 `.ioc` 釘選的 X-CUBE-MEMS1 11.3.0 pack：`arm_math.h` + `libarm_cortexM4lf_math.a`），還原 CubeMX 本來就生成的連結設定。
- **新模組 `BlueNRG_MS/App/audio_dsp.{c,h}`**：靜態緩衝區（約 14 KB BSS）；每 200 ms 半緩衝區（1600 樣本）做：
  - 全 1600 樣本 → A-weighting 3 級 biquad（fs=8 kHz，IEC 61672 雙線性轉換係數）→ 加權 RMS → dB(A)。
  - 前 1024 樣本 → Hanning + `arm_rfft_fast_f32` → 頻帶能量（bin 7.8125 Hz）。
  - 規則分類器（常數標示 calibration-pending）：安靜=能量門檻；警報=2.8–3.6 kHz 窄頻峰值比 >0.30；拍手=RMS 突跳 4×；語音=中頻佔比 >0.55。
- **`gatt_db` / `notify_queue`**：三條 char 註冊（attr 數 `1+3*7` → `1+3*10`）、`HomeCharId` 尾端附加、dispatch switch 擴充。
- **`audio_task.c`**：既有 RMS/MicLevel/LoudAlert 路徑不動；新增 DSP 呼叫與 AlarmDetected 狀態機（鏡射 LoudAlert 模式）；stack 1024→1536。

## 4. RPi 實作架構

- `services/units.py`：`parse_enum_unit()` 解析 enum unit 字串。
- `GET /channels/<id>/latest`：比照 `/flag` 端點風格（含登入保護）。
- 模板（index + detail）display 分支新增 enum 徽章 widget（**最小侵入**，避免與另一 session 的摺疊功能衝突）。
- `dashboard.js`：`enums` registry（比照 `flags`），badge 以 `textContent` 更新防 XSS。
- `data/channel_presets.json`：+3 筆（聲音類別 / 警報聲偵測 / 音量 dB(A)）——警報聲偵測沿用 `0/1` unit，**自動獲得**既有 LED + toast 通知 UI。
- Mock feed：enum 頻道週期性輸出非零類別、dBA 頻道 40±8 正弦＋週期尖峰，無硬體即可 demo。

## 5. 驗證標準與流程

| 項目 | 標準 |
| --- | --- |
| STM32 | `cmake --preset Debug` 編譯零錯誤零警告（新增檔案範圍）；RAM/FLASH 用量回報 |
| RPi | `ruff check` + `mypy src`（strict）+ `pytest` 全綠 |
| 審查 | 每 repo 一名獨立 reviewer 對抗式審查（GATT attr 數、float 序列化、緩衝區邊界、biquad 穩定性、XSS、endpoint 形狀），確認的 medium+ 發現修正後重新驗證 |
| 實機（後續，需硬體） | 沿用「先 serial 再 BLE」：VCP 觀察 `[mic] dba/class`，以拍手、語音、手機播警報聲校正門檻常數；nRF Connect 訂閱三條新 char |

## 6. 風險與待決

| 項目 | 處理 |
| --- | --- |
| 分類門檻 / dB(A) offset 為紙上預估 | 常數集中於 `audio_dsp.c` 並標示 calibration-pending；實機校正為獨立後續步驟 |
| A-weighting 係數正確性 | reviewer 驗證極點在單位圓內與 CMSIS 係數正負號慣例 |
| 與另一 session 的 `index.html`/`dashboard.js` 合併衝突 | 本輪變更刻意 additive；合併順序由使用者決定（建議摺疊功能先進 main，本分支 rebase） |
| 8 kHz 取樣上限 | Nyquist 4 kHz，涵蓋警報音；無法分析更高頻成分（接受） |
