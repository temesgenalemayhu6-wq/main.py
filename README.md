import random
import asyncio
import sqlite3

from telegram import Update, ReplyKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    MessageHandler,
    filters,
    ContextTypes,
)

TOKEN = "YOUR_BOT_TOKEN"
ADMIN_ID = 400641223

# DATABASE
db = sqlite3.connect("players.db", check_same_thread=False)
cursor = db.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users(
    user_id INTEGER PRIMARY KEY,
    balance INTEGER DEFAULT 0,
    approved INTEGER DEFAULT 0
)
""")

db.commit()

players = {}
called_numbers = []
game_running = False

# GENERATE CARD
def generate_card():
    return random.sample(range(1, 76), 9)

# CHECK WIN
def is_winner(card):

    for number in card:
        if number not in called_numbers:
            return False

    return True

# START
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    keyboard = [
        ["Join Game"],
        ["My Card"],
        ["Balance"],
        ["Deposit"],
        ["Check Win"],
        ["Start Auto Game"],
        ["Stop Game"]
    ]

    reply_markup = ReplyKeyboardMarkup(
        keyboard,
        resize_keyboard=True
    )

    user_id = update.effective_user.id

    cursor.execute(
        "INSERT OR IGNORE INTO users(user_id) VALUES(?)",
        (user_id,)
    )

    db.commit()

    await update.message.reply_text(
        "🎉 Welcome to Beteseb Bingo!",
        reply_markup=reply_markup
    )

# JOIN
async def join(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user_id = update.effective_user.id

    cursor.execute(
        "SELECT approved FROM users WHERE user_id=?",
        (user_id,)
    )

    result = cursor.fetchone()

    if result[0] == 0:
        await update.message.reply_text(
            "❌ Deposit not approved!"
        )
        return

    players[user_id] = generate_card()

    await update.message.reply_text(
        f"🎫 Your Card:\n{players[user_id]}"
    )

# MY CARD
async def my_card(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user_id = update.effective_user.id

    if user_id in players:
        await update.message.reply_text(
            f"🎫 Your Card:\n{players[user_id]}"
        )
    else:
        await update.message.reply_text(
            "Join first!"
        )

# BALANCE
async def balance(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user_id = update.effective_user.id

    cursor.execute(
        "SELECT balance FROM users WHERE user_id=?",
        (user_id,)
    )

    result = cursor.fetchone()

    await update.message.reply_text(
        f"💰 Balance: {result[0]} Birr"
    )

# DEPOSIT
async def deposit(update: Update, context: ContextTypes.DEFAULT_TYPE):

    await update.message.reply_text(
        "💳 Send payment screenshot to admin."
    )

# APPROVE
async def approve(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user_id = update.effective_user.id

    if user_id != ADMIN_ID:
        return

    try:
        target = int(context.args[0])

        cursor.execute(
            "UPDATE users SET approved=1, balance=100 WHERE user_id=?",
            (target,)
        )

        db.commit()

        await update.message.reply_text(
            "✅ User approved!"
        )

    except:
        await update.message.reply_text(
            "Usage:\n/approve USER_ID"
        )

# AUTO GAME
async def auto_game(context: ContextTypes.DEFAULT_TYPE):

    global game_running

    while game_running:

        available = []

        for num in range(1, 76):
            if num not in called_numbers:
                available.append(num)

        if len(available) == 0:
            game_running = False
            break

        number = random.choice(available)

        called_numbers.append(number)

        for user_id in players:

            try:
                await context.bot.send_message(
                    chat_id=user_id,
                    text=f"📢 Number: {number}"
                )

            except:
                pass

        await asyncio.sleep(10)

# START GAME
async def start_auto(update: Update, context: ContextTypes.DEFAULT_TYPE):

    global game_running

    if update.effective_user.id != ADMIN_ID:
        return

    if game_running:
        await update.message.reply_text(
            "Already running!"
        )
        return

    game_running = True

    await update.message.reply_text(
        "▶ Game Started!"
    )

    asyncio.create_task(auto_game(context))

# STOP GAME
async def stop_game(update: Update, context: ContextTypes.DEFAULT_TYPE):

    global game_running

    if update.effective_user.id != ADMIN_ID:
        return

    game_running = False

    await update.message.reply_text(
        "⏹ Game Stopped!"
    )

# CHECK WIN
async def check_win(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user_id = update.effective_user.id

    if user_id not in players:
        await update.message.reply_text(
            "Join first!"
        )
        return

    if is_winner(players[user_id]):

        await update.message.reply_text(
            "🏆 BINGO! YOU WIN!"
        )

    else:

        await update.message.reply_text(
            "❌ Not yet!"
        )

# HANDLE BUTTONS
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):

    text = update.message.text

    if text == "/start":
        await start(update, context)

    elif text == "Join Game":
        await join(update, context)

    elif text == "My Card":
        await my_card(update, context)

    elif text == "Balance":
        await balance(update, context)

    elif text == "Deposit":
        await deposit(update, context)

    elif text == "Check Win":
        await check_win(update, context)

    elif text == "Start Auto Game":
        await start_auto(update, context)

    elif text == "Stop Game":
        await stop_game(update, context)

# APP
app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(
    MessageHandler(filters.TEXT, handle_message)
)

from telegram.ext import CommandHandler

app.add_handler(
    CommandHandler("approve", approve)
)

print("🎉 Beteseb Bingo Running...")

app.run_polling()
