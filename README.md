from flask import Flask, render_template_string, request, redirect, url_for, session
import base64
import hashlib
import hmac
import io
import json
import uuid
import urllib.error
import urllib.request
from urllib.parse import urlencode

import qrcode
import qrcode.constants
from qrcode.image import svg as qr_svg

app = Flask(__name__)
app.secret_key = "nexora_secret_2025"

RAZORPAY_KEY_ID     = "rzp_test_XXXXXXXXXXXXXXXXXX"
RAZORPAY_KEY_SECRET = "YOUR_SECRET_HERE"

UPI_VPA          = "nexora@upi"
UPI_PAYEE_NAME   = "NexoraEduTech"



def _razorpay_keys_ok() -> bool:
    kid, sec = RAZORPAY_KEY_ID.strip(), RAZORPAY_KEY_SECRET.strip()
    return bool(kid) and "XXXXX" not in kid and sec and "YOUR_SECRET" not in sec


def create_razorpay_order(amount_paise: int, receipt: str) -> dict:
    if not _razorpay_keys_ok():
        raise RuntimeError("Razorpay keys not configured")
    url = "https://api.razorpay.com/v1/orders"
    auth = base64.b64encode(f"{RAZORPAY_KEY_ID}:{RAZORPAY_KEY_SECRET}".encode()).decode()
    body = json.dumps(
        {"amount": amount_paise, "currency": "INR", "receipt": receipt[:40]}
    ).encode()
    req = urllib.request.Request(url, data=body, method="POST")
    req.add_header("Authorization", f"Basic {auth}")
    req.add_header("Content-Type", "application/json")
    with urllib.request.urlopen(req, timeout=30) as resp:
        return json.loads(resp.read().decode())


def verify_razorpay_signature(rzp_order_id: str, rzp_payment_id: str, signature: str) -> bool:
    if not signature or not _razorpay_keys_ok():
        return False
    msg = f"{rzp_order_id}|{rzp_payment_id}".encode()
    expected = hmac.new(
        RAZORPAY_KEY_SECRET.encode(), msg, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)


def build_upi_uri(amount: str, service: str, tier: str, order_id: str) -> str:
    note = f"{service} – {tier} · {order_id}"
    if len(note) > 80:
        note = note[:77] + "…"
    params = {
        "pa": UPI_VPA,
        "pn": UPI_PAYEE_NAME,
        "am": f"{float(amount):.2f}",
        "cu": "INR",
        "tn": note,
    }
    return "upi://pay?" + urlencode(params)


def qr_svg_data_uri(payload: str) -> str:
    qr = qrcode.QRCode(
        version=None,
        error_correction=qrcode.constants.ERROR_CORRECT_M,
        box_size=4,
        border=2,
    )
    qr.add_data(payload)
    qr.make(fit=True)
    img = qr.make_image(image_factory=qr_svg.SvgPathImage)
    buf = io.BytesIO()
    img.save(buf)
    b64 = base64.b64encode(buf.getvalue()).decode("ascii")
    return f"data:image/svg+xml;base64,{b64}"

# ════════════════════════════════════════════
#  VIDEO LIBRARY
# ════════════════════════════════════════════
VIDEO_LIBRARY = {
    "Online Learning Program": {
        "Basic": [
            {"id":"v1","title":"Introduction to Online Learning","duration":"12:30","thumb":"https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
            {"id":"v2","title":"Digital Tools for Students","duration":"18:45","thumb":"https://images.unsplash.com/photo-1587620962725-abab7fe55159?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
            {"id":"v3","title":"Effective Note-Taking Strategies","duration":"22:10","thumb":"https://images.unsplash.com/photo-1434030216411-0b793f4b4173?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
        "Standard": [
            {"id":"v4","title":"Live Session: Advanced Concepts","duration":"45:00","thumb":"https://images.unsplash.com/photo-1523580494863-6f3031224c94?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
            {"id":"v5","title":"Mentorship Q&A Session","duration":"30:20","thumb":"https://images.unsplash.com/photo-1556761175-b413da4baf72?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
        "Premium": [
            {"id":"v6","title":"Master Class: Future of EdTech","duration":"60:00","thumb":"https://images.unsplash.com/photo-1552664730-d307ca884978?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
            {"id":"v7","title":"1-on-1 Career Guidance Recording","duration":"55:30","thumb":"https://images.unsplash.com/photo-1507003211169-0a1dd7228f2d?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
    },
    "Professional Training Program": {
        "Basic": [
            {"id":"t1","title":"Python for Professionals","duration":"35:00","thumb":"https://images.unsplash.com/photo-1515879218367-8466d910aaa4?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
            {"id":"t2","title":"Data Analysis Fundamentals","duration":"28:15","thumb":"https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
        "Standard": [
            {"id":"t3","title":"Real-World Project Walkthrough","duration":"50:00","thumb":"https://images.unsplash.com/photo-1522071820081-009f0129c71c?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
            {"id":"t4","title":"Resume & Interview Prep","duration":"40:00","thumb":"https://images.unsplash.com/photo-1560472354-b33ff0c44a43?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
        "Premium": [
            {"id":"t5","title":"Advanced ML & AI Projects","duration":"75:00","thumb":"https://images.unsplash.com/photo-1555949963-aa79dcee981c?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
            {"id":"t6","title":"Cloud Deployment Masterclass","duration":"65:00","thumb":"https://images.unsplash.com/photo-1451187580459-43490279c0fa?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
    },
    "EdTech Consulting Service": {
        "Basic": [
            {"id":"c1","title":"Digital Strategy 101","duration":"20:00","thumb":"https://images.unsplash.com/photo-1454165804606-c3d57bc86b40?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
        "Standard": [
            {"id":"c2","title":"Institution Transformation Case Studies","duration":"45:00","thumb":"https://images.unsplash.com/photo-1552664730-d307ca884978?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
        "Premium": [
            {"id":"c3","title":"Executive EdTech Roadmap Workshop","duration":"90:00","thumb":"https://images.unsplash.com/photo-1522202176988-66273c2fd55f?w=400","embed":"https://www.youtube.com/embed/dQw4w9WgXcQ"},
        ],
    },
}

TIER_ORDER = ["Basic", "Standard", "Premium"]

def get_accessible_videos(service_title, tier):
    videos = []
    for t in TIER_ORDER:
        videos += VIDEO_LIBRARY.get(service_title, {}).get(t, [])
        if t == tier:
            break
    return videos


def list_checkout_courses():
    """All teachable units from VIDEO_LIBRARY plus catalogue labels (courses page)."""
    out = []
    seen = set()
    for program_name, by_tier in VIDEO_LIBRARY.items():
        for tier in TIER_ORDER:
            for vid in by_tier.get(tier, []):
                label = f"{program_name} — {vid['title']}"
                if label in seen:
                    continue
                seen.add(label)
                out.append(label)
    for title in (
        "Python Programming",
        "Data Science",
        "Web Development",
        "Machine Learning",
    ):
        label = f"Open Catalogue — {title}"
        if label not in seen:
            seen.add(label)
            out.append(label)
    return out

services_data = {
    "online": {
        "title":    "Online Learning Program",
        "price":    "4999",
        "standard": "7999",
        "premium":  "12999",
        "points":   [
            "Flexibility to learn anytime",
            "Affordable compared to traditional education",
            "Global recognition of certificates",
            "Expert-led live & recorded sessions"
        ],
        "hero_image": "https://images.unsplash.com/photo-1501504905252-473c47e087f8?w=1400",
        "gallery": [
            "https://images.unsplash.com/photo-1596495577886-d920f1fb7238?w=500",
            "https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=500",
            "https://images.unsplash.com/photo-1434030216411-0b793f4b4173?w=500",
        ],
    },
    "training": {
        "title":    "Professional Training Program",
        "price":    "9999",
        "standard": "14999",
        "premium":  "19999",
        "points":   [
            "Bridges skill gaps for job-readiness",
            "Boosts career advancement",
            "Hands-on project experience",
            "Industry-certified mentors"
        ],
        "hero_image": "https://images.unsplash.com/photo-1531482615713-2afd69097998?w=1400",
        "gallery": [
            "https://images.unsplash.com/photo-1557804506-669a67965ba0?w=500",
            "https://images.unsplash.com/photo-1515879218367-8466d910aaa4?w=500",
            "https://images.unsplash.com/photo-1522071820081-009f0129c71c?w=500",
        ],
    },
    "consulting": {
        "title":    "EdTech Consulting Service",
        "price":    "14999",
        "standard": "19999",
        "premium":  "24999",
        "points":   [
            "Strategic digital transformation guidance",
            "Institution-wide innovation roadmap",
            "Competitive advantage in education sector",
            "Dedicated expert consultant"
        ],
        "hero_image": "https://images.unsplash.com/photo-1517245386807-bb43f82c33c4?w=1400",
        "gallery": [
            "https://images.unsplash.com/photo-1556761175-4b46a572b786?w=500",
            "https://images.unsplash.com/photo-1454165804606-c3d57bc86b40?w=500",
            "https://images.unsplash.com/photo-1522202176988-66273c2fd55f?w=500",
        ],
    },
}
courses_page = """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Courses</title>

<style>
body{
    margin:0;
    font-family:'Segoe UI',sans-serif;
    background:linear-gradient(135deg,#000428,#004e92);
    color:white;
    text-align:center;
}
.container{
    display:flex;
    flex-wrap:wrap;
    justify-content:center;
    margin:40px auto;
    width:90%;
}
.card{
    background:rgba(255,255,255,0.1);
    width:300px;
    margin:20px;
    border-radius:15px;
    padding:15px;
    box-shadow:0 6px 12px rgba(0,0,0,0.5);
}
.card img{
    width:100%;
    height:180px;
    object-fit:cover;
    border-radius:10px;
}
</style>
</head>
<body>

<h1>🎓 Courses We Offer</h1>

<div class="container">

<div class="card">
<img src="https://images.unsplash.com/photo-1515879218367-8466d910aaa4">
<h3>Python Programming</h3>
</div>

<div class="card">
<img src="https://images.unsplash.com/photo-1551288049-bebda4e38f71">
<h3>Data Science</h3>
</div>

<div class="card">
<img src="https://images.unsplash.com/photo-1522071820081-009f0129c71c">
<h3>Web Development</h3>
</div>

<div class="card">
<img src="https://images.unsplash.com/photo-1555949963-aa79dcee981c">
<h3>Machine Learning</h3>
</div>

</div>
</body>
</html>
"""

# ════════════════════════════════════════════
#  PAGE TEMPLATES
# ════════════════════════════════════════════

home_page = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Nexora EduTech</title>
    <style>
        body{margin:0;padding:0;font-family:'Segoe UI',sans-serif;
             background:url('https://images.unsplash.com/photo-1507525428034-b723cf961d3e') no-repeat center center fixed;
             background-size:cover;color:white;text-align:center;}
        .overlay{background:rgba(0,0,0,0.6);height:100vh;display:flex;flex-direction:column;justify-content:center;align-items:center;}
        h1{font-size:4em;margin-bottom:0.3em;text-shadow:2px 2px 8px #000;}
        p{font-size:1.5em;max-width:700px;line-height:1.6;text-shadow:1px 1px 6px #000;}
        .btn{margin-top:20px;padding:15px 30px;font-size:1.2em;border:none;border-radius:8px;
             background:linear-gradient(135deg,#ff7e5f,#feb47b);color:white;cursor:pointer;
             text-decoration:none;transition:transform 0.2s ease;display:inline-block;}
        .btn:hover{transform:scale(1.1);}
        footer{position:absolute;bottom:15px;width:100%;text-align:center;font-size:0.9em;color:#ddd;}
    </style>
</head>
<body>
<div class="overlay">
    <h1>WELCOME TO <i> Nexora</i></h1>
    <p>Empowering education with technology, innovation, and premium digital experiences.</p>
    <a href="/services" class="btn">Explore Services 🚀</a>
    <a href="/faculty"  class="btn">Meet Faculty 👨‍🏫</a>
    <a href="/courses"   class="btn">Course offered 📔</a>
</div>
<footer>📞 +91 1234567890 | ✉ support@Nexoraedutech.com<br>&copy; 2025 Nexora EduTech. All rights reserved.</footer>
</body>
</html>
"""

services_page = """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Our Services</title>
<style>
body{font-family:Segoe UI;margin:0;background:linear-gradient(135deg,#000428,#004e92);color:white;text-align:center;}
.hero{height:250px;background:url('https://images.unsplash.com/photo-1523050854058-8df90110c9f1') no-repeat center center/cover;
      display:flex;justify-content:center;align-items:center;}
.hero h1{font-size:3em;text-shadow:2px 2px 8px #000;}
.services{display:flex;flex-wrap:wrap;justify-content:center;margin:40px auto;width:90%;}
.card{background:rgba(255,255,255,0.1);width:350px;margin:20px;border-radius:15px;
      overflow:hidden;box-shadow:0 6px 12px rgba(0,0,0,0.5);text-align:left;
      transition:transform .25s,box-shadow .25s;}
.card:hover{transform:translateY(-6px);box-shadow:0 14px 28px rgba(0,0,0,0.6);}
.card img{width:100%;height:200px;object-fit:cover;}
.card-content{padding:20px;}
.card h2{margin-top:0;color:#00c6ff;}
.card p{font-size:1em;margin:10px 0;}
.btn{display:inline-block;background:linear-gradient(45deg,#ff512f,#dd2476);
     padding:10px 20px;color:white;text-decoration:none;border-radius:30px;margin-top:10px;}
.btn:hover{opacity:.88;}
</style>
</head>
<body>
<div class="hero"><h1>Our Services</h1></div>
<div class="services">
    <div class="card">
        <img src="https://images.unsplash.com/photo-1596495577886-d920f1fb7238" alt="Online Learning">
        <div class="card-content"><h2>Online Learning</h2>
            <p>Flexible, affordable, and accessible education with live &amp; recorded classes.</p>
            <a href="/services/online" class="btn">View Details</a></div>
    </div>
    <div class="card">
        <img src="https://images.unsplash.com/photo-1557804506-669a67965ba0" alt="Training Programs">
        <div class="card-content"><h2>Training Programs</h2>
            <p>Job-oriented training with real projects, mentorship, and career support.</p>
            <a href="/services/training" class="btn">View Details</a></div>
    </div>
    <div class="card">
        <img src="https://images.unsplash.com/photo-1556761175-4b46a572b786" alt="EdTech Consulting">
        <div class="card-content"><h2>EdTech Consulting</h2>
            <p>Transform your institution with expert digital strategy and innovation.</p>
            <a href="/services/consulting" class="btn">View Details</a></div>
    </div>
</div>
</body>
</html>
"""

detail_template = """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{title}}</title>
<style>
*{box-sizing:border-box;margin:0;padding:0;}
body{font-family:'Segoe UI',sans-serif;background:linear-gradient(135deg,#000428,#004e92);color:#fdfdfd;}

.hero{
    height:420px;
    background:url('{{hero_image}}') no-repeat center center/cover;
    position:relative;display:flex;align-items:flex-end;justify-content:center;
}
.hero::after{content:'';position:absolute;inset:0;background:linear-gradient(to bottom,rgba(0,0,0,.1),rgba(0,4,40,.88));}
.hero h1{position:relative;z-index:1;font-size:2.8em;padding-bottom:28px;
          text-shadow:0 2px 10px rgba(0,0,0,.8);text-align:center;}

.section{width:88%;margin:32px auto;background:rgba(255,255,255,0.08);
         padding:30px 36px;border-radius:20px;box-shadow:0 4px 16px rgba(0,0,0,0.35);}
.section h2{color:#ffd700;font-size:1.5em;margin-bottom:18px;}

.points-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(220px,1fr));gap:14px;margin-bottom:28px;}
.point{background:rgba(255,255,255,0.1);border-radius:10px;padding:14px 16px;font-size:1em;
       border-left:3px solid #00c6ff;}

/* gallery */
.gallery{display:flex;gap:14px;flex-wrap:wrap;margin-top:6px;}
.gallery img{flex:1;min-width:180px;max-width:32%;height:200px;object-fit:cover;
             border-radius:12px;box-shadow:0 4px 12px rgba(0,0,0,0.4);
             transition:transform .25s,box-shadow .25s;}
.gallery img:hover{transform:scale(1.04);box-shadow:0 8px 20px rgba(0,0,0,0.5);}

/* pricing */
.tiers{display:flex;justify-content:space-around;margin-top:10px;flex-wrap:wrap;gap:18px;}
.tier{flex:1;min-width:200px;max-width:290px;padding:26px;background:white;color:#333;
      border-radius:14px;border:2px solid #eee;box-shadow:0 4px 12px rgba(0,0,0,0.12);text-align:center;
      transition:transform .2s,box-shadow .2s;}
.tier:hover{transform:translateY(-4px);box-shadow:0 10px 24px rgba(0,0,0,0.18);}
.tier h3{color:#734b6d;margin-top:0;font-size:1.2em;}
.tier p{font-size:.9em;color:#555;margin:8px 0;}
.tier .price{font-size:2em;font-weight:800;color:#004e92;margin:14px 0 4px;}
.pay-btn{width:100%;padding:12px 0;background:linear-gradient(45deg,#ff512f,#dd2476);
         color:white;border:none;border-radius:30px;font-size:1em;font-weight:700;
         cursor:pointer;margin-top:14px;transition:opacity .2s,transform .2s;}
.pay-btn:hover{opacity:.88;transform:scale(1.02);}
</style>
</head>
<body>

<div class="hero"><h1>{{title}}</h1></div>

<div class="section">
    <h2>✨ Highlights</h2>
    <div class="points-grid">
        {% for p in points %}<div class="point">✔ {{p}}</div>{% endfor %}
    </div>
    <h2>📸 Gallery</h2>
    <div class="gallery">
        {% for img in gallery %}<img src="{{img}}" alt="Service image">{% endfor %}
    </div>
</div>

<div class="section">
    <h2>💳 Choose Your Plan</h2>
    <div class="tiers">
        <div class="tier">
            <h3>⭐ Basic</h3>
            <p>Access to core features and recorded sessions.</p>
            <div class="price">₹{{price}}</div>
            <form method="post" action="/checkout">
                <input type="hidden" name="service" value="{{title}}">
                <input type="hidden" name="tier"    value="Basic">
                <input type="hidden" name="amount"  value="{{price}}">
                <button class="pay-btn">Enroll – Basic</button>
            </form>
        </div>
        <div class="tier">
            <h3>🥈 Standard</h3>
            <p>Includes mentorship, live projects, and certificates.</p>
            <div class="price">₹{{standard}}</div>
            <form method="post" action="/checkout">
                <input type="hidden" name="service" value="{{title}}">
                <input type="hidden" name="tier"    value="Standard">
                <input type="hidden" name="amount"  value="{{standard}}">
                <button class="pay-btn">Enroll – Standard</button>
            </form>
        </div>
        <div class="tier">
            <h3>👑 Premium</h3>
            <p>Lifetime access, advanced resources, and priority support.</p>
            <div class="price">₹{{premium}}</div>
            <form method="post" action="/checkout">
                <input type="hidden" name="service" value="{{title}}">
                <input type="hidden" name="tier"    value="Premium">
                <input type="hidden" name="amount"  value="{{premium}}">
                <button class="pay-btn">Enroll – Premium</button>
            </form>
        </div>
    </div>
</div>

</body>
</html>
"""

# ── Step 1: your details + course (QR comes only after this) ──
checkout_details_template = """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Complete enrolment – Nexora EduTech</title>
<link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap" rel="stylesheet">
<style>
*{box-sizing:border-box;margin:0;padding:0;}
body{font-family:'Nunito',sans-serif;background:linear-gradient(135deg,#000428,#004e92);
     min-height:100vh;display:flex;justify-content:center;align-items:center;padding:30px 16px;}

.card{background:white;border-radius:24px;max-width:560px;width:100%;
      box-shadow:0 24px 64px rgba(0,0,0,0.45);overflow:hidden;}

.card-top{background:linear-gradient(135deg,#000428,#0077b6);padding:24px 32px;text-align:center;color:white;}
.card-top .logo{font-size:1.3em;font-weight:800;}
.card-top .subtitle{font-size:.85em;opacity:.8;margin-top:4px;}

.card-body{padding:30px 32px;}

.summary{background:#f0f7ff;border-radius:12px;padding:14px 18px;margin-bottom:20px;font-size:.9em;}
.summary-row{display:flex;justify-content:space-between;margin:4px 0;color:#333;}
.summary-row strong{color:#0077b6;}

.course-block{margin-top:6px;padding-top:4px;border-top:1px dashed #e2e8f0;}
.form-group select,.form-group textarea{
    width:100%;padding:11px 14px;border:2px solid #e2e8f0;border-radius:10px;
    font-size:.97em;font-family:inherit;color:#222;transition:border-color .2s,box-shadow .2s;
    background:white;}
.form-group textarea{min-height:72px;resize:vertical;}
.form-group select:focus,.form-group textarea:focus{
    outline:none;border-color:#0077b6;box-shadow:0 0 0 3px rgba(0,119,182,.12);}

.form-title{font-size:1em;font-weight:800;color:#1a1a2e;margin-bottom:16px;
            text-transform:uppercase;letter-spacing:.6px;border-bottom:2px solid #e2e8f0;padding-bottom:8px;}
.form-group{margin-bottom:16px;}
.form-group label{display:block;font-size:.82em;font-weight:700;color:#555;margin-bottom:5px;
                  text-transform:uppercase;letter-spacing:.4px;}
.form-group input{width:100%;padding:11px 14px;border:2px solid #e2e8f0;border-radius:10px;
                  font-size:.97em;font-family:inherit;color:#222;transition:border-color .2s,box-shadow .2s;}
.form-group input:focus{outline:none;border-color:#0077b6;box-shadow:0 0 0 3px rgba(0,119,182,.12);}
.form-group input.error,.form-group select.error{border-color:#e74c3c;}
.err-msg{color:#e74c3c;font-size:.78em;margin-top:4px;display:none;}
.row2{display:grid;grid-template-columns:1fr 1fr;gap:14px;}

.pay-btn{width:100%;padding:15px;background:linear-gradient(45deg,#ff512f,#dd2476);color:white;
         border:none;border-radius:12px;font-size:1.05em;font-weight:800;cursor:pointer;
         letter-spacing:.3px;transition:opacity .2s,transform .15s;margin-top:4px;}
.pay-btn:hover{opacity:.92;transform:translateY(-1px);}
.secure{text-align:center;margin-top:12px;font-size:.76em;color:#aaa;}
.secure span{color:#2ecc71;font-weight:700;}
</style>
</head>
<body>
<div class="card">
    <div class="card-top">
        <div class="logo">🎓 Nexora EduTech</div>
        <div class="subtitle">Step 1 of 2 · Your details</div>
    </div>
    <div class="card-body">

        <div class="summary">
            <div class="summary-row"><span>Order</span><strong>#{{order_id}}</strong></div>
            <div class="summary-row"><span>Service</span><strong>{{service}}</strong></div>
            <div class="summary-row"><span>Plan</span><strong>{{tier}}</strong></div>
            <div class="summary-row"><span>Due</span><strong>₹{{amount}}</strong></div>
        </div>
        {% if err %}<p style="color:#e74c3c;font-size:.88em;margin-bottom:12px;">{{ err }}</p>{% endif %}

        <div class="form-title">📋 Your Details</div>
        <form method="post" action="/checkout/details" id="detailsForm">
            <div class="form-group">
                <label>Full Name *</label>
                <input type="text" name="student_name" id="stu-name" placeholder="e.g. Priya Sharma" required autocomplete="name" value="{{ saved_name }}">
            </div>
            <div class="row2">
                <div class="form-group">
                    <label>Email Address *</label>
                    <input type="email" name="student_email" id="stu-email" placeholder="you@example.com" required autocomplete="email" value="{{ saved_email }}">
                </div>
                <div class="form-group">
                    <label>Phone Number *</label>
                    <input type="tel" name="student_phone" id="stu-phone" placeholder="10-digit mobile" maxlength="10" required autocomplete="tel" value="{{ saved_phone }}">
                </div>
            </div>

            <div class="form-title course-block">📚 Course &amp; schedule</div>
            <div class="form-group">
                <label>Select course *</label>
                <select name="student_course_track" id="course-select" required>
                    <option value="">— Choose a course —</option>
                    {% for c in course_choices %}
                    <option value="{{ c }}" {% if saved_course == c %}selected{% endif %}>{{ c }}</option>
                    {% endfor %}
                </select>
            </div>
            <div class="form-group">
                <label>Batch timing preference</label>
                <select name="student_batch_pref" id="batch-pref">
                    <option value="">— Select if applicable —</option>
                    <option value="Morning (6am–12pm)" {% if saved_batch == 'Morning (6am–12pm)' %}selected{% endif %}>Morning (6am–12pm)</option>
                    <option value="Afternoon (12pm–5pm)" {% if saved_batch == 'Afternoon (12pm–5pm)' %}selected{% endif %}>Afternoon (12pm–5pm)</option>
                    <option value="Evening (5pm–10pm)" {% if saved_batch == 'Evening (5pm–10pm)' %}selected{% endif %}>Evening (5pm–10pm)</option>
                    <option value="Weekend only" {% if saved_batch == 'Weekend only' %}selected{% endif %}>Weekend only</option>
                    <option value="Flexible / self-paced" {% if saved_batch == 'Flexible / self-paced' %}selected{% endif %}>Flexible / self-paced</option>
                </select>
            </div>
            <div class="form-group">
                <label>Notes for our team (optional)</label>
                <textarea name="student_course_notes" id="course-notes" placeholder="Learning goals, prior experience, accessibility needs…">{{ saved_notes }}</textarea>
            </div>

            <button class="pay-btn" type="submit">Continue to UPI payment →</button>
        </form>

        <div class="secure"><span>✔</span> Payment QR is shown only on the next step</div>
    </div>
</div>
</body>
</html>
"""

# ── Step 2: amount + selected course + UPI QR only (no buttons). ──
checkout_pay_template = """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Pay – Nexora EduTech</title>
<link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap" rel="stylesheet">
<style>
*{box-sizing:border-box;margin:0;padding:0;}
body{font-family:'Nunito',sans-serif;background:linear-gradient(135deg,#000428,#004e92);
     min-height:100vh;display:flex;justify-content:center;align-items:center;padding:30px 16px;}

.wrap{background:white;border-radius:24px;max-width:400px;width:100%;
      padding:36px 28px 32px;text-align:center;
      box-shadow:0 24px 64px rgba(0,0,0,0.45);}
.amount{font-size:2.1em;font-weight:800;color:#004e92;margin-bottom:6px;}
.course{font-size:1.02em;font-weight:600;color:#333;line-height:1.4;margin-bottom:24px;
        padding:0 4px;}
.qr-box{display:inline-block;padding:12px;background:#f8fbff;border-radius:16px;
        border:1px solid #d6e8ff;}
.qr-box img{width:min(240px,72vw);height:auto;display:block;}
</style>
</head>
<body>
<div class="wrap">
    <div class="amount">₹{{amount}}</div>
    <div class="course">{{course_selected}}</div>
    {% if payment_qr %}
    <div class="qr-box">
        <img src="{{payment_qr}}" alt="Payment QR code">
    </div>
    {% endif %}
</div>
</body>
</html>
"""

# ── Payment Success ──
success_template = """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Payment Successful – Nexora EduTech</title>
<link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap" rel="stylesheet">
<style>
*{box-sizing:border-box;margin:0;padding:0;}
body{font-family:'Nunito',sans-serif;background:linear-gradient(135deg,#000428,#004e92);
     min-height:100vh;display:flex;justify-content:center;align-items:center;padding:20px;}
.card{background:white;border-radius:24px;max-width:520px;width:100%;padding:44px 40px;
      box-shadow:0 24px 64px rgba(0,0,0,0.45);text-align:center;}
.check{font-size:4.5em;margin-bottom:8px;animation:pop .5s cubic-bezier(.36,.07,.19,.97);}
@keyframes pop{0%{transform:scale(0)}70%{transform:scale(1.2)}100%{transform:scale(1)}}
h1{color:#1a1a2e;font-size:1.9em;margin-bottom:8px;}
.welcome{color:#0077b6;font-size:1.15em;font-weight:800;margin-bottom:24px;}
.info-box{background:#f0f7ff;border-radius:14px;padding:18px 22px;text-align:left;margin-bottom:28px;}
.info-row{display:flex;justify-content:space-between;padding:7px 0;
          border-bottom:1px solid #dde8f5;font-size:.92em;color:#444;}
.info-row:last-child{border-bottom:none;}
.info-row strong{color:#004e92;}
.go-btn{display:inline-block;background:linear-gradient(45deg,#00b4d8,#0077b6);color:white;
        padding:15px 36px;border-radius:14px;font-size:1.1em;font-weight:800;
        text-decoration:none;transition:opacity .2s,transform .15s;}
.go-btn:hover{opacity:.9;transform:translateY(-1px);}
</style>
</head>
<body>
<div class="card">
    <div class="check">✅</div>
    <h1>Payment Successful!</h1>
    <div class="welcome">Welcome aboard, {{student_name}}! 🎉</div>
    <div class="info-box">
        <div class="info-row"><span>Name</span><strong>{{student_name}}</strong></div>
        <div class="info-row"><span>Email</span><strong>{{student_email}}</strong></div>
        <div class="info-row"><span>Phone</span><strong>{{student_phone}}</strong></div>
        <div class="info-row"><span>Service</span><strong>{{service}}</strong></div>
        <div class="info-row"><span>Plan</span><strong>{{tier}}</strong></div>
        <div class="info-row"><span>Amount Paid</span><strong>₹{{amount}}</strong></div>
        {% if course_track %}<div class="info-row"><span>Course focus</span><strong>{{course_track}}</strong></div>{% endif %}
        {% if batch_pref %}<div class="info-row"><span>Batch preference</span><strong>{{batch_pref}}</strong></div>{% endif %}
        {% if course_notes %}<div class="info-row"><span>Your notes</span><strong>{{course_notes}}</strong></div>{% endif %}
        <div class="info-row"><span>Payment ID</span><strong>{{payment_id}}</strong></div>
        <div class="info-row"><span>Order ID</span><strong>{{order_id}}</strong></div>
    </div>
    <a href="/videos" class="go-btn">▶ Start Learning Now</a>
</div>
</body>
</html>
"""

# ── Video Portal ──
videos_template = """
<!DOCTYPE html>
<html>
<head>
<title>My Videos – Nexora EduTech</title>
<link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800&display=swap" rel="stylesheet">
<style>
*{box-sizing:border-box;margin:0;padding:0;}
body{font-family:'Nunito',sans-serif;background:linear-gradient(135deg,#000428,#004e92);min-height:100vh;color:white;}

.topbar{background:rgba(0,0,0,0.45);padding:15px 32px;display:flex;justify-content:space-between;
        align-items:center;backdrop-filter:blur(10px);position:sticky;top:0;z-index:10;}
.topbar .logo{font-size:1.25em;font-weight:800;color:#00c6ff;}
.welcome-name{color:#ffd700;font-weight:700;}
.badge-tier{background:linear-gradient(45deg,#ff512f,#dd2476);padding:4px 12px;
            border-radius:30px;font-size:.82em;font-weight:700;margin-left:8px;}

.hero{padding:44px 32px 20px;text-align:center;}
.hero h1{font-size:2.2em;font-weight:800;margin-bottom:8px;}
.hero p{color:#a0c4ff;}

.player-section{max-width:860px;margin:0 auto 36px;padding:0 20px;}
.player-wrap{background:rgba(0,0,0,0.5);border-radius:18px;overflow:hidden;box-shadow:0 12px 40px rgba(0,0,0,0.5);}
.player-wrap iframe{width:100%;height:440px;border:none;display:block;}
.player-info{padding:16px 22px;}
.player-info h2{font-size:1.25em;color:#ffd700;margin-bottom:4px;}
.player-info span{color:#a0c4ff;font-size:.88em;}

.grid-title{text-align:center;font-size:1.35em;font-weight:800;margin:8px 0 20px;color:#ffd700;}
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(255px,1fr));
      gap:20px;max-width:1100px;margin:0 auto 60px;padding:0 20px;}
.vid-card{background:rgba(255,255,255,0.08);border-radius:14px;overflow:hidden;
          cursor:pointer;transition:transform .2s,box-shadow .2s;border:2px solid transparent;}
.vid-card:hover{transform:translateY(-4px);box-shadow:0 8px 24px rgba(0,0,0,0.4);}
.vid-card.active{border-color:#ffd700;}
.vid-card img{width:100%;height:150px;object-fit:cover;}
.vid-card .meta{padding:12px;}
.vid-card .meta h3{font-size:.95em;margin-bottom:6px;color:#e0f0ff;}
.duration-badge{display:inline-block;background:rgba(0,0,0,0.55);color:#fff;
                font-size:.75em;padding:3px 9px;border-radius:5px;}
</style>
</head>
<body>

<div class="topbar">
    <div class="logo">🎓 Nexora EduTech</div>
    <div>
        <span class="welcome-name">👋 {{student_name}}</span>
        <span> · {{service}}</span>
        <span class="badge-tier">{{tier}}</span>
    </div>
</div>

<div class="hero">
    <h1>My Learning Portal</h1>
    <p>Welcome back, <strong>{{student_name}}</strong>! You have <strong>{{count}}</strong> videos on your <strong>{{tier}}</strong> plan.</p>
</div>

<div class="player-section">
    <div class="player-wrap">
        <iframe id="mainPlayer" src="{{first_embed}}" allowfullscreen
                allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"></iframe>
        <div class="player-info">
            <h2 id="nowTitle">{{first_title}}</h2>
            <span id="nowDuration">⏱ {{first_duration}}</span>
        </div>
    </div>
</div>

<div class="grid-title">📚 All Videos</div>
<div class="grid">
    {% for v in videos %}
    <div class="vid-card {% if loop.first %}active{% endif %}" id="card-{{v.id}}"
         onclick="playVideo('{{v.id}}','{{v.embed}}','{{v.title}}','{{v.duration}}')">
        <img src="{{v.thumb}}" alt="{{v.title}}">
        <div class="meta">
            <h3>{{v.title}}</h3>
            <div class="duration-badge">⏱ {{v.duration}}</div>
        </div>
    </div>
    {% endfor %}
</div>

<script>
function playVideo(id,embed,title,duration){
    document.getElementById('mainPlayer').src=embed;
    document.getElementById('nowTitle').textContent=title;
    document.getElementById('nowDuration').textContent='⏱ '+duration;
    document.querySelectorAll('.vid-card').forEach(c=>c.classList.remove('active'));
    document.getElementById('card-'+id).classList.add('active');
    window.scrollTo({top:0,behavior:'smooth'});
}
</script>
</body>
</html>
"""

faculty_page = """
<!DOCTYPE html>
<html>
<head>
<title>Our Faculty</title>
<style>
body{font-family:Segoe UI;background:linear-gradient(135deg,#000428,#004e92);color:white;text-align:center;}
.card{background:rgba(255,255,255,0.1);display:inline-block;width:260px;
      padding:20px;margin:20px;border-radius:15px;box-shadow:0 4px 10px rgba(0,0,0,0.5);}
</style>
</head>
<body>
<h1 style="margin-top:40px;">Meet Our Faculty</h1>
<div class="card"><h2>Dr.ARUN K</h2><p>AI &amp; Machine Learning Expert</p></div>
<div class="card"><h2>Prof. Rajesh Kumar</h2><p>Cloud Computing Specialist</p></div>
<div class="card"><h2>Dr. Meera Iyer</h2><p>Education Psychology Expert</p></div>
<div class="card"><h2>Prof. Arvind Menon</h2><p>EdTech Strategy Innovator</p></div>
</body>
</html>
"""


@app.route("/")
def home():
    return render_template_string(home_page)

@app.route("/services")
def services():
    return render_template_string(services_page)

@app.route("/services/<name>")
def service_detail(name):
    if name not in services_data:
        return "Service not found", 404
    d = services_data[name]
    return render_template_string(detail_template,
        title      = d["title"],
        points     = d["points"],
        hero_image = d["hero_image"],
        gallery    = d["gallery"],
        price      = int(d["price"]),
        standard   = int(d["standard"]),
        premium    = int(d["premium"]))

@app.route("/checkout", methods=["POST"])
def checkout():
    service  = request.form.get("service")
    tier     = request.form.get("tier")
    amount   = request.form.get("amount")
    order_id = "VE-" + uuid.uuid4().hex[:8].upper()
    session["pending_checkout"] = {
        "order_id": order_id,
        "service": service or "",
        "tier": tier or "",
        "amount": amount or "",
    }
    session.pop("checkout_student", None)
    session.pop("checkout_details_done", None)
    session.pop("rzp_order_id", None)
    return redirect(url_for("checkout_details"))


def _render_checkout_details(pending, err=None, **saved):
    defaults = dict(
        saved_name="",
        saved_email="",
        saved_phone="",
        saved_course="",
        saved_batch="",
        saved_notes="",
    )
    defaults.update(saved)
    return render_template_string(
        checkout_details_template,
        service=pending["service"],
        tier=pending["tier"],
        amount=pending["amount"],
        order_id=pending["order_id"],
        course_choices=list_checkout_courses(),
        err=err,
        **defaults,
    )


@app.route("/checkout/details", methods=["GET", "POST"])
def checkout_details():
    pending = session.get("pending_checkout")
    if not pending:
        return redirect(url_for("services"))

    if request.method == "GET":
        prev = session.get("checkout_student") or {}
        return _render_checkout_details(
            pending,
            err=None,
            saved_name=prev.get("student_name", ""),
            saved_email=prev.get("student_email", ""),
            saved_phone=prev.get("student_phone", ""),
            saved_course=prev.get("student_course_track", ""),
            saved_batch=prev.get("student_batch_pref", ""),
            saved_notes=prev.get("student_course_notes", ""),
        )

    name   = request.form.get("student_name", "").strip()
    email  = request.form.get("student_email", "").strip()
    phone  = request.form.get("student_phone", "").strip()
    course = request.form.get("student_course_track", "").strip()
    batch  = request.form.get("student_batch_pref", "").strip()
    notes  = request.form.get("student_course_notes", "").strip()

    errs = []
    if not name:
        errs.append("Please enter your full name.")
    if not email or "@" not in email:
        errs.append("Enter a valid email.")
    if not phone or not phone.isdigit() or len(phone) != 10:
        errs.append("Enter a valid 10-digit mobile number.")
    if not course:
        errs.append("Please select a course.")
    if errs:
        return _render_checkout_details(
            pending,
            err=" ".join(errs),
            saved_name=name,
            saved_email=email,
            saved_phone=phone,
            saved_course=course,
            saved_batch=batch,
            saved_notes=notes,
        )

    session["checkout_student"] = {
        "student_name": name,
        "student_email": email,
        "student_phone": phone,
        "student_course_track": course,
        "student_batch_pref": batch,
        "student_course_notes": notes,
    }
    session["checkout_details_done"] = pending["order_id"]
    return redirect(url_for("checkout_pay"))


@app.route("/checkout/pay")
def checkout_pay():
    pending = session.get("pending_checkout")
    if not pending:
        return redirect(url_for("services"))
    if session.get("checkout_details_done") != pending.get("order_id"):
        return redirect(url_for("services"))
    if not session.get("checkout_student"):
        return redirect(url_for("checkout_details"))

    oid        = pending["order_id"]
    stu        = session["checkout_student"]
    upi_uri    = build_upi_uri(pending["amount"], pending["service"], pending["tier"], oid)
    payment_qr = qr_svg_data_uri(upi_uri)
    session.pop("rzp_order_id", None)

    return render_template_string(
        checkout_pay_template,
        amount=pending["amount"],
        course_selected=stu.get("student_course_track", ""),
        payment_qr=payment_qr,
    )

@app.route("/payment-success", methods=["POST"])
def payment_success():
    pending    = session.get("pending_checkout")
    done       = session.get("checkout_details_done")
    posted_oid = request.form.get("internal_order_id", "")
    student    = session.get("checkout_student")
    if (
        not pending
        or not student
        or done != pending.get("order_id")
        or posted_oid != pending.get("order_id")
    ):
        return redirect(url_for("services"))

    rzp_pay_id = request.form.get("razorpay_payment_id", "").strip()
    rzp_oid    = request.form.get("razorpay_order_id", "").strip()
    rzp_sig    = request.form.get("razorpay_signature", "").strip()

    payment_id = None
    if not (rzp_pay_id and rzp_oid and rzp_sig):
        return redirect(url_for("checkout_pay"))
    if rzp_oid != session.get("rzp_order_id"):
        return redirect(url_for("services"))
    if not verify_razorpay_signature(rzp_oid, rzp_pay_id, rzp_sig):
        return redirect(url_for("checkout_pay"))
    payment_id = rzp_pay_id
    order_id   = posted_oid
    service    = pending["service"]
    tier       = pending["tier"]
    amount     = pending["amount"]

    student_name  = student.get("student_name", "Student")
    student_email = student.get("student_email", "")
    student_phone = student.get("student_phone", "")
    course_track  = (student.get("student_course_track") or "").strip()
    batch_pref    = (student.get("student_batch_pref") or "").strip()
    course_notes  = (student.get("student_course_notes") or "").strip()

    session["subscription"] = {
        "service": service,
        "tier": tier,
        "amount": amount,
        "name": student_name,
        "email": student_email,
        "phone": student_phone,
        "course_track": course_track,
        "batch_pref": batch_pref,
        "course_notes": course_notes,
    }
    session.pop("pending_checkout", None)
    session.pop("checkout_details_done", None)
    session.pop("checkout_student", None)
    session.pop("rzp_order_id", None)

    return render_template_string(
        success_template,
        service=service,
        tier=tier,
        amount=amount,
        payment_id=payment_id,
        order_id=order_id,
        student_name=student_name,
        student_email=student_email,
        student_phone=student_phone,
        course_track=course_track,
        batch_pref=batch_pref,
        course_notes=course_notes,
    )

@app.route("/videos")
def videos():
    sub = session.get("subscription")
    if not sub:
        return redirect(url_for("services"))

    service_title = sub["service"]
    tier          = sub["tier"]
    student_name  = sub.get("name", "Student")
    video_list    = get_accessible_videos(service_title, tier)

    if not video_list:
        return "<h2 style='color:white;text-align:center;margin-top:80px'>No videos found for your plan.</h2>"

    first = video_list[0]
    return render_template_string(videos_template,
        service=service_title, tier=tier,
        student_name=student_name,
        videos=video_list, count=len(video_list),
        first_embed    = first["embed"],
        first_title    = first["title"],
        first_duration = first["duration"])

@app.route("/faculty")
def faculty():
    return render_template_string(faculty_page)

@app.route("/courses")
def courses():
    return render_template_string(courses_page)

if __name__ == "__main__":
    app.run(debug=True)
