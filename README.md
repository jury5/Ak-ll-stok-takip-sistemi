from fastapi import FastAPI, Request, Form
from fastapi.responses import HTMLResponse, RedirectResponse, JSONResponse
from fastapi.templating import Jinja2Templates
from starlette.middleware.sessions import SessionMiddleware
import sqlite3
import hashlib
import time
import secrets
import string

app = FastAPI()
app.add_middleware(SessionMiddleware, secret_key="CHANGE_THIS_TO_SOMETHING_RANDOM")
templates = Jinja2Templates(directory="templates")

DB_NAME = "users.db"

# -----------------------------
# Database setup + migration
# -----------------------------
def init_db():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()

    # base table
    c.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE,
            password_hash TEXT
        )
    """)

    # migrate: add missing columns safely
    c.execute("PRAGMA table_info(users)")
    cols = {row[1] for row in c.fetchall()}

    if "full_name" not in cols:
        c.execute("ALTER TABLE users ADD COLUMN full_name TEXT DEFAULT ''")
    if "employee_id" not in cols:
        c.execute("ALTER TABLE users ADD COLUMN employee_id TEXT DEFAULT ''")
    if "is_admin" not in cols:
        c.execute("ALTER TABLE users ADD COLUMN is_admin INTEGER DEFAULT 0")

    # ✅ PRODUCTS TABLE
    c.execute("""
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            product_code TEXT NOT NULL,
            product_name TEXT NOT NULL,
            product_weight REAL NOT NULL,
            quantity INTEGER NOT NULL,
            location TEXT NOT NULL,
            entry_date TEXT NOT NULL,
            exit_date TEXT DEFAULT '',
            created_by_user_id INTEGER NOT NULL,
            created_at INTEGER NOT NULL,
            FOREIGN KEY(created_by_user_id) REFERENCES users(id)
        )
    """)

    # ✅ migrate products: add shelf fields if missing
    c.execute("PRAGMA table_info(products)")
    pcols = {row[1] for row in c.fetchall()}

    if "shelf_num" not in pcols:
        c.execute("ALTER TABLE products ADD COLUMN shelf_num INTEGER DEFAULT 0")
    if "shelf_unit" not in pcols:
        c.execute("ALTER TABLE products ADD COLUMN shelf_unit INTEGER DEFAULT 0")
    if "shelf_id" not in pcols:
        c.execute("ALTER TABLE products ADD COLUMN shelf_id TEXT DEFAULT ''")
    if "exit_flag" not in pcols:
        c.execute("ALTER TABLE products ADD COLUMN exit_flag INTEGER DEFAULT 0")

    # ✅ EVENTS TABLE (movement/history log)
    c.execute("""
        CREATE TABLE IF NOT EXISTS events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            event_type TEXT NOT NULL,
            message TEXT NOT NULL,
            created_by_user_id INTEGER DEFAULT 0,
            created_at INTEGER NOT NULL
        )
    """)

    conn.commit()
    conn.close()


def hash_password(password: str):
    return hashlib.sha256(password.encode()).hexdigest()

def create_user(username: str, password: str, full_name: str, employee_id: str, is_admin: int):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    try:
        c.execute(
            "INSERT INTO users (username, password_hash, full_name, employee_id, is_admin) VALUES (?, ?, ?, ?, ?)",
            (username, hash_password(password), full_name, employee_id, is_admin)
        )
        conn.commit()
    except:
        pass
    conn.close()

def seed_dummy_users():
    # admin
    create_user("admin", "1234", "Admin Kullanıcı", "A-0001", 1)

    # hayali örnek kullanıcılar
    create_user("ahmet.yilmaz", "1234", "Ahmet Yılmaz", "U-1001", 0)
    create_user("ayse.kaya", "1234", "Ayşe Kaya", "U-1002", 0)
    create_user("mehmet.demir", "1234", "Mehmet Demir", "U-1003", 0)

init_db()
seed_dummy_users()

# -----------------------------
# Helpers
# -----------------------------
def get_product_by_id(product_id: int):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        SELECT
            id, product_code, product_name, product_weight, quantity, location,
            entry_date, exit_date, created_by_user_id
        FROM products
        WHERE id = ?
    """, (int(product_id),))
    row = c.fetchone()
    conn.close()
    return row

def get_product_by_code(product_code: str):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        SELECT
            id, product_code, product_name, product_weight, quantity, location,
            entry_date, exit_date, created_by_user_id
        FROM products
        WHERE product_code = ?
        ORDER BY id DESC
        LIMIT 1
    """, ((product_code or "").strip(),))
    row = c.fetchone()
    conn.close()
    return row

def delete_product_by_id(product_id: int):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("DELETE FROM products WHERE id = ?", (int(product_id),))
    conn.commit()
    conn.close()

def update_product(
    product_id: int,
    product_code: str,
    product_name: str,
    product_weight: float,
    quantity: int,
    location: str,
    entry_date: str,
    exit_date: str
):
    shelf_num, shelf_unit, shelf_id = parse_location(location)
    exit_flag = 1 if (exit_date or "").strip() else 0

    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        UPDATE products
        SET product_code=?,
            product_name=?,
            product_weight=?,
            quantity=?,
            location=?,
            entry_date=?,
            exit_date=?,
            shelf_num=?,
            shelf_unit=?,
            shelf_id=?,
            exit_flag=?
        WHERE id=?
    """, (
        product_code.strip(),
        product_name.strip(),
        float(product_weight),
        int(quantity),
        location.strip(),
        entry_date.strip(),
        (exit_date or "").strip(),
        int(shelf_num),
        int(shelf_unit),
        shelf_id,
        int(exit_flag),
        int(product_id)
    ))
    conn.commit()
    conn.close()

def parse_location(loc: str):
    """
    loc format: 'Y.X' -> Y shelf_num, X shelf_unit
    returns (shelf_num:int, shelf_unit:int, shelf_id:str) or (0,0,'') if invalid
    """
    loc = (loc or "").strip()
    if "." not in loc:
        return 0, 0, ""
    parts = loc.split(".")
    if len(parts) != 2:
        return 0, 0, ""
    y, x = parts[0].strip(), parts[1].strip()
    if not y.isdigit() or not x.isdigit():
        return 0, 0, ""
    shelf_num = int(y)
    shelf_unit = int(x)
    shelf_id = f"S-{shelf_num:03d}-{shelf_unit:03d}"  # ör: S-005-002
    return shelf_num, shelf_unit, shelf_id


def log_event(event_type: str, message: str, created_by_user_id: int = 0):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        INSERT INTO events (event_type, message, created_by_user_id, created_at)
        VALUES (?, ?, ?, ?)
    """, (event_type, message, int(created_by_user_id or 0), int(time.time())))
    conn.commit()
    conn.close()


def list_events(limit: int = 200):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        SELECT e.id, e.event_type, e.message, e.created_at,
               u.username, u.full_name, u.employee_id
        FROM events e
        LEFT JOIN users u ON u.id = e.created_by_user_id
        ORDER BY e.id DESC
        LIMIT ?
    """, (int(limit),))
    rows = c.fetchall()
    conn.close()
    return rows


def list_products_for_stock():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        SELECT
            p.id,                      -- product id num
            p.product_name,
            p.product_code,
            p.shelf_num,
            p.shelf_unit,
            p.shelf_id,
            u.employee_id,             -- registerer id
            p.entry_date,
            p.quantity,                -- amount
            p.product_weight,          -- weight per unit
            p.exit_date,
            p.exit_flag
        FROM products p
        LEFT JOIN users u ON u.id = p.created_by_user_id
        ORDER BY p.id DESC
    """)
    rows = c.fetchall()
    conn.close()
    return rows

def get_user_id_by_username(username: str) -> int:
    row = get_user_by_username(username)
    return int(row[0]) if row else 0

def create_product(
    product_code: str,
    product_name: str,
    product_weight: float,
    quantity: int,
    location: str,
    entry_date: str,
    exit_date: str,
    created_by_user_id: int
):
    shelf_num, shelf_unit, shelf_id = parse_location(location)
    exit_flag = 1 if (exit_date or "").strip() else 0

    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        INSERT INTO products
        (product_code, product_name, product_weight, quantity, location, entry_date, exit_date,
         created_by_user_id, created_at, shelf_num, shelf_unit, shelf_id, exit_flag)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        product_code.strip(),
        product_name.strip(),
        float(product_weight),
        int(quantity),
        location.strip(),
        entry_date.strip(),
        (exit_date or '').strip(),
        int(created_by_user_id),
        int(time.time()),
        int(shelf_num),
        int(shelf_unit),
        shelf_id,
        int(exit_flag)
    ))
    conn.commit()
    conn.close()


def get_user_by_username(username: str):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute(
        "SELECT id, username, full_name, employee_id, is_admin, password_hash FROM users WHERE username = ?",
        (username,)
    )
    row = c.fetchone()
    conn.close()
    return row

def list_users():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute(
        "SELECT id, username, full_name, employee_id, is_admin FROM users ORDER BY is_admin DESC, id ASC"
    )
    rows = c.fetchall()
    conn.close()
    return rows

def require_login(request: Request):
    return request.session.get("username")

def is_admin_user(username: str) -> bool:
    row = get_user_by_username(username)
    return bool(row and row[4] == 1)

def generate_password(length: int = 12) -> str:
    alphabet = string.ascii_letters + string.digits
    return "".join(secrets.choice(alphabet) for _ in range(length))

# -----------------------------
# Routes: Auth
# -----------------------------
@app.get("/", response_class=HTMLResponse)
def login_page(request: Request):
    return templates.TemplateResponse("login.html", {"request": request, "error": None})

@app.post("/login")
def login(
    request: Request,
    username: str = Form(...),
    password: str = Form(...)
):
    row = get_user_by_username(username)
    if not row:
        return templates.TemplateResponse("login.html", {"request": request, "error": "ACCESS DENIED"})

    stored_hash = row[5]
    if hash_password(password) != stored_hash:
        return templates.TemplateResponse("login.html", {"request": request, "error": "ACCESS DENIED"})

    request.session["username"] = username
    request.session.pop("admin_verified_ts", None)
    return RedirectResponse(url="/dashboard", status_code=303)

@app.get("/logout")
def logout(request: Request):
    request.session.clear()
    return RedirectResponse(url="/", status_code=303)

# -----------------------------
# Routes: Dashboard
# -----------------------------
@app.get("/dashboard", response_class=HTMLResponse)
def dashboard(request: Request):
    username = require_login(request)
    if not username:
        return RedirectResponse(url="/", status_code=303)

    return templates.TemplateResponse("dashboard.html", {"request": request, "username": username})
# -----------------------------
# Placeholder pages
# -----------------------------
@app.get("/stok-durum", response_class=HTMLResponse)
def stok_durum(request: Request):
    if not require_login(request):
        return RedirectResponse(url="/", status_code=303)

    products = list_products_for_stock()
    events = list_events(200)

    return templates.TemplateResponse("stok_durum.html", {
        "request": request,
        "products": products,
        "events": events
    })

@app.get("/stok-guncelle", response_class=HTMLResponse)
def stok_guncelle(request: Request):
    if not require_login(request):
        return RedirectResponse(url="/", status_code=303)

    products = list_products_for_stock()
    events = list_events(200)

    return templates.TemplateResponse("stok_guncelle.html", {
        "request": request,
        "products": products,
        "events": events
    })


@app.get("/yeni-urun", response_class=HTMLResponse)
def yeni_urun(request: Request):
    if not require_login(request):
        return RedirectResponse(url="/", status_code=303)
    return templates.TemplateResponse("yeni_urun.html", {"request": request})

@app.get("/urun-barkod", response_class=HTMLResponse)
def urun_barkod(request: Request):
    if not require_login(request):
        return RedirectResponse(url="/", status_code=303)
    return templates.TemplateResponse("urun-barkod.html", {"request": request})

# -----------------------------
# Users page (UI with modal + X)
# -----------------------------
@app.get("/kullanici-bilgileri", response_class=HTMLResponse)
def kullanici_bilgileri(request: Request):
    username = require_login(request)
    if not username:
        return RedirectResponse(url="/", status_code=303)

    users = list_users()
    return templates.TemplateResponse(
        "kullanici_bilgileri.html",
        {
            "request": request,
            "username": username,
            "is_admin": is_admin_user(username),
            "users": users
        }
    )

# -----------------------------
# API: Add user (auto password)
# -----------------------------
@app.post("/api/users/add")
def api_users_add(
    request: Request,
    full_name: str = Form(...),
    employee_id: str = Form(...),
    new_username: str = Form(...)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    if not is_admin_user(username):
        return JSONResponse({"ok": False, "error": "not_admin"}, status_code=403)

    if get_user_by_username(new_username):
        return JSONResponse({"ok": False, "error": "username_exists"}, status_code=400)

    generated_password = generate_password(12)
    create_user(new_username, generated_password, full_name, employee_id, 0)

    admin_user_id = get_user_id_by_username(username)
    log_event("user_added", f"Yeni kullanıcı eklendi: {new_username} | ID: {employee_id} | Ad: {full_name}", admin_user_id)

    return JSONResponse({"ok": True, "generated_password": generated_password})
# -----------------------------
# API: Delete user
# -----------------------------
@app.post("/api/users/delete")
def api_users_delete(
    request: Request,
    user_id: int = Form(...)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    if not is_admin_user(username):
        return JSONResponse({"ok": False, "error": "not_admin"}, status_code=403)

    row = get_user_by_username(username)
    current_user_id = row[0]
    if int(user_id) == int(current_user_id):
        return JSONResponse({"ok": False, "error": "cannot_delete_self"}, status_code=400)

    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("DELETE FROM users WHERE id = ?", (user_id,))
    conn.commit()
    conn.close()

    return JSONResponse({"ok": True})


@app.post("/api/products/add")
def api_products_add(
    request: Request,
    product_code: str = Form(...),
    product_name: str = Form(...),
    product_weight: float = Form(...),
    quantity: int = Form(...),
    location: str = Form(...),
    entry_date: str = Form(...),
    exit_date: str = Form(None)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    if not product_code.strip() or not product_name.strip() or not location.strip() or not entry_date.strip():
        return JSONResponse({"ok": False, "error": "missing_fields"}, status_code=400)

    if float(product_weight) <= 0:
        return JSONResponse({"ok": False, "error": "invalid_weight"}, status_code=400)

    if int(quantity) <= 0:
        return JSONResponse({"ok": False, "error": "invalid_quantity"}, status_code=400)

    created_by_user_id = get_user_id_by_username(username)
    if not created_by_user_id:
        return JSONResponse({"ok": False, "error": "user_not_found"}, status_code=400)

    create_product(
        product_code=product_code,
        product_name=product_name,
        product_weight=product_weight,
        quantity=quantity,
        location=location,
        entry_date=entry_date,
        exit_date=exit_date or "",
        created_by_user_id=created_by_user_id
    )
    # log
    if parse_location(location)[0] == 0:
        log_event("warning", f"Ürün kaydı yapıldı ama lokasyon formatı hatalı: '{location}' (beklenen: Y.X)", created_by_user_id)
    else:
        log_event("product_registered", f"Yeni ürün kaydedildi: {product_name} | Kod: {product_code} | Lokasyon: {location}", created_by_user_id)
        
    return JSONResponse({"ok": True})

@app.post("/api/products/update")
def api_products_update(
    request: Request,
    product_id: int = Form(...),
    product_code: str = Form(...),
    product_name: str = Form(...),
    product_weight: float = Form(...),
    quantity: int = Form(...),
    location: str = Form(...),
    entry_date: str = Form(...),
    exit_date: str = Form(None)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    # validation
    if not product_code.strip() or not product_name.strip() or not location.strip() or not entry_date.strip():
        return JSONResponse({"ok": False, "error": "missing_fields"}, status_code=400)

    if float(product_weight) <= 0:
        return JSONResponse({"ok": False, "error": "invalid_weight"}, status_code=400)

    if int(quantity) < 0:
        return JSONResponse({"ok": False, "error": "invalid_quantity"}, status_code=400)

    old = get_product_by_id(int(product_id))
    if not old:
        return JSONResponse({"ok": False, "error": "not_found"}, status_code=404)

    # update
    update_product(
        product_id=int(product_id),
        product_code=product_code,
        product_name=product_name,
        product_weight=product_weight,
        quantity=quantity,
        location=location,
        entry_date=entry_date,
        exit_date=exit_date or ""
    )

    editor_user_id = get_user_id_by_username(username)

    # diff message (eski -> yeni)
    old_loc = old[5]
    old_weight = old[3]
    old_quantity = old[4]

    msg = (
        f"Ürün güncellendi (ID: {product_id}) | "
        f"Ad: '{old[2]}' -> '{product_name}' | "
        f"Kod: '{old[1]}' -> '{product_code}' | "
        f"Lokasyon: '{old_loc}' -> '{location}' | "
        f"Adet: '{old_quantity}' -> '{quantity}' | "
        f"Ağırlık: '{old_weight}' -> '{product_weight}'"

    )

    # location format warning vs normal update
    if parse_location(location)[0] == 0:
        log_event("warning", f"Güncelleme yapıldı ama lokasyon formatı hatalı: '{location}' (beklenen: Y.X) | Ürün ID: {product_id}", editor_user_id)
    log_event("product_updated", msg, editor_user_id)

    return JSONResponse({"ok": True})

@app.post("/api/products/add_by_barcode")
def api_products_add_by_barcode(
    request: Request,
    barcode: str = Form(...),
    product_name: str = Form(...),
    product_weight: float = Form(...),
    quantity: int = Form(...),
    location: str = Form(...),
    entry_date: str = Form(...),
    exit_date: str = Form(None)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    if not (barcode or "").strip() or not product_name.strip() or not location.strip() or not entry_date.strip():
        return JSONResponse({"ok": False, "error": "missing_fields"}, status_code=400)

    if float(product_weight) <= 0:
        return JSONResponse({"ok": False, "error": "invalid_weight"}, status_code=400)

    if int(quantity) <= 0:
        return JSONResponse({"ok": False, "error": "invalid_quantity"}, status_code=400)

    created_by_user_id = get_user_id_by_username(username)
    if not created_by_user_id:
        return JSONResponse({"ok": False, "error": "user_not_found"}, status_code=400)

    create_product(
        product_code=(barcode or "").strip(),
        product_name=product_name,
        product_weight=product_weight,
        quantity=quantity,
        location=location,
        entry_date=entry_date,
        exit_date=exit_date or "",
        created_by_user_id=created_by_user_id
    )
    if parse_location(location)[0] == 0:
        log_event("warning", f"Barkod ile Ã¼rÃ¼n kaydÄ± ama lokasyon formatÄ± hatalÄ±: '{location}' (beklenen: Y.X)", created_by_user_id)
    else:
        log_event("product_registered", f"Barkod ile yeni Ã¼rÃ¼n: {product_name} | Barkod: {(barcode or '').strip()} | Lokasyon: {location}", created_by_user_id)

    return JSONResponse({"ok": True})

@app.post("/api/products/info_by_barcode")
def api_products_info_by_barcode(
    request: Request,
    barcode: str = Form(...)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    row = get_product_by_code((barcode or "").strip())
    if not row:
        return JSONResponse({"ok": False, "error": "not_found"}, status_code=404)

    return JSONResponse({
        "ok": True,
        "product": {
            "id": row[0],
            "product_code": row[1],
            "product_name": row[2],
            "product_weight": row[3],
            "quantity": row[4],
            "location": row[5],
            "entry_date": row[6],
            "exit_date": row[7]
        }
    })

@app.post("/api/products/delete_by_barcode")
def api_products_delete_by_barcode(
    request: Request,
    barcode: str = Form(...)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    row = get_product_by_code((barcode or "").strip())
    if not row:
        return JSONResponse({"ok": False, "error": "not_found"}, status_code=404)

    delete_product_by_id(int(row[0]))
    deleter_id = get_user_id_by_username(username)
    log_event("product_deleted", f"Barkod ile Ã¼rÃ¼n silindi: {row[2]} | Barkod: {row[1]}", deleter_id)

    return JSONResponse({"ok": True})

@app.post("/api/products/update_by_barcode")
def api_products_update_by_barcode(
    request: Request,
    barcode: str = Form(...),
    product_name: str = Form(...),
    product_weight: float = Form(...),
    quantity: int = Form(...),
    location: str = Form(...),
    entry_date: str = Form(...),
    exit_date: str = Form(None)
):
    username = require_login(request)
    if not username:
        return JSONResponse({"ok": False, "error": "not_logged_in"}, status_code=401)

    if not (barcode or "").strip() or not product_name.strip() or not location.strip() or not entry_date.strip():
        return JSONResponse({"ok": False, "error": "missing_fields"}, status_code=400)

    if float(product_weight) <= 0:
        return JSONResponse({"ok": False, "error": "invalid_weight"}, status_code=400)

    if int(quantity) < 0:
        return JSONResponse({"ok": False, "error": "invalid_quantity"}, status_code=400)

    old = get_product_by_code((barcode or "").strip())
    if not old:
        return JSONResponse({"ok": False, "error": "not_found"}, status_code=404)

    update_product(
        product_id=int(old[0]),
        product_code=(barcode or "").strip(),
        product_name=product_name,
        product_weight=product_weight,
        quantity=quantity,
        location=location,
        entry_date=entry_date,
        exit_date=exit_date or ""
    )

    editor_user_id = get_user_id_by_username(username)
    msg = (
        f"Barkod ile Ã¼rÃ¼n gÃ¼ncellendi (ID: {old[0]}) | "
        f"Ad: '{old[2]}' -> '{product_name}' | "
        f"Lokasyon: '{old[5]}' -> '{location}' | "
        f"Adet: '{old[4]}' -> '{quantity}' | "
        f"AÄŸÄ±rlÄ±k: '{old[3]}' -> '{product_weight}'"
    )
    if parse_location(location)[0] == 0:
        log_event("warning", f"Barkod gÃ¼ncelleme ama lokasyon formatÄ± hatalÄ±: '{location}' (beklenen: Y.X) | ÃœrÃ¼n ID: {old[0]}", editor_user_id)
    log_event("product_updated", msg, editor_user_id)

    return JSONResponse({"ok": True})
