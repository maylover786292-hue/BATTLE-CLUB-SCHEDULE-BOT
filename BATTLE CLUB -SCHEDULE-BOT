import sys
import os
import re
import time
import calendar
import threading
import json
from datetime import datetime, timezone, timedelta
from flask import Flask, jsonify
import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton, MessageEntity

TOKEN = os.environ.get('BOT_TOKEN', '8986851160:AAFH8By21ZBKq5QYHSB_lM713IbJSS1AE2g')
PASSWORD = os.environ.get('BOT_PASSWORD', 'monirahmed@142761?+')
DATA_FILE = 'bot_data.json'

# Bangladesh Timezone (UTC+6)
BD_TZ = timezone(timedelta(hours=6))

# Zero-Latency & Fast Connection Keep-Alive Setup
telebot.apihelper.SESSION_TIME_TO_LIVE = 5 * 60
bot = telebot.TeleBot(TOKEN, threaded=True, num_threads=10)
DATA_LOCK = threading.RLock()

COLOR_PALETTE = {
    "red": ("🔴 লাল (Red)", "danger"),
    "blue": ("🔵 নীল (Blue)", "primary"),
    "green": ("🟢 সবুজ (Green)", "success"),
    "none": ("⚪ ডিফল্ট (No Color)", None)
}

WEEKDAYS_BN = [
    (5, "শনিবার (Saturday)"),
    (6, "রবিবার (Sunday)"),
    (0, "সোমবার (Monday)"),
    (1, "মঙ্গলবার (Tuesday)"),
    (2, "বুধবার (Wednesday)"),
    (3, "বৃহস্পতিবার (Thursday)"),
    (4, "শুক্রবার (Friday)")
]
WEEKDAY_MAP = dict(WEEKDAYS_BN)

MONTH_LIST = [
    (1, "জানুয়ারি (Jan)"),
    (2, "ফেব্রুয়ারি (Feb)"),
    (3, "মার্চ (Mar)"),
    (4, "এপ্রিল (Apr)"),
    (5, "মে (May)"),
    (6, "জুন (Jun)"),
    (7, "জুলাই (Jul)"),
    (8, "আগস্ট (Aug)"),
    (9, "সেপ্টেম্বর (Sep)"),
    (10, "অক্টোবর (Oct)"),
    (11, "নভেম্বর (Nov)"),
    (12, "ডিসেম্বর (Dec)")
]

MONTH_NAME_MAP = {
    "jan": 1, "january": 1, "জানুয়ারি": 1, "জানুয়ারি": 1,
    "feb": 2, "february": 2, "ফেব্রুয়ারি": 2, "ফেব্রুয়ারি": 2,
    "mar": 3, "march": 3, "মার্চ": 3,
    "apr": 4, "april": 4, "এপ্রিল": 4,
    "may": 5, "মে": 5,
    "jun": 6, "june": 6, "জুন": 6,
    "jul": 7, "july": 7, "জুলাই": 7,
    "aug": 8, "august": 8, "আগস্ট": 8,
    "sep": 9, "sept": 9, "september": 9, "সেপ্টেম্বর": 9,
    "oct": 10, "october": 10, "অক্টোবর": 10,
    "nov": 11, "november": 11, "নভেম্বর": 11,
    "dec": 12, "december": 12, "ডিসেম্বর": 12
}

BN_DIGITS = str.maketrans("০১২৩৪৫৬৭৮৯", "0123456789")

def norm_digits(text):
    return (text or "").translate(BN_DIGITS).strip()

class CustomButton(InlineKeyboardButton):
    def __init__(self, text, style=None, **kwargs):
        super().__init__(text, **kwargs)
        self.style = style
    def to_dict(self):
        d = super().to_dict()
        if self.style:
            d['style'] = self.style
        return d

def make_btn(text, color_key="none", **kwargs):
    _, api_style = COLOR_PALETTE.get(color_key, ("⚪ ডিফল্ট (No Color)", None))
    return CustomButton(text, style=api_style, **kwargs)

def ack(cid, text=None, show_alert=False):
    try:
        if text:
            bot.answer_callback_query(cid, text, show_alert=show_alert)
        else:
            bot.answer_callback_query(cid)
    except Exception:
        pass

DEFAULT_TEMPLATES = {
    "main_menu": {
        "text": (
            "👑 ব্যাটল ক্লাব প্রো কন্ট্রোল প্যানেল 👑\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "⚡ ইঞ্জিন স্ট্যাটাস: {status}\n"
            "🕒 বর্তমান সময়: {now_bd}\n"
            "⏸️ বিরতি স্ট্যাটাস: {pause_status}\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "📊 অ্যাক্টিভ শিডিউল: মোট {total_sched} টি\n"
            "⏰ ডেইলি শিডিউল: {daily_cnt} টি\n"
            "🔄 ইন্টারভ্যাল শিডিউল: {int_cnt} টি\n"
            "🎯 ওয়ান-টাইম শিডিউল: {once_cnt} টি\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "🔗 ডিফল্ট বাটন: {def_cnt} টি\n"
            "🗑️ ডিলিট ফাইল (রিসাইকেল বিন): {recycle_cnt} টি\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "👇 নিচের আলাদা সেক্টর থেকে আপনার অপশন বেছে নিন:"
        ),
        "required": ["{status}", "{now_bd}", "{pause_status}", "{total_sched}", "{daily_cnt}", "{int_cnt}", "{once_cnt}", "{def_cnt}", "{recycle_cnt}"]
    },
    "reaction_hub": {
        "text": (
            "👍 রিঅ্যাকশন ও বাটন কন্ট্রোল হাব\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "🔔 নতুন শিডিউলে অটো রিঅ্যাকশন: {global_rx}\n"
            "👍 রিঅ্যাকশন চালু থাকা শিডিউল: {rx_on_count}/{total_sched} টি\n"
            "⚙️ ডিফল্ট বাটন চালু থাকা শিডিউল: {def_on_count}/{total_sched} টি\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "💡 এখান থেকে ১ ক্লিকেই আপনার সব শিডিউলের নিচে লাইক/রিঅ্যাকশন বাটন (👍 ❤️ 🔥 😍) এবং ডিফল্ট বাটন চালু করতে পারবেন:"
        ),
        "required": ["{global_rx}", "{rx_on_count}", "{def_on_count}", "{total_sched}"]
    },
    "default_btns": {
        "text": (
            "⚙️ ডিফল্ট বাটন সেটআপ (সব পোস্টের নিচে অটোমেটিক থাকবে)\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "{def_btn_list}\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "👇 নিচের বাটন থেকে নতুন ডিফল্ট বাটন যোগ বা রিমুভ করুন:"
        ),
        "required": ["{def_btn_list}"]
    },
    "list_schedule": {
        "text": (
            "📋 অ্যাক্টিভ শিডিউল তালিকা (মোট {total_sched} টি)\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "👇 যে শিডিউলের বিস্তারিত দেখতে বা এডিট করতে চান তার উপর ক্লিক করুন:"
        ),
        "required": ["{total_sched}"]
    },
    "recycle_bin": {
        "text": (
            "🗑️ ডিলিট ফাইল / রিসাইকেল বিন (মোট {recycle_cnt} টি)\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "👇 যে শিডিউলটি রিস্টোর বা স্থায়ীভাবে ডিলিট করতে চান তার উপর ক্লিক করুন:"
        ),
        "required": ["{recycle_cnt}"]
    },
    "pause_menu": {
        "text": (
            "⏸️ শিডিউল বিরতি (Pause) কন্ট্রোল প্যানেল\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "📡 বর্তমান অবস্থা: {pause_now}\n"
            "📋 সেট করা বিরতি নিয়ম: মোট {pause_cnt} টি\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "{pause_list}\n"
            "━━━━━━━━━━━━━━━━━━━━━━\n"
            "👇 কোন সপ্তাহে বা কোন তারিখে কয়টা থেকে কয়টা পর্যন্ত শিডিউল বন্ধ রাখতে চান তা সেট করুন:"
        ),
        "required": ["{pause_now}", "{pause_cnt}", "{pause_list}"]
    },
    "save_welcome": {
        "text": "✅ নতুন মেসেজ (বাটন ও প্রিমিয়াম লাইভ ইমোজি সহ) সফলভাবে সেভ হয়েছে!",
        "required": []
    }
}

default_data = {
    "auth_users": [],
    "schedules": {},
    "recycle_bin": {},
    "default_buttons": [],
    "reactions_data": {},
    "pause_rules": [],
    "ui_templates": {},
    "default_reactions_on": True,
    "is_running": True,
    "btn_colors": {
        "add_schedule": "green",
        "list_schedule": "blue",
        "default_btns": "blue",
        "reaction_hub": "green",
        "quick_post": "green",
        "recycle_bin": "red",
        "pause_menu": "blue",
        "ui_editor": "green",
        "color_menu": "blue",
        "toggle_bot": "blue",
        "restart_bot": "red"
    }
}

def clean_url(url_str):
    url = (url_str or "").strip()
    if not url:
        return "https://t.me/"
    url = url.split()[0]
    if url.startswith("@"):
        return "https://t.me/" + url[1:]
    if not (url.startswith("http://") or url.startswith("https://") or url.startswith("tg://")):
        url = "https://" + url
    return url

def load_data():
    with DATA_LOCK:
        if os.path.exists(DATA_FILE):
            try:
                with open(DATA_FILE, 'r', encoding='utf-8') as f:
                    saved = json.load(f)
                    for k, v in default_data.items():
                        if k not in saved:
                            saved[k] = v
                    for col_k, col_v in default_data["btn_colors"].items():
                        saved.setdefault("btn_colors", {}).setdefault(col_k, col_v)
                    for box in ("schedules", "recycle_bin"):
                        for sid, s in saved.get(box, {}).items():
                            s["forward_mode"] = False
                            if "reactions" not in s:
                                s["reactions"] = True
                            for b in s.get("buttons", []):
                                if "url" in b:
                                    b["url"] = clean_url(b["url"])
                    for b in saved.get("default_buttons", []):
                        if "url" in b:
                            b["url"] = clean_url(b["url"])
                    return saved
            except Exception:
                return default_data.copy()
        return default_data.copy()

def save_data(d):
    with DATA_LOCK:
        try:
            with open(DATA_FILE, 'w', encoding='utf-8') as f:
                json.dump(d, f, indent=4, ensure_ascii=False)
        except Exception as e:
            print(f"Data save error: {e}")

data = load_data()
save_data(data)
temp_schedule = {}

def get_c(key, default="none"):
    return data.get("btn_colors", {}).get(key, default)

def is_auth(user_id):
    return user_id in data.get("auth_users", [])

def utf16_len(s):
    return len((s or "").encode('utf-16-le')) // 2

def parse_entities(raw_list):
    if not raw_list:
        return None
    result = []
    for e in raw_list:
        try:
            if isinstance(e, dict):
                result.append(MessageEntity.de_json(json.dumps(e)))
            else:
                result.append(MessageEntity.de_json(e))
        except Exception:
            pass
    return result if result else None

def render_template_with_entities(tpl_text, raw_entities, replacements):
    if not raw_entities:
        out = tpl_text
        for k, v in replacements.items():
            out = out.replace(k, str(v))
        return out, None
    ents = [dict(e) for e in raw_entities]
    cur_text = tpl_text
    for placeholder, val_str in replacements.items():
        val_str = str(val_str)
        while placeholder in cur_text:
            idx = cur_text.find(placeholder)
            u_off = utf16_len(cur_text[:idx])
            u_old = utf16_len(placeholder)
            u_new = utf16_len(val_str)
            diff = u_new - u_old
            new_ents = []
            for e in ents:
                e_off = e["offset"]
                e_len = e["length"]
                e_end = e_off + e_len
                if e_end <= u_off:
                    new_ents.append(e)
                elif e_off >= u_off + u_old:
                    e["offset"] += diff
                    new_ents.append(e)
                elif e_off <= u_off and e_end >= u_off + u_old:
                    e["length"] += diff
                    new_ents.append(e)
            ents = new_ents
            cur_text = cur_text[:idx] + val_str + cur_text[idx + len(placeholder):]
    return cur_text, parse_entities(ents)

def smart_send_or_edit(chat_id, msg_id, text, reply_markup=None, parse_mode="Markdown", entities=None):
    if entities:
        if msg_id:
            try:
                return bot.edit_message_text(
                    text, chat_id, msg_id,
                    reply_markup=reply_markup,
                    entities=entities
                )
            except Exception:
                pass
        try:
            return bot.send_message(chat_id, text, reply_markup=reply_markup, entities=entities)
        except Exception:
            pass
    if msg_id:
        try:
            return bot.edit_message_text(
                text, chat_id, msg_id,
                reply_markup=reply_markup,
                parse_mode=parse_mode
            )
        except Exception:
            pass
    try:
        return bot.send_message(chat_id, text, reply_markup=reply_markup, parse_mode=parse_mode)
    except Exception:
        return bot.send_message(chat_id, text, reply_markup=reply_markup, parse_mode=None)

def render_ui_page(page_key, replacements):
    custom = data.get("ui_templates", {}).get(page_key)
    default_tpl = DEFAULT_TEMPLATES.get(page_key, {}).get("text", "")
    req_keys = DEFAULT_TEMPLATES.get(page_key, {}).get("required", [])
    if custom and custom.get("text"):
        tpl_text = custom["text"]
        raw_ents = custom.get("entities")
        missing = [k for k in req_keys if k not in tpl_text]
        if missing:
            tpl_text = tpl_text + "\n" + "\n".join(missing)
        return render_template_with_entities(tpl_text, raw_ents, replacements)
    else:
        out = default_tpl
        for k, v in replacements.items():
            out = out.replace(k, str(v))
        return out, None

def send_ui_page(chat_id, msg_id, page_key, replacements, markup):
    txt, ents = render_ui_page(page_key, replacements)
    pm = None if ents or data.get("ui_templates", {}).get(page_key) else "Markdown"
    return smart_send_or_edit(chat_id, msg_id, txt, reply_markup=markup, parse_mode=pm, entities=ents)

def to_minutes_since_midnight(hh_mm):
    try:
        parts = hh_mm.split(":")
        return int(parts[0]) * 60 + int(parts[1])
    except Exception:
        return None

def is_within_interval_window(now_str, start_time, end_time):
    if not start_time and not end_time:
        return True
    now_m = to_minutes_since_midnight(now_str)
    if now_m is None:
        return True
    start_m = to_minutes_since_midnight(start_time) if start_time else 0
    end_m = to_minutes_since_midnight(end_time) if end_time else 1439
    if start_m <= end_m:
        return start_m <= now_m <= end_m
    else:
        return now_m >= start_m or now_m <= end_m

def format_display_time(hh_mm):
    if not hh_mm:
        return "সেট করা নেই"
    try:
        dt = datetime.strptime(hh_mm, "%H:%M")
        return f"{dt.strftime('%I:%M %p')} ({hh_mm})"
    except Exception:
        return hh_mm

def format_display_date(yyyy_mm_dd):
    if not yyyy_mm_dd:
        return "সেট করা নেই"
    try:
        dt = datetime.strptime(yyyy_mm_dd, "%Y-%m-%d")
        m_name = dict(MONTH_LIST).get(dt.month, str(dt.month))
        return f"{dt.day} {m_name}, {dt.year} ({yyyy_mm_dd})"
    except Exception:
        return yyyy_mm_dd

def format_pause_rule(r, idx=None):
    prefix = f"{idx}. " if idx is not None else ""
    st = format_display_time(r.get("start_time", "00:00"))
    et = format_display_time(r.get("end_time", "23:59"))
    if r.get("type") == "weekly":
        day_name = WEEKDAY_MAP.get(r.get("weekday", 4), "শুক্রবার")
        return f"{prefix}🗓️ প্রতি {day_name}: {st} থেকে {et}"
    else:
        d_str = format_display_date(r.get("date", ""))
        return f"{prefix}📅 তারিখ {d_str}: {st} থেকে {et}"

def get_active_pause_rule(now_dt=None):
    if now_dt is None:
        now_dt = datetime.now(BD_TZ)
    now_str = now_dt.strftime("%H:%M")
    today_str = now_dt.strftime("%Y-%m-%d")
    wk = now_dt.weekday()
    for r in data.get("pause_rules", []):
        if not r.get("enabled", True):
            continue
        rtype = r.get("type")
        st = r.get("start_time", "00:00")
        et = r.get("end_time", "23:59")
        if rtype == "weekly" and r.get("weekday") == wk:
            if is_within_interval_window(now_str, st, et):
                return r
        elif rtype == "date" and r.get("date") == today_str:
            if is_within_interval_window(now_str, st, et):
                return r
    return None

def parse_user_time(t_input):
    if not t_input:
        return None
    cleaned = norm_digits(t_input).upper().replace(".", ":")
    for fmt in ("%I:%M %p", "%I:%M%p", "%H:%M", "%I %p", "%I%p"):
        try:
            return datetime.strptime(cleaned, fmt).strftime("%H:%M")
        except Exception:
            pass
    return None

def parse_month_value(m_input):
    if not m_input:
        return None
    cleaned = norm_digits(m_input).lower().strip()
    if cleaned.isdigit():
        m = int(cleaned)
        if 1 <= m <= 12:
            return m
    return MONTH_NAME_MAP.get(cleaned)

def parse_full_once_datetime(raw_text):
    if not raw_text:
        return None
    txt = norm_digits(raw_text)
    now_bd = datetime.now(BD_TZ)
    m_iso = re.match(r"^(\d{4})[-/.](\d{1,2})[-/.](\d{1,2})[\s,]+(.+)$", txt)
    if m_iso:
        yr, mo, dy, t_part = int(m_iso.group(1)), int(m_iso.group(2)), int(m_iso.group(3)), m_iso.group(4)
        parsed_t = parse_user_time(t_part)
        if parsed_t and 1 <= mo <= 12:
            max_d = calendar.monthrange(yr, mo)[1]
            if 1 <= dy <= max_d:
                return f"{yr:04d}-{mo:02d}-{dy:02d}", parsed_t
    m_named = re.match(r"^([A-Za-z\u0980-\u09FF]+)[\s.,/-]+(\d{1,2})(?:[\s.,/-]+(\d{4}))?[\s.,/-]+(.+)$", txt)
    if m_named:
        mo = parse_month_value(m_named.group(1))
        dy = int(m_named.group(2))
        yr = int(m_named.group(3)) if m_named.group(3) else now_bd.year
        t_part = m_named.group(4)
        parsed_t = parse_user_time(t_part)
        if mo and parsed_t:
            max_d = calendar.monthrange(yr, mo)[1]
            if 1 <= dy <= max_d:
                return f"{yr:04d}-{mo:02d}-{dy:02d}", parsed_t
    m_dmy = re.match(r"^(\d{1,2})[\s.,/-]+([A-Za-z\u0980-\u09FF]+)(?:[\s.,/-]+(\d{4}))?[\s.,/-]+(.+)$", txt)
    if m_dmy:
        dy = int(m_dmy.group(1))
        mo = parse_month_value(m_dmy.group(2))
        yr = int(m_dmy.group(3)) if m_dmy.group(3) else now_bd.year
        t_part = m_dmy.group(4)
        parsed_t = parse_user_time(t_part)
        if mo and parsed_t:
            max_d = calendar.monthrange(yr, mo)[1]
            if 1 <= dy <= max_d:
                return f"{yr:04d}-{mo:02d}-{dy:02d}", parsed_t
    return None

def format_schedule_summary(s, short=False):
    mode = s.get("mode", "daily")
    if mode == "daily":
        t_disp = format_display_time(s.get("time", ""))
        return f"ডেইলি {s.get('time', '')}" if short else f"প্রতিদিন {t_disp} (আজীবন চলবে)"
    elif mode == "once":
        t_disp = format_display_time(s.get("time", ""))
        once_date = s.get("once_date", "")
        if once_date:
            d_disp = format_display_date(once_date)
            return f"একবার {once_date} {s.get('time', '')}" if short else f"নির্দিষ্ট তারিখ: {d_disp}\n🕒 সময়: {t_disp} (শুধু ১ বার পোস্ট হয়ে ডিলিট ফাইলে যাবে)"
        return f"একবার {s.get('time', '')}" if short else f"নির্দিষ্ট সময় {t_disp} (শুধু ১ বার পোস্ট হবে)"
    else:
        mins = s.get("interval", 15)
        int_lbl = "১ ঘণ্টা (৬০ মি.)" if mins == 60 else f"{mins} মিনিট"
        st = s.get("start_time")
        et = s.get("end_time")
        rep = s.get("interval_repeat", "daily")
        rep_lbl = "প্রতিদিন (আজীবন)" if rep == "daily" else "শুধু ১ দিনের জন্য"
        rep_short = "আজীবন" if rep == "daily" else "১ দিন"
        if st and et:
            win_lbl = f"{format_display_time(st)} থেকে {format_display_time(et)}"
            win_short = f"{st}-{et}"
        elif st:
            win_lbl = f"{format_display_time(st)} থেকে শুরু"
            win_short = f"{st}~"
        else:
            win_lbl = "২৪ ঘণ্টা (এখন থেকেই)"
            win_short = "২৪ ঘণ্টা"
        if short:
            return f"প্রতি {mins}মি. ({win_short} | {rep_short})"
        return f"প্রতি {int_lbl} পর পর\n🕒 সময়সীমা: {win_lbl}\n📅 মেয়াদ: {rep_lbl}"

@bot.message_handler(commands=['start'])
def start(message):
    user_id = message.from_user.id
    if not is_auth(user_id):
        msg = bot.reply_to(message, "🔒 বটটি সুরক্ষিত। ব্যবহার করতে দয়া করে পাসওয়ার্ড দিন:")
        bot.register_next_step_handler(msg, check_password)
    else:
        bot.clear_step_handler_by_chat_id(message.chat.id)
        show_main_menu(message.chat.id)

def check_password(message):
    if message.text == PASSWORD:
        if message.from_user.id not in data["auth_users"]:
            data["auth_users"].append(message.from_user.id)
            save_data(data)
        bot.reply_to(message, "✅ পাসওয়ার্ড সঠিক! ব্যাটল ক্লাব প্রো কন্ট্রোল প্যানেলে স্বাগতম।")
        show_main_menu(message.chat.id)
    else:
        bot.reply_to(message, "❌ ভুল পাসওয়ার্ড! আবার /start লিখে চেষ্টা করুন।")

def show_main_menu(chat_id, msg_id=None):
    schedules = data.get("schedules", {})
    recycle = data.get("recycle_bin", {})
    def_btns = data.get("default_buttons", [])
    pause_rules = data.get("pause_rules", [])
    daily_cnt = sum(1 for s in schedules.values() if s.get("mode") == "daily")
    int_cnt = sum(1 for s in schedules.values() if s.get("mode") == "interval")
    once_cnt = sum(1 for s in schedules.values() if s.get("mode") == "once")
    now_bd = datetime.now(BD_TZ).strftime("%I:%M %p")
    active_pause = get_active_pause_rule()
    if not data.get("is_running", True):
        status = "বন্ধ 🔴 (পোস্টিং স্থগিত)"
    elif active_pause:
        status = "বিরতি চলছে ⏸️ (শিডিউল পজ)"
    else:
        status = "চালু 🟢"
    pause_status = format_pause_rule(active_pause) if active_pause else f"কোনো বিরতি চলছে না (মোট নিয়ম: {len(pause_rules)}টি)"
    markup = InlineKeyboardMarkup(row_width=2)
    markup.row(
        make_btn("➕ নতুন শিডিউল তৈরি", get_c("add_schedule", "green"), callback_data="add_schedule"),
        make_btn(f"📋 শিডিউল তালিকা ({len(schedules)})", get_c("list_schedule", "blue"), callback_data="list_schedule")
    )
    markup.row(
        make_btn(f"⚙️ ডিফল্ট বাটন ({len(def_btns)}টি)", get_c("default_btns", "blue"), callback_data="default_btns"),
        make_btn("👍 রিঅ্যাকশন ও বাটন হাব", get_c("reaction_hub", "green"), callback_data="reaction_hub")
    )
    markup.row(
        make_btn("🚀 ইনস্ট্যান্ট টেস্ট পোস্ট", get_c("quick_post", "green"), callback_data="quick_post"),
        make_btn(f"🗑️ ডিলিট ফাইল ({len(recycle)})", get_c("recycle_bin", "red"), callback_data="recycle_bin")
    )
    markup.row(
        make_btn(f"⏸️ শিডিউল বিরতি ({len(pause_rules)})", get_c("pause_menu", "blue"), callback_data="pause_menu"),
        make_btn("✏️ স্ট্যাটাস ও ইমোজি এডিট", get_c("ui_editor", "green"), callback_data="ui_editor_menu")
    )
    markup.row(
        make_btn("🎨 প্যানেল কালার সেটিং", get_c("color_menu", "blue"), callback_data="color_menu"),
        make_btn("⚡ বট অন/অফ করুন", get_c("toggle_bot", "blue"), callback_data="toggle_bot")
    )
    markup.row(
        make_btn("🔄 ড্যাশবোর্ড রিফ্রেশ", "none", callback_data="main_menu"),
        make_btn("🔄 বট রিস্টার্ট করুন", get_c("restart_bot", "red"), callback_data="restart_bot")
    )
    replacements = {
        "{status}": status,
        "{now_bd}": now_bd,
        "{pause_status}": pause_status,
        "{total_sched}": len(schedules),
        "{daily_cnt}": daily_cnt,
        "{int_cnt}": int_cnt,
        "{once_cnt}": once_cnt,
        "{def_cnt}": len(def_btns),
        "{recycle_cnt}": len(recycle)
    }
    send_ui_page(chat_id, msg_id, "main_menu", replacements, markup)

def show_pause_menu(chat_id, msg_id=None):
    rules = data.get("pause_rules", [])
    active_p = get_active_pause_rule()
    pause_now = f"বিরতি চালু আছে ⏸️ ({format_pause_rule(active_p)})" if active_p else "স্বাভাবিক (কোনো বিরতি চলছে না 🟢)"
    if not rules:
        p_list = "বর্তমানে কোনো সাপ্তাহিক বা তারিখের বিরতি সেট করা নেই।"
    else:
        lines = []
        for idx, r in enumerate(rules, 1):
            lines.append(format_pause_rule(r, idx))
        p_list = "\n".join(lines)
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        make_btn("🗓️ সাপ্তাহিক বিরতি যোগ করুন (যেমন: শুক্রবার ১২:০০ - ২:৩০)", "green", callback_data="add_pause_weekly"),
        make_btn("📅 নির্দিষ্ট তারিখের বিরতি যোগ করুন (তারিখ ও সময়)", "blue", callback_data="add_pause_date")
    )
    for idx, r in enumerate(rules):
        markup.add(make_btn(f"🗑️ মুছুন: {format_pause_rule(r, idx+1)}", "red", callback_data=f"delpause_{idx}"))
    if rules:
        markup.add(make_btn("🗑️ সব বিরতি নিয়ম মুছে ফেলুন", "red", callback_data="clear_all_pauses"))
    markup.add(
        make_btn("✏️ এই পেজের স্ট্যাটাস ডিজাইন ও ইমোজি এডিট", "green", callback_data="edit_ui_pause_menu"),
        make_btn("🔙 মূল ড্যাশবোর্ডে ফিরে যান", "none", callback_data="main_menu")
    )
    replacements = {
        "{pause_now}": pause_now,
        "{pause_cnt}": len(rules),
        "{pause_list}": p_list
    }
    send_ui_page(chat_id, msg_id, "pause_menu", replacements, markup)

def show_ui_editor_hub(chat_id, msg_id=None):
    markup = InlineKeyboardMarkup(row_width=1)
    pages = [
        ("main_menu", "👑 মূল ড্যাশবোর্ড স্ট্যাটাস এডিট"),
        ("reaction_hub", "👍 রিঅ্যাকশন হাব স্ট্যাটাস এডিট"),
        ("default_btns", "⚙️ ডিফল্ট বাটন পেজ স্ট্যাটাস এডিট"),
        ("list_schedule", "📋 শিডিউল তালিকা হেডার এডিট"),
        ("recycle_bin", "🗑️ ডিলিট ফাইল হেডার এডিট"),
        ("pause_menu", "⏸️ শিডিউল বিরতি পেজ স্ট্যাটাস এডিট"),
        ("save_welcome", "✅ মেসেজ সেভ কনফার্মেশন টেক্সট এডিট")
    ]
    for k, lbl in pages:
        has_custom = "✨ কাস্টম" if k in data.get("ui_templates", {}) else "ডিফল্ট"
        markup.add(make_btn(f"{lbl} ({has_custom})", "blue", callback_data=f"edit_ui_{k}"))
    markup.add(
        make_btn("♻️ সব পেজের ডিজাইন ডিফল্টে রিসেট করুন", "red", callback_data="reset_all_ui"),
        make_btn("🔙 মূল ড্যাশবোর্ডে ফিরে যান", "green", callback_data="main_menu")
    )
    text = (
        "✏️ **প্রিমিয়াম স্ট্যাটাস ও লাইভ ইমোজি এডিটর প্যানেল**\n"
        "━━━━━━━━━━━━━━━━━━━━━━\n"
        "আপনি কোন পেজের উপরের স্ট্যাটাস মেসেজ, সিরিয়াল বা ইমোজি এডিট করতে চান তা সিলেক্ট করুন:\n\n"
        "💡 *নোট: আপনি আপনার টেলিগ্রাম প্রিমিয়াম অ্যাকাউন্ট থেকে যেকোনো কাস্টম লাইভ ইমোজি, বোল্ড ফন্ট এবং নিজের পছন্দমতো লাইন সাজাতে পারবেন। কোনো স্ট্যাটাস কোড মুছে গেলে বট অটোমেটিক তা ঠিক করে বসিয়ে দেবে!*"
    )
    smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)

def start_ui_template_edit(chat_id, page_key):
    cur_custom = data.get("ui_templates", {}).get(page_key)
    cur_text = cur_custom["text"] if (cur_custom and cur_custom.get("text")) else DEFAULT_TEMPLATES.get(page_key, {}).get("text", "")
    cur_ents = parse_entities(cur_custom.get("entities")) if cur_custom else None
    req_list = DEFAULT_TEMPLATES.get(page_key, {}).get("required", [])
    req_str = ", ".join(f"`{k}`" for k in req_list) if req_list else "কোনো বাধ্যতামূলক ট্যাগ নেই"
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        make_btn("♻️ এই পেজটি ডিফল্ট ডিজাইনে ফিরিয়ে নিন", "red", callback_data=f"reset_ui_{page_key}"),
        make_btn("🔙 বাতিল করে ফিরে যান", "blue", callback_data="ui_editor_menu")
    )
    guide = (
        f"✏️ **স্ট্যাটাস ডিজাইন ও প্রিমিয়াম ইমোজি এডিটর**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"নিচের মেসেজটিতে এই পেজের বর্তমান স্ট্যাটাস ফরম্যাটটি পাঠানো হলো।\n\n"
        f"📌 **যেভাবে এডিট করবেন:**\n"
        f"১. নিচের মেসেজটি কপি করুন।\n"
        f"২. আপনার ইচ্ছেমতো লাইনের সিরিয়াল উপরে বা নিচে সাজান, ফন্ট মোটা (Bold) করুন এবং আপনার **প্রিমিয়াম কাস্টম লাইভ ইমোজি** যুক্ত করুন।\n"
        f"৩. ডাইনামিক স্ট্যাটাস ট্যাগগুলো ({req_str}) যেভাবে আছে সেভাবেই রাখুন (ভুলবশত মুছে গেলেও চিন্তা নেই, বট অটোমেটিক ঠিক করে বসিয়ে দেবে!)।\n\n"
        f"👇 **নিচের টেক্সটটি কপি করে এডিট করে এখনই চ্যাটে পাঠিয়ে দিন:**"
    )
    bot.send_message(chat_id, guide, reply_markup=markup, parse_mode="Markdown")
    msg = bot.send_message(chat_id, cur_text, entities=cur_ents)
    bot.register_next_step_handler(msg, lambda m: process_ui_template_save(m, page_key))

def process_ui_template_save(message, page_key):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    if not message.text:
        msg = bot.send_message(chat_id, "❌ দয়া করে টেক্সট মেসেজ আকারে স্ট্যাটাসটি পাঠান:")
        bot.register_next_step_handler(msg, lambda m: process_ui_template_save(m, page_key))
        return
    new_text = message.text
    raw_ents = message.json.get("entities")
    req_list = DEFAULT_TEMPLATES.get(page_key, {}).get("required", [])
    missing = [k for k in req_list if k not in new_text]
    warn_msg = ""
    if missing:
        auto_lines = "\n" + "\n".join(missing)
        new_text = new_text + auto_lines
        warn_msg = (
            f"\n⚠️ **অটো-প্রোটেকশন নোটিশ:** আপনি এডিট করার সময় `{', '.join(missing)}` স্ট্যাটাসটি মুছে ফেলেছিলেন, "
            f"যার কারণে স্ট্যাটাস দেখাতে অসুবিধা হতো। বট অটোমেটিক্যালি সেটি নিচে ঠিক করে বসিয়ে দিয়েছে!\n"
        )
    data.setdefault("ui_templates", {})[page_key] = {
        "text": new_text,
        "entities": raw_ents
    }
    save_data(data)
    bot.send_message(
        chat_id,
        f"✅ **স্ট্যাটাস ডিজাইন ও প্রিমিয়াম ইমোজি সফলভাবে আপডেট হয়েছে!**{warn_msg}",
        parse_mode="Markdown"
    )
    if page_key == "main_menu":
        show_main_menu(chat_id)
    elif page_key == "reaction_hub":
        show_reaction_hub(chat_id)
    elif page_key == "default_btns":
        show_default_buttons_menu(chat_id)
    elif page_key == "list_schedule":
        list_schedules_as_buttons(chat_id, "schedules")
    elif page_key == "recycle_bin":
        list_schedules_as_buttons(chat_id, "recycle_bin")
    elif page_key == "pause_menu":
        show_pause_menu(chat_id)
    else:
        show_ui_editor_hub(chat_id)

def show_reaction_hub(chat_id, msg_id=None):
    schedules = data.get("schedules", {})
    rx_on_count = sum(1 for s in schedules.values() if s.get("reactions", True))
    def_on_count = sum(1 for s in schedules.values() if s.get("use_default_btns", False))
    global_rx = "চালু 🟢" if data.get("default_reactions_on", True) else "বন্ধ 🔴"
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        make_btn(f"👍 নতুন পোস্টে অটো রিঅ্যাকশন: {global_rx} (টগল করুন)", "green", callback_data="hub_toggle_global_rx"),
        make_btn("✅ সব অ্যাক্টিভ শিডিউলে রিঅ্যাকশন বাটন চালু করুন", "blue", callback_data="hub_all_rx_on"),
        make_btn("🔕 সব অ্যাক্টিভ শিডিউলে রিঅ্যাকশন বাটন বন্ধ করুন", "red", callback_data="hub_all_rx_off"),
        make_btn("🔗 সব অ্যাক্টিভ শিডিউলে ডিফল্ট বাটন যুক্ত করুন", "green", callback_data="hub_all_def_on"),
        make_btn("⚙️ ডিফল্ট বাটন সেটআপে যান", "blue", callback_data="default_btns"),
        make_btn("✏️ এই পেজের স্ট্যাটাস ডিজাইন ও ইমোজি এডিট", "green", callback_data="edit_ui_reaction_hub"),
        make_btn("🔙 মূল ড্যাশবোর্ডে ফিরে যান", "none", callback_data="main_menu")
    )
    replacements = {
        "{global_rx}": global_rx,
        "{rx_on_count}": rx_on_count,
        "{def_on_count}": def_on_count,
        "{total_sched}": len(schedules)
    }
    send_ui_page(chat_id, msg_id, "reaction_hub", replacements, markup)

def show_quick_post_menu(chat_id, msg_id=None):
    schedules = data.get("schedules", {})
    markup = InlineKeyboardMarkup(row_width=1)
    if not schedules:
        markup.add(make_btn("🔙 মূল ড্যাশবোর্ড", "blue", callback_data="main_menu"))
        smart_send_or_edit(chat_id, msg_id, "🚀 **ইনস্ট্যান্ট টেস্ট পোস্ট সেক্টর:**\n\nবর্তমানে কোনো অ্যাক্টিভ শিডিউল নেই।", reply_markup=markup)
        return
    for sid, s in schedules.items():
        b_cnt = len(s.get("buttons", [])) + (len(data.get("default_buttons", [])) if s.get("use_default_btns") else 0)
        rx_lbl = "👍সহ" if s.get("reactions", True) else "👍ছাড়া"
        markup.add(make_btn(f"🚀 এখনই পোস্ট করুন: {s['channel']} ({b_cnt}টি বাটন | {rx_lbl})", "green", callback_data=f"testpost_{sid}"))
    markup.add(make_btn("🔙 মূল ড্যাশবোর্ড", "blue", callback_data="main_menu"))
    text = (
        f"🚀 **ইনস্ট্যান্ট টেস্ট ও কুইক পোস্ট সেক্টর**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"কোনো শিডিউলের বাটন, কালার এবং লাইভ ইমোজি চ্যানেলে কেমন দেখাচ্ছে তা এখনই পরীক্ষা করতে নিচের যেকোনো শিডিউলে ক্লিক করুন (সাথে সাথে চ্যানেলে পোস্ট হয়ে যাবে):"
    )
    smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)

def show_color_customizer(chat_id, msg_id):
    markup = InlineKeyboardMarkup(row_width=2)
    btn_list = [
        ("add_schedule", "➕ নতুন শিডিউল"),
        ("list_schedule", "📋 শিডিউল তালিকা"),
        ("default_btns", "⚙️ ডিফল্ট বাটন"),
        ("reaction_hub", "👍 রিঅ্যাকশন হাব"),
        ("quick_post", "🚀 টেস্ট পোস্ট"),
        ("recycle_bin", "🗑️ ডিলিট ফাইল"),
        ("pause_menu", "⏸️ শিডিউল বিরতি"),
        ("ui_editor", "✏️ স্ট্যাটাস এডিটর"),
        ("color_menu", "🎨 কালার সেটিং"),
        ("toggle_bot", "⚡ বট অন/অফ"),
        ("restart_bot", "🔄 বট রিস্টার্ট")
    ]
    for key, label in btn_list:
        markup.add(make_btn(label, get_c(key), callback_data=f"pickbtn_{key}"))
    markup.add(make_btn("🔙 মূল ড্যাশবোর্ড", "none", callback_data="main_menu"))
    smart_send_or_edit(chat_id, msg_id, "🎨 **অ্যাডমিন প্যানেলের কোন বাটনের কালার পরিবর্তন করতে চান সিলেক্ট করুন:**", reply_markup=markup)

def show_color_options(chat_id, msg_id, btn_key):
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        make_btn("লাল (Red)", "red", callback_data=f"setcol_red_{btn_key}"),
        make_btn("নীল (Blue)", "blue", callback_data=f"setcol_blue_{btn_key}"),
        make_btn("সবুজ (Green)", "green", callback_data=f"setcol_green_{btn_key}"),
        make_btn("✖️ ডিফল্ট কালার", "none", callback_data=f"setcol_none_{btn_key}")
    )
    markup.add(make_btn("🔙 ফিরে যান", "none", callback_data="color_menu"))
    smart_send_or_edit(chat_id, msg_id, "🎨 **আপনার পছন্দের ব্যাকগ্রাউন্ড কালার সিলেক্ট করুন:**", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: True)
def callback_query(call):
    chat_id = call.message.chat.id
    cid = call.id
    msg_id = call.message.message_id
    cdata = call.data
    if cdata.startswith("rx_"):
        parts = cdata.split("_")
        rx_type = parts[1]
        sid = parts[2] if len(parts) > 2 else "0"
        msg_key = f"{chat_id}_{msg_id}"
        uid = str(call.from_user.id)
        rx_store = data.setdefault("reactions_data", {}).setdefault(msg_key, {
            "counts": {"up": 0, "love": 0, "fire": 0, "wow": 0},
            "users": {}
        })
        prev = rx_store["users"].get(uid)
        if prev == rx_type:
            rx_store["counts"][rx_type] = max(0, rx_store["counts"].get(rx_type, 1) - 1)
            del rx_store["users"][uid]
            ack(cid, "আপনার রিঅ্যাকশন সরানো হয়েছে!")
        else:
            if prev and prev in rx_store["counts"]:
                rx_store["counts"][prev] = max(0, rx_store["counts"][prev] - 1)
            rx_store["counts"][rx_type] = rx_store["counts"].get(rx_type, 0) + 1
            rx_store["users"][uid] = rx_type
            ack(cid, "আপনার রিঅ্যাকশন যুক্ত হয়েছে! ❤️")
        save_data(data)
        sched = data.get("schedules", {}).get(sid) or data.get("recycle_bin", {}).get(sid)
        if sched:
            try:
                bot.edit_message_reply_markup(chat_id, msg_id, reply_markup=build_post_markup(sched, sid, rx_store["counts"]))
            except Exception:
                pass
        return
    if not is_auth(call.from_user.id):
        ack(cid, "🔒 আপনি অনুমোদিত নন!", show_alert=True)
        return
    if cdata == "main_menu":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        show_main_menu(chat_id, msg_id)
    elif cdata == "pause_menu":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        show_pause_menu(chat_id, msg_id)
    elif cdata == "ui_editor_menu":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        show_ui_editor_hub(chat_id, msg_id)
    elif cdata.startswith("edit_ui_"):
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        page_key = cdata.split("edit_ui_")[1]
        start_ui_template_edit(chat_id, page_key)
    elif cdata.startswith("reset_ui_"):
        page_key = cdata.split("reset_ui_")[1]
        data.get("ui_templates", {}).pop(page_key, None)
        save_data(data)
        ack(cid, "♻️ ডিফল্ট ডিজাইনে রিসেট করা হয়েছে!", show_alert=True)
        show_ui_editor_hub(chat_id, msg_id)
    elif cdata == "reset_all_ui":
        data["ui_templates"] = {}
        save_data(data)
        ack(cid, "♻️ সব পেজের ডিজাইন ডিফল্ট করা হয়েছে!", show_alert=True)
        show_ui_editor_hub(chat_id, msg_id)
    elif cdata == "add_schedule":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        msg = bot.send_message(chat_id, "১. চ্যানেলের ইউজারনেম বা আইডি দিন (যেমন: @battleclubofficial2):")
        bot.register_next_step_handler(msg, process_channel)
    elif cdata == "list_schedule":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        list_schedules_as_buttons(chat_id, "schedules", msg_id)
    elif cdata == "recycle_bin":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        list_schedules_as_buttons(chat_id, "recycle_bin", msg_id)
    elif cdata == "reaction_hub":
        ack(cid)
        show_reaction_hub(chat_id, msg_id)
    elif cdata == "hub_toggle_global_rx":
        data["default_reactions_on"] = not data.get("default_reactions_on", True)
        save_data(data)
        ack(cid, "✅ অটো রিঅ্যাকশন সেটিং আপডেট হয়েছে!")
        show_reaction_hub(chat_id, msg_id)
    elif cdata == "hub_all_rx_on":
        for s in data.get("schedules", {}).values():
            s["reactions"] = True
            s["forward_mode"] = False
        save_data(data)
        ack(cid, "✅ সব শিডিউলে রিঅ্যাকশন বাটন চালু করা হয়েছে!", show_alert=True)
        show_reaction_hub(chat_id, msg_id)
    elif cdata == "hub_all_rx_off":
        for s in data.get("schedules", {}).values():
            s["reactions"] = False
        save_data(data)
        ack(cid, "🔕 সব শিডিউলে রিঅ্যাকশন বাটন বন্ধ করা হয়েছে!", show_alert=True)
        show_reaction_hub(chat_id, msg_id)
    elif cdata == "hub_all_def_on":
        for s in data.get("schedules", {}).values():
            s["use_default_btns"] = True
            s["forward_mode"] = False
        save_data(data)
        ack(cid, "🔗 সব শিডিউলে ডিফল্ট বাটন চালু করা হয়েছে!", show_alert=True)
        show_reaction_hub(chat_id, msg_id)
    elif cdata == "quick_post":
        ack(cid)
        show_quick_post_menu(chat_id, msg_id)
    elif cdata.startswith("testpost_"):
        sid = cdata.split("_")[1]
        s = data.get("schedules", {}).get(sid) or data.get("recycle_bin", {}).get(sid)
        if s:
            ack(cid, "🚀 চ্যানেলে পোস্ট পাঠানো হচ্ছে...")
            ok = send_scheduled_post(s, sid)
            if ok:
                bot.send_message(chat_id, f"✅ **{s['channel']}** চ্যানেলে বাটন ও রিঅ্যাকশনসহ টেস্ট পোস্ট সফলভাবে পাঠানো হয়েছে!", parse_mode="Markdown")
            else:
                bot.send_message(chat_id, f"❌ **{s['channel']}** চ্যানেলে পোস্ট পাঠানো যায়নি! বট চ্যানেলের অ্যাডমিন আছে কিনা চেক করুন।", parse_mode="Markdown")
        else:
            ack(cid, "শিডিউল পাওয়া যায়নি!")
    elif cdata == "toggle_bot":
        data["is_running"] = not data.get("is_running", True)
        save_data(data)
        ack(cid, "⚡ বটের স্ট্যাটাস পরিবর্তন করা হয়েছে!")
        show_main_menu(chat_id, msg_id)
    elif cdata == "restart_bot":
        ack(cid, "🔄 বট রিস্টার্ট করা হচ্ছে...")
        bot.send_message(chat_id, "🔄 বট রিস্টার্ট হচ্ছে। মুহূর্তের মধ্যে চালু হবে।")
        python = sys.executable
        os.execl(python, python, *sys.argv)
    elif cdata == "color_menu":
        ack(cid)
        show_color_customizer(chat_id, msg_id)
    elif cdata.startswith("pickbtn_"):
        ack(cid)
        btn_key = cdata.split("pickbtn_")[1]
        show_color_options(chat_id, msg_id, btn_key)
    elif cdata.startswith("setcol_"):
        parts = cdata.split("_")
        col = parts[1]
        btn_key = "_".join(parts[2:])
        data.setdefault("btn_colors", {})[btn_key] = col
        save_data(data)
        ack(cid, "✅ বাটনের ব্যাকগ্রাউন্ড কালার আপডেট হয়েছে!")
        show_color_customizer(chat_id, msg_id)
    else:
        handle_more_callbacks(call, chat_id, cid, msg_id, cdata)

def ask_pause_weekday(chat_id, msg_id=None):
    markup = InlineKeyboardMarkup(row_width=2)
    for wk_num, wk_lbl in WEEKDAYS_BN:
        markup.add(make_btn(f"🗓️ {wk_lbl}", "blue", callback_data=f"pause_wk_{wk_num}"))
    markup.add(make_btn("🔙 ফিরে যান", "red", callback_data="pause_menu"))
    text = (
        "🗓️ **সাপ্তাহিক শিডিউল বিরতি সেটআপ:**\n"
        "━━━━━━━━━━━━━━━━━━━━━━\n"
        "সপ্তাহের কোন দিন আপনি শিডিউল পোস্টিং বন্ধ রাখতে চান সিলেক্ট করুন:"
    )
    smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)

def ask_pause_date_year(chat_id, msg_id=None):
    now_bd = datetime.now(BD_TZ)
    cur_yr = now_bd.year
    markup = InlineKeyboardMarkup(row_width=2)
    markup.row(make_btn(f"⚡ আজকের তারিখ ({now_bd.strftime('%d %b %Y')})", "green", callback_data="pause_dt_today"))
    markup.row(
        make_btn(f"📅 {cur_yr} সাল", "blue", callback_data=f"pause_yr_{cur_yr}"),
        make_btn(f"📅 {cur_yr + 1} সাল", "blue", callback_data=f"pause_yr_{cur_yr + 1}")
    )
    markup.row(make_btn("🔙 ফিরে যান", "red", callback_data="pause_menu"))
    text = (
        "📅 **নির্দিষ্ট তারিখে শিডিউল বিরতি সেটআপ:**\n"
        "━━━━━━━━━━━━━━━━━━━━━━\n"
        "কোন তারিখে শিডিউল বন্ধ রাখতে চান? নিচের বাটন থেকে সাল সিলেক্ট করুন অথবা সরাসরি চ্যাটে তারিখ লিখুন (যেমন: `2026-10-30` বা `October 30`):"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, process_pause_date_text)

def process_pause_date_text(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    raw = norm_digits(message.text or "")
    now_bd = datetime.now(BD_TZ)
    m_iso = re.match(r"^(\d{4})[-/.](\d{1,2})[-/.](\d{1,2})$", raw)
    if m_iso:
        yr, mo, dy = int(m_iso.group(1)), int(m_iso.group(2)), int(m_iso.group(3))
        if 1 <= mo <= 12 and 1 <= dy <= calendar.monthrange(yr, mo)[1]:
            temp_schedule.setdefault(chat_id, {})["temp_pause"] = {"type": "date", "date": f"{yr:04d}-{mo:02d}-{dy:02d}"}
            ask_pause_start_time(chat_id)
            return
    m_named = re.match(r"^([A-Za-z\u0980-\u09FF]+)[\s.,/-]+(\d{1,2})(?:[\s.,/-]+(\d{4}))?$", raw)
    if m_named:
        mo = parse_month_value(m_named.group(1))
        dy = int(m_named.group(2))
        yr = int(m_named.group(3)) if m_named.group(3) else now_bd.year
        if mo and 1 <= dy <= calendar.monthrange(yr, mo)[1]:
            temp_schedule.setdefault(chat_id, {})["temp_pause"] = {"type": "date", "date": f"{yr:04d}-{mo:02d}-{dy:02d}"}
            ask_pause_start_time(chat_id)
            return
    msg = bot.send_message(chat_id, "❌ সঠিক তারিখ লিখুন (যেমন: `2026-10-30` বা `October 30`) অথবা উপরের বাটন চাপুন:", parse_mode="Markdown")
    bot.register_next_step_handler(msg, process_pause_date_text)

def ask_pause_date_month(chat_id, yr, msg_id=None):
    markup = InlineKeyboardMarkup(row_width=2)
    for i in range(0, len(MONTH_LIST), 2):
        m1_num, m1_lbl = MONTH_LIST[i]
        m2_num, m2_lbl = MONTH_LIST[i + 1]
        markup.row(
            make_btn(f"🗓️ {m1_lbl}", "blue", callback_data=f"pause_mo_{yr}_{m1_num}"),
            make_btn(f"🗓️ {m2_lbl}", "blue", callback_data=f"pause_mo_{yr}_{m2_num}")
        )
    markup.add(make_btn("🔙 ফিরে যান", "red", callback_data="add_pause_date"))
    smart_send_or_edit(chat_id, msg_id, f"✅ সাল: **{yr}**\nএখন কোন মাসে বন্ধ রাখতে চান সিলেক্ট করুন:", reply_markup=markup)

def ask_pause_date_day(chat_id, yr, mo, msg_id=None):
    max_days = calendar.monthrange(yr, mo)[1]
    m_lbl = dict(MONTH_LIST).get(mo, str(mo))
    markup = InlineKeyboardMarkup(row_width=5)
    row_btns = []
    for d in range(1, max_days + 1):
        row_btns.append(make_btn(str(d), "green", callback_data=f"pause_dy_{yr}_{mo}_{d}"))
        if len(row_btns) == 5:
            markup.row(*row_btns)
            row_btns = []
    if row_btns:
        markup.row(*row_btns)
    markup.add(make_btn("🔙 ফিরে যান", "red", callback_data=f"pause_yr_{yr}"))
    smart_send_or_edit(chat_id, msg_id, f"✅ **{m_lbl}, {yr}**\nএখন কয় তারিখে বন্ধ থাকবে সিলেক্ট করুন:", reply_markup=markup)

def ask_pause_start_time(chat_id, msg_id=None):
    p_obj = temp_schedule.get(chat_id, {}).get("temp_pause", {})
    if p_obj.get("type") == "weekly":
        target_lbl = f"প্রতি {WEEKDAY_MAP.get(p_obj.get('weekday', 4))}"
    else:
        target_lbl = f"তারিখ {format_display_date(p_obj.get('date', ''))}"
    markup = InlineKeyboardMarkup(row_width=2)
    markup.row(make_btn("🛑 সারাদিন (২৪ ঘণ্টা) বন্ধ রাখুন", "red", callback_data="pause_allday"))
    markup.row(
        make_btn("🕛 12:00 PM (দুপুর ১২টা)", "blue", callback_data="pause_qst_12:00"),
        make_btn("🕧 12:30 PM", "blue", callback_data="pause_qst_12:30")
    )
    markup.row(
        make_btn("🕐 01:00 PM", "blue", callback_data="pause_qst_13:00"),
        make_btn("🕕 06:00 PM", "blue", callback_data="pause_qst_18:00")
    )
    markup.row(make_btn("🔙 বাতিল করুন", "none", callback_data="pause_menu"))
    text = (
        f"⏸️ নির্ধারিত দিন: **{target_lbl}**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"১️⃣ **কয়টা থেকে শিডিউল পোস্টিং বন্ধ রাখা শুরু হবে (Start Time)?**\n\n"
        f"• চ্যাটে যেকোনো শুরুর সময় লিখে পাঠান (যেমন: `12:00 PM`, `12:30 AM` বা `14:00`)\n"
        f"• অথবা নিচের কুইক বাটন থেকে সিলেক্ট করুন:"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, process_pause_start_input)

def process_pause_start_input(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    st = parse_user_time(message.text)
    if not st:
        msg = bot.send_message(chat_id, "❌ সময়ের ফরম্যাট সঠিক নয়! (যেমন: `12:00 PM` বা `14:00`) আবার লিখুন:", parse_mode="Markdown")
        bot.register_next_step_handler(msg, process_pause_start_input)
        return
    temp_schedule.setdefault(chat_id, {}).setdefault("temp_pause", {})["start_time"] = st
    ask_pause_end_time(chat_id)

def ask_pause_end_time(chat_id, msg_id=None):
    p_obj = temp_schedule.get(chat_id, {}).get("temp_pause", {})
    st_disp = format_display_time(p_obj.get("start_time", "12:00"))
    markup = InlineKeyboardMarkup(row_width=2)
    markup.row(
        make_btn("🕝 02:30 PM (দুপুর ২:৩০)", "green", callback_data="pause_qet_14:30"),
        make_btn("🕑 02:00 PM (দুপুর ২:০০)", "green", callback_data="pause_qet_14:00")
    )
    markup.row(
        make_btn("🕒 03:00 PM", "blue", callback_data="pause_qet_15:00"),
        make_btn("🌙 11:59 PM (রাত পর্যন্ত)", "blue", callback_data="pause_qet_23:59")
    )
    markup.row(make_btn("🔙 বাতিল করুন", "none", callback_data="pause_menu"))
    text = (
        f"✅ বন্ধ শুরু হওয়ার সময়: **{st_disp}**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"২️⃣ **কয়টা পর্যন্ত শিডিউল পোস্টিং বন্ধ থাকবে (End Time)?**\n\n"
        f"• চ্যাটে শেষ হওয়ার সময় লিখে পাঠান (যেমন: `02:30 PM` বা `14:30`)\n"
        f"• অথবা নিচের বাটন থেকে সিলেক্ট করুন:"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, process_pause_end_input)

def process_pause_end_input(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    et = parse_user_time(message.text)
    if not et:
        msg = bot.send_message(chat_id, "❌ সময়ের ফরম্যাট সঠিক নয়! (যেমন: `02:30 PM` বা `14:30`) আবার লিখুন:", parse_mode="Markdown")
        bot.register_next_step_handler(msg, process_pause_end_input)
        return
    finalize_pause_rule(chat_id, et)

def finalize_pause_rule(chat_id, end_time, msg_id=None):
    p_obj = temp_schedule.get(chat_id, {}).pop("temp_pause", None)
    if not p_obj:
        show_pause_menu(chat_id, msg_id)
        return
    p_obj["end_time"] = end_time
    p_obj["enabled"] = True
    data.setdefault("pause_rules", []).append(p_obj)
    save_data(data)
    bot.send_message(chat_id, f"✅ **নতুন শিডিউল বিরতি সফলভাবে যুক্ত হয়েছে!**\n{format_pause_rule(p_obj)}", parse_mode="Markdown")
    show_pause_menu(chat_id, msg_id)

def handle_more_callbacks(call, chat_id, cid, msg_id, cdata):
    if cdata == "add_pause_weekly":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        ask_pause_weekday(chat_id, msg_id)
    elif cdata.startswith("pause_wk_"):
        wk = int(cdata.split("_")[2])
        bot.clear_step_handler_by_chat_id(chat_id)
        temp_schedule.setdefault(chat_id, {})["temp_pause"] = {"type": "weekly", "weekday": wk}
        ack(cid, f"🗓️ {WEEKDAY_MAP.get(wk)} সিলেক্ট হয়েছে!")
        ask_pause_start_time(chat_id, msg_id)
    elif cdata == "add_pause_date":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        ask_pause_date_year(chat_id, msg_id)
    elif cdata == "pause_dt_today":
        bot.clear_step_handler_by_chat_id(chat_id)
        today_s = datetime.now(BD_TZ).strftime("%Y-%m-%d")
        temp_schedule.setdefault(chat_id, {})["temp_pause"] = {"type": "date", "date": today_s}
        ack(cid, f"📅 আজকের তারিখ ({today_s}) সিলেক্ট হয়েছে!")
        ask_pause_start_time(chat_id, msg_id)
    elif cdata.startswith("pause_yr_"):
        yr = int(cdata.split("_")[2])
        bot.clear_step_handler_by_chat_id(chat_id)
        ack(cid, f"📅 সাল {yr} সিলেক্ট হয়েছে!")
        ask_pause_date_month(chat_id, yr, msg_id)
    elif cdata.startswith("pause_mo_"):
        _, _, yr, mo = cdata.split("_")
        bot.clear_step_handler_by_chat_id(chat_id)
        ack(cid)
        ask_pause_date_day(chat_id, int(yr), int(mo), msg_id)
    elif cdata.startswith("pause_dy_"):
        _, _, yr, mo, dy = cdata.split("_")
        bot.clear_step_handler_by_chat_id(chat_id)
        d_str = f"{int(yr):04d}-{int(mo):02d}-{int(dy):02d}"
        temp_schedule.setdefault(chat_id, {})["temp_pause"] = {"type": "date", "date": d_str}
        ack(cid, f"📅 তারিখ {d_str} সিলেক্ট হয়েছে!")
        ask_pause_start_time(chat_id, msg_id)
    elif cdata == "pause_allday":
        bot.clear_step_handler_by_chat_id(chat_id)
        temp_schedule.setdefault(chat_id, {}).setdefault("temp_pause", {})["start_time"] = "00:00"
        ack(cid, "🛑 সারাদিন (২৪ ঘণ্টা) বিরতি সেট হয়েছে!")
        finalize_pause_rule(chat_id, "23:59", msg_id)
    elif cdata.startswith("pause_qst_"):
        st = cdata.split("pause_qst_")[1]
        bot.clear_step_handler_by_chat_id(chat_id)
        temp_schedule.setdefault(chat_id, {}).setdefault("temp_pause", {})["start_time"] = st
        ack(cid, f"🕒 শুরুর সময় {st} সেট হয়েছে!")
        ask_pause_end_time(chat_id, msg_id)
    elif cdata.startswith("pause_qet_"):
        et = cdata.split("pause_qet_")[1]
        bot.clear_step_handler_by_chat_id(chat_id)
        ack(cid, f"🕒 শেষ সময় {et} সেট হয়েছে!")
        finalize_pause_rule(chat_id, et, msg_id)
    elif cdata.startswith("delpause_"):
        idx = int(cdata.split("_")[1])
        rules = data.get("pause_rules", [])
        if 0 <= idx < len(rules):
            rules.pop(idx)
            save_data(data)
            ack(cid, "🗑️ বিরতি নিয়মটি মুছে ফেলা হয়েছে!")
        show_pause_menu(chat_id, msg_id)
    elif cdata == "clear_all_pauses":
        data["pause_rules"] = []
        save_data(data)
        ack(cid, "🗑️ সব বিরতি নিয়ম মুছে ফেলা হয়েছে!", show_alert=True)
        show_pause_menu(chat_id, msg_id)
    elif cdata == "s3_add_btn":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        msg = bot.send_message(chat_id, "➕ **নতুন বাটনের নাম লিখুন** (যেমন: `📥 Download App` বা `🎮 Join Match`):", parse_mode="Markdown")
        bot.register_next_step_handler(msg, process_button_name)
    elif cdata.startswith("s3col_"):
        col = cdata.split("_")[1]
        bot.clear_step_handler_by_chat_id(chat_id)
        if chat_id in temp_schedule and "pending_btn" in temp_schedule[chat_id]:
            b_obj = temp_schedule[chat_id].pop("pending_btn")
            b_obj["color"] = col
            temp_schedule[chat_id].setdefault("buttons", []).append(b_obj)
            ack(cid, f"✅ '{b_obj['text']}' বাটন যুক্ত হয়েছে!")
            show_step3_button_menu(chat_id, msg_id)
        else:
            ack(cid)
    elif cdata == "s3_toggle_rx":
        if chat_id in temp_schedule:
            temp_schedule[chat_id]["reactions"] = not temp_schedule[chat_id].get("reactions", True)
            ack(cid, "👍 রিঅ্যাকশন বাটন স্ট্যাটাস পরিবর্তন হয়েছে!")
            show_step3_button_menu(chat_id, msg_id)
        else:
            ack(cid)
    elif cdata == "s3_toggle_def":
        if chat_id in temp_schedule:
            temp_schedule[chat_id]["use_default_btns"] = not temp_schedule[chat_id].get("use_default_btns", False)
            ack(cid, "⚙️ ডিফল্ট বাটন স্ট্যাটাস পরিবর্তন হয়েছে!")
            show_step3_button_menu(chat_id, msg_id)
        else:
            ack(cid)
    elif cdata == "s3_clear_btns":
        if chat_id in temp_schedule:
            temp_schedule[chat_id]["buttons"] = []
            ack(cid, "🗑️ বাটনগুলো মুছে ফেলা হয়েছে!")
            show_step3_button_menu(chat_id, msg_id)
        else:
            ack(cid)
    elif cdata == "btn_done_next":
        ack(cid, "সময় নির্ধারণ ধাপে যাওয়া হচ্ছে...")
        bot.clear_step_handler_by_chat_id(chat_id)
        ask_time_method(chat_id, msg_id)
    elif cdata == "btn_skip":
        ack(cid, "স্পেসিফিক বাটন স্কিপ করা হয়েছে")
        bot.clear_step_handler_by_chat_id(chat_id)
        if chat_id in temp_schedule:
            temp_schedule[chat_id]["buttons"] = []
            temp_schedule[chat_id]["use_default_btns"] = False
        ask_time_method(chat_id, msg_id)
    elif cdata == "btn_only_default":
        ack(cid, "ডিফল্ট বাটন সিলেক্ট করা হয়েছে")
        bot.clear_step_handler_by_chat_id(chat_id)
        if chat_id in temp_schedule:
            temp_schedule[chat_id]["buttons"] = []
            temp_schedule[chat_id]["use_default_btns"] = True
        ask_time_method(chat_id, msg_id)
    elif cdata == "time_mode_daily":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        msg = smart_send_or_edit(chat_id, msg_id, "⏰ **প্রতিদিন কোন সময়ে পোস্ট হবে?**\n(যেমন: `12:00 PM` বা `02:30 PM` অথবা `14:30` লিখুন):")
        bot.register_next_step_handler(msg, lambda m: process_specific_time(m, "daily", "new"))
    elif cdata == "time_mode_once":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        ask_once_year(chat_id, "new", msg_id)
    elif cdata.startswith("once_today_"):
        ctx = cdata.split("once_today_")[1]
        bot.clear_step_handler_by_chat_id(chat_id)
        now_bd = datetime.now(BD_TZ)
        target_dict = get_interval_target_dict(chat_id, ctx)
        if target_dict is not None:
            target_dict["temp_once_year"] = now_bd.year
            target_dict["temp_once_month"] = now_bd.month
            target_dict["temp_once_day"] = now_bd.day
            target_dict["once_date"] = now_bd.strftime("%Y-%m-%d")
            ack(cid, f"📅 আজকের তারিখ ({target_dict['once_date']}) সিলেক্ট হয়েছে!")
            ask_once_time(chat_id, ctx, msg_id)
        else:
            ack(cid)
    elif cdata.startswith("once_yr_"):
        parts = cdata.split("_")
        yr = int(parts[2])
        ctx = "_".join(parts[3:])
        bot.clear_step_handler_by_chat_id(chat_id)
        target_dict = get_interval_target_dict(chat_id, ctx)
        if target_dict is not None:
            target_dict["temp_once_year"] = yr
            ack(cid, f"📅 সাল {yr} সিলেক্ট হয়েছে!")
            ask_once_month(chat_id, ctx, msg_id)
        else:
            ack(cid)
    elif cdata.startswith("once_mo_"):
        parts = cdata.split("_")
        mo = int(parts[2])
        ctx = "_".join(parts[3:])
        bot.clear_step_handler_by_chat_id(chat_id)
        target_dict = get_interval_target_dict(chat_id, ctx)
        if target_dict is not None:
            target_dict["temp_once_month"] = mo
            m_lbl = dict(MONTH_LIST).get(mo, str(mo))
            ack(cid, f"🗓️ মাস {m_lbl} সিলেক্ট হয়েছে!")
            ask_once_day(chat_id, ctx, msg_id)
        else:
            ack(cid)
    elif cdata.startswith("once_day_"):
        parts = cdata.split("_")
        dy = int(parts[2])
        ctx = "_".join(parts[3:])
        bot.clear_step_handler_by_chat_id(chat_id)
        target_dict = get_interval_target_dict(chat_id, ctx)
        if target_dict is not None:
            yr = target_dict.get("temp_once_year", datetime.now(BD_TZ).year)
            mo = target_dict.get("temp_once_month", datetime.now(BD_TZ).month)
            target_dict["temp_once_day"] = dy
            target_dict["once_date"] = f"{yr:04d}-{mo:02d}-{dy:02d}"
            ack(cid, f"📆 তারিখ {target_dict['once_date']} সিলেক্ট হয়েছে!")
            ask_once_time(chat_id, ctx, msg_id)
        else:
            ack(cid)
    elif cdata.startswith("once_qtime_"):
        parts = cdata.split("_")
        hh_mm = parts[2]
        ctx = "_".join(parts[3:])
        bot.clear_step_handler_by_chat_id(chat_id)
        ack(cid, f"🕒 সময় {format_display_time(hh_mm)} সিলেক্ট হয়েছে!")
        finish_once_schedule(chat_id, hh_mm, ctx, msg_id)
    elif cdata == "time_mode_interval":
        ack(cid)
        bot.clear_step_handler_by_chat_id(chat_id)
        ask_interval_minutes(chat_id, "new", msg_id)
    else:
        handle_extra_callbacks(call, chat_id, cid, msg_id, cdata)

def handle_extra_callbacks(call, chat_id, cid, msg_id, cdata):
    if cdata.startswith("qint_"):
        parts = cdata.split("_")
        mins = int(parts[1])
        ctx = parts[2] if len(parts) > 2 else "new"
        bot.clear_step_handler_by_chat_id(chat_id)
        ack(cid, f"প্রতি {mins} মিনিট পর পর সিলেক্ট হয়েছে!")
        set_interval_minutes_and_ask_start(chat_id, mins, ctx, msg_id)
    elif cdata.startswith("intstart_now_"):
        ctx = cdata.split("intstart_now_")[1]
        bot.clear_step_handler_by_chat_id(chat_id)
        ack(cid, "⚡ এখন থেকেই শুরু সেট হয়েছে!")
        target_dict = get_interval_target_dict(chat_id, ctx)
        if target_dict is not None:
            target_dict["start_time"] = None
            target_dict["end_time"] = None
            ask_interval_duration_mode(chat_id, ctx, msg_id)
    elif cdata.startswith("intend_"):
        parts = cdata.split("_")
        etype = parts[1]
        ctx = parts[2] if len(parts) > 2 else "new"
        bot.clear_step_handler_by_chat_id(chat_id)
        target_dict = get_interval_target_dict(chat_id, ctx)
        if target_dict is not None:
            if etype == "2359":
                target_dict["end_time"] = "23:59"
                ack(cid, "রাত ১১:৫৯ পর্যন্ত সেট হয়েছে!")
            else:
                target_dict["end_time"] = None
                ack(cid, "কোনো শেষ সময় ছাড়া সেট হয়েছে!")
            ask_interval_duration_mode(chat_id, ctx, msg_id)
    elif cdata.startswith("intdur_"):
        parts = cdata.split("_")
        rep_mode = parts[1]
        ctx = parts[2] if len(parts) > 2 else "new"
        bot.clear_step_handler_by_chat_id(chat_id)
        target_dict = get_interval_target_dict(chat_id, ctx)
        if target_dict is not None:
            target_dict["mode"] = "interval"
            target_dict["interval_repeat"] = rep_mode
            target_dict["has_posted_once"] = False
            target_dict["active_date"] = datetime.now(BD_TZ).strftime("%Y-%m-%d")
            if target_dict.get("start_time"):
                target_dict["last_timestamp"] = 0
            else:
                target_dict["last_timestamp"] = int(time.time())
            lbl = "প্রতিদিন (আজীবন)" if rep_mode == "daily" else "শুধু ১ দিনের জন্য"
            ack(cid, f"✅ {lbl} সেট হয়েছে!")
            if ctx == "new":
                finalize_or_save_new_schedule(chat_id, msg_id)
            else:
                save_data(data)
                target_loc = "schedules" if ctx in data["schedules"] else "recycle_bin"
                show_schedule_detail(chat_id, ctx, target_loc, msg_id)
    elif cdata.startswith("addmorebtn_"):
        ack(cid)
        sid = cdata.split("_")[1]
        msg = bot.send_message(chat_id, "➕ নতুন বাটনের নাম কী হবে লিখুন:")
        bot.register_next_step_handler(msg, lambda m: process_more_btn_name(m, sid))
    elif cdata.startswith("edbtncol_"):
        _, col, sid = cdata.split("_")
        s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
        if s and chat_id in temp_schedule and "temp_more_btn" in temp_schedule[chat_id]:
            b_obj = temp_schedule[chat_id].pop("temp_more_btn")
            b_obj["color"] = col
            s.setdefault("buttons", []).append(b_obj)
            s["forward_mode"] = False
            save_data(data)
        ack(cid, "✅ নতুন বাটন কালারসহ যুক্ত হয়েছে!")
        show_post_extra_options(chat_id, sid, msg_id)
    elif cdata.startswith("toggledef_"):
        sid = cdata.split("_")[1]
        s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
        if s:
            s["use_default_btns"] = not s.get("use_default_btns", False)
            if s["use_default_btns"]:
                s["forward_mode"] = False
            save_data(data)
            ack(cid, "ডিফল্ট বাটন স্ট্যাটাস পরিবর্তন হয়েছে!")
            show_post_extra_options(chat_id, sid, msg_id)
    elif cdata.startswith("togglerx_"):
        sid = cdata.split("_")[1]
        s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
        if s:
            s["reactions"] = not s.get("reactions", True)
            if s["reactions"]:
                s["forward_mode"] = False
            save_data(data)
            ack(cid, "👍 রিঅ্যাকশন বাটন স্ট্যাটাস পরিবর্তন হয়েছে!")
            show_post_extra_options(chat_id, sid, msg_id)
    elif cdata.startswith("togglefwd_"):
        sid = cdata.split("_")[1]
        s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
        if s:
            has_btns = bool(s.get("buttons") or (s.get("use_default_btns") and data.get("default_buttons")) or s.get("reactions", True))
            if has_btns:
                s["forward_mode"] = False
                save_data(data)
                ack(cid, "ℹ️ বাটন ও রিঅ্যাকশনসহ পোস্ট করার জন্য স্মার্ট কপি মোড সক্রিয় রাখা হয়েছে!", show_alert=True)
            else:
                s["forward_mode"] = not s.get("forward_mode", False)
                save_data(data)
                ack(cid, "পোস্টিং মোড পরিবর্তন হয়েছে!")
            show_post_extra_options(chat_id, sid, msg_id)
    elif cdata.startswith("btn_del_"):
        sid = cdata.split("_")[2]
        s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
        if s:
            s["buttons"] = []
            save_data(data)
            ack(cid, "🗑️ স্পেসিফিক বাটনগুলো মুছে ফেলা হয়েছে!")
            show_post_extra_options(chat_id, sid, msg_id)
    elif cdata.startswith("view_"):
        ack(cid)
        sid = cdata.split("_")[1]
        show_schedule_detail(chat_id, sid, "schedules", msg_id)
    elif cdata.startswith("viewbin_"):
        ack(cid)
        sid = cdata.split("_")[1]
        show_schedule_detail(chat_id, sid, "recycle_bin", msg_id)
    elif cdata.startswith("del_"):
        sid = cdata.split("_")[1]
        if sid in data["schedules"]:
            data["recycle_bin"][sid] = data["schedules"].pop(sid)
            save_data(data)
            ack(cid, "🗑️ শিডিউলটি ডিলিট ফাইলে চলে গেছে!")
            list_schedules_as_buttons(chat_id, "schedules", msg_id)
        else:
            ack(cid, "শিডিউল পাওয়া যায়নি!")
    elif cdata.startswith("restore_"):
        sid = cdata.split("_")[1]
        if sid in data["recycle_bin"]:
            restored = data["recycle_bin"].pop(sid)
            restored["last_posted"] = ""
            restored["has_posted_once"] = False
            restored["active_date"] = datetime.now(BD_TZ).strftime("%Y-%m-%d")
            restored["last_timestamp"] = 0 if restored.get("start_time") else int(time.time())
            restored["forward_mode"] = False
            data["schedules"][sid] = restored
            save_data(data)
            ack(cid, "♻️ শিডিউল সফলভাবে রিস্টোর করা হয়েছে!")
            show_schedule_detail(chat_id, sid, "schedules", msg_id)
    elif cdata.startswith("permdel_"):
        sid = cdata.split("_")[1]
        if sid in data["recycle_bin"]:
            del data["recycle_bin"][sid]
            save_data(data)
            ack(cid, "❌ স্থায়ীভাবে মুছে ফেলা হয়েছে!")
            list_schedules_as_buttons(chat_id, "recycle_bin", msg_id)
    elif cdata.startswith("edit_"):
        ack(cid)
        sid = cdata.split("_")[1]
        choose_edit_field(chat_id, sid, msg_id)
    elif cdata.startswith("edfield_"):
        ack(cid)
        _, field, sid = cdata.split("_")
        handle_field_edit(chat_id, sid, field, msg_id)
    elif cdata.startswith("edmode_"):
        ack(cid)
        _, mode, sid = cdata.split("_")
        s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
        if s:
            bot.clear_step_handler_by_chat_id(chat_id)
            if mode == "daily":
                msg = smart_send_or_edit(chat_id, msg_id, "⏰ **নতুন ডেইলি সময় লিখুন** (যেমন: `12:30 PM` অথবা `14:30`):")
                bot.register_next_step_handler(msg, lambda m: process_specific_time(m, "daily", sid))
            elif mode == "once":
                ask_once_year(chat_id, sid, msg_id)
            else:
                ask_interval_minutes(chat_id, sid, msg_id)
    elif cdata == "default_btns":
        ack(cid)
        show_default_buttons_menu(chat_id, msg_id)
    elif cdata == "add_def_btn":
        ack(cid)
        msg = bot.send_message(chat_id, "➕ ডিফল্ট বাটনের নাম কী হবে লিখুন:")
        bot.register_next_step_handler(msg, process_def_btn_name)
    elif cdata.startswith("setdefcol_"):
        col = cdata.split("_")[1]
        if chat_id in temp_schedule and "temp_def_btn" in temp_schedule[chat_id]:
            b_obj = temp_schedule[chat_id].pop("temp_def_btn")
            b_obj["color"] = col
            data.setdefault("default_buttons", []).append(b_obj)
            save_data(data)
            ack(cid, "✅ ডিফল্ট বাটন কালারসহ সেভ হয়েছে!")
            show_default_buttons_menu(chat_id, msg_id)
    elif cdata.startswith("deldef_"):
        idx = int(cdata.split("_")[1])
        if 0 <= idx < len(data.get("default_buttons", [])):
            data["default_buttons"].pop(idx)
            save_data(data)
            ack(cid, "ডিফল্ট বাটনটি রিমুভ করা হয়েছে!")
            show_default_buttons_menu(chat_id, msg_id)
    elif cdata == "clear_def_btns":
        data["default_buttons"] = []
        save_data(data)
        ack(cid, "সব ডিফল্ট বাটন মুছে ফেলা হয়েছে!")
        show_default_buttons_menu(chat_id, msg_id)
    else:
        ack(cid)

def get_interval_target_dict(chat_id, ctx):
    if ctx == "new":
        return temp_schedule.get(chat_id)
    return data["schedules"].get(ctx) or data["recycle_bin"].get(ctx)

def extract_message_data(message):
    content_data = {
        "from_chat": message.chat.id,
        "msg_id": message.message_id
    }
    raw_entities = message.json.get("entities") or message.json.get("caption_entities")
    content_data["entities"] = raw_entities
    if message.voice:
        content_data["type"] = "voice"
        content_data["file_id"] = message.voice.file_id
        content_data["text"] = message.caption or ""
    elif message.video:
        content_data["type"] = "video"
        content_data["file_id"] = message.video.file_id
        content_data["text"] = message.caption or ""
    elif message.sticker:
        content_data["type"] = "sticker"
        content_data["file_id"] = message.sticker.file_id
        content_data["text"] = ""
    elif message.photo:
        content_data["type"] = "photo"
        content_data["file_id"] = message.photo[-1].file_id
        content_data["text"] = message.caption or ""
    else:
        content_data["type"] = "text"
        content_data["file_id"] = None
        content_data["text"] = message.text or ""
    return content_data

def process_channel(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    temp_schedule[chat_id] = {
        "channel": (message.text or "").strip(),
        "buttons": [],
        "use_default_btns": True if data.get("default_buttons") else False,
        "reactions": data.get("default_reactions_on", True),
        "forward_mode": False
    }
    msg = bot.send_message(
        chat_id,
        "২. যা পোস্ট করতে চান তা পাঠান:\n"
        "• টেক্সট (প্রিমিয়াম লাইভ ইমোজি ও কাস্টম ইমোজি সহ)\n"
        "• ছবি, ভিডিও, ভয়েস অথবা স্টিকার\n\n"
        "সরাসরি সেন্ড বা ফরোয়ার্ড করুন:"
    )
    bot.register_next_step_handler(msg, process_content)

def process_content(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    if chat_id not in temp_schedule:
        show_main_menu(chat_id)
        return
    temp_schedule[chat_id].update(extract_message_data(message))
    show_step3_button_menu(chat_id)

def show_step3_button_menu(chat_id, msg_id=None):
    if chat_id not in temp_schedule:
        show_main_menu(chat_id, msg_id)
        return
    s = temp_schedule[chat_id]
    btns = s.get("buttons", [])
    rx_on = s.get("reactions", True)
    def_on = s.get("use_default_btns", False)
    def_cnt = len(data.get("default_buttons", []))
    btn_summary = ""
    if btns:
        btn_summary = "\n📌 **যুক্ত করা বাটনগুলো:**\n"
        for i, b in enumerate(btns, 1):
            c_lbl = COLOR_PALETTE.get(b.get("color", "green"), ("সবুজ", None))[0]
            btn_summary += f"  {i}. **{b['text']}**\n     🎨 কালার: {c_lbl} | 🔗 `{b['url']}`\n"
    else:
        btn_summary = "\n📌 **স্পেসিফিক বাটন:** এখনো যুক্ত করা হয়নি।\n"
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(make_btn("➕ নতুন URL বাটন যোগ করুন (যত খুশি)", "green", callback_data="s3_add_btn"))
    if def_cnt > 0:
        def_lbl = f"চালু আছে ({def_cnt}টি)" if def_on else f"বন্ধ আছে ({def_cnt}টি)"
        markup.add(make_btn(f"⚙️ ডিফল্ট বাটন: {def_lbl} (টগল করুন)", "blue", callback_data="s3_toggle_def"))
    rx_lbl = "চালু আছে 🟢 (👍 ❤️ 🔥 😍)" if rx_on else "বন্ধ আছে 🔴"
    markup.add(make_btn(f"👍 নিচে রিঅ্যাকশন বাটন: {rx_lbl}", "blue", callback_data="s3_toggle_rx"))
    if btns:
        markup.add(make_btn("🗑️ যুক্ত করা বাটনগুলো মুছুন", "red", callback_data="s3_clear_btns"))
    markup.add(make_btn("✅ বাটন সেটআপ সম্পন্ন ➔ পরবর্তী ধাপ (সময় নির্ধারণ)", "green", callback_data="btn_done_next"))
    text = (
        f"৩. 🔗 **পোস্টের বাটন ও লাইক/রিঅ্যাকশন সেটআপ:**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━"
        f"{btn_summary}"
        f"⚙️ **ডিফল্ট বাটন:** {'চালু 🟢' if def_on else 'বন্ধ 🔴'}\n"
        f"👍 **রিঅ্যাকশন বাটন (👍 ❤️ 🔥 😍):** {'চালু 🟢' if rx_on else 'বন্ধ 🔴'}\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"💡 *আপনি সরাসরি বাটনের নাম লিখে পাঠাতে পারেন অথবা নিচের বাটন থেকে অপশন বেছে নিন:*"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, process_button_name)

def process_button_name(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    if chat_id not in temp_schedule:
        show_main_menu(chat_id)
        return
    if message.text:
        b_name = message.text.strip()
        temp_schedule[chat_id]["temp_btn_name"] = b_name
        msg = bot.send_message(chat_id, f"🔗 **'{b_name}'** বাটনের ইউআরএল (URL) লিংকটি দিন\n(যেমন: `https://t.me/...` বা `https://google.com`):", parse_mode="Markdown")
        bot.register_next_step_handler(msg, process_button_url)
    else:
        msg = bot.send_message(chat_id, "❌ দয়া করে বাটনের নাম টেক্সট আকারে লিখুন:")
        bot.register_next_step_handler(msg, process_button_name)

def process_button_url(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    if chat_id not in temp_schedule:
        show_main_menu(chat_id)
        return
    url = clean_url(message.text)
    b_name = temp_schedule[chat_id].pop("temp_btn_name", "Download Now")
    temp_schedule[chat_id]["pending_btn"] = {"text": b_name, "url": url, "color": "green"}
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        make_btn("🟢 সবুজ (Green)", "green", callback_data="s3col_green"),
        make_btn("🔵 নীল (Blue)", "blue", callback_data="s3col_blue"),
        make_btn("🔴 লাল (Red)", "red", callback_data="s3col_red"),
        make_btn("⚪ ডিফল্ট (No Color)", "none", callback_data="s3col_none")
    )
    bot.send_message(chat_id, f"🎨 **'{b_name}'** বাটনটির ব্যাকগ্রাউন্ড কালার সিলেক্ট করুন:", reply_markup=markup, parse_mode="Markdown")

def ask_time_method(chat_id, msg_id=None):
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        make_btn("⏰ প্রতিদিন নির্দিষ্ট সময়ে (ডেইলি আনলিমিটেড)", "green", callback_data="time_mode_daily"),
        make_btn("🔄 নির্দিষ্ট মিনিট/ঘণ্টা পর পর (১০, ১৫, ২০, ৩০, ৫০মি. বা ১ ঘণ্টা)", "blue", callback_data="time_mode_interval"),
        make_btn("🎯 নির্দিষ্ট সময়ে একবার (পোস্ট হয়ে ডিলিট ফাইলে যাবে)", "red", callback_data="time_mode_once")
    )
    smart_send_or_edit(chat_id, msg_id, "৪. **পোস্টের সময় নির্ধারণ পদ্ধতি (Schedule Mode) বেছে নিন:**", reply_markup=markup)

def ask_once_year(chat_id, ctx="new", msg_id=None):
    now_bd = datetime.now(BD_TZ)
    cur_yr = now_bd.year
    markup = InlineKeyboardMarkup(row_width=2)
    markup.row(make_btn(f"⚡ আজকের তারিখেই পোস্ট করুন ({now_bd.strftime('%d %b %Y')})", "green", callback_data=f"once_today_{ctx}"))
    markup.row(
        make_btn(f"📅 {cur_yr} সাল (চলতি বছর)", "blue", callback_data=f"once_yr_{cur_yr}_{ctx}"),
        make_btn(f"📅 {cur_yr + 1} সাল", "blue", callback_data=f"once_yr_{cur_yr + 1}_{ctx}")
    )
    markup.row(
        make_btn(f"📅 {cur_yr + 2} সাল", "none", callback_data=f"once_yr_{cur_yr + 2}_{ctx}"),
        make_btn(f"📅 {cur_yr + 3} সাল", "none", callback_data=f"once_yr_{cur_yr + 3}_{ctx}")
    )
    text = (
        f"🎯 **নির্দিষ্ট সময়ে একবার পোস্ট (ডিলিট ফাইলে যাবে):**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"১️⃣ **প্রথমে কোন সালে (Year) মেসেজটি পোস্ট হবে তা সিলেক্ট করুন:**\n\n"
        f"• নিচের বাটন থেকে সাল সিলেক্ট করুন অথবা চ্যাটে যেকোনো সাল লিখুন (যেমন: `{cur_yr}`)\n"
        f"• সরাসরি মাসের নামও লিখতে পারেন (যেমন: `October` বা `November`)\n"
        f"• অথবা এক লাইনে সম্পূর্ণ তারিখ ও সময় লিখতে পারেন (যেমন: `October 23. 5:36 AM`)"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, lambda m: process_once_year_input(m, ctx))

def process_once_year_input(message, ctx="new"):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id)
        return
    raw = (message.text or "").strip()
    full_dt = parse_full_once_datetime(raw)
    if full_dt:
        once_date, parsed_time = full_dt
        target_dict["once_date"] = once_date
        finish_once_schedule(chat_id, parsed_time, ctx)
        return
    mo = parse_month_value(raw)
    if mo and not norm_digits(raw).isdigit():
        target_dict["temp_once_year"] = datetime.now(BD_TZ).year
        target_dict["temp_once_month"] = mo
        ask_once_day(chat_id, ctx)
        return
    cleaned = norm_digits(raw)
    if cleaned.isdigit() and 2024 <= int(cleaned) <= 2100:
        target_dict["temp_once_year"] = int(cleaned)
        ask_once_month(chat_id, ctx)
        return
    msg = bot.send_message(
        chat_id,
        "❌ সঠিক সাল লিখুন (যেমন: `2026`), অথবা মাসের নাম লিখুন (যেমন: `October`), অথবা নিচের বাটন থেকে সিলেক্ট করুন:",
        parse_mode="Markdown"
    )
    bot.register_next_step_handler(msg, lambda m: process_once_year_input(m, ctx))

def ask_once_month(chat_id, ctx="new", msg_id=None):
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id, msg_id)
        return
    yr = target_dict.get("temp_once_year", datetime.now(BD_TZ).year)
    markup = InlineKeyboardMarkup(row_width=2)
    for i in range(0, len(MONTH_LIST), 2):
        m1_num, m1_lbl = MONTH_LIST[i]
        m2_num, m2_lbl = MONTH_LIST[i + 1]
        markup.row(
            make_btn(f"🗓️ {m1_lbl}", "blue", callback_data=f"once_mo_{m1_num}_{ctx}"),
            make_btn(f"🗓️ {m2_lbl}", "blue", callback_data=f"once_mo_{m2_num}_{ctx}")
        )
    text = (
        f"✅ সাল সিলেক্ট হয়েছে: **{yr}**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"২️⃣ **এখন কোন মাসে (Month) মেসেজটি পোস্ট হবে সিলেক্ট করুন:**\n\n"
        f"নিচের বাটন থেকে মাস সিলেক্ট করুন অথবা চ্যাটে মাসের নাম/নম্বর লিখুন (যেমন: `October`, `November` বা `10`):"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, lambda m: process_once_month_input(m, ctx))

def process_once_month_input(message, ctx="new"):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id)
        return
    raw = (message.text or "").strip()
    full_dt = parse_full_once_datetime(raw)
    if full_dt:
        once_date, parsed_time = full_dt
        target_dict["once_date"] = once_date
        finish_once_schedule(chat_id, parsed_time, ctx)
        return
    mo = parse_month_value(raw)
    if mo:
        target_dict["temp_once_month"] = mo
        ask_once_day(chat_id, ctx)
    else:
        msg = bot.send_message(chat_id, "❌ মাসের নাম সঠিক নয়! (যেমন: `October`, `November` অথবা `1` থেকে `12` লিখুন বা বাটন চাপুন):", parse_mode="Markdown")
        bot.register_next_step_handler(msg, lambda m: process_once_month_input(m, ctx))

def ask_once_day(chat_id, ctx="new", msg_id=None):
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id, msg_id)
        return
    yr = target_dict.get("temp_once_year", datetime.now(BD_TZ).year)
    mo = target_dict.get("temp_once_month", datetime.now(BD_TZ).month)
    m_lbl = dict(MONTH_LIST).get(mo, str(mo))
    max_days = calendar.monthrange(yr, mo)[1]
    markup = InlineKeyboardMarkup(row_width=5)
    row_btns = []
    for d in range(1, max_days + 1):
        row_btns.append(make_btn(str(d), "green", callback_data=f"once_day_{d}_{ctx}"))
        if len(row_btns) == 5:
            markup.row(*row_btns)
            row_btns = []
    if row_btns:
        markup.row(*row_btns)
    text = (
        f"✅ সাল ও মাস: **{m_lbl}, {yr}**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"৩️⃣ **এখন কয় তারিখে (Date: 1 - {max_days}) মেসেজটি পোস্ট হবে সিলেক্ট করুন:**\n\n"
        f"নিচের বাটন থেকে তারিখ সিলেক্ট করুন অথবা চ্যাটে তারিখটি লিখুন (যেমন: `23`):"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, lambda m: process_once_day_input(m, ctx))

def process_once_day_input(message, ctx="new"):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id)
        return
    raw = (message.text or "").strip()
    full_dt = parse_full_once_datetime(raw)
    if full_dt:
        once_date, parsed_time = full_dt
        target_dict["once_date"] = once_date
        finish_once_schedule(chat_id, parsed_time, ctx)
        return
    yr = target_dict.get("temp_once_year", datetime.now(BD_TZ).year)
    mo = target_dict.get("temp_once_month", datetime.now(BD_TZ).month)
    max_days = calendar.monthrange(yr, mo)[1]
    cleaned = norm_digits(raw)
    if cleaned.isdigit() and 1 <= int(cleaned) <= max_days:
        dy = int(cleaned)
        target_dict["temp_once_day"] = dy
        target_dict["once_date"] = f"{yr:04d}-{mo:02d}-{dy:02d}"
        ask_once_time(chat_id, ctx)
    else:
        msg = bot.send_message(chat_id, f"❌ দয়া করে `1` থেকে `{max_days}` এর মধ্যে সঠিক তারিখ লিখুন অথবা বাটন থেকে সিলেক্ট করুন:", parse_mode="Markdown")
        bot.register_next_step_handler(msg, lambda m: process_once_day_input(m, ctx))

def ask_once_time(chat_id, ctx="new", msg_id=None):
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id, msg_id)
        return
    once_date = target_dict.get("once_date", datetime.now(BD_TZ).strftime("%Y-%m-%d"))
    d_disp = format_display_date(once_date)
    markup = InlineKeyboardMarkup(row_width=2)
    markup.row(
        make_btn("🕘 09:00 AM", "blue", callback_data=f"once_qtime_09:00_{ctx}"),
        make_btn("🕛 12:00 PM", "blue", callback_data=f"once_qtime_12:00_{ctx}")
    )
    markup.row(
        make_btn("🕒 03:00 PM", "blue", callback_data=f"once_qtime_15:00_{ctx}"),
        make_btn("🕕 06:00 PM", "blue", callback_data=f"once_qtime_18:00_{ctx}")
    )
    markup.row(
        make_btn("🕘 09:30 PM", "blue", callback_data=f"once_qtime_21:30_{ctx}"),
        make_btn("🕙 10:00 PM", "blue", callback_data=f"once_qtime_22:00_{ctx}")
    )
    text = (
        f"📅 নির্ধারিত তারিখ: **{d_disp}**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"৪️⃣ **নির্দিষ্ট কোন সময়ে (কয়টা বাজে) মেসেজটি পোস্ট হবে?**\n"
        f"(পোস্ট হওয়ার পর এটি ডিলিট ফাইলে চলে যাবে)\n\n"
        f"• চ্যাটে যেকোনো সময় লিখে পাঠান (যেমন: `05:36 AM`, `12:00 PM`, `02:30 PM` অথবা `14:30`)\n"
        f"• অথবা নিচের কুইক বাটন থেকে সময় সিলেক্ট করুন:"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, lambda m: process_specific_time(m, "once", ctx))

def finish_once_schedule(chat_id, parsed_time, ctx="new", msg_id=None):
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id, msg_id)
        return
    target_dict.pop("temp_once_year", None)
    target_dict.pop("temp_once_month", None)
    target_dict.pop("temp_once_day", None)
    if not target_dict.get("once_date"):
        target_dict["once_date"] = datetime.now(BD_TZ).strftime("%Y-%m-%d")
    target_dict["mode"] = "once"
    target_dict["time"] = parsed_time
    target_dict["last_posted"] = ""
    if ctx == "new":
        finalize_or_save_new_schedule(chat_id, msg_id)
    else:
        save_data(data)
        d_disp = format_display_date(target_dict["once_date"])
        bot.send_message(
            chat_id,
            f"✅ নির্দিষ্ট তারিখ ও সময় সফলভাবে আপডেট হয়েছে!\n📅 **তারিখ:** {d_disp}\n🕒 **সময়:** `{format_display_time(parsed_time)}`",
            parse_mode="Markdown"
        )
        target_loc = "schedules" if ctx in data["schedules"] else "recycle_bin"
        show_schedule_detail(chat_id, ctx, target_loc, msg_id)

def process_specific_time(message, mode="daily", ctx="new"):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    raw_txt = (message.text or "").strip()
    if mode == "once":
        full_dt = parse_full_once_datetime(raw_txt)
        if full_dt:
            once_date, parsed_t = full_dt
            target_dict = get_interval_target_dict(chat_id, ctx)
            if target_dict is not None:
                target_dict["once_date"] = once_date
                finish_once_schedule(chat_id, parsed_t, ctx)
                return
    parsed_time = parse_user_time(raw_txt)
    if not parsed_time:
        msg = bot.send_message(chat_id, "❌ সময়ের ফরম্যাট ভুল! (উদাহরণ: `12:00 PM` বা `02:30 PM` অথবা `14:30`) আবার লিখুন:", parse_mode="Markdown")
        bot.register_next_step_handler(msg, lambda m: process_specific_time(m, mode, ctx))
        return
    if mode == "once":
        finish_once_schedule(chat_id, parsed_time, ctx)
        return
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id)
        return
    target_dict["mode"] = mode
    target_dict["time"] = parsed_time
    target_dict["last_posted"] = ""
    if ctx == "new":
        finalize_or_save_new_schedule(chat_id)
    else:
        save_data(data)
        bot.send_message(chat_id, f"✅ সময় সফলভাবে `{format_display_time(parsed_time)}` আপডেট হয়েছে!", parse_mode="Markdown")
        target_loc = "schedules" if ctx in data["schedules"] else "recycle_bin"
        show_schedule_detail(chat_id, ctx, target_loc)

def ask_interval_minutes(chat_id, ctx="new", msg_id=None):
    markup = InlineKeyboardMarkup(row_width=3)
    markup.add(
        make_btn("১০ মিনিট", "green", callback_data=f"qint_10_{ctx}"),
        make_btn("১৫ মিনিট", "green", callback_data=f"qint_15_{ctx}"),
        make_btn("২০ মিনিট", "blue", callback_data=f"qint_20_{ctx}"),
        make_btn("৩০ মিনিট", "blue", callback_data=f"qint_30_{ctx}"),
        make_btn("৫০ মিনিট", "red", callback_data=f"qint_50_{ctx}"),
        make_btn("১ ঘণ্টা (৬০মি.)", "red", callback_data=f"qint_60_{ctx}")
    )
    prompt_txt = (
        "🔄 **কত মিনিট বা ঘণ্টা পর পর মেসেজ পোস্ট করতে চান?**\n\n"
        "নিচের বাটন থেকে সিলেক্ট করুন অথবা চ্যাটে যেকোনো সংখ্যা (যেমন: `10`, `15`, `20`, `30`, `50`, `60`) লিখে পাঠান:"
    )
    msg = smart_send_or_edit(chat_id, msg_id, prompt_txt, reply_markup=markup)
    bot.register_next_step_handler(msg, lambda m: process_interval_custom_minutes(m, ctx))

def process_interval_custom_minutes(message, ctx="new"):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    try:
        minutes = int(norm_digits(message.text))
        if minutes <= 0:
            raise ValueError
    except Exception:
        msg = bot.send_message(chat_id, "❌ দয়া করে শুধু একটি সঠিক সংখ্যা লিখুন (যেমন: 10, 15, 20, 30, 50 বা 60):")
        bot.register_next_step_handler(msg, lambda m: process_interval_custom_minutes(m, ctx))
        return
    set_interval_minutes_and_ask_start(chat_id, minutes, ctx)

def set_interval_minutes_and_ask_start(chat_id, minutes, ctx="new", msg_id=None):
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id)
        return
    target_dict["mode"] = "interval"
    target_dict["interval"] = minutes
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(make_btn("⚡ এখন থেকেই শুরু করুন (২৪ ঘণ্টা টানা চলবে)", "green", callback_data=f"intstart_now_{ctx}"))
    int_lbl = "১ ঘণ্টা (৬০ মিনিট)" if minutes == 60 else f"{minutes} মিনিট"
    text = (
        f"🟢 **প্রতি {int_lbl} পর পর মেসেজ পাঠানো কোন সময় থেকে শুরু হবে (Start Time)?**\n\n"
        f"• নির্দিষ্ট সময় থেকে শুরু করতে চাইলে সময়টি লিখুন (যেমন: `09:00 AM`, `02:00 PM` অথবা `14:00`)\n"
        f"• অথবা এখন থেকেই শুরু করতে নিচের বাটনে ক্লিক করুন:"
    )
    msg = smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)
    bot.register_next_step_handler(msg, lambda m: process_interval_start_time(m, ctx))

def process_interval_start_time(message, ctx="new"):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    parsed_start = parse_user_time(message.text)
    if not parsed_start:
        msg = bot.send_message(chat_id, "❌ সময়ের ফরম্যাট সঠিক নয়! (উদাহরণ: `09:00 AM`, `02:30 PM` অথবা `14:30`) আবার লিখুন:", parse_mode="Markdown")
        bot.register_next_step_handler(msg, lambda m: process_interval_start_time(m, ctx))
        return
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id)
        return
    target_dict["start_time"] = parsed_start
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        make_btn("🌙 রাত ১১:৫৯ পর্যন্ত চলবে (সারাদিন)", "blue", callback_data=f"intend_2359_{ctx}"),
        make_btn("♾️ কোনো শেষ সময় নেই (টানা চলবে)", "green", callback_data=f"intend_none_{ctx}")
    )
    text = (
        f"✅ শুরুর সময় (Start Time): **{format_display_time(parsed_start)}** সেট হয়েছে।\n\n"
        f"🔴 **এখন মেসেজ পাঠানো কোন সময়ে শেষ/বন্ধ হবে (End Time)?**\n"
        f"• শেষ করার সময়টি লিখে পাঠান (যেমন: `10:00 PM`, `11:30 PM` অথবা `23:00`)\n"
        f"• অথবা নিচের বাটন থেকে বেছে নিন:"
    )
    msg = bot.send_message(chat_id, text, reply_markup=markup, parse_mode="Markdown")
    bot.register_next_step_handler(msg, lambda m: process_interval_end_time(m, ctx))

def process_interval_end_time(message, ctx="new"):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    parsed_end = parse_user_time(message.text)
    if not parsed_end:
        msg = bot.send_message(chat_id, "❌ সময়ের ফরম্যাট সঠিক নয়! (উদাহরণ: `10:00 PM`, `11:30 PM` অথবা `23:00`) আবার লিখুন:", parse_mode="Markdown")
        bot.register_next_step_handler(msg, lambda m: process_interval_end_time(m, ctx))
        return
    target_dict = get_interval_target_dict(chat_id, ctx)
    if target_dict is None:
        show_main_menu(chat_id)
        return
    target_dict["end_time"] = parsed_end
    ask_interval_duration_mode(chat_id, ctx)

def ask_interval_duration_mode(chat_id, ctx="new", msg_id=None):
    target_dict = get_interval_target_dict(chat_id, ctx)
    mins = target_dict.get("interval", 15) if target_dict else 15
    st = target_dict.get("start_time") if target_dict else None
    et = target_dict.get("end_time") if target_dict else None
    win_str = f"{format_display_time(st)} থেকে {format_display_time(et)}" if (st and et) else (f"{format_display_time(st)} থেকে শুরু" if st else "২৪ ঘণ্টা (এখন থেকেই)")
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        make_btn("♾️ প্রতিদিন চলবে (আজীবন / Indefinite Daily)", "green", callback_data=f"intdur_daily_{ctx}"),
        make_btn("1️⃣ শুধু একদিনের জন্য চলবে (Single Day)", "blue", callback_data=f"intdur_once_{ctx}")
    )
    text = (
        f"📅 **শিডিউলের মেয়াদ নির্ধারণ করুন:**\n\n"
        f"🔄 **ইন্টারভ্যাল:** প্রতি {mins} মিনিট পর পর\n"
        f"🕒 **সময়সীমা:** {win_str}\n\n"
        f"এই শিডিউলটি কি **প্রতিদিন (আজীবন)** চলবে নাকি **শুধু একদিনের জন্য** চলবে?"
    )
    smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)

def finalize_or_save_new_schedule(chat_id, msg_id=None):
    if chat_id in temp_schedule and "pending_btn" in temp_schedule[chat_id]:
        b_obj = temp_schedule[chat_id].pop("pending_btn")
        temp_schedule[chat_id].setdefault("buttons", []).append(b_obj)
    sid = save_new_schedule(chat_id)
    show_post_extra_options(chat_id, sid, msg_id)

def save_new_schedule(chat_id):
    base_ms = int(time.time() * 1000)
    sid = str(base_ms)[-10:]
    while sid in data.get("schedules", {}) or sid in data.get("recycle_bin", {}):
        base_ms += 1
        sid = str(base_ms)[-10:]
    sched_item = temp_schedule.pop(chat_id)
    sched_item["forward_mode"] = False
    if "reactions" not in sched_item:
        sched_item["reactions"] = data.get("default_reactions_on", True)
    data["schedules"][sid] = sched_item
    save_data(data)
    return sid

def show_post_extra_options(chat_id, sid, msg_id=None):
    s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
    if not s:
        show_main_menu(chat_id, msg_id)
        return
    t_str = format_schedule_summary(s, short=False)
    def_status = "চালু আছে 🟢" if s.get("use_default_btns") else "বন্ধ আছে 🔴"
    rx_status = "চালু আছে 🟢" if s.get("reactions", True) else "বন্ধ আছে 🔴"
    c_btns = s.get("buttons", [])
    btn_list_str = ""
    if c_btns:
        for idx, b in enumerate(c_btns, 1):
            c_lbl = COLOR_PALETTE.get(b.get("color", "green"), ("সবুজ", None))[0]
            btn_list_str += f"   {idx}. {b['text']}\n      🎨 কালার: {c_lbl} | 🔗 `{b['url']}`\n"
    text = (
        f"✅ **শিডিউল ও বাটন কনফিগারেশন সেভ হয়েছে!**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"📢 **চ্যানেল:** {s['channel']}\n"
        f"⏰ **শিডিউল:** {t_str}\n"
        f"🔗 **স্পেসিফিক বাটন:** {len(c_btns)} টি\n"
        f"{btn_list_str}"
        f"⚙️ **ডিফল্ট বাটন:** {def_status}\n"
        f"👍 **রিঅ্যাকশন বাটন (👍 ❤️ 🔥 😍):** {rx_status}\n\n"
        f"👇 *নিচের বাটন থেকে চাইলে এখনই চ্যানেলে টেস্ট পোস্ট করতে পারেন অথবা আরও বাটন যোগ করতে পারেন:*"
    )
    markup = InlineKeyboardMarkup(row_width=1)
    markup.add(
        make_btn("🚀 এখনই চ্যানেলে টেস্ট পোস্ট পাঠান", "green", callback_data=f"testpost_{sid}"),
        make_btn("➕ এই পোস্টে আরও URL বাটন যোগ করুন", "green", callback_data=f"addmorebtn_{sid}"),
        make_btn(f"👍 রিঅ্যাকশন বাটন ({rx_status}) - অন/অফ করুন", "blue", callback_data=f"togglerx_{sid}"),
        make_btn(f"⚙️ ডিফল্ট বাটন ({def_status}) - অন/অফ করুন", "blue", callback_data=f"toggledef_{sid}")
    )
    if c_btns:
        markup.add(make_btn("🗑️ স্পেসিফিক বাটনগুলো মুছে ফেলুন", "red", callback_data=f"btn_del_{sid}"))
    markup.add(make_btn("👑 মূল ড্যাশবোর্ডে ফিরে যান", "green", callback_data="main_menu"))
    smart_send_or_edit(chat_id, msg_id, text, reply_markup=markup)

def process_more_btn_name(message, sid):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    b_name = (message.text or "").strip()
    temp_schedule.setdefault(chat_id, {})["temp_more_btn"] = {"text": b_name}
    msg = bot.send_message(chat_id, f"🔗 **'{b_name}'** বাটনের ইউআরএল (URL) লিংকটি দিন:", parse_mode="Markdown")
    bot.register_next_step_handler(msg, lambda m: process_more_btn_url(m, sid))

def process_more_btn_url(message, sid):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    url = clean_url(message.text)
    temp_schedule[chat_id]["temp_more_btn"]["url"] = url
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        make_btn("🟢 সবুজ (Green)", "green", callback_data=f"edbtncol_green_{sid}"),
        make_btn("🔵 নীল (Blue)", "blue", callback_data=f"edbtncol_blue_{sid}"),
        make_btn("🔴 লাল (Red)", "red", callback_data=f"edbtncol_red_{sid}"),
        make_btn("⚪ ডিফল্ট (No Color)", "none", callback_data=f"edbtncol_none_{sid}")
    )
    bot.send_message(chat_id, "🎨 এই বাটনটির ব্যাকগ্রাউন্ড কালার সিলেক্ট করুন:", reply_markup=markup)

def show_default_buttons_menu(chat_id, msg_id=None):
    markup = InlineKeyboardMarkup(row_width=1)
    def_btns = data.get("default_buttons", [])
    if not def_btns:
        b_list_str = "বর্তমানে কোনো ডিফল্ট বাটন সেট করা নেই।"
    else:
        lines = []
        for idx, b in enumerate(def_btns):
            c_label = COLOR_PALETTE.get(b.get("color", "none"), ("ডিফল্ট", None))[0]
            lines.append(
                f"🔹 {idx+1}. বাটনের নাম: {b['text']}\n"
                f"   🎨 কালার: {c_label}\n"
                f"   🔗 লিংক: {b['url']}"
            )
            markup.add(make_btn(f"🗑️ রিমুভ করুন: {b['text']}", "red", callback_data=f"deldef_{idx}"))
        b_list_str = "\n\n".join(lines)
    markup.add(make_btn("➕ নতুন ডিফল্ট বাটন যোগ করুন", "green", callback_data="add_def_btn"))
    if def_btns:
        markup.add(make_btn("🗑️ সব ডিফল্ট বাটন মুছে ফেলুন", "red", callback_data="clear_def_btns"))
    markup.add(
        make_btn("✏️ এই পেজের স্ট্যাটাস ডিজাইন ও ইমোজি এডিট", "green", callback_data="edit_ui_default_btns"),
        make_btn("🔙 মূল ড্যাশবোর্ড", "blue", callback_data="main_menu")
    )
    send_ui_page(chat_id, msg_id, "default_btns", {"{def_btn_list}": b_list_str}, markup)

def process_def_btn_name(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    b_name = (message.text or "").strip()
    temp_schedule.setdefault(chat_id, {})["temp_def_btn"] = {"text": b_name}
    msg = bot.send_message(chat_id, f"🔗 **'{b_name}'** ডিফল্ট বাটনের ইউআরএল (URL) লিংকটি দিন:", parse_mode="Markdown")
    bot.register_next_step_handler(msg, process_def_btn_url)

def process_def_btn_url(message):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    chat_id = message.chat.id
    url = clean_url(message.text)
    temp_schedule[chat_id]["temp_def_btn"]["url"] = url
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        make_btn("🟢 সবুজ (Green)", "green", callback_data="setdefcol_green"),
        make_btn("🔵 নীল (Blue)", "blue", callback_data="setdefcol_blue"),
        make_btn("🔴 লাল (Red)", "red", callback_data="setdefcol_red"),
        make_btn("⚪ ডিফল্ট (No Color)", "none", callback_data="setdefcol_none")
    )
    bot.send_message(chat_id, "🎨 ডিফল্ট বাটনটির ব্যাকগ্রাউন্ড কালার সিলেক্ট করুন:", reply_markup=markup)

def list_schedules_as_buttons(chat_id, target="schedules", msg_id=None):
    source = data.get(target, {})
    markup = InlineKeyboardMarkup(row_width=1)
    page_key = "list_schedule" if target == "schedules" else "recycle_bin"
    if not source:
        empty_msg = "📋 কোনো অ্যাক্টিভ শিডিউল সেট করা নেই।" if target == "schedules" else "🗑️ ডিলিট ফাইলে কোনো শিডিউল নেই।"
        markup.add(
            make_btn("✏️ এই পেজের স্ট্যাটাস ডিজাইন ও ইমোজি এডিট", "blue", callback_data=f"edit_ui_{page_key}"),
            make_btn("🔙 মূল ড্যাশবোর্ড", "green", callback_data="main_menu")
        )
        smart_send_or_edit(chat_id, msg_id, empty_msg, reply_markup=markup)
        return
    for sid, s in source.items():
        t_label = format_schedule_summary(s, short=True)
        b_cnt = len(s.get("buttons", [])) + (len(data.get("default_buttons", [])) if s.get("use_default_btns") else 0)
        btn_text = f"📢 {s['channel']} | 🕒 {t_label} | 🔗{b_cnt}"
        cb = f"view_{sid}" if target == "schedules" else f"viewbin_{sid}"
        col = "blue" if target == "schedules" else "red"
        markup.add(make_btn(btn_text, col, callback_data=cb))
    markup.add(
        make_btn("✏️ এই পেজের স্ট্যাটাস ডিজাইন ও ইমোজি এডিট", "blue", callback_data=f"edit_ui_{page_key}"),
        make_btn("🔙 মূল ড্যাশবোর্ড", "green", callback_data="main_menu")
    )
    repl = {"{total_sched}": len(source)} if target == "schedules" else {"{recycle_cnt}": len(source)}
    send_ui_page(chat_id, msg_id, page_key, repl, markup)

def show_schedule_detail(chat_id, sid, target="schedules", msg_id=None):
    s = data.get(target, {}).get(sid)
    if not s:
        bot.send_message(chat_id, "❌ শিডিউলটি পাওয়া যায়নি!")
        return
    stype = s.get("type", "text").upper()
    time_info = format_schedule_summary(s, short=False)
    c_btns = s.get("buttons", [])
    info_text = (
        f"📄 **শিডিউলের সম্পূর্ণ বিবরণ:**\n"
        f"━━━━━━━━━━━━━━━━━━━━━━\n"
        f"🔹 **চ্যানেল:** {s['channel']}\n"
        f"⏰ **শিডিউল টাইপ:** {time_info}\n"
        f"📌 **কন্টেন্ট টাইপ:** {stype}\n"
        f"🔗 **স্পেসিফিক বাটন:** {len(c_btns)} টি\n"
        f"⚙️ **ডিফল্ট বাটন:** {'চালু 🟢' if s.get('use_default_btns') else 'বন্ধ 🔴'}\n"
        f"👍 **রিঅ্যাকশন বাটন (👍 ❤️ 🔥 😍):** {'চালু 🟢' if s.get('reactions', True) else 'বন্ধ 🔴'}\n"
    )
    markup = InlineKeyboardMarkup(row_width=2)
    if target == "schedules":
        markup.row(make_btn("🚀 এখনই চ্যানেলে টেস্ট পোস্ট করুন", "green", callback_data=f"testpost_{sid}"))
        markup.row(
            make_btn("✏️ এডিট করুন", "blue", callback_data=f"edit_{sid}"),
            make_btn("🗑️ ডিলিট ফাইলে পাঠান", "red", callback_data=f"del_{sid}")
        )
        markup.row(make_btn("🔙 তালিকায় ফিরুন", "none", callback_data="list_schedule"))
    else:
        markup.row(
            make_btn("♻️ রিস্টোর করুন", "green", callback_data=f"restore_{sid}"),
            make_btn("✏️ এডিট করুন", "blue", callback_data=f"edit_{sid}")
        )
        markup.row(
            make_btn("❌ চিরতরে ডিলিট করুন", "red", callback_data=f"permdel_{sid}"),
            make_btn("🔙 ডিলিট ফাইলে ফিরুন", "none", callback_data="recycle_bin")
        )
    smart_send_or_edit(chat_id, msg_id, info_text, reply_markup=markup)
    threading.Thread(target=send_scheduled_post, args=(s, sid, chat_id), daemon=True).start()

def choose_edit_field(chat_id, sid, msg_id=None):
    back_cb = f"view_{sid}" if sid in data["schedules"] else f"viewbin_{sid}"
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        make_btn("📝 মেসেজ ফরোয়ার্ড ও এডিট", "blue", callback_data=f"edfield_text_{sid}"),
        make_btn("⏰ সময়, ইন্টারভ্যাল ও মেয়াদ পরিবর্তন", "green", callback_data=f"edfield_time_{sid}"),
        make_btn("📢 চ্যানেল পরিবর্তন", "blue", callback_data=f"edfield_channel_{sid}"),
        make_btn("🔗 বাটন ও রিঅ্যাকশন সেটআপ", "green", callback_data=f"edfield_btn_{sid}"),
        make_btn("🔙 ফিরে যান", "red", callback_data=back_cb)
    )
    smart_send_or_edit(chat_id, msg_id, "✏️ **আপনি এই শিডিউলের কোনটি পরিবর্তন করতে চান?**", reply_markup=markup)

def handle_field_edit(chat_id, sid, field, msg_id=None):
    s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
    if not s:
        return
    if field == "text":
        bot.send_message(chat_id, "👇 **আপনার সেট করা মেসেজটি নিচে ফরোয়ার্ড করে দেওয়া হলো।**\nএখান থেকে মেসেজটি এডিট করে বা নতুন করে যে মেসেজটি দিতে চান তা পাঠিয়ে দিন:", parse_mode="Markdown")
        threading.Thread(target=send_scheduled_post, args=(s, sid, chat_id, True), daemon=True).start()
        bot.register_next_step_handler_by_chat_id(chat_id, lambda m: update_text(m, sid))
    elif field == "time":
        markup = InlineKeyboardMarkup(row_width=1)
        markup.add(
            make_btn("⏰ প্রতিদিন নির্দিষ্ট সময়ে (ডেইলি আনলিমিটেড)", "green", callback_data=f"edmode_daily_{sid}"),
            make_btn("🔄 নির্দিষ্ট মিনিট/ঘণ্টা পর পর (Start/End ও মেয়াদ সহ)", "blue", callback_data=f"edmode_interval_{sid}"),
            make_btn("🎯 নির্দিষ্ট সময়ে একবার (ডিলিট ফাইলে যাবে)", "red", callback_data=f"edmode_once_{sid}")
        )
        smart_send_or_edit(chat_id, msg_id, "⏰ **নতুন শিডিউল টাইপ বেছে নিন:**", reply_markup=markup)
    elif field == "channel":
        msg = bot.send_message(chat_id, "📢 নতুন চ্যানেলের ইউজারনেম বা আইডি দিন (যেমন: @newchannel):")
        bot.register_next_step_handler(msg, lambda m: update_channel(m, sid))
    elif field == "btn":
        show_post_extra_options(chat_id, sid, msg_id)

def update_text(message, sid):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
    if s:
        s.update(extract_message_data(message))
        s["forward_mode"] = False
        save_data(data)
        w_txt, w_ents = render_ui_page("save_welcome", {})
        bot.send_message(message.chat.id, w_txt, entities=w_ents)
        target = "schedules" if sid in data["schedules"] else "recycle_bin"
        show_schedule_detail(message.chat.id, sid, target)

def update_channel(message, sid):
    if message.text == "/start":
        show_main_menu(message.chat.id)
        return
    s = data["schedules"].get(sid) or data["recycle_bin"].get(sid)
    if s and message.text:
        s["channel"] = message.text.strip()
        save_data(data)
        bot.send_message(message.chat.id, "✅ চ্যানেল সফলভাবে আপডেট হয়েছে!")
        target = "schedules" if sid in data["schedules"] else "recycle_bin"
        show_schedule_detail(message.chat.id, sid, target)

def build_post_markup(sched, sid="0", rx_counts=None, plain_fallback=False):
    if not sched or sched.get("type") == "sticker":
        return None
    markup = InlineKeyboardMarkup(row_width=1)
    all_url_btns = []
    for b in sched.get("buttons", []):
        u = clean_url(b.get("url", ""))
        col = "none" if plain_fallback else b.get("color", "green")
        if plain_fallback:
            all_url_btns.append(InlineKeyboardButton(text=b["text"], url=u))
        else:
            all_url_btns.append(make_btn(b["text"], col, url=u))
    if sched.get("use_default_btns"):
        for b in data.get("default_buttons", []):
            u = clean_url(b.get("url", ""))
            col = "none" if plain_fallback else b.get("color", "green")
            if plain_fallback:
                all_url_btns.append(InlineKeyboardButton(text=b["text"], url=u))
            else:
                all_url_btns.append(make_btn(b["text"], col, url=u))
    for btn in all_url_btns:
        markup.row(btn)
    if sched.get("reactions", True):
        c = rx_counts or {"up": 0, "love": 0, "fire": 0, "wow": 0}
        if plain_fallback:
            markup.row(
                InlineKeyboardButton(f"👍 {c.get('up', 0)}", callback_data=f"rx_up_{sid}"),
                InlineKeyboardButton(f"❤️ {c.get('love', 0)}", callback_data=f"rx_love_{sid}"),
                InlineKeyboardButton(f"🔥 {c.get('fire', 0)}", callback_data=f"rx_fire_{sid}"),
                InlineKeyboardButton(f"😍 {c.get('wow', 0)}", callback_data=f"rx_wow_{sid}")
            )
        else:
            markup.row(
                make_btn(f"👍 {c.get('up', 0)}", "none", callback_data=f"rx_up_{sid}"),
                make_btn(f"❤️ {c.get('love', 0)}", "none", callback_data=f"rx_love_{sid}"),
                make_btn(f"🔥 {c.get('fire', 0)}", "none", callback_data=f"rx_fire_{sid}"),
                make_btn(f"😍 {c.get('wow', 0)}", "none", callback_data=f"rx_wow_{sid}")
            )
    return markup if markup.keyboard else None

def send_with_entities_and_markup(target_channel, sched, markup):
    stype = sched.get("type", "text")
    text_content = sched.get("text", "")
    entities_list = parse_entities(sched.get("entities"))
    if stype == "photo":
        bot.send_photo(target_channel, sched["file_id"], caption=text_content, caption_entities=entities_list, reply_markup=markup)
    elif stype == "voice":
        bot.send_voice(target_channel, sched["file_id"], caption=text_content, caption_entities=entities_list, reply_markup=markup)
    elif stype == "video":
        bot.send_video(target_channel, sched["file_id"], caption=text_content, caption_entities=entities_list, reply_markup=markup)
    elif stype == "sticker":
        bot.send_sticker(target_channel, sched["file_id"])
    else:
        bot.send_message(target_channel, text_content, entities=entities_list, reply_markup=markup)

def send_scheduled_post(sched, sid="0", custom_target_chat=None, without_buttons=False):
    target_channel = custom_target_chat or sched["channel"]
    markup = None if without_buttons else build_post_markup(sched, sid, plain_fallback=False)
    plain_markup = None if without_buttons else build_post_markup(sched, sid, plain_fallback=True)
    if sched.get("from_chat") and sched.get("msg_id"):
        try:
            bot.copy_message(
                chat_id=target_channel,
                from_chat_id=sched["from_chat"],
                message_id=sched["msg_id"],
                reply_markup=markup
            )
            return True
        except Exception as e1:
            print(f"copy_message (styled) failed: {e1}")
            if plain_markup:
                try:
                    bot.copy_message(
                        chat_id=target_channel,
                        from_chat_id=sched["from_chat"],
                        message_id=sched["msg_id"],
                        reply_markup=plain_markup
                    )
                    return True
                except Exception as e2:
                    print(f"copy_message (plain) failed: {e2}")
    try:
        send_with_entities_and_markup(target_channel, sched, markup)
        return True
    except Exception as e3:
        print(f"send_with_entities (styled) failed: {e3}")
        try:
            send_with_entities_and_markup(target_channel, sched, plain_markup)
            return True
        except Exception as e4:
            print(f"send_with_entities (plain) failed: {e4}")
    if markup is None and sched.get("from_chat") and sched.get("msg_id"):
        try:
            bot.forward_message(
                chat_id=target_channel,
                from_chat_id=sched["from_chat"],
                message_id=sched["msg_id"]
            )
            return True
        except Exception as e5:
            print(f"forward_message fallback failed: {e5}")
    return False

def run_scheduler():
    while True:
        try:
            now = datetime.now(BD_TZ)
            if data.get("is_running", True) and not get_active_pause_rule(now):
                now_str = now.strftime("%H:%M")
                today_str = now.strftime("%Y-%m-%d")
                curr_epoch = int(time.time())
                for sid, s in list(data.get("schedules", {}).items()):
                    mode = s.get("mode", "daily")
                    if mode == "daily":
                        if s.get("time") == now_str and s.get("last_posted") != today_str:
                            if send_scheduled_post(s, sid):
                                if sid in data.get("schedules", {}):
                                    data["schedules"][sid]["last_posted"] = today_str
                                    save_data(data)
                    elif mode == "once":
                        target_date = s.get("once_date")
                        date_ok = (target_date == today_str) if target_date else True
                        if date_ok and s.get("time") == now_str and s.get("last_posted") != today_str:
                            if send_scheduled_post(s, sid):
                                if sid in data.get("schedules", {}):
                                    s["last_posted"] = today_str
                                    data["recycle_bin"][sid] = data["schedules"].pop(sid)
                                    save_data(data)
                    elif mode == "interval":
                        st = s.get("start_time")
                        et = s.get("end_time")
                        rep = s.get("interval_repeat", "daily")
                        in_win = is_within_interval_window(now_str, st, et)
                        if rep == "once" and s.get("has_posted_once", False):
                            active_date = s.get("active_date", today_str)
                            if (not in_win) or (not et and today_str != active_date):
                                if sid in data.get("schedules", {}):
                                    data["recycle_bin"][sid] = data["schedules"].pop(sid)
                                    save_data(data)
                                continue
                        if in_win:
                            interval_sec = s.get("interval", 15) * 60
                            last_time = s.get("last_timestamp", 0)
                            if curr_epoch - last_time >= interval_sec:
                                if send_scheduled_post(s, sid):
                                    if sid in data.get("schedules", {}):
                                        data["schedules"][sid]["last_timestamp"] = curr_epoch
                                        data["schedules"][sid]["has_posted_once"] = True
                                        data["schedules"][sid]["active_date"] = today_str
                                        save_data(data)
                        else:
                            if st and s.get("last_timestamp", 0) != 0:
                                if sid in data.get("schedules", {}):
                                    data["schedules"][sid]["last_timestamp"] = 0
                                    save_data(data)
        except Exception as e:
            print(f"Scheduler error: {e}")
        time.sleep(5)

# ==================== MODERN WEB UI/UX DASHBOARD ====================
app = Flask(__name__)

@app.route('/')
def web_dashboard():
    schedules = data.get("schedules", {})
    recycle = data.get("recycle_bin", {})
    def_btns = data.get("default_buttons", [])
    pause_rules = data.get("pause_rules", [])
    now_bd = datetime.now(BD_TZ).strftime("%I:%M:%S %p | %d %b %Y")
    active_pause = get_active_pause_rule()
    
    if not data.get("is_running", True):
        status_txt, status_col = "বন্ধ 🔴 (Paused)", "#ef4444"
    elif active_pause:
        status_txt, status_col = "শিডিউল বিরতি চলছে ⏸️", "#f59e0b"
    else:
        status_txt, status_col = "সচল ও লাইভ 🟢", "#10b981"

    rows = ""
    for sid, s in schedules.items():
        t_sum = format_schedule_summary(s, short=True)
        stype = s.get("type", "text").upper()
        b_cnt = len(s.get("buttons", []))
        rows += f"""
        <tr>
            <td><code>#{sid[-4:]}</code></td>
            <td><strong>{s.get('channel')}</strong></td>
            <td><span class="tag">{stype}</span></td>
            <td>{t_sum}</td>
            <td>{b_cnt}টি বাটন</td>
        </tr>
        """
    if not rows:
        rows = '<tr><td colspan="5" style="text-align:center;color:#94a3b8;padding:24px;">কোনো অ্যাক্টিভ শিডিউল নেই</td></tr>'

    return f"""
    <!DOCTYPE html>
    <html lang="bn">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="refresh" content="15">
        <title>Battle Club Schedule Bot — Live Dashboard</title>
        <style>
            * {{ box-sizing: border-box; margin: 0; padding: 0; font-family: system-ui, -apple-system, sans-serif; }}
            body {{ background: #0f172a; color: #f8fafc; padding: 20px; }}
            .container {{ max-width: 960px; margin: 0 auto; }}
            .header {{ background: linear-gradient(135deg, #1e293b, #0f172a); border: 1px solid #334155; border-radius: 16px; padding: 24px; display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 12px; margin-bottom: 20px; }}
            .title {{ font-size: 1.4rem; font-weight: 700; color: #38bdf8; }}
            .badge {{ background: rgba(255,255,255,0.06); border: 1px solid {status_col}; color: {status_col}; padding: 8px 16px; border-radius: 999px; font-weight: 600; font-size: 0.9rem; }}
            .grid {{ display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 16px; margin-bottom: 24px; }}
            .card {{ background: #1e293b; border: 1px solid #334155; border-radius: 14px; padding: 20px; }}
            .card h4 {{ color: #94a3b8; font-size: 0.85rem; margin-bottom: 8px; }}
            .card .num {{ font-size: 1.9rem; font-weight: 700; color: #f8fafc; }}
            .table-box {{ background: #1e293b; border: 1px solid #334155; border-radius: 14px; overflow-x: auto; }}
            table {{ width: 100%; border-collapse: collapse; text-align: left; }}
            th, td {{ padding: 14px 18px; border-bottom: 1px solid #334155; font-size: 0.92rem; }}
            th {{ background: #0f172a; color: #38bdf8; }}
            .tag {{ background: #334155; color: #38bdf8; padding: 4px 10px; border-radius: 6px; font-size: 0.78rem; font-weight: 600; }}
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <div>
                    <div class="title">👑 ব্যাটল ক্লাব প্রো শিডিউল বট</div>
                    <div style="color:#94a3b8;font-size:0.85rem;margin-top:4px;">🕒 বাংলাদেশ সময়: {now_bd}</div>
                </div>
                <div class="badge">{status_txt}</div>
            </div>
            <div class="grid">
                <div class="card"><h4>📊 অ্যাক্টিভ শিডিউল</h4><div class="num">{len(schedules)}</div></div>
                <div class="card"><h4>🔗 ডিফল্ট বাটন</h4><div class="num">{len(def_btns)}</div></div>
                <div class="card"><h4>⏸️ বিরতি নিয়ম</h4><div class="num">{len(pause_rules)}</div></div>
                <div class="card"><h4>🗑️ রিসাইকেল বিন</h4><div class="num">{len(recycle)}</div></div>
            </div>
            <div class="table-box">
                <table>
                    <thead>
                        <tr><th>আইডি</th><th>চ্যানেল</th><th>টাইপ</th><th>শিডিউল সময়</th><th>বাটন</th></tr>
                    </thead>
                    <tbody>{rows}</tbody>
                </table>
            </div>
        </div>
    </body>
    </html>
    """

@app.route('/health')
def health():
    return jsonify({"status": "running", "schedules": len(data.get("schedules", {}))})

def run_web():
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port, use_reloader=False)

if __name__ == "__main__":
    threading.Thread(target=run_scheduler, daemon=True).start()
    threading.Thread(target=run_web, daemon=True).start()
    print("Battle Club Pro Bot & Web Dashboard is running 100%...")
    bot.infinity_polling(timeout=20, long_polling_timeout=10)
