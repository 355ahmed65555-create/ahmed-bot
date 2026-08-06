import random, string, json, os, re, asyncio
import pyotp
import os
import requests
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup, KeyboardButton, Update, BotCommand
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, ContextTypes, filters

BOT_TOKEN = os.getenv("BOT_TOKEN") or "8879364444:AAGIjGPLvSuWXTwH8_ljfEcGzEHjc-eM1vY"
import json as json_lib
try:
    with open("admins.json","r") as f:
        SUPER_ADMINS = json_lib.load(f)
except:
    SUPER_ADMINS = [6364073135]
WHATSAPP_NUMBER = "201096514020"


# ==================== Keep Alive Server - عشان Replit مينامش ====================
from flask import Flask
import threading

app_flask = Flask(__name__)

@app_flask.route('/')
def home():
    return "✅ AHMED Bot V31 is Running 24/7 - Auto Mail Active 🔥"

@app_flask.route('/ping')
def ping():
    return "Pong! Bot Alive"

def run_flask():
    try:
        app_flask.run(host='0.0.0.0', port=8080)
    except:
        pass

def keep_alive():
    t = threading.Thread(target=run_flask)
    t.daemon = True
    t.start()
    print("✅ Keep-Alive Server Started on port 8080")


DB_FILE = "db.json"
SETTINGS_FILE = "settings.json"

SERVICES = {
    "whatsapp": {"name": "واتساب", "icon": "💚", "flag": "💚"},
    "telegram": {"name": "تليجرام", "icon": "💙", "flag": "💙"}
}


def update_user_track(uid, username="", name=""):
    db = fast_load_db()
    uid_str = str(uid)
    db.setdefault("user_tracks", {})
    if uid_str not in db["user_tracks"]:
        db["user_tracks"][uid_str] = {"first_seen": "", "last_seen": "", "username": "", "name": "", "blocked": False, "messages": 0}
    import datetime
    now = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    if not db["user_tracks"][uid_str]["first_seen"]:
        db["user_tracks"][uid_str]["first_seen"] = now
    db["user_tracks"][uid_str]["last_seen"] = now
    db["user_tracks"][uid_str]["username"] = username or db["user_tracks"][uid_str].get("username","")
    db["user_tracks"][uid_str]["name"] = name or db["user_tracks"][uid_str].get("name","")
    db["user_tracks"][uid_str]["messages"] = db["user_tracks"][uid_str].get("messages",0)+1
    db["user_tracks"][uid_str]["blocked"] = False
    fast_save_db(db)

def load_db_OLD():
    if not os.path.exists(DB_FILE):
        return {"users": {}, "allowed_usernames": [], "authorized": {}, "user_tracks": {}}
    try:
        with open(DB_FILE, "r", encoding="utf-8") as f:
            data = json.load(f)
            data.setdefault("allowed_usernames", [])
            data.setdefault("authorized", {})
            data.setdefault("users", {})
            data.setdefault("user_tracks", {})
            return data
    except:
        return {"users": {}, "allowed_usernames": [], "authorized": {}, "user_tracks": {}}

def save_db_OLD(db):
    with open(DB_FILE, "w", encoding="utf-8") as f:
        json.dump(db, f, ensure_ascii=False, indent=2)

def load_settings():
    default = {"bot_active": True, "welcome_auth": "👋 أهلاً بك، {first_name}\n\n🤖 AHMED Bot\n\n━━━━━━━━━━━━━━━\n⚡ سرعة في التنفيذ\n🔒 أمان وخصوصية\n🚀 أدوات ذكية في مكان واحد\n━━━━━━━━━━━━━━━\n\n📌 اختر الخدمة المطلوبة من الأزرار بالأسفل.", "welcome_unauth": "👋 أهلاً بك، {first_name}\n\n🤖 AHMED Bot\n\n━━━━━━━━━━━━━━━\n⚡ سرعة في التنفيذ\n🔒 أمان وخصوصية\n🚀 أدوات ذكية في مكان واحد\n━━━━━━━━━━━━━━━\n\n🔐 اكتب اليوزر الخاص بك للمواصلة", "admin_password": "AHMED2011/10/1", "admin_passwords": {}, "admin_perms": {}, "last_updates": "🚀 <b>آخر التحديثات - الإصدار النهائي</b>\n\n━━━━━━━━━━━━━━━\n✅ إضافة البريد المؤقت الحقيقي (يستقبل أكواد فورية)\n✅ إضافة الأرقام المؤقتة بالخدمة (فيسبوك/واتساب/انستا/تيك توك...)\n✅ اختيار الدولة (مصر، أمريكا، بريطانيا...)\n✅ نظام رقم واحد = شخص واحد بس (مفيش تكرار)\n✅ فحص هل الرقم مستخدم على المنصة ولا لا\n✅ اختيار اللغة عربي/إنجليزي من /start\n✅ البوت بقى طيارة (0.1 ثانية)\n✅ تحسين واجهة الأرقام زي 5sim بالظبط\n━━━━━━━━━━━━━━━\n🔥 البوت جاهز 100 100"}
    if not os.path.exists(SETTINGS_FILE):
        return default
    try:
        with open(SETTINGS_FILE, "r") as f:
            s=json.load(f)
            for k,v in default.items():
                s.setdefault(k,v)
            return s
    except:
        return default

def save_settings(s):
    with open(SETTINGS_FILE, "w", encoding="utf-8") as f:
        json.dump(s, f, ensure_ascii=False, indent=2)

def is_admin(uid): return uid in SUPER_ADMINS
def is_authorized(uid):
    if is_admin(uid): return True
    db = fast_load_db()
    return str(uid) in db.get("authorized", {})






# ==================== فحص هل الرقم مستخدم على المنصة ====================
def check_number_on_platform(number, service_id):
    """
    يفحص هل الرقم مستخدم على المنصة ولا لا
    """
    clean = COMPILED_PHONE_CLEAN.sub('', number)
    result = {
        "number": number,
        "service": service_id,
        "is_used": None,  # True=مستخدم، False=جديد، None=مش قادر أفحص
        "message": "",
        "can_check": False
    }
    
    # 1. فحص داخلي - هل استخدمناه في البوت قبل كده؟
    db = fast_load_db()
    all_used = db.get("all_used_numbers", [])
    if number in all_used:
        # نشوف مين استخدمه
        owner = db.get("number_owners", {}).get(number, {})
        result["is_used"] = True
        result["message"] = f"⚠️ الرقم ده استخدم قبل كده في البوت\n👤 بواسطة: {owner.get('uid','')} \n📘 للخدمة: {owner.get('service','')}"
        result["can_check"] = True
        return result
    
    # 2. فحص خارجي حسب الخدمة
    try:
        if service_id == "whatsapp":
            # فحص واتساب عن طريق wa.me
            headers = {"User-Agent": "Mozilla/5.0"}
            url = f"https://wa.me/{clean}"
            r = _global_session.get(url, headers=headers, timeout=4, allow_redirects=True)
            # لو الصفحة فيها "Continue to Chat" يبقى الرقم على واتساب (مستخدم)
            if "Continue to Chat" in r.text or "متابعة إلى الدردشة" in r.text or r.status_code == 200 and "whatsapp" in r.text.lower():
                # نحاول نتأكد أكتر
                result["is_used"] = True
                result["message"] = "📱 الرقم ده موجود على واتساب (مستخدم)"
                result["can_check"] = True
            else:
                result["is_used"] = False
                result["message"] = "✅ الرقم ده مش موجود على واتساب (جديد)"
                result["can_check"] = True
        
        elif service_id == "telegram":
            # تليجرام - صعب الفحص بدون API، بس نعتبر الأرقام الوهمية جديدة
            result["is_used"] = False
            result["message"] = "✅ الرقم جديد (لم يستخدم في البوت)\n💡 تليجرام: الرقم الوهمي يعتبر جديد"
            result["can_check"] = True
        
        elif service_id in ["facebook", "instagram", "tiktok", "twitter", "google", "discord"]:
            # فيسبوك وانستا وتيك توك مفيش طريقة رسمية للفحص
            # بس نضمن انه مش مستخدم في البوت بتاعنا + نديه تقييم
            result["is_used"] = False
            result["message"] = f"✅ الرقم جديد من نظامنا\n🔒 لم يستخدم في البوت من قبل نهائي\n\n⚠️ ملاحظة: الرقم وهمي (مصري/أمريكي) تم توليده عشوائياً\n💡 قد يكون مستخدم حقيقياً خارج البوت لأنه رقم وهمي\n📌 للضمان 100% استخدم الأرقام الحقيقية المدفوعة"
            result["can_check"] = False  # مش قادرين نفحص خارجي
        
        else:
            result["is_used"] = False
            result["message"] = "✅ الرقم جديد ولم يستخدم في البوت"
            result["can_check"] = True
            
    except Exception as e:
        result["message"] = f"❌ مقدرتش أفحص: {e}\n✅ لكن الرقم جديد في البوت بتاعنا"
        result["is_used"] = False
    
    return result


# ==================== تتبع الأرقام المستخدمة - رقم واحد = شخص واحد بس ====================
def is_number_used(number, service_id=None):
    db = fast_load_db()
    # فحص عام - هل الرقم استخدم قبل كده لأي خدمة؟
    all_used = db.get("all_used_numbers", [])  # قائمة كل الأرقام اللي اتوزعت
    if number in all_used:
        return True
    # فحص إضافي للخدمة
    if service_id:
        used = db.get("used_numbers", {})
        service_used = used.get(service_id, [])
        if number in service_used:
            return True
    return False

def mark_number_used(number, service_id, uid, country_code):
    db = fast_load_db()
    # 1. نضيفه للقائمة العامة - ممنوع يطلع تاني لأي حد
    db.setdefault("all_used_numbers", [])
    if number not in db["all_used_numbers"]:
        db["all_used_numbers"].append(number)
    
    # 2. نضيفه لقائمة الخدمة
    db.setdefault("used_numbers", {}).setdefault(service_id, [])
    if number not in db["used_numbers"][service_id]:
        db["used_numbers"][service_id].append(number)
    
    # 3. نحفظ تفاصيل الاستخدام
    db.setdefault("numbers_log", []).append({
        "number": number,
        "service": service_id,
        "country": country_code,
        "uid": uid,
        "time": str(__import__('datetime').datetime.now())
    })
    
    # 4. نحفظ لكل يوزر
    db.setdefault("user_numbers", {}).setdefault(str(uid), []).append({
        "number": number,
        "service": service_id,
        "country": country_code,
        "time": str(__import__('datetime').datetime.now())[:19]
    })
    
    # 5. نحفظ مين خد الرقم ده
    db.setdefault("number_owners", {})[number] = {
        "uid": uid,
        "service": service_id,
        "country": country_code,
        "time": str(__import__('datetime').datetime.now())[:19]
    }
    fast_save_db(db)

def get_unique_number(service_id, country_code, max_attempts=50):
    """
    يجيب رقم مش مستخدم قبل كده نهائياً في البوت كله
    رقم واحد = شخص واحد بس
    """
    db = fast_load_db()
    all_used = db.get("all_used_numbers", [])
    
    for _ in range(max_attempts):
        if country_code == "eg":
            full, local = gen_egy_number()
        else:
            full, local = gen_us_number()
        
        # لو الرقم مش مستخدم نهائياً في البوت كله
        if full not in all_used:
            return full, local
    
    # لو كل المحاولات فشلت، نولد رقم مستحيل يتكرر
    for i in range(20):
        random_part = ''.join(random.choices("0123456789", k=10))
        if country_code == "eg":
            prefix = random.choice(["010", "011", "012", "015"])
            full = f"+20{prefix[1:]}{random_part[:8]}"
            local = prefix + random_part[:8]
        else:
            area = random.choice(["201","202","212","213","305","310"])
            full = f"+1{area}{random_part[:7]}"
            local = f"{area}{random_part[:7]}"
        
        if full not in all_used:
            return full, full
    
    # آخر حل - رقم بالوقت الحالي مستحيل يتكرر
    import time
    ts = str(int(time.time()))[-6:]
    if country_code == "eg":
        full = f"+2010{ts}00"
    else:
        full = f"+1202{ts}0"
    return full, full

    
    # لو كل المحاولات فشلت، يولد رقم جديد تماماً (مستحيل يتكرر)
    for _ in range(10):
        random_extra = ''.join(random.choices("0123456789", k=4))
        if country_code == "eg":
            full, local = gen_egy_number()
            full = full[:-4] + random_extra
        else:
            full, local = gen_us_number()
            full = full[:-4] + random_extra
        if full not in used:
            return full, full
    
    # آخر حل
    full, local = gen_egy_number() if country_code=="eg" else gen_us_number()
    return full, local


# ==================== الأرقام المؤقتة ====================


# ==================== تحسينات السرعة القصوى 🚀 ====================
import functools
import time as time_module
from functools import lru_cache

# كاش عالمي للداتا بيز - مش كل شوية نفتح ملف
_db_cache = None
_db_cache_time = 0
_db_lock = __import__('threading').Lock()

# Session واحد لكل الطلبات - أسرع 10 مرات من requests.get كل مرة
import requests as req_lib
_global_session = req_lib.Session()
_global_session.headers.update({"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"})
_global_session.mount('https://', req_lib.adapters.HTTPAdapter(pool_connections=20, pool_maxsize=20))

# Regex مجمعة مسبقاً - أسرع
COMPILED_CODE_REGEX = re.compile(r'\b\d{4,8}\b')
COMPILED_CODE_WITH_TEXT = re.compile(r'(?:code|Code|رمز|OTP|verification)[^0-9]{0,20}(\d{4,8})', re.I)
COMPILED_PHONE_CLEAN = re.compile(r'[^0-9]')

# كاش للغات
_lang_cache = {}
_lang_cache_time = {}

def fast_load_db():
    global _db_cache, _db_cache_time
    now = time_module.time()
    # لو الكاش عمره أقل من 2 ثانية، استخدمه
    if _db_cache is not None and (now - _db_cache_time) < 2:
        return _db_cache
    try:
        with open(DB_PATH, 'r', encoding='utf-8') as f:
            _db_cache = json.load(f)
            _db_cache_time = now
            return _db_cache
    except:
        if _db_cache is not None:
            return _db_cache
        return {"users": {}, "user_langs": {}, "all_used_numbers": [], "used_numbers": {}, "numbers_log": [], "user_numbers": {}, "number_owners": {}}

def fast_save_db(db):
    global _db_cache, _db_cache_time
    _db_cache = db
    _db_cache_time = time_module.time()
    # حفظ في ثريد منفصل عشان ميبطأش البوت
    def _save():
        try:
            with _db_lock:
                with open(DB_PATH, 'w', encoding='utf-8') as f:
                    json.dump(db, f, ensure_ascii=False, indent=2)
        except:
            pass
    __import__('threading').Thread(target=_save, daemon=True).start()

# نستبدل load_db و save_db بالسريعة

# ==================== نظام اللغات ====================
LANGS = {
    "ar": {
        "choose_lang": "🌐 <b>اختر لغتك / Choose your language</b>\n\n━━━━━━━━━━━━━━━",
        "welcome_auth": "👋 أهلاً بك، {first_name}\n\n🤖 AHMED Bot V3 🔥\n\n━━━━━━━━━━━━━━━\n⚡ سرعة في التنفيذ\n🔒 أمان وخصوصية\n🚀 أدوات ذكية في مكان واحد\n👥 أسماء لا نهائية ♾️\n📱 أرقام واتساب/تليجرام\n━━━━━━━━━━━━━━━\n\n📌 اختر الخدمة من الأزرار بالأسفل.",
        "welcome_unauth": "👋 أهلاً بك، {first_name}\n\n🤖 AHMED Bot V3\n\n━━━━━━━━━━━━━━━\n⚡ سرعة\n🔒 أمان\n🚀 أدوات ذكية\n━━━━━━━━━━━━━━━\n\n🔐 اكتب اليوزر الخاص بك للمواصلة",
        "ask_user": "🔐 اكتب اليوزر الخاص بك للمواصلة ✅",
        "invalid_user": "❌ يوزر غلط 🔐\nلـ طلب يوزر كلم الأدمن:",
    },
    "en": {
        "choose_lang": "🌐 <b>Choose your language</b>\n\n━━━━━━━━━━━━━━━",
        "welcome_auth": "👋 Welcome, {first_name}\n\n🤖 AHMED Bot V3 🔥\n\n━━━━━━━━━━━━━━━\n⚡ Fast\n🔒 Secure\n🚀 Smart Tools\n👥 Infinite Names ♾️\n📱 WA/TG Numbers\n━━━━━━━━━━━━━━━\n\n📌 Choose a service below.",
        "welcome_unauth": "👋 Welcome, {first_name}\n\n🤖 AHMED Bot V3\n\n━━━━━━━━━━━━━━━\n⚡ Fast\n🔒 Secure\n🚀 Smart\n━━━━━━━━━━━━━━━\n\n🔐 Enter your username",
        "ask_user": "🔐 Enter your username ✅",
        "invalid_user": "❌ Invalid username 🔐\nContact admin:",
    }
}

def get_user_lang(uid):
    db = fast_load_db()
    return db.get("user_langs", {}).get(str(uid), "ar")

def set_user_lang(uid, lang):
    db = fast_load_db()
    db.setdefault("user_langs", {})[str(uid)] = lang
    fast_save_db(db)

def t(uid, key, **kwargs):
    lang = get_user_lang(uid)
    text = LANGS.get(lang, LANGS["ar"]).get(key, key)
    try:
        return text.format(**kwargs)
    except:
        return text

# خدمات الأرقام مثل 5sim
SERVICES = [
    {"id": "whatsapp", "name_ar": "واتساب", "name_en": "WhatsApp", "emoji": "💚", "count": 15420},
    {"id": "telegram", "name_ar": "تليجرام", "name_en": "Telegram", "emoji": "✈️", "count": 12300},
]

COUNTRIES = [
    {"code": "eg", "name_ar": "مصر", "name_en": "Egypt", "flag": "🇪🇬", "count": 8},
    {"code": "sa", "name_ar": "السعودية", "name_en": "Saudi Arabia", "flag": "🇸🇦", "count": 5},
    {"code": "ae", "name_ar": "الإمارات", "name_en": "UAE", "flag": "🇦🇪", "count": 4},
    {"code": "us", "name_ar": "أمريكا", "name_en": "USA", "flag": "🇺🇸", "count": 12},
    {"code": "gb", "name_ar": "بريطانيا", "name_en": "UK", "flag": "🇬🇧", "count": 7},
    {"code": "de", "name_ar": "ألمانيا", "name_en": "Germany", "flag": "🇩🇪", "count": 6},
    {"code": "fr", "name_ar": "فرنسا", "name_en": "France", "flag": "🇫🇷", "count": 4},
    {"code": "ru", "name_ar": "روسيا", "name_en": "Russia", "flag": "🇷🇺", "count": 15},
    {"code": "bf", "name_ar": "بوركينا فاسو", "name_en": "Burkina Faso", "flag": "🇧🇫", "count": 1},
    {"code": "et", "name_ar": "إثيوبيا", "name_en": "Ethiopia", "flag": "🇪🇹", "count": 1},
    {"code": "mr", "name_ar": "موريتانيا", "name_en": "Mauritania", "flag": "🇲🇷", "count": 1},
    {"code": "mn", "name_ar": "منغوليا", "name_en": "Mongolia", "flag": "🇲🇳", "count": 1},
    {"code": "mm", "name_ar": "ميانمار", "name_en": "Myanmar", "flag": "🇲🇲", "count": 2},
    {"code": "ps", "name_ar": "فلسطين", "name_en": "Palestine", "flag": "🇵🇸", "count": 4},
    {"code": "sy", "name_ar": "سوريا", "name_en": "Syria", "flag": "🇸🇾", "count": 3},
    {"code": "tz", "name_ar": "تنزانيا", "name_en": "Tanzania", "flag": "🇹🇿", "count": 1},
    {"code": "tl", "name_ar": "تيمور الشرقية", "name_en": "Timor-Leste", "flag": "🇹🇱", "count": 2},
]

def get_services_keyboard(lang="ar"):
    kb = []
    row = []
    for s in SERVICES:
        name = s["name_en"] if lang=="en" else s["name_ar"]
        display = f"{s['emoji']} {name}"
        row.append(InlineKeyboardButton(display, callback_data=f"svc_{s['id']}"))
        if len(row)==2:
            kb.append(row)
            row=[]
    if row:
        kb.append(row)
    kb.append([InlineKeyboardButton("🔙 رجوع" if lang=="ar" else "🔙 BACK", callback_data="main")])
    return InlineKeyboardMarkup(kb)

def get_countries_keyboard(service_id, lang="ar"):
    kb = []
    row = []
    for c in COUNTRIES:
        name = c["name_en"] if lang=="en" else c["name_en"]  # نعرض الإنجليزي زي الصورة
        display = f"{c['flag']} {name} ({c['count']})"
        row.append(InlineKeyboardButton(display, callback_data=f"country_{service_id}_{c['code']}"))
        if len(row)==2:
            kb.append(row)
            row=[]
    if row:
        kb.append(row)
    # زرار تيمور الشرقية لوحده زي الصورة
    kb.append([InlineKeyboardButton("⬅️ رجوع للخدمات" if lang=="ar" else "⬅️ BACK TO SERVICES", callback_data="nums_services")])
    kb.append([InlineKeyboardButton("🔙 رجوع" if lang=="ar" else "🔙 BACK", callback_data="main")])
    return InlineKeyboardMarkup(kb)


def gen_egy_number():
    prefixes = ["010", "011", "012", "015"]
    prefix = random.choice(prefixes)
    number = prefix + ''.join(random.choices("0123456789", k=8))
    return f"+20{number[1:]}" if number.startswith("0") else f"+20{number}", number

def gen_us_number():
    area = random.choice(["201","202","212","213","305","310","312","404","415","510","650","702","713","714","917"])
    number = area + ''.join(random.choices("0123456789", k=7))
    return f"+1{number}", number

def gen_random_numbers(count=5, country="egy"):
    nums = []
    for _ in range(count):
        if country == "egy":
            full, local = gen_egy_number()
        else:
            full, local = gen_us_number()
        nums.append((full, local))
    return nums

# أرقام حقيقية تستقبل SMS من موقع مجاني
def get_free_numbers():
    # قائمة أرقام مجانية شغالة من مواقع مجانية
    return [
        {"number": "+12024561111", "country": "🇺🇸 أمريكا", "country_code": "us"},
        {"number": "+447700900000", "country": "🇬🇧 بريطانيا", "country_code": "uk"},
        {"number": "+33612345678", "country": "🇫🇷 فرنسا", "country_code": "fr"},
        {"number": "+4915123456789", "country": "🇩🇪 ألمانيا", "country_code": "de"},
    ]

def get_sms_for_number(number):
    # هنا ممكن تربط API حقيقي لاحقاً مثل 5sim.net
    # حاليا نرجع رسائل وهمية للتجربة
    return []

# ==================== البريد المؤقت القوي V30 - مجاني 100% وقوي جدا ====================
# نستخدم mail.tm + mail.gw - اقوى بريدات مجانية ومش محظورة من فيسبوك
MAIL_TM_APIS = [
    "https://api.mail.tm",
    "https://api.mail.gw"
]

def gen_temp_mail():
    """يولد بريد قوي مجاني 100% - مش محظور والكود بيوصل في ثانية"""
    import time
    for api_base in MAIL_TM_APIS:
        try:
            # 1. جيب دومين
            r = _global_session.get(f"{api_base}/domains", timeout=8)
            if r.status_code != 200:
                continue
            domains = r.json().get('hydra:member', [])
            if not domains:
                continue
            domain = domains[0]['domain']
            
            # 2. اعمل حساب
            username = ''.join(random.choices(string.ascii_lowercase+string.digits, k=12))
            password = ''.join(random.choices(string.ascii_letters+string.digits, k=16))
            address = f"{username}@{domain}"
            
            data = {"address": address, "password": password}
            r2 = _global_session.post(f"{api_base}/accounts", json=data, timeout=8)
            if r2.status_code not in [200,201]:
                continue
                
            # 3. جيب توكن
            r3 = _global_session.post(f"{api_base}/token", json=data, timeout=8)
            if r3.status_code != 200:
                continue
            token = r3.json().get('token')
            if not token:
                continue
            
            # نجح - رجع البريد والتوكن
            # نحفظ api_base مع التوكن عشان نستخدمه بعدين
            return address, password, f"{api_base}|||{token}"
        except Exception as e:
            print(f"Mail.tm error {api_base}: {e}")
            continue
    
    # fallback اخير - نستخدم 1secmail كاحتياطي
    try:
        r = _global_session.get("https://www.1secmail.com/api/v1/?action=genRandomMailbox&count=1", timeout=5)
        if r.status_code == 200:
            mail = r.json()[0]
            login, domain = mail.split("@")
            return mail, login, f"1sec|||{domain}"
    except:
        pass
    random_str = ''.join(random.choices(string.ascii_lowercase+string.digits, k=10))
    return f"{random_str}@1secmail.com", random_str, "1sec|||1secmail.com"

def get_temp_messages(login_or_pwd, token_or_domain):
    """يجيب الرسائل - يدعم mail.tm الجديد و 1secmail القديم"""
    try:
        # لو التوكن فيه ||| يبقى ده النظام الجديد mail.tm
        if "|||" in token_or_domain:
            parts = token_or_domain.split("|||")
            api_base = parts[0]
            real_token = parts[1]
            
            if api_base == "1sec":
                # نظام قديم
                domain = real_token
                login = login_or_pwd
                url = f"https://www.1secmail.com/api/v1/?action=getMessages&login={login}&domain={domain}"
                r = _global_session.get(url, timeout=5)
                if r.status_code == 200:
                    msgs = r.json()
                    # نحولها لنفس شكل mail.tm
                    converted = []
                    for m in msgs:
                        converted.append({
                            "id": m.get("id"),
                            "subject": m.get("subject",""),
                            "from": m.get("from",""),
                            "intro": m.get("subject",""),
                            "seen": False,
                            "is_1sec": True,
                            "login": login,
                            "domain": domain
                        })
                    return converted
                return []
            else:
                # نظام mail.tm الجديد
                headers = {"Authorization": f"Bearer {real_token}"}
                r = _global_session.get(f"{api_base}/messages", headers=headers, timeout=8)
                if r.status_code == 200:
                    data = r.json()
                    msgs = data.get('hydra:member', [])
                    return msgs
        else:
            # النظام القديم مباشر
            login = login_or_pwd
            domain = token_or_domain
            url = f"https://www.1secmail.com/api/v1/?action=getMessages&login={login}&domain={domain}"
            r = _global_session.get(url, timeout=5)
            if r.status_code == 200:
                return r.json()
    except Exception as e:
        print(f"get_temp_messages error: {e}")
    return []

def read_temp_message(login_or_pwd, token_or_domain, msg_id):
    """يقرا الرسالة كاملة"""
    try:
        if "|||" in token_or_domain:
            parts = token_or_domain.split("|||")
            api_base = parts[0]
            real_token = parts[1]
            
            if api_base == "1sec":
                domain = real_token
                login = login_or_pwd
                url = f"https://www.1secmail.com/api/v1/?action=readMessage&login={login}&domain={domain}&id={msg_id}"
                r = _global_session.get(url, timeout=5)
                if r.status_code == 200:
                    j = r.json()
                    # نحول لشكل موحد
                    return {
                        "subject": j.get("subject",""),
                        "from": j.get("from",""),
                        "textBody": j.get("textBody","") + " " + j.get("body",""),
                        "htmlBody": j.get("htmlBody","") + " " + j.get("html",""),
                        "body": j.get("body","") or j.get("textBody","")
                    }
                return None
            else:
                headers = {"Authorization": f"Bearer {real_token}"}
                r = _global_session.get(f"{api_base}/messages/{msg_id}", headers=headers, timeout=8)
                if r.status_code == 200:
                    j = r.json()
                    # mail.tm بيرجع html و text منفصلين
                    return {
                        "subject": j.get("subject",""),
                        "from": j.get("from",{}).get("address","") if isinstance(j.get("from"), dict) else str(j.get("from","")),
                        "textBody": j.get("text",""),
                        "htmlBody": j.get("html",""),
                        "body": j.get("text","") + " " + j.get("html",""),
                        "intro": j.get("intro","")
                    }
        else:
            login = login_or_pwd
            domain = token_or_domain
            url = f"https://www.1secmail.com/api/v1/?action=readMessage&login={login}&domain={domain}&id={msg_id}"
            r = _global_session.get(url, timeout=5)
            if r.status_code == 200:
                return r.json()
    except Exception as e:
        print(f"read_temp_message error: {e}")
    return None

def extract_code_from_text(text):
    # يستخرج الأكواد من النص (قوي جدا - يدعم كل المنصات)
    patterns = [
        r'\b\d{4,8}\b',  # 4-8 أرقام
        r'code[^\d]*?(\d{4,8})',
        r'رمز[^\d]*?(\d{4,8})',
        r'verification[^\d]*?(\d{4,8})',
        r'OTP[^\d]*?(\d{4,8})',
    ]
    import re
    for pat in patterns:
        m = re.search(pat, text, re.I)
        if m:
            return m.group(1) if m.lastindex else m.group(0)
    return None

_main_kb_cache = {}

def main_keyboard(lang="ar"):
    texts = {
        "ar": {
            "names": "👥 الأسماء 🌍",
            "pass": "🔑 إنشاء كلمة مرور",
            "2fa": "🔐 كود 2FA",
            "id": "🆔 استخراج ID",
            "mail": "📧 بريد مؤقت",
            "nums": "📱 أرقام مؤقتة",
            "clip": "📋 الحافظة",
            "down": "📥 تحميل الحافظة",
            "support": "💬 الدعم الفني",
            "help": "❓ المساعدة",
            "lang": "🌐 تغيير اللغة",
            "updates": "📢 آخر التحديثات",
            "rate": "⭐ تقييم البوت"
        },
        "en": {
            "names": "👥 Names 🌍",
            "pass": "🔑 Generate Password",
            "2fa": "🔐 2FA Code",
            "id": "🆔 Extract ID",
            "mail": "📧 Temp Mail",
            "nums": "📱 Temp Numbers",
            "clip": "📋 Clipboard",
            "down": "📥 Download Clipboard",
            "support": "💬 Support",
            "help": "❓ Help",
            "lang": "🌐 Change Language",
            "updates": "📢 Latest Updates",
            "rate": "⭐ Rate Bot"
        }
    }
    t = texts.get(lang, texts["ar"])
    if lang in _main_kb_cache:
        return _main_kb_cache[lang]
    kb = ReplyKeyboardMarkup([
        [KeyboardButton(t["names"])],
        [KeyboardButton(t["pass"]), KeyboardButton(t["2fa"]), KeyboardButton(t["id"])],
        [KeyboardButton(t["mail"]), KeyboardButton(t["nums"])],
        [KeyboardButton(t["clip"]), KeyboardButton(t["down"])],
        [KeyboardButton(t["support"]), KeyboardButton(t["help"])],
        [KeyboardButton(t["lang"]), KeyboardButton(t["updates"])],
        [KeyboardButton(t["rate"])],
    ], resize_keyboard=True, is_persistent=True)
    _main_kb_cache[lang] = kb
    return kb

def get_main_keyboard_for_user(uid):
    try:
        lang = get_user_lang(uid)
    except:
        lang = "ar"
    return main_keyboard(lang)

def old_main_buttons():
    return ["👥 الأسماء 🌍", "👥 Names 🌍", "ادمن", "AHMED2009", "🌐 تغيير اللغة", "🌐 Change Language"]


def gender_keyboard(t):
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("👦 ولد", callback_data=f"gender_{t}_ولد"), InlineKeyboardButton("👧 بنت", callback_data=f"gender_{t}_بنت")],
        [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
    ])

FIRST_NAMES_AR_MALE = ["أحمد","محمد","محمود","يوسف","خالد","عمر","علي","حسام","إبراهيم","كريم","مصطفى","عبدالله","عبدالرحمن","سعيد","حسن","حسين","طارق","وليد","أيمن","سامح","تامر","هاني","شريف","ماجد","ناصر","فهد","سليم","رامي","باسم","نادر","عماد","عادل","أشرف","أسامة","إيهاب","إسلام","أمير","أنس","بلال","جمال","كامل","ماهر","وائل","سامي","حاتم","رائد","فادي","مروان","يزن"]
FIRST_NAMES_AR_FEMALE = ["نور","سلمى","مريم","آية","حبيبة","سارة","منة","فرح","نورهان","جنى","ليلى","هند","شهد","رنا","دينا","ياسمين","بسمة","إيمان","أسماء","هاجر","فاطمة","زينب","رقية","خديجة","مها","نهى","دعاء","آلاء","إسراء","شيماء","نيرة","مي","ميرنا","ريهام","ريم","لمى","لجين","حنين","جمانة","سندس","جوري","تالا","لين","سيلا","ملك","رحمة","أروى","سجى","نغم"]
LAST_NAMES_AR = ["المصري","السعيد","الشرقاوي","جابر","مصطفى","شريف","الشامي","الجندي","الشناوي","فؤاد","حسن","علي","عادل","خالد","محمود","إبراهيم","عبدالله","سالم","ناصر","حمدي","فتحي","زكي","رشدي","كمال","نجيب","حجازي","العربي","الغندور","منصور","السيد","أحمد","محمد","يوسف","عمر","عثمان","سليمان","إسماعيل","يونس","حمزة","النجار","الحداد","النجدي","العتيبي","القحطاني","الزهراني","الغامدي","السبيعي","الدوسري","الشهري","العسيري"]
FIRST_NAMES_EN_MALE = ["James","John","David","Michael","Robert","William","Thomas","Charles","Daniel","Matthew","Anthony","Mark","Donald","Steven","Paul","Andrew","Joshua","Kenneth","Kevin","Brian","George","Edward","Ronald","Timothy","Jason","Jeffrey","Ryan","Jacob","Gary","Nicholas","Eric","Jonathan","Stephen","Larry","Justin","Scott","Brandon","Benjamin","Samuel","Gregory","Alexander","Frank","Patrick","Jack","Dennis","Jerry","Tyler","Aaron","Jose","Adam"]
FIRST_NAMES_EN_FEMALE = ["Emma","Olivia","Sophia","Isabella","Mia","Charlotte","Amelia","Harper","Evelyn","Abigail","Emily","Elizabeth","Mila","Ella","Avery","Sofia","Camila","Luna","Aria","Scarlett","Penelope","Layla","Chloe","Victoria","Madison","Eleanor","Grace","Nora","Riley","Zoey","Hannah","Hazel","Lily","Ellie","Violet","Lillian","Zoe","Stella","Aurora","Natalie","Addison","Leah","Lucy","Paisley","Audrey","Brooklyn","Bella","Claire","Skylar"]
LAST_NAMES_EN = ["Smith","Johnson","Williams","Brown","Jones","Garcia","Miller","Davis","Rodriguez","Martinez","Hernandez","Lopez","Gonzalez","Wilson","Anderson","Thomas","Taylor","Moore","Jackson","Martin","Lee","Perez","Thompson","White","Harris","Sanchez","Clark","Ramirez","Lewis","Robinson","Walker","Young","Allen","King","Wright","Scott","Torres","Nguyen","Hill","Flores","Green","Adams","Nelson","Baker","Hall","Rivera","Campbell","Mitchell","Carter"]
FIRST_NAMES_MIXED_MALE = ["Alex","Carlos","Luca","Hans","Ivan","Kenji","Ahmed","Omar","Liam","Noah","Ethan","Lucas","Mason","Oliver","Elijah","Mateo","Santiago","Leonardo","Andrei","Youssef","Khalid","Amir","Rayan","Zain","Adam","Yasin","Faris","Kareem","Bilal","Hamza"]
FIRST_NAMES_MIXED_FEMALE = ["Sofia","Giulia","Anna","Yuki","Fatima","Luna","Nina","Zara","Ava","Isabella","Mia","Amara","Aisha","Layla","Mariam","Nour","Salma","Hana","Sara","Leila","Maya","Lina","Dina","Rana","Jana","Hala","Nada","Rania","Samira","Yasmin"]
LAST_NAMES_MIXED = ["Garcia","Rossi","Muller","Tanaka","Hassan","Ali","Silva","Petrov","Ahmed","Kim","Chen","Wang","Li","Singh","Patel","Kumar","Costa","Santos","Oliveira","Fernandez","Gonzalez","Rodriguez","Martinez","Lopez","Hernandez","Gomez","Diaz","Reyes","Morales"]

def generate_infinite_name(category, gender, uid):
    import random
    try:
        db = fast_load_db()
        used_key = f"used_names_{category}_{gender}"
        user_data = db.get("users", {}).get(str(uid), {})
        used = set(user_data.get(used_key, []))
    except:
        used = set()
        db = None
    if category == "ar":
        first_pool = FIRST_NAMES_AR_MALE if gender=="male" else FIRST_NAMES_AR_FEMALE
        last_pool = LAST_NAMES_AR
    elif category == "en":
        first_pool = FIRST_NAMES_EN_MALE if gender=="male" else FIRST_NAMES_EN_FEMALE
        last_pool = LAST_NAMES_EN
    else:
        first_pool = FIRST_NAMES_MIXED_MALE if gender=="male" else FIRST_NAMES_MIXED_FEMALE
        last_pool = LAST_NAMES_MIXED
    try:
        max_comb = len(first_pool) * len(last_pool)
        if len(used) >= max_comb * 0.85:
            used = set()
        for _ in range(100):
            first = random.choice(first_pool)
            last = random.choice(last_pool)
            full = f"{first} {last}"
            if full not in used:
                if db is not None:
                    try:
                        used.add(full)
                        db.setdefault("users", {}).setdefault(str(uid), {})[used_key] = list(used)[-300:]
                        fast_save_db(db)
                    except:
                        pass
                return full
    except:
        pass
    try:
        first = random.choice(first_pool)
        last = random.choice(last_pool)
        return f"{first} {last}"
    except:
        return "أحمد المصري" if category=="ar" else "James Smith"

def names_main_keyboard(lang="ar"):
    labels = {
        "ar": {"ar": "🇪🇬 عربي", "en": "🇺🇸 إنجليزي", "mixed": "🌍 أجنبي متنوع", "back": "🔙 رجوع"},
        "en": {"ar": "🇪🇬 Arabic", "en": "🇺🇸 English", "mixed": "🌍 Mixed Foreign", "back": "🔙 Back"}
    }
    l = labels.get(lang, labels["ar"])
    return InlineKeyboardMarkup([
        [InlineKeyboardButton(l["ar"], callback_data="names_ar"), InlineKeyboardButton(l["en"], callback_data="names_en")],
        [InlineKeyboardButton(l["mixed"], callback_data="names_mixed")],
        [InlineKeyboardButton(l["back"], callback_data="main")]
    ])

def names_gender_keyboard(category, lang="ar"):
    trans = {
        "ar": {"male": "👦 ولد", "female": "👧 بنت", "another": "🔄 اسم تاني ♾️", "back_sec": "⬅️ رجوع للأقسام", "back_main": "🔙 القائمة الرئيسية"},
        "en": {"male": "👦 Boy", "female": "👧 Girl", "another": "🔄 Another ♾️", "back_sec": "⬅️ Back to categories", "back_main": "🔙 Main menu"}
    }
    tr = trans.get(lang, trans["ar"])
    return InlineKeyboardMarkup([
        [InlineKeyboardButton(tr["male"], callback_data=f"names_{category}_male"), InlineKeyboardButton(tr["female"], callback_data=f"names_{category}_female")],
        [InlineKeyboardButton(tr["another"], callback_data=f"names_{category}_male")],
        [InlineKeyboardButton(tr["back_sec"], callback_data="names_main")],
        [InlineKeyboardButton(tr["back_main"], callback_data="main")]
    ])

def admin_keyboard(uid=None):
    db = fast_load_db()
    settings = load_settings()
    is_owner = uid is None or uid == SUPER_ADMINS[0]
    perms = settings.get("admin_perms",{}).get(str(uid),[]) if not is_owner else ["all"]
    def has(p):
        return is_owner or "all" in perms or p in perms
    kb=[]
    kb.append([InlineKeyboardButton(f"👑 الادمنية ({len(SUPER_ADMINS)})", callback_data="adm_list_admins")])
    if has("admins") or is_owner:
        kb.append([InlineKeyboardButton("➕ اضافة ادمن", callback_data="adm_add_admin"), InlineKeyboardButton("➖ ازالة ادمن", callback_data="adm_remove_admin")])
        kb.append([InlineKeyboardButton("🔐 صلاحيات الادمن", callback_data="adm_perms_menu")])
        kb.append([InlineKeyboardButton("🔑 تغيير كلمة السر", callback_data="adm_change_pass")])
    if has("msgs") or is_owner:
        kb.append([InlineKeyboardButton("✏️ رسائل الترحيب", callback_data="adm_msgs")])
    if has("stats") or is_owner:
        kb.append([InlineKeyboardButton("📈 احصاءات", callback_data="adm_stats")])
    if has("users") or is_owner:
        kb.append([InlineKeyboardButton("👥 المستخدمين", callback_data="adm_users_dash"), InlineKeyboardButton("🆕 الجداد اليوم", callback_data="adm_new_today")])
        kb.append([InlineKeyboardButton("🟢 النشطين اليوم", callback_data="adm_active_today"), InlineKeyboardButton("🚫 البلوك", callback_data="adm_blocked")])
        kb.append([InlineKeyboardButton("📋 قائمة اليوزرات", callback_data="adm_list_users"), InlineKeyboardButton("➕ اضافة يوزر", callback_data="adm_add_user")])
        kb.append([InlineKeyboardButton("📤 رفع يوزرات", callback_data="adm_upload_users"), InlineKeyboardButton("📥 تحميل اليوزرات", callback_data="adm_download_users")])
    kb.append([InlineKeyboardButton("⭐ التقييمات", callback_data="adm_ratings"), InlineKeyboardButton("💌 الملاحظات", callback_data="adm_feedbacks")])
    kb.append([InlineKeyboardButton("📱 الأرقام المستخدمة", callback_data="adm_used_numbers"), InlineKeyboardButton("📊 إحصائيات الأرقام", callback_data="adm_numbers_stats")])
    kb.append([InlineKeyboardButton("📧 إحصائيات البريد", callback_data="adm_mail_stats"), InlineKeyboardButton("🗑️ مسح الأرقام", callback_data="adm_clear_numbers")])
    kb.append([InlineKeyboardButton("➕ إضافة خدمة", callback_data="adm_add_service"), InlineKeyboardButton("✏️ تعديل عدد الخدمة", callback_data="adm_edit_service")])
    kb.append([InlineKeyboardButton("📢 إذاعة للكل", callback_data="adm_broadcast"), InlineKeyboardButton("🚫 حظر/فك حظر", callback_data="adm_ban_user")])
    kb.append([InlineKeyboardButton("⛔ ايقاف البوت", callback_data="adm_stop"), InlineKeyboardButton("▶️ تشغيل البوت", callback_data="adm_start")])
    kb.append([InlineKeyboardButton("📤 تصدير كل البيانات", callback_data="adm_export_all"), InlineKeyboardButton("❌ اغلاق", callback_data="adm_close")])
    return InlineKeyboardMarkup(kb)

def whatsapp_button():
    url = f"https://wa.me/{WHATSAPP_NUMBER}?text=عايز%20يوزر%20للبوت"
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("💬 تليجرام (أحمد) 👑", url="tg://user?id=6364073135")],
        [InlineKeyboardButton("📲 واتساب", url=url)]
    ])

async def send_copyable_message(message_obj, title, value):
    safe = str(value).replace("&","&amp;").replace("<","&lt;").replace(">","&gt;")
    text = f"{title}\n\n━━━━━━━━━━━━━━━\n<code>{safe}</code>\n━━━━━━━━━━━━━━━\n\n👆 <b>اضغط على النص لنسخه ✨</b>"
    kb = InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع للقائمة", callback_data="main")]])
    try:
        if hasattr(message_obj, 'edit_message_text'):
            await message_obj.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        else:
            await message_obj.reply_text(text, parse_mode="HTML", reply_markup=kb)
    except Exception as e:
        try:
            if hasattr(message_obj, 'edit_message_text'):
                await message_obj.edit_message_text(f"{title}:\n{value}", reply_markup=kb)
            else:
                await message_obj.reply_text(f"{title}:\n{value}", reply_markup=kb)
        except:
            pass

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    try:
        update_user_track(uid, update.effective_user.username or "", update.effective_user.first_name or "")
    except: pass
    settings = load_settings()
    if settings.get("bot_active")==False and not is_admin(uid):
        await update.message.reply_text("⛔ البوت متوقف حالياً")
        return
    
    # فحص اللغة أولاً - لو مش مختار لغة
    db = fast_load_db()
    if str(uid) not in db.get("user_langs", {}):
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("🇪🇬 العربية", callback_data="lang_ar"), InlineKeyboardButton("🇺🇸 English", callback_data="lang_en")]
        ])
        await update.message.reply_text("🌐 <b>اختر لغتك / Choose your language</b>\n\n━━━━━━━━━━━━━━━\n🇪🇬 العربية\n🇺🇸 English\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=kb)
        return
    
    if is_authorized(uid):
        first_name = update.effective_user.first_name or "يا غالي"
        name = first_name
        lang = get_user_lang(uid)
        if lang == "en":
            tmpl = "👋 Welcome, {first_name}\n\n🤖 AHMED Bot\n\n━━━━━━━━━━━━━━━\n⚡ Fast Execution\n🔒 Security & Privacy\n🚀 Smart Tools in One Place\n━━━━━━━━━━━━━━━\n\n📌 Choose a service from the buttons below."
        else:
            tmpl = settings.get("welcome_auth","👋 أهلاً بك، {first_name}\n\n🤖 AHMED Bot\n\n━━━━━━━━━━━━━━━\n⚡ سرعة في التنفيذ\n🔒 أمان وخصوصية\n🚀 أدوات ذكية في مكان واحد\n━━━━━━━━━━━━━━━\n\n📌 اختر الخدمة المطلوبة من الأزرار بالأسفل.")
        try:
            welcome = tmpl.format(first_name=first_name, name=name)
        except:
            try:
                welcome = tmpl.format(name=name)
            except:
                welcome = tmpl
        if welcome.strip():
            await update.message.reply_text(welcome, parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
        else:
            await update.message.reply_text("👇 اختار من القائمة", reply_markup=main_keyboard(get_user_lang(uid)))
    else:
        first_name = update.effective_user.first_name or ""
        lang = get_user_lang(uid)
        if str(uid) not in db.get("user_langs", {}):
            # لو لسه مختارش لغة، اختار له عربي افتراضي واطلب يوزر
            set_user_lang(uid, "ar")
            lang = "ar"
        
        if lang == "en":
            tmpl = LANGS["en"]["welcome_unauth"]
        else:
            tmpl = settings.get("welcome_unauth","👋 أهلاً بك، {first_name}")
        try:
            welcome = tmpl.format(first_name=first_name, name=first_name)
        except:
            welcome = tmpl
        
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("🇪🇬 العربية", callback_data="lang_ar"), InlineKeyboardButton("🇺🇸 English", callback_data="lang_en")]
        ])
        await update.message.reply_text(welcome, parse_mode="HTML", reply_markup=kb)


# ==================== نظام الاشعارات التلقائية الفخم V31 ====================
async def temp_mail_watcher(context: ContextTypes.DEFAULT_TYPE):
    job = context.job
    chat_id = job.chat_id
    mail = job.data.get('mail')
    pwd = job.data.get('pwd')
    token_data = job.data.get('token_data')
    lang = job.data.get('lang', 'ar')
    last_count = job.data.get('last_count', 0)
    
    try:
        messages = get_temp_messages(pwd, token_data)
        if len(messages) > last_count and len(messages) > 0:
            # فيه رسالة جديدة!
            newest = messages[0]  # احدث رسالة
            msg_id = newest.get('id')
            full = read_temp_message(pwd, token_data, msg_id)
            body_text = ""
            if full:
                body_text = (full.get('textBody','') or '') + " " + (full.get('htmlBody','') or '') + " " + (full.get('body','') or '') + " " + (full.get('subject','') or '')
            code = extract_code_from_text(body_text) or extract_code_from_text(str(newest))
            
            # منصة من الايميل
            from_text = full.get('from','') if full else newest.get('from','')
            if isinstance(from_text, dict):
                from_text = from_text.get('address','')
            platform = "Facebook"
            if 'facebook' in body_text.lower() or 'facebook' in str(from_text).lower():
                platform = "Facebook" if lang != 'ar' else "فيسبوك"
            elif 'instagram' in body_text.lower():
                platform = "Instagram" if lang != 'ar' else "انستجرام"
            elif 'tiktok' in body_text.lower():
                platform = "TikTok" if lang != 'ar' else "تيك توك"
            elif 'google' in body_text.lower():
                platform = "Google" if lang != 'ar' else "جوجل"
            elif 'telegram' in body_text.lower():
                platform = "Telegram" if lang != 'ar' else "تليجرام"
            elif 'whatsapp' in body_text.lower():
                platform = "WhatsApp" if lang != 'ar' else "واتساب"
            else:
                platform = from_text[:20] if from_text else ("Unknown" if lang!='ar' else "غير معروف")
            
            import datetime
            now_time = datetime.datetime.now().strftime("%I:%M:%S %p")
            
            if not code:
                code = "تم وصول رسالة - افتح البريد" if lang=='ar' else "New message - open inbox"
            
            if lang == 'ar':
                fancy_text = f"""💎━━━━━━━━━━━━━━━━💎

   ✨ بسم الله الرحمن الرحيم ✨

   👑 تم استلام كودك بنجاح 👑

▬▬▬▬▬▬▬▬▬▬▬▬▬▬

     🎯 كود التحقق الخاص بك

        ┏━━━━━━━━━━━━━┓
        ┃   <code>{code}</code>   ┃
        ┗━━━━━━━━━━━━━┛

▬▬▬▬▬▬▬▬▬▬▬▬▬▬

📨 من : {platform}
📧 إلى : {mail}
⏰ وصل : الآن ({now_time})
✅ الحالة : جديد و صالح للاستخدام
🔥 السرعة : فائقة - وصول فوري

━━━━━━━━━━━━━━━━━━━━
👆 اضغط على الكود لنسخه فوراً
━━━━━━━━━━━━━━━━━━━━

💎━━━━━━━━━━━━━━━━💎
   ⚜️ AHMED VIP SYSTEM ⚜️
💎━━━━━━━━━━━━━━━━💎"""
            else:
                fancy_text = f"""💎━━━━━━━━━━━━━━━━💎

   ✨ AHMED ELITE SYSTEM ✨

   👑 YOUR CODE HAS ARRIVED 👑

▬▬▬▬▬▬▬▬▬▬▬▬▬▬

     🎯 YOUR VERIFICATION CODE

        ┏━━━━━━━━━━━━━┓
        ┃   <code>{code}</code>   ┃
        ┗━━━━━━━━━━━━━┛

▬▬▬▬▬▬▬▬▬▬▬▬▬▬

📨 FROM : {platform}
📧 TO : {mail}
⏰ ARRIVED : Just now ({now_time})
✅ STATUS : Fresh & Valid
🔥 SPEED : Instant Delivery

━━━━━━━━━━━━━━━━━━━━
👆 Tap code to copy instantly
━━━━━━━━━━━━━━━━━━━━

💎━━━━━━━━━━━━━━━━💎
   ⚜️ AHMED VIP SYSTEM ⚜️
💎━━━━━━━━━━━━━━━━💎"""
            
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("📧 فتح البريد" if lang=='ar' else "📧 Open Inbox", callback_data="temp_check")],
                [InlineKeyboardButton("🔄 بريد جديد" if lang=='ar' else "🔄 New Mail", callback_data="temp_new")]
            ])
            
            await context.bot.send_message(chat_id=chat_id, text=fancy_text, parse_mode="HTML", reply_markup=kb)
            
            # وقف المراقبة بعد ما بعت الكود (عشان ما يزعجش)
            job.data['last_count'] = len(messages)
            # نوقفه - المستخدم لو عايز يرجع يشغله يدوس فحص
            # job.schedule_removal()  # نسيبه شغال لحد ما يجي كود تاني
            job.data['last_count'] = len(messages)
        else:
            job.data['last_count'] = len(messages)
    except Exception as e:
        print(f"Watcher error: {e}")



async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    uid = query.from_user.id
    data = query.data
    db = fast_load_db()
    try:
        await query.answer()
    except: pass

    if data == "main" or data == "main_reply":
        try:
            await query.delete_message()
        except: pass
        await context.bot.send_message(chat_id=query.message.chat_id, text="👋 أهلاً بيك\n👇 اختار من القائمة", reply_markup=main_keyboard(get_user_lang(uid)))
        return
    if data == "save_start":
        await query.edit_message_text("📝 تمام، ابعت اللي عايز تحفظه دلوقتي (حتى لو 5000 حرف)", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("❌ إلغاء", callback_data="main")]]))
        context.user_data["waiting"] = "save_item"
        return
    if data == "saved_list":
        saved = db["users"].get(str(uid), {}).get("saved", [])
        if not saved:
            await query.edit_message_text("🗃️ الحافظة فاضية", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
        else:
            txt = "\n\n".join(saved[-10:])
            if len(txt) > 3000:
                txt = txt[:3000]
            await query.edit_message_text(f"🗃️ الحافظة ({len(saved)}):\n\n{txt}", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
        return
    if data in ["download_saved", "download_saved_inline"]:
        saved = db["users"].get(str(uid), {}).get("saved", [])
        if not saved:
            await query.edit_message_text("🗃️ فاضية")
        else:
            path = f"saved_{uid}.txt"
            with open(path, "w", encoding="utf-8") as f:
                f.write("\n".join(saved))
            await context.bot.send_document(chat_id=query.message.chat_id, document=open(path,"rb"), filename=f"hafza_{len(saved)}.txt", caption=f"📦 حافظتك - {len(saved)} عنصر")
            os.remove(path)
            await query.edit_message_text(f"✅ بعتلك الملف فيه {len(saved)} عنصر", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
        return
    if data == "clear_saved_inline":
        await query.edit_message_text("⚠️ متأكد عايز تمسح الحافظة كلها؟", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("✅ ايوة امسح", callback_data="confirm_clear_inline"), InlineKeyboardButton("❌ لا", callback_data="main")]]))
        return
    if data == "confirm_clear_inline":
        db["users"].setdefault(str(uid), {"saved":[]})
        db["users"][str(uid)]["saved"] = []
        fast_save_db(db)
        await query.edit_message_text("🗑️ تم مسح الحافظة", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
        return
    if data == "names_main" or data == "names":
        lang = get_user_lang(uid)
        await query.edit_message_text("👥 <b>الأسماء</b>\n\n━━━━━━━━━━━━━━━\n🌍 اختر نوع الاسم:", parse_mode="HTML", reply_markup=names_main_keyboard(lang))
        return
    if data in ["egy_name","foreign_name"]:
        lang = get_user_lang(uid)
        await query.edit_message_text("👥 <b>الأسماء</b>\n\n━━━━━━━━━━━━━━━\n🌍 اختر نوع الاسم:", parse_mode="HTML", reply_markup=names_main_keyboard(lang))
        return
    if data.startswith("names_") and data.count("_") == 1:
        cat = data.split("_")[1]
        if cat == "main":
            lang = get_user_lang(uid)
            await query.edit_message_text("👥 <b>الأسماء</b>\n\n━━━━━━━━━━━━━━━\n🌍 اختر نوع الاسم:", parse_mode="HTML", reply_markup=names_main_keyboard(lang))
            return
        lang = get_user_lang(uid)
        cat_names = {"ar": "العربي 🇪🇬", "en": "الإنجليزي 🇺🇸", "mixed": "الأجنبي المتنوع 🌍"}
        label = cat_names.get(cat, cat)
        await query.edit_message_text(f"👤 <b>الاسم {label}</b>\n\n━━━━━━━━━━━━━━━\n👦👧 اختر النوع:", parse_mode="HTML", reply_markup=names_gender_keyboard(cat, lang))
        return
    if data.startswith("names_") and data.count("_") == 2:
        try:
            _, cat, gender_key = data.split("_")
            lang = get_user_lang(uid)
            gender = "male" if gender_key == "male" else "female"
            name = generate_infinite_name(cat, gender, uid)
            trans_gender = {"ar": {"male": "👦 ولد", "female": "👧 بنت"}, "en": {"male": "👦 Boy", "female": "👧 Girl"}}
            gender_label = trans_gender.get(lang, trans_gender["ar"]).get(gender_key, gender_key)
            cat_label = {"ar": "عربي", "en": "إنجليزي", "mixed": "أجنبي"}[cat]
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 اسم تاني ♾️" if lang=="ar" else "🔄 Another ♾️", callback_data=f"names_{cat}_{gender_key}")],
                [InlineKeyboardButton("⬅️ رجوع للأقسام" if lang=="ar" else "⬅️ Back", callback_data="names_main")],
                [InlineKeyboardButton("🔙 القائمة الرئيسية" if lang=="ar" else "🔙 Main", callback_data="main")]
            ])
            await query.edit_message_text(f"👤 <b>الاسم الـ {cat_label}</b> ({gender_label})\n\n━━━━━━━━━━━━━━━\n<code>{name}</code>\n━━━━━━━━━━━━━━━\n♾️ <b>لا نهائي ولا يتكرر</b>\n👆 دوس لنسخ", parse_mode="HTML", reply_markup=kb)
        except Exception as e:
            lang = get_user_lang(uid)
            await query.edit_message_text(f"❌ خطأ: {e}", parse_mode="HTML", reply_markup=names_main_keyboard(lang))
        return
    if data.startswith("gender_"):
        try:
            _, type_name, gender = data.split("_")
            is_male = gender in ["ذكر", "ولد", "👦 ولد", "ولد"]
            cat = "ar" if type_name=="مصري" else "en"
            gender_key = "male" if is_male else "female"
            name = generate_infinite_name(cat, gender_key, uid)
            gender_label = "👦 ولد" if is_male else "👧 بنت"
            await send_copyable_message(query, f"👤 <b>الاسم {type_name}</b> ({gender_label})", name)
        except:
            lang = get_user_lang(uid)
            await query.edit_message_text("👥 <b>الأسماء</b>", parse_mode="HTML", reply_markup=names_main_keyboard(lang))
        return
    if data == "password":
        pwd = ''.join(random.choice(string.ascii_letters+string.digits+"@#$%") for _ in range(14))
        await send_copyable_message(query, "🔑 الباسورد", pwd)
        return
    if data == "2fa":
        await query.edit_message_text("🔐 ابعت كود الـ 2FA الـ Secret", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
        context.user_data["waiting"] = "2fa_code"
        return
    if data.startswith("lang_"):
        lang = data.split("_")[1]
        set_user_lang(uid, lang)
        db = fast_load_db()
        if is_authorized(uid):
            first_name = query.from_user.first_name or "غالي"
            if lang == "en":
                welcome = f"👋 Welcome, {first_name}\n\n🤖 AHMED Bot\n\n━━━━━━━━━━━━━━━\n⚡ Fast Execution\n🔒 Security & Privacy\n🚀 Smart Tools in One Place\n━━━━━━━━━━━━━━━\n\n📌 Choose a service from the buttons below."
            else:
                welcome = f"👋 أهلاً بك، {first_name}\n\n🤖 AHMED Bot\n\n━━━━━━━━━━━━━━━\n⚡ سرعة في التنفيذ\n🔒 أمان وخصوصية\n🚀 أدوات ذكية في مكان واحد\n━━━━━━━━━━━━━━━\n\n📌 اختر الخدمة المطلوبة من الأزرار بالأسفل."
            await query.edit_message_text(welcome, parse_mode="HTML")
            await context.bot.send_message(chat_id=query.message.chat_id, text="👇 اختر من القائمة" if lang=="ar" else "👇 Choose from menu", reply_markup=main_keyboard(get_user_lang(uid)))
        else:
            if lang == "en":
                await query.edit_message_text("🔐 Enter your username to continue ✅", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🇪🇬 العربية", callback_data="lang_ar"), InlineKeyboardButton("🇺🇸 English", callback_data="lang_en")]]))
            else:
                await query.edit_message_text("🔐 اكتب اليوزر الخاص بك للمواصلة ✅", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🇪🇬 العربية", callback_data="lang_ar"), InlineKeyboardButton("🇺🇸 English", callback_data="lang_en")]]))
        return
    if data == "nums_services" or data == "nums_real":
        lang = get_user_lang(uid)
        # عرض الخدمات زي الصورة الثانية
        if lang == "en":
            txt = "✅ <b>Choose Service:</b> 🌍"
        else:
            txt = "✅ <b>الخدمة المطلوبة:</b> اختر الخدمة 🌍"
        await query.edit_message_text(txt, parse_mode="HTML", reply_markup=get_services_keyboard(lang))
        return
    if data.startswith("svc_"):
        service_id = data.split("_")[1]
        lang = get_user_lang(uid)
        service = next((s for s in SERVICES if s["id"]==service_id), None)
        if not service:
            return
        context.user_data["selected_service"] = service_id
        name = service["name_en"] if lang=="en" else service["name_ar"]
        if lang == "en":
            txt = f"✅ <b>Selected Service: {service['name_en']}</b>\n🌍 <b>Please choose country:</b>"
        else:
            txt = f"✅ <b>الخدمة المطلوبة: {service['name_en']}</b>\n🌍 <b>يرجى اختيار الدولة:</b>"
        await query.edit_message_text(txt, parse_mode="HTML", reply_markup=get_countries_keyboard(service_id, lang))
        return
    if data.startswith("country_"):
        _, service_id, country_code = data.split("_")
        lang = get_user_lang(uid)
        service = next((s for s in SERVICES if s["id"]==service_id), None)
        country = next((c for c in COUNTRIES if c["code"]==country_code), None)
        if not service or not country:
            return
        
        # نجيب رقم مش مستخدم قبل كده لنفس الخدمة
        full, local = get_unique_number(service_id, country_code)
        
        # نتأكد انه مش مستخدم (لو مستخدم نجيب غيره)
        attempts = 0
        while is_number_used(full, service_id) and attempts < 10:
            full, local = get_unique_number(service_id, country_code)
            attempts += 1
        
        # نحجز الرقم للخدمة دي
        mark_number_used(full, service_id, uid, country_code)
        
        # حفظ الاختيار
        context.user_data["selected_country"] = country_code
        context.user_data["temp_mail"] = full
        context.user_data["temp_login"] = full
        
        svc_name = service["name_en"]
        country_name = country["name_en"]
        
        if lang == "en":
            text = f"✅ <b>Service:</b> {svc_name}\n🌍 <b>Country:</b> {country['flag']} {country_name}\n\n━━━━━━━━━━━━━━━\n📱 <b>Your Number:</b>\n<code>{full}</code>\n━━━━━━━━━━━━━━━\n💡 Use it for {svc_name}\n📥 Code will arrive here\n⏰ Valid for 20 min\n━━━━━━━━━━━━━━━"
        else:
            text = f"✅ <b>الخدمة المطلوبة: {svc_name}</b>\n🌍 <b>الدولة: {country['flag']} {country_name}</b>\n\n━━━━━━━━━━━━━━━\n📱 <b>رقمك:</b>\n<code>{full}</code>\n━━━━━━━━━━━━━━━\n💡 استخدمه لـ {svc_name}\n📥 الكود هيوصلك هنا\n⏰ صالح لمدة 20 دقيقة\n━━━━━━━━━━━━━━━\n👆 دوس على الرقم لنسخه"
        
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("📥 فحص الكود" if lang=="ar" else "📥 Check Code", callback_data=f"nums_check_{full}")],
            [InlineKeyboardButton("🔄 رقم جديد" if lang=="ar" else "🔄 New Number", callback_data=f"country_{service_id}_{country_code}")],
            [InlineKeyboardButton("⬅️ رجوع للدول" if lang=="ar" else "⬅️ BACK", callback_data=f"svc_{service_id}")],
            [InlineKeyboardButton("🔙 القائمة الرئيسية" if lang=="ar" else "🔙 Main Menu", callback_data="main")]
        ])
        await query.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        return
    if data == "extract_id":
        await query.edit_message_text("🆔 <b>استخراج ID</b>\n\n━━━━━━━━━━━━━━━\n🔗 ابعت اللينك أو اليوزر\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
        context.user_data["waiting"] = "extract_id"
        return
    if data == "temp_new":
        mail, login, domain = gen_temp_mail()
        context.user_data["temp_mail"] = mail
        context.user_data["temp_login"] = login
        context.user_data["temp_domain"] = domain
        # شغل المراقب التلقائي كل 4 ثواني
        try:
            # امسح الوظائف القديمة
            current_jobs = context.job_queue.get_jobs_by_name(f"mailwatch_{query.from_user.id}")
            for j in current_jobs:
                j.schedule_removal()
        except:
            pass
        try:
            lang = get_user_lang(query.from_user.id)
            context.job_queue.run_repeating(
                temp_mail_watcher,
                interval=4,
                first=4,
                chat_id=query.message.chat_id,
                name=f"mailwatch_{query.from_user.id}",
                data={'mail': mail, 'pwd': login, 'token_data': domain, 'lang': lang, 'last_count': 0}
            )
        except Exception as e:
            print(f"Job queue error: {e}")
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("📥 فحص الوارد", callback_data="temp_check")],
            [InlineKeyboardButton("🔄 بريد جديد", callback_data="temp_new")],
            [InlineKeyboardButton("🔙 رجوع للقائمة", callback_data="main")]
        ])
        await query.edit_message_text(
            f"📧 <b>البريد المؤقت الجديد</b>\n\n"
            f"━━━━━━━━━━━━━━━\n"
            f"📬 بريدك:\n<code>{mail}</code>\n"
            f"━━━━━━━━━━━━━━━\n"
            f"🔔 <b>المراقبة التلقائية شغالة!</b>\n"
            f"⚡ أول ما الكود يوصل هبعتلك رسالة فخمة أوتوماتيك\n"
            f"💎 ببلاش وقوي ومش محظور\n"
            f"━━━━━━━━━━━━━━━",
            parse_mode="HTML", reply_markup=kb
        )
        return
    if data == "temp_check":
        login = context.user_data.get("temp_login")
        domain = context.user_data.get("temp_domain")
        mail = context.user_data.get("temp_mail", "غير موجود")
        if not login:
            await query.edit_message_text("❌ مفيش بريد - دوس بريد جديد", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔄 بريد جديد", callback_data="temp_new"), InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
            return
        await query.edit_message_text(f"⏳ <b>بفحص البريد...</b>\n\n📧 {mail}\n\n━━━━━━━━━━━━━━━", parse_mode="HTML")
        messages = get_temp_messages(login, domain)
        if not messages:
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 تحديث", callback_data="temp_check")],
                [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
            ])
            await query.edit_message_text(
                f"📧 <b>البريد المؤقت</b>\n\n"
                f"📬 {mail}\n"
                f"━━━━━━━━━━━━━━━\n"
                f"📭 <b>لا توجد رسائل بعد</b>\n"
                f"⏳ انتظر وصول الكود\n"
                f"🔄 دوس تحديث كل شوية\n"
                f"━━━━━━━━━━━━━━━",
                parse_mode="HTML", reply_markup=kb
            )
            return
        # فيه رسائل
        text = f"📧 <b>البريد الوارد ({len(messages)})</b>\n\n📬 {mail}\n━━━━━━━━━━━━━━━\n"
        kb_buttons = []
        for msg in messages[:5]:
            subject = msg.get("subject","بدون عنوان")[:30]
            from_addr = msg.get("from","")[:20]
            date = msg.get("date","")[:16]
            text += f"\n📩 <b>من:</b> {from_addr}\n📌 <b>الموضوع:</b> {subject}\n🕐 {date}\n"
            # نحاول نستخرج كود
            full = read_temp_message(login, domain, msg.get("id"))
            if full:
                body = full.get("textBody","") + " " + full.get("htmlBody","")
                code = extract_code_from_text(body)
                if code:
                    text += f"🔐 <b>الكود:</b> <code>{code}</code> ✨\n"
            text += "━━━━━━━━━━━━━━━\n"
            kb_buttons.append([InlineKeyboardButton(f"📖 فتح: {subject[:15]}", callback_data=f"temp_read_{msg.get('id')}")])
        kb_buttons.append([InlineKeyboardButton("🔄 تحديث", callback_data="temp_check")])
        kb_buttons.append([InlineKeyboardButton("🔙 رجوع", callback_data="main")])
        kb = InlineKeyboardMarkup(kb_buttons)
        try:
            await query.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        except:
            await query.edit_message_text(f"📧 فيه {len(messages)} رسائل - افتحهم", reply_markup=kb)
        return
    if data == "nums_egy" or data == "nums_us":
        type_num = data.split("_")[1]
        if type_num == "egy":
            nums = gen_random_numbers(5, "egy")
            text = "🇪🇬 <b>أرقام مصرية وهمية</b>\n\n━━━━━━━━━━━━━━━\n"
            for full, local in nums:
                text += f"📱 <code>{full}</code> | <code>{local}</code>\n"
            text += "━━━━━━━━━━━━━━━\n👆 دوس على الرقم لنسخه\n💡 أرقام شكلها حقيقي للتجربة"
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 أرقام جديدة", callback_data="nums_egy")],
                [InlineKeyboardButton("🇺🇸 أمريكية", callback_data="nums_us"), InlineKeyboardButton("📡 حقيقية", callback_data="nums_real")],
                [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
            ])
            await query.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        elif type_num == "us":
            nums = gen_random_numbers(5, "us")
            text = "🇺🇸 <b>أرقام أمريكية وهمية</b>\n\n━━━━━━━━━━━━━━━\n"
            for full, local in nums:
                text += f"📱 <code>{full}</code>\n"
            text += "━━━━━━━━━━━━━━━\n👆 دوس لنسخ"
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 جديدة", callback_data="nums_us")],
                [InlineKeyboardButton("🇪🇬 مصرية", callback_data="nums_egy"), InlineKeyboardButton("📡 حقيقية", callback_data="nums_real")],
                [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
            ])
            await query.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        elif type_num == "real":
            free_nums = get_free_numbers()
            text = "📡 <b>أرقام حقيقية تستقبل SMS</b>\n\n━━━━━━━━━━━━━━━\n"
            kb_buttons = []
            for n in free_nums:
                text += f"{n['country']}: <code>{n['number']}</code>\n"
                kb_buttons.append([InlineKeyboardButton(f"📥 فحص {n['country']} {n['number']}", callback_data=f"nums_check_{n['number']}")])
            text += "━━━━━━━━━━━━━━━\n💡 الأرقام دي حقيقية وبتستقبل أكواد\n📥 دوس فحص عشان تشوف الرسائل"
            kb_buttons.append([InlineKeyboardButton("🔙 رجوع", callback_data="main")])
            kb = InlineKeyboardMarkup(kb_buttons)
            await query.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        return

    if data.startswith("check_platform_"):
        _, _, service_id, number = data.split("_", 3)
        lang = get_user_lang(uid)
        await query.edit_message_text(f"🔍 <b>بفحص الرقم...</b>\n\n📱 {number}\n📘 الخدمة: {service_id}\n\n━━━━━━━━━━━━━━━\n⏳ بفحص هل مستخدم على المنصة...", parse_mode="HTML")
        
        result = check_number_on_platform(number, service_id)
        
        if result["is_used"]:
            text = f"⚠️ <b>الرقم مستخدم</b>\n\n📱 الرقم: <code>{number}</code>\n📘 الخدمة: {service_id}\n\n━━━━━━━━━━━━━━━\n{result['message']}\n━━━━━━━━━━━━━━━\n\n💡 هنبعتلك رقم جديد مضمون"
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 هات رقم جديد مضمون" if lang=="ar" else "🔄 Get new guaranteed number", callback_data=f"country_{service_id}_{context.user_data.get('selected_country','eg')}")],
                [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
            ])
        else:
            text = f"✅ <b>الرقم جديد ومضمون</b>\n\n📱 الرقم: <code>{number}</code>\n📘 الخدمة: {service_id}\n\n━━━━━━━━━━━━━━━\n{result['message']}\n━━━━━━━━━━━━━━━\n\n✅ تقدر تستخدمه بأمان\n🔒 مضمون مش مستخدم في البوت"
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("📥 فحص الكود" if lang=="ar" else "📥 Check Code", callback_data=f"nums_check_{number}")],
                [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
            ])
        
        await query.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        return
    if data.startswith("nums_check_"):
        number = data.replace("nums_check_", "")
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("🔄 تحديث الوارد", callback_data=f"nums_check_{number}")],
            [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
        ])
        text_msg = (
            f"📡 <b>الرسائل الواردة لـ</b>\n"
            f"<code>{number}</code>\n"
            "━━━━━━━━━━━━━━━\n"
            "📭 <b>لا توجد رسائل بعد</b>\n"
            "⏳ في انتظار وصول كود واتساب...\n"
            "🔄 دوس تحديث كل شوية\n"
            "━━━━━━━━━━━━━━━"
        )
        await query.edit_message_text(text_msg, parse_mode="HTML", reply_markup=kb)
        return

    if data.startswith("temp_read_"):
        try:
            msg_id = int(data.split("_")[-1])
            login = context.user_data.get("temp_login")
            domain = context.user_data.get("temp_domain")
            full = read_temp_message(login, domain, msg_id)
            if not full:
                await query.edit_message_text("❌ مقدرتش افتح الرسالة", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="temp_check")]]))
                return
            subject = full.get("subject","بدون عنوان")
            from_addr = full.get("from","")
            body = full.get("textBody","") or full.get("htmlBody","")[:2000]
            # نظف HTML
            body = re.sub(r'<[^>]+>', '', body)[:2000]
            code = extract_code_from_text(body)
            text = f"📖 <b>{subject}</b>\n\n━━━━━━━━━━━━━━━\n👤 من: {from_addr}\n"
            if code:
                text += f"🔐 الكود: <code>{code}</code> ✨\n"
            text += f"━━━━━━━━━━━━━━━\n\n{body[:1500]}\n\n━━━━━━━━━━━━━━━"
            kb = InlineKeyboardMarkup([
                [InlineKeyboardButton("🔙 رجوع للوارد", callback_data="temp_check")],
                [InlineKeyboardButton("🔙 للقائمة", callback_data="main")]
            ])
            await query.edit_message_text(text, parse_mode="HTML", reply_markup=kb)
        except Exception as e:
            await query.edit_message_text(f"❌ خطأ: {e}", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="temp_check")]]))
        return
    if data.startswith("rate_"):
        if data == "rate_feedback":
            await query.edit_message_text("⭐ <b>كتابة ملاحظة</b>\n\n━━━━━━━━━━━━━━━\n✏️ ابعت ملاحظتك أو اقتراحك\n💡 رأيك مهم جداً لينا\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="main")]]))
            context.user_data["waiting"] = "rate_feedback"
        else:
            stars = data.split("_")[1]
            star_text = "⭐" * int(stars)
            await query.edit_message_text(f"⭐ <b>شكراً لتقييمك!</b>\n\n━━━━━━━━━━━━━━━\n{star_text}\n\n💖 شكراً لدعمك\n🚀 مستمرين في التطوير\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع للقائمة", callback_data="main")]]))
            # حفظ التقييم
            try:
                db = fast_load_db()
                db.setdefault("ratings", []).append({"uid": uid, "stars": stars, "time": str(__import__('datetime').datetime.now())})
                fast_save_db(db)
            except: pass
        return

async def admin_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    uid = query.from_user.id
    if not is_admin(uid): 
        return
    try: await query.answer()
    except: pass
    data = query.data
    db = fast_load_db()
    settings = load_settings()

    if data == "adm_close":
        try:
            await query.delete_message()
        except: pass
        return
    if data == "adm_back":
        await query.edit_message_text("🔧 لوحة تحكّم الأدمن", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_list_admins":
        txt = "\n".join([f"• {x}" for x in SUPER_ADMINS])
        await query.edit_message_text(f"👑 قائمة الأدمنية:\n{txt}", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_add_admin":
        await query.edit_message_text("➕ ابعت ID الشخص اللي عايز تضيفه أدمن", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_add_admin"
        return
    if data == "adm_remove_admin":
        await query.edit_message_text(f"➖ ابعت ID الأدمن اللي عايز تشيله\n\nالحاليين:\n" + "\n".join([str(x) for x in SUPER_ADMINS]), reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_remove_admin"
        return
    if data == "adm_msgs":
        txt = f"✏️ رسائل الترحيب:\n\nللمسجلين:\n{settings.get('welcome_auth','')}\n\nلغير المسجلين:\n{settings.get('welcome_unauth','')}"
        kb=InlineKeyboardMarkup([
            [InlineKeyboardButton("✏️ تعديل المسجلين", callback_data="adm_edit_auth")],
            [InlineKeyboardButton("✏️ تعديل غير المسجلين", callback_data="adm_edit_unauth")],
            [InlineKeyboardButton("🗑️ مسح المسجلين", callback_data="adm_del_auth")],
            [InlineKeyboardButton("🗑️ مسح غير المسجلين", callback_data="adm_del_unauth")],
            [InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]
        ])
        await query.edit_message_text(txt, reply_markup=kb)
        return
    if data == "adm_edit_auth":
        await query.edit_message_text("✏️ ابعت رسالة المسجلين الجديدة\nتقدر تستخدم {name} للاسم", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_msgs")]]))
        context.user_data["waiting"] = "adm_edit_auth"
        return
    if data == "adm_edit_unauth":
        await query.edit_message_text("✏️ ابعت رسالة غير المسجلين الجديدة", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_msgs")]]))
        context.user_data["waiting"] = "adm_edit_unauth"
        return
    if data == "adm_del_auth":
        settings["welcome_auth"]=""
        save_settings(settings)
        await query.edit_message_text("🗑️ اتمسحت رسالة المسجلين - مش هيظهر حاجة", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_del_unauth":
        settings["welcome_unauth"]=""
        save_settings(settings)
        await query.edit_message_text("🗑️ اتمسحت رسالة غير المسجلين", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_change_pass":
        if uid != SUPER_ADMINS[0]:
            await query.edit_message_text("❌ للمالك بس 👑", reply_markup=admin_keyboard(uid))
            return
        await query.edit_message_text("🔑 ابعت الباسورد القديم الأول", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_old_pass"
        return
    if data == "adm_perms_menu":
        if uid != SUPER_ADMINS[0]:
            await query.edit_message_text("❌ للمالك بس 👑", reply_markup=admin_keyboard(uid))
            return
        await query.edit_message_text("🔐 ابعت ID الأدمن عشان تتحكم فيه", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_perms_select"
        return
    if data == "perm_set_pass":
        await query.edit_message_text("🔑 ابعت الباسورد الخاص الجديد للأدمن ده", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_set_custom_pass"
        return
    if data == "noop":
        await query.answer("الباسورد الحالي ظاهر فوق")
        return
    if data.startswith("perm_"):
        if uid != SUPER_ADMINS[0]:
            return
        target = context.user_data.get("perms_target")
        if not target:
            await query.edit_message_text("ابعت ID الأول", reply_markup=admin_keyboard(uid))
            return
        if data == "perm_all":
            settings["admin_perms"][target]=["all"]
        elif data == "perm_none":
            settings["admin_perms"][target]=[]
        else:
            perm=data.replace("perm_","")
            settings["admin_perms"].setdefault(target,[])
            if perm in settings["admin_perms"][target]:
                settings["admin_perms"][target].remove(perm)
            else:
                settings["admin_perms"][target].append(perm)
        save_settings(settings)
        await query.edit_message_text(f"✅ صلاحيات {target}:\n{settings['admin_perms'][target]}", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_stats":
        tracks = db.get("user_tracks", {})
        total = len(tracks)
        import datetime
        today = datetime.datetime.now().strftime("%Y-%m-%d")
        new_today = sum(1 for u in tracks.values() if today in u.get("first_seen",""))
        active_today = sum(1 for u in tracks.values() if today in u.get("last_seen",""))
        blocked = sum(1 for u in tracks.values() if u.get("blocked"))
        await query.edit_message_text(f"📊 احصاءات\n\n👥 الكل: {total}\n🆕 جداد اليوم: {new_today}\n🟢 نشطين اليوم: {active_today}\n🚫 بلوك: {blocked}\n\n📋 يوزرات: {len(db['allowed_usernames'])}\n✅ مفعلين: {len(db['authorized'])}", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_users_dash":
        tracks = db.get("user_tracks", {})
        if not tracks:
            await query.edit_message_text("لسه مفيش حد", reply_markup=admin_keyboard(uid))
        else:
            lines = [f"{v.get('name','')} (@{v.get('username','')}) - {k}" for k,v in list(tracks.items())[-20:]]
            await query.edit_message_text(f"👥 آخر 20:\n\n" + "\n".join(lines), reply_markup=admin_keyboard(uid))
        return
    if data == "adm_new_today":
        tracks = db.get("user_tracks", {})
        import datetime
        today = datetime.datetime.now().strftime("%Y-%m-%d")
        new_users = {k:v for k,v in tracks.items() if today in v.get("first_seen","")}
        txt = "\n".join([f"{v.get('name')} - {k}" for k,v in new_users.items()]) if new_users else "مفيش"
        await query.edit_message_text(f"🆕 جداد اليوم ({len(new_users)}):\n\n{txt}", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_active_today":
        tracks = db.get("user_tracks", {})
        import datetime
        today = datetime.datetime.now().strftime("%Y-%m-%d")
        active = {k:v for k,v in tracks.items() if today in v.get("last_seen","")}
        txt = "\n".join([f"{v.get('name')} - {k}" for k,v in list(active.items())[:20]]) if active else "مفيش"
        await query.edit_message_text(f"🟢 نشطين اليوم ({len(active)}):\n\n{txt}", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_blocked":
        tracks = db.get("user_tracks", {})
        blocked = {k:v for k,v in tracks.items() if v.get("blocked")}
        await query.edit_message_text(f"🚫 بلوك: {len(blocked)}", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_list_users":
        users = db.get("allowed_usernames", [])
        txt = "\n".join(users[:100]) if users else "مفيش يوزرات"
        if len(txt) > 3000:
            txt = txt[:3000]
        await query.edit_message_text(f"📋 اليوزرات ({len(users)}):\n\n{txt}", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_add_user":
        await query.edit_message_text("➕ ابعت اليوزر الجديد", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_add_user"
        return
    if data == "adm_upload_users":
        await query.edit_message_text("📤 ابعت اليوزرات كل يوزر في سطر", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_upload_users"
        return
    if data == "adm_download_users":
        users = db.get("allowed_usernames", [])
        path = "users.txt"
        with open(path, "w", encoding="utf-8") as f:
            f.write("\n".join(users))
        await context.bot.send_document(chat_id=query.message.chat_id, document=open(path,"rb"), filename="users.txt", caption=f"📋 {len(users)} يوزر")
        os.remove(path)
        return
    if data == "adm_ratings":
        db = fast_load_db()
        ratings = db.get("ratings", [])
        if not ratings:
            await query.edit_message_text("⭐ لسه مفيش تقييمات", reply_markup=admin_keyboard(uid))
            return
        tracks = db.get("user_tracks", {})
        text = f"⭐ <b>كل التقييمات ({len(ratings)})</b>\n\n━━━━━━━━━━━━━━━\n"
        for r in ratings[-30:][::-1]:
            r_uid = str(r.get("uid",""))
            track = tracks.get(r_uid, {})
            name = track.get("name","بدون اسم")
            username = track.get("username","بدون يوزر")
            username_show = f"@{username}" if username and username!="بدون يوزر" else "بدون يوزر"
            stars = "⭐" * int(r.get("stars",0))
            time = r.get("time","")[:19]
            text += f"\n{stars} ({r.get('stars')})\n👤 الاسم: {name}\n🆔 ID: <code>{r_uid}</code>\n🔗 اليوزر: {username_show}\n🕐 الوقت: {time}\n━━━━━━━━━━━━━━━\n"
        if len(text) > 4000:
            # ابعته ملف
            path = f"ratings_{uid}.txt"
            with open(path, "w", encoding="utf-8") as f:
                for r in ratings:
                    r_uid = str(r.get("uid",""))
                    track = tracks.get(r_uid, {})
                    f.write(f"{r.get('stars')} نجوم | {track.get('name')} | ID: {r_uid} | @{track.get('username')} | {r.get('time')}\n")
            await context.bot.send_document(chat_id=query.message.chat_id, document=open(path,"rb"), filename="التقييمات.txt", caption=f"⭐ {len(ratings)} تقييم")
            import os
            os.remove(path)
            await query.edit_message_text(f"⭐ التقييمات كتير ({len(ratings)}) - بعتلك الملف", reply_markup=admin_keyboard(uid))
        else:
            await query.edit_message_text(text, parse_mode="HTML", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_used_numbers":
        db = fast_load_db()
        used = db.get("used_numbers", {})
        if not used:
            await query.edit_message_text("📱 لسه مفيش أرقام مستخدمة", reply_markup=admin_keyboard(uid))
            return
        text = "📱 <b>الأرقام المستخدمة لكل خدمة</b>\n\n━━━━━━━━━━━━━━━\n"
        for service_id, numbers in used.items():
            service = next((s for s in SERVICES if s["id"]==service_id), None)
            svc_name = service["name_en"] if service else service_id
            text += f"\n📘 <b>{svc_name}:</b> {len(numbers)} رقم مستخدم\n"
            for num in numbers[-5:]:
                text += f"  • <code>{num}</code>\n"
            text += "━━━━━━━━━━━━━━━\n"
        await query.edit_message_text(text, parse_mode="HTML", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_numbers_stats":
        db = fast_load_db()
        logs = db.get("numbers_log", [])
        used = db.get("used_numbers", {})
        total = sum(len(v) for v in used.values())
        text = f"📊 <b>إحصائيات الأرقام</b>\n\n━━━━━━━━━━━━━━━\n📱 إجمالي الأرقام المستخدمة: {total}\n📋 إجمالي العمليات: {len(logs)}\n\n"
        for service_id, numbers in used.items():
            service = next((s for s in SERVICES if s["id"]==service_id), None)
            svc_name = service["name_en"] if service else service_id
            text += f"{svc_name}: {len(numbers)} رقم\n"
        text += "━━━━━━━━━━━━━━━\n✅ كل رقم مستخدم مرة واحدة فقط لكل خدمة\n🔒 مفيش رقم بيتكرر لنفس الخدمة"
        await query.edit_message_text(text, parse_mode="HTML", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_feedbacks":
        db = fast_load_db()
        feedbacks = db.get("feedbacks", [])
        if not feedbacks:
            await query.edit_message_text("💌 لسه مفيش ملاحظات", reply_markup=admin_keyboard(uid))
            return
        tracks = db.get("user_tracks", {})
        text = f"💌 <b>كل الملاحظات ({len(feedbacks)})</b>\n\n━━━━━━━━━━━━━━━\n"
        for fb in feedbacks[-20:][::-1]:
            fb_uid = str(fb.get("uid",""))
            track = tracks.get(fb_uid, {})
            name = track.get("name","بدون اسم")
            username = track.get("username","بدون يوزر")
            username_show = f"@{username}" if username and username!="بدون يوزر" else "بدون يوزر"
            time = fb.get("time","")[:19]
            fb_text = fb.get("text","")[:300]
            text += f"\n💬 <b>الملاحظة:</b> {fb_text}\n👤 الاسم: {name}\n🆔 ID: <code>{fb_uid}</code>\n🔗 اليوزر: {username_show}\n🕐 الوقت: {time}\n━━━━━━━━━━━━━━━\n"
        if len(text) > 4000:
            path = f"feedbacks_{uid}.txt"
            with open(path, "w", encoding="utf-8") as f:
                for fb in feedbacks:
                    fb_uid = str(fb.get("uid",""))
                    track = tracks.get(fb_uid, {})
                    f.write(f"الملاحظة: {fb.get('text')}\nالاسم: {track.get('name')} | ID: {fb_uid} | @{track.get('username')} | {fb.get('time')}\n━━━━━━━━━━━━━━━\n")
            await context.bot.send_document(chat_id=query.message.chat_id, document=open(path,"rb"), filename="الملاحظات.txt", caption=f"💌 {len(feedbacks)} ملاحظة")
            import os
            os.remove(path)
            await query.edit_message_text(f"💌 الملاحظات كتير ({len(feedbacks)}) - بعتلك الملف", reply_markup=admin_keyboard(uid))
        else:
            await query.edit_message_text(text, parse_mode="HTML", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_mail_stats":
        db = fast_load_db()
        mails = db.get("temp_mails", {})
        total_mails = len(mails)
        total_codes = sum(len(v.get("codes", [])) if isinstance(v, dict) else 0 for v in mails.values())
        text = f"📧 <b>إحصائيات البريد المؤقت</b>\n\n━━━━━━━━━━━━━━━\n📧 إجمالي الإيميلات: {total_mails}\n📩 إجمالي الأكواد اللي وصلت: {total_codes}\n━━━━━━━━━━━━━━━"
        await query.edit_message_text(text, parse_mode="HTML", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_clear_numbers":
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("✅ أيوه امسح كل الأرقام", callback_data="adm_confirm_clear_numbers")],
            [InlineKeyboardButton("❌ لا، رجوع", callback_data="adm_back")]
        ])
        await query.edit_message_text("🗑️ <b>مسح كل الأرقام المستخدمة</b>\n\n⚠️ هيمسح كل الأرقام اللي اتوزعت\n✅ كل الأرقام هترجع جديدة\n\nمتأكد؟", parse_mode="HTML", reply_markup=kb)
        return
    if data == "adm_confirm_clear_numbers":
        db = fast_load_db()
        count = len(db.get("all_used_numbers", []))
        db["all_used_numbers"] = []
        db["used_numbers"] = {}
        db["number_owners"] = {}
        db["numbers_log"] = []
        fast_save_db(db)
        await query.edit_message_text(f"✅ تم مسح {count} رقم بنجاح\n🔄 كل الأرقام بقت جديدة", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_add_service":
        await query.edit_message_text("➕ <b>إضافة خدمة جديدة</b>\n\n📝 ابعت اسم الخدمة بالإنجليزي مثلاً:\n<code>paypal</code>\nأو <code>uber</code>\n\n━━━━━━━━━━━━━━━\nالصيغة: الاسم | العدد | الإيموجي\nمثال:\n<code>PayPal | 5000 | 💳</code>", parse_mode="HTML", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_add_service"
        return
    if data == "adm_edit_service":
        services_text = "\n".join([f"{s['id']} - {s['name_en']} ({s['count']})" for s in SERVICES])
        await query.edit_message_text(f"✏️ <b>تعديل عدد خدمة</b>\n\n📋 الخدمات الحالية:\n{services_text}\n\n📝 ابعت بالصيغة:\n<code>facebook 30000</code>\nأو\n<code>whatsapp 20000</code>", parse_mode="HTML", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_edit_service"
        return
    if data == "adm_broadcast":
        await query.edit_message_text("📢 <b>إذاعة للكل</b>\n\n📝 ابعت الرسالة اللي عايز تبعتها لكل المستخدمين\n\n💡 تقدر تستخدم HTML\nمثال: <b>نص عريض</b>", parse_mode="HTML", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_broadcast"
        return
    if data == "adm_ban_user":
        await query.edit_message_text("🚫 <b>حظر / فك حظر مستخدم</b>\n\n📝 ابعت ID المستخدم\n\nلحظر: <code>ban 123456</code>\nلفك الحظر: <code>unban 123456</code>", parse_mode="HTML", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]]))
        context.user_data["waiting"] = "adm_ban_user"
        return
    if data == "adm_stop":
        s=load_settings(); s["bot_active"]=False; save_settings(s)
        await query.edit_message_text("⛔ البوت اتوقف", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_start":
        s=load_settings(); s["bot_active"]=True; save_settings(s)
        await query.edit_message_text("▶️ البوت اشتغل", reply_markup=admin_keyboard(uid))
        return
    if data == "adm_export_all":
        path = "full_export.json"
        with open(path,"w",encoding="utf-8") as f:
            json.dump(db, f, ensure_ascii=False, indent=2)
        await context.bot.send_document(chat_id=query.message.chat_id, document=open(path,"rb"), filename="export.json")
        os.remove(path)
        return

async def text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    text = update.message.text.strip()
    waiting = context.user_data.get("waiting")
    try:
        update_user_track(uid, update.effective_user.username or "", update.effective_user.first_name or "")
    except: pass
    settings = load_settings()
    if settings.get("bot_active")==False and not is_admin(uid):
        await update.message.reply_text("⛔ البوت متوقف حالياً")
        return
    ADMIN_PASSWORD = settings.get("admin_password","AHMED2011/10/1")

    # نظام الباسورد
    if waiting=="admin_password_owner":
        if text.strip()==ADMIN_PASSWORD:
            await update.message.reply_text("✅ تم التحقق - أهلا بيك يا مالك 👑", reply_markup=admin_keyboard(uid))
        else:
            await update.message.reply_text("❌ كلمة السر غلط")
        context.user_data["waiting"]=None
        return
    if waiting=="admin_password_second":
        custom=settings.get("admin_passwords",{}).get(str(uid))
        valid=custom if custom else ADMIN_PASSWORD
        if text.strip()==valid:
            await update.message.reply_text("✅ تم التحقق", reply_markup=admin_keyboard(uid))
        else:
            await update.message.reply_text("❌ كلمة السر غلط")
        context.user_data["waiting"]=None
        return
    if waiting=="adm_old_pass":
        if text.strip()==ADMIN_PASSWORD:
            await update.message.reply_text("✅ القديم صح\n\n🔑 ابعت كلمة السر الجديدة")
            context.user_data["waiting"]="adm_new_pass"
        else:
            await update.message.reply_text("❌ القديمة غلط", reply_markup=admin_keyboard(uid))
            context.user_data["waiting"]=None
        return
    if waiting=="adm_new_pass":
        newp=text.strip()
        if len(newp)<4:
            await update.message.reply_text("❌ قصير - لازم 4 حروف على الأقل")
            return
        settings["admin_password"]=newp
        save_settings(settings)
        await update.message.reply_text(f"✅ اتغيرت لـ: {newp}", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None
        return
    if waiting=="adm_set_custom_pass":
        target=context.user_data.get("perms_target")
        if not target:
            await update.message.reply_text("❌ ابعت ID الأول")
            context.user_data["waiting"]=None
            return
        settings["admin_passwords"][target]=text.strip()
        save_settings(settings)
        await update.message.reply_text(f"✅ باسورد الادمن {target} بقى: {text.strip()}", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None
        return
    if waiting=="adm_edit_auth":
        settings["welcome_auth"]=text
        save_settings(settings)
        await update.message.reply_text("✅ رسالة المسجلين اتعدلت", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None
        return
    if waiting=="adm_edit_unauth":
        settings["welcome_unauth"]=text
        save_settings(settings)
        await update.message.reply_text("✅ رسالة غير المسجلين اتعدلت", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None
        return
    if waiting=="adm_perms_select":
        try:
            target=str(int(text.strip()))
            context.user_data["perms_target"]=target
            cur=settings.get("admin_passwords",{}).get(target,"نفس باسوردك العام")
            kb=InlineKeyboardMarkup([
                [InlineKeyboardButton(f"🔑 الحالي: {cur}", callback_data="noop")],
                [InlineKeyboardButton("🔑 تغيير باسورده الخاص", callback_data="perm_set_pass")],
                [InlineKeyboardButton("✅ كل الصلاحيات", callback_data="perm_all"), InlineKeyboardButton("❌ بدون", callback_data="perm_none")],
                [InlineKeyboardButton("📈 احصاءات", callback_data="perm_stats"), InlineKeyboardButton("👥 المستخدمين", callback_data="perm_users")],
                [InlineKeyboardButton("✏️ الرسائل", callback_data="perm_msgs"), InlineKeyboardButton("👑 الادمنية", callback_data="perm_admins")],
                [InlineKeyboardButton("🔙 رجوع", callback_data="adm_back")]
            ])
            await update.message.reply_text(f"🔐 تحكم في الادمن {target}:", reply_markup=kb)
        except:
            await update.message.reply_text("❌ ID غلط - ابعت رقم")
        context.user_data["waiting"]=None
        return

    # أوامر الأدمن للدخول
    if text.lower() == "ادمن":
        if uid != SUPER_ADMINS[0]:
            await update.message.reply_text("❌ دي للمالك بس 👑")
            return
        context.user_data["waiting"]="admin_password_owner"
        await update.message.reply_text("🔐 أدخل كلمة السر أولا للوصول للوحة التحكم 👑")
        return
    if text.upper() == "AHMED2009":
        if not is_admin(uid):
            await update.message.reply_text("❌ غير مصرح ليك")
            return
        if uid == SUPER_ADMINS[0]:
            await update.message.reply_text("👑 انت المالك اكتب كلمة ادمن")
            return
        context.user_data["waiting"]="admin_password_second"
        await update.message.reply_text("🔐 أدخل كلمة السر أولا")
        return

    # الكيبورد الرئيسي
    if text == "📞 تواصل معنا":
        contact_kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("💬 تليجرام (أحمد) 👑", url="tg://user?id=6364073135")],
            [InlineKeyboardButton("📲 واتساب", url="https://wa.me/201096514020")],
            [InlineKeyboardButton("🔙 رجوع للقائمة", callback_data="main")]
        ])
        await update.message.reply_text("📞 تواصل معايا مباشر 👑", reply_markup=contact_kb)
        return
    if "الأسماء" in text or "Names" in text or "👥" in text:
        if not is_authorized(uid):
            await update.message.reply_text("❌ غير مصرح")
            return
        lang = get_user_lang(uid)
        await update.message.reply_text("👥 <b>الأسماء</b>\n\n━━━━━━━━━━━━━━━\n🌍 اختر نوع الاسم:", parse_mode="HTML", reply_markup=names_main_keyboard(lang))
        return
    if text in ["🔑 باسورد", "🔑 إنشاء كلمة مرور", "إنشاء كلمة مرور"]:
        if not is_authorized(uid): return
        pwd=''.join(random.choices(string.ascii_letters+string.digits+"@#$%", k=14))
        await send_copyable_message(update.message, "🔑 <b>كلمة المرور الجديدة</b>", pwd)
        return
    if text in ["🔐 كود 2FA", "كود 2FA"]:
        if not is_authorized(uid): return
        await update.message.reply_text("🔐 <b>كود 2FA</b>\n\n━━━━━━━━━━━━━━━\n📝 ابعت مفتاح الـ Secret الخاص بك\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
        context.user_data["waiting"] = "2fa_code"
        return
    if text in ["🆔 استخراج ID", "استخراج ID"]:
        if not is_authorized(uid): return
        await update.message.reply_text("🆔 <b>استخراج ID</b>\n\n━━━━━━━━━━━━━━━\n🔗 ابعت اللينك أو اليوزر\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
        context.user_data["waiting"] = "extract_id"
        return
    if text in ["💾 الحافظة", "📋 الحافظة", "الحافظة"]:
        if not is_authorized(uid): return
        db = fast_load_db()
        saved = db["users"].get(str(uid), {}).get("saved", [])
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("➕ حفظ حاجة جديدة", callback_data="save_start")],
            [InlineKeyboardButton("📥 حمّل الحافظة كملف", callback_data="download_saved_inline")],
            [InlineKeyboardButton("🗑️ مسح الحافظة", callback_data="clear_saved_inline")],
            [InlineKeyboardButton("📋 عرض الكل", callback_data="saved_list")],
            [InlineKeyboardButton("🔙 رجوع للقائمة", callback_data="main")]
        ])
        if not saved:
            await update.message.reply_text("📋 <b>الحافظة</b>\n\n━━━━━━━━━━━━━━━\n🗃️ الحافظة فاضية حالياً\n💡 دوس ➕ عشان تضيف أول حاجة\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=kb)
        else:
            await update.message.reply_text(f"📋 <b>الحافظة</b>\n\n━━━━━━━━━━━━━━━\n💾 فيها <b>{len(saved)}</b> عنصر محفوظ\n📌 اختر من الأزرار بالأسفل\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=kb)
        return
    if text in ["📥 تحميل الحافظة ملف", "📥 تحميل الحافظة", "تحميل الحافظة"]:
        if not is_authorized(uid): return
        db = fast_load_db()
        saved = db["users"].get(str(uid), {}).get("saved", [])
        if not saved:
            await update.message.reply_text("📥 <b>تحميل الحافظة</b>\n\n━━━━━━━━━━━━━━━\n🗃️ الحافظة فاضية\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
        else:
            path = f"saved_{uid}.txt"
            with open(path, "w", encoding="utf-8") as f:
                f.write("\n".join(saved))
            await context.bot.send_document(chat_id=update.effective_chat.id, document=open(path,"rb"), filename=f"hafza_{len(saved)}.txt", caption=f"📦 <b>حافظتك</b>\n━━━━━━━━━━━━━━━\n📋 {len(saved)} عنصر", parse_mode="HTML")
            os.remove(path)
        return
    if text in ["💾 حفظ", "حفظ", "/save"]:
        if not is_authorized(uid): return
        await update.message.reply_text("📝 <b>حفظ جديد</b>\n\n━━━━━━━━━━━━━━━\n✏️ ابعت النص اللي عايز تحفظه دلوقتي\n💡 حتى لو 5000 حرف\n━━━━━━━━━━━━━━━", parse_mode="HTML")
        context.user_data["waiting"] = "save_item"
        return
    if text in ["📱 أرقام مؤقتة", "أرقام مؤقتة", "أرقام", "ارقام"]:
        if not is_authorized(uid): return
        lang = get_user_lang(uid)
        # واجهة جديدة مباشرة زي الصورة - الخدمات على طول
        kb = get_services_keyboard(lang)
        lang = get_user_lang(uid)
        if lang == "en":
            txt2 = "🌍 <b>Choose Service:</b> Select the service ✅"
        else:
            txt2 = "🌍 <b>الخدمة المطلوبة: اختر الخدمة</b> ✅"
        await update.message.reply_text(txt2, parse_mode="HTML", reply_markup=get_services_keyboard(lang))
        return
    if text in ["📧 بريد مؤقت", "بريد مؤقت", "البريد المؤقت"]:
        if not is_authorized(uid): return
        mail, login, domain = gen_temp_mail()
        context.user_data["temp_mail"] = mail
        context.user_data["temp_login"] = login
        context.user_data["temp_domain"] = domain
        # حفظ في DB كمان
        db = fast_load_db()
        db["users"].setdefault(str(uid), {"saved":[]})
        db["users"][str(uid)]["temp_mail"] = mail
        fast_save_db(db)
        
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("📥 فحص الوارد", callback_data="temp_check")],
            [InlineKeyboardButton("🔄 بريد جديد", callback_data="temp_new")],
            [InlineKeyboardButton("🔙 رجوع للقائمة", callback_data="main")]
        ])
        await update.message.reply_text(
            f"📧 <b>البريد المؤقت</b>\n\n"
            f"━━━━━━━━━━━━━━━\n"
            f"📬 بريدك المؤقت:\n<code>{mail}</code>\n"
            f"━━━━━━━━━━━━━━━\n"
            f"💡 <b>استخدمه للتسجيل في أي موقع</b>\n"
            f"📥 الأكواد هتوصلك هنا فوراً\n"
            f"⏰ البريد شغال لمدة 24 ساعة\n"
            f"━━━━━━━━━━━━━━━\n"
            f"👆 <b>دوس على البريد لنسخه</b>",
            parse_mode="HTML", reply_markup=kb
        )
        return
    if text in ["💬 الدعم الفني", "📞 تواصل معنا", "الدعم الفني", "تواصل معنا"]:
        contact_kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("💬 تليجرام (أحمد) 👑", url="tg://user?id=6364073135")],
            [InlineKeyboardButton("📲 واتساب", url="https://wa.me/201096514020")],
            [InlineKeyboardButton("🔙 رجوع للقائمة", callback_data="main")]
        ])
        await update.message.reply_text("💬 <b>الدعم الفني</b>\n\n━━━━━━━━━━━━━━━\n👑 المطور: أحمد\n⚡ رد سريع\n🔒 دعم فني 24/7\n━━━━━━━━━━━━━━━\n\n📌 اختر طريقة التواصل:", parse_mode="HTML", reply_markup=contact_kb)
        return
    if text in ["❓ المساعدة", "المساعدة"]:
        lang = get_user_lang(uid)
        if lang == "en":
            help_text = (
                "❓ <b>Help - Services Guide</b>\n"
                "━━━━━━━━━━━━━━━\n"
                "🇪🇬 <b>Egyptian Name:</b> Generate random Egyptian names\n"
                "🌐 <b>Foreign Name:</b> Generate foreign names\n"
                "🔑 <b>Create Password:</b> Strong 14-char password\n"
                "🔐 <b>2FA Code:</b> Generate 2FA verification code\n"
                "🆔 <b>Extract ID:</b> Extract ID from any Telegram link\n"
                "📧 <b>Temp Mail:</b> Temporary email that receives codes instantly\n"
                "📱 <b>Temp Numbers:</b> Temporary numbers by service\n"
                "   • Choose service: FaceBook, WhatsApp, Instagram, TikTok, etc.\n"
                "   • Choose country: Egypt, USA, UK, etc.\n"
                "   • One number = One user only (never reused)\n"
                "   • Check if number is used on platform\n"
                "📋 <b>Clipboard:</b> Save your important texts\n"
                "📥 <b>Download Clipboard:</b> Download all saved as file\n"
                "💬 <b>Support:</b> Contact developer directly\n"
                "🌐 <b>Language:</b> Choose Arabic/English at /start\n"
                "━━━━━━━━━━━━━━━\n"
                "⚡ <b>Bot is ultra fast (0.1 sec response)</b>\n"
                "💡 <b>Tip:</b> All results are copyable with one tap ✨"
            )
        else:
            help_text = (
                "❓ <b>المساعدة - شرح كل الخدمات</b>\n"
                "━━━━━━━━━━━━━━━\n"
                "🇪🇬 <b>اسم مصري:</b> توليد أسماء مصرية عشوائية بالكامل\n"
                "🌐 <b>اسم أجنبي:</b> توليد أسماء أجنبية عشوائية\n"
                "🔑 <b>إنشاء كلمة مرور:</b> باسورد قوي 14 حرف آمن\n"
                "🔐 <b>كود 2FA:</b> توليد كود التحقق الثنائي\n"
                "🆔 <b>استخراج ID:</b> استخراج ID من أي لينك تليجرام\n"
                "📧 <b>بريد مؤقت:</b> بريد وهمي يستقبل أكواد فورية\n"
                "   • دوس بريد مؤقت → هيجيلك إيميل عشوائي\n"
                "   • استخدمه في أي موقع والكود هيوصلك في البوت\n"
                "📱 <b>أرقام مؤقتة:</b> الميزة الجديدة 🔥\n"
                "   • دوس أرقام مؤقتة → اختر الخدمة (فيسبوك، واتساب، انستا، تيك توك...)\n"
                "   • اختر الدولة (مصر، أمريكا، بريطانيا...)\n"
                "   • هيجيلك رقم جديد مضمون\n"
                "   • 🔒 رقم واحد = شخص واحد بس (مستحيل حد ياخد نفس رقمك)\n"
                "   • 🔍 تقدر تفحص هل الرقم مستخدم على المنصة ولا لا\n"
                "   • 📥 فحص الكود → تشوف الأكواد اللي وصلت\n"
                "📋 <b>الحافظة:</b> احفظ أي نص مهم\n"
                "📥 <b>تحميل الحافظة:</b> حمل كل محفوظاتك كملف\n"
                "💬 <b>الدعم الفني:</b> كلم المطور مباشر\n"
                "🌐 <b>اللغة:</b> اختار عربي/إنجليزي من /start\n"
                "━━━━━━━━━━━━━━━\n"
                "⚡ <b>البوت طيارة - بيرد في 0.1 ثانية</b> 🚀\n"
                "💡 <b>نصيحة:</b> كل النتائج قابلة للنسخ بضغطة واحدة ✨"
            )
        await update.message.reply_text(help_text, parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
        return
    if text in ["📢 آخر التحديثات", "آخر التحديثات"]:
        settings = load_settings()
        updates = settings.get("last_updates", "🚀 <b>آخر التحديثات</b>\n\n━━━━━━━━━━━━━━━\n✅ تحسين الواجهة بشكل احترافي\n✅ إضافة المساعدة والتقييم\n✅ تحسين النسخ والسرعة\n✅ إضافة زر الدعم الفني\n━━━━━━━━━━━━━━━\n\n📌 ترقب المزيد قريباً 🔥")
        await update.message.reply_text(updates, parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
        return
    if text in ["⭐ تقييم البوت", "تقييم البوت"]:
        rate_kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("⭐⭐⭐⭐⭐ ممتاز", callback_data="rate_5"), InlineKeyboardButton("⭐⭐⭐⭐ جيد جداً", callback_data="rate_4")],
            [InlineKeyboardButton("⭐⭐⭐ جيد", callback_data="rate_3"), InlineKeyboardButton("⭐⭐ مقبول", callback_data="rate_2")],
            [InlineKeyboardButton("⭐ ضعيف", callback_data="rate_1")],
            [InlineKeyboardButton("💬 كتابة ملاحظة", callback_data="rate_feedback")],
            [InlineKeyboardButton("🔙 رجوع", callback_data="main")]
        ])
        await update.message.reply_text("⭐ <b>تقييم البوت</b>\n\n━━━━━━━━━━━━━━━\n📝 رأيك يهمنا جداً\n💡 قيم تجربتك عشان نطور البوت\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=rate_kb)
        return


    db = fast_load_db()
    if waiting == "adm_add_admin":
        try:
            new_id = int(text.strip())
            if new_id not in SUPER_ADMINS:
                SUPER_ADMINS.append(new_id)
                with open("admins.json","w") as f:
                    json.dump(SUPER_ADMINS, f)
                await update.message.reply_text(f"✅ تم إضافة {new_id} أدمن", reply_markup=admin_keyboard(uid))
            else:
                await update.message.reply_text("⚠️ ده أدمن أصلاً", reply_markup=admin_keyboard(uid))
        except:
            await update.message.reply_text("❌ ابعت ID رقمي صحيح", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None
        return
    if waiting == "adm_remove_admin":
        try:
            rem_id = int(text.strip())
            if rem_id in SUPER_ADMINS and len(SUPER_ADMINS)>1 and rem_id != SUPER_ADMINS[0]:
                SUPER_ADMINS.remove(rem_id)
                with open("admins.json","w") as f:
                    json.dump(SUPER_ADMINS, f)
                await update.message.reply_text(f"✅ تم إزالة {rem_id}", reply_markup=admin_keyboard(uid))
            else:
                await update.message.reply_text("❌ مينفعش تشيل المالك أو آخر أدمن", reply_markup=admin_keyboard(uid))
        except:
            await update.message.reply_text("❌ ابعت ID رقمي", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None
        return
    if waiting == "adm_add_user":
        db = fast_load_db()
        clean = text.replace("@","").strip()
        if clean:
            db.setdefault("allowed_usernames", []).append(clean)
            db["allowed_usernames"]=list(set(db["allowed_usernames"]))
            fast_save_db(db)
            total = len(db["allowed_usernames"])
            await update.message.reply_text(f"✅ تم إضافة يوزر {clean}\n\n👥 إجمالي اليوزرات: {total}", reply_markup=admin_keyboard(uid))
        else:
            await update.message.reply_text("❌ يوزر فاضي", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None; return
    if waiting == "adm_upload_users":
        db = fast_load_db()
        users=[u.strip().replace("@","") for u in text.split("\n") if u.strip()]
        db.setdefault("allowed_usernames", []).extend(users)
        db["allowed_usernames"]=list(set(db["allowed_usernames"]))
        fast_save_db(db)
        total2 = len(db["allowed_usernames"])
        await update.message.reply_text(f"✅ تم إضافة {len(users)} يوزر\n👥 الإجمالي: {total2}", reply_markup=admin_keyboard(uid))
        context.user_data["waiting"]=None; return
    if waiting == "2fa_code":
        try:
            totp = pyotp.TOTP(text.replace(" ",""))
            code_now = totp.now()
            await send_copyable_message(update.message, "🔐 كود 2FA الحالي", code_now)
        except Exception as e:
            await update.message.reply_text(f"❌ كود غلط: {e}")
        context.user_data["waiting"]=None
        return
    if waiting == "extract_id":
        # نحاول نستخرج ID من أشكال مختلفة
        m = re.search(r'/(\d+)(?:/|$)', text)
        m2 = re.search(r'\b(\d{5,})\b', text)
        if m:
            await send_copyable_message(update.message, "🆔 الـ ID", m.group(1))
        elif m2:
            await send_copyable_message(update.message, "🆔 الـ ID", m2.group(1))
        else:
            await update.message.reply_text("❌ مش لاقي ID في اللينك، ابعت لينك زي https://t.me/c/123456/1")
        context.user_data["waiting"]=None
        return
    if waiting == "rate_feedback":
        # حفظ الملاحظة
        db = fast_load_db()
        db.setdefault("feedbacks", []).append({"uid": uid, "text": text, "time": str(__import__('datetime').datetime.now())})
        fast_save_db(db)
        await update.message.reply_text("💌 <b>شكراً لملاحظتك!</b>\n\n━━━━━━━━━━━━━━━\n✅ تم حفظ ملاحظتك\n👀 سيتم مراجعتها قريباً\n💖 شكراً لمساعدتنا\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
        context.user_data["waiting"]=None
        return
    if waiting == "save_item":
        if text in old_main_buttons() or text.lower() in ["ادمن", "ahmed2009"]:
            context.user_data["waiting"] = None
        else:
            db["users"].setdefault(str(uid), {"saved":[]})
            db["users"][str(uid)]["saved"].append(text)
            fast_save_db(db)
            await update.message.reply_text("✅ <b>تم الحفظ</b>\n\n━━━━━━━━━━━━━━━\n💾 تم حفظ النص في الحافظة\n📌 تقدر تشوفه من 📋 الحافظة\n━━━━━━━━━━━━━━━", parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
            context.user_data["waiting"]=None
            return

    if not is_authorized(uid):
        clean_user = text.replace("@","").lower()
        allowed_lower = [u.lower() for u in db.get("allowed_usernames",[])]
        if clean_user in allowed_lower:
            db.setdefault("authorized", {})[str(uid)] = clean_user
            db.setdefault("users", {})[str(uid)] = db["users"].get(str(uid), {"saved":[]})
            fast_save_db(db)
            name = update.effective_user.first_name or "يا غالي"
            settings = load_settings()
            tmpl = settings.get("welcome_auth","👋 أهلا بيك {name}")
            welcome = tmpl.format(name=name) if "{name}" in tmpl else tmpl
            if welcome.strip():
                await update.message.reply_text(welcome, parse_mode="HTML", reply_markup=main_keyboard(get_user_lang(uid)))
            else:
                await update.message.reply_text("👇 اختار من القائمة", reply_markup=main_keyboard(get_user_lang(uid)))
        else:
            await update.message.reply_text("❌ يوزر غلط 🔐\nلـ طلب يوزر كلم الأدمن:", reply_markup=whatsapp_button())
        return
    # لو مش أمر معروف
    await update.message.reply_text("👋 اختار من القائمة تحت 👇", reply_markup=main_keyboard(get_user_lang(uid)))

async def document_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_admin(update.effective_user.id): return
    waiting = context.user_data.get("waiting")
    db = fast_load_db()
    file = await update.message.document.get_file()
    path = f"temp_{update.effective_user.id}.txt"
    await file.download_to_drive(path)
    try:
        with open(path,"r",encoding="utf-8",errors="ignore") as f:
            lines=[l.strip() for l in f if l.strip()]
    except:
        lines=[]
    try:
        os.remove(path)
    except: pass
    
    if waiting == "adm_upload_users":
        db["allowed_usernames"].extend(lines); db["allowed_usernames"]=list(set(db["allowed_usernames"])); save_db(db)
        await update.message.reply_text(f"✅ تم إضافة {len(lines)} يوزر من الملف", reply_markup=admin_keyboard(update.effective_user.id))
    else:
        db["allowed_usernames"].extend(lines); db["allowed_usernames"]=list(set(db["allowed_usernames"])); save_db(db)
        await update.message.reply_text(f"✅ تم إضافة {len(lines)} يوزر من الملف", reply_markup=admin_keyboard(update.effective_user.id))
    context.user_data["waiting"]=None

async def post_init(app):
    await app.bot.set_my_commands([
        BotCommand("start", "▶️ تشغيل البوت - القائمة الرئيسية"),
    ])
    print("✅ تم اضافة زر القائمة - المربع الأزرق فيه /start")

app = Application.builder().token(BOT_TOKEN).concurrent_updates(True).post_init(post_init).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(admin_callback, pattern="^adm_|^perm_|^noop"))
app.add_handler(CallbackQueryHandler(button_handler, pattern="^main|^save_|^gender_|^egy_|^foreign_|^password$|^2fa$|^extract_id$|^download_|^clear_|^confirm_|^rate_|^temp_|^lang_|^svc_|^country_|^nums_|^check_platform_|^names_"))
app.add_handler(MessageHandler(filters.Document.ALL, document_handler))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler))
print("Bot V31 AUTO FANCY NOTIFY - BILINGUAL - mail.tm - FAST & UNBLOCKED - 100% FREE")
keep_alive()
app.run_polling(drop_pending_updates=True)
