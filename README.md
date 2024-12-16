import logging
from aiogram import Bot, Dispatcher, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.utils import executor
from aiogram.dispatcher.filters.state import State, StatesGroup
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters import Text
import random
from aiocryptopay import AioCryptoPay
import sqlite3
import asyncio
from datetime import datetime

active_invoices = {}  # Хранение активных инвойсов пользователя
withdraw_requests = {}  # Хранение заявок на вывод средств
API_TOKEN = "7800223663:AAFJgrB8_Wvsh229fsSZeWavutD3hT0jBa0"
ADMIN_ID = "6532747969"
CRYPTOPAY_API_KEY = "306796:AA5KDo1nwuvFlIxUntKRGWY3nu8X3S3AMxw"

logging.basicConfig(level=logging.INFO)

# Initialize bot and dispatcher
bot = Bot(token=API_TOKEN)
storage = MemoryStorage()
dp = Dispatcher(bot, storage=storage)
cryptopay = AioCryptoPay(token=CRYPTOPAY_API_KEY)


def get_main_menu(user_id):
    menu = ReplyKeyboardMarkup(resize_keyboard=True)
    menu.add(KeyboardButton("🎲 Игра"), KeyboardButton("💳 Баланс"))
    menu.add(KeyboardButton("👤 Профиль"))  # Кнопка "Профиль"
    if str(user_id) == ADMIN_ID:  # Если пользователь - администратор
        menu.add(KeyboardButton("⚙️ Админ панель"))
    return menu


# Initialize database
def init_db():
    conn = sqlite3.connect("users.db", check_same_thread=False)
    cursor = conn.cursor()

    # Таблица пользователей
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            balance REAL DEFAULT 0,
            bonus_balance REAL DEFAULT 0,
            frozen_balance REAL DEFAULT 0
        )
    """)

    # Таблица депозитов
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS deposits (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            amount REAL,
            wagered REAL DEFAULT 0,
            is_completed INTEGER DEFAULT 0,
            FOREIGN KEY (user_id) REFERENCES users (user_id)
        )
    """)

    # Проверяем и добавляем недостающие столбцы
    try:
        cursor.execute("ALTER TABLE deposits ADD COLUMN wagered REAL DEFAULT 0")
    except sqlite3.OperationalError:
        pass  # Если столбец уже существует, игнорируем ошибку

    conn.commit()
    conn.close()

def add_deposit(user_id, amount):
    conn = sqlite3.connect("users.db", check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO deposits (user_id, amount, wagered, is_completed)
        VALUES (?, ?, 0, 0)
    """, (user_id, amount))
    conn.commit()
    conn.close()

def check_all_deposits_completed(user_id):
    conn = sqlite3.connect("users.db", check_same_thread=False)
    cursor = conn.cursor()

    # Проверяем депозиты пользователя
    cursor.execute("""
        SELECT amount, wagered, is_completed
        FROM deposits
        WHERE user_id = ? AND is_completed = 0
    """, (user_id,))
    deposits = cursor.fetchall()
    conn.close()

    # Проверяем, выполнены ли условия по всем активным депозитам
    for deposit in deposits:
        amount, wagered, is_completed = deposit
        if wagered < amount:  # Если проставлено меньше, чем сумма депозита
            return False
    return True

def update_wagered(user_id, bet_amount):
    conn = sqlite3.connect("users.db", check_same_thread=False)
    cursor = conn.cursor()

    # Обновляем проставленные суммы для активных депозитов
    cursor.execute("""
        SELECT id, amount, wagered FROM deposits
        WHERE user_id = ? AND is_completed = 0
    """, (user_id,))
    deposits = cursor.fetchall()

    for deposit in deposits:
        deposit_id, amount, wagered = deposit
        remaining = amount - wagered

        if bet_amount >= remaining:
            # Условие депозита выполнено
            cursor.execute("""
                UPDATE deposits
                SET wagered = ?, is_completed = 1
                WHERE id = ?
            """, (amount, deposit_id))
            bet_amount -= remaining
        else:
            # Частично обновляем wagered
            cursor.execute("""
                UPDATE deposits
                SET wagered = ?
                WHERE id = ?
            """, (wagered + bet_amount, deposit_id))
            bet_amount = 0

        if bet_amount <= 0:
            break

    conn.commit()
    conn.close()
def get_user_data(user_id):
    conn = sqlite3.connect("users.db", check_same_thread=False)
    cursor = conn.cursor()

    # Получаем основные данные пользователя
    cursor.execute("""
        SELECT balance, bonus_balance, frozen_balance
        FROM users
        WHERE user_id = ?
    """, (user_id,))
    result = cursor.fetchone()

    if result is None:  # Если пользователь не найден, создаём запись
        cursor.execute("""
            INSERT INTO users (user_id) VALUES (?)
        """, (user_id,))
        conn.commit()
        result = (0, 0, 0)  # Значения по умолчанию для balance, bonus_balance, frozen_balance

    # Подсчитываем проставленную сумму по активным депозитам
    cursor.execute("""
        SELECT SUM(wagered)
        FROM deposits
        WHERE user_id = ?
    """, (user_id,))
    total_wagered = cursor.fetchone()[0] or 0  # Если NULL, то заменить на 0

    # Подсчитываем сумму депозитов
    cursor.execute("""
        SELECT SUM(amount)
        FROM deposits
        WHERE user_id = ?
    """, (user_id,))
    total_deposits = cursor.fetchone()[0] or 0  # Если NULL, то заменить на 0

    conn.close()

    return {
        "balance": result[0],
        "bonus_balance": result[1],
        "frozen_balance": result[2],
        "wagered": total_wagered,
        "total_deposits": total_deposits
    }

from aiogram.types import ReplyKeyboardRemove

def number_keyboard_with_emojis():
    keyboard = ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
    keyboard.add(
        KeyboardButton("🎲 1"),
        KeyboardButton("🎲 2"),
        KeyboardButton("🎲 3"),
        KeyboardButton("🎲 4"),
        KeyboardButton("🎲 5"),
        KeyboardButton("🎲 6")
    )
    return keyboard

def update_user_data(user_id, balance=None, bonus_balance=None, wagered=None, total_deposits=None, frozen_balance=None):
    conn = sqlite3.connect("users.db", check_same_thread=False)
    cursor = conn.cursor()
    if balance is not None:
        cursor.execute("UPDATE users SET balance = ? WHERE user_id = ?", (balance, user_id))
    if bonus_balance is not None:
        cursor.execute("UPDATE users SET bonus_balance = ? WHERE user_id = ?", (bonus_balance, user_id))
    if wagered is not None:
        cursor.execute("UPDATE users SET wagered = ? WHERE user_id = ?", (wagered, user_id))
    if total_deposits is not None:
        cursor.execute("UPDATE users SET total_deposits = ? WHERE user_id = ?", (total_deposits, user_id))
    if frozen_balance is not None:
        cursor.execute("UPDATE users SET frozen_balance = ? WHERE user_id = ?", (frozen_balance, user_id))
    conn.commit()
    conn.close()

init_db()

# Keyboards
main_menu = ReplyKeyboardMarkup(resize_keyboard=True).add(KeyboardButton("🎲 Игра"), KeyboardButton("💳 Баланс"))
admin_menu = ReplyKeyboardMarkup(resize_keyboard=True).add(
    KeyboardButton("📤 Просмотр заявок на выплату"),
    KeyboardButton("💳 Управление балансом"),
    KeyboardButton("📋 База пользователей"),
    KeyboardButton("⬅️ Назад")
)

class AdminState(StatesGroup):
    select_user = State()
    set_balance = State()
game_menu = ReplyKeyboardMarkup(resize_keyboard=True).add(KeyboardButton("🎲 Кубик 🎯"), KeyboardButton("⬅️ Назад"))
balance_menu = ReplyKeyboardMarkup(resize_keyboard=True).add(
    KeyboardButton("💵 Пополнить баланс 💳"),
    KeyboardButton("📤 Вывести деньги 💰"),
    KeyboardButton("⬅️ Назад")
)
# Добавьте здесь клавиатуру для повторения ставки
repeat_menu = ReplyKeyboardMarkup(resize_keyboard=True)
repeat_menu.add(
    KeyboardButton("🔄 Повторить ставку"),
    KeyboardButton("⬅️ Назад")
)
# States
class GameState(StatesGroup):
    guess_number = State()
    set_bet = State()

class DepositState(StatesGroup):
    set_amount = State()

class WithdrawState(StatesGroup):
    set_amount = State()

# Helper function to handle "Back" button
async def back_to_main_menu(message: types.Message, state: FSMContext):
    await state.finish()
    await message.answer("🔙 Главное меню:", reply_markup=main_menu)

# Handlers
@dp.message_handler(Text("⬅️ Назад"), state="*")
async def handle_back(message: types.Message, state: FSMContext):
    await state.finish()
    menu = get_main_menu(message.from_user.id)  # Динамическое создание меню
    await message.answer("🔙 Главное меню:", reply_markup=menu)

@dp.message_handler(Text("⬅️ Назад"), state="*")
async def handle_back(message: types.Message, state: FSMContext):
    if str(message.from_user.id) == ADMIN_ID:
        await message.answer("⚙️ Админ-панель:", reply_markup=admin_menu)
    else:
        await message.answer("🔙 Главное меню:", reply_markup=main_menu)
    await state.finish()

@dp.message_handler(commands=['start'])
async def start_handler(message: types.Message):
    user_id = message.from_user.id
    # Убедимся, что пользователь есть в базе и получим его данные
    user_data = get_user_data(user_id)

    # Формируем меню с учётом ID пользователя
    menu = get_main_menu(user_id)

    # Приветственное сообщение
    profile_message = (
        f"👋 Привет, {message.from_user.first_name}!\n\n"
        f"🎲 Добро пожаловать в нашу игру! Здесь вы можете испытать удачу, делая ставки и выигрывая реальные деньги.\n\n"
        f"🆔 Ваш ID: `{user_id}`\n\n"  # ID в формате моноширинного текста
        f"📊 Ваша статистика:\n"
        f"💰 Баланс: ${user_data['balance']:.2f}\n"
        f"🎁 Бонусный баланс: ${user_data['bonus_balance']:.2f}\n"
        f"📊 Проставлено: ${user_data['wagered']:.2f}\n"
        f"💵 Общий депозит: ${user_data['total_deposits']:.2f}\n\n"
        f"🎮 Логика игры: выберите число от 1 до 6, сделайте ставку и выиграйте x5 от вашей ставки!"
    )

    # Прикрепляем изображение и отправляем сообщение
    try:
        with open("welcome_image.jpg", "rb") as photo:  # Укажите путь к вашему изображению
            await message.answer_photo(photo, caption=profile_message, reply_markup=menu, parse_mode="Markdown")
    except FileNotFoundError:
        # Если изображение не найдено, отправляем только текст
        await message.answer(profile_message, reply_markup=menu, parse_mode="Markdown")

@dp.message_handler(Text("👤 Профиль"))
async def profile_button_handler(message: types.Message):
    user_id = message.from_user.id
    user_data = get_user_data(user_id)  # Получаем данные пользователя
    logging.info(f"Данные пользователя {user_id}: {user_data}")  # Логируем данные
    await start_handler(message)  # Вызов функции обработки команды /start

@dp.message_handler(Text("🎲 Игра"))
async def game_handler(message: types.Message):
    await message.answer("🎯 Выберите игру и начинайте выигрывать!", reply_markup=game_menu)

@dp.message_handler(Text("🎲 Кубик 🎯"))
async def start_dice_game(message: types.Message):
    await message.answer(
        "🎲 Выберите число от 1 до 6:",
        reply_markup=number_keyboard_with_emojis()
    )
    await GameState.guess_number.set()

@dp.message_handler(state=GameState.guess_number)
async def set_dice_guess(message: types.Message, state: FSMContext):
    guess = message.text
    # Проверяем, что введённое значение совпадает с кнопками
    valid_guesses = ["🎲 1", "🎲 2", "🎲 3", "🎲 4", "🎲 5", "🎲 6"]
    if guess not in valid_guesses:
        await message.answer("⚠️ Пожалуйста, выберите число от 1 до 6, используя клавиатуру ниже.")
        return

    # Извлекаем число из текста (например, "🎲 1" -> 1)
    selected_number = int(guess.split()[1])  # Разделяем текст и извлекаем число
    await state.update_data(guess=selected_number)
    await message.answer(
        "💸 Введите сумму ставки в долларах (от 0.5):",
        reply_markup=ReplyKeyboardRemove()  # Убираем клавиатуру
    )
    await GameState.set_bet.set()

@dp.message_handler(state=GameState.set_bet)
async def process_bet(message: types.Message, state: FSMContext):
    bet = message.text
    if not bet.replace('.', '', 1).isdigit() or float(bet) < 0.1:
        await message.answer("⚠️ Минимальная ставка $0.1. Введите корректную сумму.")
        return

    user_id = message.from_user.id
    user_data = get_user_data(user_id)

    bet = float(bet)
    if user_data["balance"] < bet:
        await message.answer("❌ Недостаточно средств на балансе.")
        await state.finish()
        return

    # Списываем ставку
    update_user_data(user_id, balance=user_data["balance"] - bet, wagered=user_data["wagered"] + bet)

    # Отправляем анимацию кубика
    dice_message = await message.answer_dice()
    await asyncio.sleep(5)  # Ожидание завершения анимации
    dice_result = dice_message.dice.value

    data = await state.get_data()
    guess = data["guess"]

    if guess == dice_result:
        # Выигрыш
        win_amount = bet * 3.5 - bet
        update_user_data(user_id, balance=user_data["balance"] + win_amount)
        updated_balance = get_user_data(user_id)["balance"]  # Получаем обновлённый баланс
        await message.answer(
            f"🎉 Поздравляем! Выпало число {dice_result}. Ваш выигрыш (чистая прибыль): ${win_amount:.2f}! 🥳\n"
            f"💰 Ваш обновлённый баланс: ${updated_balance:.2f}",
            reply_markup=repeat_menu  # Показать меню для повторения ставки
        )
    else:
        # Проигрыш
        updated_balance = get_user_data(user_id)["balance"]  # Получаем обновлённый баланс
        await message.answer(
            f"😔 Вы не угадали. Выпало число {dice_result}. Ставка проиграна.\n"
            f"💰 Ваш обновлённый баланс: ${updated_balance:.2f}",
            reply_markup=repeat_menu  # Показать меню для повторения ставки
        )
    await state.finish()

@dp.message_handler(Text("🔄 Повторить ставку"))
async def repeat_bet(message: types.Message):
    await message.answer(
        "🎲 Выберите число от 1 до 6:",
        reply_markup=number_keyboard_with_emojis()  # Вызов клавиатуры с числами
    )
    await GameState.guess_number.set()

@dp.message_handler(Text("⬅️ Назад"))
async def back_to_main_menu_from_repeat(message: types.Message, state: FSMContext):
    await state.finish()
    menu = get_main_menu(message.from_user.id)
    await message.answer("🔙 Главное меню:", reply_markup=menu)



@dp.message_handler(Text("💳 Баланс"))
async def balance_handler(message: types.Message):
    user_id = message.from_user.id
    user_data = get_user_data(user_id)
    await message.answer(
        f"💰 Ваш баланс: ${user_data['balance']:.2f}\n🎁 Бонусный баланс: ${user_data['bonus_balance']:.2f}\n📊 Проставлено: ${user_data['wagered']:.2f}",
        reply_markup=balance_menu
    )

@dp.message_handler(Text("💵 Пополнить баланс 💳"))
async def deposit_handler(message: types.Message):
    user_id = message.from_user.id

    if user_id in active_invoices:  # Проверяем, есть ли активный инвойс
        await message.answer(
            "❌ У вас уже есть активное пополнение. Сначала завершите его или отмените."
        )
        return

    await message.answer("💵 Введите сумму пополнения в USDT (минимум 0.5):")
    await DepositState.set_amount.set()

@dp.message_handler(state=DepositState.set_amount)
async def process_deposit(message: types.Message, state: FSMContext):
    amount = message.text
    if not amount.replace('.', '', 1).isdigit() or float(amount) < 0.5:
        await message.answer("⚠️ Введите корректную сумму в USDT (минимум 0.5).")
        return

    amount = float(amount)
    user_id = message.from_user.id

    if user_id in active_invoices:  # Проверяем, есть ли активный инвойс
        await message.answer(
            "❌ У вас уже есть активное пополнение. Сначала завершите его или отмените."
        )
        return

    try:
        # Создаём инвойс через CryptoPay
        invoice = await cryptopay.create_invoice(
            asset="USDT",
            amount=amount,
            description=f"Пополнение баланса для пользователя {user_id}",
            payload=str(user_id)
        )

        # Сохраняем активный инвойс
        active_invoices[user_id] = {
            "invoice_id": invoice.invoice_id,
            "amount": amount
        }

        # URL для оплаты
        bot_invoice_url = invoice.bot_invoice_url

        # Клавиатура с кнопками "Оплатить" и "Отменить"
        keyboard = InlineKeyboardMarkup()
        keyboard.add(InlineKeyboardButton(text=f"Оплатить ${amount:.2f} USDT 💵", url=bot_invoice_url))
        keyboard.add(InlineKeyboardButton(text="❌ Отменить", callback_data=f"cancel_deposit_{user_id}"))

        # Отправляем клавиатуру пользователю
        await message.answer("✅ Оплатите счёт через CryptoBot:", reply_markup=keyboard)

        # Бесконечный цикл проверки оплаты
        async def wait_for_payment():
            while user_id in active_invoices:  # Пока инвойс активен
                try:
                    invoices = await cryptopay.get_invoices(invoice_ids=[invoice.invoice_id])
                    logging.info(f"Инвойс: {invoices[0]}")  # Выводим всю информацию об инвойсе
                    if invoices[0].status == "paid":  # Если инвойс оплачен
                        user_data = get_user_data(user_id)

                        # Обновляем баланс
                        new_balance = user_data["balance"] + amount
                        update_user_data(
                            user_id,
                            balance=new_balance,
                            total_deposits=user_data["total_deposits"] + amount,
                            wagered=0  # Сброс проставленной суммы
                        )
                        logging.info(f"Баланс пользователя {user_id} обновлён: {new_balance}")

                        # Уведомляем пользователя об успешной оплате
                        await message.answer(
                            f"💵 Оплата успешно зачислена на ваш баланс!\n"
                            f"💰 Новый баланс: ${new_balance:.2f}"
                        )

                        # Удаляем активный инвойс
                        del active_invoices[user_id]
                        break
                except Exception as e:
                    logging.error(f"Ошибка при проверке инвойса {invoice.invoice_id}: {e}")

                await asyncio.sleep(10)  # Ждём 10 секунд перед следующей проверкой

        asyncio.create_task(wait_for_payment())
    except Exception as e:
        logging.error(f"Error creating invoice: {e}")
        await message.answer("❌ Не удалось создать счёт. Попробуйте позже.")

    await state.finish()

@dp.message_handler(Text("📤 Вывести деньги 💰"))
async def withdraw_handler(message: types.Message):
    user_id = message.from_user.id
    user_data = get_user_data(user_id)

    #if user_data["wagered"] < user_data["total_deposits"] * 2:
    #    await message.answer(
    #        f"❌ Для вывода необходимо проставить минимум в два раза больше суммы депозита.\n📊 Проставлено: ${user_data['wagered']:.2f}\n💵 Необходимо: ${user_data['total_deposits'] * 2:.2f}"
    #    )
    #    return

    await message.answer("📤 Введите сумму вывода в USDT:")
    await WithdrawState.set_amount.set()

@dp.message_handler(Text("⚙️ Админ панель"))
async def admin_panel_handler(message: types.Message):
    if str(message.from_user.id) != ADMIN_ID:
        await message.answer("⛔ Доступ запрещён.")
        return
    await message.answer("⚙️ Добро пожаловать в админ-панель.", reply_markup=admin_menu)

# Новый хендлер для просмотра заявок
@dp.message_handler(Text("📤 Просмотр заявок на выплату"))
async def view_withdraw_requests(message: types.Message):
    if str(message.from_user.id) != ADMIN_ID:
        await message.answer("⛔ Доступ запрещён.")
        return

    if not withdraw_requests:
        await message.answer("📭 Нет активных заявок на выплату.")
        return

    for user_id, requests in withdraw_requests.items():
        for idx, request in enumerate(requests):
            username = f"@{request['username']}" if request["username"] else "не указан"
            inline_kb = InlineKeyboardMarkup().add(
                InlineKeyboardButton(text="✅ Выплатить", callback_data=f"approve_withdraw_{user_id}_{idx}"),
                InlineKeyboardButton(text="❌ Отменить", callback_data=f"cancel_withdraw_{user_id}_{idx}")
            )
            await message.answer(
                f"📄 Заявка #{idx + 1} от пользователя {user_id} ({username}):\n"
                f"💸 Сумма: ${request['amount']:.2f}\n"
                f"📅 Дата: {request['date']}",
                reply_markup=inline_kb
            )

@dp.message_handler(state=WithdrawState.set_amount)
async def process_withdraw(message: types.Message, state: FSMContext):
    amount = message.text
    user_id = message.from_user.id
    user_data = get_user_data(user_id)
    username = message.from_user.username

    # Проверяем выполнение условий по депозитам
    if not check_all_deposits_completed(user_id):
        await message.answer(
            "❌ Вы не можете вывести средства, пока не выполните условия по всем вашим депозитам.\n"
            "Проставьте сумму каждого депозита хотя бы один раз."
        )
        await state.finish()
        return

    # Проверка корректности введённой суммы
    if not amount.replace('.', '', 1).isdigit() or float(amount) <= 0:
        await message.answer("⚠️ Введите корректную сумму для вывода.")
        return

    amount = float(amount)

    # Проверка баланса пользователя
    if user_data["balance"] < amount:
        await message.answer("❌ Недостаточно средств на балансе.")
        await state.finish()
        return

    # Замораживаем средства
    new_balance = user_data["balance"] - amount
    new_frozen_balance = user_data.get("frozen_balance", 0) + amount
    update_user_data(user_id, balance=new_balance, frozen_balance=new_frozen_balance)

    # Сохраняем заявку
    if user_id not in withdraw_requests:
        withdraw_requests[user_id] = []

    withdraw_requests[user_id].append({
        "amount": amount,
        "date": datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        "username": username
    })

    # Уведомляем пользователя
    await message.answer(
        f"📤 Запрос на вывод средств принят на сумму ${amount:.2f}.\n⏳ Ожидайте обработки заявки.",
        parse_mode="Markdown"
    )

    # Уведомляем администратора
    await bot.send_message(
        ADMIN_ID,
        f"🚨 Новая заявка на выплату от пользователя:\n"
        f"👤 ID: {user_id} ({f'@{username}' if username else 'username отсутствует'})\n"
        f"💸 Сумма: ${amount:.2f}\n📅 Дата: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n"
        f"Для обработки используйте команду /view_requests."
    )

    await state.finish()

    @dp.message_handler(commands=["view_requests"])
    async def view_withdraw_requests(message: types.Message):
        if str(message.from_user.id) != ADMIN_ID:
            await message.answer("⛔ Доступ запрещён.")
            return

        if not withdraw_requests:
            await message.answer("📭 Нет активных заявок на выплату.")
            return

        for user_id, requests in withdraw_requests.items():
            for idx, request in enumerate(requests):
                username = f"@{request['username']}" if request["username"] else "не указан"
                inline_kb = InlineKeyboardMarkup().add(
                    InlineKeyboardButton(text="✅ Выплатить", callback_data=f"approve_withdraw_{user_id}_{idx}"),
                    InlineKeyboardButton(text="❌ Отменить", callback_data=f"cancel_withdraw_{user_id}_{idx}")
                )
                await message.answer(
                    f"📄 Заявка #{idx + 1} от пользователя {user_id} ({username}):\n"
                    f"💸 Сумма: ${request['amount']:.2f}\n"
                    f"📅 Дата: {request['date']}",
                    reply_markup=inline_kb
                )

    @dp.callback_query_handler(lambda call: call.data.startswith("cancel_withdraw_"))
    async def cancel_withdraw_request(call: types.CallbackQuery):
        user_id = int(call.data.split("_")[-1])

        if user_id not in withdraw_requests:
            await call.answer("❌ Заявка уже отменена или не найдена.", show_alert=True)
            return

        # Отменяем заявку
        request = withdraw_requests.pop(user_id)
        update_user_data(
            user_id,
            frozen_balance=get_user_data(user_id)["frozen_balance"] - request["amount"],
            balance=get_user_data(user_id)["balance"] + request["amount"]
        )

        await call.message.edit_text("✅ Заявка на вывод отменена. Средства возвращены на баланс.")
        await call.answer("Заявка отменена.")

    @dp.callback_query_handler(lambda call: call.data.startswith("approve_withdraw_"))
    async def approve_withdraw_request(call: types.CallbackQuery):
        print(f"Callback data: {call.data}")
        # Извлекаем user_id и индекс заявки из callback_data
        _, user_id, idx = call.data.split("_")
        user_id = int(user_id)
        idx = int(idx)

        if user_id not in withdraw_requests or idx >= len(withdraw_requests[user_id]):
            await call.message.edit_text("❌ Заявка уже обработана или не найдена.")
            return

        # Получаем заявку по индексу
        request = withdraw_requests[user_id].pop(idx)
        if not withdraw_requests[user_id]:  # Удаляем пользователя из списка, если больше нет заявок
            del withdraw_requests[user_id]

        amount = request['amount']

        try:
            # Получаем текущий баланс аккаунта
            balances = await cryptopay.get_balance()
            usdt_balance = next((balance.available for balance in balances if balance.currency_code == "USDT"), 0)

            # Проверяем, достаточно ли средств для перевода
            if float(usdt_balance) < amount:
                await call.message.edit_text(
                    f"❌ Недостаточно средств на счёте для выполнения выплаты.\n"
                    f"Доступно: ${usdt_balance:.2f}, требуется: ${amount:.2f}."
                )
                # Возвращаем заявку в список
                if user_id not in withdraw_requests:
                    withdraw_requests[user_id] = []
                withdraw_requests[user_id].insert(idx, request)
                return

            # Выполняем перевод
            transfer_result = await cryptopay.transfer(
                user_id=user_id,
                asset="USDT",
                amount=f"{amount:.2f}",
                spend_id=f"withdraw_{user_id}_{datetime.now().timestamp()}"
            )

            # Проверяем результат перевода
            if transfer_result.status == 'completed':
                # Уменьшаем замороженный баланс пользователя
                user_data = get_user_data(user_id)
                update_user_data(user_id, frozen_balance=user_data["frozen_balance"] - amount)

                # Уведомляем пользователя о выплате
                await bot.send_message(
                    user_id,
                    f"✅ Ваша заявка на вывод средств выполнена!\n"
                    f"💸 Сумма: ${amount:.2f}\n"
                    f"📅 Дата: {request['date']}\n"
                    f"🔗 Транзакция: {transfer_result.transfer_id}",
                    parse_mode="Markdown"
                )

                # Уведомляем администратора об успешной выплате
                await call.message.edit_text(
                    f"✅ Выплата пользователю {user_id} на сумму ${amount:.2f} успешно выполнена."
                )
                await bot.send_message(
                    ADMIN_ID,
                    f"✅ Выплата пользователю {user_id} на сумму ${amount:.2f} успешно выполнена.\n"
                    f"🔗 Транзакция: {transfer_result.transfer_id}"
                )
            else:
                raise Exception(f"Transfer failed with status: {transfer_result.status}")

        except Exception as e:
            logging.error(f"Error processing transfer for user {user_id}: {e}")
            await call.message.edit_text(
                f"❌ Ошибка при обработке выплаты для пользователя {user_id}. Попробуйте снова позже."
            )
            # Возвращаем заявку в список в случае ошибки
            if user_id not in withdraw_requests:
                withdraw_requests[user_id] = []
            withdraw_requests[user_id].insert(idx, request)

@dp.callback_query_handler(lambda call: call.data.startswith("cancel_deposit_"))
async def cancel_deposit_callback(call: types.CallbackQuery):
    logging.info(f"Обработчик отмены сработал: {call.data}")
    user_id = int(call.data.split("_")[-1])

    if user_id not in active_invoices:  # Если нет активного инвойса
        await call.answer("❌ У вас нет активного пополнения.", show_alert=True)
        return

    # Удаляем активный инвойс
    del active_invoices[user_id]
    await call.message.edit_text("✅ Ваше активное пополнение отменено.")
    await call.answer("Пополнение отменено.")

@dp.message_handler(Text("💳 Управление балансом"))
async def manage_balance_handler(message: types.Message):
    if str(message.from_user.id) != ADMIN_ID:
        await message.answer("⛔ Доступ запрещён.")
        return
    await message.answer("Введите ID пользователя для управления балансом:")
    await AdminState.select_user.set()

@dp.message_handler(state=AdminState.select_user)
async def select_user_for_balance(message: types.Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("⚠️ Введите корректный ID пользователя.")
        return
    await state.update_data(user_id=int(message.text))
    await message.answer("Введите новую сумму баланса для пользователя:")
    await AdminState.set_balance.set()

@dp.message_handler(state=AdminState.set_balance)
async def set_user_balance(message: types.Message, state: FSMContext):
    if not message.text.replace('.', '', 1).isdigit():
        await message.answer("⚠️ Введите корректную сумму.")
        return
    new_balance = float(message.text)
    data = await state.get_data()
    user_id = data['user_id']

    update_user_data(user_id, balance=new_balance)
    await message.answer(f"✅ Баланс пользователя с ID {user_id} обновлён до ${new_balance:.2f}.")
    await state.finish()

@dp.message_handler(Text("📋 База пользователей"))
async def user_database_handler(message: types.Message):
    # Проверяем, что пользователь — администратор
    if str(message.from_user.id) != ADMIN_ID:
        await message.answer("⛔ Доступ запрещён.")
        return

    # Подключаемся к базе данных и считаем количество пользователей
    conn = sqlite3.connect("users.db", check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM users")
    user_count = cursor.fetchone()[0]
    conn.close()

    # Отправляем администратору количество пользователей
    await message.answer(f"📋 В базе зарегистрировано {user_count} пользователей.")


if __name__ == "__main__":
    executor.start_polling(dp, skip_updates=True)
