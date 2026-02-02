> Artem:
import os
import logging
import asyncio
from datetime import datetime
from pathlib import Path
from typing import Dict, Any

from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command, StateFilter
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.context import FSMContext
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import FSInputFile, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.enums import ParseMode

from converters.pdf_converter import convert_pdf_to_docx, convert_docx_to_pdf, split_pdf
from converters.image_converter import convert_image
from converters.audio_converter import convert_audio
from converters.video_converter import extract_audio_from_video
from utils.file_handler import save_file, cleanup_temp_files, get_file_size

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
)
logger = logging.getLogger(name)

TOKEN = ""
MAX_FILE_SIZE = 20 * 1024 * 1024  # 20MB для бесплатных пользователей
MAX_PREMIUM_SIZE = 100 * 1024 * 1024  # 100MB для премиум
ADMIN_ID = 000000000  # Ваш ID в Telegram

Path("temp_files").mkdir(exist_ok=True)
Path("converted").mkdir(exist_ok=True)

class ConverterStates(StatesGroup):
    registration_question = State()
    registration_email = State()
    awaiting_file = State()
    conversion_choice = State()
    premium_question = State()
    batch_mode = State()

users_db: Dict[int, Dict[str, Any]] = {}
conversion_history = []

bot = Bot(token=TOKEN)
dp = Dispatcher(storage=MemoryStorage())




@dp.message(Command("start"))
async def cmd_start(message: types.Message, state: FSMContext):
    user_id = message.from_user.id

    if user_id in users_db:
        await message.answer(
            f"С возвращением, {users_db[user_id].get('username', 'друг')}!\n\n"
            f"Email: {users_db[user_id].get('email', 'Не указан')}\n"
            f"Статус: {'Премиум' if users_db[user_id].get('premium', False) else 'Бесплатный'}\n\n"
            "Отправьте мне файл для конвертации!\n"
            "Доступные команды:\n"
            "/formats - Поддерживаемые форматы\n"
            "/history - История конвертаций\n"
            "/premium - Информация о премиуме\n"
            "/cancel - Отмена операции",
            parse_mode=ParseMode.HTML
        )
        await state.set_state(ConverterStates.awaiting_file)
    else:
        await message.answer(
            "<b>Конвертер файлов</b>\n\n"
            "У вас есть регистрация? (да/нет)\n\n"
            "<i>Для использования всех функций требуется регистрация</i>",
            parse_mode=ParseMode.HTML
        )
        await state.set_state(ConverterStates.registration_question)

@dp.message(ConverterStates.registration_question, F.text.lower().in_(["да", "нет"]))
async def handle_registration_question(message: types.Message, state: FSMContext):
    if message.text.lower() == "да":
        await message.answer("Введите ваш email для входа:")
        await state.set_state(ConverterStates.registration_email)
    else:
        await message.answer(
            "Для начала работы необходимо зарегистрироваться.\n"
            "Отправьте 'да' для регистрации или email для входа."
        )

@dp.message(ConverterStates.registration_email)
async def handle_registration_email(message: types.Message, state: FSMContext):

> Artem:
email = message.text.strip()
    user_id = message.from_user.id

    if "@" not in email or "." not in email:
        await message.answer("Неверный формат email. Попробуйте снова:")
        return

    users_db[user_id] = {
        "user_id": user_id,
        "username": message.from_user.full_name,
        "email": email,
        "premium": False,
        "registration_date": datetime.now().isoformat(),
        "conversions_count": 0
    }

    await message.answer(
        f"<b>Регистрация успешна!</b>\n"
        f"Email: {email}\n"
        f"Имя: {message.from_user.full_name}\n\n"
        f"Теперь вы можете отправлять мне файлы для конвертации!\n\n"
        f"<i>Доступные форматы: PDF, JPG, PNG, GIF, DOCX, MP3, MP4</i>",
        parse_mode=ParseMode.HTML
    )
    await state.set_state(ConverterStates.awaiting_file)

@dp.message(ConverterStates.awaiting_file, F.document | F.photo | F.video | F.audio)
async def handle_file_upload(message: types.Message, state: FSMContext):
    user_id = message.from_user.id
    user_data = users_db.get(user_id, {})
    is_premium = user_data.get("premium", False)

    max_size = MAX_PREMIUM_SIZE if is_premium else MAX_FILE_SIZE

    file_info = None
    original_filename = ""

    if message.document:
        file_info = message.document
        original_filename = file_info.file_name
    elif message.photo:
        file_info = message.photo[-1]  # Берем фото самого высокого качества
        original_filename = f"photo_{message.photo[-1].file_id}.jpg"
    elif message.video:
        file_info = message.video
        original_filename = file_info.file_name or f"video_{file_info.file_id}.mp4"
    elif message.audio:
        file_info = message.audio
        original_filename = file_info.file_name or f"audio_{file_info.file_id}.mp3"

    if not file_info:
        await message.answer("Не удалось получить информацию о файле.")
        return

    file_size = file_info.file_size
    if file_size > max_size:
        size_mb = max_size / (1024 * 1024)
        await message.answer(
            f"Файл слишком большой!\n"
            f"Лимит: {size_mb}MB\n"
            f"Ваш файл: {file_size / (1024 * 1024):.1f}MB\n\n"
            f"Обновитесь до <b>премиум</b> для загрузки до 100MB",
            parse_mode=ParseMode.HTML
        )
        return

    try:
        file_path = await save_file(bot, file_info, original_filename)
        await state.update_data({
            "file_path": file_path,
            "original_filename": original_filename,
            "file_type": file_info.mime_type if hasattr(file_info, 'mime_type') else None
        })

        keyboard = InlineKeyboardMarkup(inline_keyboard=[
            [
                InlineKeyboardButton(text=" PDF → DOCX", callback_data="pdf_docx"),
                InlineKeyboardButton(text=" JPG → PNG", callback_data="jpg_png"),
            ],
            [
                InlineKeyboardButton(text=" MP4 → MP3", callback_data="mp4_mp3"),
                InlineKeyboardButton(text=" DOCX → PDF", callback_data="docx_pdf"),
            ],
            [
                InlineKeyboardButton(text=" PNG → JPG", callback_data="png_jpg"),
                InlineKeyboardButton(text=" Видео → GIF", callback_data="video_gif"),
            ],
            [
                InlineKeyboardButton(text=" Другие форматы", callback_data="more_formats"),
                InlineKeyboardButton(text=" Премиум", callback_data="premium_info"),
            ]
        ])

> Artem:
file_size_mb = file_size / (1024 * 1024)
        await message.answer(
            f" Файл получен: <b>{original_filename}</b>\n"
            f" Размер: {file_size_mb:.1f} MB\n\n"
            f" <b>Выберите формат конвертации:</b>",
            reply_markup=keyboard,
            parse_mode=ParseMode.HTML
        )
        await state.set_state(ConverterStates.conversion_choice)

    except Exception as e:
        logger.error(f"Ошибка сохранения файла: {e}")
        await message.answer(" Ошибка при сохранении файла. Попробуйте еще раз.")

@dp.callback_query(ConverterStates.conversion_choice)
async def handle_conversion_choice(callback: types.CallbackQuery, state: FSMContext):
    user_id = callback.from_user.id
    data = await state.get_data()
    file_path = data.get("file_path")
    original_filename = data.get("original_filename")

    conversion_history.append({
        "user_id": user_id,
        "filename": original_filename,
        "conversion_type": callback.data,
        "timestamp": datetime.now().isoformat()
    })

    if user_id in users_db:
        users_db[user_id]["conversions_count"] = users_db[user_id].get("conversions_count", 0) + 1

    await callback.answer()

    await callback.message.edit_text(" Конвертация... Пожалуйста, подождите.")

    try:
        converted_path = None
        result_filename = ""

        if callback.data == "pdf_docx":
            converted_path = await convert_pdf_to_docx(file_path)
            result_filename = original_filename.replace('.pdf', '.docx')
        elif callback.data == "jpg_png":
            converted_path = await convert_image(file_path, 'png')
            result_filename = original_filename.replace('.jpg', '.png').replace('.jpeg', '.png')
        elif callback.data == "png_jpg":
            converted_path = await convert_image(file_path, 'jpg')
            result_filename = original_filename.replace('.png', '.jpg')
        elif callback.data == "mp4_mp3":
            converted_path = await extract_audio_from_video(file_path)
            result_filename = original_filename.replace('.mp4', '.mp3')
        elif callback.data == "docx_pdf":
            converted_path = await convert_docx_to_pdf(file_path)
            result_filename = original_filename.replace('.docx', '.pdf')
        elif callback.data == "video_gif":
            converted_path = await convert_video_to_gif(file_path)
            result_filename = original_filename.replace('.mp4', '.gif')
        elif callback.data == "more_formats":
            await show_more_formats(callback.message, state)
            return
        elif callback.data == "premium_info":
            await show_premium_info(callback.message)
            return

        if converted_path and os.path.exists(converted_path):
            # Отправляем конвертированный файл
            file_to_send = FSInputFile(converted_path, filename=result_filename)

            await callback.message.answer_document(
                document=file_to_send,
                caption=f"<b>Конвертация завершена!</b>\n"
                       f"Файл: {result_filename}\n"
                       f"Тип: {callback.data}\n\n"
                       f"Отправьте новый файл для конвертации",
                parse_mode=ParseMode.HTML
            )

            # Очищаем временные файлы
            await cleanup_temp_files([file_path, converted_path])

            await state.set_state(ConverterStates.awaiting_file)
        else:
            await callback.message.answer("Ошибка при конвертации файла. Попробуйте другой формат.")

> Artem:
await state.set_state(ConverterStates.awaiting_file)

    except Exception as e:
        logger.error(f"Ошибка конвертации: {e}")
        await callback.message.answer("Произошла ошибка при конвертации. Попробуйте еще раз.")
        await state.set_state(ConverterStates.awaiting_file)

@dp.message(Command("formats"))
async def cmd_formats(message: types.Message):
    formats_text = """
 <b>Поддерживаемые форматы конвертации:</b>

<b> Документы:</b>
• PDF → DOCX
• DOCX → PDF
• PDF → TXT
• Разделение PDF

<b> Изображения:</b>
• JPG → PNG
• PNG → JPG
• WEBP → PNG/JPG
• Сжатие изображений
• Изменение размера

<b> Аудио:</b>
• MP4 → MP3
• WAV → MP3
• Извлечение аудио из видео

<b> Видео:</b>
• Видео → GIF
• Изменение формата видео
• Обрезка видео

<b> Премиум функции:</b>
• Пакетная конвертация
• Файлы до 100MB
• Приоритетная обработка
• Конвертация архивов
"""
    await message.answer(formats_text, parse_mode=ParseMode.HTML)

@dp.message(Command("premium"))
async def cmd_premium(message: types.Message):
    await show_premium_info(message)

async def show_premium_info(message: types.Message):
    premium_text = """
 <b>Премиум подписка</b>

<b>Что включено:</b>
 Файлы до 100MB (вместо 20MB)
 Пакетная конвертация
 Приоритетная обработка
 Конвертация архивов
 Поддержка редких форматов
 Без водяных знаков

<b>Цены:</b>
• 100₽/месяц
• 1000₽/6 месяцев
• 2000₽/год

Для покупки отправьте "куплить" или нажмите кнопку ниже.
    """

    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text=" Купить премиум", callback_data="buy_premium")],
        [InlineKeyboardButton(text=" Примеры работы", callback_data="premium_examples")]
    ])

    await message.answer(premium_text, reply_markup=keyboard, parse_mode=ParseMode.HTML)

@dp.message(Command("history"))
async def cmd_history(message: types.Message):
    user_id = message.from_user.id

    user_history = [h for h in conversion_history if h.get("user_id") == user_id]

    if not user_history:
        await message.answer(" У вас еще не было конвертаций.")
        return

    history_text = " <b>История конвертаций:</b>\n\n"
    for i, item in enumerate(user_history[-10:], 1):  # Последние 10 записей
        timestamp = datetime.fromisoformat(item["timestamp"]).strftime("%d.%m.%Y %H:%M")
        history_text += f"{i}. {item['filename']} → {item['conversion_type']}\n    {timestamp}\n\n"

    await message.answer(history_text, parse_mode=ParseMode.HTML)

@dp.message(Command("cancel"))
async def cmd_cancel(message: types.Message, state: FSMContext):
    current_state = await state.get_state()
    if current_state is None:
        await message.answer("Нет активных операций для отмены.")
        return

    await message.answer("Операция отменена. Вы в главном меню.")
    await state.clear()
    await state.set_state(ConverterStates.awaiting_file)

@dp.message(F.text.lower().in_(["куптить", "куплю", "купить", "премиум"]))
@dp.callback_query(F.data == "buy_premium")
async def handle_premium_purchase(update: types.Message | types.CallbackQuery, state: FSMContext):
    if isinstance(update, types.CallbackQuery):
        message = update.message
        await update.answer()
    else:
        message = update

    user_id = message.from_user.id

    if user_id in users_db:
        users_db[user_id]["premium"] = True
        users_db[user_id]["premium_since"] = datetime.now().isoformat()

        await message.answer(
            "<b>Поздравляем с приобретением премиум подписки!</b>\n\n"

> Artem:
"Ваш статус обновлен\n"
            "Доступны файлы до 100MB\n"
            "Включена пакетная конвертация\n"
            "Приоритетная обработка\n\n"
            "Спасибо за поддержку!",
            parse_mode=ParseMode.HTML
        )
    else:
        await message.answer("Сначала зарегистрируйтесь с помощью /start")

@dp.message(F.text.lower() == "да")
async def handle_yes(message: types.Message, state: FSMContext):
    current_state = await state.get_state()

    if current_state == ConverterStates.registration_question.state:
        await message.answer("Введите ваш email:")
        await state.set_state(ConverterStates.registration_email)
    elif current_state == ConverterStates.conversion_choice.state:
        await message.answer("Выберите формат конвертации из меню выше")
    else:
        await message.answer("Отправьте файл для конвертации или используйте команды:\n/formats, /history, /premium")

@dp.message(F.text.lower() == "нет")
async def handle_no(message: types.Message, state: FSMContext):
    current_state = await state.get_state()

    if current_state == ConverterStates.registration_question.state:
        await message.answer("Для начала работы зарегистрируйтесь. Отправьте 'да' для регистрации.")
    else:
        await message.answer("Операция отменена. Отправьте файл для конвертации.")

@dp.message(F.text.lower() == "регистрация")
async def handle_registration_command(message: types.Message, state: FSMContext):
    user_id = message.from_user.id

    if user_id in users_db:
        await message.answer(f"Вы уже зарегистрированы как {users_db[user_id]['email']}")
    else:
        await message.answer("Для регистрации отправьте 'да' на вопрос о регистрации или введите email")
        await state.set_state(ConverterStates.registration_question)

async def show_more_formats(message: types.Message, state: FSMContext):
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(text="PDF → TXT", callback_data="pdf_txt"),
            InlineKeyboardButton(text="Сжать PDF", callback_data="compress_pdf"),
        ],
        [
            InlineKeyboardButton(text="Разделить PDF", callback_data="split_pdf"),
            InlineKeyboardButton(text="WEBP → PNG", callback_data="webp_png"),
        ],
        [
            InlineKeyboardButton(text="Сжать изображение", callback_data="compress_img"),
            InlineKeyboardButton(text="Изменить размер", callback_data="resize_img"),
        ],
        [
            InlineKeyboardButton(text="Назад", callback_data="back_to_main"),
        ]
    ])

    await message.answer("<b>Дополнительные форматы:</b>", reply_markup=keyboard, parse_mode=ParseMode.HTML)

@dp.message()
async def handle_unknown(message: types.Message, state: FSMContext):
    current_state = await state.get_state()

    if current_state == ConverterStates.registration_question.state:
        await message.answer("Пожалуйста, ответьте 'да' или 'нет' на вопрос о регистрации.")
    elif current_state == ConverterStates.registration_email.state:
        await message.answer("Пожалуйста, введите корректный email адрес.")
    else:
        await message.answer(
            "Не понял ваше сообщение.\n\n"
            "Доступные команды:\n"
            "/start - Начать работу\n"
            "/formats - Поддерживаемые форматы\n"
            "/history - История конвертаций\n"
            "/cancel - Отмена операции\n\n"
            "Или просто отправьте мне файл для конвертации!"
        )

async def main():
    logger.info("Бот-конвертер запускается...")
    await dp.start_polling(bot)

if name == "main":
    asyncio.run(main())
