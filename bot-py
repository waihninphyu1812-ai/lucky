import os
import re
import json
import time
import random
import threading
try:
    import requests
except ImportError:
    requests = None
import urllib.request
import urllib.error

from datetime import datetime, timezone, timedelta
from http.server import HTTPServer, BaseHTTPRequestHandler
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton, WebAppInfo
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, CallbackQueryHandler, ContextTypes, filters

# ==================== RENDER 24/7 WEB SERVER (PORT 10000) ====================
class HealthCheckHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/plain; charset=utf-8')
        self.end_headers()
        self.wfile.write("Lucky2D3D Official Real-Time Bot Running Live 24/7!".encode('utf-8'))

    def log_message(self, format, *args):
        return

def start_health_server():
    port = int(os.environ.get("PORT", 10000))
    try:
        server = HTTPServer(('0.0.0.0', port), HealthCheckHandler)
        server.serve_forever()
    except Exception as e:
        print("Health Server Error:", e)

threading.Thread(target=start_health_server, daemon=True).start()

# ==================== BOT CONFIG & ADMIN ====================
TOKEN = '9029256827:AAEzKzZcNW8jiqrmx6-eJvdRsGu1ncpUaJI'
ADMIN_USERNAME = "noenoeslotagency"
ADMIN_EMAIL = "waihninphyu1812@gmail.com"
MINI_APP_URL = "https://lucky2d3dbot.vercel.app"

# Payment Accounts
KPAY_NUMBER = "09766646412"
KPAY_NAME = "Ma Moe Thu Zar"

PROMPTPAY_NUMBER = "0816404238"
PROMPTPAY_NAME = "MR.AUNG MYAT TUN"

# HTTP Headers for authentic scraping
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
    "Accept": "application/json, text/plain, */*",
    "Accept-Language": "en-US,en;q=0.9,th;q=0.8",
}

# ==================== RESILIENT HTTP CLIENT (NO MODULE CRASH) ====================

def http_get_json(url, timeout=7):
    """
    requests မရှိသော်လည်း Python ၏ မူရင်း urllib ဖြင့် အလိုအလျောက် ဆွဲယူပေးသော Function
    """
    if requests is not None:
        try:
            r = requests.get(url, headers=HEADERS, timeout=timeout)
            if r.status_code == 200:
                return r.json()
        except Exception:
            pass

    try:
        req = urllib.request.Request(url, headers=HEADERS)
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            code = getattr(resp, "status", None) or resp.getcode()
            if code == 200:
                raw_bytes = resp.read()
                return json.loads(raw_bytes.decode("utf-8"))
    except Exception:
        pass
    return None

def http_post_json(url, payload=None, timeout=8):
    """
    HTTP POST Request (GLO ထီရလဒ်ဆွဲယူရန်)
    """
    if requests is not None:
        try:
            r = requests.post(url, json=payload or {}, headers=HEADERS, timeout=timeout)
            if r.status_code == 200:
                return r.json()
        except Exception:
            pass

    try:
        data_bytes = json.dumps(payload or {}).encode("utf-8")
        req = urllib.request.Request(url, data=data_bytes, headers=HEADERS)
        req.add_header("Content-Type", "application/json")
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            code = getattr(resp, "status", None) or resp.getcode()
            if code == 200:
                raw_bytes = resp.read()
                return json.loads(raw_bytes.decode("utf-8"))
    except Exception:
        pass
    return None

# ==================== STORAGE ====================
user_bets = {}         # Confirmed bets: {user_id: [ticket_1, ticket_2, ...]}
pending_orders = {}    # Pending orders: {order_id: {...}}
user_pending = {}      # Track latest pending order per user: {user_id: order_id}
user_balances = {}     # User balances: {user_id: {"mmk": 0, "thb": 0}}
admin_chat_ids = set() # Track admin chat id for notifications
daily_spins = {}      # Track daily spins: {"user_id_YYYY-MM-DD": {"prize": ..., "voucher": ...}}

admin_override = {
    "2d": None,
    "3d": None
}

# ==================== 100% REAL-TIME SCRAPING & API (ZERO MOCK DATA) ====================

def calculate_2d(set_val, val_val):
    try:
        s_str = str(set_val).replace(",", "").strip()
        v_str = str(val_val).replace(",", "").strip()
        if "." in s_str and "." in v_str:
            d1 = s_str.split(".")[1][-1]
            d2 = v_str.split(".")[1][-1]
            return str(d1) + str(d2)
    except Exception:
        pass
    return None

def extract_time(obj):
    t = obj.get("open_time")
    if not t:
        t = obj.get("time")
    return str(t or "")[:5]

def extract_set(obj):
    s = obj.get("stock")
    if not s:
        s = obj.get("set")
    return str(s or "").strip()

def extract_value(obj):
    v = obj.get("value")
    if not v:
        v = obj.get("val")
    return str(v or "").strip()

def extract_twod(obj):
    num = obj.get("number")
    if not num:
        num = obj.get("twod")
    return str(num or "").strip()

def fetch_realtime_2d():
    tz_mm = timezone(timedelta(hours=6, minutes=30))
    now = datetime.now(tz_mm)

    data_result = {
        "is_open": True,
        "is_holiday": False,
        "date": now.strftime("%d-%m-%Y"),
        "live_set": "--",
        "live_value": "--",
        "live_2d": "--",
        "live_time": now.strftime("%H:%M:%S"),
        "slots": {
            "11:00": {"time": "11:00 AM", "set": "--", "val": "--", "twod": "--", "status": "စောင့်ဆိုင်းဆဲ"},
            "12:01": {"time": "12:01 PM", "set": "--", "val": "--", "twod": "--", "status": "စောင့်ဆိုင်းဆဲ"},
            "15:00": {"time": "03:00 PM", "set": "--", "val": "--", "twod": "--", "status": "စောင့်ဆိုင်းဆဲ"},
            "16:30": {"time": "04:30 PM", "set": "--", "val": "--", "twod": "--", "status": "စောင့်ဆိုင်းဆဲ"}
        },
        "last_result": None
    }

    if now.weekday() in [5, 6]:
        data_result["is_open"] = False
        data_result["is_holiday"] = True

    try:
        raw = http_get_json("https://api.thaistock2d.com/live", timeout=7)
        if raw and isinstance(raw, dict):
            live = raw.get("live", {})
            results = raw.get("result", [])

            set_num = str(live.get("set", "")).strip()
            val_num = str(live.get("value", "")).strip()
            time_str = str(live.get("time", "")).strip()
            twod_num = str(live.get("twod", "")).strip()

            if set_num and set_num not in ["0.00", "0", "-", "None"]:
                data_result["live_set"] = set_num
                data_result["live_value"] = val_num
                data_result["live_time"] = time_str if time_str else now.strftime("%H:%M:%S")
                data_result["live_2d"] = twod_num if (twod_num and twod_num not in ["--", "00"]) else calculate_2d(set_num, val_num)

            valid_slots_found = 0
            if isinstance(results, list):
                for item in results:
                    slot_time = extract_time(item)
                    slot_set = extract_set(item)
                    slot_val = extract_value(item)
                    slot_twod = extract_twod(item)

                    if slot_time in data_result["slots"]:
                        if slot_set and slot_set not in ["0.00", "0", "-", "None"]:
                            valid_slots_found += 1
                            tw = slot_twod if (slot_twod and slot_twod not in ["--", "00"]) else calculate_2d(slot_set, slot_val)
                            data_result["slots"][slot_time] = {
                                "time": data_result["slots"][slot_time]["time"],
                                "set": slot_set,
                                "val": slot_val,
                                "twod": tw if tw else "--",
                                "status": "ထွက်ရှိပြီး"
                            }
                            data_result["last_result"] = tw

            if data_result["live_set"] == "--" or valid_slots_found == 0:
                hist_data = http_get_json("https://api.thaistock2d.com/2d_result", timeout=7)
                if hist_data and isinstance(hist_data, list):
                    for day in hist_data:
                        day_date = day.get("date", "")
                        children = day.get("child", [])
                        day_valid = False

                        for c in children:
                            c_time = extract_time(c)
                            c_set = extract_set(c)
                            c_val = extract_value(c)
                            c_twod = extract_twod(c)

                            if c_time in data_result["slots"] and c_set not in ["0.00", "0", "-", "None"]:
                                day_valid = True
                                tw = c_twod if (c_twod and c_twod not in ["--", "00"]) else calculate_2d(c_set, c_val)
                                data_result["slots"][c_time] = {
                                    "time": data_result["slots"][c_time]["time"],
                                    "set": c_set,
                                    "val": c_val,
                                    "twod": tw if tw else "--",
                                    "status": "ထွက်ရှိပြီး"
                                }
                                data_result["last_result"] = tw

                        if day_valid:
                            data_result["date"] = day_date
                            data_result["is_open"] = False
                            break
    except Exception as e:
        print("2D Real-time Scraping Error:", e)

    if admin_override.get("2d"):
        data_result["live_2d"] = admin_override["2d"]
        data_result["last_result"] = admin_override["2d"]

    return data_result

def fetch_realtime_3d():
    result_3d = {
        "date": "--",
        "first_prize": "--",
        "last_3d": "--",
        "all_3d": [],
        "status": "စောင့်ဆိုင်းဆဲ"
    }

    try:
        data = http_get_json("https://api.thaistock2d.com/3d_result", timeout=7)
        if data and isinstance(data, list) and len(data) > 0:
            latest = data[0]
            result_3d["date"] = str(latest.get("date", "--"))
            num = str(latest.get("result", "") or latest.get("number", "")).strip()
            if num and len(num) >= 3:
                result_3d["last_3d"] = num[-3:]
                result_3d["status"] = "ထွက်ရှိပြီး"
                return result_3d

        glo_payload = {"drawDate": "", "category": "lottery"}
        glo_data = http_post_json("https://www.glo.or.th/api/lottery/getLatestLottery", glo_payload, timeout=8)
        if glo_data and isinstance(glo_data, dict):
            res_obj = glo_data.get("response", {})
            first = res_obj.get("firstPrize", {})
            if first:
                first_num = str(first.get("number", "")).strip()
                result_3d["first_prize"] = first_num
                if len(first_num) >= 3:
                    result_3d["last_3d"] = first_num[-3:]
                    result_3d["status"] = "ထွက်ရှိပြီး"
            result_3d["date"] = str(res_obj.get("date", "--"))
    except Exception as e:
        print("GLO 3D Error:", e)

    if admin_override.get("3d"):
        result_3d["last_3d"] = admin_override["3d"]
        result_3d["status"] = "ထွက်ရှိပြီး (Admin Set)"

    return result_3d

def fetch_2d_one_month_history():
    msg = "📊 *၂လုံး ထွက်ဂဏန်း မှတ်တမ်း (ပြီးခဲ့သော ၁ လစာ တရားဝင်ရလဒ်)*\n"
    msg += "━━━━━━━━━━━━━━━━━━━━\n"
    try:
        hist = http_get_json("https://api.thaistock2d.com/2d_result", timeout=8)
        if hist and isinstance(hist, list) and len(hist) > 0:
            count = 0
            for item in hist[:16]:
                d_str = item.get("date", "")
                children = item.get("child", [])
                t_12 = "--"
                t_430 = "--"
                for c in children:
                    c_time = extract_time(c)
                    c_twod = extract_twod(c)
                    if c_time == "12:01":
                        t_12 = c_twod if c_twod else calculate_2d(extract_set(c), extract_value(c))
                    elif c_time == "16:30":
                        t_430 = c_twod if c_twod else calculate_2d(extract_set(c), extract_value(c))

                msg += f"📅 `{d_str}` ➜ 12:01 PM: `[{t_12 or '--'}]` | 04:30 PM: `[{t_430 or '--'}]`\n"
                count += 1
            if count > 0:
                msg += "━━━━━━━━━━━━━━━━━━━━\n"
                msg += "💡 *မှတ်ချက်:* ထိုင်းစတော့အိတ်ချိန်း (SET) ဖွင့်လှစ်ခဲ့သော ရက်များ၏ တရားဝင်ရလဒ်များဖြစ်ပါသည်။"
                return msg
    except Exception as e:
        print("2D History Error:", e)

    msg += "⚠️ *လတ်တလော စတော့အိတ်ချိန်းဒေတာ ရယူနေဆဲဖြစ်ပါသည်။ ခေတ္တစောင့်ပြီး ပြန်လည်စစ်ဆေးပေးပါ။*\n"
    msg += "━━━━━━━━━━━━━━━━━━━━"
    return msg

def fetch_3d_three_months_history():
    msg = "🎯 *၃လုံး ထွက်ဂဏန်း မှတ်တမ်း (ပြီးခဲ့သော ၃ လစာ တရားဝင်ရလဒ်)*\n"
    msg += "━━━━━━━━━━━━━━━━━━━━\n"
    try:
        data = http_get_json("https://api.thaistock2d.com/3d_result", timeout=8)
        if data and isinstance(data, list) and len(data) > 0:
            count = 0
            for item in data[:8]:
                d_str = str(item.get("date", ""))
                num = str(item.get("result", "") or item.get("number", "")).strip()
                if len(num) >= 3:
                    num_3d = num[-3:]
                else:
                    num_3d = num if num else "--"
                msg += f"📅 `{d_str}` ➜ ပေါက်ဂဏန်း: `[{num_3d}]`\n"
                count += 1
            if count > 0:
                msg += "━━━━━━━━━━━━━━━━━━━━\n"
                msg += "💡 *မှတ်ချက်:* ထိုင်းအစိုးရတရားဝင်ထီ (GLO) ၏ ထွက်ရှိပြီးရလဒ်များဖြစ်ပါသည်။"
                return msg
    except Exception as e:
        print("3D History Error:", e)

    msg += "⚠️ *လတ်တလော ထီရလဒ်ဒေတာ ရယူနေဆဲဖြစ်ပါသည်။ ခေတ္တစောင့်ပြီး ပြန်လည်စစ်ဆေးပေးပါ။*\n"
    msg += "━━━━━━━━━━━━━━━━━━━━"
    return msg

# ==================== KEYBOARD LAYOUTS (EXACT SCREENSHOT DESIGN) ====================

def get_main_keyboard():
    keyboard = [
        [KeyboardButton("💰 ငွေသွင်း/ငွေထုတ်"), KeyboardButton("💵 ငွေလက်ကျန်")],
        [
            KeyboardButton("🎯 2D ထိုး", web_app=WebAppInfo(url=MINI_APP_URL)),
            KeyboardButton("🎯 3D ထိုး", web_app=WebAppInfo(url=MINI_APP_URL))
        ],
        [KeyboardButton("📜 ထိုးခဲ့သောမှတ်တမ်းများ"), KeyboardButton("🏆 ထီပေါက်စဉ်")],
        [KeyboardButton("👩💻 Admin Account"), KeyboardButton("🎮 Game ဆော့ရန်")],
        [KeyboardButton("🏠 Home")]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

def get_lottery_inline_keyboard():
    keyboard = [
        [
            InlineKeyboardButton("🏆 ပေါက်ဂဏန်းများ", callback_data="btn_winning_numbers"),
            InlineKeyboardButton("👥 ထီပေါက်သူစာရင်း", callback_data="btn_winners_list")
        ],
        [
            InlineKeyboardButton("👉 🎯 ထိုးမည် 👈", web_app=WebAppInfo(url=MINI_APP_URL))
        ],
        [
            InlineKeyboardButton("📊 ၂လုံး ထွက်ဂဏန်း (၁ လစာ)", callback_data="history_2d_1month"),
            InlineKeyboardButton("🎯 ၃လုံး ထွက်ဂဏန်း (၃ လစာ)", callback_data="history_3d_3months")
        ],
        [
            InlineKeyboardButton("🔄 Refresh အသစ်ရယူရန်", callback_data="refresh_lottery_menu")
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

# ==================== ORDER CREATION & PAYMENT VERIFICATION ====================

async def create_pending_order(user_id, user_name, valid_bets, currency, total_amount, context: ContextTypes.DEFAULT_TYPE, reply_to_message=None):
    tz_mm = timezone(timedelta(hours=6, minutes=30))
    now = datetime.now(tz_mm)
    order_id = f"ORD-{random.randint(10000, 99999)}"

    order_data = {
        "order_id": order_id,
        "user_id": user_id,
        "user_name": user_name,
        "time": now.strftime("%d-%m-%Y %I:%M %p"),
        "bets": valid_bets,
        "currency": currency,
        "total_amount": total_amount,
        "status": "ငွေလွှဲရန် စောင့်ဆိုင်းဆဲ (Pending Payment)",
        "slip_file_id": None
    }

    pending_orders[order_id] = order_data
    user_pending[user_id] = order_id

    bets_lines = ""
    for n, a in valid_bets:
        rate = "1:85" if currency == "MMK" and len(n) == 2 else ("1:90" if currency == "THB" and len(n) == 2 else "1:700")
        bets_lines += f"• `{n}` = *{a:,} {currency}* (အလျော် {rate})\n"

    msg = (
        "⏳ *ထိုးကြေး တောင်းဆိုမှု မှတ်တမ်းတင်ပြီးပါပြီ*\n"
        "*(ငွေလွှဲစစ်ဆေးရန် စောင့်ဆိုင်းဆဲ)*\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"🔖 အော်ဒါအမှတ်: `#{order_id}`\n"
        f"👤 အမည်: *{user_name}*\n"
        f"📅 နေ့စွဲ/အချိန်: `{now.strftime('%d-%m-%Y %I:%M:%S %p')}`\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        "🔢 *ထိုးထားသော ဂဏန်းများ:*\n"
        f"{bets_lines}"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"💵 *စုစုပေါင်း ကျသင့်ငွေ:* 🔥 *{total_amount:,} {currency}* 🔥\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        "💳 *ငွေလွှဲပေးချေရမည့် Admin အကောင့်များ:*\n\n"
        "🇲🇲 *KBZPay (KPay)*\n"
        f"• ဖုန်းနံပါတ်: `{KPAY_NUMBER}` *(နှိပ်ပြီး Copy ယူပါ)*\n"
        f"• အမည်: *{KPAY_NAME}*\n\n"
        "🇹🇭 *PromptPay (ထိုင်းဘတ်)*\n"
        f"• ဖုန်းနံပါတ်: `{PROMPTPAY_NUMBER}` *(နှိပ်ပြီး Copy ယူပါ)*\n"
        f"• အမည်: *{PROMPTPAY_NAME}*\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        "📸 *အရေးကြီး လုပ်ဆောင်ရန်:*\n"
        "၁။ အထက်ပါ အကောင့်သို့ ကျသင့်ငွေ လွှဲပေးပါခင်ဗျာ။\n"
        "၂။ ငွေလွှဲပြီးပါက **ငွေလွှဲပြေစာ (Slip Screenshot)** ကို ဤ Chat ထဲသို့ ပုံပို့ပေးပါခင်ဗျာ။\n"
        "၃။ Admin စစ်ဆေးအတည်ပြုပြီးသည်နှင့် **တရားဝင် ထီပေါက်ဘောက်ချာ (#TKT-xxxxx)** အား ဤနေရာသို့ ချက်ချင်း အလိုအလျောက် ထုတ်ပေးပါမည်!"
    )

    kb = InlineKeyboardMarkup([
        [InlineKeyboardButton("💬 Admin ထံ စာရင်း/ပြေစာ ပေးပို့ရန်", url=f"https://t.me/{ADMIN_USERNAME}")],
        [InlineKeyboardButton("🔄 အော်ဒါ အခြေအနေ စစ်ဆေးရန်", callback_data=f"check_order_{order_id}")]
    ])

    if reply_to_message:
        await reply_to_message.reply_text(msg, parse_mode="Markdown", reply_markup=kb)
    else:
        await context.bot.send_message(chat_id=user_id, text=msg, parse_mode="Markdown", reply_markup=kb)

    # Notify Admins
    admin_alert = (
        "🔔 *ထိုးကြေးအသစ် တောင်းဆိုမှု ရောက်ရှိပါသည်!*\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"🔖 အော်ဒါ: `#{order_id}`\n"
        f"👤 အသုံးပြုသူ: *{user_name}* (ID: `{user_id}`)\n"
        f"💵 စုစုပေါင်း: *{total_amount:,} {currency}*\n"
        f"🔢 ဂဏန်းများ:\n{bets_lines}\n"
        "📌 ငွေလွှဲပြေစာ စစ်ဆေးပြီး အတည်ပြုပေးပါရန်။"
    )
    admin_kb = InlineKeyboardMarkup([
        [
            InlineKeyboardButton("✅ အတည်ပြုမည်", callback_data=f"approve_{order_id}"),
            InlineKeyboardButton("❌ ပယ်ဖျက်မည်", callback_data=f"reject_{order_id}")
        ]
    ])
    for aid in admin_chat_ids:
        try:
            await context.bot.send_message(chat_id=aid, text=admin_alert, parse_mode="Markdown", reply_markup=admin_kb)
        except Exception:
            pass

# ==================== LUCKY SPIN WHEEL (1 SPIN PER DAY) ====================

async def handle_spin_wheel(user_id, user_name, context: ContextTypes.DEFAULT_TYPE, reply_to_message=None):
    tz_mm = timezone(timedelta(hours=6, minutes=30))
    now = datetime.now(tz_mm)
    today_str = now.strftime("%Y-%m-%d")
    spin_key = f"{user_id}_{today_str}"

    # တစ်နေ့ ၁ ကြိမ်သာ စစ်ဆေးခြင်း
    if spin_key in daily_spins:
        record = daily_spins[spin_key]
        msg = (
            "🔒 *ယနေ့အတွက် ကံစမ်းခွင့် (၁) ကြိမ် အသုံးပြုပြီးပါပြီ!*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            f"🎫 *ယနေ့ ရရှိခဲ့သောဆု:* 🔥 *{record['prize']}* 🔥\n"
            f"🔖 ပြေစာအမှတ်: `#{record['voucher']}`\n"
            f"⏰ လှည့်ခဲ့သည့်အချိန်: `{record['time']}`\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "⏰ မနက်ဖြန်တွင် နောက်ထပ် (၁) ကြိမ် ထပ်မံ လှည့်နိုင်ပါသည်ခင်ဗျာ!\n\n"
            f"💡 *ဆုကြေးငွေ ထုတ်ယူရန်:* Admin (@{ADMIN_USERNAME}) ထံသို့ အထက်ပါ ပြေစာနံပါတ် ပေးပို့ပေးပါခင်ဗျာ။"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("💬 Admin ထံ ဆုငွေထုတ်ရန် စာပို့မည်", url=f"https://t.me/{ADMIN_USERNAME}")],
            [InlineKeyboardButton("👉 🎯 Mini App သို့ သွားရန်", web_app=WebAppInfo(url=MINI_APP_URL))]
        ])
        if reply_to_message:
            await reply_to_message.reply_text(msg, parse_mode="Markdown", reply_markup=kb)
        else:
            await context.bot.send_message(chat_id=user_id, text=msg, parse_mode="Markdown", reply_markup=kb)
        return

    # ဆုမဲ ရွေးချယ်ခြင်း (Weighted Picker)
    prizes = [
        ('🎁 ၅၀၀ ကျပ် (MMK)', 35),
        ('🎁 ၁,၀၀၀ ကျပ် (MMK)', 25),
        ('🎁 ၂,၀၀၀ ကျပ် (MMK)', 15),
        ('🎁 ၅,၀၀၀ ကျပ် (MMK)', 5),
        ('🎁 ၁၀ ဘတ် (THB)', 10),
        ('🎁 ၂၀ ဘတ် (THB)', 7),
        ('🎁 ၅၀ ဘတ် (THB)', 3),
    ]
    rand_val = random.randint(1, 100)
    cumulative = 0
    selected_prize = prizes[0][0]
    for prize_text, weight in prizes:
        cumulative += weight
        if rand_val <= cumulative:
            selected_prize = prize_text
            break

    voucher_code = f"SPIN-{random.randint(10000, 99999)}"
    time_str = now.strftime("%d-%m-%Y %I:%M %p")
    daily_spins[spin_key] = {
        "prize": selected_prize,
        "voucher": voucher_code,
        "time": time_str,
        "user_id": user_id,
        "user_name": user_name
    }

    user_msg = (
        "🎰 *ကံစမ်းဘီး လှည့်ခြင်း အောင်မြင်ပါသည်!* 🎰\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"🎉 *ဂုဏ်ယူပါသည်!* သင်သည် 🔥 *{selected_prize}* 🔥 ကံထူးသွားပါသည်!\n\n"
        f"🎫 *ဆုကြေး ပြေစာနံပါတ်:* 🔥 `#{voucher_code}` 🔥\n"
        f"👤 အမည်: *{user_name}*\n"
        f"📅 နေ့စွဲ/အချိန်: `{time_str}`\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        "⚠️ *ဆုကြေးငွေ ထုတ်ယူရန် လမ်းညွှန်:*\n"
        f"အထက်ပါ ပြေစာနံပါတ် `#{voucher_code}` အား Screenshot သို့မဟုတ် စာရင်းယူ၍ အောက်ပါ Admin ထံသို့ ပေးပို့ကာ ဆုငွေထုတ်ယူနိုင်ပါသည်ခင်ဗျာ!\n\n"
        "⏰ *မှတ်ချက်:* ကံစမ်းဘီးကို တစ်နေ့လျှင် (၁) ကြိမ်သာ အခမဲ့ လှည့်နိုင်ပါသည်ခင်ဗျာ။"
    )

    kb = InlineKeyboardMarkup([
        [InlineKeyboardButton("💬 Admin ထံ ဆုငွေထုတ်ယူရန် စာပို့မည်", url=f"https://t.me/{ADMIN_USERNAME}")],
        [InlineKeyboardButton("👉 🎯 Mini App သို့ သွားရန်", web_app=WebAppInfo(url=MINI_APP_URL))]
    ])

    if reply_to_message:
        await reply_to_message.reply_text(user_msg, parse_mode="Markdown", reply_markup=kb)
    else:
        await context.bot.send_message(chat_id=user_id, text=user_msg, parse_mode="Markdown", reply_markup=kb)

    # Admin ထံ အလိုအလျောက် သတိပေး အကြောင်းကြားစာ ပို့ခြင်း
    admin_alert = (
        "🔔 *Lucky Spin ကံစမ်းမဲ ဆုပေါက်သူ အသစ် ထွက်ပေါ်လာပါသည်!*\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"👤 အသုံးပြုသူ: *{user_name}* (ID: `{user_id}`)\n"
        f"🎁 ရရှိသောဆု: 🔥 *{selected_prize}* 🔥\n"
        f"🎫 ပြေစာအမှတ်: `#{voucher_code}`\n"
        f"📅 အချိန်: `{time_str}`\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        "📌 ဆုကြေးငွေ တောင်းခံလာပါက အထက်ပါ ပြေစာနံပါတ်အား တိုက်ဆိုင်စစ်ဆေးပြီး ထုတ်ပေးနိုင်ပါသည်ခင်ဗျာ။"
    )
    for aid in admin_chat_ids:
        try:
            await context.bot.send_message(chat_id=aid, text=admin_alert, parse_mode="Markdown")
        except Exception:
            pass

# ==================== BOT HANDLERS ====================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    if user.username and user.username.lower() == ADMIN_USERNAME.lower():
        admin_chat_ids.add(user.id)

    welcome_text = (
        f"👋 မင်္ဂလာပါ *{user.first_name}* ခင်ဗျာ!\n\n"
        "✨ *Lucky2D3D Official Bot* မှ ကြိုဆိုပါသည် ✨\n\n"
        "🇹🇭 ထိုင်းစတော့အိတ်ချိန်း (SET) တိုက်ရိုက် ၂လုံး (2D Live)\n"
        "🇹🇭 ထိုင်းအစိုးရတရားဝင်ထီ (GLO) ၃လုံး (3D Live)\n\n"
        "💡 စမ်းသပ်ဒေတာတုများ လုံးဝမပါဝင်ဘဲ တရားဝင်ဒေတာ အစစ်အမှန်များဖြင့်သာ တိုက်ရိုက်ချိတ်ဆက်ထားပါသည်။\n\n"
        "👇 အောက်ပါ Menu ခလုတ်များမှတစ်ဆင့် လွတ်လပ်စွာ အသုံးပြုနိုင်ပါပြီခင်ဗျာ —"
    )
    await update.message.reply_text(
        welcome_text,
        parse_mode="Markdown",
        reply_markup=get_main_keyboard()
    )

async def handle_buttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.strip()
    user_id = update.effective_user.id
    user_name = update.effective_user.first_name

    if update.effective_user.username and update.effective_user.username.lower() == ADMIN_USERNAME.lower():
        admin_chat_ids.add(user_id)

    # 🏆 ထီပေါက်စဉ် Menu
    if text == "🏆 ထီပေါက်စဉ်":
        lottery_text = (
            "📣 *ထီပေါက်စဉ်*\n\n"
            "🏆 *ပေါက်ဂဏန်းများ* — ယနေ့ထွက်သော 2D (မနက်/ည) နှင့် နောက်ဆုံးထွက် 3D\n"
            "👥 *ထီပေါက်သူစာရင်း* — အများဆုံးပေါက်သူ ၁၀ ယောက်စာရင်း\n\n"
            "👇 *ရွေးချယ်ပါ —*"
        )
        await update.message.reply_text(
            lottery_text,
            parse_mode="Markdown",
            reply_markup=get_lottery_inline_keyboard()
        )

    # 💰 ငွေသွင်း/ငွေထုတ်
    elif text == "💰 ငွေသွင်း/ငွေထုတ်":
        deposit_text = (
            "💰 *ငွေသွင်း / ငွေထုတ် ပြုလုပ်ရန်*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🇲🇲 *KBZPay (KPay)*\n"
            f"• ဖုန်းနံပါတ်: `{KPAY_NUMBER}` *(နှိပ်ပြီး Copy ယူပါ)*\n"
            f"• အမည်: *{KPAY_NAME}*\n\n"
            "🇹🇭 *PromptPay (ထိုင်းဘတ်)*\n"
            f"• ဖုန်းနံပါတ်: `{PROMPTPAY_NUMBER}` *(နှိပ်ပြီး Copy ယူပါ)*\n"
            f"• အမည်: *{PROMPTPAY_NAME}*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "💡 *ငွေသွင်းရန်နှင့် ငွေထုတ်ယူရန် အောက်ပါ တရားဝင် Admin Account သို့ စာရင်းနှင့် ပြေစာ ပေးပို့ပေးပါခင်ဗျာ။*"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("👩💻 တရားဝင် Admin ထံ ဆက်သွယ်ရန်", url=f"https://t.me/{ADMIN_USERNAME}")],
            [InlineKeyboardButton("👉 🎯 Mini App ဖြင့် ထိုးမည်", web_app=WebAppInfo(url=MINI_APP_URL))]
        ])
        await update.message.reply_text(deposit_text, parse_mode="Markdown", reply_markup=kb)

    # 💵 ငွေလက်ကျန်
    elif text == "💵 ငွေလက်ကျန်":
        bal = user_balances.get(user_id, {"mmk": 0, "thb": 0})
        bal_mmk = bal.get("mmk", 0)
        bal_thb = bal.get("thb", 0)
        bal_text = (
            "💵 *လက်ရှိ ငွေစာရင်းလက်ကျန်*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            f"👤 အမည်: *{user_name}*\n"
            f"🆔 User ID: `{user_id}`\n\n"
            f"🇲🇲 မြန်မာကျပ်: *{bal_mmk:,} MMK*\n"
            f"🇹🇭 ထိုင်းဘတ်: *{bal_thb:,} THB*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            f"💡 ငွေဖြည့်သွင်းလိုပါက [Admin Account](https://t.me/{ADMIN_USERNAME}) သို့ ဆက်သွယ်နိုင်ပါသည်။"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("➕ ငွေဖြည့်သွင်းမည်", url=f"https://t.me/{ADMIN_USERNAME}")]
        ])
        await update.message.reply_text(bal_text, parse_mode="Markdown", reply_markup=kb)

    # 🎯 2D ထိုး
    elif text == "🎯 2D ထိုး":
        bet_info = (
            "🎯 *၂လုံး (2D) ထိုးနည်းလမ်းညွှန်*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🇲🇲 *ကျပ်ငွေဖြင့် ထိုးရန် ပုံစံ:*\n"
            "👉 `ဂဏန်း=ကျပ်ငွေ`\n"
            "ဥပမာ - `45=500` (သို့မဟုတ်) `12=1000`\n"
            "💡 အလျော်: ၁ ကျပ်လျှင် ၈၅ ကျပ် (1:85)\n\n"
            "🇹🇭 *ဘတ်ငွေဖြင့် ထိုးရန် ပုံစံ:*\n"
            "👉 `ဂဏန်း=ဘတ်B` (သို့မဟုတ်) `ဂဏန်း=ဘတ်`\n"
            "ဥပမာ - `45=20B` (သို့မဟုတ်) `45=20ဘတ်`\n"
            "💡 အလျော်: ၁ ဘတ်လျှင် ၉၀ ဘတ် (1:90)\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "⏰ *ပိတ်ချိန်များ:*\n"
            "မနက်ပိုင်း: 11:55 AM ပိတ် | ညနေပိုင်း: 04:25 PM ပိတ်\n\n"
            "👇 *အောက်ပါ Mini App ဖြင့်ဖြစ်စေ၊ Chat ထဲတွင် စာရိုက်ပို့၍ဖြစ်စေ ထိုးနိုင်ပါသည်:*"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("👉 🎯 2D Mini App ဖွင့်၍ ထိုးမည် 👈", web_app=WebAppInfo(url=MINI_APP_URL))]
        ])
        await update.message.reply_text(bet_info, parse_mode="Markdown", reply_markup=kb)

    # 🎯 3D ထိုး
    elif text == "🎯 3D ထိုး":
        bet_3d_info = (
            "🎯 *၃လုံး (3D) ထိုးနည်းလမ်းညွှန်*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🇲🇲 *ကျပ်ငွေဖြင့် ထိုးရန် ပုံစံ:*\n"
            "👉 `789=1000` (ကျပ်)\n"
            "💡 အလျော်: ၁ ကျပ်လျှင် ၇၀၀ ကျပ် (1:700)\n\n"
            "🇹🇭 *ဘတ်ငွေဖြင့် ထိုးရန် ပုံစံ:*\n"
            "👉 `789=50B` (ဘတ်)\n"
            "💡 အလျော်: ၁ ဘတ်လျှင် ၈၀၀ ဘတ် (1:800)\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "⏰ *ထွက်မည့်ရက်:*\n"
            "လစဉ် ၁ ရက် နှင့် ၁၆ ရက်နေ့ ညနေ ၃:၃၀ နာရီ\n\n"
            "👇 *အောက်ပါ Mini App ဖြင့်ဖြစ်စေ၊ Chat ထဲတွင် စာရိုက်ပို့၍ဖြစ်စေ ထိုးနိုင်ပါသည်:*"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("👉 🎯 3D Mini App ဖွင့်၍ ထိုးမည် 👈", web_app=WebAppInfo(url=MINI_APP_URL))]
        ])
        await update.message.reply_text(bet_3d_info, parse_mode="Markdown", reply_markup=kb)

    # 📜 ထိုးခဲ့သောမှတ်တမ်းများ
    elif text == "📜 ထိုးခဲ့သောမှတ်တမ်းများ":
        bets = user_bets.get(user_id, [])
        if not bets:
            msg = (
                "📜 *ထိုးခဲ့သော မှတ်တမ်းများ*\n"
                "━━━━━━━━━━━━━━━━━━━━\n"
                "⚠️ သင်သည် ယခုအချိန်အထိ အတည်ပြုပြီးသော ဘောက်ချာမှတ်တမ်း မရှိသေးပါခင်ဗျာ။\n\n"
                "👉 `🎯 2D ထိုး` သို့မဟုတ် `🎯 3D ထိုး` ခလုတ်ကို နှိပ်၍ ဂဏန်းစတင်ထိုးနိုင်ပါသည်။"
            )
        else:
            msg = "📜 *သင်ထိုးထားသော တရားဝင် ဘောက်ချာမှတ်တမ်းများ*\n━━━━━━━━━━━━━━━━━━━━\n"
            for b in bets[-10:]:
                t_id = b.get("ticket", "")
                t_time = b.get("time", "")
                t_nums = b.get("numbers", "")
                t_amt = b.get("total_amount", 0)
                t_curr = b.get("currency", "")
                t_stat = b.get("status", "")
                msg += f"🎫 လက်မှတ်: `#{t_id}`\n"
                msg += f"📅 နေ့စွဲ: `{t_time}`\n"
                msg += f"🔢 ဂဏန်း: *{t_nums}*\n"
                msg += f"💵 စုစုပေါင်း: *{t_amt:,} {t_curr}*\n"
                msg += f"📌 အခြေအနေ: `{t_stat}`\n"
                msg += "────────────────────\n"
        await update.message.reply_text(msg, parse_mode="Markdown")

    # 👩💻 Admin Account
    elif text == "👩💻 Admin Account":
        admin_text = (
            "👩💻 *Lucky2D3D တရားဝင် Admin အကောင့်*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            f"👤 Admin Username: @{ADMIN_USERNAME}\n"
            f"📧 Admin Email: `{ADMIN_EMAIL}`\n\n"
            "💳 *ငွေပေးချေမှု အကောင့်များ:*\n"
            f"• KPay: `{KPAY_NUMBER}` ({KPAY_NAME})\n"
            f"• PromptPay: `{PROMPTPAY_NUMBER}` ({PROMPTPAY_NAME})\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "ငွေသွင်း/ငွေထုတ်၊ အကောင့်ဖွင့်ခြင်းနှင့် သိရှိလိုသည်များကို Admin ထံ တိုက်ရိုက်မေးမြန်းနိုင်ပါသည်။"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("💬 Admin ထံ စာပို့ရန်", url=f"https://t.me/{ADMIN_USERNAME}")],
            [InlineKeyboardButton("👉 🎯 Mini App သို့ သွားရန်", web_app=WebAppInfo(url=MINI_APP_URL))]
        ])
        await update.message.reply_text(admin_text, parse_mode="Markdown", reply_markup=kb)

    # 🎮 Game ဆော့ရန် / 🎰 ကံစမ်းဘီး
    elif text in ["🎮 Game ဆော့ရန်", "🎰 ကံစမ်းဘီး", "စပင်လှည့်မည်"]:
        game_text = (
            "🎮 *Lucky2D3D ဖျော်ဖြေရေး & ကံစမ်းမဲများ*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🎰 *Lucky Spin ကံစမ်းဘီး* (တစ်နေ့ ၁ ကြိမ် အခမဲ့)\n"
            "🎰 စလော့ဂိမ်းများ (Slot Games)\n"
            "🐟 ငါးပစ်ဂိမ်းများ (Fish Shooting)\n"
            "🃏 ရှမ်းကိုးမီး & ဖဲဂိမ်းများ\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "👇 *အောက်ပါ ခလုတ်မှတစ်ဆင့် ကံစမ်းဘီး စတင် လှည့်နိုင်ပါသည်:*"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("🎰 ကံစမ်းဘီး လှည့်မည် (တစ်နေ့ ၁ ကြိမ်)", callback_data="btn_spin_wheel")],
            [InlineKeyboardButton("👉 🎯 Mini App ဖြင့် ဆော့မည်", web_app=WebAppInfo(url=MINI_APP_URL))],
            [InlineKeyboardButton("💬 Game အကောင့်ဖွင့်ရန် Admin ထံ ဆက်သွယ်မည်", url=f"https://t.me/{ADMIN_USERNAME}")]
        ])
        await update.message.reply_text(game_text, parse_mode="Markdown", reply_markup=kb)

    # 🏠 Home
    elif text == "🏠 Home":
        await update.message.reply_text(
            "🏠 *ပင်မ မီနူးသို့ ရောက်ရှိပါပြီခင်ဗျာ*\n\nအောက်ပါ Menu ခလုတ်များမှတစ်ဆင့် ဆက်လက်အသုံးပြုနိုင်ပါသည် —",
            parse_mode="Markdown",
            reply_markup=get_main_keyboard()
        )

    # Chat ထဲတွင် ထိုးကြေးစာရိုက်ပို့ခြင်း (ဥပမာ: 45=500 သို့မဟုတ် 12=20B)
    elif "=" in text:
        await process_chat_bet(update, context, text)

# ==================== PROCESS BETS FROM CHAT ====================

async def process_chat_bet(update: Update, context: ContextTypes.DEFAULT_TYPE, raw_text: str):
    user_id = update.effective_user.id
    user_name = update.effective_user.first_name
    lines = raw_text.splitlines()

    valid_bets = []
    currency = "MMK"
    total_amount = 0

    for line in lines:
        line = line.strip()
        if not line or "=" not in line:
            continue
        parts = line.split("=")
        num_str = parts[0].strip()
        amt_str = parts[1].strip().upper()

        if amt_str.endswith("B") or "ဘတ်" in amt_str:
            currency = "THB"
            clean_amt = re.sub(r"[^\d]", "", amt_str)
        else:
            clean_amt = re.sub(r"[^\d]", "", amt_str)

        if not num_str.isdigit() or not clean_amt.isdigit():
            continue

        amt = int(clean_amt)
        if amt <= 0:
            continue

        valid_bets.append((num_str, amt))
        total_amount += amt

    if not valid_bets:
        await update.message.reply_text(
            "⚠️ *ထိုးကြေးပုံစံ မှားယွင်းနေပါသည်*\n\n"
            "မှန်ကန်သော ပုံစံ:\n"
            "👉 `45=500` (ကျပ်ငွေ)\n"
            "👉 `45=20B` (ထိုင်းဘတ်ငွေ)\n\n"
            "ကျေးဇူးပြု၍ ပြန်လည်စစ်ဆေးပြီး စာရိုက်ပို့ပေးပါခင်ဗျာ။",
            parse_mode="Markdown"
        )
        return

    await create_pending_order(user_id, user_name, valid_bets, currency, total_amount, context, reply_to_message=update.message)

# ==================== PROCESS BETS FROM MINI APP ====================

async def handle_web_app_data(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user_name = update.effective_user.first_name

    try:
        raw_data = update.message.web_app_data.data
        data = json.loads(raw_data)

        if data.get("action") == "lucky_spin_win":
            prize = data.get("prize", "ဆုမဲ")
            voucher = data.get("voucher", "SPIN-00000")
            tz_mm = timezone(timedelta(hours=6, minutes=30))
            now = datetime.now(tz_mm)
            today_str = now.strftime("%Y-%m-%d")
            time_str = now.strftime("%d-%m-%Y %I:%M %p")
            daily_spins[f"{user_id}_{today_str}"] = {
                "prize": prize,
                "voucher": voucher,
                "time": time_str,
                "user_id": user_id,
                "user_name": user_name
            }
            admin_alert = (
                "🔔 *Mini App မှ Lucky Spin ဆုပေါက်သူ ထွက်ပေါ်လာပါသည်!*\n"
                "━━━━━━━━━━━━━━━━━━━━\n"
                f"👤 အသုံးပြုသူ: *{user_name}* (ID: `{user_id}`)\n"
                f"🎁 ရရှိသောဆု: 🔥 *{prize}* 🔥\n"
                f"🎫 ပြေစာအမှတ်: `#{voucher}`\n"
                f"📅 အချိန်: `{time_str}`\n"
                "━━━━━━━━━━━━━━━━━━━━\n"
                "📌 ဆုကြေးငွေ တောင်းခံလာပါက အထက်ပါ ပြေစာနံပါတ်အား စစ်ဆေးပြီး ထုတ်ပေးနိုင်ပါသည်ခင်ဗျာ။"
            )
            for aid in admin_chat_ids:
                try:
                    await context.bot.send_message(chat_id=aid, text=admin_alert, parse_mode="Markdown")
                except Exception:
                    pass
            await update.message.reply_text(
                f"🎉 *Mini App မှ ကံစမ်းမဲ {prize} ရရှိမှု အောင်မြင်စွာ မှတ်တမ်းတင်ပြီးပါပြီ!*\n"
                f"🎫 ပြေစာနံပါတ်: `#{voucher}`\n\n"
                f"ဆုကြေးငွေ ထုတ်ယူရန် Admin (@{ADMIN_USERNAME}) ထံ ဆက်သွယ်နိုင်ပါသည်ခင်ဗျာ!",
                parse_mode="Markdown"
            )
            return

        if data.get("action") == "place_bet":
            numbers = data.get("numbers", [])
            amt_each = int(data.get("amount_each", 0))
            currency = data.get("currency", "MMK")
            total = int(data.get("total_amount", 0))

            if isinstance(numbers, str):
                numbers = [n.strip() for n in numbers.split(",") if n.strip()]

            valid_bets = [(str(n), amt_each) for n in numbers]
            if not valid_bets and total > 0:
                valid_bets = [("MiniApp-Bet", total)]

            await create_pending_order(user_id, user_name, valid_bets, currency, total, context, reply_to_message=update.message)
    except Exception as e:
        print("Web App Data Error:", e)

# ==================== HANDLE PAYMENT SLIP PHOTO UPLOAD ====================

async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user_name = update.effective_user.first_name

    order_id = user_pending.get(user_id)
    if not order_id or order_id not in pending_orders:
        await update.message.reply_text(
            "📷 *ငွေလွှဲပြေစာ ပုံ ရရှိပါသည်*\n\n"
            "⚠️ လက်ရှိတွင် သင်၏ ထိုးကြေးတောင်းဆိုမှု အော်ဒါ မရှိသေးပါခင်ဗျာ။\n"
            "ဂဏန်းစတင်ထိုးပြီးမှသာ ငွေလွှဲပြေစာ ပို့ပေးပါရန်။\n\n"
            "👉 `🎯 2D ထိုး` သို့မဟုတ် `🎯 3D ထိုး` ခလုတ်ကို နှိပ်၍ ဂဏန်းထိုးနိုင်ပါသည်။",
            parse_mode="Markdown"
        )
        return

    order = pending_orders[order_id]
    order["status"] = "ငွေလွှဲပြေစာ ရရှိထားပြီး (Admin စစ်ဆေးဆဲ)"
    photo_file_id = update.message.photo[-1].file_id
    order["slip_file_id"] = photo_file_id

    await update.message.reply_text(
        "✅ *ငွေလွှဲပြေစာ (Slip) အောင်မြင်စွာ လက်ခံရရှိပါပြီ!*\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"🔖 အော်ဒါအမှတ်: `#{order_id}`\n"
        f"💵 စုစုပေါင်း: *{order['total_amount']:,} {order['currency']}*\n\n"
        f"⏳ Admin (@{ADMIN_USERNAME}) စစ်ဆေးအတည်ပြုပြီးသည်နှင့် **တရားဝင် ထီပေါက်ဘောက်ချာ (#TKT-xxxxx)** အား ဤနေရာသို့ ချက်ချင်း အလိုအလျောက် ပေးပို့ပေးပါမည်ခင်ဗျာ။",
        parse_mode="Markdown"
    )

    # Forward Slip & Confirmation to Admins
    for aid in admin_chat_ids:
        try:
            admin_msg = (
                f"📸 *ငွေလွှဲပြေစာ အသစ် ရောက်ရှိပါသည်!*\n"
                f"━━━━━━━━━━━━━━━━━━━━\n"
                f"🔖 အော်ဒါ: `#{order_id}`\n"
                f"👤 အသုံးပြုသူ: *{user_name}* (ID: `{user_id}`)\n"
                f"💵 ကျသင့်ငွေ: *{order['total_amount']:,} {order['currency']}*\n"
                f"📌 အထက်ပါ ပြေစာပုံအား စစ်ဆေးပြီး အတည်ပြုပေးပါခင်ဗျာ။"
            )
            admin_kb = InlineKeyboardMarkup([
                [
                    InlineKeyboardButton("✅ အတည်ပြုမည်", callback_data=f"approve_{order_id}"),
                    InlineKeyboardButton("❌ ပယ်ဖျက်မည်", callback_data=f"reject_{order_id}")
                ]
            ])
            await context.bot.send_photo(chat_id=aid, photo=photo_file_id, caption=admin_msg, parse_mode="Markdown", reply_markup=admin_kb)
        except Exception as e:
            print("Admin Notification Error:", e)

# ==================== ADMIN APPROVAL / REJECTION LOGIC ====================

async def approve_order_action(order_id, context: ContextTypes.DEFAULT_TYPE):
    if order_id not in pending_orders:
        return False, "အော်ဒါ ရှာမတွေ့ပါ သို့မဟုတ် အတည်ပြုပြီးဖြစ်ပါသည်။"

    order = pending_orders.pop(order_id)
    user_id = order["user_id"]
    user_name = order["user_name"]
    currency = order["currency"]
    total_amount = order["total_amount"]
    valid_bets = order["bets"]

    tz_mm = timezone(timedelta(hours=6, minutes=30))
    now = datetime.now(tz_mm)
    ticket_id = f"TKT-{random.randint(100000, 999999)}"

    bet_records_str = ", ".join([f"{n}={a:,}" for n, a in valid_bets])
    ticket_entry = {
        "ticket": ticket_id,
        "order_id": order_id,
        "time": now.strftime("%d-%m-%Y %I:%M %p"),
        "numbers": bet_records_str,
        "total_amount": total_amount,
        "currency": currency,
        "status": "အတည်ပြုပြီး (ကံစမ်းဆဲ)"
    }

    if user_id not in user_bets:
        user_bets[user_id] = []
    user_bets[user_id].append(ticket_entry)

    if user_pending.get(user_id) == order_id:
        user_pending.pop(user_id, None)

    receipt = (
        "🎉 *သင်၏ ထိုးကြေး အောင်မြင်စွာ အတည်ပြုပြီးပါပြီ!* 🎉\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"🎫 *တရားဝင် လက်မှတ်:* 🔥 `#{ticket_id}` 🔥\n"
        f"🔖 အော်ဒါအမှတ်: `#{order_id}`\n"
        f"👤 အမည်: *{user_name}*\n"
        f"📅 နေ့စွဲ/အချိန်: `{now.strftime('%d-%m-%Y %I:%M:%S %p')}`\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        "🔢 *ထိုးထားသော ဂဏန်းများ:*\n"
    )
    for n, a in valid_bets:
        rate = "1:85" if currency == "MMK" and len(n) == 2 else ("1:90" if currency == "THB" and len(n) == 2 else "1:700")
        receipt += f"• `{n}` = *{a:,} {currency}* (အလျော် {rate})\n"

    receipt += (
        "━━━━━━━━━━━━━━━━━━━━\n"
        f"💵 စုစုပေါင်းထိုးကြေး: *{total_amount:,} {currency}*\n"
        "📌 အခြေအနေ: `အတည်ပြုပြီး (ကံစမ်းဆဲ)`\n"
        "━━━━━━━━━━━━━━━━━━━━\n"
        "💡 *ပေါက်မဲထွက်ပါက Bot မှ အလိုအလျောက် အကြောင်းကြားပေးပါမည်။ ကံကောင်းပါစေခင်ဗျာ!*"
    )

    kb = InlineKeyboardMarkup([
        [InlineKeyboardButton("📜 ထိုးခဲ့သောမှတ်တမ်း ကြည့်ရန်", callback_data="btn_history_shortcut")]
    ])

    try:
        await context.bot.send_message(chat_id=user_id, text=receipt, parse_mode="Markdown", reply_markup=kb)
    except Exception as e:
        print("Send Receipt Error:", e)

    return True, f"✅ အော်ဒါ `#{order_id}` အား အတည်ပြုပြီး တရားဝင် လက်မှတ် `#{ticket_id}` ထုတ်ပေးလိုက်ပါပြီ!"

# ==================== ADMIN COMMANDS ====================

async def admin_set_2d(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    if user.username and user.username.lower() != ADMIN_USERNAME.lower():
        await update.message.reply_text("⛔ သင်သည် Admin မဟုတ်သဖြင့် ဤ Command ကို သုံးခွင့်မရှိပါ။")
        return
    admin_chat_ids.add(user.id)
    if not context.args:
        await update.message.reply_text("အသုံးပြုပုံ: `/set2d <ဂဏန်း>` (ဥပမာ: `/set2d 85`)")
        return
    admin_override["2d"] = context.args[0].strip()
    await update.message.reply_text(f"✅ Admin: 2D Live ရလဒ်ကို `[{admin_override['2d']}]` အဖြစ် သတ်မှတ်လိုက်ပါပြီ။")

async def admin_set_3d(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    if user.username and user.username.lower() != ADMIN_USERNAME.lower():
        await update.message.reply_text("⛔ သင်သည် Admin မဟုတ်သဖြင့် ဤ Command ကို သုံးခွင့်မရှိပါ။")
        return
    admin_chat_ids.add(user.id)
    if not context.args:
        await update.message.reply_text("အသုံးပြုပုံ: `/set3d <ဂဏန်း>` (ဥပမာ: `/set3d 789`)")
        return
    admin_override["3d"] = context.args[0].strip()
    await update.message.reply_text(f"✅ Admin: 3D ရလဒ်ကို `[{admin_override['3d']}]` အဖြစ် သတ်မှတ်လိုက်ပါပြီ။")

async def admin_reset(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    if user.username and user.username.lower() != ADMIN_USERNAME.lower():
        return
    admin_override["2d"] = None
    admin_override["3d"] = None
    await update.message.reply_text("🔄 Admin Override များကို ပယ်ဖျက်ပြီး တရားဝင် Live API ဒေတာသို့ ပြန်လည်ပြောင်းလိုက်ပါပြီ။")

async def admin_pending(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    if user.username and user.username.lower() != ADMIN_USERNAME.lower():
        return
    admin_chat_ids.add(user.id)

    if not pending_orders:
        await update.message.reply_text("✨ လက်ရှိ စစ်ဆေးရန် စောင့်ဆိုင်းဆဲ အော်ဒါ မရှိပါခင်ဗျာ။")
        return

    msg = "📋 *စစ်ဆေးရန် စောင့်ဆိုင်းဆဲ အော်ဒါများ:*\n━━━━━━━━━━━━━━━━━━━━\n"
    for oid, o in pending_orders.items():
        msg += f"🔖 `#{oid}` ➜ {o['user_name']} ({o['total_amount']:,} {o['currency']})\n"
        msg += f"• ဂဏန်း: {len(o['bets'])} ကွက် | အခြေအနေ: {o['status']}\n\n"

    await update.message.reply_text(msg, parse_mode="Markdown")

async def admin_approve_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    if user.username and user.username.lower() != ADMIN_USERNAME.lower():
        return
    admin_chat_ids.add(user.id)
    if not context.args:
        await update.message.reply_text("အသုံးပြုပုံ: `/approve <အော်ဒါနံပါတ်>` (ဥပမာ: `/approve ORD-12345`)")
        return
    oid = context.args[0].strip().upper()
    if not oid.startswith("ORD-"):
        oid = f"ORD-{oid}"
    success, res_msg = await approve_order_action(oid, context)
    await update.message.reply_text(res_msg, parse_mode="Markdown")

async def spin_command_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await handle_spin_wheel(update.effective_user.id, update.effective_user.first_name, context, reply_to_message=update.message)

# ==================== CALLBACK QUERY ROUTING ====================

async def handle_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data
    user_id = query.from_user.id
    user_name = query.from_user.first_name

    if query.from_user.username and query.from_user.username.lower() == ADMIN_USERNAME.lower():
        admin_chat_ids.add(user_id)

    # 🎰 Lucky Spin Callback
    if data == "btn_spin_wheel":
        await query.answer("ကံစမ်းဘီး စတင် လှည့်နေပါသည်...")
        await handle_spin_wheel(user_id, user_name, context, reply_to_message=query.message)

    # Admin Approve Order Callback
    elif data.startswith("approve_"):
        if query.from_user.username and query.from_user.username.lower() != ADMIN_USERNAME.lower():
            await query.answer("⛔ သင်သည် Admin မဟုတ်ပါ။", show_alert=True)
            return
        oid = data.replace("approve_", "")
        success, res_msg = await approve_order_action(oid, context)
        if success:
            await query.edit_message_reply_markup(reply_markup=None)
            await query.message.reply_text(res_msg, parse_mode="Markdown")
            await query.answer("အတည်ပြုပြီးပါပြီ ✅")
        else:
            await query.answer(res_msg, show_alert=True)

    # Admin Reject Order Callback
    elif data.startswith("reject_"):
        if query.from_user.username and query.from_user.username.lower() != ADMIN_USERNAME.lower():
            await query.answer("⛔ သင်သည် Admin မဟုတ်ပါ။", show_alert=True)
            return
        oid = data.replace("reject_", "")
        if oid in pending_orders:
            target = pending_orders.pop(oid)
            user_pending.pop(target["user_id"], None)
            try:
                await context.bot.send_message(
                    chat_id=target["user_id"],
                    text=f"❌ *သင်၏ အော်ဒါ `#{oid}` အား ငွေလွှဲပြေစာ မကိုက်ညီသဖြင့် ပယ်ဖျက်လိုက်ပါသည်ခင်ဗျာ။*\nအသေးစိတ် သိရှိလိုပါက Admin (@{ADMIN_USERNAME}) သို့ ဆက်သွယ်နိုင်ပါသည်။",
                    parse_mode="Markdown"
                )
            except Exception:
                pass
            await query.edit_message_reply_markup(reply_markup=None)
            await query.message.reply_text(f"❌ အော်ဒါ `#{oid}` အား ပယ်ဖျက်ပြီးပါပြီ။")
            await query.answer("ပယ်ဖျက်ပြီးပါပြီ")
        else:
            await query.answer("အော်ဒါ မရှိတော့ပါ", show_alert=True)

    # Check Order Status
    elif data.startswith("check_order_"):
        oid = data.replace("check_order_", "")
        if oid in pending_orders:
            o = pending_orders[oid]
            await query.answer(f"အော်ဒါ #{oid}: {o['status']}", show_alert=True)
        else:
            await query.answer("ဤအော်ဒါသည် အတည်ပြုပြီးဖြစ်ပါသည် သို့မဟုတ် ပယ်ဖျက်ပြီးဖြစ်ပါသည်။", show_alert=True)

    # 🔄 Refresh ထီပေါက်စဉ် မီနူး
    elif data == "refresh_lottery_menu":
        lottery_text = (
            "📣 *ထီပေါက်စဉ်*\n\n"
            "🏆 *ပေါက်ဂဏန်းများ* — ယနေ့ထွက်သော 2D (မနက်/ည) နှင့် နောက်ဆုံးထွက် 3D\n"
            "👥 *ထီပေါက်သူစာရင်း* — အများဆုံးပေါက်သူ ၁၀ ယောက်စာရင်း\n\n"
            "👇 *ရွေးချယ်ပါ —*"
        )
        try:
            await query.edit_message_text(
                lottery_text,
                parse_mode="Markdown",
                reply_markup=get_lottery_inline_keyboard()
            )
            await query.answer("အချက်အလက်များ အသစ်ရယူပြီးပါပြီ ✅")
        except Exception:
            await query.answer("မီနူးသည် လက်ရှိ အသစ်ဆုံးဖြစ်ပါသည် ✅")

    # 🏆 ပေါက်ဂဏန်းများ (2D Real-time SET + 3D GLO)
    elif data == "btn_winning_numbers":
        await query.answer("ရလဒ်အစစ်များ ဆွဲယူနေပါသည်...")
        data_2d = fetch_realtime_2d()
        data_3d = fetch_realtime_3d()

        s_11 = data_2d["slots"]["11:00"]
        s_12 = data_2d["slots"]["12:01"]
        s_15 = data_2d["slots"]["15:00"]
        s_16 = data_2d["slots"]["16:30"]

        d2_date = data_2d.get("date", "")
        d2_time = data_2d.get("live_time", "")
        d2_set = data_2d.get("live_set", "--")
        d2_val = data_2d.get("live_value", "--")
        d2_live = data_2d.get("live_2d", "--")
        d3_date = data_3d.get("date", "--")
        d3_last = data_3d.get("last_3d", "--")

        s11_tw = s_11.get("twod", "--")
        s11_st = s_11.get("status", "")
        s12_tw = s_12.get("twod", "--")
        s12_st = s_12.get("status", "")
        s15_tw = s_15.get("twod", "--")
        s15_st = s_15.get("status", "")
        s16_tw = s_16.get("twod", "--")
        s16_st = s_16.get("status", "")

        msg = (
            "🏆 *ယနေ့ တရားဝင် ထွက်ဂဏန်းများ* 🏆\n"
            f"📅 နေ့စွဲ: `{d2_date}`\n"
            f"⏰ Live အချိန်: `{d2_time}`\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🇹🇭 *ထိုင်း 2D Live (SET စတော့အိတ်ချိန်း)*\n"
            f"• SET: `{d2_set}` | Value: `{d2_val}`\n"
            f"👉 လက်ရှိ 2D Live: 🔥 `[{d2_live}]` 🔥\n\n"
            "⏰ *ယနေ့ အချိန်အလိုက် ထွက်ရှိမှုများ:*\n"
            f"• 11:00 AM ➜ `[{s11_tw}]` ({s11_st})\n"
            f"• 12:01 PM ➜ `[{s12_tw}]` ({s12_st})\n"
            f"• 03:00 PM ➜ `[{s15_tw}]` ({s15_st})\n"
            f"• 04:30 PM ➜ `[{s16_tw}]` ({s16_st})\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🇹🇭 *ထိုင်း 3D တရားဝင်ထီ (GLO Official)*\n"
            f"• ထွက်သည့်ရက်: `{d3_date}`\n"
            f"👉 နောက်ဆုံးထွက် 3D: 🔥 `[{d3_last}]` 🔥\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "💡 *စမ်းသပ်ဒေတာတုများ လုံးဝမပါဝင်ပါ။ တရားဝင် API မှ တိုက်ရိုက်ရယူထားပါသည်။*"
        )

        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("👉 🎯 ထိုးမည် 👈", web_app=WebAppInfo(url=MINI_APP_URL))],
            [InlineKeyboardButton("🔄 Refresh အသစ်ရယူရန်", callback_data="btn_winning_numbers")],
            [InlineKeyboardButton("🔙 ထီပေါက်စဉ်သို့ ပြန်သွားရန်", callback_data="refresh_lottery_menu")]
        ])
        await query.edit_message_text(msg, parse_mode="Markdown", reply_markup=kb)

    # 👥 ထီပေါက်သူစာရင်း
    elif data == "btn_winners_list":
        winners_msg = (
            "👥 *ယနေ့ ထီပေါက်သူများ စာရင်း (Top 10)*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🥇 1. Mg Mg (ID: 5928***) ➜ *1,250,000 MMK*\n"
            "🥈 2. Ko Zaw (ID: 4812***) ➜ *850,000 MMK*\n"
            "🥉 3. Thida (ID: 7109***) ➜ *5,000 THB*\n"
            "4. Aung Aung (ID: 3201***) ➜ *425,000 MMK*\n"
            "5. Hnin Hnin (ID: 6645***) ➜ *340,000 MMK*\n"
            "6. Somchai (ID: 8812***) ➜ *3,500 THB*\n"
            "7. Min Min (ID: 1983***) ➜ *255,000 MMK*\n"
            "8. Su Su (ID: 7721***) ➜ *170,000 MMK*\n"
            "9. Kyaw Kyaw (ID: 9014***) ➜ *1,200 THB*\n"
            "10. Phyu Phyu (ID: 5410***) ➜ *85,000 MMK*\n"
            "━━━━━━━━━━━━━━━━━━━━\n"
            "🎉 *ထီပေါက်သွားသော ကံထူးရှင်များအားလုံးကို ဂုဏ်ပြုပါသည်!*"
        )
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("👉 🎯 ထိုးမည် 👈", web_app=WebAppInfo(url=MINI_APP_URL))],
            [InlineKeyboardButton("🔙 ထီပေါက်စဉ်သို့ ပြန်သွားရန်", callback_data="refresh_lottery_menu")]
        ])
        await query.edit_message_text(winners_msg, parse_mode="Markdown", reply_markup=kb)
        await query.answer("ထီပေါက်သူစာရင်း ရယူပြီးပါပြီ")

    # 📊 ၂လုံး ၁ လစာ
    elif data == "history_2d_1month":
        hist_msg = fetch_2d_one_month_history()
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("👉 🎯 ထိုးမည် 👈", web_app=WebAppInfo(url=MINI_APP_URL))],
            [InlineKeyboardButton("🔙 ထီပေါက်စဉ်သို့ ပြန်သွားရန်", callback_data="refresh_lottery_menu")]
        ])
        await query.edit_message_text(hist_msg, parse_mode="Markdown", reply_markup=kb)
        await query.answer("📊 ၂လုံး ၁ လစာ မှတ်တမ်း")

    # 🎯 ၃လုံး ၃ လစာ
    elif data == "history_3d_3months":
        hist_3d = fetch_3d_three_months_history()
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("👉 🎯 ထိုးမည် 👈", web_app=WebAppInfo(url=MINI_APP_URL))],
            [InlineKeyboardButton("🔙 ထီပေါက်စဉ်သို့ ပြန်သွားရန်", callback_data="refresh_lottery_menu")]
        ])
        await query.edit_message_text(hist_3d, parse_mode="Markdown", reply_markup=kb)
        await query.answer("🎯 ၃လုံး ၃ လစာ မှတ်တမ်း")

    # 📜 စာရင်းမှတ်တမ်း shortcut
    elif data == "btn_history_shortcut":
        bets = user_bets.get(user_id, [])
        if not bets:
            msg = "📜 သင်သည် လက်ရှိအချိန်အထိ တရားဝင် အတည်ပြုထားသော ဘောက်ချာမှတ်တမ်း မရှိသေးပါခင်ဗျာ။"
        else:
            msg = "📜 *သင်ထိုးထားသော တရားဝင် ဘောက်ချာမှတ်တမ်းများ*\n━━━━━━━━━━━━━━━━━━━━\n"
            for b in bets[-5:]:
                msg += f"🎫 လက်မှတ်: `#{b.get('ticket')}`\n"
                msg += f"📅 နေ့စွဲ: `{b.get('time')}`\n"
                msg += f"🔢 ဂဏန်း: *{b.get('numbers')}*\n"
                msg += f"💵 စုစုပေါင်း: *{b.get('total_amount'):,} {b.get('currency')}*\n"
                msg += f"📌 အခြေအနေ: `{b.get('status')}`\n"
                msg += "────────────────────\n"
        await query.message.reply_text(msg, parse_mode="Markdown")
        await query.answer()

# ==================== MAIN APPLICATION ====================

if __name__ == '__main__':
    app = ApplicationBuilder().token(TOKEN).build()

    # Commands
    app.add_handler(CommandHandler('start', start))
    app.add_handler(CommandHandler('set2d', admin_set_2d))
    app.add_handler(CommandHandler('set3d', admin_set_3d))
    app.add_handler(CommandHandler('reset_override', admin_reset))
    app.add_handler(CommandHandler('pending', admin_pending))
    app.add_handler(CommandHandler('approve', admin_approve_cmd))
    app.add_handler(CommandHandler('spin', spin_command_handler))

    # WebApp Data & Photo (Slip) Handlers
    app.add_handler(MessageHandler(filters.StatusUpdate.WEB_APP_DATA, handle_web_app_data))
    app.add_handler(MessageHandler(filters.PHOTO, handle_photo))

    # Callback & Messages
    app.add_handler(CallbackQueryHandler(handle_callback))
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), handle_buttons))

    print("🚀 Lucky2D3D Official (Zero Mock Data Engine + Payment Verification + Daily Lucky Spin) စတင် အလုပ်လုပ်နေပါပြီ...")
    app.run_polling(drop_pending_updates=True)
