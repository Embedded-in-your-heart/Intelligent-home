# Phase 3d — Device / Channel CRUD Blueprint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add server-rendered device & channel CRUD management pages (list / add / detail / delete) to the RPi web server, wiring the existing `DeviceService` / `ChannelService` into the Flask app factory.

**Architecture:** `create_app()` instantiates a `BLEManager` (default `MockBLEManager`), a `RateLimiter`, and the two services, storing them in `app.extensions`; blueprints read them through typed accessors in `web/services.py`. Two blueprints (`devices`, `channels`) handle HTTP routes. Operations that touch BLE/validation go through the services; simple reads and channel deletion call db repos directly (matching the established `auth.py` pattern). Mutations use plain `POST` forms with full-page reloads; delete uses `POST .../delete`.

**Tech Stack:** Python 3.11+, Flask, Flask-WTF (CSRF + forms), Flask-Login, Jinja2, Bootstrap (vendored), pytest, ruff, mypy (strict).

> **All commands run from `Intelligent-home-RPi-server/`** (the submodule; commits land in that repo). File paths below are relative to that directory.

> **Spec:** `docs/superpowers/specs/2026-05-31-phase-3d-device-channel-crud-design.md` (in the parent repo).

---

## File Structure

Created:
- `src/home_server/web/services.py` — typed accessors for app-scoped services.
- `src/home_server/web/devices.py` — device blueprint (list, add, detail, delete).
- `src/home_server/web/channels.py` — channel blueprint (add, delete).
- `src/home_server/web/templates/devices/list.html` — device list + add form.
- `src/home_server/web/templates/devices/detail.html` — device detail + channels + add-channel form.
- `tests/test_web_services.py`, `tests/test_web_devices.py`, `tests/test_web_channels.py`.

Modified:
- `src/home_server/web/__init__.py` — build + register services; register blueprints.
- `src/home_server/web/templates/base.html` — add "Devices" nav link.
- `tests/conftest.py` — add `logged_in_client` fixture + `_csrf_token` helper.

Not modified: `services/`, `db/`, `ble/`, `config.py` (used, not changed).

---

### Task 1: Wire BLE manager + services into the app factory

**Files:**
- Create: `src/home_server/web/services.py`
- Modify: `src/home_server/web/__init__.py`
- Test: `tests/test_web_services.py`

- [ ] **Step 1: Write the failing test**

Create `tests/test_web_services.py`:

```python
from flask import Flask

from home_server.services.channel_service import ChannelService
from home_server.services.device_service import DeviceService
from home_server.web.services import get_channel_service, get_device_service


def test_services_registered_in_app(app: Flask) -> None:
    with app.app_context():
        assert isinstance(get_device_service(), DeviceService)
        assert isinstance(get_channel_service(), ChannelService)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest tests/test_web_services.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'home_server.web.services'`

- [ ] **Step 3: Create `web/services.py`**

```python
"""Typed accessors for app-scoped services stored in ``app.extensions``.

Kept separate from ``web/__init__`` so blueprints import these without an
import cycle through the application factory (mirrors ``web/db.py``).
"""

from __future__ import annotations

from flask import current_app

from home_server.services.channel_service import ChannelService
from home_server.services.device_service import DeviceService

DEVICE_SERVICE_KEY = "home_device_service"
CHANNEL_SERVICE_KEY = "home_channel_service"


def get_device_service() -> DeviceService:
    svc = current_app.extensions[DEVICE_SERVICE_KEY]
    assert isinstance(svc, DeviceService)
    return svc


def get_channel_service() -> ChannelService:
    svc = current_app.extensions[CHANNEL_SERVICE_KEY]
    assert isinstance(svc, ChannelService)
    return svc
```

- [ ] **Step 4: Wire services in `web/__init__.py`**

Add these imports to the existing import block:

```python
from home_server.ble.interface import BLEManager
from home_server.ble.mock_manager import MockBLEManager
from home_server.ble.rate_limiter import RateLimiter
from home_server.services.channel_service import ChannelService
from home_server.services.device_service import DeviceService
from home_server.web.services import CHANNEL_SERVICE_KEY, DEVICE_SERVICE_KEY
```

Add this module-level function above `create_app`:

```python
def _noop_reading(channel_id: int, value: float, timestamp: str) -> None:
    """Placeholder UI push. Phase 3e replaces this with a SocketIO emit."""
```

Change the signature and add the wiring block right after `app.config["DB_PATH"] = config.db_path`:

```python
def create_app(config: Config, ble: BLEManager | None = None) -> Flask:
    app = Flask(__name__)
    app.config["SECRET_KEY"] = config.secret_key
    app.config["DB_PATH"] = config.db_path

    if ble is None:
        ble = MockBLEManager()
    limiter = RateLimiter(config.reading_min_interval)
    app.extensions[DEVICE_SERVICE_KEY] = DeviceService(ble)
    app.extensions[CHANNEL_SERVICE_KEY] = ChannelService(ble, limiter, _noop_reading)
```

(Leave the rest of `create_app` unchanged for now.)

- [ ] **Step 5: Run test to verify it passes**

Run: `uv run pytest tests/test_web_services.py -v`
Expected: PASS

- [ ] **Step 6: Lint + type-check**

Run: `uv run ruff check && uv run mypy src`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add src/home_server/web/services.py src/home_server/web/__init__.py tests/test_web_services.py
git commit -m "feat(web): wire BLE manager and services into app factory"
```

---

### Task 2: Device list page (`GET /devices`) + login fixture

**Files:**
- Create: `src/home_server/web/devices.py`
- Create: `src/home_server/web/templates/devices/list.html`
- Modify: `src/home_server/web/__init__.py` (register blueprint)
- Modify: `src/home_server/web/templates/base.html` (nav link)
- Modify: `tests/conftest.py` (fixture + helper)
- Test: `tests/test_web_devices.py`

- [ ] **Step 1: Add `logged_in_client` fixture + helper to `tests/conftest.py`**

Add `import re` to the top imports, then append at the end of the file:

```python
def _csrf_token(client: FlaskClient, path: str) -> str:
    html = client.get(path).get_data(as_text=True)
    match = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    assert match, f"csrf_token not found at {path}"
    return match.group(1)


@pytest.fixture
def logged_in_client(client: FlaskClient) -> FlaskClient:
    """A client that has registered and logged in as 'tester' (user id 1)."""
    token = _csrf_token(client, "/auth/register")
    client.post(
        "/auth/register",
        data={
            "username": "tester",
            "password": "password1",
            "confirm": "password1",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    return client
```

- [ ] **Step 2: Write the failing tests**

Create `tests/test_web_devices.py`:

```python
import re

from flask import Flask
from flask.testing import FlaskClient

from home_server.db import connection, devices


def _csrf_token(client: FlaskClient, path: str) -> str:
    html = client.get(path).get_data(as_text=True)
    match = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    assert match, f"csrf_token not found at {path}"
    return match.group(1)


def test_devices_requires_login(client: FlaskClient) -> None:
    resp = client.get("/devices")
    assert resp.status_code == 302
    assert "/auth/login" in resp.headers["Location"]


def test_devices_empty_list(logged_in_client: FlaskClient) -> None:
    resp = logged_in_client.get("/devices")
    assert resp.status_code == 200
    assert b"No devices yet" in resp.data


def test_devices_list_shows_device(app: Flask, logged_in_client: FlaskClient) -> None:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        devices.create(conn, address="AA:BB:CC:DD:EE:03", name="Fan", owner_user_id=1)
    finally:
        conn.close()
    resp = logged_in_client.get("/devices")
    assert resp.status_code == 200
    assert b"Fan" in resp.data
    assert b"AA:BB:CC:DD:EE:03" in resp.data
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: FAIL — `/devices` returns 404 (no such route), so assertions fail.

- [ ] **Step 4: Create `web/devices.py`**

```python
"""Device management blueprint: list, add, detail, delete."""

from __future__ import annotations

from flask import Blueprint, render_template
from flask_login import login_required

from home_server.web.db import get_conn
from home_server.web.services import get_device_service

bp = Blueprint("devices", __name__)


@bp.get("/devices")
@login_required
def list_devices() -> str:
    items = get_device_service().list_devices(get_conn())
    return render_template("devices/list.html", devices=items)
```

- [ ] **Step 5: Create `templates/devices/list.html`**

```html
{% extends "base.html" %}
{% block title %}Devices{% endblock %}
{% block content %}
<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h1 class="card-title h4 mb-3">Devices</h1>
    {% if devices %}
    <table class="table table-sm align-middle">
      <thead>
        <tr><th>Name</th><th>Address</th><th>Created</th></tr>
      </thead>
      <tbody>
        {% for device in devices %}
        <tr>
          <td>{{ device.name }}</td>
          <td><code>{{ device.address }}</code></td>
          <td>{{ device.created_at }}</td>
        </tr>
        {% endfor %}
      </tbody>
    </table>
    {% else %}
    <p class="text-muted mb-0">No devices yet.</p>
    {% endif %}
  </div>
</div>
{% endblock %}
```

- [ ] **Step 6: Register the blueprint in `web/__init__.py`**

Inside `create_app`, after `app.register_blueprint(auth_bp)`, add:

```python
    from home_server.web.devices import bp as devices_bp

    app.register_blueprint(devices_bp)
```

- [ ] **Step 7: Add the nav link in `templates/base.html`**

Replace the existing `{% if current_user.is_authenticated %}` block inside `<nav>` with:

```html
      {% if current_user.is_authenticated %}
      <ul class="navbar-nav me-auto">
        <li class="nav-item">
          <a class="nav-link" href="{{ url_for('devices.list_devices') }}">Devices</a>
        </li>
      </ul>
      <form method="post" action="{{ url_for('auth.logout') }}" class="d-flex">
        <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
        <button class="btn btn-outline-light btn-sm" type="submit">Logout</button>
      </form>
      {% endif %}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: PASS (3 tests)

- [ ] **Step 9: Run full suite + lint + type-check**

Run: `uv run pytest && uv run ruff check && uv run mypy src`
Expected: all green (existing auth tests still pass — they render `base.html` which now references `devices.list_devices`, registered above).

- [ ] **Step 10: Commit**

```bash
git add src/home_server/web/devices.py src/home_server/web/templates/devices/list.html src/home_server/web/__init__.py src/home_server/web/templates/base.html tests/conftest.py tests/test_web_devices.py
git commit -m "feat(web): add device list page and login_required fixture"
```

---

### Task 3: Device detail page (`GET /devices/<id>`)

**Files:**
- Modify: `src/home_server/web/devices.py`
- Create: `src/home_server/web/templates/devices/detail.html`
- Modify: `src/home_server/web/templates/devices/list.html` (link name → detail)
- Test: `tests/test_web_devices.py`

- [ ] **Step 1: Write the failing tests**

Append to `tests/test_web_devices.py` (add `channels` to the existing db import: `from home_server.db import channels, connection, devices`):

```python
def test_detail_shows_device_and_channels(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        device_id = devices.create(
            conn, address="AA:BB:CC:DD:EE:02", name="Lamp", owner_user_id=1
        )
        channels.create(
            conn,
            device_id=device_id,
            name="Power",
            type="controller",
            char_uuid="uuid-1",
            data_format="uint8",
            unit=None,
        )
    finally:
        conn.close()
    resp = logged_in_client.get(f"/devices/{device_id}")
    assert resp.status_code == 200
    assert b"Lamp" in resp.data
    assert b"Power" in resp.data


def test_detail_404_for_missing(logged_in_client: FlaskClient) -> None:
    resp = logged_in_client.get("/devices/999")
    assert resp.status_code == 404
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_web_devices.py -k detail -v`
Expected: FAIL — `/devices/<id>` returns 404 for the existing device too (no route).

- [ ] **Step 3: Add the detail route to `web/devices.py`**

Update imports at the top of `web/devices.py`:

```python
from flask import Blueprint, abort, render_template
from flask_login import login_required

from home_server.db import devices
from home_server.web.db import get_conn
from home_server.web.services import get_channel_service, get_device_service
```

Add the route:

```python
@bp.get("/devices/<int:device_id>")
@login_required
def detail(device_id: int) -> str:
    conn = get_conn()
    device = devices.get_by_id(conn, device_id)
    if device is None:
        abort(404)
    device_channels = get_channel_service().list_by_device(conn, device_id)
    return render_template("devices/detail.html", device=device, channels=device_channels)
```

- [ ] **Step 4: Create `templates/devices/detail.html`**

```html
{% extends "base.html" %}
{% block title %}{{ device.name }}{% endblock %}
{% block content %}
<p><a href="{{ url_for('devices.list_devices') }}">&larr; Devices</a></p>

<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h1 class="card-title h4">{{ device.name }}</h1>
    <p class="card-text mb-0"><code>{{ device.address }}</code></p>
  </div>
</div>

<div class="card shadow-sm mb-4">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Channels</h2>
    {% if channels %}
    <table class="table table-sm align-middle">
      <thead>
        <tr><th>Name</th><th>Type</th><th>UUID</th><th>Format</th><th>Unit</th></tr>
      </thead>
      <tbody>
        {% for channel in channels %}
        <tr>
          <td>{{ channel.name }}</td>
          <td>{{ channel.type }}</td>
          <td><code>{{ channel.char_uuid }}</code></td>
          <td>{{ channel.data_format }}</td>
          <td>{{ channel.unit or "" }}</td>
        </tr>
        {% endfor %}
      </tbody>
    </table>
    {% else %}
    <p class="text-muted mb-0">No channels yet.</p>
    {% endif %}
  </div>
</div>
{% endblock %}
```

- [ ] **Step 5: Link device name to detail in `templates/devices/list.html`**

Replace the name cell `<td>{{ device.name }}</td>` with:

```html
          <td><a href="{{ url_for('devices.detail', device_id=device.id) }}">{{ device.name }}</a></td>
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: PASS (5 tests)

- [ ] **Step 7: Lint + type-check**

Run: `uv run ruff check && uv run mypy src`
Expected: no errors

- [ ] **Step 8: Commit**

```bash
git add src/home_server/web/devices.py src/home_server/web/templates/devices/detail.html src/home_server/web/templates/devices/list.html tests/test_web_devices.py
git commit -m "feat(web): add device detail page with channels"
```

---

### Task 4: Add device (`POST /devices`)

**Files:**
- Modify: `src/home_server/web/devices.py`
- Modify: `src/home_server/web/templates/devices/list.html` (add form)
- Test: `tests/test_web_devices.py`

- [ ] **Step 1: Write the failing tests**

Append to `tests/test_web_devices.py`:

```python
def test_add_device_persists_with_owner(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    logged_in_client.post(
        "/devices",
        data={"address": "AA:BB:CC:DD:EE:FF", "name": "Sensor", "csrf_token": token},
        follow_redirects=True,
    )
    conn = connection.connect(app.config["DB_PATH"])
    try:
        device = devices.get_by_address(conn, "AA:BB:CC:DD:EE:FF")
    finally:
        conn.close()
    assert device is not None
    assert device.name == "Sensor"
    assert device.owner_user_id == 1


def test_add_device_invalid_address_flashes(logged_in_client: FlaskClient) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    resp = logged_in_client.post(
        "/devices",
        data={"address": "not-a-mac", "name": "X", "csrf_token": token},
    )
    assert resp.status_code == 200
    assert b"Invalid BLE address" in resp.data


def test_add_device_duplicate_flashes(logged_in_client: FlaskClient) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    logged_in_client.post(
        "/devices",
        data={"address": "AA:BB:CC:DD:EE:01", "name": "A", "csrf_token": token},
        follow_redirects=True,
    )
    token = _csrf_token(logged_in_client, "/devices")
    resp = logged_in_client.post(
        "/devices",
        data={"address": "AA:BB:CC:DD:EE:01", "name": "B", "csrf_token": token},
    )
    assert resp.status_code == 200
    assert b"Address already exists" in resp.data
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_web_devices.py -k add_device -v`
Expected: FAIL — `POST /devices` returns 405 (Method Not Allowed; route is GET-only).

- [ ] **Step 3: Implement the form + POST handling in `web/devices.py`**

Update the imports:

```python
from flask import Blueprint, abort, flash, redirect, render_template, url_for
from flask_login import current_user, login_required
from flask_wtf import FlaskForm
from werkzeug.wrappers import Response
from wtforms import StringField
from wtforms.validators import DataRequired

from home_server.db import devices
from home_server.db.devices import DuplicateAddressError
from home_server.services.device_service import InvalidAddressError
from home_server.web.db import get_conn
from home_server.web.services import get_channel_service, get_device_service
```

Add the form class after `bp = Blueprint(...)`:

```python
class AddDeviceForm(FlaskForm):
    address = StringField("Address", validators=[DataRequired()])
    name = StringField("Name", validators=[DataRequired()])
```

Replace the existing `list_devices` view with this GET+POST version:

```python
@bp.route("/devices", methods=["GET", "POST"])
@login_required
def list_devices() -> Response | str:
    form = AddDeviceForm()
    if form.validate_on_submit():
        try:
            get_device_service().add_device(
                get_conn(),
                owner_user_id=int(current_user.get_id()),
                address=form.address.data,
                name=form.name.data,
            )
        except InvalidAddressError:
            flash("Invalid BLE address")
        except DuplicateAddressError:
            flash("Address already exists")
        else:
            return redirect(url_for("devices.list_devices"))
    items = get_device_service().list_devices(get_conn())
    return render_template("devices/list.html", devices=items, form=form)
```

- [ ] **Step 4: Add the "Add device" form to `templates/devices/list.html`**

Append this card after the closing `</div>` of the existing list card (before `{% endblock %}`):

```html
<div class="card shadow-sm">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Add device</h2>
    <form method="post" action="{{ url_for('devices.list_devices') }}" class="row g-2">
      <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
      <div class="col-sm-5">
        <input class="form-control" name="address" placeholder="AA:BB:CC:DD:EE:FF"
               value="{{ form.address.data or '' }}" required>
      </div>
      <div class="col-sm-5">
        <input class="form-control" name="name" placeholder="Living room sensor"
               value="{{ form.name.data or '' }}" required>
      </div>
      <div class="col-sm-2 d-grid">
        <button class="btn btn-primary" type="submit">Add</button>
      </div>
    </form>
  </div>
</div>
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: PASS (8 tests)

- [ ] **Step 6: Lint + type-check**

Run: `uv run ruff check && uv run mypy src`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add src/home_server/web/devices.py src/home_server/web/templates/devices/list.html tests/test_web_devices.py
git commit -m "feat(web): add device creation form and route"
```

---

### Task 5: Delete device (`POST /devices/<id>/delete`)

**Files:**
- Modify: `src/home_server/web/devices.py`
- Modify: `src/home_server/web/templates/devices/list.html` (delete button)
- Test: `tests/test_web_devices.py`

- [ ] **Step 1: Write the failing tests**

Append to `tests/test_web_devices.py` (ensure `channels` is imported — added in Task 3):

```python
def test_delete_device_removes_it(app: Flask, logged_in_client: FlaskClient) -> None:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        device_id = devices.create(
            conn, address="AA:BB:CC:DD:EE:04", name="Heater", owner_user_id=1
        )
    finally:
        conn.close()
    token = _csrf_token(logged_in_client, "/devices")
    resp = logged_in_client.post(
        f"/devices/{device_id}/delete",
        data={"csrf_token": token},
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert b"Heater" not in resp.data


def test_delete_device_cascades_channels(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        device_id = devices.create(
            conn, address="AA:BB:CC:DD:EE:05", name="Hub", owner_user_id=1
        )
        channels.create(
            conn,
            device_id=device_id,
            name="Temp",
            type="display",
            char_uuid="u",
            data_format="float32_le",
            unit="C",
        )
    finally:
        conn.close()
    token = _csrf_token(logged_in_client, "/devices")
    logged_in_client.post(f"/devices/{device_id}/delete", data={"csrf_token": token})
    conn = connection.connect(app.config["DB_PATH"])
    try:
        remaining = channels.list_by_device(conn, device_id)
    finally:
        conn.close()
    assert remaining == []


def test_delete_missing_device_404(logged_in_client: FlaskClient) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    resp = logged_in_client.post("/devices/999/delete", data={"csrf_token": token})
    assert resp.status_code == 404
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_web_devices.py -k delete_ -v`
Expected: FAIL — `POST /devices/<id>/delete` returns 404 (no such route).

- [ ] **Step 3: Add the delete route to `web/devices.py`**

Add `DeviceNotFoundError` to the devices-db import:

```python
from home_server.db.devices import DeviceNotFoundError, DuplicateAddressError
```

Add the route:

```python
@bp.post("/devices/<int:device_id>/delete")
@login_required
def delete(device_id: int) -> Response:
    try:
        get_device_service().remove_device(get_conn(), device_id)
    except DeviceNotFoundError:
        abort(404)
    return redirect(url_for("devices.list_devices"))
```

- [ ] **Step 4: Add the delete button to `templates/devices/list.html`**

Add a trailing header cell `<th></th>` to the table's `<thead>` row, then add this cell as the last `<td>` in the device row (after the `created_at` cell):

```html
          <td class="text-end">
            <form method="post"
                  action="{{ url_for('devices.delete', device_id=device.id) }}"
                  onsubmit="return confirm('Delete this device?');">
              <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
              <button class="btn btn-outline-danger btn-sm" type="submit">Delete</button>
            </form>
          </td>
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_web_devices.py -v`
Expected: PASS (11 tests)

- [ ] **Step 6: Lint + type-check**

Run: `uv run ruff check && uv run mypy src`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add src/home_server/web/devices.py src/home_server/web/templates/devices/list.html tests/test_web_devices.py
git commit -m "feat(web): add device deletion route and button"
```

---

### Task 6: Add channel (`POST /devices/<id>/channels`)

**Files:**
- Create: `src/home_server/web/channels.py`
- Modify: `src/home_server/web/__init__.py` (register blueprint)
- Modify: `src/home_server/web/templates/devices/detail.html` (add-channel form)
- Test: `tests/test_web_channels.py`

- [ ] **Step 1: Write the failing tests**

Create `tests/test_web_channels.py`:

```python
import re

from flask import Flask
from flask.testing import FlaskClient

from home_server.db import channels, connection, devices


def _csrf_token(client: FlaskClient, path: str) -> str:
    html = client.get(path).get_data(as_text=True)
    match = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    assert match, f"csrf_token not found at {path}"
    return match.group(1)


def _make_device(app: Flask, address: str, name: str) -> int:
    conn = connection.connect(app.config["DB_PATH"])
    try:
        return devices.create(conn, address=address, name=name, owner_user_id=1)
    finally:
        conn.close()


def test_add_channel_appears_on_detail(
    app: Flask, logged_in_client: FlaskClient
) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:06", "Board")
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/devices/{device_id}/channels",
        data={
            "name": "Humidity",
            "type": "display",
            "char_uuid": "uuid-h",
            "data_format": "uint16_le",
            "unit": "%",
            "csrf_token": token,
        },
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert b"Humidity" in resp.data


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
        data={
            "name": "Dup",
            "type": "display",
            "char_uuid": "u2",
            "data_format": "uint8",
            "unit": "",
            "csrf_token": token,
        },
    )
    assert resp.status_code == 200
    assert b"Channel name already exists" in resp.data
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_web_channels.py -v`
Expected: FAIL — `POST /devices/<id>/channels` returns 404 (no route).

- [ ] **Step 3: Create `web/channels.py`**

```python
"""Channel management blueprint: add, delete."""

from __future__ import annotations

from flask import Blueprint, abort, flash, redirect, render_template, url_for
from flask_login import login_required
from flask_wtf import FlaskForm
from werkzeug.wrappers import Response
from wtforms import SelectField, StringField
from wtforms.validators import DataRequired

from home_server.ble import parser
from home_server.db import channels, devices
from home_server.db.channels import DuplicateChannelNameError
from home_server.web.db import get_conn
from home_server.web.services import get_channel_service

bp = Blueprint("channels", __name__)


class AddChannelForm(FlaskForm):
    name = StringField("Name", validators=[DataRequired()])
    type = SelectField(
        "Type", choices=[("controller", "controller"), ("display", "display")]
    )
    char_uuid = StringField("Characteristic UUID", validators=[DataRequired()])
    data_format = SelectField("Data format")
    unit = StringField("Unit")

    def __init__(self) -> None:
        super().__init__()
        self.data_format.choices = [(f, f) for f in parser.supported_formats()]


@bp.post("/devices/<int:device_id>/channels")
@login_required
def add_channel(device_id: int) -> Response | str:
    conn = get_conn()
    device = devices.get_by_id(conn, device_id)
    if device is None:
        abort(404)
    form = AddChannelForm()
    if form.validate_on_submit():
        try:
            get_channel_service().add_channel(
                conn,
                device_id=device_id,
                name=form.name.data,
                type=form.type.data,
                char_uuid=form.char_uuid.data,
                data_format=form.data_format.data,
                unit=form.unit.data or None,
            )
        except DuplicateChannelNameError:
            flash("Channel name already exists on this device")
        else:
            return redirect(url_for("devices.detail", device_id=device_id))
    device_channels = channels.list_by_device(conn, device_id)
    return render_template(
        "devices/detail.html", device=device, channels=device_channels, channel_form=form
    )
```

- [ ] **Step 4: Register the blueprint in `web/__init__.py`**

After the `devices_bp` registration, add:

```python
    from home_server.web.channels import bp as channels_bp

    app.register_blueprint(channels_bp)
```

- [ ] **Step 5: Add the "Add channel" form to `templates/devices/detail.html`**

Append this card after the channels card (before `{% endblock %}`):

```html
<div class="card shadow-sm">
  <div class="card-body">
    <h2 class="card-title h5 mb-3">Add channel</h2>
    <form method="post"
          action="{{ url_for('channels.add_channel', device_id=device.id) }}"
          class="row g-2">
      <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
      <div class="col-sm-3">
        <input class="form-control" name="name" placeholder="Temperature" required>
      </div>
      <div class="col-sm-2">
        <select class="form-select" name="type">
          <option value="display">display</option>
          <option value="controller">controller</option>
        </select>
      </div>
      <div class="col-sm-3">
        <input class="form-control" name="char_uuid" placeholder="Characteristic UUID" required>
      </div>
      <div class="col-sm-2">
        <select class="form-select" name="data_format">
          {% for fmt in formats %}
          <option value="{{ fmt }}">{{ fmt }}</option>
          {% endfor %}
        </select>
      </div>
      <div class="col-sm-1">
        <input class="form-control" name="unit" placeholder="Unit">
      </div>
      <div class="col-sm-1 d-grid">
        <button class="btn btn-primary" type="submit">Add</button>
      </div>
    </form>
  </div>
</div>
```

The template needs `formats` for the data-format options. Update the **detail route** in `web/devices.py` to pass it — change its `render_template` call to:

```python
    from home_server.ble import parser

    return render_template(
        "devices/detail.html",
        device=device,
        channels=device_channels,
        formats=parser.supported_formats(),
    )
```

And in `web/channels.py`'s `add_channel`, also pass `formats` so the re-rendered page on error keeps its dropdown — change its final `render_template` to:

```python
    return render_template(
        "devices/detail.html",
        device=device,
        channels=device_channels,
        formats=parser.supported_formats(),
    )
```

(Add `from home_server.ble import parser` to `web/channels.py` imports — it is already imported there.)

- [ ] **Step 6: Run tests to verify they pass**

Run: `uv run pytest tests/test_web_channels.py -v`
Expected: PASS (2 tests)

- [ ] **Step 7: Run full suite + lint + type-check**

Run: `uv run pytest && uv run ruff check && uv run mypy src`
Expected: all green (the detail route now passes `formats`; existing detail tests still pass).

- [ ] **Step 8: Commit**

```bash
git add src/home_server/web/channels.py src/home_server/web/__init__.py src/home_server/web/devices.py src/home_server/web/templates/devices/detail.html tests/test_web_channels.py
git commit -m "feat(web): add channel creation form and route"
```

---

### Task 7: Delete channel (`POST /channels/<id>/delete`)

**Files:**
- Modify: `src/home_server/web/channels.py`
- Modify: `src/home_server/web/templates/devices/detail.html` (delete button)
- Test: `tests/test_web_channels.py`

- [ ] **Step 1: Write the failing tests**

Append to `tests/test_web_channels.py`:

```python
def test_delete_channel_removes_it(app: Flask, logged_in_client: FlaskClient) -> None:
    device_id = _make_device(app, "AA:BB:CC:DD:EE:08", "B3")
    conn = connection.connect(app.config["DB_PATH"])
    try:
        channel_id = channels.create(
            conn,
            device_id=device_id,
            name="Gone",
            type="display",
            char_uuid="u",
            data_format="uint8",
            unit=None,
        )
    finally:
        conn.close()
    token = _csrf_token(logged_in_client, f"/devices/{device_id}")
    resp = logged_in_client.post(
        f"/channels/{channel_id}/delete",
        data={"csrf_token": token},
        follow_redirects=True,
    )
    assert resp.status_code == 200
    assert b"Gone" not in resp.data


def test_delete_missing_channel_404(logged_in_client: FlaskClient) -> None:
    token = _csrf_token(logged_in_client, "/devices")
    resp = logged_in_client.post("/channels/999/delete", data={"csrf_token": token})
    assert resp.status_code == 404
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/test_web_channels.py -k delete -v`
Expected: FAIL — `POST /channels/<id>/delete` returns 404 (no route).

- [ ] **Step 3: Add the delete route to `web/channels.py`**

Add the route (the needed imports — `abort`, `redirect`, `url_for`, `channels`, `get_conn` — are already present from Task 6):

```python
@bp.post("/channels/<int:channel_id>/delete")
@login_required
def delete_channel(channel_id: int) -> Response:
    conn = get_conn()
    channel = channels.get_by_id(conn, channel_id)
    if channel is None:
        abort(404)
    channels.delete(conn, channel_id)
    return redirect(url_for("devices.detail", device_id=channel.device_id))
```

- [ ] **Step 4: Add the delete button to `templates/devices/detail.html`**

Add a trailing header cell `<th></th>` to the channels table's `<thead>` row, then add this as the last `<td>` in the channel row (after the `unit` cell):

```html
          <td class="text-end">
            <form method="post"
                  action="{{ url_for('channels.delete_channel', channel_id=channel.id) }}"
                  onsubmit="return confirm('Delete this channel?');">
              <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
              <button class="btn btn-outline-danger btn-sm" type="submit">Delete</button>
            </form>
          </td>
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/test_web_channels.py -v`
Expected: PASS (4 tests)

- [ ] **Step 6: Lint + type-check**

Run: `uv run ruff check && uv run mypy src`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add src/home_server/web/channels.py src/home_server/web/templates/devices/detail.html tests/test_web_channels.py
git commit -m "feat(web): add channel deletion route and button"
```

---

### Task 8: CSRF enforcement test + final verification

**Files:**
- Test: `tests/test_web_devices.py`

- [ ] **Step 1: Write the failing test**

Append to `tests/test_web_devices.py`:

```python
def test_post_without_csrf_rejected(logged_in_client: FlaskClient) -> None:
    resp = logged_in_client.post(
        "/devices",
        data={"address": "AA:BB:CC:DD:EE:09", "name": "NoCSRF"},
    )
    assert resp.status_code == 400
```

- [ ] **Step 2: Run the test**

Run: `uv run pytest tests/test_web_devices.py::test_post_without_csrf_rejected -v`
Expected: PASS immediately — `CSRFProtect` is active in the app factory and the test app does not disable `WTF_CSRF`, so the missing token yields 400. (This test documents/locks in that behavior; no implementation change needed.)

- [ ] **Step 3: Full verification**

Run: `uv run pytest && uv run ruff check && uv run mypy src`
Expected: all green. Confirm the total test count increased from 103 by the number of new tests (1 services + 12 devices + 4 channels = 17 → ~120 tests).

- [ ] **Step 4: Manual smoke test (optional but recommended)**

```bash
HOME_SERVER_DEBUG=1 HOME_SERVER_PORT=5001 HOME_SERVER_ADMIN_PASSWORD=admin123 uv run python -m home_server
```

In a browser at `http://127.0.0.1:5001`: log in as `admin`/`admin123`, open **Devices**, add a device (`AA:BB:CC:DD:EE:FF` / "Test"), open its detail, add a channel, delete the channel, delete the device. Confirm each step round-trips with a full page reload. (Note: port 5000 is taken by AirPlay on this Mac.)

- [ ] **Step 5: Commit**

```bash
git add tests/test_web_devices.py
git commit -m "test(web): lock in CSRF enforcement on device POST"
```

- [ ] **Step 6: Update progress doc (parent repo)**

In the **parent repo** (`/Users/eric/ESLAB/Intelligent-home`), edit `docs/RPi-Server 開發文件.md` §11.1: mark sub-phase **3d** as ✅ 完成 and update the test count. Then commit the doc and the bumped submodule pointer:

```bash
cd /Users/eric/ESLAB/Intelligent-home
git add "docs/RPi-Server 開發文件.md" Intelligent-home-RPi-server
git commit -m "docs: mark RPi-server phase 3d complete"
```

---

## Self-Review

**1. Spec coverage:**
- Service wiring (spec §2.1) → Task 1. Typed accessors (§2.2) → Task 1.
- `GET /devices` list (§3.1) → Task 2. `GET /devices/<id>` detail + 404 (§3.1) → Task 3.
- `POST /devices` add + invalid-MAC/duplicate errors (§3.1, §5) → Task 4.
- `POST /devices/<id>/delete` + cascade (§3.1) → Task 5.
- `POST /devices/<id>/channels` add + duplicate-name error (§3.2, §5) → Task 6.
- `POST /channels/<id>/delete` (§3.2) → Task 7.
- Forms (§4) → Tasks 4, 6. Flat ownership (§2.4) → Task 4 (`owner_user_id = current_user`), list uses `list_devices`/`list_all`.
- Templates + nav link (§6) → Tasks 2, 3, 4, 5, 6, 7.
- POST-form delete mechanism (§2.3) → Tasks 5, 7. CSRF (§7) → Task 8.
- Tests + `logged_in_client` fixture (§7) → all tasks; fixture in Task 2.
- All spec requirements map to a task. No gaps.

**2. Placeholder scan:** No "TBD"/"add error handling"/"similar to Task N". Every code step shows complete code. ✓

**3. Type consistency:**
- Blueprint endpoints referenced consistently: `devices.list_devices`, `devices.detail`, `devices.delete`, `channels.add_channel`, `channels.delete_channel`.
- Service accessors `get_device_service()` / `get_channel_service()` named identically across Tasks 1–7.
- Extension keys `DEVICE_SERVICE_KEY` / `CHANNEL_SERVICE_KEY` defined in Task 1, used in Task 1 wiring.
- Exceptions match source: `InvalidAddressError` (from `services.device_service`), `DuplicateAddressError` / `DeviceNotFoundError` (from `db.devices`), `DuplicateChannelNameError` (from `db.channels`).
- `detail` route's `render_template` passes `formats` from Task 6 onward — Task 6 Step 5 updates the Task 3 version; consistent.

No issues found.
