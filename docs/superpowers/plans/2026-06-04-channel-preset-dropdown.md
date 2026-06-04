# Channel 功能預設下拉 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把「Add channel」表單從手動輸入 char_uuid/type/data_format/unit，改為從一份 JSON 設定檔載入的「功能」單一下拉，選功能 + 取名即可建立 channel。

**Architecture:** 新增 `presets.py`（讀 JSON + 驗證）與 `data/channel_presets.json`（可編輯清單，初始對齊 STM32 GATT 表）。應用工廠啟動時載入一次存入 `app.config["CHANNEL_PRESETS"]`；web 表單改為 `name` + `preset`（`SelectField`），route 由 preset 解析出四個值再呼叫**既有** `ChannelService.add_channel`。DB schema / channels 表 / service / db 層完全不動。

**Tech Stack:** Python 3.12、Flask、Flask-WTF（WTForms）、SQLite、pytest、ruff（check）、mypy（strict）。

> **路徑慣例：** 本計畫所有相對路徑均相對於 submodule `Intelligent-home-RPi-server/`；所有 `git` 與 `task` / `uv` 指令都在該 submodule 目錄內執行。對應 spec：`docs/superpowers/specs/2026-06-04-channel-preset-dropdown-design.md`（位於外層主 repo）。

---

## File Structure

| 檔案 | 動作 | 職責 |
| --- | --- | --- |
| `src/home_server/presets.py` | 建立 | `ChannelPreset` dataclass、`PresetError`、`load_presets(path)`（讀檔 + 驗證） |
| `data/channel_presets.json` | 建立 | 初始功能清單（納入版控） |
| `tests/test_presets.py` | 建立 | `load_presets` 合法/非法案例 + 驗證 shipped JSON 可載入 |
| `src/home_server/config.py` | 修改 | `Config` 加 `channel_presets_path`；`from_env` 讀 `HOME_SERVER_CHANNEL_PRESETS_PATH` |
| `.env.example` | 修改 | 補環境變數註解 |
| `src/home_server/web/__init__.py` | 修改 | 工廠啟動載入 presets 存入 `app.config["CHANNEL_PRESETS"]` |
| `src/home_server/web/channels.py` | 修改 | `AddChannelForm` 改 `name`+`preset`；`add_channel` route 由 preset 解析 |
| `src/home_server/web/devices.py` | 修改 | `detail` route 傳 `presets` 取代 `formats`，移除未用的 `parser` import |
| `src/home_server/web/templates/devices/detail.html` | 修改 | Add channel 表單改下拉 + 小段 JS 自動帶入 Name |
| `tests/conftest.py` | 修改 | `app` fixture 寫測試用 presets JSON 並設入 `Config.channel_presets_path` |
| `tests/test_web_channels.py` | 修改 | 新增流程改用 `preset`+`name`；加未知 preset 被擋的測試 |

---

## Task 1: Config 加 `channel_presets_path`

**Files:**
- Modify: `src/home_server/config.py`
- Modify: `.env.example`

- [ ] **Step 1: 確認沒有其他地方建構 Config（避免漏改）**

Run: `grep -rn "Config(" src tests | grep -v "Config.from_env\|class Config\|: Config"`
Expected: 只出現 `tests/conftest.py` 一處（Task 5 會更新）。若有其他處，一併記下。

- [ ] **Step 2: 在 `Config` dataclass 尾端新增欄位（含預設值）**

`Config` 既有的兩個帶預設欄位 `ble_backend` / `scan_name_prefix` 之後，加上：

```python
    ble_backend: str = "auto"
    scan_name_prefix: str = "HOME-"
    channel_presets_path: Path = Path("./data/channel_presets.json")
```

（帶預設值，放在所有預設欄位之後，符合 dataclass 規則；`Path` 已在檔案頂部 import。）

- [ ] **Step 3: 在 `from_env` 的回傳中讀取環境變數**

在 `from_env` 的 `return cls(...)` 內、`scan_name_prefix=...` 之後加上：

```python
            scan_name_prefix=_env_str("HOME_SERVER_SCAN_NAME_PREFIX", "HOME-"),
            channel_presets_path=Path(
                _env_str(
                    "HOME_SERVER_CHANNEL_PRESETS_PATH", "./data/channel_presets.json"
                )
            ),
```

- [ ] **Step 4: 更新 `.env.example`**

在 `HOME_SERVER_SCAN_NAME_PREFIX=HOME-` 區塊之後追加：

```bash
# Channel preset catalog. Editable JSON list mapping a friendly label to the
# BLE characteristic (char_uuid, type, data_format, unit) it represents. The
# "add channel" UI offers these as a dropdown. Relative paths resolve from the
# project root.
HOME_SERVER_CHANNEL_PRESETS_PATH=./data/channel_presets.json
```

- [ ] **Step 5: 型別檢查**

Run: `uv run mypy src/home_server/config.py`
Expected: `Success: no issues found`

- [ ] **Step 6: Commit**

```bash
git add src/home_server/config.py .env.example
git commit -m "feat(config): add channel_presets_path setting"
```

---

## Task 2: `presets.py` 模組（TDD）

**Files:**
- Test: `tests/test_presets.py`
- Create: `src/home_server/presets.py`

- [ ] **Step 1: 先寫失敗測試**

建立 `tests/test_presets.py`：

```python
import json
from pathlib import Path

import pytest

from home_server.presets import ChannelPreset, PresetError, load_presets


def _write(tmp_path: Path, data: object) -> Path:
    p = tmp_path / "presets.json"
    p.write_text(json.dumps(data), encoding="utf-8")
    return p


def test_load_valid_presets(tmp_path: Path) -> None:
    path = _write(
        tmp_path,
        [
            {
                "label": "Temp",
                "char_uuid": "uuid-temp",
                "type": "display",
                "data_format": "float32_le",
                "unit": "°C",
            },
            {
                "label": "LED",
                "char_uuid": "uuid-led",
                "type": "controller",
                "data_format": "uint8",
            },
        ],
    )
    presets = load_presets(path)
    assert presets == [
        ChannelPreset("Temp", "uuid-temp", "display", "float32_le", "°C"),
        ChannelPreset("LED", "uuid-led", "controller", "uint8", None),
    ]


def test_missing_unit_becomes_none(tmp_path: Path) -> None:
    path = _write(
        tmp_path,
        [{"label": "X", "char_uuid": "u", "type": "display", "data_format": "uint8"}],
    )
    assert load_presets(path)[0].unit is None


def test_missing_file_raises(tmp_path: Path) -> None:
    with pytest.raises(PresetError):
        load_presets(tmp_path / "nope.json")


def test_invalid_json_raises(tmp_path: Path) -> None:
    p = tmp_path / "presets.json"
    p.write_text("{not json", encoding="utf-8")
    with pytest.raises(PresetError):
        load_presets(p)


def test_top_level_not_list_raises(tmp_path: Path) -> None:
    with pytest.raises(PresetError):
        load_presets(_write(tmp_path, {"label": "X"}))


def test_bad_type_raises(tmp_path: Path) -> None:
    path = _write(
        tmp_path,
        [{"label": "X", "char_uuid": "u", "type": "sensor", "data_format": "uint8"}],
    )
    with pytest.raises(PresetError):
        load_presets(path)


def test_unknown_data_format_raises(tmp_path: Path) -> None:
    path = _write(
        tmp_path,
        [{"label": "X", "char_uuid": "u", "type": "display", "data_format": "bogus"}],
    )
    with pytest.raises(PresetError):
        load_presets(path)


def test_empty_label_raises(tmp_path: Path) -> None:
    path = _write(
        tmp_path,
        [{"label": "", "char_uuid": "u", "type": "display", "data_format": "uint8"}],
    )
    with pytest.raises(PresetError):
        load_presets(path)


def test_duplicate_label_raises(tmp_path: Path) -> None:
    path = _write(
        tmp_path,
        [
            {"label": "X", "char_uuid": "u1", "type": "display", "data_format": "uint8"},
            {"label": "X", "char_uuid": "u2", "type": "display", "data_format": "uint8"},
        ],
    )
    with pytest.raises(PresetError):
        load_presets(path)


def test_duplicate_char_uuid_raises(tmp_path: Path) -> None:
    path = _write(
        tmp_path,
        [
            {"label": "A", "char_uuid": "u", "type": "display", "data_format": "uint8"},
            {"label": "B", "char_uuid": "u", "type": "display", "data_format": "uint8"},
        ],
    )
    with pytest.raises(PresetError):
        load_presets(path)


def test_empty_list_is_valid(tmp_path: Path) -> None:
    assert load_presets(_write(tmp_path, [])) == []
```

- [ ] **Step 2: 執行確認失敗**

Run: `uv run pytest tests/test_presets.py -q`
Expected: 收集/匯入失敗 — `ModuleNotFoundError: No module named 'home_server.presets'`。

- [ ] **Step 3: 實作 `src/home_server/presets.py`**

```python
"""Channel preset catalog loaded from a JSON file.

A preset bundles the technical fields of a BLE channel (char_uuid, type,
data_format, unit) behind a human-friendly label, so the "add channel" UI can
offer a single dropdown instead of asking the user to type a UUID/unit and pick
a type/format. The catalog lives in a JSON file (``Config.channel_presets_path``)
so it can be edited without code changes.
"""

from __future__ import annotations

import json
from dataclasses import dataclass
from pathlib import Path

from home_server.ble import parser
from home_server.db.channels import ChannelType

_VALID_TYPES: tuple[ChannelType, ...] = ("controller", "display")


class PresetError(ValueError):
    pass


@dataclass(frozen=True)
class ChannelPreset:
    label: str
    char_uuid: str
    type: ChannelType
    data_format: str
    unit: str | None


def load_presets(path: Path) -> list[ChannelPreset]:
    try:
        raw = path.read_text(encoding="utf-8")
    except OSError as e:
        raise PresetError(f"cannot read channel presets file {path}: {e}") from e
    try:
        data = json.loads(raw)
    except json.JSONDecodeError as e:
        raise PresetError(f"invalid JSON in {path}: {e}") from e
    if not isinstance(data, list):
        raise PresetError(f"{path}: top level must be a JSON array")

    presets: list[ChannelPreset] = []
    seen_labels: set[str] = set()
    seen_uuids: set[str] = set()
    for i, entry in enumerate(data):
        if not isinstance(entry, dict):
            raise PresetError(f"{path}[{i}]: entry must be an object")
        label = entry.get("label")
        char_uuid = entry.get("char_uuid")
        ctype = entry.get("type")
        data_format = entry.get("data_format")
        unit = entry.get("unit")
        if not isinstance(label, str) or not label:
            raise PresetError(f"{path}[{i}]: 'label' must be a non-empty string")
        if not isinstance(char_uuid, str) or not char_uuid:
            raise PresetError(f"{path}[{i}]: 'char_uuid' must be a non-empty string")
        if ctype not in _VALID_TYPES:
            raise PresetError(
                f"{path}[{i}]: 'type' must be one of {_VALID_TYPES}, got {ctype!r}"
            )
        if data_format not in parser.supported_formats():
            raise PresetError(f"{path}[{i}]: unknown 'data_format' {data_format!r}")
        if unit is not None and not isinstance(unit, str):
            raise PresetError(f"{path}[{i}]: 'unit' must be a string or null")
        if label in seen_labels:
            raise PresetError(f"{path}[{i}]: duplicate label {label!r}")
        if char_uuid in seen_uuids:
            raise PresetError(f"{path}[{i}]: duplicate char_uuid {char_uuid!r}")
        seen_labels.add(label)
        seen_uuids.add(char_uuid)
        presets.append(
            ChannelPreset(
                label=label,
                char_uuid=char_uuid,
                type=ctype,
                data_format=data_format,
                unit=unit,
            )
        )
    return presets
```

- [ ] **Step 4: 執行確認通過**

Run: `uv run pytest tests/test_presets.py -q`
Expected: 11 passed。

- [ ] **Step 5: Lint + 型別**

Run: `uv run ruff check src/home_server/presets.py tests/test_presets.py && uv run mypy src/home_server/presets.py`
Expected: ruff 無錯；mypy `Success`。

- [ ] **Step 6: Commit**

```bash
git add src/home_server/presets.py tests/test_presets.py
git commit -m "feat(presets): add channel preset catalog loader"
```

---

## Task 3: 初始 `data/channel_presets.json`

**Files:**
- Create: `data/channel_presets.json`
- Modify: `tests/test_presets.py`（加一個載入 shipped 檔的測試）

- [ ] **Step 1: 建立 `data/channel_presets.json`**

```json
[
  {"label": "溫度 (°C)", "char_uuid": "1A220002-8E22-4541-9D4C-21EDAE82ED19", "type": "display", "data_format": "float32_le", "unit": "°C"},
  {"label": "濕度 (% RH)", "char_uuid": "1A220003-8E22-4541-9D4C-21EDAE82ED19", "type": "display", "data_format": "float32_le", "unit": "% RH"},
  {"label": "加速度量值 (g)", "char_uuid": "1A220004-8E22-4541-9D4C-21EDAE82ED19", "type": "display", "data_format": "float32_le", "unit": "g"},
  {"label": "陀螺儀量值 (dps)", "char_uuid": "1A220005-8E22-4541-9D4C-21EDAE82ED19", "type": "display", "data_format": "float32_le", "unit": "dps"},
  {"label": "異常晃動警示", "char_uuid": "1A220006-8E22-4541-9D4C-21EDAE82ED19", "type": "display", "data_format": "uint8"},
  {"label": "麥克風音量", "char_uuid": "1A220007-8E22-4541-9D4C-21EDAE82ED19", "type": "display", "data_format": "uint16_le"},
  {"label": "大聲警示", "char_uuid": "1A220008-8E22-4541-9D4C-21EDAE82ED19", "type": "display", "data_format": "uint8"},
  {"label": "LED1 開關", "char_uuid": "1A22F002-8E22-4541-9D4C-21EDAE82ED19", "type": "controller", "data_format": "uint8"},
  {"label": "控制旗標", "char_uuid": "1A22F003-8E22-4541-9D4C-21EDAE82ED19", "type": "controller", "data_format": "uint8"}
]
```

- [ ] **Step 2: 加一個測試：shipped 檔能無誤載入**

在 `tests/test_presets.py` 末尾加入（驗證實際出貨清單沒有 typo）：

```python
def test_shipped_catalog_loads() -> None:
    repo_root = Path(__file__).parent.parent
    presets = load_presets(repo_root / "data" / "channel_presets.json")
    assert len(presets) == 9
    labels = {p.label for p in presets}
    assert "溫度 (°C)" in labels
```

- [ ] **Step 3: 確認 `.gitignore` 不排除此檔並執行測試**

Run: `git check-ignore data/channel_presets.json; uv run pytest tests/test_presets.py -q`
Expected: `git check-ignore` 無輸出（代表未被忽略，退出碼 1）；pytest 12 passed。

- [ ] **Step 4: Commit**

```bash
git add data/channel_presets.json tests/test_presets.py
git commit -m "feat(presets): seed channel preset catalog from STM32 GATT table"
```

---

## Task 4: 應用工廠載入 presets

**Files:**
- Modify: `src/home_server/web/__init__.py`

- [ ] **Step 1: 在 `create_app` 載入 presets**

在 `create_app` 內、`app.config["BLE_SCAN_DURATION"] = config.ble_scan_duration` 之後加入：

```python
    app.config["BLE_SCAN_DURATION"] = config.ble_scan_duration

    from home_server.presets import load_presets

    app.config["CHANNEL_PRESETS"] = load_presets(config.channel_presets_path)
```

（區域 import 與檔內既有 blueprint 區域 import 風格一致，避免任何匯入循環。）

- [ ] **Step 2: 型別檢查**

Run: `uv run mypy src/home_server/web/__init__.py`
Expected: `Success: no issues found`

- [ ] **Step 3: Commit**

```bash
git add src/home_server/web/__init__.py
git commit -m "feat(web): load channel presets into app config at startup"
```

---

## Task 5: 測試 fixture 提供測試用 preset 清單

**Files:**
- Modify: `tests/conftest.py`

- [ ] **Step 1: 在 `app` fixture 寫測試用 presets JSON 並設入 Config**

`tests/conftest.py` 頂部 import 區加入 `import json`。

把 `app` fixture 改為（在建 `Config` 前寫檔、`Config(...)` 內加 `channel_presets_path`）：

```python
@pytest.fixture
def app(tmp_path: Path, mock_ble: MockBLEManager) -> Flask:
    # File-backed DB (not :memory:) so each per-request connection sees the
    # same database — distinct :memory: connections would each be empty.
    db_path = tmp_path / "test.db"
    connection.initialize(db_path)
    presets_path = tmp_path / "channel_presets.json"
    presets_path.write_text(
        json.dumps(
            [
                {
                    "label": "Temp",
                    "char_uuid": "uuid-temp",
                    "type": "display",
                    "data_format": "float32_le",
                    "unit": "°C",
                },
                {
                    "label": "LED",
                    "char_uuid": "uuid-led",
                    "type": "controller",
                    "data_format": "uint8",
                },
            ]
        ),
        encoding="utf-8",
    )
    config = Config(
        db_path=db_path,
        secret_key="test-secret",
        host="127.0.0.1",
        port=5000,
        log_level="INFO",
        ble_scan_duration=1.0,
        reading_min_interval=1.0,
        admin_username="admin",
        admin_password=None,
        debug=True,
        channel_presets_path=presets_path,
    )
    flask_app = create_app(config, ble=mock_ble)
    flask_app.config["TESTING"] = True
    return flask_app
```

- [ ] **Step 2: 確認既有測試套件仍可收集（尚未改 web 行為，detail 頁這時可能因 template/route 不一致而失敗，下一 Task 修）**

Run: `uv run pytest tests/test_presets.py -q`
Expected: 12 passed（presets 測試與 conftest 改動相容）。

> 注意：此時先不要跑 `tests/test_web_channels.py`，因為 web 層（route/template）尚未切換，會處於中間狀態。Task 6 一次完成切換並驗證。

- [ ] **Step 3: Commit**

```bash
git add tests/conftest.py
git commit -m "test: provide channel preset catalog in app fixture"
```

---

## Task 6: 切換 web 層為 preset 下拉（TDD，一次改齊 route/template/devices）

> channels.py、devices.py、detail.html 必須一起改才能讓 detail 頁與新增流程一致，故合為一個 commit。先更新 `test_web_channels.py` 看它失敗，再實作三檔讓它通過。

**Files:**
- Modify: `tests/test_web_channels.py`
- Modify: `src/home_server/web/channels.py`
- Modify: `src/home_server/web/devices.py`
- Modify: `src/home_server/web/templates/devices/detail.html`

- [ ] **Step 1: 更新 `tests/test_web_channels.py`（先失敗）**

頂部 import 區確認有 `from home_server.db import channels, connection, devices, readings`（已存在）。

取代 `test_add_channel_appears_on_detail`、`test_add_channel_duplicate_name_flashes`、`test_add_channel_device_not_found` 三個函式，並新增 `test_add_channel_unknown_preset_rejected`：

```python
def test_add_channel_appears_on_detail(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:06", "Board")
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/devices/{device_id}/channels",
        data={"name": "Humidity", "preset": "uuid-temp", "csrf_token": token},
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert b"Humidity" in resp.data
    # The preset (uuid-temp) determines the stored technical fields.
    conn = connection.connect(app.config["DB_PATH"])
    try:
        created = channels.list_by_device(conn, device_id)
    finally:
        conn.close()
    assert len(created) == 1
    ch = created[0]
    assert ch.name == "Humidity"
    assert ch.char_uuid == "uuid-temp"
    assert ch.type == "display"
    assert ch.data_format == "float32_le"
    assert ch.unit == "°C"


def test_add_channel_duplicate_name_flashes(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:07", "Board2")
    conn = connection.connect(app.config["DB_PATH"])
    try:
        channels.create(
            conn,
            device_id=device_id,
            name="Dup",
            type="display",
            char_uuid="u",
            data_format="uint8",
            unit=None,
        )
    finally:
        conn.close()
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/devices/{device_id}/channels",
        data={"name": "Dup", "preset": "uuid-temp", "csrf_token": token},
    )
    assert resp.status_code == 200
    assert b"Channel name already exists" in resp.data


def test_add_channel_device_not_found(logged_in_client: FlaskClient) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    resp = logged_in_client.post(
        "/devices/999/channels",
        data={"name": "X", "preset": "uuid-temp", "csrf_token": token},
    )
    assert resp.status_code == 404


def test_add_channel_unknown_preset_rejected(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:09", "Board3")
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/devices/{device_id}/channels",
        data={"name": "X", "preset": "uuid-nope", "csrf_token": token},
        follow_redirects=True,
    )
    # SelectField choices validation rejects an unknown preset; no channel created.
    assert resp.status_code == 200
    conn = connection.connect(app.config["DB_PATH"])
    try:
        assert channels.list_by_device(conn, device_id) == []
    finally:
        conn.close()
```

- [ ] **Step 2: 執行確認失敗**

Run: `uv run pytest tests/test_web_channels.py -q`
Expected: 失敗（目前 route 仍要求 `char_uuid` 等舊欄位、detail 模板仍用 `formats`）。

- [ ] **Step 3: 改 `src/home_server/web/channels.py` 的 import 與表單**

把 flask import 區加上 `current_app`：

```python
from flask import (
    Blueprint,
    abort,
    current_app,
    flash,
    jsonify,
    redirect,
    render_template,
    request,
    url_for,
)
```

把 `AddChannelForm` 改為：

```python
class AddChannelForm(FlaskForm):
    name = StringField("Name", validators=[DataRequired()])
    preset = SelectField("Function", validators=[DataRequired()])
```

（`SelectField`、`StringField`、`DataRequired`、`parser` 等 import 都仍在使用，保留不動。）

- [ ] **Step 4: 改 `add_channel` route**

把 `add_channel` 函式本體改為：

```python
@bp.post("/devices/<int:device_id>/channels")
@login_required
def add_channel(device_id: int) -> Response | str:
    conn = get_conn()
    device = devices.get_by_id(conn, device_id)
    if device is None:
        abort(404)
    presets = current_app.config["CHANNEL_PRESETS"]
    form = AddChannelForm()
    form.preset.choices = [(p.char_uuid, p.label) for p in presets]
    if form.validate_on_submit():
        preset = next((p for p in presets if p.char_uuid == form.preset.data), None)
        if preset is None:
            flash("Unknown channel function")
        else:
            try:
                get_channel_service().add_channel(
                    conn,
                    device_id=device_id,
                    name=form.name.data,
                    type=preset.type,
                    char_uuid=preset.char_uuid,
                    data_format=preset.data_format,
                    unit=preset.unit,
                )
            except DuplicateChannelNameError:
                flash("Channel name already exists on this device")
            else:
                return redirect(url_for("devices.detail", device_id=device_id))
    device_channels = channels.list_by_device(conn, device_id)
    return render_template(
        "devices/detail.html",
        device=device,
        channels=device_channels,
        presets=presets,
    )
```

- [ ] **Step 5: 改 `src/home_server/web/devices.py` 的 `detail` route**

移除未使用的 import（第 12 行）：

```python
from home_server.ble import parser
```

把 `detail` 的 `render_template` 由：

```python
        formats=parser.supported_formats(),
```

改為：

```python
        presets=current_app.config["CHANNEL_PRESETS"],
```

（`current_app` 已在第 5 行 import；移除 `parser` import 後該檔不再用到 `parser`。）

- [ ] **Step 6: 改 `src/home_server/web/templates/devices/detail.html` 的「Add channel」區塊**

把最後的 `<div class="card shadow-sm"> … Add channel … </div>`（整個卡片）替換為：

```html
<div class="card shadow-sm">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Add channel</h2>
    {% if presets %}
    <form method="post"
          action="{{ url_for('channels.add_channel', device_id=device.id) }}"
          class="row g-2">
      <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
      <div class="col-sm-5">
        <select id="channel-preset" class="form-select" name="preset">
          {% for p in presets %}
          <option value="{{ p.char_uuid }}" data-label="{{ p.label }}">{{ p.label }}</option>
          {% endfor %}
        </select>
      </div>
      <div class="col-sm-5">
        <input id="channel-name" class="form-control" name="name" placeholder="名稱" required>
      </div>
      <div class="col-sm-2 d-grid">
        <button class="btn btn-primary" type="submit">Add</button>
      </div>
    </form>
    <script>
      (function () {
        const sel = document.getElementById('channel-preset');
        const nameInput = document.getElementById('channel-name');
        if (!sel || !nameInput) return;
        let edited = false;
        nameInput.addEventListener('input', function () { edited = true; });
        function fill() {
          if (edited && nameInput.value.trim() !== '') return;
          const opt = sel.selectedOptions[0];
          if (opt) nameInput.value = opt.dataset.label || opt.textContent;
        }
        sel.addEventListener('change', fill);
        fill();
      })();
    </script>
    {% else %}
    <p class="text-muted mb-0">尚未設定任何功能，請編輯 <code>data/channel_presets.json</code>。</p>
    {% endif %}
  </div>
</div>
```

- [ ] **Step 7: 執行 web 測試確認通過**

Run: `uv run pytest tests/test_web_channels.py -q`
Expected: 全數 passed（含新增的 unknown-preset 測試）。

- [ ] **Step 8: 全套件 + lint + 型別**

Run: `uv run pytest -q && uv run ruff check src tests && uv run mypy src`
Expected: 全綠（pytest all passed、ruff 無錯、mypy `Success`）。

> 若 mypy 對 `presets = current_app.config["CHANNEL_PRESETS"]` 報 `Any`，屬可接受（`app.config` 本就是動態 dict）；勿為此加無謂的 cast，除非 mypy strict 真的報錯。若報錯，於 `add_channel` 內標註 `presets: list[ChannelPreset] = current_app.config["CHANNEL_PRESETS"]` 並 `from home_server.presets import ChannelPreset`。

- [ ] **Step 9: Commit**

```bash
git add tests/test_web_channels.py src/home_server/web/channels.py \
        src/home_server/web/devices.py \
        src/home_server/web/templates/devices/detail.html
git commit -m "feat(web): replace channel uuid/type/unit inputs with preset dropdown"
```

---

## Task 7: 最終驗收

**Files:** 無（僅驗證）

- [ ] **Step 1: 跑專案標準驗收**

Run: `task lint && task test`
Expected: 兩者皆綠（等同 ruff check + mypy strict + pytest；不跑 `ruff format`）。

- [ ] **Step 2: 手動煙測（非自動化，可選）**

啟動：`set -a; source .env; set +a; uv run python -m home_server`（或 `task run`），於本機開裝置詳情頁，確認：
1. 「Add channel」只剩「功能下拉 + 名稱」兩欄。
2. 切換下拉時 Name 自動帶入該功能 label；手動改 Name 後再切換不被覆蓋。
3. 送出後 channel 出現在列表，UUID/Format/Unit 欄顯示對應 preset 的值。

---

## Self-Review（計畫對照 spec）

- **Spec §3.1 presets.py**：Task 2（dataclass + load_presets + 全部驗證分支：缺檔/壞JSON/非list/壞type/未知format/空label/重複label/重複uuid/unit型別）。✓
- **Spec §3.2 config**：Task 1。✓
- **Spec §3.3 工廠載入**：Task 4。✓
- **Spec §3.4 channels.py 表單/route**：Task 6 Steps 3-4，含 choices 設定須在 `validate_on_submit` 前、未知 preset 防禦、`DuplicateChannelNameError` 沿用。✓
- **Spec §3.4 兩個渲染點**：Task 6 Step 4（channels.py 重渲染傳 presets）+ Step 5（devices.py detail 傳 presets、移除 parser import）。✓
- **Spec §3.5 模板 + JS + 空清單提示**：Task 6 Step 6。✓
- **Spec §3.6 初始 JSON**：Task 3（9 筆對齊 GATT 表）。✓
- **Spec §3.7 gitignore**：Task 3 Step 3 驗證。✓
- **Spec §4 錯誤處理**：未知 preset（Task 6 unknown-preset 測試）、缺檔/格式錯（Task 2 測試 + Task 4 啟動載入 fail-fast）、空清單（Task 6 模板分支）。✓
- **Spec §5 測試**：test_presets.py（Task 2/3）、conftest（Task 5）、test_web_channels.py（Task 6）。✓
- **型別一致性**：`ChannelPreset(label, char_uuid, type, data_format, unit)` 在 presets.py 定義、conftest JSON、route 解析、template 取用（`p.char_uuid`/`p.label`）一致；`load_presets(path)` 簽名於 Task 2/3/4/5 一致。✓
- **Placeholder 掃描**：無 TBD/TODO；每個改碼步驟都附完整程式碼。✓
