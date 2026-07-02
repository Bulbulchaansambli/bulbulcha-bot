# BULBULCHA qabul boti

Shermat Yormatov nomidagi “BULBULCHA” bolalar xor va raqs ansambli uchun Telegram qabul boti.

## Funksiyalar
- Onlayn ariza qabul qilish
- Tug‘ilgan sana bo‘yicha yoshni avtomatik hisoblash
- 4–7, 7–10, 10–14 yosh guruhlarini avtomatik aniqlash
- Xor va raqs jadvali
- Foto/video qabul qilish
- Admin ID ga yangi ariza yuborish
- Galereya
- `/stats` statistikasi
- `/export` CSV/Excel uchun arizalarni olish

## Render sozlamalari
Environment variables:

- `BOT_TOKEN` — BotFather bergan yangi token
- `ADMIN_ID` — sizning Telegram ID raqamingiz

Botdagi `/myid` buyrug‘i orqali Telegram ID ni bilib olasiz.

## Ishga tushirish
```bash
pip install -r requirements.txt
BOT_TOKEN="TOKEN" ADMIN_ID="123456789" python bot.py
```
import os
import sqlite3
from datetime import datetime, date
from pathlib import Path
from typing import Dict, Any

from telegram import (
    Update,
    ReplyKeyboardMarkup,
    KeyboardButton,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
)
import csv

from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ConversationHandler,
    ContextTypes,
    filters,
)

BOT_TOKEN = os.getenv("BOT_TOKEN", "")
ADMIN_ID = int(os.getenv("ADMIN_ID", "0"))
DB_PATH = Path("bulbulcha_applications.db")
ASSETS_DIR = Path("assets")

# Tinglov sanasi admin tomonidan belgilanadi. Bot arizani qabul qiladi, admin keyin ota-onani chaqiradi.
AUDITION_SLOTS = {}

(
    CHILD_NAME,
    BIRTH_DATE,
    GENDER,
    DIRECTION,
    EXPERIENCE,
    EXPERIENCE_DETAIL,
    PARENT_NAME,
    PHONE,
    DISTRICT,
    PHOTO,
    VIDEO,
    AUDITION_DATE,
    AUDITION_TIME,
) = range(13)

MAIN_MENU = ReplyKeyboardMarkup(
    [
        ["🎼 Ansambl haqida", "👶 Yosh guruhlari"],
        ["🎤 Xor", "💃 Raqs"],
        ["📅 Mashg‘ulot jadvali", "🖼 Galereya"],
        ["📝 Qabulga yozilish"],
        ["📍 Manzil", "☎️ Bog‘lanish"],
        ["📢 Telegram kanal", "📸 Instagram"],
        ["❓ Savol-javob"],
    ],
    resize_keyboard=True,
)


def init_db() -> None:
    with sqlite3.connect(DB_PATH) as con:
        con.execute(
            """
            CREATE TABLE IF NOT EXISTS applications (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                created_at TEXT,
                telegram_id INTEGER,
                username TEXT,
                child_name TEXT,
                birth_date TEXT,
                age_text TEXT,
                age_years INTEGER,
                group_name TEXT,
                gender TEXT,
                direction TEXT,
                experience TEXT,
                experience_detail TEXT,
                parent_name TEXT,
                phone TEXT,
                district TEXT,
                photo_file_id TEXT,
                video_file_id TEXT,
                audition_date TEXT,
                audition_time TEXT,
                status TEXT DEFAULT 'Yangi'
            )
            """
        )


def calculate_age(birth: date, today: date | None = None) -> tuple[int, int, str]:
    today = today or date.today()
    years = today.year - birth.year - ((today.month, today.day) < (birth.month, birth.day))
    months = today.month - birth.month
    if today.day < birth.day:
        months -= 1
    if months < 0:
        months += 12
    return years, months, f"{years} yosh {months} oy"


def group_by_age(years: int) -> str:
    if 4 <= years < 7:
        return "4–7 yosh — Kichik/Tayyorlov guruh"
    if 7 <= years < 10:
        return "7–10 yosh — O‘rta guruh"
    if 10 <= years <= 14:
        return "10–14 yosh — Katta guruh"
    return "Qabul yoshiga mos emas"


def parse_birth_date(text: str) -> date | None:
    for fmt in ("%d.%m.%Y", "%d/%m/%Y", "%Y-%m-%d"):
        try:
            return datetime.strptime(text.strip(), fmt).date()
        except ValueError:
            pass
    return None


def taken_slots(audition_date: str) -> set[str]:
    with sqlite3.connect(DB_PATH) as con:
        rows = con.execute(
            "SELECT audition_time FROM applications WHERE audition_date=?", (audition_date,)
        ).fetchall()
    return {r[0] for r in rows}


def free_dates_keyboard() -> InlineKeyboardMarkup:
    buttons = []
    for d, times in AUDITION_SLOTS.items():
        free = [t for t in times if t not in taken_slots(d)]
        if free:
            label = datetime.strptime(d, "%Y-%m-%d").strftime("%d.%m.%Y")
            buttons.append([InlineKeyboardButton(f"📅 {label}", callback_data=f"date:{d}")])
    return InlineKeyboardMarkup(buttons)


def free_times_keyboard(audition_date: str) -> InlineKeyboardMarkup:
    taken = taken_slots(audition_date)
    rows = []
    for t in AUDITION_SLOTS.get(audition_date, []):
        if t not in taken:
            rows.append([InlineKeyboardButton(f"⏰ {t}", callback_data=f"time:{t}")])
    return InlineKeyboardMarkup(rows)


def save_application(data: Dict[str, Any], user) -> int:
    with sqlite3.connect(DB_PATH) as con:
        cur = con.execute(
            """
            INSERT INTO applications (
                created_at, telegram_id, username, child_name, birth_date, age_text,
                age_years, group_name, gender, direction, experience, experience_detail,
                parent_name, phone, district, photo_file_id, video_file_id,
                audition_date, audition_time
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """,
            (
                datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                user.id,
                user.username or "",
                data.get("child_name"),
                data.get("birth_date"),
                data.get("age_text"),
                data.get("age_years"),
                data.get("group_name"),
                data.get("gender"),
                data.get("direction"),
                data.get("experience"),
                data.get("experience_detail", ""),
                data.get("parent_name"),
                data.get("phone"),
                data.get("district"),
                data.get("photo_file_id", ""),
                data.get("video_file_id", ""),
                data.get("audition_date"),
                data.get("audition_time"),
            ),
        )
        return cur.lastrowid


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "🎼 Assalomu alaykum!\n\n"
        "BULBULCHA bolalar ansamblining qabul botiga xush kelibsiz.\n"
        "Kerakli bo‘limni tanlang.",
        reply_markup=MAIN_MENU,
    )


async def myid(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(f"Sizning Telegram ID raqamingiz: {update.effective_user.id}")


async def menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    text = update.message.text
    if text == "🎼 Ansambl haqida":
        await update.message.reply_text(
            "🎼 BULBULCHA — bolalarda musiqa, xor, raqs, sahna madaniyati "
            "va jamoada ishlash ko‘nikmalarini rivojlantiradigan ijodiy ansambl."
        )
    elif text == "👶 Yosh guruhlari":
        await update.message.reply_text(
            "👶 Yosh guruhlari:\n\n"
            "🟢 4–7 yosh — Kichik/Tayyorlov guruh\n"
            "🔵 7–10 yosh — O‘rta guruh\n"
            "🟣 10–14 yosh — Katta guruh\n\n"
            "Bot tug‘ilgan sana bo‘yicha bolaning aniq yoshini avtomatik hisoblaydi."
        )
    elif text == "🎤 Xor":
        await update.message.reply_text(
            "🎤 Xor yo‘nalishi:\n\n"
            "Kichik xor 4–7 yosh:\n"
            "Chorshanba 14:00–17:00\n"
            "Shanba/Yakshanba 11:00–14:00\n\n"
            "O‘rta xor 7–10 yosh:\n"
            "Chorshanba/Shanba 17:00–19:30\n"
            "Yakshanba 10:00–13:00\n\n"
            "Tinglovda yoshiga mos she’r aytish so‘raladi. Vaqtlarda o‘zgarish bo‘lsa, oldindan xabar beriladi."
        )
    elif text == "💃 Raqs":
        await update.message.reply_text(
            "💃 Raqs yo‘nalishi:\n\n"
            "Tayyorlov raqs 4–7 yosh:\n"
            "Seshanba/Payshanba/Shanba 16:30–18:00\n\n"
            "O‘rta raqs 7–10 yosh:\n"
            "Seshanba/Payshanba/Shanba 14:00–16:00\n\n"
            "Katta raqs 10–14 yosh:\n"
            "Seshanba/Payshanba/Shanba 18:00–20:00\n\n"
            "Tinglovda yoshiga mos she’r aytish so‘raladi."
        )
    elif text == "📅 Mashg‘ulot jadvali":
        await update.message.reply_text(
            "📅 Mashg‘ulot jadvali\n\n"
            "🎤 Kichik xor 4–7 yosh: Chorshanba 14:00–17:00, Shanba/Yakshanba 11:00–14:00.\n"
            "🎤 O‘rta xor 7–10 yosh: Chorshanba/Shanba 17:00–19:30, Yakshanba 10:00–13:00.\n\n"
            "💃 Tayyorlov raqs 4–7 yosh: Seshanba/Payshanba/Shanba 16:30–18:00.\n"
            "💃 O‘rta raqs 7–10 yosh: Seshanba/Payshanba/Shanba 14:00–16:00.\n"
            "💃 Katta raqs 10–14 yosh: Seshanba/Payshanba/Shanba 18:00–20:00.\n\n"
            "*Vaqtlarda o‘zgarish bo‘lishi mumkin, bu haqda oldindan xabar beriladi.*"
        )
    elif text == "🖼 Galereya":
        await update.message.reply_text("🖼 BULBULCHA galereyasi. Rasmlar yuborilmoqda...")
        photos = sorted(ASSETS_DIR.glob("*.jpeg"))[:8]
        if not photos:
            await update.message.reply_text("Galereya rasmlari keyinroq qo‘shiladi.")
        else:
            for photo_path in photos:
                try:
                    await update.message.reply_photo(photo=photo_path.open("rb"))
                except Exception:
                    pass
    elif text == "📍 Manzil":
        await update.message.reply_text(
            "📍 BULBULCHA\n"
            "Toshkent shahri, Yunusobod tumani, Amir Temur tor ko‘chasi, 119A uy.\n\n"
            "🗺 https://maps.app.goo.gl/3dNnULeHAbknYGak6"
        )
    elif text == "☎️ Bog‘lanish":
        await update.message.reply_text("☎️ Aloqa uchun: +998 98 123 15 25")
    elif text == "📢 Telegram kanal":
        await update.message.reply_text("📢 Rasmiy Telegram kanal: https://t.me/bulbulcha_uz")
    elif text == "📸 Instagram":
        await update.message.reply_text("📸 Instagram: https://www.instagram.com/bulbulcha_official")
    elif text == "❓ Savol-javob":
        await update.message.reply_text(
            "❓ Ko‘p so‘raladigan savollar:\n\n"
            "1. Qabul yoshi: 4–14 yosh.\n"
            "2. Yo‘nalishlar: Xor va Raqs.\n"
            "3. Qabul yil davomida.\n"
            "4. Qabul bepul.\n"
            "5. Tinglov vaqtini administrator belgilaydi va ota-onani chaqiradi.\n"
            "6. Tinglovga bola yoshiga mos she’r tayyorlab keladi."
        )


async def register_start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data.clear()
    await update.message.reply_text("👦 Farzandingizning F.I.Sh.ni yozing:")
    return CHILD_NAME


async def child_name(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["child_name"] = update.message.text.strip()
    await update.message.reply_text("📅 Tug‘ilgan sanasini yozing. Masalan: 15.08.2020")
    return BIRTH_DATE


async def birth_date(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    birth = parse_birth_date(update.message.text)
    if not birth:
        await update.message.reply_text("Sana noto‘g‘ri. Iltimos, shu formatda yozing: 15.08.2020")
        return BIRTH_DATE
    years, months, age_text = calculate_age(birth)
    group = group_by_age(years)
    if group == "Qabul yoshiga mos emas":
        await update.message.reply_text(
            f"Farzandingiz yoshi: {age_text}.\n\n"
            "Kechirasiz, hozircha 4–14 yosh oralig‘idagi bolalar qabul qilinadi.\n"
            "Qo‘shimcha ma’lumot: +998 98 123 15 25",
            reply_markup=MAIN_MENU,
        )
        return ConversationHandler.END
    context.user_data.update(
        birth_date=birth.strftime("%d.%m.%Y"),
        age_years=years,
        age_text=age_text,
        group_name=group,
    )
    await update.message.reply_text(
        f"✅ Yoshi: {age_text}\n📚 Guruh: {group}\n\nJinsini tanlang:",
        reply_markup=ReplyKeyboardMarkup([["👦 O‘g‘il", "👧 Qiz"]], resize_keyboard=True, one_time_keyboard=True),
    )
    return GENDER


async def gender(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["gender"] = update.message.text
    await update.message.reply_text(
        "Qaysi yo‘nalishga yozmoqchisiz?",
        reply_markup=ReplyKeyboardMarkup([["🎤 Xor", "💃 Raqs", "🎭 Xor va raqs"]], resize_keyboard=True, one_time_keyboard=True),
    )
    return DIRECTION


async def direction(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["direction"] = update.message.text
    await update.message.reply_text(
        "Farzandingiz avval musiqa yoki raqs bilan shug‘ullanganmi?",
        reply_markup=ReplyKeyboardMarkup([["✅ Ha", "❌ Yo‘q"]], resize_keyboard=True, one_time_keyboard=True),
    )
    return EXPERIENCE


async def experience(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["experience"] = update.message.text
    if "Ha" in update.message.text:
        await update.message.reply_text("Qayerda va necha yil shug‘ullanganini qisqacha yozing:")
        return EXPERIENCE_DETAIL
    context.user_data["experience_detail"] = ""
    await update.message.reply_text("👨‍👩‍👦 Ota yoki onaning F.I.Sh.ni yozing:")
    return PARENT_NAME


async def experience_detail(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["experience_detail"] = update.message.text.strip()
    await update.message.reply_text("👨‍👩‍👦 Ota yoki onaning F.I.Sh.ni yozing:")
    return PARENT_NAME


async def parent_name(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["parent_name"] = update.message.text.strip()
    await update.message.reply_text(
        "☎️ Telefon raqamingizni yuboring yoki yozing:",
        reply_markup=ReplyKeyboardMarkup([[KeyboardButton("📱 Kontaktni ulashish", request_contact=True)]], resize_keyboard=True, one_time_keyboard=True),
    )
    return PHONE


async def phone(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if update.message.contact:
        context.user_data["phone"] = update.message.contact.phone_number
    else:
        context.user_data["phone"] = update.message.text.strip()
    await update.message.reply_text("🏠 Yashash tumaningizni yozing:")
    return DISTRICT


async def district(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["district"] = update.message.text.strip()
    await update.message.reply_text("📷 Farzandingiz fotosuratini yuboring. O‘tkazib yuborish uchun /skip yozing.")
    return PHOTO


async def photo(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["photo_file_id"] = update.message.photo[-1].file_id
    await update.message.reply_text("🎥 30–60 soniyalik video yuboring. O‘tkazib yuborish uchun /skip yozing.")
    return VIDEO


async def skip_photo(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["photo_file_id"] = ""
    await update.message.reply_text("🎥 30–60 soniyalik video yuboring. O‘tkazib yuborish uchun /skip yozing.")
    return VIDEO


async def video(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["video_file_id"] = update.message.video.file_id
    return await finish_application(update, context)


async def skip_video(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data["video_file_id"] = ""
    return await finish_application(update, context)


async def finish_application(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    user = update.effective_user
    context.user_data["audition_date"] = "Admin belgilaydi"
    context.user_data["audition_time"] = "Admin belgilaydi"
    app_id = save_application(context.user_data, user)

    summary = (
        f"✅ Arizangiz qabul qilindi!\n\n"
        f"Nomzod № {app_id}\n"
        f"👦 Ismi: {context.user_data['child_name']}\n"
        f"🎂 Yoshi: {context.user_data['age_text']}\n"
        f"📚 Guruh: {context.user_data['group_name']}\n"
        f"🎤 Yo‘nalish: {context.user_data['direction']}\n\n"
        "📅 Tinglov vaqti administrator tomonidan belgilanadi. Tez orada siz bilan bog‘lanamiz.\n"
        "🎙 Tinglovga farzandingiz yoshiga mos she’r tayyorlab kelishi kerak.\n\n"
        "📍 Manzil: Toshkent shahri, Yunusobod tumani, Amir Temur tor ko‘chasi, 119A uy.\n"
        "☎️ +998 98 123 15 25"
    )
    await update.effective_message.reply_text(summary, reply_markup=MAIN_MENU)

    admin_msg = (
        f"🟢 YANGI ARIZA № {app_id}\n\n"
        f"👦 Bola: {context.user_data['child_name']}\n"
        f"📅 Tug‘ilgan sana: {context.user_data['birth_date']}\n"
        f"🎂 Yoshi: {context.user_data['age_text']}\n"
        f"📚 Guruh: {context.user_data['group_name']}\n"
        f"👤 Jinsi: {context.user_data['gender']}\n"
        f"🎤 Yo‘nalish: {context.user_data['direction']}\n"
        f"🎼 Tajriba: {context.user_data['experience']} {context.user_data.get('experience_detail','')}\n"
        f"👨‍👩‍👦 Ota-ona: {context.user_data['parent_name']}\n"
        f"☎️ Telefon: {context.user_data['phone']}\n"
        f"🏠 Tuman: {context.user_data['district']}\n"
        "📅 Tinglov: Admin chaqiradi\n"
        "🎙 Tayyorlanadigan narsa: yoshiga mos she’r\n"
        f"Telegram: @{user.username or 'yo‘q'}"
    )
    if ADMIN_ID:
        await context.bot.send_message(chat_id=ADMIN_ID, text=admin_msg)
        if context.user_data.get("photo_file_id"):
            await context.bot.send_photo(chat_id=ADMIN_ID, photo=context.user_data["photo_file_id"], caption=f"Nomzod № {app_id} foto")
        if context.user_data.get("video_file_id"):
            await context.bot.send_video(chat_id=ADMIN_ID, video=context.user_data["video_file_id"], caption=f"Nomzod № {app_id} video")
    return ConversationHandler.END


async def choose_date(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query
    await query.answer()
    audition_date = query.data.split(":", 1)[1]
    context.user_data["audition_date"] = audition_date
    await query.edit_message_text("⏰ Qulay vaqtni tanlang:", reply_markup=free_times_keyboard(audition_date))
    return AUDITION_TIME


async def choose_time(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query
    await query.answer()
    time_value = query.data.split(":", 1)[1]
    context.user_data["audition_time"] = time_value
    app_id = save_application(context.user_data, query.from_user)

    d = datetime.strptime(context.user_data["audition_date"], "%Y-%m-%d").strftime("%d.%m.%Y")
    summary = (
        f"✅ Arizangiz qabul qilindi!\n\n"
        f"Nomzod № {app_id}\n"
        f"👦 Ismi: {context.user_data['child_name']}\n"
        f"🎂 Yoshi: {context.user_data['age_text']}\n"
        f"📚 Guruh: {context.user_data['group_name']}\n"
        f"🎤 Yo‘nalish: {context.user_data['direction']}\n"
        f"📅 Tinglov: {d}, {time_value}\n\n"
        "📍 Manzil: Toshkent shahri, Yunusobod tumani, Amir Temur tor ko‘chasi, 119A uy.\n"
        "Iltimos, belgilangan vaqtdan 15 daqiqa oldin keling."
    )
    await query.edit_message_text(summary)

    admin_msg = (
        f"🟢 YANGI ARIZA № {app_id}\n\n"
        f"👦 Bola: {context.user_data['child_name']}\n"
        f"📅 Tug‘ilgan sana: {context.user_data['birth_date']}\n"
        f"🎂 Yoshi: {context.user_data['age_text']}\n"
        f"📚 Guruh: {context.user_data['group_name']}\n"
        f"👤 Jinsi: {context.user_data['gender']}\n"
        f"🎤 Yo‘nalish: {context.user_data['direction']}\n"
        f"🎼 Tajriba: {context.user_data['experience']} {context.user_data.get('experience_detail','')}\n"
        f"👨‍👩‍👦 Ota-ona: {context.user_data['parent_name']}\n"
        f"☎️ Telefon: {context.user_data['phone']}\n"
        f"🏠 Tuman: {context.user_data['district']}\n"
        f"📅 Tinglov: {d}, {time_value}\n"
        f"Telegram: @{query.from_user.username or 'yo‘q'}"
    )
    if ADMIN_ID:
        await context.bot.send_message(chat_id=ADMIN_ID, text=admin_msg)
    await context.bot.send_message(chat_id=query.from_user.id, text="Bosh menyu:", reply_markup=MAIN_MENU)
    return ConversationHandler.END


async def stats(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if ADMIN_ID and update.effective_user.id != ADMIN_ID:
        return
    with sqlite3.connect(DB_PATH) as con:
        total = con.execute("SELECT COUNT(*) FROM applications").fetchone()[0]
        by_direction = con.execute("SELECT direction, COUNT(*) FROM applications GROUP BY direction").fetchall()
        by_group = con.execute("SELECT group_name, COUNT(*) FROM applications GROUP BY group_name").fetchall()
    msg = f"📊 BULBULCHA statistikasi\n\nJami arizalar: {total}\n\n🎤 Yo‘nalishlar:\n"
    msg += "\n".join([f"• {d or 'Belgilanmagan'}: {c}" for d, c in by_direction]) or "Hali ariza yo‘q"
    msg += "\n\n👶 Guruhlar:\n"
    msg += "\n".join([f"• {g or 'Belgilanmagan'}: {c}" for g, c in by_group]) or "Hali ariza yo‘q"
    await update.message.reply_text(msg)


async def export_csv(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if ADMIN_ID and update.effective_user.id != ADMIN_ID:
        return
    csv_path = Path("bulbulcha_applications_export.csv")
    with sqlite3.connect(DB_PATH) as con, csv_path.open("w", newline="", encoding="utf-8-sig") as f:
        cur = con.execute("SELECT * FROM applications ORDER BY id DESC")
        writer = csv.writer(f)
        writer.writerow([d[0] for d in cur.description])
        writer.writerows(cur.fetchall())
    await update.message.reply_document(document=csv_path.open("rb"), filename="bulbulcha_arizalar.csv")


async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text("Ro‘yxatdan o‘tish bekor qilindi.", reply_markup=MAIN_MENU)
    return ConversationHandler.END


def main() -> None:
    if not BOT_TOKEN:
        raise RuntimeError("BOT_TOKEN topilmadi. Uni environment orqali kiriting.")
    init_db()
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    conv = ConversationHandler(
        entry_points=[MessageHandler(filters.Regex("^📝 Qabulga yozilish$"), register_start), CommandHandler("register", register_start)],
        states={
            CHILD_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, child_name)],
            BIRTH_DATE: [MessageHandler(filters.TEXT & ~filters.COMMAND, birth_date)],
            GENDER: [MessageHandler(filters.TEXT & ~filters.COMMAND, gender)],
            DIRECTION: [MessageHandler(filters.TEXT & ~filters.COMMAND, direction)],
            EXPERIENCE: [MessageHandler(filters.TEXT & ~filters.COMMAND, experience)],
            EXPERIENCE_DETAIL: [MessageHandler(filters.TEXT & ~filters.COMMAND, experience_detail)],
            PARENT_NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, parent_name)],
            PHONE: [MessageHandler((filters.CONTACT | filters.TEXT) & ~filters.COMMAND, phone)],
            DISTRICT: [MessageHandler(filters.TEXT & ~filters.COMMAND, district)],
            PHOTO: [MessageHandler(filters.PHOTO, photo), CommandHandler("skip", skip_photo)],
            VIDEO: [MessageHandler(filters.VIDEO, video), CommandHandler("skip", skip_video)],
            AUDITION_DATE: [CallbackQueryHandler(choose_date, pattern="^date:")],
            AUDITION_TIME: [CallbackQueryHandler(choose_time, pattern="^time:")],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("myid", myid))
    app.add_handler(CommandHandler("stats", stats))
    app.add_handler(CommandHandler("export", export_csv))
    app.add_handler(conv)
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, menu_handler))
    app.run_polling()


if __name__ == "__main__":
    main()
python-telegram-bot==22.8
