import os
import json
import datetime
import asyncio
import logging
import re
from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.types import (
    ReplyKeyboardMarkup, KeyboardButton,
    InlineKeyboardMarkup, InlineKeyboardButton,
    Message
)
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.lib.units import cm
from reportlab.lib.utils import ImageReader

# ---------------- НАСТРОЙКИ ----------------
BOT_TOKEN = os.getenv("BOT_TOKEN")  # токен бота
ADMIN_USERNAME = "krulovdima"       # ник админа без @
PAYMENT_URL = "https://tbank.ru/cf/3i7dTLc1FEP"  # ссылка на оплату
LOGO_PATH = "logo.jpg"  # логотип для сертификата

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()

# ---------------- БАЗА ДАННЫХ ----------------
users_file = "users.json"

def load_users():
    try:
        with open(users_file, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return {}

def save_users(users):
    with open(users_file, "w", encoding="utf-8") as f:
        json.dump(users, f, indent=4, ensure_ascii=False)

users = load_users()
user_states = {}

# ---------------- УРОКИ И ДЗ ----------------
lessons = {
    1: "🎓 Урок 1: Введение в 3D моделирование. Ссылка на видео: ...",
    2: "🎓 Урок 2: Основы работы в Blender. Ссылка на видео: ...",
    3: "🎓 Урок 3: Моделирование простых объектов. Ссылка на видео: ..."
}
homeworks = {
    1: "📝 Домашнее задание 1: Создайте куб и примените материал. Отправьте рендер + .blend файл.",
    2: "📝 Домашнее задание 2: Создайте простую сцену с освещением. Отправьте рендер + .blend файл.",
    3: "📝 Домашнее задание 3: Смоделируйте стул. Отправьте рендер + .blend файл."
}

# ---------------- ВСПОМОГАТЕЛЬНЫЕ ----------------
def is_admin(message: Message) -> bool:
    return message.from_user and message.from_user.username and \
           message.from_user.username.lower() == ADMIN_USERNAME.lower()

def validate_phone_number(phone: str) -> bool:
    phone_clean = re.sub(r'[^\d+]', '', phone)
    return bool(re.match(r'^(\+7|8)\d{10}$', phone_clean))

def generate_certificate(user_name: str, file_path: str):
    c = canvas.Canvas(file_path, pagesize=A4)
    width, height = A4

    c.setStrokeColorRGB(0.2, 0.2, 0.2)
    c.setLineWidth(4)
    c.rect(1*cm, 1*cm, width-2*cm, height-2*cm)

    try:
        logo = ImageReader(LOGO_PATH)
        c.drawImage(logo, width/2-3*cm, height-6*cm, width=6*cm, height=6*cm, mask='auto')
    except:
        pass

    c.setFont("Helvetica-Bold", 28)
    c.drawCentredString(width/2, height-7.5*cm, "СЕРТИФИКАТ")

    c.setFont("Helvetica", 16)
    c.drawCentredString(width/2, height-9*cm, "о прохождении курса")

    c.setFont("Helvetica-Bold", 24)
    c.drawCentredString(width/2, height-11*cm, user_name)

    c.setFont("Helvetica", 14)
    c.drawCentredString(width/2, height-12.5*cm, "«3D Моделирование для начинающих»")

    date_str = datetime.datetime.now().strftime("%d.%m.%Y")
    c.setFont("Helvetica-Oblique", 12)
    c.drawCentredString(width/2, height-14*cm, f"Дата выдачи: {date_str}")

    c.setFont("Helvetica", 12)
    c.drawString(2*cm, 2.5*cm, "_________________")
    c.drawString(2*cm, 2*cm, "Руководитель курса")

    c.save()

# ---------------- КНОПКИ ----------------
def get_start_keyboard():
    kb = [[KeyboardButton(text="Записаться на курс")]]
    return ReplyKeyboardMarkup(keyboard=kb, resize_keyboard=True)

def get_payment_keyboard():
    kb = [[KeyboardButton(text="Оплата")]]
    return ReplyKeyboardMarkup(keyboard=kb, resize_keyboard=True)

def get_progress_keyboard():
    kb = [[KeyboardButton(text="📊 Прогресс обучения")]]
    return ReplyKeyboardMarkup(keyboard=kb, resize_keyboard=True)

# ---------------- ОБРАБОТЧИКИ ----------------
@dp.message(Command("start"))
async def start(message: Message):
    await message.answer("Привет! Я помогу тебе пройти курс по 3D моделированию.",
                         reply_markup=get_start_keyboard())

# регистрация
@dp.message(F.text == "Записаться на курс")
async def register(message: Message):
    user_states[message.from_user.id] = "waiting_name"
    await message.answer("📝 Введите ваше полное имя:")

@dp.message(F.text)
async def handle_text_messages(message: Message):
    user_id = message.from_user.id
    if user_id in user_states:
        if user_states[user_id] == "waiting_name":
            users[str(user_id)] = {"name": message.text}
            user_states[user_id] = "waiting_phone"
            await message.answer("📞 Введите телефон (+7XXXXXXXXXX или 8XXXXXXXXXX):")

        elif user_states[user_id] == "waiting_phone":
            phone = message.text.strip()
            if not validate_phone_number(phone):
                await message.answer("❌ Неверный формат. Попробуйте ещё раз.")
                return
            users[str(user_id)]["phone"] = phone
            user_states[user_id] = "waiting_email"
            await message.answer("📧 Введите email:")

        elif user_states[user_id] == "waiting_email":
            email = message.text.strip().lower()
            users[str(user_id)]["email"] = email
            users[str(user_id)]["paid"] = False
            users[str(user_id)]["homework_status"] = {}
            save_users(users)
            del user_states[user_id]
            await message.answer("🎉 Регистрация завершена! Теперь оплатите курс:",
                                 reply_markup=get_payment_keyboard())

# оплата
@dp.message(F.text == "Оплата")
async def payment(message: Message):
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💳 Оплатить курс", url=PAYMENT_URL)]
    ])
    await message.answer("Перейдите по ссылке для оплаты:", reply_markup=kb)

# прогресс
@dp.message(F.text == "📊 Прогресс обучения")
async def progress(message: Message):
    user_id = str(message.from_user.id)
    if user_id not in users or not users[user_id].get("paid", False):
        await message.answer("❌ Сначала оплатите курс!")
        return

    user = users[user_id]
    completed = sum(1 for k,v in user["homework_status"].items() if v=="approved")
    total = len(lessons)
    status = f"📚 Пройдено уроков: {completed}/{total}\n"

    pending = [k for k,v in user["homework_status"].items() if v=="submitted"]
    if pending:
        status += f"🏠 Ожидает проверки: {len(pending)}\n"

    if completed == total:
        cert_file = f"certificate_{user_id}.pdf"
        generate_certificate(user["name"], cert_file)
        await bot.send_document(message.chat.id, types.FSInputFile(cert_file),
                                caption="🏆 Ваш сертификат о прохождении курса!")
    else:
        status += "➡️ Для получения сертификата завершите все уроки!"

    await message.answer(status)

# ---------------- ДОМАШКА ----------------
@dp.callback_query(F.data.startswith("homework_"))
async def send_homework(callback: types.CallbackQuery):
    user_id = str(callback.from_user.id)
    if user_id not in users or not users[user_id].get("paid", False):
        return await callback.answer("❌ Сначала оплатите курс!", show_alert=True)

    lesson_num = int(callback.data.split("_")[1])
    hw_text = homeworks.get(lesson_num, "Домашнее задание недоступно.")

    users[user_id]["homework_status"][lesson_num] = "waiting_upload"
    save_users(users)

    await bot.send_message(callback.from_user.id, hw_text)
    await bot.send_message(callback.from_user.id, "📤 Отправьте файлы с домашним заданием одним сообщением.")

@dp.message(F.document | F.photo | F.text)
async def handle_homework_submission(message: Message):
    user_id = str(message.from_user.id)
    if user_id not in users:
        return

    pending = [k for k,v in users[user_id]["homework_status"].items() if v=="waiting_upload"]
    if not pending:
        return

    lesson_num = pending[0]
    users[user_id]["homework_status"][lesson_num] = "submitted"
    save_users(users)

    await message.answer(f"✅ Домашнее задание {lesson_num} отправлено на проверку.")
    await bot.send_message(message.chat.id, "Ожидайте проверки админом.")

# ---------------- АДМИН ----------------
@dp.message(Command("confirm_payment"))
async def confirm_payment(message: Message):
    if not is_admin(message):
        return await message.answer("❌ Нет прав.")

    parts = message.text.split()
    if len(parts) != 2:
        return await message.answer("Используйте: /confirm_payment USER_ID")

    uid = parts[1]
    if uid not in users:
        return await message.answer("❌ Пользователь не найден.")

    users[uid]["paid"] = True
    save_users(users)

    await message.answer(f"✅ Оплата подтверждена для {users[uid]['name']}")
    await bot.send_message(int(uid), "🎉 Ваша оплата подтверждена! Вот первый урок 👇",
                           reply_markup=get_progress_keyboard())

    if 1 in lessons:
        await bot.send_message(int(uid), lessons[1])
        kb = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="📝 Получить домашнее задание 1", callback_data="homework_1")]
        ])
        await bot.send_message(int(uid), "Когда будете готовы — нажмите, чтобы получить ДЗ:", reply_markup=kb)

@dp.message(Command("approve_homework"))
async def approve_homework(message: Message):
    if not is_admin(message):
        return await message.answer("❌ Нет прав.")

    parts = message.text.split()
    if len(parts) != 3:
        return await message.answer("Используйте: /approve_homework USER_ID LESSON_NUM")

    uid, lesson_str = parts[1], parts[2]
    if uid not in users:
        return await message.answer("❌ Пользователь не найден.")

    try:
        lesson_num = int(lesson_str)
    except:
        return await message.answer("❌ Урок должен быть числом.")

    users[uid]["homework_status"][lesson_num] = "approved"
    save_users(users)

    await message.answer(f"✅ ДЗ {lesson_num} у {users[uid]['name']} принято.")

    next_lesson = lesson_num + 1
    if next_lesson in lessons:
        await bot.send_message(int(uid), f"🎉 Домашка {lesson_num} принята! Вот следующий урок:")
        await bot.send_message(int(uid), lessons[next_lesson])
        kb = InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text=f"📝 Получить домашнее задание {next_lesson}", callback_data=f"homework_{next_lesson}")]
        ])
        await bot.send_message(int(uid), "Когда будете готовы — нажмите, чтобы получить ДЗ:", reply_markup=kb)
    else:
        await bot.send_message(int(uid), "🏆 Поздравляем! Вы прошли все уроки. Заберите свой сертификат через «📊 Прогресс обучения».")

# ---------------- ЗАПУСК ----------------
async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())