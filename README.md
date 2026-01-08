# TG06. Разработка чат-бота

## Результат дня
Закрепим материалы по Telegram-ботам на примере кейса «Финансовый бот-помощник»

## Теория на сегодня:

### 1. Подготовка

Сегодня мы рассмотрим разработку чат-бота на конкретном кейсе.

**План урока:**
- разработаем кейс "Финансовый бот-помощник";
- закрепим всё, что изучили в модуле по Telegram-ботам.

Открываем PyCharm.
Переносим всё, что может пригодиться, из предыдущих уроков.

```python
import asyncio
from aiogram import Bot, Dispatcher, F
from aiogram.filters import CommandStart, Command
from aiogram.types import Message, FSInputFile
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage

from config import TOKEN, WEATHER_API_KEY
import sqlite3
import aiohttp
import logging

bot = Bot(token=TOKEN)
dp = Dispatcher()

logging.basicConfig(level=logging.INFO)

async def main():
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())
```

Удаляем WEATHER_API_KEY, потому что здесь мы работать с погодой не будем.

```python
import asyncio
from aiogram import Bot, Dispatcher, F
from aiogram.filters import CommandStart, Command
from aiogram.types import Message, FSInputFile
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage

from config import TOKEN
import sqlite3
import aiohttp
import logging

bot = Bot(token=TOKEN)
dp = Dispatcher()

logging.basicConfig(level=logging.INFO)

async def main():
    await dp.start_polling(bot)

if __name__ == '__main__':
    asyncio.run(main())
```

Далее нам нужно определить функционал нашего бота. Бот будет иметь несколько возможностей:
- регистрация пользователя в Telegram;
- просмотр курса валют с помощью API или парсинга;
- получение советов по экономии в виде текста;
- ведение учёта личных финансов по трём категориям.

Продолжаем работу над ботом:
Импортируем библиотеки для работы с клавиатурами.

```python
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils.keyboard import ReplyKeyboardBuilder, InlineKeyboardBuilder
```

Создаём кнопку регистрации.

```python
button_registr = KeyboardButton(text="Регистрация в телеграм боте")
```

Создаём кнопку для выдачи курса валют.

```python
button_exchange_rates = KeyboardButton(text="Курс валют")
```

Создаём кнопку для выдачи курса валют.

```python
button_tips = KeyboardButton(text="Советы по экономии")
```

Создаём кнопку для учёта расходов.

```python
button_finances = KeyboardButton(text="Личные финансы")
```

Для работы кнопок нужно создавать клавиатуру. Клавиатура будет обычная, которая находится снизу. Создаём переменную с помощью класса ReplyKeyboardMarkup. В круглых скобках указываем список, внутри которого будут находиться другие списки. Таким образом настраиваем размещение кнопок.

```python
keyboard = ReplyKeyboardMarkup(keyboard=[
    [button_registr, button_exchange_rates],
    [button_tips, button_finances]
])
```

Кнопки выходят крупными, поэтому настроим изменение размера клавиатуры.

```python
keyboard = ReplyKeyboardMarkup(keyboard=[
    [button_registr, button_exchange_rates],
    [button_tips, button_finances]
], resize_keyboard=True)
```

Чтобы сохранять данные о пользователях, нам нужно создать базу данных. Сделаем мы это м помощью SQLite в этом же файле. Создаём подключение, курсор.

```python
conn = sqlite3.connect('user.db')
cursor = conn.cursor()
```

Выполняем действие — создаём таблицу. Указываем поля таблицы. Для ID пользователя указываем UNIQUE, потому что идентификатор пользователя не может быть неуникальным. Создаём поля для категорий (в TEXT будут названия категорий) и для расходов по этим категориям (REAL — это дробный тип данных).

```python
conn = sqlite3.connect('user.db')
cursor = conn.cursor()

cursor.execute('''
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    telegram_id INTEGER UNIQUE,
    name TEXT,
    category1 TEXT,
    category2 TEXT,
    category3 TEXT,
    expenses1 REAL,
    expenses2 REAL,
    expenses3 REAL
    )
''')
```

Прописываем сохранение после выполнения этого действия.

```python
conn.commit()
```

Чтобы запрашивать информацию и ждать ответа, нужно использовать состояния. Создаём класс, в котором будут прописаны эти состояния для каждой категории и каждого значения категории.

```python
class FinancesForm(StatesGroup):
    category1 = State()
    expenses1 = State()
    category2 = State()
    expenses2 = State()
    category3 = State()
    expenses3 = State()
```

Теперь мы подготовились к написанию функций бота.

### 2. Функции бота

Прописываем функцию /start, которая базово есть у всех ботов. Прописываем асинхронную функцию и стартовое сообщение. После этого сообщения должна отправляться клавиатура, поэтому после сообщения через запятую указываем переменную клавиатуру.

```python
@dp.message(Command('start'))
async def send_start(message: Message):
    await message.answer("Привет! Я ваш личный финансовый помощник. Выберите одну из опций в меню:", reply_markup=keyboard)
```

Запускаем программу и проверяем работу бота. Кнопки сейчас ничего не делают.

Прописываем декоратор для регистрации в боте. В кавычках указываем текст, который будет вызывать работу этой функции.

```python
@dp.message(F.text == "Регистрация в телеграм боте")
```

Указываем саму асинхронную функцию. При получении сообщения мы можем получить информацию — прописываем сохранение этой информации в переменных: ID пользователя и имя.

```python
@dp.message(F.text == "Регистрация в телеграм боте")
async def registration(message: Message):
    telegram_id = message.from_user.id
    name = message.from_user.full_name
```

Проверяем существование пользователя с помощью действия с курсором. (telegram_id,) — это название столбца, в котором нужно искать.

```python
@dp.message(F.text == "Регистрация в телеграм боте")
async def registration(message: Message):
    telegram_id = message.from_user.id
    name = message.from_user.full_name
    cursor.execute('''SELECT * FROM users WHERE telegram_id = ?''', (telegram_id,))
```

Берём выданную информацию. Переменная user = cursor.fetchone() будет брать первый попавшийся результат, но он и будет один.

```python
@dp.message(F.text == "Регистрация в телеграм боте")
async def registration(message: Message):
    telegram_id = message.from_user.id
    name = message.from_user.full_name
    cursor.execute('''SELECT * FROM users WHERE telegram_id = ?''', (telegram_id,))
    user = cursor.fetchone()
```

Проверяем, существует ли юзер. Указываем, что должно происходить, если юзер существует или не существует.

```python
@dp.message(F.text == "Регистрация в телеграм боте")
async def registration(message: Message):
    telegram_id = message.from_user.id
    name = message.from_user.full_name
    cursor.execute('''SELECT * FROM users WHERE telegram_id = ?''', (telegram_id,))
    user = cursor.fetchone()
    if user:
        await message.answer("Вы уже зарегистрированы!")
    else:
        cursor.execute('''INSERT INTO users (telegram_id, name) VALUES (?, ?)''', (telegram_id, name))
```

Сохраняем изменения и отправляем сообщение об успешной регистрации.

```python
@dp.message(F.text == "Регистрация в телеграм боте")
async def registration(message: Message):
    telegram_id = message.from_user.id
    name = message.from_user.full_name
    cursor.execute('''SELECT * FROM users WHERE telegram_id = ?''', (telegram_id,))
    user = cursor.fetchone()
    if user:
        await message.answer("Вы уже зарегистрированы!")
    else:
        cursor.execute('''INSERT INTO users (telegram_id, name) VALUES (?, ?)''', (telegram_id, name))
        conn.commit()
        await message.answer("Вы успешно зарегистрированы!")
```

Создаём функцию, которая будет выдавать курс валют. Создаём декоратор и асинхронную функцию.

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
```

Для выдачи курса валют мы будем запрашивать информацию без API, с помощью URL-ссылки. Чтобы получить ссылку:
1. Переходим на сайт app.exchangerate-api.com/sign-up
2. Вводим адрес электронной почты, придумываем пароль. Бесплатный период — две недели.
3. Нажимаем на кнопку Accept Terms & Create API Key!.
4. Если потребуется, проходим капчу.
5. Переходим по ссылке из письма и копируем ссылку Example Request.

Вставляем ссылку в переменную url. Используем конструкцию TRY—EXCEPT, чтобы избежать ошибок.

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
    url = "https://v6.exchangerate-api.com/v6/YOUR-API-KEY/latest/USD"
    try:
```

Импортируем библиотеку request

```python
from config import TOKEN
import sqlite3
import aiohttp
import logging
import requests
```

Отправляем GET-запрос по адресу URL.

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
    url = "https://v6.exchangerate-api.com/v6/YOUR-API-KEY/latest/USD"
    try:
        response = requests.get(url)
```

Создаём переменную для сохранения ответа. Прописываем проверку (если статус-код не равен 200, то есть не успешен).

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
    url = "https://v6.exchangerate-api.com/v6/YOUR-API-KEY/latest/USD"
    try:
        response = requests.get(url)
        data = response.json()
        if response.status_code != 200:
            await message.answer("Не удалось получить данные о курсе валют!")
            return
```

Создаём переменную для перевода долларов по отношению к рублям и евро по отношению к доллару.

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
    url = "https://v6.exchangerate-api.com/v6/YOUR-API-KEY/latest/USD"
    try:
        response = requests.get(url)
        data = response.json()
        if response.status_code != 200:
            await message.answer("Не удалось получить данные о курсе валют!")
            return
        usd_to_rub = data['conversion_rates']['RUB']
        eur_to_usd = data['conversion_rates']['EUR']

        euro_to_rub = eur_to_usd * usd_to_rub

        await message.answer(f"1 USD - {usd_to_rub:.2f}  RUB\n"
                             f"1 EUR - {euro_to_rub:.2f}  RUB")
```

Используем математику, чтобы переводить евро в рубли.

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
    url = "https://v6.exchangerate-api.com/v6/YOUR-API-KEY/latest/USD"
    try:
        response = requests.get(url)
        data = response.json()
        if response.status_code != 200:
            await message.answer("Не удалось получить данные о курсе валют!")
            return
        usd_to_rub = data['conversion_rates']['RUB']
        eur_to_usd = data['conversion_rates']['EUR']

        euro_to_rub = eur_to_usd * usd_to_rub
```

Отправляем сообщение с курсом валют. ".2f" означает, что мы форматируем строку так, чтобы число было с двумя знаками после запятой.

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
    url = "https://v6.exchangerate-api.com/v6/YOUR-API-KEY/latest/USD"
    try:
        response = requests.get(url)
        data = response.json()
        if response.status_code != 200:
            await message.answer("Не удалось получить данные о курсе валют!")
            return
        usd_to_rub = data['conversion_rates']['RUB']
        eur_to_usd = data['conversion_rates']['EUR']

        euro_to_rub = eur_to_usd * usd_to_rub

        await message.answer(f"1 USD - {usd_to_rub:.2f}  RUB\n"
                             f"1 EUR - {euro_to_rub:.2f}  RUB")
```

В случае ошибки отправляем сообщение.

```python
@dp.message(F.text == "Курс валют")
async def exchange_rates(message: Message):
    url = "https://v6.exchangerate-api.com/v6/YOUR-API-KEY/latest/USD"
    try:
        response = requests.get(url)
        data = response.json()
        if response.status_code != 200:
            await message.answer("Не удалось получить данные о курсе валют!")
            return
        usd_to_rub = data['conversion_rates']['RUB']
        eur_to_usd = data['conversion_rates']['EUR']

        euro_to_rub = eur_to_usd * usd_to_rub

        await message.answer(f"1 USD - {usd_to_rub:.2f}  RUB\n"
                             f"1 EUR - {euro_to_rub:.2f}  RUB")
                            
    except:
        await message.answer("Произошла ошибка")
```

Запускаем программу и проверяем работу бота.

### 3. Советы по экономии

Чтобы получить советы, генерируем их в ChatGPT.
Создаём асинхронную функцию для отправки текста.

```python
@dp.message(F.text == "Советы по экономии")
async def send_tips(message: Message):
```

Создаём список с советами.

```python
@dp.message(F.text == "Советы по экономии")
async def send_tips(message: Message):
    tips = [
        "Совет 1: Ведите бюджет и следите за своими расходами.",
        "Совет 2: Откладывайте часть доходов на сбережения.",
        "Совет 3: Покупайте товары по скидкам и распродажам."
    ]
```

Настраиваем рандомную выдачу советов с помощью переменной. Отправляем совет из переменной.

```python
@dp.message(F.text == "Советы по экономии")
async def send_tips(message: Message):
    tips = [
        "Совет 1: Ведите бюджет и следите за своими расходами.",
        "Совет 2: Откладывайте часть доходов на сбережения.",
        "Совет 3: Покупайте товары по скидкам и распродажам."
    ]
    tip = random.choice(tips)
    await message.answer(tip)
```

Работаем с личными финансами.
Создаём асинхронную функцию для работы с личными финансами.

```python
@dp.message(F.text == "Личные финансы")
async def finances(message: Message): 
```

Начинаем работу с состояниями. Вводим второй атрибут функции.

```python
@dp.message(F.text == "Личные финансы")
async def finances(message: Message, state: FSMContext):
```

Устанавливаем новое состояние. В круглых скобках указываем класс и категорию этого состояния.

```python
@dp.message(F.text == "Личные финансы")
async def finances(message: Message, state: FSMContext):
    await state.set_state(FinancesForm.category1)
```

Отправляем сообщение пользователю.

```python
@dp.message(F.text == "Личные финансы")
async def finances(message: Message, state: FSMContext):
    await state.set_state(FinancesForm.category1)
    await message.reply("Введите первую категорию расходов:") 
```

Создаём декоратор, который сработает не по фразе, а по категории.

```python
@dp.message(FinancesForm.category1)
```

Настраиваем обновление данных. Теперь в category1 будет сохраняться текст сообщения.

```python
@dp.message(FinancesForm.category1)
async def finances(message: Message, state: FSMContext):
    await state.update_data(category1 = message.text)
```

Начинаем использовать новое состояние. Теперь нужно значение денег, которые уходят на эту категорию товаров.

```python
@dp.message(FinancesForm.category1)
async def finances(message: Message, state: FSMContext):
    await state.update_data(category1 = message.text)
    await state.set_state(FinancesForm.expenses1)
    await message.reply("Введите расходы для категории 1:") 
```

Прописываем функцию, которая сработает после получения предыдущего значения.

```python
@dp.message(FinancesForm.expenses1)
async def finances(message: Message, state: FSMContext):
```

Используем float, чтобы преобразовывать тип данных.

```python
@dp.message(FinancesForm.expenses1)
async def finances(message: Message, state: FSMContext):
    await state.update_data(expenses1 = float(message.text))
```

Устанавливаем вторую категорию.

```python
@dp.message(FinancesForm.expenses1)
async def finances(message: Message, state: FSMContext):
    await state.update_data(expenses1 = float(message.text))
    await state.set_state(FinancesForm.category2)
    await message.reply("Введите вторую категорию расходов:")
```

Создаём функцию для расходов по второй категории.

```python
@dp.message(FinancesForm.category2)
async def finances(message: Message, state: FSMContext):
    await state.update_data(category2 = message.text)
    await state.set_state(FinancesForm.expenses2)
    await message.reply("Введите расходы для категории 2:")
```

Повторяем для третьей категории.

```python
@dp.message(FinancesForm.expenses2)
async def finances(message: Message, state: FSMContext):
    await state.update_data(expenses2 = float(message.text))
    await state.set_state(FinancesForm.category3)
    await message.reply("Введите третью категорию расходов:")

@dp.message(FinancesForm.category3)
async def finances(message: Message, state: FSMContext):
    await state.update_data(category3 = message.text)
    await state.set_state(FinancesForm.expenses3)
    await message.reply("Введите расходы для категории 3:")
```

Создаём функцию, которая сработает после третьего ответа по расходам.

```python
@dp.message(FinancesForm.expenses3)
async def finances(message: Message, state: FSMContext):
```

Создаём переменную data, в которую сохраним всю информацию по состояниям.

```python
@dp.message(FinancesForm.expenses3)
async def finances(message: Message, state: FSMContext):
    data = await state.get_data()
```

Сохраняем telegram ID пользователя, чтобы сохранить информацию в нужную строчку базы данных.

```python
@dp.message(FinancesForm.expenses3)
async def finances(message: Message, state: FSMContext):
    data = await state.get_data()
    telegram_id = message.from_user.id
```

Отравляем запрос. Обновляем информацию и устанавливаем значения для категорий в базе данных.

```python
@dp.message(FinancesForm.expenses3)
async def finances(message: Message, state: FSMContext):
    data = await state.get_data()
    telegram_id = message.from_user.id
    cursor.execute('''UPDATE users SET category1 = ?, expenses1 = ?, category2 = ?, expenses2 = ?, category3 = ?, expenses3 = ? WHERE telegram_id = ?''',
                   (data['category1'], data['expenses1'], data['category2'], data['expenses2'], data['category3'], float(message.text), telegram_id))
```

Сохраняем изменения. Очищаем состояния. Прописываем сообщение о сохранении категорий и расходов.

```python
@dp.message(FinancesForm.expenses3)
async def finances(message: Message, state: FSMContext):
    data = await state.get_data()
    telegarm_id = message.from_user.id
    cursor.execute('''UPDATE users SET category1 = ?, expenses1 = ?, category2 = ?, expenses2 = ?, category3 = ?, expenses3 = ? WHERE telegram_id = ?''',
                   (data['category1'], data['expenses1'], data['category2'], data['expenses2'], data['category3'], float(message.text), telegram_id))
    conn.commit()
    await state.clear()

    await message.answer("Категории и расходы сохранены!")
```

### 4. Тестирование

Запускаем программу и проверяем все функции, нажимая на все кнопки по порядку и выполняя указания бота.
Просмотр базы данных доступен в PyCharm Professional.

## Результаты урока:
- Разработали кейс "Финансовый бот-помощник"
- Закрепили все, что изучили в модуле по Telegram-ботам.

## Дополнительные материалы
- https://www.exchangerate-api.com/ - api для просмотра курса валют

