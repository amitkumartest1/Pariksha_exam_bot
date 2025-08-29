import os
import hmac
import hashlib
import asyncio
import random
from threading import Thread
from datetime import datetime, timedelta

from aiogram import Bot, Dispatcher, executor, types
from flask import Flask, request
from dotenv import load_dotenv
import razorpay

# -------------------- Load environment --------------------
load_dotenv()
BOT_TOKEN = os.getenv("BOT_TOKEN")
RAZORPAY_KEY_ID = os.getenv("RAZORPAY_KEY_ID")
RAZORPAY_KEY_SECRET = os.getenv("RAZORPAY_KEY_SECRET")
WEBHOOK_SECRET = os.getenv("WEBHOOK_SECRET")

# -------------------- Init bot & razorpay --------------------
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(bot)
razorpay_client = razorpay.Client(auth=(RAZORPAY_KEY_ID, RAZORPAY_KEY_SECRET))

# -------------------- Import your question bank --------------------
# Expecting: questions_gk = [{ "id": 1, "question": "...", "options": ["A","B",...], "answer": "A" }, ...]
from bot import questions_gk

# -------------------- Subscription store (in-memory) --------------------
# NOTE: In production, store this in a database (e.g., SQLite/Postgres/Redis)
user_subscriptions = {}  # {user_id: expiry_datetime}

def has_active_subscription(user_id: int) -> bool:
    expiry = user_subscriptions.get(user_id)
    return bool(expiry and datetime.now() < expiry)

# -------------------- Quiz state (in-memory) --------------------
user_progress = {}  # {user_id: {"current_index": int, "questions": list, "answers": {}, "score": float, "end_time": datetime}}
TEST_DURATION = 15        # minutes
TOTAL_QUESTIONS = 20      # per test

# -------------------- Commands --------------------
@dp.message_handler(commands=['start'])
async def start_cmd(message: types.Message):
    user_id = message.from_user.id
    if has_active_subscription(user_id):
        left_days = (user_subscriptions[user_id] - datetime.now()).days
        await message.answer(
            f"👋 Welcome!\n\n✅ आपका plan active है (लगभग {left_days} दिन बाकी)।\n"
            f"📝 Test देने के लिए /quiz दबाएँ।"
        )
    else:
        await message.answer(
            "👋 Welcome!\n\n"
            "❌ आपका plan active नहीं है।\n"
            "यदि आप test देना चाहते हैं तो ₹49 से 28 दिनों का plan लें।\n"
            "👉 Recharge करने के लिए /buy दबाएँ।"
        )

@dp.message_handler(commands=['help'])
async def help_cmd(message: types.Message):
    await message.answer(
        "ℹ️ Commands:\n"
        "/buy – ₹49 का 28 दिन का plan खरीदें\n"
        "/quiz – 20 सवाल, 15 मिनट का टेस्ट (केवल active subscription पर)\n"
        "/start – स्टेटस देखें"
    )

@dp.message_handler(commands=['buy'])
async def buy_cmd(message: types.Message):
    """
    Creates a Razorpay Payment Link and sends it to the user.
    We embed Telegram user_id in the payment link notes for webhook mapping.
    """
    user_id = message.from_user.id

    try:
        payment_link = razorpay_client.payment_link.create({
            "amount": 4900,                # ₹49 in paise
            "currency": "INR",
            "description": "28 days plan (Student Quiz)",
            "customer": {
                "name": message.from_user.full_name or "TG User",
            },
            "notify": {"sms": False, "email": False},
            "reminder_enable": True,
            "notes": {"user_id": str(user_id)},
            "callback_method": "get",
        })
        short_url = payment_link.get("short_url")
        await message.answer(
            "💳 28 दिन का plan खरीदें (₹49):\n"
            f"{short_url}\n\n"
            "✅ Payment successful होते ही आपका access अपने-आप चालू हो जाएगा।"
        )
    except Exception as e:
        await message.answer("⚠️ Payment link बनाते समय दिक्कत आई। बाद में कोशिश करें।")
        print("Razorpay error:", e)

@dp.message_handler(commands=['quiz'])
async def quiz_cmd(message: types.Message):
    user_id = message.from_user.id

    # ---- Subscription gate ----
    if not has_active_subscription(user_id):
        await message.reply(
            "❌ आपका plan खत्म हो चुका है!\n\n"
            "अगर आप आगे test करना चाहते हैं तो ₹49 से recharge करें।\n"
            "👉 Recharge करने के लिए /buy दबाएं।"
        )
        return

    # ---- Start a fresh test ----
    if len(questions_gk) == 0:
        await message.answer("⚠️ प्रश्न उपलब्ध नहीं हैं.")
        return

    selected = random.sample(questions_gk, min(TOTAL_QUESTIONS, len(questions_gk)))
    user_progress[user_id] = {
        "questions": selected,
        "current_index": 0,
        "answers": {},           # {index: selected_option}
        "score": 0.0,
        "end_time": datetime.now() + timedelta(minutes=TEST_DURATION),
        "chat_id": message.chat.id
    }

    await message.answer(
        f"🕒 आपका test शुरू हो गया!\n"
        f"⏳ कुल समय: {TEST_DURATION} मिनट\n"
        f"🧮 कुल प्रश्न: {TOTAL_QUESTIONS}\n"
        f"📌 Scoring: सही +1 | गलत -⅓\n"
        f"👉 Navigation: Next / Previous / Skip / Finish"
    )

    # Start auto-finish timer
    asyncio.create_task(auto_finish_timer(user_id))

    await send_question(user_id)

async def send_question(user_id: int):
    """
    Sends the current question to the user's chat with navigation buttons.
    """
    progress = user_progress.get(user_id)
    if not progress:
        return

    # Time check
    if datetime.now() >= progress["end_time"]:
        await finish_test(user_id)
        return

    chat_id = progress["chat_id"]
    index = progress["current_index"]
    question = progress["questions"][index]

    kb = types.InlineKeyboardMarkup(row_width=2)

    # Options
    for opt in question["options"]:
        # Disable re-scoring here; we only record first selection for a question index
        kb.add(types.InlineKeyboardButton(text=opt, callback_data=f"ans|{index}|{opt}"))

    # Navigation
    nav_row = []
    if index > 0:
        nav_row.append(types.InlineKeyboardButton("⬅ Previous", callback_data="nav|prev"))
    if index < len(progress["questions"]) - 1:
        nav_row.append(types.InlineKeyboardButton("➡ Next", callback_data="nav|next"))
        nav_row.append(types.InlineKeyboardButton("⏭ Skip", callback_data="nav|skip"))
    else:
        nav_row.append(types.InlineKeyboardButton("🏁 Finish", callback_data="nav|finish"))

    if nav_row:
        kb.row(*nav_row)

    mins_left = max(0, (progress["end_time"] - datetime.now()).seconds // 60)
    await bot.send_message(
        chat_id,
        f"Q{index+1}/{TOTAL_QUESTIONS}: {question['question']}\n"
        f"⏳ Time Left: {mins_left} min",
        reply_markup=kb
    )

# -------------------- Callbacks: answers & navigation --------------------
@dp.callback_query_handler(lambda c: c.data.startswith("ans|"))
async def cb_answer(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    progress = user_progress.get(user_id)
    if not progress:
        await callback_query.answer("⚠️ कोई active test नहीं मिला.")
        return

    # Time check
    if datetime.now() >= progress["end_time"]:
        await finish_test(user_id)
        return

    _, idx_str, selected = callback_query.data.split("|")
    idx = int(idx_str)

    # Record first answer only (no re-scoring on change)
    if idx not in progress["answers"]:
        question = progress["questions"][idx]
        correct = question["answer"]
        if selected == correct:
            progress["score"] += 1.0
        else:
            progress["score"] -= (1/3)

        progress["answers"][idx] = selected

    await callback_query.answer("✅ Answer Saved!")

@dp.callback_query_handler(lambda c: c.data.startswith("nav|"))
async def cb_nav(callback_query: types.CallbackQuery):
    user_id = callback_query.from_user.id
    progress = user_progress.get(user_id)
    if not progress:
        await callback_query.answer("⚠️ कोई active test नहीं मिला.")
        return

    # Time check
    if datetime.now() >= progress["end_time"]:
        await finish_test(user_id)
        return

    action = callback_query.data.split("|")[1]

    if action == "prev" and progress["current_index"] > 0:
        progress["current_index"] -= 1
    elif action == "next" and progress["current_index"] < len(progress["questions"]) - 1:
        progress["current_index"] += 1
    elif action == "skip" and progress["current_index"] < len(progress["questions"]) - 1:
        progress["current_index"] += 1
    elif action == "finish":
        await finish_test(user_id)
        return

    await send_question(user_id)

# -------------------- Auto-finish timer --------------------
async def auto_finish_timer(user_id: int):
    progress = user_progress.get(user_id)
    if not progress:
        return
    now = datetime.now()
    secs = max(0, int((progress["end_time"] - now).total_seconds()))
    await asyncio.sleep(secs)
    if user_id in user_progress:  # still active
        await finish_test(user_id)

# -------------------- Finish test --------------------
async def finish_test(user_id: int):
    progress = user_progress.pop(user_id, None)
    if not progress:
        return
    score = progress["score"]
    total = len(progress["questions"])
    chat_id = progress["chat_id"]

    # Summary (right/wrong/attempted)
    attempted = len(progress["answers"])
    # To compute right/wrong from stored answers:
    right = 0
    wrong = 0
    for idx, sel in progress["answers"].items():
        if sel == progress["questions"][idx]["answer"]:
            right += 1
        else:
            wrong += 1

    await bot.send_message(
        chat_id,
        "🏁 Test Finished!\n"
        "Scoring: ✅ +1 | ❌ -⅓\n\n"
        f"📊 Final Score: {score} / {total}\n"
        f"✔️ Correct: {right} | ❌ Wrong: {wrong} | ⏭ Skipped/Unanswered: {total - attempted}"
    )

# -------------------- Flask webhook (Razorpay) --------------------
app = Flask(__name__)

@app.route("/razorpay-webhook", methods=["POST"])
def razorpay_webhook():
    """
    Handles Razorpay webhooks.
    - We verify signature with WEBHOOK_SECRET.
    - We accept either `payment.captured` or `payment_link.paid`.
    - We read user_id from `notes.user_id`.
    """
    try:
        payload = request.data
        received_sig = request.headers.get("X-Razorpay-Signature", "")

        # Verify signature
        generated_sig = hmac.new(
            bytes(WEBHOOK_SECRET or "", "utf-8"),
            msg=payload,
            digestmod=hashlib.sha256
        ).hexdigest()

        if not hmac.compare_digest(generated_sig, received_sig):
            return "Invalid signature", 400

        data = request.json
        event = data.get("event", "")

        # Extract user_id from notes depending on event type
        user_id = None
        if event == "payment.captured":
            # Payment entity path
            entity = data.get("payload", {}).get("payment", {}).get("entity", {})
            notes = entity.get("notes", {})
            if "user_id" in notes:
                user_id = int(notes["user_id"])
        elif event == "payment_link.paid":
            # Payment link entity path
            entity = data.get("payload", {}).get("payment_link", {}).get("entity", {})
            # payment_link doesn't always echo notes; we try
            notes = entity.get("notes", {})
            if "user_id" in notes:
                user_id = int(notes["user_id"])

        if user_id:
            user_subscriptions[user_id] = datetime.now() + timedelta(days=28)
            # Notify user
            try:
                asyncio.get_event_loop().create_task(
                    bot.send_message(user_id, "✅ Payment Successful! आपका plan 28 दिनों के लिए activate हो गया है.\n📝 Test के लिए /quiz दबाएँ.")
                )
            except RuntimeError:
                # If no running loop (rare), fallback
                asyncio.run(bot.send_message(user_id, "✅ Payment Successful! आपका plan 28 दिनों के लिए activate हो गया है.\n📝 Test के लिए /quiz दबाएँ."))

        return "OK", 200
    except Exception as e:
        print("Webhook error:", e)
        return "Error", 500

# -------------------- Runner (Bot + Webhook server) --------------------
def run_bot():
    executor.start_polling(dp, skip_updates=True)

def run_web():
    # Make sure you expose this port and set webhook URL in Razorpay dashboard:
    # https://yourdomain.com/razorpay-webhook
    app.run(host="0.0.0.0", port=5000)

if __name__ == "__main__":
    Thread(target=run_bot, daemon=True).start()
    run_web()