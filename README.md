import sqlite3
from aiogram import Bot, Dispatcher, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils import executor

TOKEN = "8478651409:AAGH8cALx1HFfT47yCRyr5nJnGi909SVsV8"
ADMIN_ID = 7549390143

bot = Bot(token=TOKEN)
dp = Dispatcher(bot)

conn = sqlite3.connect("bot.db", check_same_thread=False)
cursor = conn.cursor()

# Создаем таблицы, если их нет
cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    bonuses INTEGER DEFAULT 0
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS keys (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    days INTEGER,
    key_text TEXT UNIQUE,
    used INTEGER DEFAULT 0
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS purchases (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER,
    days INTEGER,
    payment_method TEXT,
    confirmed INTEGER DEFAULT 0,
    key_text TEXT
)
""")

# Проверяем наличие колонки receipt_file_id и добавляем, если нет
try:
    cursor.execute("ALTER TABLE purchases ADD COLUMN receipt_file_id TEXT")
    conn.commit()
except sqlite3.OperationalError:
    # Колонка уже существует
    pass

PRICES = {
    1: {"price": 145, "bonus": 30},
    3: {"price": 300, "bonus": 90},
    7: {"price": 700, "bonus": 210},
    30: {"price": 1300, "bonus": 900},
    60: {"price": 1600, "bonus": 2000},
}

TEXT = {
    "menu": "🏠 Главное меню",
    "catalog": "📦 Каталог",
    "cabinet": "👤 Мой кабинет",
    "bonus": "🎁 Бонусы",
    "admin_panel": "⚙️ Админ панель",
    "back_to_menu": "⬅️ Назад в меню",
    "choose_product": "🔥 Выберите продукт:",
    "choose_tariff": "💳 Выберите тариф:",
    "choose_pay": "💰 Выберите способ оплаты:",
    "send_receipt": "Пожалуйста, отправьте чек (фото или файл) для подтверждения оплаты.",
    "no_keys": "Ключи для этого тарифа временно отсутствуют. Обратитесь к администратору.",
    "thanks_for_purchase": "Спасибо за покупку! Ожидайте подтверждения оплаты.",
    "not_enough_bonus": "Недостаточно бонусов",
    "bonus_pay_success": "Оплата бонусами прошла успешно!\nВаш ключ: {key}",
    "confirm_payment": "Ваша оплата подтверждена. Ваш ключ: {key}",
    "add_keys_prompt": "Выберите категорию (кол-во дней) для добавления ключей:",
    "invalid_days": "Неверное количество дней. Попробуйте снова.",
    "max_keys": "Максимум 100 ключей за раз.",
    "keys_added": "Добавлено ключей: {count}",
}

# --- Клавиатуры ---

def main_keyboard(is_admin=False):
    kb = ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add(KeyboardButton(TEXT["catalog"]))
    kb.add(KeyboardButton(TEXT["cabinet"]), KeyboardButton(TEXT["bonus"]))
    kb.add(KeyboardButton("🗝 Мои ключи"))
    if is_admin:
        kb.add(KeyboardButton(TEXT["admin_panel"]))
    return kb

catalog_kb = InlineKeyboardMarkup(row_width=1).add(
    InlineKeyboardButton(text="🔥 CamelMOD", callback_data="prod_camel")
)

tariff_kb = InlineKeyboardMarkup(row_width=2).add(
    InlineKeyboardButton("1 день | 145₽", callback_data="tar_1"),
    InlineKeyboardButton("3 дня | 300₽", callback_data="tar_3"),
    InlineKeyboardButton("7 дней | 700₽", callback_data="tar_7"),
    InlineKeyboardButton("30 дней | 1300₽", callback_data="tar_30"),
    InlineKeyboardButton("60 дней | 1600₽", callback_data="tar_60"),
)

pay_kb = InlineKeyboardMarkup(row_width=1).add(
    InlineKeyboardButton("Оплата бонусами", callback_data="pay_bonus"),
    InlineKeyboardButton("💳 Приват24", callback_data="pay_privat"),
    InlineKeyboardButton("🏦 Тинькофф", callback_data="pay_tinkoff"),
)

admin_panel_kb = ReplyKeyboardMarkup(resize_keyboard=True).add(
    KeyboardButton("Добавить ключи"),
    KeyboardButton("Подтвердить покупки"),
    KeyboardButton(TEXT["back_to_menu"])
)

days_selection_kb = InlineKeyboardMarkup(row_width=3).add(
    InlineKeyboardButton("1 день", callback_data="addkey_1"),
    InlineKeyboardButton("3 дня", callback_data="addkey_3"),
    InlineKeyboardButton("7 дней", callback_data="addkey_7"),
    InlineKeyboardButton("30 дней", callback_data="addkey_30"),
    InlineKeyboardButton("60 дней", callback_data="addkey_60"),
    InlineKeyboardButton(TEXT["back_to_menu"], callback_data="admin_back")
)

# --- Вспомогательные функции ---

def get_user(user_id: int):
    cursor.execute("SELECT user_id, bonuses FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    if not row:
        cursor.execute("INSERT INTO users(user_id) VALUES (?)", (user_id,))
        conn.commit()
        return {"user_id": user_id, "bonuses": 0}
    else:
        return {"user_id": row[0], "bonuses": row[1]}

def update_user_bonuses(user_id: int, bonuses: int):
    cursor.execute("UPDATE users SET bonuses = ? WHERE user_id = ?", (bonuses, user_id))
    conn.commit()

def add_purchase(user_id: int, days: int, payment_method: str, receipt_file_id=None):
    cursor.execute(
        "INSERT INTO purchases (user_id, days, payment_method, confirmed, receipt_file_id) VALUES (?, ?, ?, 0, ?)",
        (user_id, days, payment_method, receipt_file_id)
    )
    conn.commit()
    return cursor.lastrowid

def confirm_purchase(purchase_id: int, key_text: str):
    cursor.execute("UPDATE purchases SET confirmed = 1, key_text = ? WHERE id = ?", (key_text, purchase_id))
    conn.commit()

def get_unused_key(days: int):
    cursor.execute("SELECT id, key_text FROM keys WHERE days = ? AND used = 0 LIMIT 1", (days,))
    return cursor.fetchone()

def mark_key_used(key_id: int):
    cursor.execute("UPDATE keys SET used = 1 WHERE id = ?", (key_id,))
    conn.commit()

def get_user_keys(user_id: int, limit=30):
    cursor.execute("SELECT key_text, days FROM purchases WHERE user_id = ? AND confirmed = 1 ORDER BY id DESC LIMIT ?", (user_id, limit))
    return cursor.fetchall()

def get_unconfirmed_purchases(limit=10):
    cursor.execute(
        "SELECT id, user_id, days, payment_method, receipt_file_id FROM purchases WHERE confirmed = 0 ORDER BY id DESC LIMIT ?",
        (limit,)
    )
    return cursor.fetchall()

def add_keys(days: int, keys: list):
    added = 0
    for key_text in keys:
        try:
            cursor.execute("INSERT OR IGNORE INTO keys (days, key_text) VALUES (?, ?)", (days, key_text.strip()))
            if cursor.rowcount > 0:
                added += 1
        except Exception:
            pass
    conn.commit()
    return added

# --- Состояния ---

user_states = {}
admin_states = {}

# --- Обработчики ---

@dp.message_handler(commands=['start'])
async def cmd_start(message: types.Message):
    user = get_user(message.from_user.id)
    is_admin = message.from_user.id == ADMIN_ID
    await message.answer(TEXT["menu"], reply_markup=main_keyboard(is_admin))

@dp.message_handler(lambda message: message.text == TEXT["catalog"])
async def catalog(message: types.Message):
    await message.answer(TEXT["choose_product"], reply_markup=catalog_kb)

@dp.callback_query_handler(lambda c: c.data == "prod_camel")
async def choose_tariff(callback: types.CallbackQuery):
    await callback.message.edit_text(TEXT["choose_tariff"], reply_markup=tariff_kb)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data.startswith("tar_"))
async def choose_payment(callback: types.CallbackQuery):
    days = int(callback.data.split("_")[1])
    user_states[callback.from_user.id] = {"days": days}
    await callback.message.edit_text(TEXT["choose_pay"], reply_markup=pay_kb)
    await callback.answer()

@dp.callback_query_handler(lambda c: c.data.startswith("pay_"))
async def callback_payment(callback: types.CallbackQuery):
    user_id = callback.from_user.id
    user = get_user(user_id)
    if user_id not in user_states or "days" not in user_states[user_id]:
        await callback.answer("Сначала выберите тариф.", show_alert=True)
        return
    days = user_states[user_id]["days"]
    pay_method = callback.data[4:]  # bonus, privat, tinkoff

    if pay_method == "bonus":
        cost = PRICES[days]["bonus"]
        if user["bonuses"] < cost:
            await callback.answer(TEXT["not_enough_bonus"], show_alert=True)
            return

        key = get_unused_key(days)
        if not key:
            await callback.message.edit_text(TEXT["no_keys"])
            await callback.answer()
            return

        new_bonus = user["bonuses"] - cost
        update_user_bonuses(user_id, new_bonus)

        mark_key_used(key[0])
        add_purchase(user_id, days, "bonus", None)

        await callback.message.edit_text(TEXT["bonus_pay_success"].format(key=key[1]))
        await callback.answer()
        return

    purchase_id = add_purchase(user_id, days, pay_method)
    user_states[user_id]["purchase_id"] = purchase_id

    if pay_method == "privat":
        payment_info = (
            f"Оплата через Приват24\n\n"
            f"Номер карты: 4149 1234 5678 9012\n"
            f"Получатель: Иван Иванов\n"
            f"Сумма: {PRICES[days]['price']}₽\n\n"
            + TEXT["send_receipt"]
        )
    elif pay_method == "tinkoff":
        payment_info = (
            f"Оплата через Тинькофф\n\n"
            f"Номер карты: 1234 5678 9012 3456\n"
            f"Получатель: Иван Иванов\n"
            f"Сумма: {PRICES[days]['price']}₽\n\n"
            + TEXT["send_receipt"]
        )
    else:
        payment_info = TEXT["send_receipt"]

    await callback.message.edit_text(payment_info)
    await callback.answer()

@dp.message_handler(content_types=['photo', 'document'])
async def receive_receipt(message: types.Message):
    user_id = message.from_user.id
    if user_id not in user_states or "purchase_id" not in user_states[user_id]:
        await message.answer("Сначала выберите тариф и способ оплаты.")
        return

    purchase_id = user_states[user_id]["purchase_id"]

    file_id = None
    if message.photo:
        file_id = message.photo[-1].file_id
    elif message.document:
        file_id = message.document.file_id

    if file_id is None:
        await message.answer("Неподдерживаемый формат файла.")
        return

    cursor.execute("UPDATE purchases SET receipt_file_id = ? WHERE id = ?", (file_id, purchase_id))
    conn.commit()

    text = f"Новая заявка на подтверждение оплаты от пользователя {user_id}.\n" \
           f"Покупка ID: {purchase_id}\n" \
           f"Тариф: {user_states[user_id]['days']} дней\n" \
           f"Способ оплаты: {get_purchase_payment_method(purchase_id)}\n" \
           f"Для подтверждения нажмите кнопку ниже."

    keyboard = InlineKeyboardMarkup().add(
        InlineKeyboardButton("Подтвердить оплату", callback_data=f"confirm_{purchase_id}")
    )

    await bot.send_message(ADMIN_ID, text, reply_markup=keyboard)

    if message.photo:
        await bot.send_photo(ADMIN_ID, file_id)
    elif message.document:
        await bot.send_document(ADMIN_ID, file_id)

    await message.answer(TEXT["thanks_for_purchase"])

def get_purchase_payment_method(purchase_id: int):
    cursor.execute("SELECT payment_method FROM purchases WHERE id = ?", (purchase_id,))
    row = cursor.fetchone()
    return row[0] if row else "Неизвестно"

@dp.callback_query_handler(lambda c: c.data.startswith("confirm_"))
async def confirm_payment_handler(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        await callback.answer("Только админ может подтверждать оплаты.", show_alert=True)
        return

    purchase_id = int(callback.data.split("_")[1])
    cursor.execute("SELECT user_id, days, confirmed FROM purchases WHERE id = ?", (purchase_id,))
    purchase = cursor.fetchone()

    if not purchase:
        await callback.answer("Заявка не найдена.", show_alert=True)
        return

    user_id, days, confirmed = purchase
    if confirmed == 1:
        await callback.answer("Заявка уже подтверждена.", show_alert=True)
        return

    key = get_unused_key(days)
    if not key:
        await callback.answer("Нет доступных ключей для этого тарифа.", show_alert=True)
        return

    mark_key_used(key[0])
    confirm_purchase(purchase_id, key[1])

    cursor.execute("SELECT bonuses FROM users WHERE user_id = ?", (user_id,))
    user_bonuses = cursor.fetchone()
    current_bonuses = user_bonuses[0] if user_bonuses else 0
    cursor.execute("UPDATE users SET bonuses = ? WHERE user_id = ?", (current_bonuses + PRICES[days]["bonus"], user_id))
    conn.commit()

    try:
        await bot.send_message(user_id, TEXT["confirm_payment"].format(key=key[1]))
    except:
        pass

    await callback.answer("Оплата подтверждена и ключ выдан.")
    await callback.message.edit_text(f"Заявка #{purchase_id} подтверждена. Ключ выдан: {key[1]}")

@dp.message_handler(lambda message: message.text == TEXT["cabinet"])
async def cabinet_handler(message: types.Message):
    user_id = message.from_user.id
    user = get_user(user_id)
    keys = get_user_keys(user_id)
    bonuses = user["bonuses"]

    keys_text = "\n".join([f"{k[1]} дней — {k[0]}" for k in keys]) if keys else "У вас нет ключей."
    text = (
        f"👤 Мой кабинет\n\n"
        f"🎁 Бонусы: {bonuses}\n\n"
        f"🗝 Ваши ключи (последние {min(len(keys), 30)}):\n{keys_text}"
    )
    await message.answer(text, reply_markup=main_keyboard(message.from_user.id == ADMIN_ID))

@dp.message_handler(lambda message: message.text == TEXT["bonus"])
async def bonus_handler(message: types.Message):
    await message.answer(
        "🎁 БОНУСЫ\n\n"
        "При покупке ключа вам начисляются бонусы:\n"
        "🔹 1 день — 30 бонусов\n"
        "🔹 3 дня — 90 бонусов\n"
        "🔹 7 дней — 210 бонусов\n"
        "🔹 30 дней — 900 бонусов\n"
        "🔹 60 дней — 1800 бонусов\n\n"
        "Бонусы начисляются после подтверждения оплаты."
    )

@dp.message_handler(lambda message: message.text == "🗝 Мои ключи")
async def user_keys_handler(message: types.Message):
    user_id = message.from_user.id
    keys = get_user_keys(user_id)
    if not keys:
        await message.answer("У вас нет ключей.")
        return

    keys_text = "\n".join([f"{k[1]} дней — {k[0]}" for k in keys])
    await message.answer(f"Ваши ключи:\n{keys_text}")

# --- Админ панель ---

@dp.message_handler(lambda message: message.text == TEXT["admin_panel"] and message.from_user.id == ADMIN_ID)
async def admin_panel(message: types.Message):
    admin_states[message.from_user.id] = {}
    await message.answer("⚙️ Админ панель", reply_markup=admin_panel_kb)

@dp.message_handler(lambda message: message.from_user.id == ADMIN_ID and message.text == "Добавить ключи")
async def admin_add_keys_start(message: types.Message):
    admin_states[message.from_user.id] = {"adding_keys": True}
    await message.answer(TEXT["add_keys_prompt"], reply_markup=days_selection_kb)

@dp.callback_query_handler(lambda c: c.data.startswith("addkey_") or c.data == "admin_back")
async def admin_days_selection(callback: types.CallbackQuery):
    if callback.data == "admin_back":
        admin_states[callback.from_user.id] = {}
        await callback.message.edit_text("⚙️ Админ панель", reply_markup=admin_panel_kb)
        await callback.answer()
        return

    days = int(callback.data.split("_")[1])
    admin_states[callback.from_user.id]["adding_keys"] = True
    admin_states[callback.from_user.id]["days"] = days
    admin_states[callback.from_user.id]["waiting_for_keys"] = True

    await callback.message.edit_text(
        f"Отправьте список ключей для тарифа {days} дней.\n"
        f"Максимум 100 ключей за раз.\n"
        f"Каждый ключ с новой строки."
    )
    await callback.answer()

@dp.message_handler(lambda message: message.from_user.id == ADMIN_ID)
async def admin_add_keys_receive(message: types.Message):
    state = admin_states.get(message.from_user.id, {})
    if not state.get("adding_keys") or not state.get("waiting_for_keys"):
        # Если не в режиме добавления ключей — игнор
        return  # <== добавлен отступ и тело условия

    keys_text = message.text.strip()
    keys_list = keys_text.split("\n")

    if len(keys_list) > 100:
        await message.answer("Максимум 100 ключей за раз. Попробуйте ещё раз.")
        return

    days = state.get("days")
    count = add_keys(days, keys_list)
    admin_states[message.from_user.id] = {}

    await message.answer(f"Добавлено ключей: {count}", reply_markup=admin_panel_kb)
    
    from aiogram import executor

async def on_shutdown(dp):
    await bot.session.close()

if __name__ == "__main__":
    executor.start_polling(dp, on_shutdown=on_shutdown)