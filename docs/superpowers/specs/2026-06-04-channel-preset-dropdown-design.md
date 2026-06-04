# Channel 功能預設下拉（取代手動輸入 uuid/type/unit）設計文件

> 對應原始碼目錄：`Intelligent-home-RPi-server/src/home_server/`
> 日期：2026-06-04
> 前置：Phase 3d（device/channel CRUD）已完成；「Add channel」表單目前要求使用者手動輸入 `char_uuid`、`unit` 並從下拉選 `type` / `data_format`。
> 緣由：使用者新增 channel 時不應需要知道 BLE characteristic UUID、編碼格式或單位。STM32 GATT 表（`docs/STM32 Client 開發文件.md` §5.4）在燒錄後即固定，RPi 端「新增頻道」本質上只是從一份已知清單中挑一個 characteristic。改以單一「功能」下拉，讓使用者只選功能 + 取名即可。

---

## 1. 目標與範圍

把「Add channel」表單從 5 個技術欄位（name / type / char_uuid / data_format / unit）簡化為 **2 個欄位**：

- **Name**：自由輸入；選功能時自動帶入該功能 label，使用者可改；維持必填。
- **功能（Function）**：單一下拉。選定後 `char_uuid` / `type` / `data_format` / `unit` 由該功能的預設值決定。

功能清單來自可編輯的 JSON 設定檔，初始內容對齊 STM32 GATT 表。**資料庫 schema、`channels` 資料表、service 與 db 層完全不變**——preset 只是在「建立 channel」時提供原本由表單帶入的四個值。

**驗收標準**：
- 開發機（macOS）：`task lint`（ruff check + mypy strict）+ `task test`（pytest）全綠。
- `load_presets` 有合法 / 非法（壞 type、未知 format、重複 label/uuid、缺檔、JSON 格式錯）單元測試。
- `test_web_channels.py` 的新增流程改用 `preset`（char_uuid）+ `name`，並驗證建立後的 channel 帶有正確的 uuid/type/format/unit；未知 preset 被擋。
- 手動煙測（不自動化）：開啟裝置詳情頁，下拉可選功能、選後 Name 自動帶入、送出後 channel 出現在列表並帶正確 UUID/格式/單位。

### 1.1 範圍內

- `config.py`：新增 `channel_presets_path`（env `HOME_SERVER_CHANNEL_PRESETS_PATH`，預設 `./data/channel_presets.json`）。
- 新模組 `home_server/presets.py`：`ChannelPreset` dataclass + `load_presets(path) -> list[ChannelPreset]`，含驗證。
- `web/__init__.py`：啟動時載入一次，存入 `app.config["CHANNEL_PRESETS"]`。
- `web/channels.py`：`AddChannelForm` 改為 `name` + `preset`（`SelectField`）；route 從 preset 解析出四個值再呼叫既有 `add_channel(...)`。
- `templates/devices/detail.html`：表單改為 name + 功能下拉 + 一段極小 vanilla JS（選功能自動帶入 Name）。
- `data/channel_presets.json`：初始功能清單（納入 git）。
- 測試：新增 `tests/test_presets.py`；更新 `tests/test_web_channels.py` 與 `tests/conftest.py`。

### 1.2 範圍外（YAGNI）

- preset 的管理 UI / DB 資料表（決定用 JSON 設定檔，不做 CRUD 介面）。
- 由連線裝置動態 GATT 探索來產生清單（`BLEManager` 目前無 `discover_characteristics`，不新增）。
- 既有 channel 的回溯遷移（schema 不變，舊資料照常運作）。
- 「同一 preset 不可在同裝置重複新增」之額外限制——沿用現有 `UNIQUE(device_id, name)`：重複加同一功能會因名稱重複被擋，使用者改名即可。
- `data_format` 仍維持下拉的舊行為（被 preset 取代，不再單獨出現在表單）。

---

## 2. 現況盤點

讀過程式碼後確認資料流：

`templates/devices/detail.html`「Add channel」表單 POST `name`+`type`+`char_uuid`+`data_format`+`unit` → `web/channels.py::add_channel`（`AddChannelForm`，五欄）→ `ChannelService.add_channel(...)` → `db.channels.create(...)` 寫入 `channels` 表（欄位 `name/type/char_uuid/data_format/unit`）。

- `db/channels.py`、`services/channel_service.py`、`db/schema.sql`：四個值原樣儲存與使用，**本次不動**。
- `ble/parser.py::supported_formats()`：合法 `data_format` 的單一真實來源（`uint8` / `int16_le` / `float32_le` … 共 10 種）。preset 驗證即以此為準。
- `config.py`：純 env 載入的 frozen `Config`；`data/` 目錄已存在（放 `home.db`、`.gitkeep`）。
- `web/__init__.py::create_app`：應用工廠；已有把服務塞進 `app.extensions` / `app.config` 的慣例。
- `tests/conftest.py`：`app` fixture 以 `tmp_path` 建 DB 並組 `Config`；新增 `channel_presets_path` 需在此設定（指向測試用 JSON）。

---

## 3. 設計細節

### 3.1 `home_server/presets.py`（新模組）

```python
from home_server.db.channels import ChannelType  # Literal["controller", "display"]

class PresetError(ValueError): ...

@dataclass(frozen=True)
class ChannelPreset:
    label: str
    char_uuid: str
    type: ChannelType
    data_format: str
    unit: str | None

def load_presets(path: Path) -> list[ChannelPreset]: ...
```

`load_presets` 行為：
1. 讀檔；缺檔或 JSON 解析失敗 → `PresetError`（訊息含路徑）。
2. 頂層必須是 list；每筆必須是 object，含 `label` / `char_uuid` / `type` / `data_format`，`unit` 可省略（視為 `None`）。
3. 逐筆驗證：`type ∈ {"controller", "display"}`；`data_format ∈ parser.supported_formats()`；`label`、`char_uuid` 為非空字串。任一違反 → `PresetError`（含該筆 index 與原因）。
4. `label` 與 `char_uuid` 在清單內各自唯一（重複 → `PresetError`）。`char_uuid` 作為下拉選項值，故必須唯一。
5. 回傳 `list[ChannelPreset]`（保留 JSON 順序，即下拉顯示順序）。

### 3.2 `config.py`

`Config` 新增欄位 `channel_presets_path: Path`，`from_env` 以 `HOME_SERVER_CHANNEL_PRESETS_PATH`（預設 `./data/channel_presets.json`）載入。`.env.example` 補上對應註解。

### 3.3 `web/__init__.py`

`create_app` 內、註冊 blueprint 前，呼叫 `load_presets(config.channel_presets_path)` 一次，結果存入 `app.config["CHANNEL_PRESETS"]`（型別 `list[ChannelPreset]`）。載入失敗會在啟動時直接拋錯（fail-fast），符合「設定檔錯就別啟動」。

### 3.4 `web/channels.py`

`AddChannelForm` 改為：
```python
class AddChannelForm(FlaskForm):
    name = StringField("Name", validators=[DataRequired()])
    preset = SelectField("Function", validators=[DataRequired()])
```
- `add_channel` route 內，先以 `app.config["CHANNEL_PRESETS"]` 設定 `form.preset.choices = [(p.char_uuid, p.label) for p in presets]`（須在 `validate_on_submit()` 前設定，否則 `SelectField` 的內建 choices 驗證會擋掉所有值）。
- 驗證通過後，以提交的 `form.preset.data`（char_uuid）在 presets 中找對應 `ChannelPreset`；找不到 → flash 錯誤並重新渲染（理論上 choices 驗證已擋下，這是防禦性處理）。
- 呼叫既有 `get_channel_service().add_channel(conn, device_id=..., name=form.name.data, type=preset.type, char_uuid=preset.char_uuid, data_format=preset.data_format, unit=preset.unit)`。
- `DuplicateChannelNameError` 維持現有 flash 行為。
- 重新渲染 `devices/detail.html` 時改傳 `presets=presets`（取代原本的 `formats=...`）。

> 同一頁有兩個渲染點：`detail` route（`web/devices.py:53`，目前傳 `formats=...`）與本 route 的重新渲染。兩處都需把 `formats` 換成 `presets`（`formats` 變成未使用，一併移除）。

### 3.5 `templates/devices/detail.html`

「Add channel」表單 region 改為：
```html
<input id="channel-name" name="name" placeholder="名稱" required>
<select id="channel-preset" name="preset">
  {% for p in presets %}
  <option value="{{ p.char_uuid }}" data-label="{{ p.label }}">{{ p.label }}</option>
  {% endfor %}
</select>
<button type="submit">Add</button>
```
- presets 為空：下拉與按鈕停用，顯示提示「尚未設定任何功能，請編輯 data/channel_presets.json」。
- 極小 vanilla JS（inline 於本頁，約 12 行）：監聽 `#channel-preset` 的 `change`，把選項的 `data-label` 帶入 `#channel-name`；若使用者已手動編輯過 Name（`input` 事件設旗標）則不覆蓋；載入時先帶一次。
- 既有「Channels」表格與「Live & control」區塊不變（仍顯示已儲存的 char_uuid / data_format / unit）。

### 3.6 `data/channel_presets.json`（初始內容）

對齊 STM32 GATT 表（§5.4）。Base UUID `…-8E22-4541-9D4C-21EDAE82ED19`，short ID 為各列前 4 byte。

| label | char_uuid（短碼） | type | data_format | unit |
| --- | --- | --- | --- | --- |
| 溫度 (°C) | `1A220002` | display | `float32_le` | `°C` |
| 濕度 (% RH) | `1A220003` | display | `float32_le` | `% RH` |
| 加速度量值 (g) | `1A220004` | display | `float32_le` | `g` |
| 陀螺儀量值 (dps) | `1A220005` | display | `float32_le` | `dps` |
| 異常晃動警示 | `1A220006` | display | `uint8` | （無） |
| 麥克風音量 | `1A220007` | display | `uint16_le` | （無） |
| 大聲警示 | `1A220008` | display | `uint8` | （無） |
| LED1 開關 | `1A22F002` | controller | `uint8` | （無） |
| 控制旗標 | `1A22F003` | controller | `uint8` | （無） |

JSON 內 `char_uuid` 寫完整 128-bit（如 `1A220002-8E22-4541-9D4C-21EDAE82ED19`）；無單位者 `unit` 省略或 `null`。
> 註：STM32 文件標示「UUID 待最終定」；humidity 在 RPi 文件曾提 `uint16_le ×100`，此處以 GATT 表（§5.4）的 `float32_le` 為準。最終值確定後直接編輯本 JSON 即可，無需改程式。

### 3.7 git / `.gitignore`

已確認 `.gitignore` 只排除 `data/*.db` / `*.db-journal` / `*.db-wal` / `*.db-shm`，未排除 `data/channel_presets.json`；此檔需納入版控。

---

## 4. 錯誤處理

- 提交未知 preset（竄改請求）：`SelectField` 內建 choices 驗證擋下 → 重新渲染，不 500。
- presets 設定檔缺檔 / 格式錯：`create_app` 啟動即 `PresetError`，不進入服務狀態。
- presets 清單為空（合法的空 list）：頁面下拉停用 + 提示，不會誤建空 channel。

---

## 5. 測試

- **`tests/test_presets.py`（新增）**：`load_presets` 的合法清單；非法案例（壞 `type`、未知 `data_format`、重複 `label`、重複 `char_uuid`、缺 `label`、缺檔、JSON 格式錯）各自斷言 `PresetError`；`unit` 省略 → `None`。
- **`tests/conftest.py`（更新）**：`app` fixture 寫一份小的測試用 presets JSON 到 `tmp_path` 並設入 `Config.channel_presets_path`，使 web 測試自足、不依賴正式 `data/channel_presets.json` 內容。
- **`tests/test_web_channels.py`（更新）**：
  - 新增 channel 改 POST `{name, preset=<某 preset 的 char_uuid>}`；斷言建立後的 channel 其 `char_uuid/type/data_format/unit` 等於該 preset。
  - 未知 preset 值 → 表單驗證失敗 / 重新渲染（不建立、不 500）。
  - 重複名稱 flash 行為沿用（用同 preset 兩次、名稱相同 → flash）。
- **驗收**：`task lint` + `task test` 全綠（CI = ruff check + mypy + pytest；不跑 `ruff format`）。

---

## 6. 對外介面變更摘要

| 項目 | 變更 |
| --- | --- |
| `POST /devices/<id>/channels` 表單欄位 | `name` + `type` + `char_uuid` + `data_format` + `unit` → **`name` + `preset`** |
| `Config` | 新增 `channel_presets_path` |
| 環境變數 | 新增 `HOME_SERVER_CHANNEL_PRESETS_PATH` |
| 新檔 | `home_server/presets.py`、`data/channel_presets.json`、`tests/test_presets.py` |
| DB schema / `channels` 表 / service / db 層 | **不變** |
