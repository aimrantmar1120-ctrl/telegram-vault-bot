import sqlite3
from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    ContextTypes,
)

# --- إعداد قاعدة البيانات ---
conn = sqlite3.connect("vault_bot.db", check_same_thread=False)
cursor = conn.cursor()

# جدول لتخزين العناصر المحفوظة
cursor.execute(
    """
CREATE TABLE IF NOT EXISTS items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    content TEXT,
    tag TEXT
)
"""
)

# جدول لإعدادات وحالة قفل المستخدمين والرمز السري
cursor.execute(
    """
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    pin TEXT,
    is_locked INTEGER DEFAULT 0
)
"""
)
conn.commit()


# --- دالة التحقق من القفل ---
async def check_lock(update: Update) -> bool:
    user_id = update.effective_user.id
    cursor.execute(
        "SELECT is_locked, pin FROM users WHERE user_id = ?", (user_id,)
    )
    res = cursor.fetchone()
    if res and res[0] == 1:
        await update.message.reply_text(
            "🔒 **Your vault is locked!**\n"
            "Please unlock it first using:\n`/unlock [your_pin]`",
            parse_mode="Markdown",
        )
        return True
    return False


# --- دالة البدء /start ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    cursor.execute(
        "INSERT OR IGNORE INTO users (user_id, pin, is_locked) VALUES (?, ?, ?)",
        (user_id, None, 0),
    )
    conn.commit()

    welcome_text = (
        "🚀 **Welcome to your Personal Vault!**\n\n"
        "Here is your secure space to store notes and files.\n\n"
        "📌 **Available Commands:**\n"
        "• `/setpin 1234` - Set your security PIN\n"
        "• `/save [text]` - Save a new note\n"
        "• `/list` - View your saved items\n"
        "• `/search [keyword]` - Search data\n"
        "• `/lock` - Lock your vault\n"
        "• `/unlock 1234` - Unlock your vault\n"
        "• `/export` - Download backup\n"
        "• `/help` - Show usage guide"
    )
    await update.message.reply_text(welcome_text, parse_mode="Markdown")


# --- دالة تعيين الرمز السري /setpin ---
async def set_pin(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if not context.args:
        await update.message.reply_text(
            "⚠️ Please provide a PIN number. Example:\n`/setpin 1234`",
            parse_mode="Markdown",
        )
        return

    new_pin = context.args[0]
    cursor.execute(
        "UPDATE users SET pin = ? WHERE user_id = ?", (new_pin, user_id)
    )
    conn.commit()
    await update.message.reply_text(
        "🔐 **PIN set successfully!** Keep it safe, you will need it to unlock your vault."
    )


# --- دالة قفل الخزنة /lock ---
async def lock_vault(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    cursor.execute(
        "SELECT pin FROM users WHERE user_id = ?", (user_id,)
    )
    row = cursor.fetchone()

    if not row or not row[0]:
        await update.message.reply_text(
            "⚠️ You haven't set a PIN yet! Please set one first using:\n`/setpin [your_pin]`",
            parse_mode="Markdown",
        )
        return

    cursor.execute(
        "UPDATE users SET is_locked = 1 WHERE user_id = ?", (user_id,)
    )
    conn.commit()
    await update.message.reply_text(
        "🔒 **Your vault has been locked.**\nUse `/unlock [your_pin]` to open it."
    )


# --- دالة فتح الخزنة /unlock ---
async def unlock_vault(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if not context.args:
        await update.message.reply_text(
            "⚠️ Please enter your PIN. Example:\n`/unlock 1234`",
            parse_mode="Markdown",
        )
        return

    entered_pin = context.args[0]
    cursor.execute(
        "SELECT pin FROM users WHERE user_id = ?", (user_id,)
    )
    row = cursor.fetchone()

    if row and row[0] == entered_pin:
        cursor.execute(
            "UPDATE users SET is_locked = 0 WHERE user_id = ?", (user_id,)
        )
        conn.commit()
        await update.message.reply_text(
            "🔓 **Vault unlocked successfully!** Welcome back."
        )
    else:
        await update.message.reply_text(
            "❌ **Incorrect PIN!** Please try again."
        )


# --- دالة الحفظ /save ---
async def save_item(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if await check_lock(update):
        return

    user_id = update.effective_user.id
    text = " ".join(context.args)

    if not text:
        await update.message.reply_text(
            "⚠️ Please provide text to save. Example:\n`/save Buy groceries`",
            parse_mode="Markdown",
        )
        return

    cursor.execute(
        "INSERT INTO items (user_id, content, tag) VALUES (?, ?, ?)",
        (user_id, text, "general"),
    )
    conn.commit()
    await update.message.reply_text(
        "📥 Saved successfully to your personal vault!"
    )


# --- دالة العرض /list ---
async def list_items(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if await check_lock(update):
        return

    user_id = update.effective_user.id
    cursor.execute(
        "SELECT id, content, tag FROM items WHERE user_id = ?", (user_id,)
    )
    rows = cursor.fetchall()

    if not rows:
        await update.message.reply_text("📋 Your vault is currently empty.")
        return

    response = "📋 **Your Saved Items:**\n\n"
    for row in rows:
        response += f"🆔 `#{row[0]}` | [{row[2]}] {row[1]}\n"

    await update.message.reply_text(response, parse_mode="Markdown")


# --- دالة البحث /search ---
async def search_items(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if await check_lock(update):
        return

    user_id = update.effective_user.id
    query = " ".join(context.args)

    if not query:
        await update.message.reply_text(
            "⚠️ Please specify a keyword. Example:\n`/search groceries`",
            parse_mode="Markdown",
        )
        return

    cursor.execute(
        "SELECT id, content FROM items WHERE user_id = ? AND content LIKE ?",
        (user_id, f"%{query}%"),
    )
    rows = cursor.fetchall()

    if not rows:
        await update.message.reply_text("🔍 No matching items found.")
        return

    response = f"🔍 **Search Results for '{query}':**\n\n"
    for row in rows:
        response += f"🆔 `#{row[0]}` - {row[1]}\n"

    await update.message.reply_text(response, parse_mode="Markdown")


# --- دالة تصدير البيانات /export ---
async def export_data(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if await check_lock(update):
        return

    user_id = update.effective_user.id
    cursor.execute("SELECT content, tag FROM items WHERE user_id = ?", (user_id,))
    rows = cursor.fetchall()

    if not rows:
        await update.message.reply_text("📦 No data to export.")
        return

    filename = f"vault_backup_{user_id}.txt"
    with open(filename, "w", encoding="utf-8") as f:
        for row in rows:
            f.write(f"Tag: [{row[1]}] - Content: {row[0]}\n")

    await update.message.reply_document(
        document=open(filename, "rb"),
        caption="📦 Here is your data backup file!",
    )


# --- دالة المساعدة /help ---
async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    help_text = (
        "❓ **Help & Usage Guide:**\n\n"
        "• `/setpin [pin]` - Set your security PIN code\n"
        "• `/save [text]` - Store notes securely\n"
        "• `/list` - View all saved items\n"
        "• `/lock` - Lock your vault immediately\n"
        "• `/unlock [pin]` - Unlock your vault using your PIN"
    )
    await update.message.reply_text(help_text, parse_Mode="Markdown")


# --- تشغيل البوت ---
def main():
    TOKEN = 8242727769:AAGl2pQ7XYl78Byct83lA3tJx1umjChl8qg"

    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("setpin", set_pin))
    app.add_handler(CommandHandler("lock", lock_vault))
    app.add_handler(CommandHandler("unlock", unlock_vault))
    app.add_handler(CommandHandler("save", save_item))
    app.add_handler(CommandHandler("list", list_items))
    app.add_handler(CommandHandler("search", search_items))
    app.add_handler(CommandHandler("export", export_data))
    app.add_handler(CommandHandler("help", help_command))

    print("🤖 Secure Vault Bot is running...")
    app.run_polling()


if __name__ == "__main__":
    main()