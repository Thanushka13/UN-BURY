
import os
import sqlite3
import secrets
from datetime import datetime
from functools import wraps

from flask import (
    Flask, request, redirect, url_for, session,
    flash, render_template_string, abort
)
from werkzeug.security import generate_password_hash, check_password_hash
from markupsafe import Markup

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY", "rewave-local-development-key")
DB = "rewave.db"

CATEGORIES = [
    "Water Leakage", "Water Shortage", "Sewage Overflow",
    "Broken Public Tap", "Suspected Contamination", "Other"
]

STATUSES = [
    "Submitted", "Community Verified", "Under Review",
    "In Progress", "Marked Resolved", "Citizen Confirmed", "Disputed"
]


# ---------------- DATABASE ----------------

def connect():
    db = sqlite3.connect(DB)
    db.row_factory = sqlite3.Row
    db.execute("PRAGMA foreign_keys = ON")
    return db


def init_db():
    with connect() as db:
        db.executescript("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            role TEXT NOT NULL DEFAULT 'citizen'
        );

        CREATE TABLE IF NOT EXISTS reports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            code TEXT UNIQUE NOT NULL,
            user_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            description TEXT NOT NULL,
            category TEXT NOT NULL,
            location TEXT NOT NULL,
            households INTEGER NOT NULL DEFAULT 1,
            severity INTEGER NOT NULL DEFAULT 1,
            duration INTEGER NOT NULL DEFAULT 1,
            status TEXT NOT NULL DEFAULT 'Submitted',
            created TEXT NOT NULL,
            resolved TEXT,
            FOREIGN KEY(user_id) REFERENCES users(id)
        );

        CREATE TABLE IF NOT EXISTS verifications (
            user_id INTEGER NOT NULL,
            report_id INTEGER NOT NULL,
            PRIMARY KEY(user_id, report_id),
            FOREIGN KEY(user_id) REFERENCES users(id),
            FOREIGN KEY(report_id) REFERENCES reports(id)
        );

        CREATE TABLE IF NOT EXISTS history (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            report_id INTEGER NOT NULL,
            status TEXT NOT NULL,
            note TEXT,
            created TEXT NOT NULL,
            FOREIGN KEY(report_id) REFERENCES reports(id)
        );
        """)


def now():
    return datetime.now().strftime("%Y-%m-%d %H:%M")


def query(sql, args=(), one=False):
    with connect() as db:
        rows = db.execute(sql, args).fetchall()
    return rows[0] if one and rows else (None if one else rows)


def user():
    if "uid" not in session:
        return None
    return query("SELECT * FROM users WHERE id=?", (session["uid"],), True)


def login_required(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        if not user():
            return redirect(url_for("login"))
        return fn(*args, **kwargs)
    return wrapper


def authority_required(fn):
    @wraps(fn)
    @login_required
    def wrapper(*args, **kwargs):
        if user()["role"] != "authority":
            abort(403)
        return fn(*args, **kwargs)
    return wrapper


# ---------------- SMART PYTHON LOGIC ----------------

def classify(description):
    s = description.lower()

    rules = [
        ("Sewage Overflow", ["sewage", "drain", "sewer", "overflow"]),
        ("Water Shortage", ["shortage", "no water", "dry tap", "no supply"]),
        ("Broken Public Tap", ["broken tap", "public tap", "tap damaged"]),
        ("Suspected Contamination", ["dirty", "smell", "contaminated", "brown water"]),
        ("Water Leakage", ["leak", "burst pipe", "pipe burst", "dripping"])
    ]

    for category, keywords in rules:
        if any(word in s for word in keywords):
            return category

    return "Other"


def score(report):
    count = query(
        "SELECT COUNT(*) AS n FROM verifications WHERE report_id=?",
        (report["id"],), True
    )["n"]

    value = (
        report["severity"] * 2
        + min(report["households"] // 5, 4)
        + min(report["duration"] // 3, 4)
        + min(count, 4)
    )

    if value >= 10:
        return "High"
    if value >= 5:
        return "Medium"
    return "Low"


def metrics():
    total = query("SELECT COUNT(*) AS n FROM reports", one=True)["n"]

    resolved = query("""
        SELECT COUNT(*) AS n FROM reports
        WHERE status IN ('Marked Resolved','Citizen Confirmed')
    """, one=True)["n"]

    pending = total - resolved

    reports = query("SELECT * FROM reports")
    high = sum(score(r) == "High" for r in reports)

    return {
        "total": total,
        "resolved": resolved,
        "pending": pending,
        "high": high,
        "percentage": round(resolved / total * 100) if total else 0
    }


# ---------------- DESIGN ----------------

CSS = """
:root {
    --navy:#102B3F;
    --teal:#087E8B;
    --aqua:#D9F3F0;
    --white:#F3F7F8;
    --coral:#E76F51;
    --muted:#607680;
}

* { box-sizing:border-box; }

body {
    margin:0;
    background:var(--white);
    color:var(--navy);
    font-family:Arial,Helvetica,sans-serif;
    line-height:1.6;
}

a { color:var(--teal); text-decoration:none; }
a:hover { text-decoration:underline; }

h1,h2,h3 { line-height:1.2; letter-spacing:-.035em; }
h1 { font-size:clamp(2.3rem,5vw,4rem); margin:10px 0; }
h2 { font-size:1.7rem; }
p { color:var(--muted); }

header {
    display:flex;
    justify-content:space-between;
    align-items:center;
    padding:17px 6%;
    background:white;
    border-bottom:1px solid #e0eaec;
    gap:20px;
}

.logo {
    font-size:25px;
    font-weight:900;
    letter-spacing:-1px;
    color:var(--navy);
}

.logo span {
    background:var(--teal);
    color:white;
    padding:5px 10px;
    border-radius:10px;
}

nav { display:flex; gap:18px; align-items:center; flex-wrap:wrap; }
nav a { font-size:14px; font-weight:600; }

main { max-width:1180px; margin:auto; padding:45px 22px 90px; min-height:80vh; }

.btn {
    display:inline-block;
    padding:12px 22px;
    background:var(--teal);
    color:white;
    border:1px solid var(--teal);
    border-radius:11px;
    cursor:pointer;
    font-size:15px;
    font-weight:700;
    text-decoration:none;
}

.btn:hover { background:#066773; color:white; text-decoration:none; }
.btn.secondary { background:white; color:var(--teal); }
.btn.coral { background:var(--coral); border-color:var(--coral); }

.hero {
    display:grid;
    grid-template-columns:1.2fr .8fr;
    align-items:center;
    gap:35px;
    min-height:480px;
}

.hero p { font-size:19px; max-width:550px; }
.hero em { font-style:normal; color:var(--teal); }

.art {
    height:390px;
    border-radius:32px;
    background:var(--aqua);
    display:grid;
    place-items:center;
    position:relative;
    overflow:hidden;
}

.art .wave {
    font-size:220px;
    color:var(--teal);
    transform:rotate(-20deg);
}

.art .float {
    position:absolute;
    bottom:30px;
    background:white;
    padding:20px 30px;
    border-radius:16px;
    box-shadow:0 15px 40px #102b3f20;
    font-weight:bold;
}

.eyebrow {
    color:var(--teal);
    font-weight:900;
    font-size:12px;
    letter-spacing:2px;
}

.cards {
    display:grid;
    grid-template-columns:repeat(3,1fr);
    gap:18px;
    margin:25px 0;
}

.card {
    background:white;
    padding:25px;
    border-radius:19px;
    border:1px solid #e1eaec;
    box-shadow:0 8px 25px #102b3f08;
    margin-bottom:18px;
}

.card h3 { margin:10px 0; }

.banner {
    margin-top:45px;
    background:var(--navy);
    color:white;
    padding:35px;
    border-radius:22px;
}

.banner p { color:var(--aqua); }

.stats {
    display:grid;
    grid-template-columns:repeat(4,1fr);
    gap:15px;
    margin:25px 0 35px;
}

.stats strong {
    display:block;
    font-size:40px;
    line-height:1.3;
}

.stats small { color:var(--muted); }

.heading {
    display:flex;
    justify-content:space-between;
    align-items:center;
    gap:15px;
    flex-wrap:wrap;
}

.tag {
    display:inline-block;
    background:var(--aqua);
    color:var(--teal);
    padding:5px 11px;
    border-radius:30px;
    font-size:12px;
    font-weight:800;
}

.tag.High { background:#ffe1d8; color:#a43c26; }
.tag.Medium { background:#fff1c8; color:#856000; }
.tag.Low { background:#e2f6ec; color:#276c4d; }

.between {
    display:flex;
    justify-content:space-between;
    align-items:center;
    gap:10px;
    flex-wrap:wrap;
}

.report-card { color:var(--navy); transition:transform .15s; }
.report-card:hover { transform:translateY(-3px); text-decoration:none; }

label {
    display:block;
    font-size:14px;
    font-weight:700;
    margin:17px 0;
}

input,select,textarea {
    width:100%;
    display:block;
    margin-top:7px;
    padding:13px;
    border:1px solid #ccdade;
    border-radius:10px;
    font:inherit;
    background:white;
    color:var(--navy);
}

input:focus,select:focus,textarea:focus { outline:2px solid var(--teal); }

.form { max-width:760px; }
.auth { max-width:440px; margin:35px auto; }

.two { display:grid; grid-template-columns:1fr 1fr; gap:15px; }

.flash {
    background:var(--aqua);
    padding:13px 18px;
    border-radius:10px;
    margin-bottom:18px;
}

.flash.error { background:#ffe1d8; }

table { width:100%; border-collapse:collapse; }
th,td { padding:13px; border-bottom:1px solid #e1eaec; text-align:left; }
.tablewrap { overflow-x:auto; }

.timeline {
    border-left:3px solid var(--aqua);
    margin-left:10px;
    padding-left:22px;
}

.event { position:relative; margin-bottom:25px; }
.event:before {
    content:"";
    position:absolute;
    left:-30px;
    top:5px;
    width:13px;
    height:13px;
    background:var(--teal);
    border-radius:50%;
}

.event p { margin:3px 0; }

.detail-grid {
    display:grid;
    grid-template-columns:1.5fr 1fr;
    gap:20px;
}

.progress {
    height:13px;
    background:#e7eff0;
    border-radius:20px;
    overflow:hidden;
    margin:10px 0;
}

.progress div {
    height:100%;
    background:var(--teal);
    border-radius:20px;
}

footer {
    text-align:center;
    padding:25px;
    color:var(--muted);
    font-size:13px;
}

.mobile-nav { display:none; }

@media(max-width:850px) {
    header nav { display:none; }
    .hero,.detail-grid { grid-template-columns:1fr; }
    .art { height:260px; }
    .cards { grid-template-columns:repeat(2,1fr); }
    .stats { grid-template-columns:repeat(2,1fr); }
    .mobile-nav {
        display:flex;
        position:fixed;
        bottom:0;
        left:0;
        right:0;
        background:white;
        border-top:1px solid #dce6e8;
        justify-content:space-around;
        align-items:center;
        padding:12px 5px;
        z-index:10;
        font-size:12px;
    }
    footer { padding-bottom:90px; }
}

@media(max-width:550px) {
    main { padding:25px 15px 85px; }
    .cards,.two { grid-template-columns:1fr; }
    .stats strong { font-size:30px; }
    .hero { min-height:auto; }
    .art { height:210px; }
}
"""


BASE = """
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>REWAVE | Clean Water. Stronger Communities.</title>
<style>""" + CSS + """</style>
</head>
<body>

<header>
<a class="logo" href="/"><span>≈</span> REWAVE</a>
<nav>
{% if me %}
<a href="/dashboard">Dashboard</a>
<a href="/reports">Explore</a>
<a href="/map">Map</a>
<a href="/impact">Impact</a>
<a href="/my-reports">My Reports</a>
<a href="/profile">Profile</a>
{% if me.role == 'authority' %}
<a href="/authority">Authority</a>
{% else %}
<a class="btn" href="/report">+ Report</a>
{% endif %}
<a href="/logout">Logout</a>
{% else %}
<a href="/login">Login</a>
<a class="btn" href="/register">Get Started</a>
{% endif %}
</nav>
</header>

<main>
{% with messages = get_flashed_messages(with_categories=true) %}
{% for category,message in messages %}
<div class="flash {{category}}">{{message}}</div>
{% endfor %}
{% endwith %}

{{ content | safe }}
</main>

<footer>REWAVE · Clean Water. Stronger Communities. · SDG 6</footer>

{% if me %}
<div class="mobile-nav">
<a href="/dashboard">Home</a>
<a href="/reports">Explore</a>
{% if me.role == 'citizen' %}
<a class="btn" href="/report">+ Report</a>
{% else %}
<a class="btn" href="/authority">Manage</a>
{% endif %}
<a href="/impact">Impact</a>
<a href="/profile">Profile</a>
</div>
{% endif %}

</body>
</html>
"""


def page(template, **data):
    me = user()
    content = render_template_string(
        template, me=me, categories=CATEGORIES,
        statuses=STATUSES, score=score, **data
    )
    return render_template_string(BASE, content=Markup(content), me=me)


# ---------------- LANDING PAGE ----------------

@app.route("/")
def index():
    return page("""
    <section class="hero">
      <div>
        <div class="eyebrow">COMMUNITY-POWERED WATER ACTION · SDG 6</div>
        <h1>Every drop matters.<br><em>Every voice counts.</em></h1>
        <p>Report local water issues, rally your community and follow
        every complaint from first report to verified resolution.</p>
        <a class="btn" href="/register">Get Started →</a>
        <a class="btn secondary" href="/login">Login</a>
      </div>
      <div class="art">
        <div class="wave">≈</div>
        <div class="float">REWAVE<br><small>Report · Verify · Resolve</small></div>
      </div>
    </section>

    <section>
      <div class="eyebrow">HOW REWAVE WORKS</div>
      <h2>From awareness to action.</h2>
      <div class="cards">
        <div class="card"><div class="eyebrow">01</div><h3>Report</h3>
        <p>Share water problems with your community.</p></div>

        <div class="card"><div class="eyebrow">02</div><h3>Verify</h3>
        <p>Community members independently confirm issues.</p></div>

        <div class="card"><div class="eyebrow">03</div><h3>Resolve</h3>
        <p>Track authority updates and confirm the outcome.</p></div>
      </div>
    </section>

    <section class="banner">
      <h2>Clean water is a community effort.</h2>
      <p>Supporting Sustainable Development Goal 6.</p>
      <a class="btn secondary" href="/register">Join REWAVE →</a>
    </section>
    """)


# ---------------- AUTHENTICATION ----------------

@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        name = request.form.get("name", "").strip()
        email = request.form.get("email", "").strip().lower()
        password = request.form.get("password", "")

        if not name or "@" not in email or len(password) < 8:
            flash("Enter valid details and a password of at least 8 characters.", "error")
        else:
            try:
                with connect() as db:
                    cursor = db.execute(
                        "INSERT INTO users(name,email,password,role) VALUES(?,?,?,?)",
                        (name, email, generate_password_hash(password), "citizen")
                    )
                    uid = cursor.lastrowid

                session.clear()
                session["uid"] = uid
                return redirect("/dashboard")

            except sqlite3.IntegrityError:
                flash("This email is already registered.", "error")

    return page("""
    <div class="card auth">
      <div class="eyebrow">WELCOME TO REWAVE</div>
      <h2>Create your account</h2>
      <form method="POST">
        <label>Full Name<input name="name" required maxlength="100"></label>
        <label>Email<input type="email" name="email" required></label>
        <label>Password<input type="password" name="password" minlength="8" required></label>
        <button class="btn" style="width:100%">Create Account</button>
      </form>
      <p>Already registered? <a href="/login">Login</a></p>
    </div>
    """)


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form.get("email", "").strip().lower()
        password = request.form.get("password", "")

        account = query(
            "SELECT * FROM users WHERE email=?", (email,), True
        )

        if account and check_password_hash(account["password"], password):
            session.clear()
            session["uid"] = account["id"]
            return redirect(
                "/authority" if account["role"] == "authority" else "/dashboard"
            )

        flash("Incorrect email or password.", "error")

    return page("""
    <div class="card auth">
      <div class="eyebrow">WELCOME BACK</div>
      <h2>Login to REWAVE</h2>
      <form method="POST">
        <label>Email<input type="email" name="email" required></label>
        <label>Password<input type="password" name="password" required></label>
        <button class="btn" style="width:100%">Login</button>
      </form>
      <p>New here? <a href="/register">Create an account</a></p>
    </div>
    """)


@app.route("/logout")
def logout():
    session.clear()
    return redirect("/")


# ---------------- REPORT CARDS ----------------

REPORT_CARDS = """
<div class="cards">
{% for r in reports %}
<a href="/reports/{{r.id}}" class="card report-card">
  <div class="between">
    <span class="tag">{{r.category}}</span>
    <span class="tag {{score(r)}}">{{score(r)}} Priority</span>
  </div>
  <h3>{{r.title}}</h3>
  <p>📍 {{r.location}}</p>
  <div class="between">
    <small>{{r.code}}</small>
    <small>{{r.status}}</small>
  </div>
</a>
{% else %}
<div class="card">
<h3>No reports yet</h3>
<p>Start by reporting an issue in your community.</p>
</div>
{% endfor %}
</div>
"""


# ---------------- CITIZEN DASHBOARD ----------------

@app.route("/dashboard")
@login_required
def dashboard():
    if user()["role"] == "authority":
        return redirect("/authority")

    m = metrics()
    reports = query("SELECT * FROM reports ORDER BY id DESC LIMIT 6")

    return page("""
    <div class="heading">
      <div>
        <div class="eyebrow">YOUR COMMUNITY HUB</div>
        <h1>Welcome, {{me.name}}.</h1>
        <p>Small actions. Real change.</p>
      </div>
      <a class="btn" href="/report">+ Report an Issue</a>
    </div>

    <div class="stats">
      <div class="card"><small>Total Reports</small><strong>{{m.total}}</strong></div>
      <div class="card"><small>Pending</small><strong>{{m.pending}}</strong></div>
      <div class="card"><small>Resolved</small><strong>{{m.resolved}}</strong></div>
      <div class="card"><small>High Priority</small><strong>{{m.high}}</strong></div>
    </div>

    <div class="heading">
      <h2>Recent Community Reports</h2>
      <a href="/reports">View All →</a>
    </div>

    """ + REPORT_CARDS + """

    <div class="banner">
      <h2>See what your community is changing.</h2>
      <a class="btn secondary" href="/impact">Explore Impact →</a>
    </div>
    """, m=m, reports=reports)


# ---------------- REPORT AN ISSUE ----------------

@app.route("/report", methods=["GET", "POST"])
@login_required
def new_report():
    if user()["role"] != "citizen":
        abort(403)

    if request.method == "POST":
        title = request.form.get("title", "").strip()
        description = request.form.get("description", "").strip()
        location = request.form.get("location", "").strip()
        category = request.form.get("category", "")

        try:
            households = int(request.form.get("households", 1))
            severity = int(request.form.get("severity", 1))
            duration = int(request.form.get("duration", 1))
        except ValueError:
            households = severity = duration = 0

        if category not in CATEGORIES:
            category = classify(title + " " + description)

        if (
            len(title) < 4 or len(description) < 12 or not location
            or not 1 <= households <= 100000
            or severity not in (1, 2, 3)
            or not 1 <= duration <= 365
        ):
            flash("Please complete the form with valid information.", "error")
        else:
            code = "RW-" + secrets.token_hex(4).upper()

            with connect() as db:
                cursor = db.execute("""
                    INSERT INTO reports
                    (code,user_id,title,description,category,location,
                     households,severity,duration,status,created)
                    VALUES (?,?,?,?,?,?,?,?,?,?,?)
                """, (
                    code, user()["id"], title, description, category,
                    location, households, severity, duration,
                    "Submitted", now()
                ))

                rid = cursor.lastrowid

                db.execute("""
                    INSERT INTO history(report_id,status,note,created)
                    VALUES (?,?,?,?)
                """, (rid, "Submitted", "Complaint registered successfully.", now()))

            flash("Your complaint was submitted successfully!", "success")
            return redirect(f"/reports/{rid}")

    return page("""
    <div class="eyebrow">MAKE A DIFFERENCE</div>
    <h1>Report a Water Issue</h1>
    <p>Help your community identify problems that need attention.</p>

    <form class="card form" method="POST" id="reportForm">

      <div class="eyebrow">STEP 01 — SELECT ISSUE</div>

      <label>Issue Category
        <select name="category" id="category">
          <option value="">Suggest automatically from description</option>
          {% for c in categories %}
          <option>{{c}}</option>
          {% endfor %}
        </select>
      </label>

      <hr>

      <div class="eyebrow">STEP 02 — ADD DETAILS</div>

      <label>Issue Title
        <input name="title" id="title" required minlength="4"
        maxlength="140" placeholder="Burst pipe near public market">
      </label>

      <label>Description
        <textarea name="description" id="description" rows="5"
        required minlength="12" maxlength="3000"
        placeholder="Describe the problem..."></textarea>
      </label>

      <label>Approximate Location
        <input name="location" id="location" required maxlength="180"
        placeholder="Jayanagar, Bengaluru">
      </label>

      <div class="two">
        <label>Households Affected
          <input type="number" name="households" id="households"
          value="1" min="1" max="100000" required>
        </label>

        <label>Duration (Days)
          <input type="number" name="duration" id="duration"
          value="1" min="1" max="365" required>
        </label>
      </div>

      <label>Severity
        <select name="severity" id="severity">
          <option value="1">Low</option>
          <option value="2">Medium</option>
          <option value="3">High</option>
        </select>
      </label>

      <hr>

      <div class="eyebrow">STEP 03 — REVIEW AND SUBMIT</div>

      <button type="button" class="btn secondary"
      onclick="reviewReport()">Review My Report</button>

      <div id="review" class="card" style="margin-top:15px">
        Your report summary will appear here.
      </div>

      <button class="btn">Submit Report →</button>

    </form>

    <script>
    function reviewReport() {
        let form = document.getElementById("reportForm");
        if (!form.reportValidity()) return;

        let title = document.getElementById("title").value;
        let description = document.getElementById("description").value;
        let location = document.getElementById("location").value;
        let category = document.getElementById("category").value;

        document.getElementById("review").textContent =
          "Title: " + title + "\\n" +
          "Category: " + (category || "Automatically suggested") + "\\n" +
          "Location: " + location + "\\n" +
          "Description: " + description;

        document.getElementById("review").style.whiteSpace = "pre-line";
    }
    </script>
    """)


# ---------------- COMMUNITY REPORTS ----------------

@app.route("/reports")
@login_required
def reports():
    category = request.args.get("category", "")
    status = request.args.get("status", "")

    sql = "SELECT * FROM reports WHERE 1=1"
    args = []

    if category in CATEGORIES:
        sql += " AND category=?"
        args.append(category)

    if status in STATUSES:
        sql += " AND status=?"
        args.append(status)

    sql += " ORDER BY id DESC"
    rows = query(sql, args)

    return page("""
    <div class="heading">
      <div>
        <div class="eyebrow">COMMUNITY ACTION</div>
        <h1>Explore Reports</h1>
      </div>
      {% if me.role == 'citizen' %}
      <a class="btn" href="/report">+ New Report</a>
      {% endif %}
    </div>

    <form method="GET" class="card two">
      <label>Category
        <select name="category">
          <option value="">All Categories</option>
          {% for c in categories %}
          <option {% if selected_category==c %}selected{% endif %}>{{c}}</option>
          {% endfor %}
        </select>
      </label>

      <label>Status
        <select name="status">
          <option value="">All Statuses</option>
          {% for s in statuses %}
          <option {% if selected_status==s %}selected{% endif %}>{{s}}</option>
          {% endfor %}
        </select>
      </label>

      <button class="btn">Apply Filters</button>
    </form>

    """ + REPORT_CARDS,
    reports=rows, selected_category=category, selected_status=status)


@app.route("/my-reports")
@login_required
def my_reports():
    rows = query(
        "SELECT * FROM reports WHERE user_id=? ORDER BY id DESC",
        (user()["id"],)
    )

    return page("""
    <div class="eyebrow">YOUR CONTRIBUTIONS</div>
    <h1>My Reports</h1>
    <p>Track every issue you've submitted.</p>
    """ + REPORT_CARDS, reports=rows)


# ---------------- TRACKING ----------------

@app.route("/reports/<int:rid>")
@login_required
def report_detail(rid):
    r = query("SELECT * FROM reports WHERE id=?", (rid,), True)
    if not r:
        abort(404)

    history = query(
        "SELECT * FROM history WHERE report_id=? ORDER BY id",
        (rid,)
    )

    count = query(
        "SELECT COUNT(*) AS n FROM verifications WHERE report_id=?",
        (rid,), True
    )["n"]

    return page("""
    <a href="/reports">← Back to Reports</a>

    <div class="heading">
      <div>
        <div class="eyebrow">{{r.code}} · {{r.category}}</div>
        <h1>{{r.title}}</h1>
        <p>📍 {{r.location}}</p>
      </div>
      <span class="tag {{score(r)}}">{{score(r)}} Priority</span>
    </div>

    <div class="detail-grid">

      <div>
        <div class="card">
          <h2>Issue Details</h2>
          <p style="white-space:pre-wrap">{{r.description}}</p>
          <p><b>Households affected:</b> {{r.households}}</p>
          <p><b>Duration:</b> {{r.duration}} day(s)</p>
          <p><b>Submitted:</b> {{r.created}}</p>

          <div class="between">
            <span class="tag">{{r.status}}</span>
            <b>{{count}} Community Confirmations</b>
          </div>

          {% if me.role == 'citizen' and me.id != r.user_id %}
          <form method="POST" action="/reports/{{r.id}}/verify">
            <p><button class="btn">I Can Confirm This Issue</button></p>
          </form>
          {% endif %}

          {% if me.id == r.user_id and r.status == 'Marked Resolved' %}
          <form method="POST" action="/reports/{{r.id}}/confirm">
            <button class="btn" name="decision" value="confirm">
              Confirm Resolution
            </button>
            <button class="btn coral" name="decision" value="dispute">
              Dispute Resolution
            </button>
          </form>
          {% endif %}
        </div>

        {% if me.role == 'authority' %}
        <div class="card">
          <h2>Authority Controls</h2>
          <form method="POST" action="/authority/{{r.id}}/update">
            <label>Update Status
              <select name="status">
                <option>Under Review</option>
                <option>In Progress</option>
                <option>Marked Resolved</option>
              </select>
            </label>

            <label>Resolution Notes
              <textarea name="note" maxlength="1000" rows="4"></textarea>
            </label>

            <button class="btn">Save Update</button>
          </form>
        </div>
        {% endif %}
      </div>

      <div class="card">
        <h2>Resolution Timeline</h2>

        <div class="timeline">
          {% for h in history %}
          <div class="event">
            <b>{{h.status}}</b>
            <p>{{h.note}}</p>
            <small>{{h.created}}</small>
          </div>
          {% endfor %}
        </div>
      </div>

    </div>
    """, r=r, history=history, count=count)


# ---------------- COMMUNITY VERIFICATION ----------------

@app.route("/reports/<int:rid>/verify", methods=["POST"])
@login_required
def verify(rid):
    r = query("SELECT * FROM reports WHERE id=?", (rid,), True)

    if not r:
        abort(404)

    if user()["role"] != "citizen":
        abort(403)

    if r["user_id"] == user()["id"]:
        flash("You cannot verify your own report.", "error")
        return redirect(f"/reports/{rid}")

    try:
        with connect() as db:
            db.execute(
                "INSERT INTO verifications(user_id,report_id) VALUES(?,?)",
                (user()["id"], rid)
            )

            count = db.execute(
                "SELECT COUNT(*) FROM verifications WHERE report_id=?",
                (rid,)
            ).fetchone()[0]

            if count >= 2 and r["status"] == "Submitted":
                db.execute(
                    "UPDATE reports SET status=? WHERE id=?",
                    ("Community Verified", rid)
                )

                db.execute("""
                    INSERT INTO history(report_id,status,note,created)
                    VALUES(?,?,?,?)
                """, (
                    rid, "Community Verified",
                    "The community confirmed this issue.", now()
                ))

        flash("Thank you! Your verification has been recorded.", "success")

    except sqlite3.IntegrityError:
        flash("You have already verified this report.", "error")

    return redirect(f"/reports/{rid}")


# ---------------- AUTHORITY DASHBOARD ----------------

@app.route("/authority")
@authority_required
def authority():
    m = metrics()
    rows = query("SELECT * FROM reports ORDER BY id DESC")

    return page("""
    <div class="eyebrow">DEMO AUTHORITY PORTAL</div>
    <h1>Resolution Command Centre</h1>
    <p>Manage incoming complaints and track community concerns.</p>

    <div class="stats">
      <div class="card"><small>All Reports</small><strong>{{m.total}}</strong></div>
      <div class="card"><small>Pending</small><strong>{{m.pending}}</strong></div>
      <div class="card"><small>Resolved</small><strong>{{m.resolved}}</strong></div>
      <div class="card"><small>High Priority</small><strong>{{m.high}}</strong></div>
    </div>

    <div class="card">
      <h2>Incoming Complaints</h2>

      <div class="tablewrap">
      <table>
        <tr>
          <th>Complaint ID</th>
          <th>Issue</th>
          <th>Priority</th>
          <th>Status</th>
          <th>Action</th>
        </tr>

        {% for r in reports %}
        <tr>
          <td>{{r.code}}</td>
          <td>{{r.title}}</td>
          <td><span class="tag {{score(r)}}">{{score(r)}}</span></td>
          <td>{{r.status}}</td>
          <td><a href="/reports/{{r.id}}">Manage →</a></td>
        </tr>
        {% endfor %}
      </table>
      </div>
    </div>
    """, m=m, reports=rows)


@app.route("/authority/<int:rid>/update", methods=["POST"])
@authority_required
def update_report(rid):
    r = query("SELECT * FROM reports WHERE id=?", (rid,), True)
    if not r:
        abort(404)

    status = request.form.get("status", "")
    note = request.form.get("note", "").strip()[:1000]

    if status not in ["Under Review", "In Progress", "Marked Resolved"]:
        abort(400)

    with connect() as db:
        db.execute("""
            UPDATE reports SET status=?, resolved=? WHERE id=?
        """, (
            status,
            now() if status == "Marked Resolved" else None,
            rid
        ))

        db.execute("""
            INSERT INTO history(report_id,status,note,created)
            VALUES(?,?,?,?)
        """, (
            rid, status, note or "Status updated by demo authority.", now()
        ))

    flash("Complaint status updated successfully.", "success")
    return redirect(f"/reports/{rid}")


# ---------------- CITIZEN CONFIRMATION ----------------

@app.route("/reports/<int:rid>/confirm", methods=["POST"])
@login_required
def confirm_resolution(rid):
    r = query("SELECT * FROM reports WHERE id=?", (rid,), True)

    if not r:
        abort(404)

    if r["user_id"] != user()["id"] or r["status"] != "Marked Resolved":
        abort(403)

    decision = request.form.get("decision")

    if decision not in ["confirm", "dispute"]:
        abort(400)

    status = "Citizen Confirmed" if decision == "confirm" else "Disputed"

    with connect() as db:
        db.execute(
            "UPDATE reports SET status=? WHERE id=?",
            (status, rid)
        )

        db.execute("""
            INSERT INTO history(report_id,status,note,created)
            VALUES(?,?,?,?)
        """, (
            rid, status,
            "Reporter reviewed the resolution outcome.", now()
        ))

    flash("Your response has been recorded.", "success")
    return redirect(f"/reports/{rid}")


# ---------------- COMMUNITY MAP / LOCATION EXPLORER ----------------

@app.route("/map")
@login_required
def map_page():
    rows = query("SELECT * FROM reports ORDER BY id DESC")

    return page("""
    <div class="eyebrow">COMMUNITY ISSUE EXPLORER</div>
    <h1>Water Issue Map</h1>
    <p>Explore reported water problems by their approximate public location.</p>

    <div class="card" style="background:#D9F3F0;text-align:center;padding:55px 20px">
      <div style="font-size:90px;color:#087E8B">≈</div>
      <h2>Community Water Explorer</h2>
      <p>Offline-friendly location directory — no external map API required.</p>
    </div>

    <h2>Reported Locations</h2>

    """ + REPORT_CARDS, reports=rows)


# ---------------- IMPACT DASHBOARD ----------------

@app.route("/impact")
@login_required
def impact():
    m = metrics()

    counts = query("""
        SELECT category,COUNT(*) AS n
        FROM reports GROUP BY category ORDER BY n DESC
    """)

    total = m["total"]

    return page("""
    <div class="eyebrow">MEASURING PROGRESS</div>
    <h1>Community Impact</h1>
    <p>Live statistics calculated from the REWAVE database.</p>

    <div class="stats">
      <div class="card"><small>Total Reports</small><strong>{{m.total}}</strong></div>
      <div class="card"><small>Resolved</small><strong>{{m.resolved}}</strong></div>
      <div class="card"><small>Pending</small><strong>{{m.pending}}</strong></div>
      <div class="card"><small>Resolution Rate</small><strong>{{m.percentage}}%</strong></div>
    </div>

    <div class="card">
      <h2>Overall Resolution Progress</h2>
      <div class="progress">
        <div style="width:{{m.percentage}}%"></div>
      </div>
      <p>{{m.percentage}}% of reported issues are marked resolved.</p>
    </div>

    <div class="card">
      <h2>Most Common Water Problems</h2>

      {% for c in counts %}
      <div class="between">
        <b>{{c.category}}</b>
        <span>{{c.n}} reports</span>
      </div>

      <div class="progress">
        <div style="width:{{c.n / total * 100 if total else 0}}%"></div>
      </div>
      {% else %}
      <p>No data available yet.</p>
      {% endfor %}
    </div>

    <div class="banner">
      <h2>Community participation creates accountability.</h2>
      <p>REWAVE tracks reported issues and resolution progress.</p>
    </div>
    """, m=m, counts=counts, total=total)


# ---------------- PROFILE ----------------

@app.route("/profile", methods=["GET", "POST"])
@login_required
def profile():
    if request.method == "POST":
        name = request.form.get("name", "").strip()

        if name:
            with connect() as db:
                db.execute(
                    "UPDATE users SET name=? WHERE id=?",
                    (name[:100], user()["id"])
                )

            flash("Profile updated successfully.", "success")

    return page("""
    <div class="eyebrow">YOUR ACCOUNT</div>
    <h1>Profile & Settings</h1>

    <div class="card form">
      <form method="POST">
        <label>Display Name
          <input name="name" value="{{me.name}}" maxlength="100" required>
        </label>

        <label>Email
          <input value="{{me.email}}" disabled>
        </label>

        <label>Account Type
          <input value="{{me.role|capitalize}}" disabled>
        </label>

        <button class="btn">Save Profile</button>
      </form>

      <p><a href="/my-reports">View My Reports →</a></p>
      <p><a href="/logout">Logout →</a></p>
    </div>
    """)


# ---------------- DEMO AUTHORITY ACCOUNT ----------------

def create_demo_authority():
    password = os.environ.get("REWAVE_DEMO_PASSWORD")

    if not password:
        return

    existing = query(
        "SELECT id FROM users WHERE email=?",
        ("authority@rewave.local",), True
    )

    if not existing:
        with connect() as db:
            db.execute("""
                INSERT INTO users(name,email,password,role)
                VALUES(?,?,?,?)
            """, (
                "Demo Authority",
                "authority@rewave.local",
                generate_password_hash(password),
                "authority"
            ))


# ---------------- START APPLICATION ----------------

init_db()
create_demo_authority()

if __name__ == "__main__":
    app.run(debug=True, host="127.0.0.1", port=5000)
# UN-BURY
..
