from flask import Flask, render_template_string, request, redirect, url_for, session, flash, jsonify
import sqlite3, hashlib, os, random, string, time
from datetime import datetime

app = Flask(__name__)
app.secret_key = 'vpduy_aov_2024_secret'
DB = 'aov.db'

ADMIN_USER = 'vophucduy'
ADMIN_PASS = '0345543851'

def get_db():
    conn = sqlite3.connect(DB)
    conn.row_factory = sqlite3.Row
    return conn

def hash_pw(pw): return hashlib.sha256(pw.encode()).hexdigest()
def rand_code(n=8): return ''.join(random.choices(string.ascii_uppercase + string.digits, k=n))

# Mã chuyển khoản thay đổi mỗi 5 phút
def get_transfer_code():
    slot = int(time.time()) // 300
    random.seed(slot)
    code = ''.join(random.choices(string.ascii_uppercase + string.digits, k=8)) + str(random.randint(10,99))
    random.seed()
    return code

def init_db():
    db = get_db()
    db.executescript('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL,
            email TEXT,
            xu INTEGER DEFAULT 100,
            invite_code TEXT UNIQUE,
            invited_by TEXT,
            banned INTEGER DEFAULT 0,
            created_at TEXT
        );
        CREATE TABLE IF NOT EXISTS accounts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ten_acc TEXT, rank TEXT, tuong INTEGER,
            xu_gia INTEGER, mo_ta TEXT,
            claimed_by TEXT, created_at TEXT
        );
        CREATE TABLE IF NOT EXISTS nhiem_vu (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER, task TEXT, done INTEGER DEFAULT 0, created_at TEXT
        );
        CREATE TABLE IF NOT EXISTS trao_doi (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            nguoi_ban TEXT, nguoi_mua TEXT, acc_id INTEGER,
            gia INTEGER, status TEXT DEFAULT 'cho', created_at TEXT
        );
        CREATE TABLE IF NOT EXISTS don_nap (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT, so_tien INTEGER, xu_nhan INTEGER,
            ma_don TEXT, ma_ck TEXT, status TEXT DEFAULT 'cho',
            created_at TEXT
        );
        CREATE TABLE IF NOT EXISTS link_rut_gon (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            tieu_de TEXT, url TEXT, mo_ta TEXT, created_at TEXT
        );
    ''')
    # Migration: add missing columns to existing databases
    cols = [r[1] for r in db.execute("PRAGMA table_info(users)").fetchall()]
    if 'banned' not in cols:
        db.execute("ALTER TABLE users ADD COLUMN banned INTEGER DEFAULT 0")
    if 'email' not in cols:
        db.execute("ALTER TABLE users ADD COLUMN email TEXT")
    db.commit()

    if db.execute('SELECT COUNT(*) FROM accounts').fetchone()[0] == 0:
        accs = [
            ('Sát Thủ Bóng Đêm','Chinh Phục',120,2000,'Acc xịn full tướng'),
            ('Hoàng Đế AOV','Kim Cương',80,1500,'Rank cao nhiều skin'),
            ('Ninja Rừng Già','Đại Cao Thủ',95,1800,'Full tướng giới hạn'),
            ('Pháp Sư Lửa','Bạch Kim',45,800,'Acc sạch mới tạo'),
            ('Rồng Vàng','Chinh Phục',200,3500,'Acc khủng full server'),
            ('Thiên Long','Kim Cương',60,1200,'Nhiều skin hiếm'),
        ]
        for a in accs:
            db.execute('INSERT INTO accounts (ten_acc,rank,tuong,xu_gia,mo_ta,created_at) VALUES (?,?,?,?,?,?)',
                       (*a, datetime.now().strftime('%Y-%m-%d %H:%M')))
    db.commit()
    db.close()

def sync_xu():
    if 'user' in session:
        db = get_db()
        u = db.execute('SELECT xu FROM users WHERE username=?', (session['user'],)).fetchone()
        if u: session['xu'] = u['xu']
        db.close()

def login_required(f):
    from functools import wraps
    @wraps(f)
    def dec(*a, **kw):
        if 'user' not in session: return redirect('/dang-nhap')
        db = get_db()
        u = db.execute('SELECT banned FROM users WHERE username=?', (session['user'],)).fetchone()
        db.close()
        if u and u['banned']: session.clear(); return redirect('/dang-nhap?banned=1')
        return f(*a, **kw)
    return dec

def admin_required(f):
    from functools import wraps
    @wraps(f)
    def dec(*a, **kw):
        if session.get('user') != ADMIN_USER: return redirect('/')
        return f(*a, **kw)
    return dec

# ══════════════════════════════════════════
#  BASE TEMPLATE
# ══════════════════════════════════════════
BASE = '''<!DOCTYPE html>
<html lang="vi">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>vpduy.site — AOV FREE 🎮</title>
<link href="https://fonts.googleapis.com/css2?family=Be+Vietnam+Pro:wght@400;600;700;900&display=swap" rel="stylesheet">
<style>
:root{
  --bg:#07071a;--card:#0d0d26;--card2:#12122e;--card3:#17173a;
  --gold:#f5c542;--gold2:#ff9f00;--gold3:#fde68a;
  --purple:#8b5cf6;--purple2:#6d28d9;--purple3:#a78bfa;
  --blue:#3b82f6;--green:#22c55e;--red:#ef4444;--orange:#f97316;
  --text:#e2e8f0;--muted:#94a3b8;--muted2:#64748b;
  --border:rgba(139,92,246,.18);--border2:rgba(139,92,246,.35);
  --glow-purple:rgba(139,92,246,.25);--glow-gold:rgba(245,197,66,.25);
  --radius:14px;--radius-sm:9px;--radius-lg:20px;
}
*{margin:0;padding:0;box-sizing:border-box}
html{scroll-behavior:smooth}
body{background:var(--bg);color:var(--text);font-family:'Be Vietnam Pro',sans-serif;min-height:100vh;
     background-image:radial-gradient(ellipse 80% 40% at 50% -5%,rgba(109,40,217,.15) 0%,transparent 60%)}
a{text-decoration:none;color:inherit}

/* ── NAV ── */
nav{background:rgba(7,7,26,.92);backdrop-filter:blur(16px);-webkit-backdrop-filter:blur(16px);
    border-bottom:1px solid var(--border);padding:0 1rem;position:sticky;top:0;z-index:200;
    height:58px;display:flex;align-items:center;gap:.6rem;
    box-shadow:0 1px 20px rgba(0,0,0,.4)}
.nav-logo{font-size:1rem;font-weight:900;white-space:nowrap;flex-shrink:0;
          background:linear-gradient(135deg,var(--gold),var(--gold2));-webkit-background-clip:text;
          -webkit-text-fill-color:transparent;background-clip:text}
.nav-xu{background:linear-gradient(135deg,rgba(245,197,66,.15),rgba(255,159,0,.08));
        border:1px solid rgba(245,197,66,.3);border-radius:20px;
        padding:.28rem .75rem;font-size:.78rem;font-weight:800;color:var(--gold);
        white-space:nowrap;flex-shrink:0;letter-spacing:.3px}
.hamburger{background:rgba(139,92,246,.1);border:1px solid var(--border2);border-radius:9px;
           padding:.4rem .55rem;cursor:pointer;color:var(--purple3);font-size:1.05rem;
           flex-shrink:0;margin-left:auto;transition:.2s}
.hamburger:hover{background:rgba(139,92,246,.2);border-color:var(--purple)}

/* ── DRAWER ── */
.overlay{position:fixed;inset:0;background:rgba(0,0,0,.65);z-index:300;display:none;backdrop-filter:blur(3px)}
.overlay.open{display:block}
.drawer{position:fixed;right:-290px;top:0;bottom:0;width:270px;
        background:linear-gradient(180deg,#0f0f2a 0%,#0a0a1f 100%);
        border-left:1px solid var(--border2);z-index:400;transition:right .28s cubic-bezier(.4,0,.2,1);
        overflow-y:auto;box-shadow:-10px 0 40px rgba(0,0,0,.5)}
.drawer.open{right:0}
.drawer-header{padding:1rem 1.25rem;border-bottom:1px solid var(--border);
               display:flex;align-items:center;justify-content:space-between;
               background:rgba(139,92,246,.05)}
.drawer-logo{font-size:1rem;font-weight:900;
             background:linear-gradient(135deg,var(--gold),var(--gold2));
             -webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text}
.drawer-close{background:rgba(239,68,68,.1);border:1px solid rgba(239,68,68,.2);
              border-radius:7px;color:var(--red);font-size:1rem;cursor:pointer;
              padding:.2rem .5rem;transition:.2s}
.drawer-close:hover{background:rgba(239,68,68,.2)}
.drawer-user{padding:.9rem 1.25rem;border-bottom:1px solid var(--border);
             background:rgba(139,92,246,.04)}
.drawer-avatar{width:36px;height:36px;border-radius:50%;
               background:linear-gradient(135deg,var(--purple),var(--purple2));
               display:flex;align-items:center;justify-content:center;
               font-size:1rem;margin-bottom:.4rem}
.drawer-username{font-weight:800;font-size:.9rem;color:var(--text)}
.drawer-xu{font-size:.75rem;color:var(--gold);font-weight:700;margin-top:.1rem}
.drawer-links{padding:.6rem 0}
.drawer-link{display:flex;align-items:center;gap:.8rem;padding:.6rem 1.25rem;
             font-size:.85rem;font-weight:600;color:var(--muted);transition:.15s;cursor:pointer;
             border-left:3px solid transparent;margin:.05rem 0}
.drawer-link:hover{background:rgba(139,92,246,.1);color:var(--purple3);border-left-color:var(--purple)}
.drawer-link.active{background:rgba(139,92,246,.12);color:var(--purple3);border-left-color:var(--purple)}
.drawer-link .di{font-size:1rem;width:22px;text-align:center;flex-shrink:0}
.drawer-sep{height:1px;background:var(--border);margin:.4rem 1.25rem}
.drawer-link.danger:hover{background:rgba(239,68,68,.1);color:var(--red);border-left-color:var(--red)}
.drawer-link.admin-link{color:var(--gold)}
.drawer-link.admin-link:hover,.drawer-link.admin-link.active{background:rgba(245,197,66,.08);color:var(--gold);border-left-color:var(--gold)}

/* ── HERO ── */
.hero{background:linear-gradient(160deg,#0a0a22 0%,#130d2e 45%,#0a1428 100%);
      padding:2.75rem 1.25rem;text-align:center;position:relative;overflow:hidden}
.hero::before{content:'';position:absolute;inset:0;
  background:radial-gradient(ellipse 75% 55% at 50% -5%,rgba(139,92,246,.22) 0%,transparent 65%)}
.hero::after{content:'';position:absolute;bottom:0;left:0;right:0;height:1px;
  background:linear-gradient(90deg,transparent,var(--purple),transparent)}
.hero h1{font-size:clamp(1.7rem,5vw,2.9rem);font-weight:900;line-height:1.1;margin-bottom:.65rem;
         position:relative;letter-spacing:-0.5px}
.hero h1 .hl{background:linear-gradient(135deg,var(--gold),var(--gold2));
             -webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;
             text-shadow:none;filter:drop-shadow(0 0 20px rgba(245,197,66,.3))}
.hero p{color:var(--muted);font-size:.88rem;max-width:480px;margin:0 auto 1.4rem;position:relative;line-height:1.6}
.stats-row{display:flex;gap:.55rem;justify-content:center;flex-wrap:wrap;position:relative}
.stat-pill{background:rgba(255,255,255,.04);border:1px solid var(--border);border-radius:18px;
           padding:.3rem .85rem;font-size:.74rem;font-weight:700;backdrop-filter:blur(4px)}
.stat-pill b{color:var(--gold)}

/* ── SECTION ── */
.section{padding:1.25rem;max-width:880px;margin:0 auto}
.sec-title{font-size:1rem;font-weight:800;margin-bottom:.9rem;display:flex;align-items:center;gap:.45rem;
           letter-spacing:.2px}

/* ── CARDS ── */
.grid2{display:grid;gap:.9rem;grid-template-columns:repeat(auto-fill,minmax(240px,1fr))}
.card{background:linear-gradient(145deg,var(--card),var(--card2));border:1px solid var(--border);
      border-radius:var(--radius);padding:1.15rem;transition:.22s;position:relative;overflow:hidden}
.card::before{content:'';position:absolute;top:0;left:0;right:0;height:2px;
  background:linear-gradient(90deg,var(--purple),var(--purple3),var(--gold))}
.card:hover{border-color:var(--border2);transform:translateY(-2px);
            box-shadow:0 8px 30px rgba(0,0,0,.3),0 0 0 1px rgba(139,92,246,.1)}
.card-name{font-size:.95rem;font-weight:800;margin-bottom:.45rem;color:var(--text)}
.metas{display:flex;gap:.4rem;flex-wrap:wrap;margin-bottom:.7rem}
.badge{padding:.2rem .58rem;border-radius:6px;font-size:.68rem;font-weight:700;letter-spacing:.2px}
.b-rank{background:rgba(245,197,66,.1);color:var(--gold);border:1px solid rgba(245,197,66,.25)}
.b-hero{background:rgba(59,130,246,.1);color:var(--blue);border:1px solid rgba(59,130,246,.25)}
.b-free{background:rgba(34,197,94,.1);color:var(--green);border:1px solid rgba(34,197,94,.25)}
.b-hot{background:rgba(249,115,22,.1);color:var(--orange);border:1px solid rgba(249,115,22,.25)}
.card-bot{display:flex;align-items:center;justify-content:space-between;margin-top:.65rem;padding-top:.65rem;
          border-top:1px solid rgba(255,255,255,.04)}
.price{font-size:.88rem;font-weight:800;color:var(--gold)}

/* ── BUTTONS ── */
.btn{padding:.45rem 1rem;border-radius:var(--radius-sm);font-size:.78rem;font-weight:700;cursor:pointer;
     border:none;transition:.18s;display:inline-block;text-align:center;font-family:inherit}
.btn-p{background:linear-gradient(135deg,var(--purple),var(--purple2));color:#fff;
       box-shadow:0 2px 12px rgba(139,92,246,.3)}
.btn-p:hover{transform:translateY(-1px);box-shadow:0 4px 18px rgba(139,92,246,.4)}
.btn-g{background:linear-gradient(135deg,var(--gold),var(--gold2));color:#000;font-weight:800;
       box-shadow:0 2px 12px rgba(245,197,66,.25)}
.btn-g:hover{transform:translateY(-1px);box-shadow:0 4px 18px rgba(245,197,66,.35)}
.btn-r{background:rgba(239,68,68,.12);border:1px solid rgba(239,68,68,.3);color:var(--red)}
.btn-r:hover{background:rgba(239,68,68,.2)}
.btn-sm{padding:.3rem .65rem;font-size:.72rem}
.btn-o{background:transparent;border:1px solid var(--border2);color:var(--muted)}
.btn-o:hover{border-color:var(--purple);color:var(--purple)}
.btn-full{width:100%;padding:.72rem;font-size:.88rem}
.btn-green{background:rgba(34,197,94,.12);border:1px solid rgba(34,197,94,.3);color:var(--green)}
.btn-green:hover{background:rgba(34,197,94,.22)}

/* ── FORM ── */
.fcard{background:linear-gradient(145deg,var(--card),var(--card2));border:1px solid var(--border);
       border-radius:var(--radius-lg);padding:1.85rem;max-width:420px;margin:2rem auto;
       box-shadow:0 20px 50px rgba(0,0,0,.4)}
.ftitle{font-size:1.2rem;font-weight:900;text-align:center;margin-bottom:1.25rem}
.ftitle .fi{font-size:1.7rem;display:block;margin-bottom:.4rem}
.fg{margin-bottom:.95rem}
.fg label{display:block;font-size:.74rem;font-weight:700;color:var(--muted);margin-bottom:.38rem;letter-spacing:.3px;text-transform:uppercase}
.fg input,.fg select,.fg textarea{width:100%;background:rgba(255,255,255,.04);border:1px solid var(--border);
  border-radius:var(--radius-sm);padding:.62rem .95rem;color:var(--text);font-size:.85rem;font-family:inherit;transition:.2s}
.fg input:focus,.fg select:focus,.fg textarea:focus{outline:none;border-color:var(--purple);
  background:rgba(139,92,246,.06);box-shadow:0 0 0 3px rgba(139,92,246,.1)}
.fg textarea{resize:vertical;min-height:70px}
.fhint{font-size:.72rem;color:var(--muted2);margin-top:.25rem}
.flink{text-align:center;margin-top:.9rem;font-size:.78rem;color:var(--muted)}
.flink a{color:var(--purple3);font-weight:700}

/* ── ALERTS ── */
.al{padding:.65rem 1rem;border-radius:var(--radius-sm);font-size:.8rem;font-weight:600;margin-bottom:.85rem;
    display:flex;align-items:center;gap:.5rem}
.al-ok{background:rgba(34,197,94,.1);border:1px solid rgba(34,197,94,.25);color:var(--green)}
.al-err{background:rgba(239,68,68,.1);border:1px solid rgba(239,68,68,.25);color:var(--red)}
.al-info{background:rgba(59,130,246,.1);border:1px solid rgba(59,130,246,.25);color:var(--blue)}

/* ── TASK ── */
.task-row{background:linear-gradient(135deg,var(--card),var(--card2));border:1px solid var(--border);
          border-radius:var(--radius);padding:.9rem 1.1rem;display:flex;align-items:center;
          gap:.85rem;margin-bottom:.6rem;transition:.2s}
.task-row:hover{border-color:var(--border2)}
.task-icon{font-size:1.4rem;flex-shrink:0}
.task-info{flex:1}
.task-t{font-weight:700;font-size:.85rem;margin-bottom:.18rem}
.task-r{font-size:.74rem;color:var(--gold);font-weight:700}

/* ── LEADERBOARD ── */
.lb-row{background:linear-gradient(135deg,var(--card),var(--card2));border:1px solid var(--border);
        border-radius:var(--radius-sm);padding:.72rem 1rem;display:flex;align-items:center;
        gap:.75rem;margin-bottom:.45rem;transition:.2s}
.lb-row:hover{border-color:var(--border2)}
.lb-rank{font-size:1rem;font-weight:900;min-width:30px;text-align:center}
.t1{color:#ffd700}.t2{color:#c0c0c0}.t3{color:#cd7f32}
.lb-name{flex:1;font-weight:700;font-size:.85rem}
.lb-xu{color:var(--gold);font-weight:800;font-size:.8rem}

/* ── INVITE ── */
.inv-box{background:rgba(139,92,246,.06);border:2px dashed rgba(139,92,246,.3);border-radius:var(--radius);
         padding:1rem 1.1rem;display:flex;align-items:center;gap:.75rem;flex-wrap:wrap}
.inv-code{font-size:1.15rem;font-weight:900;color:var(--gold);font-family:monospace;flex:1;letter-spacing:2px}

/* ── XU BADGE ── */
.xu-big{background:rgba(245,197,66,.07);border:1px solid rgba(245,197,66,.18);
        border-radius:var(--radius);padding:1rem;text-align:center;margin-bottom:1rem}
.xu-num{font-size:2.2rem;font-weight:900;background:linear-gradient(135deg,var(--gold),var(--gold2));
        -webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text}
.xu-lbl{font-size:.72rem;color:var(--muted);margin-top:.15rem}

/* ── NAP TIEN ── */
.nap-info{background:linear-gradient(145deg,var(--card2),var(--card3));border:1px solid var(--border);
          border-radius:var(--radius);padding:1.25rem;margin:1rem 0}
.nap-row{display:flex;justify-content:space-between;align-items:center;padding:.5rem 0;
         border-bottom:1px solid rgba(255,255,255,.04)}
.nap-row:last-child{border:none}
.nap-lbl{font-size:.78rem;color:var(--muted)}
.nap-val{font-size:.85rem;font-weight:700}
.ma-ck{font-size:1.5rem;font-weight:900;color:var(--gold);font-family:monospace;
       background:rgba(245,197,66,.07);border:1px solid rgba(245,197,66,.22);
       border-radius:10px;padding:.55rem 1.1rem;display:inline-block;letter-spacing:3px}
.qr-img{width:185px;height:185px;border-radius:var(--radius);border:3px solid var(--purple);display:block;margin:0 auto}
.luu-y{background:rgba(239,68,68,.06);border:1px solid rgba(239,68,68,.18);
        border-radius:var(--radius-sm);padding:.9rem 1rem;font-size:.78rem;line-height:1.75;color:var(--muted)}
.luu-y b{color:var(--red)}
.countdown{font-size:.72rem;color:var(--muted);margin-top:.3rem}
.countdown span{color:var(--orange);font-weight:700}

/* ── ADMIN ── */
.admin-table{width:100%;border-collapse:collapse;font-size:.76rem}
.admin-table th{padding:.55rem .75rem;background:var(--card3);color:var(--muted);font-weight:700;text-align:left;
                border-bottom:2px solid var(--border2)}
.admin-table td{padding:.52rem .75rem;border-top:1px solid var(--border)}
.admin-table tr:hover td{background:rgba(139,92,246,.04)}

/* ── RANDOM ACC ── */
.random-box{background:linear-gradient(145deg,var(--card),var(--card2));border:2px solid var(--border2);
            border-radius:var(--radius-lg);padding:2rem;text-align:center;margin:1rem 0;
            box-shadow:0 0 40px rgba(139,92,246,.1)}
.random-box .icon-big{font-size:4rem;margin-bottom:.5rem;display:block}
.random-result{background:linear-gradient(145deg,var(--card),var(--card2));border:1px solid var(--border);
               border-radius:var(--radius);padding:1.3rem;margin:1rem 0;text-align:left}

/* ── MODAL ── */
.modal-bg{position:fixed;inset:0;background:rgba(0,0,0,.8);z-index:500;
          display:flex;align-items:center;justify-content:center;padding:1rem;backdrop-filter:blur(4px)}
.modal{background:linear-gradient(145deg,var(--card),var(--card2));border:1px solid var(--border2);
       border-radius:var(--radius-lg);padding:1.6rem;max-width:390px;width:100%;
       max-height:90vh;overflow-y:auto;box-shadow:0 25px 60px rgba(0,0,0,.6)}
.modal-title{font-size:1rem;font-weight:800;margin-bottom:1rem}

footer{text-align:center;padding:1.75rem 1rem;color:var(--muted2);font-size:.72rem;
       border-top:1px solid var(--border);margin-top:2rem;
       background:linear-gradient(0deg,rgba(0,0,0,.2),transparent)}

/* ── AUTH PAGES ── */
.auth-page{min-height:100vh;display:flex;align-items:center;justify-content:center;
           padding:1.5rem;position:relative;overflow:hidden;
           background:radial-gradient(ellipse 130% 90% at 50% 0%,#160828 0%,var(--bg) 55%)}
.auth-glow{position:absolute;width:600px;height:600px;border-radius:50%;
           background:radial-gradient(circle,rgba(109,40,217,.14) 0%,transparent 70%);
           top:-150px;left:50%;transform:translateX(-50%);pointer-events:none}
.auth-particles{position:absolute;inset:0;pointer-events:none;overflow:hidden}
.auth-wrap{width:100%;max-width:390px;position:relative;z-index:1}
.auth-brand{text-align:center;margin-bottom:1.85rem}
.auth-icon{font-size:3rem;margin-bottom:.5rem;display:block;
           filter:drop-shadow(0 0 25px rgba(245,197,66,.5))}
.auth-logo{font-size:1.75rem;font-weight:900;letter-spacing:-0.5px;
           background:linear-gradient(135deg,#fff 30%,var(--gold));
           -webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text}
.auth-logo span{background:linear-gradient(135deg,var(--gold),var(--gold2));
                -webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text}
.auth-sub{font-size:.75rem;color:var(--muted);margin-top:.3rem}
.auth-card{background:rgba(13,13,38,.9);backdrop-filter:blur(24px);-webkit-backdrop-filter:blur(24px);
           border:1px solid rgba(139,92,246,.28);border-radius:22px;
           padding:2rem 1.85rem;box-shadow:0 30px 70px rgba(0,0,0,.6),
           inset 0 1px 0 rgba(255,255,255,.07),0 0 0 1px rgba(139,92,246,.1)}
.auth-title{font-size:1.15rem;font-weight:900;text-align:center;
            margin-bottom:1.5rem;color:var(--text)}
.auth-field{position:relative;margin-bottom:.9rem}
.auth-field-icon{position:absolute;left:.9rem;top:50%;transform:translateY(-50%);
                 font-size:.85rem;pointer-events:none;z-index:1;opacity:.8}
.auth-field input{width:100%;background:rgba(255,255,255,.04);
                  border:1px solid rgba(139,92,246,.22);border-radius:11px;
                  padding:.72rem .95rem .72rem 2.65rem;color:var(--text);
                  font-size:.85rem;font-family:inherit;transition:.22s}
.auth-field input:focus{outline:none;border-color:var(--purple);
                         background:rgba(139,92,246,.07);
                         box-shadow:0 0 0 4px rgba(139,92,246,.1)}
.auth-field input::placeholder{color:rgba(148,163,184,.45)}
.auth-bonus{background:linear-gradient(135deg,rgba(245,197,66,.08),rgba(255,159,0,.05));
            border:1px solid rgba(245,197,66,.22);
            border-radius:9px;padding:.58rem .9rem;font-size:.75rem;
            color:var(--gold3);margin-bottom:1.1rem;text-align:center}
.auth-btn{width:100%;padding:.8rem;border:none;border-radius:11px;
          font-size:.9rem;font-weight:800;cursor:pointer;transition:.22s;
          font-family:inherit;letter-spacing:.4px}
.auth-btn-purple{background:linear-gradient(135deg,#7c3aed,#4338ca);color:#fff;
                 box-shadow:0 4px 20px rgba(124,58,237,.4)}
.auth-btn-purple:hover{transform:translateY(-2px);box-shadow:0 8px 28px rgba(124,58,237,.5)}
.auth-btn-purple:active{transform:translateY(0)}
.auth-btn-gold{background:linear-gradient(135deg,var(--gold),var(--gold2));color:#000;
               box-shadow:0 4px 20px rgba(245,197,66,.35)}
.auth-btn-gold:hover{transform:translateY(-2px);box-shadow:0 8px 28px rgba(245,197,66,.45)}
.auth-btn-gold:active{transform:translateY(0)}
.auth-switch{text-align:center;margin-top:1.15rem;font-size:.78rem;color:var(--muted)}
.auth-switch a{color:var(--purple3);font-weight:700}
.auth-switch a:hover{color:var(--gold)}
.auth-divider{display:flex;align-items:center;gap:.75rem;margin:.75rem 0;color:var(--muted2);font-size:.72rem}
.auth-divider::before,.auth-divider::after{content:'';flex:1;height:1px;background:var(--border)}
</style>
</head>
<body>
{% if session.get('user') %}
<nav>
  <div class="nav-logo">🎮 vpduy.site</div>
  <div class="nav-xu">🪙 {{ session.get('xu',0) }}</div>
  <button class="hamburger" onclick="toggleDrawer()">☰</button>
</nav>

<div class="overlay" id="overlay" onclick="closeDrawer()"></div>
<div class="drawer" id="drawer">
  <div class="drawer-header">
    <div class="drawer-logo">🎮 vpduy.site</div>
    <button class="drawer-close" onclick="closeDrawer()">✕</button>
  </div>
  <div class="drawer-user">
    <div class="drawer-avatar">👤</div>
    <div class="drawer-username">{{ session.get('user') }}</div>
    <div class="drawer-xu">🪙 {{ session.get('xu',0) }} xu</div>
  </div>
  <div class="drawer-links">
    <a href="/" class="drawer-link {% if page=='home' %}active{% endif %}">
      <span class="di">🏠</span> Trang Chủ</a>
    <a href="/lay-acc" class="drawer-link {% if page=='acc' %}active{% endif %}">
      <span class="di">🎁</span> Lấy Acc</a>
    <a href="/random-acc" class="drawer-link {% if page=='random' %}active{% endif %}">
      <span class="di">🎰</span> Random Acc</a>
    <a href="/nhiem-vu" class="drawer-link {% if page=='task' %}active{% endif %}">
      <span class="di">📋</span> Nhiệm Vụ</a>
    <a href="/trao-doi" class="drawer-link {% if page=='trade' %}active{% endif %}">
      <span class="di">🔄</span> Trao Đổi</a>
    <a href="/moi-ban" class="drawer-link {% if page=='invite' %}active{% endif %}">
      <span class="di">👥</span> Mời Bạn</a>
    <a href="/dua-top" class="drawer-link {% if page=='top' %}active{% endif %}">
      <span class="di">🏆</span> Đua Top</a>
    <div class="drawer-sep"></div>
    <a href="/nap-tien" class="drawer-link {% if page=='nap' %}active{% endif %}">
      <span class="di">💳</span> Nạp Tiền</a>
    <a href="/link-hay" class="drawer-link {% if page=='link' %}active{% endif %}">
      <span class="di">🔗</span> Link Hay</a>
    {% if session.get('user') == 'vophucduy' %}
    <div class="drawer-sep"></div>
    <a href="/admin" class="drawer-link admin-link {% if page=='admin' %}active{% endif %}">
      <span class="di">⚙️</span> Admin Panel</a>
    {% endif %}
    <div class="drawer-sep"></div>
    <a href="/logout" class="drawer-link danger">
      <span class="di">🚪</span> Đăng Xuất</a>
  </div>
</div>
{% endif %}

{% block content %}{% endblock %}

<footer>
  © 2025 <b style="color:var(--purple3)">vpduy.site</b> — AOV FREE 🎮
  <br><span style="color:var(--muted2);font-size:.65rem">Made with 💜 by VÕ PHÚC DUY · Không bán không lừa đảo ❤️</span>
</footer>

<script>
function toggleDrawer(){
  document.getElementById('drawer').classList.toggle('open');
  document.getElementById('overlay').classList.toggle('open');
}
function closeDrawer(){
  document.getElementById('drawer').classList.remove('open');
  document.getElementById('overlay').classList.remove('open');
}
// Close drawer on ESC
document.addEventListener('keydown',e=>{if(e.key==='Escape')closeDrawer()});
</script>
</body>
</html>'''

def render(tpl, **kw):
    sync_xu()
    return render_template_string(BASE.replace('{% block content %}{% endblock %}', tpl), **kw)

# ══════════════════════════════════════════
#  HOME
# ══════════════════════════════════════════
@app.route('/')
@login_required
def home():
    db = get_db()
    featured = db.execute('SELECT * FROM accounts WHERE claimed_by IS NULL LIMIT 4').fetchall()
    stats = {
        'users': db.execute('SELECT COUNT(*) FROM users').fetchone()[0],
        'accs': db.execute('SELECT COUNT(*) FROM accounts WHERE claimed_by IS NULL').fetchone()[0],
        'trades': db.execute('SELECT COUNT(*) FROM trao_doi WHERE status="xong"').fetchone()[0],
        'xu': db.execute('SELECT SUM(xu) FROM users').fetchone()[0] or 0,
    }
    db.close()
    tpl = '''{% block content %}
<div class="hero">
  <h1>LẤY ACC <span class="hl">AOV</span> FREE 🎮</h1>
  <p>Cộng đồng chia sẻ tài khoản miễn phí. Làm nhiệm vụ, kiếm xu, đổi acc xịn!</p>
  <div class="stats-row">
    <div class="stat-pill">👤 Thành viên: <b>{{s.users}}</b></div>
    <div class="stat-pill">🎁 Acc còn: <b>{{s.accs}}</b></div>
    <div class="stat-pill">🔄 Giao dịch: <b>{{s.trades}}</b></div>
    <div class="stat-pill">🪙 Xu: <b>{{s.xu}}</b></div>
  </div>
</div>
<div class="section">
  <div class="xu-big">
    <div class="xu-num">{{session.get('xu',0)}}</div>
    <div class="xu-lbl">Xu của bạn — làm nhiệm vụ để kiếm thêm hoặc nạp tiền</div>
  </div>
  <div class="sec-title">🔥 Acc Nổi Bật</div>
  <div class="grid2">
    {% for a in featured %}
    <div class="card">
      <div class="card-name">{{a['ten_acc']}}</div>
      <div class="metas">
        <span class="badge b-rank">{{a['rank']}}</span>
        <span class="badge b-hero">{{a['tuong']}} tướng</span>
        <span class="badge b-hot">HOT</span>
      </div>
      <div class="card-bot">
        <span class="price">🪙 {{a['xu_gia']}} xu</span>
        <a href="/lay-acc" class="btn btn-p btn-sm">Xem Ngay</a>
      </div>
    </div>
    {% endfor %}
  </div>
  <div style="display:grid;grid-template-columns:1fr 1fr;gap:.75rem;margin-top:1rem">
    <a href="/nap-tien" class="btn btn-g btn-full" style="display:flex;align-items:center;justify-content:center;gap:.4rem">💳 Nạp Tiền</a>
    <a href="/random-acc" class="btn btn-p btn-full" style="display:flex;align-items:center;justify-content:center;gap:.4rem">🎰 Random Acc</a>
  </div>
</div>{% endblock %}'''
    return render(tpl, page='home', featured=featured, s=stats)

# ══════════════════════════════════════════
#  LAY ACC
# ══════════════════════════════════════════
@app.route('/lay-acc')
@login_required
def lay_acc():
    db = get_db()
    accounts = db.execute('SELECT * FROM accounts ORDER BY claimed_by IS NOT NULL, id').fetchall()
    db.close()
    tpl = '''{% block content %}
<div class="section">
  {% for m in get_flashed_messages(category_filter=['success']) %}
  <div class="al al-ok">✅ {{m}}</div>{% endfor %}
  {% for m in get_flashed_messages(category_filter=['error']) %}
  <div class="al al-err">❌ {{m}}</div>{% endfor %}
  <div class="sec-title">🎁 Kho Acc AOV Free</div>
  <div class="grid2">
    {% for a in accounts %}
    <div class="card">
      <div class="card-name">{{a['ten_acc']}}</div>
      <div class="metas">
        <span class="badge b-rank">{{a['rank']}}</span>
        <span class="badge b-hero">{{a['tuong']}} tướng</span>
        {% if not a['claimed_by'] %}<span class="badge b-free">TRỐNG</span>{% endif %}
      </div>
      {% if a['mo_ta'] %}<div style="font-size:.75rem;color:var(--muted);margin-bottom:.5rem">{{a['mo_ta']}}</div>{% endif %}
      <div class="card-bot">
        <span class="price">🪙 {{a['xu_gia']}} xu</span>
        {% if a['claimed_by'] %}
          <span style="font-size:.72rem;color:var(--muted)">Đã lấy</span>
        {% else %}
          <form method="POST" action="/lay-acc/{{a['id']}}">
            <button class="btn btn-g btn-sm" type="submit">Lấy Acc</button>
          </form>
        {% endif %}
      </div>
    </div>
    {% else %}
    <div style="grid-column:1/-1;text-align:center;padding:2rem;color:var(--muted)">📭 Chưa có acc</div>
    {% endfor %}
  </div>
</div>{% endblock %}'''
    return render(tpl, page='acc', accounts=accounts)

@app.route('/lay-acc/<int:aid>', methods=['POST'])
@login_required
def claim(aid):
    db = get_db()
    a = db.execute('SELECT * FROM accounts WHERE id=?', (aid,)).fetchone()
    if not a: flash('Không tồn tại!','error'); db.close(); return redirect('/lay-acc')
    if a['claimed_by']: flash('Acc đã được lấy rồi!','error'); db.close(); return redirect('/lay-acc')
    u = db.execute('SELECT * FROM users WHERE username=?', (session['user'],)).fetchone()
    if u['xu'] < a['xu_gia']:
        flash(f'Không đủ xu! Cần {a["xu_gia"]} xu, bạn có {u["xu"]} xu.','error')
        db.close(); return redirect('/lay-acc')
    db.execute('UPDATE accounts SET claimed_by=? WHERE id=?', (session['user'], aid))
    db.execute('UPDATE users SET xu=xu-? WHERE username=?', (a['xu_gia'], session['user']))
    db.commit(); db.close(); sync_xu()
    flash(f'🎉 Lấy acc "{a["ten_acc"]}" thành công! Liên hệ admin để nhận.','success')
    return redirect('/lay-acc')

# ══════════════════════════════════════════
#  RANDOM ACC
# ══════════════════════════════════════════
@app.route('/random-acc', methods=['GET','POST'])
@login_required
def random_acc():
    result = None
    if request.method == 'POST':
        db = get_db()
        u = db.execute('SELECT * FROM users WHERE username=?', (session['user'],)).fetchone()
        if u['xu'] < 2000:
            flash('Không đủ xu! Cần 2000 xu để random acc.','error')
            db.close()
        else:
            avail = db.execute('SELECT * FROM accounts WHERE claimed_by IS NULL').fetchall()
            if not avail:
                flash('Hết acc rồi! Chờ admin thêm nhé.','error'); db.close()
            else:
                picked = random.choice(avail)
                db.execute('UPDATE accounts SET claimed_by=? WHERE id=?', (session['user'], picked['id']))
                db.execute('UPDATE users SET xu=xu-2000 WHERE username=?', (session['user'],))
                db.commit(); db.close(); sync_xu()
                result = dict(picked)
    tpl = '''{% block content %}
<div class="section">
  {% for m in get_flashed_messages(category_filter=['error']) %}
  <div class="al al-err">❌ {{m}}</div>{% endfor %}
  {% if result %}
  <div class="al al-ok">🎉 Chúc mừng! Bạn nhận được acc ngẫu nhiên!</div>
  <div class="random-result">
    <div style="font-size:1.1rem;font-weight:900;margin-bottom:.5rem">{{result['ten_acc']}}</div>
    <div class="metas" style="margin-bottom:.7rem">
      <span class="badge b-rank">{{result['rank']}}</span>
      <span class="badge b-hero">{{result['tuong']}} tướng</span>
    </div>
    <div style="font-size:.78rem;color:var(--muted)">{{result['mo_ta'] or ''}}</div>
    <div style="margin-top:.75rem;font-size:.78rem;color:var(--green)">✅ Liên hệ admin để nhận thông tin đăng nhập</div>
  </div>
  {% else %}
  <div class="random-box">
    <div class="icon-big">🎰</div>
    <div style="font-size:1.1rem;font-weight:800;margin-bottom:.4rem">Random Acc Ngẫu Nhiên</div>
    <div style="font-size:.82rem;color:var(--muted);margin-bottom:1.25rem">
      Thử vận may — nhận acc ngẫu nhiên trong kho!<br>
      <span style="color:var(--gold);font-weight:700">Chi phí: 2000 xu / lượt</span>
    </div>
    <div style="font-size:.78rem;color:var(--muted);margin-bottom:1rem">Xu hiện tại: <b style="color:var(--gold)">{{session.get('xu',0)}}</b></div>
    {% if session.get('xu',0) >= 2000 %}
    <form method="POST">
      <button class="btn btn-g btn-full" type="submit" style="max-width:220px;margin:0 auto;display:block">
        🎰 Quay Ngay!
      </button>
    </form>
    {% else %}
    <a href="/nap-tien" class="btn btn-p btn-full" style="max-width:220px;margin:0 auto;display:block">
      💳 Nạp Xu Để Chơi
    </a>
    {% endif %}
  </div>
  {% endif %}
</div>{% endblock %}'''
    return render(tpl, page='random', result=result)

# ══════════════════════════════════════════
#  NHIEM VU
# ══════════════════════════════════════════
@app.route('/nhiem-vu')
@login_required
def nhiem_vu():
    db = get_db()
    u = db.execute('SELECT id FROM users WHERE username=?', (session['user'],)).fetchone()
    done = {r['task'] for r in db.execute('SELECT task FROM nhiem_vu WHERE user_id=? AND done=1', (u['id'],)).fetchall()}
    db.close()
    tasks = [
        {'id':'dang_nhap','icon':'🌅','title':'Đăng nhập hàng ngày','xu':20},
        {'id':'chia_se','icon':'📢','title':'Chia sẻ link web cho bạn','xu':50},
        {'id':'xem_acc','icon':'👀','title':'Xem 5 acc trong kho','xu':10},
        {'id':'moi_ban','icon':'👥','title':'Mời 1 người bạn đăng ký','xu':100},
        {'id':'binh_luan','icon':'💬','title':'Để lại đánh giá về web','xu':30},
        {'id':'profile','icon':'✏️','title':'Hoàn thiện hồ sơ cá nhân','xu':40},
        {'id':'link_hay','icon':'🔗','title':'Click vào 1 link hay','xu':15},
    ]
    for t in tasks: t['done'] = t['id'] in done
    tpl = '''{% block content %}
<div class="section">
  {% for m in get_flashed_messages(category_filter=['success']) %}
  <div class="al al-ok">✅ {{m}}</div>{% endfor %}
  {% for m in get_flashed_messages(category_filter=['error']) %}
  <div class="al al-err">❌ {{m}}</div>{% endfor %}
  <div class="sec-title">📋 Nhiệm Vụ Kiếm Xu</div>
  {% for t in tasks %}
  <div class="task-row">
    <div class="task-icon">{{t['icon']}}</div>
    <div class="task-info">
      <div class="task-t">{{t['title']}}</div>
      <div class="task-r">+{{t['xu']}} xu</div>
    </div>
    {% if t['done'] %}
      <span style="font-size:.72rem;font-weight:700;color:var(--green)">✅ Xong</span>
    {% else %}
      <form method="POST" action="/nhiem-vu/{{t['id']}}">
        <button class="btn btn-p btn-sm" type="submit">Làm</button>
      </form>
    {% endif %}
  </div>
  {% endfor %}
</div>{% endblock %}'''
    return render(tpl, page='task', tasks=tasks)

@app.route('/nhiem-vu/<tid>', methods=['POST'])
@login_required
def do_task(tid):
    xm = {'dang_nhap':20,'chia_se':50,'xem_acc':10,'moi_ban':100,'binh_luan':30,'profile':40,'link_hay':15}
    if tid not in xm: return redirect('/nhiem-vu')
    db = get_db()
    u = db.execute('SELECT * FROM users WHERE username=?', (session['user'],)).fetchone()
    if db.execute('SELECT id FROM nhiem_vu WHERE user_id=? AND task=? AND done=1', (u['id'],tid)).fetchone():
        flash('Đã hoàn thành rồi!','error'); db.close(); return redirect('/nhiem-vu')
    db.execute('INSERT INTO nhiem_vu (user_id,task,done,created_at) VALUES (?,?,1,?)',
               (u['id'],tid,datetime.now().strftime('%Y-%m-%d %H:%M')))
    db.execute('UPDATE users SET xu=xu+? WHERE id=?', (xm[tid],u['id']))
    db.commit(); db.close(); sync_xu()
    flash(f'Hoàn thành! +{xm[tid]} xu 🎉','success')
    return redirect('/nhiem-vu')

# ══════════════════════════════════════════
#  TRAO DOI
# ══════════════════════════════════════════
@app.route('/trao-doi')
@login_required
def trao_doi():
    db = get_db()
    trades = db.execute('''SELECT t.*,a.ten_acc,a.rank,a.tuong FROM trao_doi t
                           JOIN accounts a ON t.acc_id=a.id ORDER BY t.id DESC''').fetchall()
    db.close()
    tpl = '''{% block content %}
<div class="section">
  {% for m in get_flashed_messages(category_filter=['success']) %}
  <div class="al al-ok">✅ {{m}}</div>{% endfor %}
  {% for m in get_flashed_messages(category_filter=['error']) %}
  <div class="al al-err">❌ {{m}}</div>{% endfor %}
  <div class="sec-title">🔄 Sàn Trao Đổi</div>
  <div class="grid2">
    {% for t in trades %}
    <div class="card">
      <div class="card-name">{{t['ten_acc']}}</div>
      <div class="metas">
        <span class="badge b-rank">{{t['rank']}}</span>
        <span class="badge b-hero">{{t['tuong']}} tướng</span>
      </div>
      <div style="font-size:.72rem;color:var(--muted);margin-bottom:.5rem">Bán bởi: <b>{{t['nguoi_ban']}}</b></div>
      <div class="card-bot">
        <span class="price">🪙 {{t['gia']}} xu</span>
        {% if t['nguoi_ban'] != session.get('user') and t['status']=='cho' %}
          <form method="POST" action="/trao-doi/mua/{{t['id']}}">
            <button class="btn btn-g btn-sm" type="submit">Mua</button>
          </form>
        {% elif t['nguoi_ban']==session.get('user') %}
          <span style="font-size:.72rem;color:var(--muted)">Của bạn</span>
        {% else %}
          <span style="font-size:.72rem;color:var(--green)">✅ Đã bán</span>
        {% endif %}
      </div>
    </div>
    {% else %}
    <div style="grid-column:1/-1;text-align:center;padding:2rem;color:var(--muted)">🔄 Chưa có giao dịch</div>
    {% endfor %}
  </div>
  <div style="margin-top:1.25rem">
    <div class="sec-title">➕ Đăng Acc Trao Đổi</div>
    <div class="card" style="max-width:480px">
      <form method="POST" action="/trao-doi/dang">
        <div class="fg"><label>Tên Acc</label><input name="ten_acc" placeholder="Sát Thủ Bóng Đêm" required></div>
        <div class="fg"><label>Rank</label><input name="rank" placeholder="Kim Cương / Chinh Phục..." required></div>
        <div style="display:grid;grid-template-columns:1fr 1fr;gap:.6rem">
          <div class="fg"><label>Số Tướng</label><input name="tuong" type="number" placeholder="60" required></div>
          <div class="fg"><label>Giá (xu)</label><input name="gia" type="number" placeholder="1000" required></div>
        </div>
        <button class="btn btn-p btn-full" type="submit">Đăng Lên Sàn 🚀</button>
      </form>
    </div>
  </div>
</div>{% endblock %}'''
    return render(tpl, page='trade', trades=trades)

@app.route('/trao-doi/dang', methods=['POST'])
@login_required
def dang_td():
    db = get_db()
    db.execute('INSERT INTO accounts (ten_acc,rank,tuong,xu_gia,created_at) VALUES (?,?,?,?,?)',
               (request.form['ten_acc'],request.form['rank'],int(request.form['tuong']),
                int(request.form['gia']),datetime.now().strftime('%Y-%m-%d %H:%M')))
    aid = db.execute('SELECT last_insert_rowid()').fetchone()[0]
    db.execute('INSERT INTO trao_doi (nguoi_ban,gia,acc_id,status,created_at) VALUES (?,?,?,?,?)',
               (session['user'],int(request.form['gia']),aid,'cho',datetime.now().strftime('%Y-%m-%d %H:%M')))
    db.commit(); db.close()
    flash('Đăng thành công! 🚀','success'); return redirect('/trao-doi')

@app.route('/trao-doi/mua/<int:tid>', methods=['POST'])
@login_required
def mua_td(tid):
    db = get_db()
    t = db.execute('SELECT * FROM trao_doi WHERE id=?', (tid,)).fetchone()
    if not t or t['status']!='cho': flash('Không hợp lệ!','error'); db.close(); return redirect('/trao-doi')
    u = db.execute('SELECT * FROM users WHERE username=?', (session['user'],)).fetchone()
    if u['xu'] < t['gia']: flash(f'Không đủ xu!','error'); db.close(); return redirect('/trao-doi')
    db.execute('UPDATE users SET xu=xu-? WHERE username=?', (t['gia'],session['user']))
    db.execute('UPDATE users SET xu=xu+? WHERE username=?', (t['gia'],t['nguoi_ban']))
    db.execute('UPDATE trao_doi SET nguoi_mua=?,status="xong" WHERE id=?', (session['user'],tid))
    db.execute('UPDATE accounts SET claimed_by=? WHERE id=?', (session['user'],t['acc_id']))
    db.commit(); db.close(); sync_xu()
    flash('Mua thành công! Liên hệ người bán nhận acc. 🎉','success'); return redirect('/trao-doi')

# ══════════════════════════════════════════
#  MOI BAN
# ══════════════════════════════════════════
@app.route('/moi-ban')
@login_required
def moi_ban():
    db = get_db()
    u = db.execute('SELECT * FROM users WHERE username=?', (session['user'],)).fetchone()
    friends = db.execute('SELECT username,created_at FROM users WHERE invited_by=?', (session['user'],)).fetchall()
    db.close()
    tpl = '''{% block content %}
<div class="section">
  <div class="sec-title">👥 Mời Bạn Bè — Kiếm Xu</div>
  <div class="card" style="margin-bottom:1rem">
    <div style="font-size:.78rem;color:var(--muted);margin-bottom:.6rem">Mã mời của bạn</div>
    <div class="inv-box">
      <div class="inv-code">{{user['invite_code']}}</div>
      <button class="btn btn-g btn-sm" onclick="navigator.clipboard.writeText('{{user[\'invite_code\']}}');this.textContent='✅ Đã copy!'">Copy</button>
    </div>
    <div style="margin-top:.9rem;font-size:.78rem;color:var(--muted);line-height:1.7">
      Bạn mời đăng ký → Bạn nhận <b style="color:var(--gold)">+50 xu</b><br>
      Người bạn nhận thêm <b style="color:var(--gold)">+30 xu</b> khi dùng mã của bạn!
    </div>
  </div>
  <div class="sec-title">📊 Người bạn đã mời ({{friends|length}})</div>
  {% for f in friends %}
  <div class="lb-row">
    <div style="font-size:1rem">👤</div>
    <div class="lb-name">{{f['username']}}</div>
    <div style="font-size:.72rem;color:var(--green)">+50 xu</div>
  </div>
  {% else %}
  <div style="text-align:center;padding:1.5rem;color:var(--muted)">👥 Chưa có ai dùng mã của bạn</div>
  {% endfor %}
</div>{% endblock %}'''
    return render(tpl, page='invite', user=u, friends=friends)

# ══════════════════════════════════════════
#  DUA TOP
# ══════════════════════════════════════════
@app.route('/dua-top')
@login_required
def dua_top():
    db = get_db()
    leaders = db.execute('SELECT username,xu FROM users ORDER BY xu DESC LIMIT 20').fetchall()
    db.close()
    tpl = '''{% block content %}
<div class="section">
  <div class="sec-title">🏆 Bảng Xếp Hạng Xu</div>
  {% for u in leaders %}
  <div class="lb-row">
    <div class="lb-rank {{'t1' if loop.index==1 else 't2' if loop.index==2 else 't3' if loop.index==3 else ''}}">
      {{'🥇' if loop.index==1 else '🥈' if loop.index==2 else '🥉' if loop.index==3 else '#'+loop.index|string}}
    </div>
    <div class="lb-name">{{u['username']}}{% if u['username']==session.get('user') %} <span style="font-size:.68rem;color:var(--purple)">(bạn)</span>{% endif %}</div>
    <div class="lb-xu">🪙 {{u['xu']}}</div>
  </div>
  {% endfor %}
</div>{% endblock %}'''
    return render(tpl, page='top', leaders=leaders)

# ══════════════════════════════════════════
#  NAP TIEN
# ══════════════════════════════════════════
@app.route('/nap-tien', methods=['GET','POST'])
@login_required
def nap_tien():
    step = request.args.get('step','1')
    so_tien = request.args.get('so_tien','')
    ma_don = request.args.get('ma_don','')
    ma_ck = get_transfer_code()

    if request.method == 'POST':
        action = request.form.get('action')
        if action == 'nhap_tien':
            st = int(request.form.get('so_tien', 0))
            if st < 1000 or st > 2000:
                flash('Số tiền từ 1.000đ đến 2.000đ!', 'error')
                return redirect('/nap-tien')
            md = rand_code(8)[:6] + str(random.randint(10,99))
            return redirect(f'/nap-tien?step=2&so_tien={st}&ma_don={md}')
        elif action == 'da_chuyen':
            st = int(request.form.get('so_tien', 0))
            md = request.form.get('ma_don','')
            mck = request.form.get('ma_ck','')
            db = get_db()
            db.execute('INSERT INTO don_nap (username,so_tien,xu_nhan,ma_don,ma_ck,status,created_at) VALUES (?,?,?,?,?,?,?)',
                       (session['user'], st, st, md, mck, 'cho', datetime.now().strftime('%Y-%m-%d %H:%M')))
            db.commit(); db.close()
            flash(f'✅ Đã gửi đơn nạp {st:,}đ! Admin sẽ duyệt sớm nhất.', 'success')
            return redirect('/nap-tien?step=3')

    tpl = '''{% block content %}
<div class="section" style="max-width:480px">
  {% for m in get_flashed_messages(category_filter=['success']) %}
  <div class="al al-ok">{{m}}</div>{% endfor %}
  {% for m in get_flashed_messages(category_filter=['error']) %}
  <div class="al al-err">❌ {{m}}</div>{% endfor %}

  {% if step=='1' %}
  <div class="sec-title">💳 Nạp Tiền — Mua Xu</div>
  <div class="al al-info">💡 1.000đ = 1.000 xu &nbsp;|&nbsp; Tối đa 2.000đ/lần</div>
  <div class="card">
    <form method="POST" action="/nap-tien">
      <input type="hidden" name="action" value="nhap_tien">
      <div class="fg">
        <label>Số tiền muốn nạp (đ)</label>
        <input type="number" name="so_tien" placeholder="VD: 2000" min="1000" max="2000" step="1000" required>
        <div class="fhint">Tối thiểu 1.000đ — Tối đa 2.000đ</div>
      </div>
      <button class="btn btn-g btn-full" type="submit">Tiếp Tục →</button>
    </form>
  </div>

  {% elif step=='2' %}
  <div class="sec-title">💳 Thông Tin Chuyển Khoản</div>
  <div class="nap-info">
    <div class="nap-row"><span class="nap-lbl">Người nhận</span><span class="nap-val">VÕ PHÚC DUY</span></div>
    <div class="nap-row"><span class="nap-lbl">Ngân hàng</span><span class="nap-val">ZaloPay</span></div>
    <div class="nap-row"><span class="nap-lbl">Số tiền</span><span class="nap-val" style="color:var(--gold)">{{'{:,}'.format(so_tien|int)}} đ</span></div>
    <div class="nap-row"><span class="nap-lbl">Mã đơn</span><span class="nap-val" style="color:var(--blue)">{{ma_don}}</span></div>
    <div class="nap-row" style="flex-direction:column;gap:.5rem">
      <span class="nap-lbl">Nội dung chuyển khoản</span>
      <div style="text-align:center">
        <div class="ma-ck" id="maCK">{{ma_ck}}</div>
        <div class="countdown">Mã thay đổi sau: <span id="timer">--</span> giây</div>
      </div>
    </div>
  </div>
  <div style="text-align:center;margin:1rem 0">
    <div style="font-size:.78rem;color:var(--muted);margin-bottom:.75rem">Quét mã QR ZaloPay</div>
    <img src="https://i.ibb.co/kggJHx54/file-0000000057fc71fa99d32f47ae399a21.png" class="qr-img" alt="QR ZaloPay">
  </div>
  <div class="luu-y">
    ⚠️ <b>Lưu ý quan trọng:</b><br>
    • Nhớ <b>dán đúng mã chuyển khoản</b> vào nội dung CK<br>
    • Mã thay đổi mỗi 5 phút — copy trước khi CK<br>
    • Admin nghèo nên <b>không có auto bank</b> 😅<br>
    • Sau khi CK xong ấn nút bên dưới để thông báo admin<br>
    • Cảm ơn bạn đã ủng hộ web! ❤️
  </div>
  <form method="POST" action="/nap-tien" style="margin-top:1rem">
    <input type="hidden" name="action" value="da_chuyen">
    <input type="hidden" name="so_tien" value="{{so_tien}}">
    <input type="hidden" name="ma_don" value="{{ma_don}}">
    <input type="hidden" name="ma_ck" value="{{ma_ck}}">
    <button class="btn btn-green btn-full" type="submit">✅ Đã Chuyển Khoản Xong</button>
  </form>

  {% elif step=='3' %}
  <div style="text-align:center;padding:2rem">
    <div style="font-size:3rem;margin-bottom:.75rem">✅</div>
    <div style="font-size:1.1rem;font-weight:800;margin-bottom:.5rem">Đã nhận đơn!</div>
    <div style="font-size:.82rem;color:var(--muted);margin-bottom:1.5rem;line-height:1.7">
      Admin sẽ kiểm tra và duyệt xu cho bạn sớm nhất có thể.<br>
      Cảm ơn bạn đã ủng hộ vpduy.site! ❤️
    </div>
    <a href="/" class="btn btn-p">🏠 Về Trang Chủ</a>
  </div>
  {% endif %}
</div>
<script>
function updateTimer(){
  var now=Math.floor(Date.now()/1000);
  var slot=Math.floor(now/300);
  var next=(slot+1)*300;
  var left=next-now;
  var el=document.getElementById('timer');
  if(el) el.textContent=left;
  setTimeout(updateTimer,1000);
}
updateTimer();
</script>
{% endblock %}'''
    return render(tpl, page='nap', step=step, so_tien=so_tien, ma_don=ma_don, ma_ck=ma_ck)

# ══════════════════════════════════════════
#  LINK HAY
# ══════════════════════════════════════════
@app.route('/link-hay')
@login_required
def link_hay():
    db = get_db()
    links = db.execute('SELECT * FROM link_rut_gon ORDER BY id DESC').fetchall()
    db.close()
    tpl = '''{% block content %}
<div class="section">
  <div class="sec-title">🔗 Link Hay</div>
  {% if links %}
    {% for l in links %}
    <div class="card" style="margin-bottom:.7rem">
      <div style="font-weight:700;margin-bottom:.3rem">{{l['tieu_de']}}</div>
      {% if l['mo_ta'] %}<div style="font-size:.76rem;color:var(--muted);margin-bottom:.6rem">{{l['mo_ta']}}</div>{% endif %}
      <a href="{{l['url']}}" target="_blank" class="btn btn-p btn-sm">Truy Cập →</a>
    </div>
    {% endfor %}
  {% else %}
    <div style="text-align:center;padding:2rem;color:var(--muted)">🔗 Admin chưa thêm link nào</div>
  {% endif %}
</div>{% endblock %}'''
    return render(tpl, page='link', links=links)

# ══════════════════════════════════════════
#  ADMIN PANEL
# ══════════════════════════════════════════
@app.route('/admin')
@login_required
@admin_required
def admin():
    db = get_db()
    users = db.execute('SELECT * FROM users ORDER BY xu DESC').fetchall()
    accs = db.execute('SELECT * FROM accounts ORDER BY id DESC LIMIT 20').fetchall()
    don_nap = db.execute('SELECT * FROM don_nap ORDER BY id DESC LIMIT 30').fetchall()
    links = db.execute('SELECT * FROM link_rut_gon ORDER BY id DESC').fetchall()
    pending = db.execute('SELECT COUNT(*) FROM don_nap WHERE status="cho"').fetchone()[0]
    db.close()
    tpl = '''{% block content %}
<div class="section" style="max-width:900px">
  <div style="display:flex;align-items:center;gap:.75rem;margin-bottom:1rem">
    <div style="font-size:1.3rem;font-weight:900;color:var(--gold)">⚙️ Admin Panel</div>
    {% if pending > 0 %}
    <span style="background:var(--red);color:#fff;font-size:.72rem;font-weight:700;padding:.2rem .6rem;border-radius:10px">{{pending}} đơn chờ</span>
    {% endif %}
  </div>

  <!-- DON NAP -->
  <div class="sec-title">💳 Đơn Nạp Chờ Duyệt</div>
  <div style="overflow-x:auto;margin-bottom:1.5rem">
    <table class="admin-table">
      <tr><th>User</th><th>Tiền</th><th>Xu</th><th>Mã Đơn</th><th>Mã CK</th><th>Thời gian</th><th>Thao tác</th></tr>
      {% for d in don_nap %}
      <tr>
        <td><b>{{d['username']}}</b></td>
        <td>{{d['so_tien']}}đ</td>
        <td style="color:var(--gold)">{{d['xu_nhan']}} xu</td>
        <td style="font-size:.72rem;font-family:monospace">{{d['ma_don']}}</td>
        <td style="font-size:.72rem;font-family:monospace;color:var(--purple)">{{d['ma_ck']}}</td>
        <td style="font-size:.7rem;color:var(--muted)">{{d['created_at'][:16] if d['created_at'] else ''}}</td>
        <td>
          {% if d['status']=='cho' %}
          <form method="POST" action="/admin/duyet-don/{{d['id']}}" style="display:inline">
            <input type="hidden" name="action" value="duyet">
            <button class="btn btn-green btn-sm" type="submit">✅ Duyệt</button>
          </form>
          <form method="POST" action="/admin/duyet-don/{{d['id']}}" style="display:inline">
            <input type="hidden" name="action" value="tu_choi">
            <button class="btn btn-r btn-sm" type="submit">❌</button>
          </form>
          {% else %}
          <span style="font-size:.72rem;color:{{'var(--green)' if d['status']=='duyet' else 'var(--red)'}}">
            {{'✅ Đã duyệt' if d['status']=='duyet' else '❌ Từ chối'}}
          </span>
          {% endif %}
        </td>
      </tr>
      {% else %}
      <tr><td colspan="7" style="text-align:center;color:var(--muted);padding:1rem">Chưa có đơn nào</td></tr>
      {% endfor %}
    </table>
  </div>

  <!-- BUFF XU / BAN -->
  <div class="sec-title">👤 Quản Lý Người Dùng</div>
  <div style="overflow-x:auto;margin-bottom:1.5rem">
    <table class="admin-table">
      <tr><th>Username</th><th>Email</th><th>Xu</th><th>Trạng thái</th><th>Buff Xu</th><th>Trừ Xu</th><th>Ban</th></tr>
      {% for u in users %}
      <tr>
        <td><b>{{u['username']}}</b></td>
        <td style="font-size:.72rem;color:var(--muted)">{{u['email'] or '—'}}</td>
        <td style="color:var(--gold)">{{u['xu']}}</td>
        <td><span style="font-size:.7rem;color:{{'var(--red)' if u['banned'] else 'var(--green)'}}">
          {{'🚫 Bị ban' if u['banned'] else '✅ OK'}}
        </span></td>
        <td>
          <form method="POST" action="/admin/buff-xu" style="display:flex;gap:.3rem">
            <input type="hidden" name="username" value="{{u['username']}}">
            <input type="hidden" name="action" value="buff">
            <input type="number" name="so_xu" style="width:65px;padding:.25rem .4rem;font-size:.72rem;background:var(--card2);border:1px solid var(--border);border-radius:5px;color:var(--text)" placeholder="xu">
            <button class="btn btn-green btn-sm" type="submit">+</button>
          </form>
        </td>
        <td>
          <form method="POST" action="/admin/buff-xu" style="display:flex;gap:.3rem">
            <input type="hidden" name="username" value="{{u['username']}}">
            <input type="hidden" name="action" value="tru">
            <input type="number" name="so_xu" style="width:65px;padding:.25rem .4rem;font-size:.72rem;background:var(--card2);border:1px solid var(--border);border-radius:5px;color:var(--text)" placeholder="xu">
            <button class="btn btn-r btn-sm" type="submit">-</button>
          </form>
        </td>
        <td>
          {% if u['username'] != 'vophucduy' %}
          <form method="POST" action="/admin/ban/{{u['id']}}">
            <button class="btn btn-sm {{'btn-green' if u['banned'] else 'btn-r'}}" type="submit">
              {{'Bỏ Ban' if u['banned'] else '🚫 Ban'}}
            </button>
          </form>
          {% endif %}
        </td>
      </tr>
      {% endfor %}
    </table>
  </div>

  <!-- THEM ACC -->
  <div class="sec-title">🎁 Thêm Acc Mới</div>
  <div class="card" style="max-width:500px;margin-bottom:1.5rem">
    <form method="POST" action="/admin/them-acc">
      <div class="fg"><label>Tên Acc</label><input name="ten_acc" required placeholder="Sát Thủ Rừng Xanh"></div>
      <div class="fg"><label>Rank</label><input name="rank" required placeholder="Kim Cương / Chinh Phục..."></div>
      <div class="fg"><label>Mô tả</label><textarea name="mo_ta" placeholder="Thông tin thêm về acc..."></textarea></div>
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:.6rem">
        <div class="fg"><label>Số Tướng</label><input type="number" name="tuong" required placeholder="80"></div>
        <div class="fg"><label>Giá (xu)</label><input type="number" name="xu_gia" required placeholder="1500"></div>
      </div>
      <button class="btn btn-g btn-full" type="submit">Thêm Acc ✅</button>
    </form>
  </div>

  <!-- LINK RUT GON -->
  <div class="sec-title">🔗 Quản Lý Link Hay</div>
  <div class="card" style="max-width:500px;margin-bottom:1rem">
    <form method="POST" action="/admin/them-link">
      <div class="fg"><label>Tiêu đề</label><input name="tieu_de" required placeholder="Link kiếm tiền xịn"></div>
      <div class="fg"><label>URL</label><input name="url" required placeholder="https://..."></div>
      <div class="fg"><label>Mô tả</label><input name="mo_ta" placeholder="Kiếm được bao nhiêu..."></div>
      <button class="btn btn-p btn-full" type="submit">Thêm Link</button>
    </form>
  </div>
  {% for l in links %}
  <div style="display:flex;align-items:center;gap:.6rem;margin-bottom:.4rem">
    <div style="flex:1;font-size:.8rem">🔗 {{l['tieu_de']}} — <span style="color:var(--muted)">{{l['url'][:40]}}...</span></div>
    <form method="POST" action="/admin/xoa-link/{{l['id']}}">
      <button class="btn btn-r btn-sm" type="submit">Xóa</button>
    </form>
  </div>
  {% endfor %}
</div>{% endblock %}'''
    return render(tpl, page='admin', users=users, accs=accs, don_nap=don_nap, links=links, pending=pending)

@app.route('/admin/duyet-don/<int:did>', methods=['POST'])
@login_required
@admin_required
def duyet_don(did):
    action = request.form.get('action')
    db = get_db()
    don = db.execute('SELECT * FROM don_nap WHERE id=?', (did,)).fetchone()
    if don and don['status']=='cho':
        if action=='duyet':
            db.execute('UPDATE don_nap SET status="duyet" WHERE id=?', (did,))
            db.execute('UPDATE users SET xu=xu+? WHERE username=?', (don['xu_nhan'], don['username']))
            db.commit()
        elif action=='tu_choi':
            db.execute('UPDATE don_nap SET status="tu_choi" WHERE id=?', (did,))
            db.commit()
    db.close()
    return redirect('/admin')

@app.route('/admin/buff-xu', methods=['POST'])
@login_required
@admin_required
def buff_xu():
    uname = request.form.get('username')
    so_xu = int(request.form.get('so_xu', 0))
    action = request.form.get('action')
    db = get_db()
    if action=='buff':
        db.execute('UPDATE users SET xu=xu+? WHERE username=?', (so_xu, uname))
    elif action=='tru':
        db.execute('UPDATE users SET xu=MAX(0,xu-?) WHERE username=?', (so_xu, uname))
    db.commit(); db.close()
    return redirect('/admin')

@app.route('/admin/ban/<int:uid>', methods=['POST'])
@login_required
@admin_required
def ban_user(uid):
    db = get_db()
    u = db.execute('SELECT banned FROM users WHERE id=?', (uid,)).fetchone()
    if u: db.execute('UPDATE users SET banned=? WHERE id=?', (0 if u['banned'] else 1, uid))
    db.commit(); db.close()
    return redirect('/admin')

@app.route('/admin/them-acc', methods=['POST'])
@login_required
@admin_required
def them_acc():
    db = get_db()
    db.execute('INSERT INTO accounts (ten_acc,rank,tuong,xu_gia,mo_ta,created_at) VALUES (?,?,?,?,?,?)',
               (request.form['ten_acc'],request.form['rank'],int(request.form['tuong']),
                int(request.form['xu_gia']),request.form.get('mo_ta',''),
                datetime.now().strftime('%Y-%m-%d %H:%M')))
    db.commit(); db.close()
    return redirect('/admin')

@app.route('/admin/them-link', methods=['POST'])
@login_required
@admin_required
def them_link():
    db = get_db()
    db.execute('INSERT INTO link_rut_gon (tieu_de,url,mo_ta,created_at) VALUES (?,?,?,?)',
               (request.form['tieu_de'],request.form['url'],request.form.get('mo_ta',''),
                datetime.now().strftime('%Y-%m-%d %H:%M')))
    db.commit(); db.close()
    return redirect('/admin')

@app.route('/admin/xoa-link/<int:lid>', methods=['POST'])
@login_required
@admin_required
def xoa_link(lid):
    db = get_db(); db.execute('DELETE FROM link_rut_gon WHERE id=?', (lid,)); db.commit(); db.close()
    return redirect('/admin')

# ══════════════════════════════════════════
#  AUTH
# ══════════════════════════════════════════
@app.route('/dang-nhap', methods=['GET','POST'])
def login():
    if 'user' in session: return redirect('/')
    banned = request.args.get('banned')
    if request.method == 'POST':
        u = request.form.get('username','').strip()
        p = request.form.get('password','')
        # Admin hardcoded
        if u == ADMIN_USER and p == ADMIN_PASS:
            session['user'] = ADMIN_USER
            db = get_db()
            if not db.execute('SELECT id FROM users WHERE username=?', (ADMIN_USER,)).fetchone():
                db.execute('INSERT INTO users (username,password,xu,invite_code,created_at) VALUES (?,?,?,?,?)',
                           (ADMIN_USER, hash_pw(ADMIN_PASS), 999999, 'ADMIN001', datetime.now().strftime('%Y-%m-%d %H:%M')))
                db.commit()
            xu = db.execute('SELECT xu FROM users WHERE username=?', (ADMIN_USER,)).fetchone()
            session['xu'] = xu['xu'] if xu else 999999
            db.close()
            return redirect('/admin')
        db = get_db()
        user = db.execute('SELECT id,username,xu,banned FROM users WHERE username=? AND password=?', (u, hash_pw(p))).fetchone()
        db.close()
        if user:
            if user['banned']: flash('Tài khoản bị ban!','error')
            else:
                session['user'] = user['username']; session['xu'] = user['xu']
                return redirect('/')
        else: flash('Sai tên hoặc mật khẩu!','error')
    tpl = '''{% block content %}
<div class="auth-page">
  <div class="auth-glow"></div>
  <div class="auth-wrap">
    <div class="auth-brand">
      <div class="auth-icon">🎮</div>
      <div class="auth-logo">vpduy<span>.site</span></div>
      <div class="auth-sub">Nền tảng acc AOV hàng đầu</div>
    </div>
    <div class="auth-card">
      <div class="auth-title">Đăng Nhập</div>
      {% if banned %}<div class="al al-err">🚫 Tài khoản của bạn đã bị ban!</div>{% endif %}
      {% for m in get_flashed_messages(category_filter=['error']) %}
      <div class="al al-err">❌ {{m}}</div>{% endfor %}
      {% for m in get_flashed_messages(category_filter=['success']) %}
      <div class="al al-ok">✅ {{m}}</div>{% endfor %}
      <form method="POST">
        <div class="auth-field">
          <div class="auth-field-icon">👤</div>
          <input name="username" placeholder="Tên đăng nhập" required autocomplete="username">
        </div>
        <div class="auth-field">
          <div class="auth-field-icon">🔒</div>
          <input type="password" name="password" placeholder="Mật khẩu" required autocomplete="current-password">
        </div>
        <button type="submit" class="auth-btn auth-btn-purple">Đăng Nhập 🚀</button>
      </form>
      <div class="auth-switch">Chưa có tài khoản? <a href="/dang-ky">Đăng ký miễn phí</a></div>
    </div>
  </div>
</div>{% endblock %}'''
    return render_template_string(BASE.replace('{% block content %}{% endblock %}', tpl), page='login', banned=banned)

@app.route('/dang-ky', methods=['GET','POST'])
def register():
    if 'user' in session: return redirect('/')
    if request.method == 'POST':
        u = request.form.get('username','').strip()
        p = request.form.get('password','')
        email = request.form.get('email','').strip()
        inv = request.form.get('invite_code','').strip()
        if len(u)<4: flash('Tên ít nhất 4 ký tự!','error')
        elif len(p)<6: flash('Mật khẩu ít nhất 6 ký tự!','error')
        elif email and '@' not in email: flash('Email không hợp lệ!','error')
        else:
            db = get_db()
            if db.execute('SELECT id FROM users WHERE username=?', (u,)).fetchone():
                flash('Tên đã tồn tại!','error'); db.close()
            elif email and db.execute('SELECT id FROM users WHERE email=?', (email,)).fetchone():
                flash('Email đã được dùng!','error'); db.close()
            else:
                xu = 100; invited = None
                if inv:
                    inviter = db.execute('SELECT username FROM users WHERE invite_code=?', (inv,)).fetchone()
                    if inviter:
                        invited = inviter['username']; xu += 30
                        db.execute('UPDATE users SET xu=xu+50 WHERE username=?', (inviter['username'],))
                db.execute('INSERT INTO users (username,password,email,xu,invite_code,invited_by,created_at) VALUES (?,?,?,?,?,?,?)',
                           (u,hash_pw(p),email or None,xu,rand_code(),invited,datetime.now().strftime('%Y-%m-%d %H:%M')))
                db.commit(); db.close()
                flash(f'Đăng ký thành công! Nhận {xu} xu khởi đầu 🎉','success')
                return redirect('/dang-nhap')
    tpl = '''{% block content %}
<div class="auth-page">
  <div class="auth-glow"></div>
  <div class="auth-wrap">
    <div class="auth-brand">
      <div class="auth-icon">🎮</div>
      <div class="auth-logo">vpduy<span>.site</span></div>
      <div class="auth-sub">Nền tảng acc AOV hàng đầu</div>
    </div>
    <div class="auth-card">
      <div class="auth-title">Tạo Tài Khoản</div>
      {% for m in get_flashed_messages(category_filter=['error']) %}
      <div class="al al-err">❌ {{m}}</div>{% endfor %}
      {% for m in get_flashed_messages(category_filter=['success']) %}
      <div class="al al-ok">✅ {{m}}</div>{% endfor %}
      <form method="POST">
        <div class="auth-field">
          <div class="auth-field-icon">👤</div>
          <input name="username" placeholder="Tên đăng nhập (tối thiểu 4 ký tự)" required autocomplete="username">
        </div>
        <div class="auth-field">
          <div class="auth-field-icon">✉️</div>
          <input type="email" name="email" placeholder="Email của bạn (không bắt buộc)">
        </div>
        <div class="auth-field">
          <div class="auth-field-icon">🔒</div>
          <input type="password" name="password" placeholder="Mật khẩu (tối thiểu 6 ký tự)" required autocomplete="new-password">
        </div>
        <div class="auth-field">
          <div class="auth-field-icon">🎁</div>
          <input name="invite_code" placeholder="Mã mời (nhận thêm +30 xu)">
        </div>
        <div class="auth-bonus">
          <span>🪙 Khởi đầu nhận <b>100 xu</b> miễn phí!</span>
        </div>
        <button type="submit" class="auth-btn auth-btn-gold">Tạo Tài Khoản ✨</button>
      </form>
      <div class="auth-switch">Đã có tài khoản? <a href="/dang-nhap">Đăng nhập ngay</a></div>
    </div>
  </div>
</div>{% endblock %}'''
    return render_template_string(BASE.replace('{% block content %}{% endblock %}', tpl), page='register')

@app.route('/logout')
def logout():
    session.clear(); return redirect('/dang-nhap')

if __name__ == '__main__':
    init_db()
    port = int(os.environ.get('PORT', 8080))
    print(f"\n🎮 vpduy.site AOV FREE đang chạy!")
    print(f"👉 http://localhost:{port}")
    print(f"🔑 Admin: vophucduy / 0345543851")
    print(f"📌 Ctrl+C để dừng\n")
    app.run(debug=True, port=port, host='0.0.0.0')
