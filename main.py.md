import os
import json
import logging
from pathlib import Path
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import Application, CommandHandler, MessageHandler, filters, CallbackQueryHandler, PreCheckoutQueryHandler, ContextTypes

# Токен берётся из переменной окружения (её зададим на хостинге)
TEST_MODE = True  # Если True – оплата не списывается, а сразу засчитывается
TOKEN = os.getenv("TOKEN")
if not TOKEN:
    raise ValueError("Переменная окружения TOKEN не задана!")

logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)
logger = logging.getLogger(__name__)

DATA_FILE = "users.json"

def load_data():
    if Path(DATA_FILE).exists():
        with open(DATA_FILE, "r") as f:
            return json.load(f)
    return {}

def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=2)

users_data = load_data()

def update_user_stats(user_id, username, full_name, stars_amount):
    user_id = str(user_id)
    if user_id not in users_data:
        users_data[user_id] = {
            "username": username,
            "name": full_name,
            "total_stars": 0
        }
    else:
        users_data[user_id]["username"] = username
        users_data[user_id]["name"] = full_name
    users_data[user_id]["total_stars"] += stars_amount
    save_data(users_data)

def main_menu_keyboard():
    keyboard = [[KeyboardButton("📊 Лидеры"), KeyboardButton("⭐ Закинуть stars")]]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

def after_payment_keyboard():
    keyboard = [[KeyboardButton("🔄 Хочу ещё"), KeyboardButton("📊 Лидеры")]]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

def leaders_inline_keyboard():
    keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data="back_to_main")]]
    return InlineKeyboardMarkup(keyboard)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    await update.message.reply_text(f"Привет, {user.first_name}! 👋\nЯ помогу тебе задонатить звёзды и попасть в лидеры.", reply_markup=main_menu_keyboard())

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text
    if text in ("⭐ Закинуть stars", "🔄 Хочу ещё"):
        await send_stars_invoice(update, context)
    elif text == "📊 Лидеры":
        await show_leaders(update, context)
    else:
        await update.message.reply_text("Используй кнопки меню.")

async def send_stars_invoice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    await context.bot.send_invoice(
        chat_id=chat_id,
        title="Пополнение баланса звёзд",
        description="100 000 Telegram Stars",
        payload="stars_payment",
        provider_token="",
        currency="XTR",
        prices=[{"label": "100 000 Stars", "amount": 100000}],
        start_parameter="test_payment"
    )

async def pre_checkout_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.pre_checkout_query.answer(ok=True)

async def successful_payment_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message = update.message
    user = message.from_user
    update_user_stats(user.id, user.username, user.full_name, 100000)
    await message.reply_text(f"✅ Спасибо за пополнение на 100000 Stars!\nТеперь ты в лидерах?", reply_markup=after_payment_keyboard())

async def show_leaders(update: Update, context: ContextTypes.DEFAULT_TYPE):
    sorted_users = sorted(users_data.items(), key=lambda x: x[1]["total_stars"], reverse=True)
    if not sorted_users:
        text = "Пока нет лидеров. Будь первым!"
    else:
        lines = ["🏆 *Топ пользователей по звёздам:*\n"]
        for idx, (uid, data) in enumerate(sorted_users[:10], 1):
            name = data.get("name", "No name")
            username = data.get("username")
            display_name = f"@{username}" if username else name
            stars = data["total_stars"]
            lines.append(f"{idx}. {display_name} (ID: {uid}) — {stars} ⭐")
        text = "\n".join(lines)
    await update.message.reply_text(text, parse_mode="Markdown", reply_markup=leaders_inline_keyboard())

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    if query.data == "back_to_main":
        await query.edit_message_text(text="Главное меню:")
        await query.message.reply_text("Выбери действие:", reply_markup=main_menu_keyboard())

def main():
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    app.add_handler(PreCheckoutQueryHandler(pre_checkout_callback))
    app.add_handler(MessageHandler(filters.SUCCESSFUL_PAYMENT, successful_payment_callback))
    app.add_handler(CallbackQueryHandler(button_callback, pattern="^back_to_main$"))
    logger.info("Бот запущен...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
