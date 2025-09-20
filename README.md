# WMS Demo – Pallet/Box/Folder Index (Flask + Postgres)

A minimal, **presentable** demo of a warehouse indexing system for **pallet → box → folder/paper** with location hierarchy (**warehouse → shelf → pallet slot**), statuses, audit logs, and barcode/QR support.

This is designed for a quick **demo & presentation** to your boss, but with a clean path to production:
- Modular Flask backend
- PostgreSQL with SQLAlchemy models + Alembic migration
- JWT auth for demo (swap to LDAP/AD/OIDC later)
- REST API + example requests
- QR/Barcode generator endpoint
- Seed script to showcase a real flow

---

## 1) Data Model (ERD & Concepts)

```
[Warehouse] 1───* [Shelf] 1───* [PalletSlot]
                               │
                               └── 0..1 ── holds ── [Pallet] 1───* [Box] 1───* [Folder]

[Shipment] 1───* [ShipmentItem] ──→ references Pallet(s) (and counts)

[AuditLog] tracks create/update/move/checkout events (user, ts, action, object, before/after)

[User] ── [Role] ── [Permission]  (demo via JWT; prod via LDAP/AD/OIDC)
```

### Core Tables
- **warehouse**(id, code, name)
- **shelf**(id, warehouse_id, code, name)
- **pallet_slot**(id, shelf_id, code, position_idx, is_active)
- **pallet**(id, code, status, slot_id, shipment_id, nda_flag, iso_cert, notes)
- **box**(id, pallet_id, code, label, nda_flag)
- **folder**(id, box_id, code, title, doc_count, keywords)
- **shipment**(id, supplier, bill_of_lading, arrival_at, drive_id, pallet_expected, pallet_received)
- **shipment_item**(id, shipment_id, pallet_code, box_count_expected, box_count_received)
- **audit_log**(id, ts, user_id, action, entity, entity_id, before, after)

### Enums / Status
- **pallet.status** ∈ {`arriving`, `warehoused`, `unpacked`, `repacked`, `outgoing`}

### Indexing/IDs
- Use short codes printable as QR/Barcode:
  - Warehouse: `W-{code}` (e.g., `W-MTL-01`)
  - Shelf: `SH-{warehouseCode}-{code}`
  - Slot: `SL-{shelfCode}-{pos}`
  - Pallet: `PL-{yyyymm}-{seq}`
  - Box: `BX-{palletCode}-{seq}`
  - Folder: `FD-{boxCode}-{seq}`

---

## 2) PostgreSQL Schema (DDL)

```sql
CREATE TYPE pallet_status AS ENUM ('arriving','warehoused','unpacked','repacked','outgoing');

CREATE TABLE warehouse (
  id SERIAL PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL
);

CREATE TABLE shelf (
  id SERIAL PRIMARY KEY,
  warehouse_id INT NOT NULL REFERENCES warehouse(id) ON DELETE CASCADE,
  code TEXT NOT NULL,
  name TEXT,
  UNIQUE (warehouse_id, code)
);

CREATE TABLE pallet_slot (
  id SERIAL PRIMARY KEY,
  shelf_id INT NOT NULL REFERENCES shelf(id) ON DELETE CASCADE,
  code TEXT NOT NULL,
  position_idx INT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  UNIQUE (shelf_id, code)
);

CREATE TABLE shipment (
  id SERIAL PRIMARY KEY,
  supplier TEXT NOT NULL,
  bill_of_lading TEXT,
  arrival_at TIMESTAMP WITH TIME ZONE NOT NULL,
  drive_id TEXT, -- external ref (e.g., IP drive / Google Drive / Share)
  pallet_expected INT,
  pallet_received INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE shipment_item (
  id SERIAL PRIMARY KEY,
  shipment_id INT NOT NULL REFERENCES shipment(id) ON DELETE CASCADE,
  pallet_code TEXT NOT NULL,
  box_count_expected INT,
  box_count_received INT DEFAULT 0
);

CREATE TABLE pallet (
  id SERIAL PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,
  status pallet_status NOT NULL DEFAULT 'arriving',
  slot_id INT REFERENCES pallet_slot(id) ON DELETE SET NULL,
  shipment_id INT REFERENCES shipment(id) ON DELETE SET NULL,
  nda_flag BOOLEAN NOT NULL DEFAULT FALSE,
  iso_cert TEXT,
  notes TEXT
);

CREATE TABLE box (
  id SERIAL PRIMARY KEY,
  pallet_id INT NOT NULL REFERENCES pallet(id) ON DELETE CASCADE,
  code TEXT UNIQUE NOT NULL,
  label TEXT,
  nda_flag BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE folder (
  id SERIAL PRIMARY KEY,
  box_id INT NOT NULL REFERENCES box(id) ON DELETE CASCADE,
  code TEXT UNIQUE NOT NULL,
  title TEXT,
  doc_count INT DEFAULT 0,
  keywords TEXT
);

CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  ts TIMESTAMPTZ NOT NULL DEFAULT now(),
  user_id TEXT,
  action TEXT NOT NULL,
  entity TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  before JSONB,
  after JSONB
);

-- Useful indexes
CREATE INDEX idx_pallet_status ON pallet(status);
CREATE INDEX idx_pallet_slot ON pallet(slot_id);
CREATE INDEX idx_box_pallet ON box(pallet_id);
CREATE INDEX idx_folder_box ON folder(box_id);
```

---

## 3) Minimal Flask App (Modular)

**Structure**
```
backend/
  app.py
  database.py
  auth.py
  models.py
  routes/
    pallets.py
    boxes.py
    folders.py
    locations.py
    shipments.py
    utils.py
  services/
    qr.py
    audit.py
  seeds.py
  alembic/  (after init)
```

### `app.py`
```python
from flask import Flask
from flask_cors import CORS
from auth import jwt_init
from database import db, migrate
from routes.pallets import pallets_bp
from routes.boxes import boxes_bp
from routes.folders import folders_bp
from routes.locations import locations_bp
from routes.shipments import shipments_bp


def create_app():
    app = Flask(__name__)
    app.config.update(
        SQLALCHEMY_DATABASE_URI='postgresql+psycopg://wms:wms@localhost:5432/wms',
        SQLALCHEMY_TRACK_MODIFICATIONS=False,
        JWT_SECRET_KEY='dev-secret-change',
    )

    CORS(app)
    db.init_app(app)
    migrate.init_app(app, db)
    jwt_init(app)

    app.register_blueprint(pallets_bp, url_prefix='/api/pallets')
    app.register_blueprint(boxes_bp, url_prefix='/api/boxes')
    app.register_blueprint(folders_bp, url_prefix='/api/folders')
    app.register_blueprint(locations_bp, url_prefix='/api/locations')
    app.register_blueprint(shipments_bp, url_prefix='/api/shipments')

    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
```

### `database.py`
```python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()
```

### `auth.py` (demo JWT; swap with Authentik/Keycloak later)
```python
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity
from flask import Blueprint, request, jsonify

jwt = JWTManager()
auth_bp = Blueprint('auth', __name__)

def jwt_init(app):
    jwt.init_app(app)
    app.register_blueprint(auth_bp, url_prefix='/api/auth')

@auth_bp.post('/login')
def login():
    data = request.json or {}
    # demo only: accept any username, no password
    token = create_access_token(identity=data.get('username','demo'))
    return jsonify(access_token=token)
```

### `models.py`
```python
from database import db
from sqlalchemy.dialects.postgresql import JSONB, ENUM

pallet_status = ENUM('arriving','warehoused','unpacked','repacked','outgoing', name='pallet_status', create_type=False)

class Warehouse(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    code = db.Column(db.String, unique=True, nullable=False)
    name = db.Column(db.String, nullable=False)

class Shelf(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    warehouse_id = db.Column(db.Integer, db.ForeignKey('warehouse.id', ondelete='CASCADE'), nullable=False)
    code = db.Column(db.String, nullable=False)
    name = db.Column(db.String)
    __table_args__ = (db.UniqueConstraint('warehouse_id','code', name='uq_shelf_wh_code'),)

class PalletSlot(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    shelf_id = db.Column(db.Integer, db.ForeignKey('shelf.id', ondelete='CASCADE'), nullable=False)
    code = db.Column(db.String, nullable=False)
    position_idx = db.Column(db.Integer, nullable=False)
    is_active = db.Column(db.Boolean, default=True, nullable=False)
    __table_args__ = (db.UniqueConstraint('shelf_id','code', name='uq_slot_shelf_code'),)

class Shipment(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    supplier = db.Column(db.String, nullable=False)
    bill_of_lading = db.Column(db.String)
    arrival_at = db.Column(db.DateTime(timezone=True), nullable=False)
    drive_id = db.Column(db.String)
    pallet_expected = db.Column(db.Integer)
    pallet_received = db.Column(db.Integer, default=0)
    created_at = db.Column(db.DateTime(timezone=True), server_default=db.func.now())

class ShipmentItem(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    shipment_id = db.Column(db.Integer, db.ForeignKey('shipment.id', ondelete='CASCADE'), nullable=False)
    pallet_code = db.Column(db.String, nullable=False)
    box_count_expected = db.Column(db.Integer)
    box_count_received = db.Column(db.Integer, default=0)

class Pallet(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    code = db.Column(db.String, unique=True, nullable=False)
    status = db.Column(pallet_status, nullable=False, server_default='arriving')
    slot_id = db.Column(db.Integer, db.ForeignKey('pallet_slot.id', ondelete='SET NULL'))
    shipment_id = db.Column(db.Integer, db.ForeignKey('shipment.id', ondelete='SET NULL'))
    nda_flag = db.Column(db.Boolean, default=False, nullable=False)
    iso_cert = db.Column(db.String)
    notes = db.Column(db.Text)

class Box(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    pallet_id = db.Column(db.Integer, db.ForeignKey('pallet.id', ondelete='CASCADE'), nullable=False)
    code = db.Column(db.String, unique=True, nullable=False)
    label = db.Column(db.String)
    nda_flag = db.Column(db.Boolean, default=False, nullable=False)

class Folder(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    box_id = db.Column(db.Integer, db.ForeignKey('box.id', ondelete='CASCADE'), nullable=False)
    code = db.Column(db.String, unique=True, nullable=False)
    title = db.Column(db.String)
    doc_count = db.Column(db.Integer, default=0)
    keywords = db.Column(db.Text)

class AuditLog(db.Model):
    id = db.Column(db.BigInteger, primary_key=True)
    ts = db.Column(db.DateTime(timezone=True), server_default=db.func.now(), nullable=False)
    user_id = db.Column(db.String)
    action = db.Column(db.String, nullable=False)
    entity = db.Column(db.String, nullable=False)
    entity_id = db.Column(db.String, nullable=False)
    before = db.Column(JSONB)
    after = db.Column(JSONB)
```

### Example Routes (Pallets)
`routes/pallets.py`
```python
from flask import Blueprint, request, jsonify
from flask_jwt_extended import jwt_required, get_jwt_identity
from database import db
from models import Pallet, AuditLog

pallets_bp = Blueprint('pallets', __name__)

@pallets_bp.post('/')
@jwt_required()
def create_pallet():
    data = request.json or {}
    p = Pallet(
        code=data['code'],
        status=data.get('status','arriving'),
        slot_id=data.get('slot_id'),
        shipment_id=data.get('shipment_id'),
        nda_flag=bool(data.get('nda_flag', False)),
        iso_cert=data.get('iso_cert'),
        notes=data.get('notes')
    )
    db.session.add(p)
    db.session.flush()
    db.session.add(AuditLog(user_id=get_jwt_identity(), action='create', entity='pallet', entity_id=str(p.id), before=None, after={'code':p.code,'status':p.status}))
    db.session.commit()
    return jsonify(id=p.id, code=p.code), 201

@pallets_bp.get('/')
@jwt_required()
def list_pallets():
    q = Pallet.query
    status = request.args.get('status')
    if status:
        q = q.filter(Pallet.status == status)
    rows = q.order_by(Pallet.id.desc()).limit(200).all()
    return jsonify([{'id':r.id,'code':r.code,'status':r.status} for r in rows])

@pallets_bp.post('/move/<int:pallet_id>')
@jwt_required()
def move_pallet(pallet_id):
    p = Pallet.query.get_or_404(pallet_id)
    data = request.json or {}
    before = {'slot_id': p.slot_id, 'status': p.status}
    p.slot_id = data.get('slot_id', p.slot_id)
    if 'status' in data:
        p.status = data['status']
    db.session.add(p)
    db.session.flush()
    db.session.add(AuditLog(user_id=get_jwt_identity(), action='move', entity='pallet', entity_id=str(p.id), before=before, after={'slot_id':p.slot_id,'status':p.status}))
    db.session.commit()
    return jsonify({'ok': True})
```

You can mirror this pattern for `boxes.py`, `folders.py`, `locations.py` (CRUD for Warehouse/Shelf/Slot), and `shipments.py` (receive shipments, reconcile expected vs received).

---

## 4) QR/Barcode Utilities

`services/qr.py`
```python
import io
import qrcode
from flask import Blueprint, send_file

qr_bp = Blueprint('qr', __name__)

@qr_bp.get('/qr/<path:payload>')
def qr(payload):
    img = qrcode.make(payload)
    buf = io.BytesIO()
    img.save(buf, format='PNG')
    buf.seek(0)
    return send_file(buf, mimetype='image/png')
```
Register `qr_bp` like others (e.g., under `/api/utils`). For barcodes, you can use `python-barcode`.

**QR Payload Convention**
- Encode JSON or a compact string, e.g.: `PL|code=PL-202509|id=123`.
- Scanning app posts to `/api/pallets/lookup?code=PL-202509` to fetch details.

---

## 5) Seed Script (Demo Flow)

`seeds.py`
```python
from app import create_app
from database import db
from models import Warehouse, Shelf, PalletSlot, Shipment, ShipmentItem, Pallet, Box, Folder
from datetime import datetime

app = create_app()
with app.app_context():
    db.drop_all()
    db.create_all()

    w = Warehouse(code='MTL-01', name='Montreal Main')
    db.session.add(w)
    s1 = Shelf(warehouse_id=1, code='A', name='Aisle A')
    s2 = Shelf(warehouse_id=1, code='B', name='Aisle B')
    db.session.add_all([s1,s2])

    slots = []
    for i in range(1,6):
        slots.append(PalletSlot(shelf_id=1, code=f'A-{i}', position_idx=i))
    db.session.add_all(slots)

    sh = Shipment(supplier='ACME Docs', bill_of_lading='BOL-123', arrival_at=datetime.utcnow(), drive_id='share://drive01', pallet_expected=2)
    db.session.add(sh)
    db.session.flush()

    db.session.add_all([
        ShipmentItem(shipment_id=sh.id, pallet_code='PL-202509-001', box_count_expected=40),
        ShipmentItem(shipment_id=sh.id, pallet_code='PL-202509-002', box_count_expected=38),
    ])

    p1 = Pallet(code='PL-202509-001', status='warehoused', slot_id=1, shipment_id=sh.id, nda_flag=True, iso_cert='ISO27001')
    p2 = Pallet(code='PL-202509-002', status='arriving', slot_id=2, shipment_id=sh.id)
    db.session.add_all([p1,p2])
    db.session.flush()

    # Boxes for p1
    boxes = []
    for i in range(1,6):
        boxes.append(Box(pallet_id=p1.id, code=f'BX-001-{i:02}', label=f'Client-{i}', nda_flag=True))
    db.session.add_all(boxes)
    db.session.flush()

    # Folders for first box
    for i in range(1,4):
        db.session.add(Folder(box_id=boxes[0].id, code=f'FD-001-01-{i:02}', title=f'Contract-{i}', doc_count=10))

    db.session.commit()
    print('Seeded demo data.')
```

Run: `python seeds.py`

---

## 6) Example API Calls (for the live demo)

1) **Login** (demo JWT)
```
curl -X POST http://localhost:5000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo"}'
```
Use returned `access_token` for next calls: `-H "Authorization: Bearer <token>"`

2) **List pallets / by status**
```
curl -H "Authorization: Bearer $T" http://localhost:5000/api/pallets?status=warehoused
```

3) **Move a pallet to a new slot & update status**
```
curl -X POST http://localhost:5000/api/pallets/move/1 \
  -H "Authorization: Bearer $T" \
  -H 'Content-Type: application/json' \
  -d '{"slot_id":3, "status":"outgoing"}'
```

4) **Generate a QR** for a pallet code
```
open http://localhost:5000/api/utils/qr/PL-202509-001
```

---

## 7) Docker & Local Dev

**docker-compose.yml (dev DB only)**
```yaml
version: '3.9'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: wms
      POSTGRES_PASSWORD: wms
      POSTGRES_DB: wms
    ports:
      - '5432:5432'
    volumes:
      - pg_data:/var/lib/postgresql/data
volumes:
  pg_data:
```

**Install & Run**
```
python -m venv .venv && source .venv/bin/activate
pip install flask flask-cors flask-jwt-extended flask-sqlalchemy flask-migrate psycopg[binary] qrcode pillow
export FLASK_APP=app.py
python seeds.py
python app.py
```

---

## 8) Path to Production

- **AuthN/Z**: Replace demo JWT with OIDC (e.g., Authentik/Keycloak) or LDAP/AD for SSO and group-based RBAC.
- **Audit & Compliance**: Expand `AuditLog` (IP, device, reason) and add report endpoints.
- **Barcode/Label Printing**: Integrate ZPL/EPL for Zebra printers; add `/api/labels/print`.
- **Scanning UX**: Simple React/Vue SPA: scan → auto-lookup → move/update.
- **Attachments**: Optionally link scanned PDFs/photos to `folder` or `box` (S3/MinIO bucket + signed URLs).
- **Multi-warehouse**: Keep a single DB with `warehouse_id` scoping and role-based access; or separate DB per warehouse if you must isolate strictly.
- **Background Jobs**: Use Celery/RQ for big imports and periodic reconciliations.

---

## 9) Slides Outline (for your presentation)

1. **Problem**: Tracking NDA-sensitive pallets/boxes/folders across lifecycle.
2. **Solution**: Lightweight WMS demo with QR, statuses, audit.
3. **Flows**: Receiving → Putaway → Picking/Outgoing.
4. **Security**: NDA flags, ISO fields, role-based access, audit logs.
5. **Live Demo**: Seeded data, list & move pallets, scan QR.
6. **Roadmap**: Printers, floor UI, OIDC/LDAP, mobile scanners, reporting.

---

## 10) Notes & Variations
- If you prefer **one DB per warehouse**, keep the same schema and deploy N instances; add a global **Directory** service that maps warehouse code → DB URL (service discovery).
- For **search speed**, add pgTrgm/GIN indexes on `code`, `keywords`.
- For **box/folder counts**, add triggers/materialized views for quick stats.

---

**You now have a clean demo you can run locally, show endpoints, and talk through the upgrade path to production.**

