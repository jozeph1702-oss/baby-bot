# main.py
import os
import json
import re
import random
import datetime
import pytz
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    filters,
    CallbackQueryHandler,
)
from telegram.constants import ParseMode

# === НАСТРОЙКИ ===
TOKEN = os.getenv("TOKEN")
if not TOKEN:
    raise RuntimeError("TOKEN не установлен. Зайди в Railway → Variables → добавь TOKEN.")

PROXY_URL = ""
DATA_FILE = "babies.json"
EVENTS_FILE = "events.json"

# Глобальные списки
BABIES = []
EVENTS = []

# === ЗАГРУЗКА ДАННЫХ ===
def load_babies():
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
                return data if isinstance(data, list) else [{"name": k, "birth": v} for k, v in data.items()]
        except Exception as e:
            print(f"Ошибка загрузки {DATA_FILE}: {e}")
    return []

def load_events():
    if os.path.exists(EVENTS_FILE):
        try:
            with open(EVENTS_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception as e:
            print(f"Ошибка загрузки {EVENTS_FILE}: {e}")
    return []

def save_babies(data):
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print(f"Ошибка сохранения: {e}")

def save_events(data):
    try:
        with open(EVENTS_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print(f"Ошибка сохранения: {e}")

# === ЭКРАНИРОВАНИЕ ===
def escape_md(text: str) -> str:
    if not text:
        return ""
    for ch in r'_*[]()~`>#+-=|{}.!':
        text = text.replace(ch, f'\\{ch}')
    return text

# === НОРМАЛИЗАЦИЯ ИМЕНИ ===
def normalize_name(name: str) -> str:
    return ' '.join(word.capitalize() for word in name.lstrip('+-').strip().split())

# === АНТИМАТ ===
BAD_WORDS = ['бля', 'сука', 'хуй', 'пизд', 'ебан', 'ёбан', 'мудак', 'идиот', 'нах', 'блять']
TIRED_WORDS = ['устала', 'плохо', 'грустно', 'тяжело', 'сложно', 'не могу']

def contains_bad_word(text: str) -> bool:
    if not text:
        return False
    text = text.lower()
    for k, v in {'0': 'о', '@': 'а', '3': 'з', '.': '', ' ': ''}.items():
        text = text.replace(k, v)
    return any(re.search(rf'\b{re.escape(word)}', text) for word in BAD_WORDS)

def contains_tired(text: str) -> bool:
    return any(w in text.lower() for w in TIRED_WORDS)

# === СООБЩЕНИЯ ===
TIPS = [
    "🍼 *Совет дня:* При грудном вскармливании пейте больше воды.",
    "😴 *Совет дня:* Ложитесь спать пораньше — это важно!",
    "🧸 *Совет дня:* Уделяйте себе 15 минут в день.",
    "📚 *Совет дня:* Читайте с малышом каждый день.",
    "🥦 *Совет дня:* Ешьте больше овощей и фруктов.",
    "💕 *Совет дня:* Не сравнивайте себя с другими.",
    "🧠 *Совет дня:* Малыш учится на ваших эмоциях — улыбайтесь!",
    "🧼 *Совет дня:* Мойте руки перед кормлением.",
    "🌟 *Совет дня:* Отмечайте каждую победу!",
    "📞 *Совет дня:* Обращайтесь за помощью, если тяжело."
]

MOTIVATIONAL = [
    "🌸 *Ты — супермама! Держись, всё будет отлично!*",
    "💖 *Помни: ты не одна. Мы рядом!*",
    "🌟 *Ты сильнее, чем думаешь!*"
]

# === МЕНЮ ===
async def show_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("📄 Список детей", callback_data='list_babies')],
        [InlineKeyboardButton("💡 Совет дня", callback_data='tip')],
        [InlineKeyboardButton("📊 Статистика", callback_data='stats')],
        [InlineKeyboardButton("🎂 Ближайшие ДР", callback_data='birthdays')],
        [InlineKeyboardButton("🗓 Календарь", callback_data='calendar')],
        [InlineKeyboardButton("❓ Помощь", callback_data='help')],
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text('Выберите действие:', reply_markup=reply_markup)

# === ОБРАБОТКА КНОПОК ===
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data

    if data == 'list_babies':
        if not BABIES:
            await query.edit_message_text("📝 Список пуст.")
            return
        text = "*👶 Наши малыши:*\n\n"
        now = datetime.datetime.now(pytz.timezone('Asia/Krasnoyarsk')).date()
        for b in sorted(BABIES, key=lambda x: x["name"]):
            name = b["name"]
            birth = datetime.datetime.strptime(b["birth"], "%Y-%m-%d").date()
            m = (now.year - birth.year) * 12 + (now.month - birth.month)
            if now.day < birth.day:
                m -= 1
            age = f"{m} мес." if m < 12 else f"{m // 12} г." + (f" {m % 12} мес." if m % 12 else "")
            text += f"• *{name}* — {birth.strftime('%d.%m.%Y')} ({age})\n"
        await query.edit_message_text(escape_md(text), parse_mode=ParseMode.MARKDOWN_V2)

    elif data == 'tip':
        await query.edit_message_text(escape_md(random.choice(TIPS)), parse_mode=ParseMode.MARKDOWN_V2)

    elif data == 'stats':
        if not BABIES:
            text = "📊 Статистика: список пуст."
        else:
            now = datetime.datetime.now(pytz.timezone('Asia/Krasnoyarsk')).date()
            groups = {"0-3": 0, "4-6": 0, "7-9": 0, "10-12": 0, "1-2 г": 0, "2+ г": 0}
            for b in BABIES:
                try:
                    bd = datetime.datetime.strptime(b["birth"], "%Y-%m-%d").date()
                    m = (now.year - bd.year) * 12 + (now.month - bd.month)
                    if now.day < bd.day:
                        m -= 1
                    if m < 4: groups["0-3"] += 1
                    elif m < 7: groups["4-6"] += 1
                    elif m < 10: groups["7-9"] += 1
                    elif m < 13: groups["10-12"] += 1
                    elif m < 25: groups["1-2 г"] += 1
                    else: groups["2+ г"] += 1
                except: pass
            text = "*📊 Статистика:*\n\n" + "\n".join(f"• {k}: {v}" for k, v in groups.items() if v > 0)
        await query.edit_message_text(escape_md(text), parse_mode=ParseMode.MARKDOWN_V2)

    elif data == 'birthdays':
        now = datetime.datetime.now(pytz.timezone('Asia/Krasnoyarsk')).date()
        upcoming = []
        for b in BABIES:
            try:
                name = b["name"]
                bd = datetime.datetime.strptime(b["birth"], "%Y-%m-%d").date()
                next_bday = bd.replace(year=now.year) if bd.replace(year=now.year) >= now else bd.replace(year=now.year + 1)
                days = (next_bday - now).days
                if 0 <= days <= 30:
                    upcoming.append((name, next_bday, days))
            except: pass
        upcoming.sort(key=lambda x: x[2])
        text = "*🎂 Ближайшие ДР (30 дней):*\n\n"
        text += "\n".join(
            f"• *{n}* — {d.strftime('%d.%m')} ({'сегодня' if da==0 else 'завтра' if da==1 else f'через {da} дн.'})"
            for n, d, da in upcoming
        ) if upcoming else "Никто не празднует."
        await query.edit_message_text(escape_md(text), parse_mode=ParseMode.MARKDOWN_V2)

    elif data == 'calendar':
        now = datetime.datetime.now(pytz.timezone('Asia/Krasnoyarsk')).date()
        upcoming = []
        for e in EVENTS:
            try:
                title = e["title"]
                date_str = e["date"]
                ed = datetime.datetime.strptime(date_str, "%Y-%m-%d").date()
                days = (ed - now).days
                if 0 <= days <= 14:
                    upcoming.append((title, ed, days))
            except: pass
        upcoming.sort(key=lambda x: x[2])
        text = "*🗓 Ближайшие события (14 дней):*\n\n"
        text += "\n".join(
            f"• *{t}* — {d.strftime('%d.%m.%Y')} ({'сегодня' if da==0 else 'завтра' if da==1 else f'через {da} дн.'})"
            for t, d, da in upcoming
        ) if upcoming else "Нет событий."
        await query.edit_message_text(escape_md(text), parse_mode=ParseMode.MARKDOWN_V2)

    elif data == 'help':
        help_text = (
            "🤖 *Помощь:*\n\n"
            "/menu — Меню\n"
            "/tip — Совет дня\n"
            "/ustala — Поддержка\n\n"
            "*Добавить ребёнка:* `+Имя дд.мм.гггг`\n"
            "Пример: `+Маша 15.03.2024`\n\n"
            "*Добавить событие:* `!событие Прививка 20.03.2025`\n\n"
            "*Удалить:* `-Имя` (только админы)"
        )
        await query.edit_message_text(escape_md(help_text), parse_mode=ParseMode.MARKDOWN_V2)

# === ДОБАВЛЕНИЕ/УДАЛЕНИЕ ===
async def check_baby_commands(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.message
    if not msg or not msg.text: return
    text = msg.text.strip()

    if text.startswith('+'):
        match = re.search(r'\b(\d{2})\.(\d{2})\.(\d{4})\b$', text)
        if not match: return
        date_str = match.group(0)
        name = normalize_name(text[1:match.start()].strip())
        if not name: return
        try:
            birth = datetime.datetime.strptime(date_str, "%d.%m.%Y")
            BABIES.append({"name": name, "birth": birth.strftime("%Y-%m-%d")})
            save_babies(BABIES)
            await msg.reply_text(escape_md(f"✅ *{name}* добавлен! Дата: {date_str}"), parse_mode=ParseMode.MARKDOWN_V2)
        except: await msg.reply_text("❌ Ошибка даты.")

    elif text.startswith('-'):
        user_id = msg.from_user.id
        chat_id = msg.chat_id
        try:
            member = await context.bot.get_chat_member(chat_id, user_id)
            is_admin = member.status in ['administrator', 'creator']
        except: is_admin = False
        if not is_admin:
            await msg.reply_text("❌ Только админы могут удалять.")
            return
        name = normalize_name(text[1:].strip())
        found = False
        for i, b in enumerate(BABIES):
            if b["name"] == name:
                del BABIES[i]
                save_babies(BABIES)
                await msg.reply_text(escape_md(f"✅ *{name}* удалён."), parse_mode=ParseMode.MARKDOWN_V2)
                found = True
                break
        if not found:
            await msg.reply_text(escape_md(f"❌ *{name}* не найден."), parse_mode=ParseMode.MARKDOWN_V2)

# === СОБЫТИЯ ===
async def check_event_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.message
    if not msg or not msg.text: return
    text = msg.text.strip()
    if not text.startswith('!событие '): return
    rest = text[10:].strip()
    match = re.search(r'\b(\d{2})\.(\d{2})\.(\d{4})\b$', rest)
    if not match: return
    date_str = match.group(0)
    title = rest[:match.start()].strip()
    if not title: return
    try:
        event_date = datetime.datetime.strptime(date_str, "%d.%m.%Y")
        EVENTS.append({"title": title, "date": event_date.strftime("%Y-%m-%d")})
        save_events(EVENTS)
        await msg.reply_text(escape_md(f"✅ *{title}* на {date_str} добавлено!"), parse_mode=ParseMode.MARKDOWN_V2)
    except:
        await msg.reply_text("❌ Ошибка даты.")

# === ПОДДЕРЖКА И АНТИМАТ ===
async def check_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.message
    if not msg or not msg.from_user or msg.from_user.is_bot: return
    text = (msg.text or msg.caption or "").lower()
    if contains_tired(text):
        await msg.reply_text(escape_md(random.choice(MOTIVATIONAL)), parse_mode=ParseMode.MARKDOWN_V2)
    elif contains_bad_word(text):
        try:
            await msg.delete()
            user = msg.from_user
            mention = f"@{user.username}" if user.username else user.first_name
            await context.bot.send_message(msg.chat_id, f"{mention}, давайте уважительно 🙏")
        except: pass

# === ПРИВЕТСТВИЕ ===
async def greet_new_member(update: Update, context: ContextTypes.DEFAULT_TYPE):
    for member in update.message.new_chat_members:
        if member.is_bot: continue
        try:
            await context.bot.send_message(
                update.effective_chat.id,
                f"Здравствуй, {member.first_name}! 🌸\n\nДобро пожаловать в наш тёплый чат! 💖"
            )
        except: pass

# === ЕЖЕДНЕВНЫЕ ПОЗДРАВЛЕНИЯ ===
async def daily_birthday_check(context: ContextTypes.DEFAULT_TYPE):
    try:
        chat_id = context.application.bot_data.get('main_chat_id')
        if not chat_id: return
        now = datetime.datetime.now(pytz.timezone('Asia/Krasnoyarsk')).date()
        greeted = context.application.bot_data.setdefault("greeted", set())
        new_greeted = set()

        for baby in BABIES:
            try:
                name = baby["name"]
                birth = datetime.datetime.strptime(baby["birth"], "%Y-%m-%d").date()
                months = (now.year - birth.year) * 12 + (now.month - birth.month)
                if now.day < birth.day:
                    months -= 1

                # День рождения
                if now.month == birth.month and now.day == birth.day:
                    years = now.year - birth.year
                    key = f"{name}_bday_{now}"
                    if key not in greeted:
                        await context.bot.send_message(
                            chat_id,
                            escape_md(f"🎉 Ура! Сегодня *{name}* празднует *{years}* лет! 🎂"),
                            parse_mode=ParseMode.MARKDOWN_V2
                        )
                        new_greeted.add(key)

                # Месяцы
                if months >= 1:
                    key = f"{name}_month_{months}_{now}"
                    if key not in greeted:
                        msg = "Первый месяц — огромный шаг! 🥳" if months == 1 else \
                              "Полгода — уже почти ходит! 🚶" if months == 6 else \
                              "Первый День Рождения! 🎉" if months == 12 else \
                              "Каждый месяц — новые победы! 🌟"
                        await context.bot.send_message(
                            chat_id,
                            escape_md(f"👶 Поздравляем *{name}* с *{months}* месяцем жизни! 🎊\n\n{msg}"),
                            parse_mode=ParseMode.MARKDOWN_V2
                        )
                        new_greeted.add(key)

                # Годы
                if months >= 12:
                    years = months // 12
                    if years >= 2:
                        key = f"{name}_year_{years}_{now}"
                        if key not in greeted:
                            await context.bot.send_message(
                                chat_id,
                                escape_md(f"🎈 Удивительно! *{name}* уже *{years}* года(лет)! Пусть каждый год будет счастливым! 💖"),
                                parse_mode=ParseMode.MARKDOWN_V2
                            )
                            new_greeted.add(key)

            except Exception as e:
                print(f"Ошибка: {e}")
        context.application.bot_data["greeted"] = new_greeted
    except Exception as e:
        print(f"Ошибка в daily: {e}")

# === ЗАПУСК ===
async def remember_chat(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.application.bot_data['main_chat_id'] = update.effective_chat.id
    await show_menu(update, context)

def main():
    global BABIES, EVENTS
    BABIES = load_babies()
    EVENTS = load_events()

    app = Application.builder().token(TOKEN).build()
    app.bot_data['main_chat_id'] = None

    app.add_handler(CommandHandler("start", remember_chat))
    app.add_handler(CommandHandler("menu", remember_chat))
    app.add_handler(CommandHandler("tip", lambda u, c: u.message.reply_text(escape_md(random.choice(TIPS)), parse_mode=ParseMode.MARKDOWN_V2)))
    app.add_handler(CommandHandler("ustala", lambda u, c: u.message.reply_text(escape_md(random.choice(MOTIVATIONAL)), parse_mode=ParseMode.MARKDOWN_V2)))

    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, check_baby_commands), group=1)
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, check_event_command), group=1)
    app.add_handler(MessageHandler(filters.TEXT | filters.CAPTION, check_message), group=2)
    app.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, greet_new_member), group=3)

    if app.job_queue:
        tz = pytz.timezone('Asia/Krasnoyarsk')
        app.job_queue.run_daily(daily_birthday_check, time=datetime.time(9, 0, tzinfo=tz))

    print("✅ Бот запущен. Слушаю чат...")
    app.run_polling(drop_pending_updates=True)

if __name__ == '__main__':
    main()
