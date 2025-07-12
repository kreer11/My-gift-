from pyrogram import Client, filters
from pyrogram.types import InlineKeyboardMarkup, InlineKeyboardButton
import os

# ✅ التوكن وبيانات تسجيل الحساب
API_ID = 28880406  # ← استبدله بـ API_ID الحقيقي
API_HASH = "b9239e46289ad1f7f933b51b078f9cf0"  # ← استبدله بـ API_HASH الحقيقي
BOT_TOKEN = "8128627590:AAERfvy1SePQbQBm9rXGg32t3s3vSyBVEl8"  # ← التوكن الجديد

bot = Client("gift_bot", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

ADMIN_ID = 7694609633
ALLOWED_FILE = "allowed_users.txt"

def load_allowed_users():
    if not os.path.exists(ALLOWED_FILE):
        with open(ALLOWED_FILE, "w") as f:
            f.write(f"{ADMIN_ID}\n")
    with open(ALLOWED_FILE, "r") as f:
        return set(int(line.strip()) for line in f if line.strip().isdigit())

def save_allowed_users(users):
    with open(ALLOWED_FILE, "w") as f:
        for uid in sorted(users):
            f.write(f"{uid}\n")

allowed_users = load_allowed_users()
user_sessions = {}

GIFT_TYPES = {
    "1": "Gift A",
    "2": "Gift B",
    "3": "Gift C",
}

def buy_gift(sender_client, recipient_username, gift_type, price):
    print(f"شراء هدية {gift_type} بسعر {price} للمستخدم @{recipient_username}")
    return True

@bot.on_message(filters.command("start"))
async def start(client, message):
    user_id = message.from_user.id
    if user_id not in allowed_users:
        await message.reply_text("❌ لا تملك صلاحية استخدام هذا البوت.")
        return
    await message.reply_text("أرسل معرف الشخص المستلم للهدايا:")
    user_sessions[user_id] = {"step": "awaiting_recipient"}

@bot.on_message(filters.command("users") & filters.user(ADMIN_ID))
async def list_users(client, message):
    text = "🟢 المستخدمون المسموح لهم:\n"
    for uid in sorted(allowed_users):
        text += f"• `{uid}`\n"
    await message.reply_text(text)

@bot.on_message(filters.command("add") & filters.user(ADMIN_ID))
async def add_user(client, message):
    try:
        uid = int(message.text.split()[1])
        allowed_users.add(uid)
        save_allowed_users(allowed_users)
        await message.reply_text(f"✅ تم السماح للمستخدم `{uid}`.")
    except:
        await message.reply_text("❌ الصيغة الصحيحة: `/add 123456789`", parse_mode="markdown")

@bot.on_message(filters.command("remove") & filters.user(ADMIN_ID))
async def remove_user(client, message):
    try:
        uid = int(message.text.split()[1])
        if uid == ADMIN_ID:
            await message.reply_text("❌ لا يمكنك إزالة نفسك كمشرف.")
            return
        allowed_users.discard(uid)
        save_allowed_users(allowed_users)
        await message.reply_text(f"🚫 تم إزالة المستخدم `{uid}` من الصلاحيات.")
    except:
        await message.reply_text("❌ الصيغة الصحيحة: `/remove 123456789`", parse_mode="markdown")

@bot.on_message(filters.text & filters.private)
async def handle_steps(client, message):
    user_id = message.from_user.id
    if user_id not in allowed_users:
        await message.reply_text("❌ لا تملك صلاحية استخدام هذا البوت.")
        return

    session = user_sessions.get(user_id, {})
    step = session.get("step")

    if step == "awaiting_recipient":
        recipient = message.text.strip().lstrip("@")
        session["recipient"] = recipient
        session["step"] = "select_gift"
        user_sessions[user_id] = session

        buttons = [
            [InlineKeyboardButton(name, callback_data=f"gift_{key}")]
            for key, name in GIFT_TYPES.items()
        ]
        await message.reply_text(f"المستلم: @{recipient}\nاختر نوع الهدية:",
                                 reply_markup=InlineKeyboardMarkup(buttons))

    elif step == "awaiting_min_price":
        try:
            min_price = int(message.text.strip())
            if min_price <= 0:
                raise ValueError
            session["min_price"] = min_price
            session["step"] = "awaiting_max_price"
            user_sessions[user_id] = session
            await message.reply_text("أدخل الحد الأقصى للسعر (نجوم):")
        except:
            await message.reply_text("❌ يرجى إدخال رقم صحيح أكبر من صفر.")

    elif step == "awaiting_max_price":
        try:
            max_price = int(message.text.strip())
            if max_price < session["min_price"]:
                await message.reply_text("❌ الحد الأقصى يجب أن يكون أكبر من أو يساوي الحد الأدنى.")
                return
            session["max_price"] = max_price
            session["step"] = "ready_to_buy"
            user_sessions[user_id] = session
            await message.reply_text("🚀 جاري بدء عملية الشراء التلقائي...")
            await start_buying(client, message, session)
        except:
            await message.reply_text("❌ يرجى إدخال رقم صحيح للحد الأقصى.")

@bot.on_callback_query(filters.regex(r"^gift_\d+$"))
async def gift_selection(client, callback_query):
    user_id = callback_query.from_user.id
    if user_id not in allowed_users:
        await callback_query.answer("❌ لا تملك صلاحية.")
        return

    gift_type_key = callback_query.data.split("_")[1]
    session = user_sessions.get(user_id, {})
    session["gift_type"] = gift_type_key
    session["step"] = "awaiting_min_price"
    user_sessions[user_id] = session

    await callback_query.answer()
    await callback_query.message.reply_text("أدخل الحد الأدنى للسعر (نجوم):")

async def start_buying(client, message, session):
    recipient = session["recipient"]
    gift_type_key = session["gift_type"]
    gift_name = GIFT_TYPES[gift_type_key]
    min_price = session["min_price"]
    max_price = session["max_price"]

    await message.reply_text(f"📦 شراء هدايا '{gift_name}' لـ @{recipient} بين {min_price}-{max_price} نجوم")

    count = 0
    for price in range(min_price, max_price + 1):
        success = buy_gift(None, recipient, gift_name, price)
        if success:
            count += 1
            await message.reply_text(f"🎁 تم شراء هدية رقم {count} بسعر {price} نجوم.")
        else:
            await message.reply_text(f"⚠️ فشل الشراء بسعر {price} نجوم.")
            break

    await message.reply_text(f"✅ اكتملت العملية! عدد الهدايا: {count}")

if __name__ == "__main__":
    print("Bot is running...")
    bot.run()
