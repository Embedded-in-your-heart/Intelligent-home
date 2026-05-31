# Phase 3d — Device / Channel CRUD Blueprint 設計文件

> 對應主文件：[RPi-Server 開發文件](../../RPi-Server%20開發文件.md) §11.1（子階段 3d）
> 對應原始碼目錄：`Intelligent-home-RPi-server/src/home_server/`
> 日期：2026-05-31

---

## 1. 目標與範圍

實作裝置（device）與頻道（channel）的 CRUD 管理頁，全部 server-render HTML、純整頁重載，沿用既有 auth blueprint 的慣例（Flask-WTF 表單、CSRF、flash 錯誤訊息、`@login_required`）。

**驗收標準**：瀏覽器可完成「登入 → 新增裝置 → 進裝置詳情 → 新增頻道 → 刪除頻道 → 刪除裝置」全流程，無即時數據與圖表。

### 1.1 範圍內（3d）

- 裝置：列表、手動輸入 MAC 新增、詳情頁（含其頻道）、刪除
- 頻道：在裝置詳情頁新增、刪除
- service 層接線（`DeviceService` / `ChannelService`）進 application factory
- 對應單元測試；維持 `ruff check`、`mypy src`（strict）全綠

### 1.2 範圍外（留待 3e）

- `GET /devices/scan`（BLE 掃描，HTMX partial）
- `POST /channels/<id>/write`（控制型頻道指令寫入）
- `GET /channels/<id>/history`（歷史資料 JSON）+ Chart.js 圖表
- SocketIO 即時推播、notify subscribe 接線、自動重連背景迴圈
- 真實 `BluepyManager` 依平台選用、HTMX 漸進增強

---

## 2. 關鍵設計決策

### 2.1 Service 接線（採方案 A）

`DeviceService` / `ChannelService` 是需注入 `BLEManager` 的有狀態類別，與無狀態的 module-level `user_service` 不同。在 `create_app(config, ble=None)` 內建立 BLE manager 與兩個 service，存入 `app.extensions`，blueprint 透過 `current_app` 取用。

- `ble` 參數預設 `MockBLEManager()`（3d 開發機無真實 BLE）；3e 再改為依平台選 `BluepyManager` 並接上 worker thread／自動重連。
- `RateLimiter(config.reading_min_interval)` 建立後注入 `ChannelService`。
- `ChannelService` 的 `on_reading` 在 3d 為 no-op（`lambda *_: None`）；3e 換成 SocketIO emit。
- 不修改 service 層既有契約（surgical change）；3e 只需替換注入的 BLE manager 與 `on_reading`。

被否決的替代方案：

- **module-level 全域單例**：service 需注入 BLE，全域變數隱藏狀態、難測。
- **每請求建立 service**：BLE manager 是長壽連線池，不該綁 request scope。

### 2.2 型別化存取器

新增 `web/services.py`，提供 `get_device_service()` / `get_channel_service()`，集中從 `app.extensions`（`dict[str, Any]`）取出並斷言型別，避免 mypy strict 下到處 cast。仿 `web/db.py` 的 `get_conn()` 模式，避免與 application factory 形成 import cycle。

### 2.3 互動機制：純 POST 表單

新增用標準 HTML `<form>` POST；刪除改用 `POST /devices/<id>/delete`、`POST /channels/<id>/delete`（純 HTML form 只能 GET/POST，故偏離文件路由表的 `DELETE` method 以換取零 JS）。刪除按鈕用裸 `<form>` + `csrf_token()`，同 `base.html` 既有 logout 寫法。3e 再以 HTMX `hx-delete` 漸進增強。

### 2.4 權限模型

沿用文件 §4.3.2 的 flat 模型：所有登入使用者皆可管理所有裝置。

- 新增裝置時 `owner_user_id = current_user.id`。
- 列表用 `devices.list_all`（非 `list_by_owner`）。
- 詳情／刪除不做 owner 檢查。

---

## 3. 路由設計

兩個 blueprint，皆 `@login_required`，CSRF 由 Flask-WTF 保護。

### 3.1 Device blueprint（`web/devices.py`）

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/devices` | 裝置列表（`list_all`）+ 內嵌「新增裝置」表單 |
| POST | `/devices` | 新增裝置（form: `address`, `name`；owner = current_user） |
| GET | `/devices/<int:device_id>` | 裝置詳情（裝置資訊 + 其頻道表）+ 內嵌「新增頻道」表單 |
| POST | `/devices/<int:device_id>/delete` | 刪除裝置（cascade 頻道），導回 `/devices` |

### 3.2 Channel blueprint（`web/channels.py`）

| Method | Path | 說明 |
| --- | --- | --- |
| POST | `/devices/<int:device_id>/channels` | 新增頻道，導回該裝置詳情 |
| POST | `/channels/<int:channel_id>/delete` | 刪除頻道，導回所屬裝置詳情 |

---

## 4. 表單（Flask-WTF）

- `AddDeviceForm`：`address` StringField（`DataRequired`）、`name` StringField（`DataRequired`）
- `AddChannelForm`：
  - `name` StringField（`DataRequired`）
  - `type` SelectField，choices = `[("controller", ...), ("display", ...)]`
  - `char_uuid` StringField（`DataRequired`）
  - `data_format` SelectField，choices 由 `parser.supported_formats()` 動態產生
  - `unit` StringField（選填）
- 刪除按鈕：裸 `<form method="post">` + 隱藏 `csrf_token()`，不需獨立 form 類別。

---

## 5. 錯誤處理（flash + 重新渲染，仿 auth）

| 操作 | 例外 | 行為 |
| --- | --- | --- |
| 新增裝置 | `InvalidAddressError`（MAC 格式） | flash「Invalid BLE address」，重渲染列表 |
| 新增裝置 | `DuplicateAddressError` | flash「Address already exists」，重渲染列表 |
| 新增頻道 | `DuplicateChannelNameError` | flash「Channel name already exists on this device」，重渲染詳情 |
| 詳情頁 | 裝置不存在 | `abort(404)` |
| 刪除裝置／頻道 | 目標不存在（`DeviceNotFoundError` / `ChannelNotFoundError`） | `abort(404)` |

> `type` / `data_format` 由 SelectField 限制，理論上不會觸發 `InvalidChannelTypeError` / `UnknownFormatError`；仍保留 service 層既有防呆。

---

## 6. 模板（沿用 `base.html` + Bootstrap，vendored）

- `templates/devices/list.html`：裝置表格（name / address / created_at / 詳情連結 / 刪除鈕）+ 新增裝置表單卡片。
- `templates/devices/detail.html`：裝置資訊 + 頻道表格（name / type / char_uuid / data_format / unit / 刪除鈕）+ 新增頻道表單卡片。
- `base.html`：navbar 新增「Devices」連結（小幅 surgical 改動，方便導覽）。

---

## 7. 測試策略

新增 `tests/test_web_devices.py`、`tests/test_web_channels.py`。conftest 新增 `logged_in_client` fixture（註冊並登入後的 client，供重用）。

意義導向案例（非為覆蓋率灌水）：

- 未登入存取 `/devices` → 302 導向登入
- 新增裝置後出現在列表，且 `owner_user_id` 為當前使用者
- MAC 格式錯誤 → flash 錯誤且未寫入 DB
- 重複位址 → flash 錯誤且未新增
- 詳情頁顯示其頻道；裝置不存在 → 404
- 在裝置下新增頻道後，出現在詳情頁
- 頻道名稱於同裝置重複 → flash 錯誤
- 刪除裝置後從列表消失（且其頻道因 cascade 一併刪除）
- 刪除頻道後從詳情頁消失
- 缺 CSRF token 的 POST → 400

驗證指令（沿用既有）：`uv run ruff check`、`uv run mypy src`、`uv run pytest`（含現有 103 tests 仍須全綠）。

---

## 8. 影響的檔案

新增：
- `src/home_server/web/devices.py`
- `src/home_server/web/channels.py`
- `src/home_server/web/services.py`（型別化 service 存取器）
- `src/home_server/web/templates/devices/list.html`
- `src/home_server/web/templates/devices/detail.html`
- `tests/test_web_devices.py`
- `tests/test_web_channels.py`

修改：
- `src/home_server/web/__init__.py`（建立並注入 BLE manager + service，註冊兩個 blueprint）
- `src/home_server/web/templates/base.html`（navbar 加「Devices」連結）
- `tests/conftest.py`（新增 `logged_in_client` fixture）

不修改 service 層、db 層、ble 層、config（除非實作中發現必要，屆時於 plan 提出）。

---

*文件結束。實作前若需調整任何決策，請於此討論。*
