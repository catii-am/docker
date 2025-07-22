Ниже — **самый быстрый практичный маршрут** запуска локального Telegram Bot API Server в режиме `--local` на Windows Server / Windows 10 (server use-case). Дам два пути:

* **A. Через Docker Desktop** — быстрее, меньше возни с зависимостями. *Рекомендую.*
* **B. Через WSL2 (Ubuntu внутри Windows)** — если Docker нельзя.

В конце — как подключить твоего Python-бота к локальному серверу.

---

## Подготовь данные (из my.telegram.org)

* `api_id = 20780149`
* `api_hash = d7ea720267d033521347531410f0b4c2`
* Токен бота: `123456:ABC...` (замени на свой).
* Папка на диске: `C:\tgbotapi-data` (создадим).

---

# A. Быстрый запуск через Docker Desktop (рекомендуется)

### 1. Установи Docker Desktop

1. Скачай: [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
2. Включи опцию **Use WSL 2 based engine**.
3. Перезагрузи.

### 2. Создай рабочую папку

```powershell
mkdir C:\tgbotapi
mkdir C:\tgbotapi\data
cd C:\tgbotapi
```

### 3. Создай `docker-compose.yml`

Содержимое (замени токен при необходимости только на этапе работы бота; тут не нужен токен):

```yaml
version: '3.8'
services:
  telegram-bot-api:
    image: aiogram/telegram-bot-api:latest
    container_name: telegram-local-api
    restart: unless-stopped
    ports:
      - "8081:8081"
    volumes:
      - C:/tgbotapi/data:/var/lib/telegram-bot-api
    environment:
      TELEGRAM_API_ID: 20780149
      TELEGRAM_API_HASH: d7ea720267d033521347531410f0b4c2
      TELEGRAM_LOCAL: "true"
      TELEGRAM_VERBOSITY: "2"
```

> Путь слева Windows-стиль, Docker сам смонтирует.

### 4. Запусти

В PowerShell в `C:\tgbotapi`:

```powershell
docker compose up -d
```

Проверка:

```powershell
curl http://localhost:8081
```

Должен вернуть JSON с ошибкой «Not Found» или подобное — значит сервер отвечает.

Проверим API с токеном:

```powershell
curl "http://localhost:8081/bot<ТОКЕН>/getMe"
```

---

# B. Быстрый запуск через WSL2 (если Docker нельзя)

### 1. Включи WSL2

Админ PowerShell:

```powershell
wsl --install
```

(Если просит ребут — перезагрузи. По умолчанию поставится Ubuntu.)

### 2. Запусти Ubuntu (через Start → Ubuntu).

### 3. Зависимости:

```bash
sudo apt update && sudo apt install -y \
  gperf cmake g++ zlib1g-dev libssl-dev \
  libcurl4-openssl-dev liblz4-dev libzstd-dev git
```

### 4. Сборка:

```bash
git clone https://github.com/tdlib/telegram-bot-api.git
cd telegram-bot-api
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . --target install -j$(nproc)
```

После сборки бинарь обычно в `/usr/local/bin/telegram-bot-api` (или смотри вывод).

### 5. Директория данных:

```bash
sudo mkdir -p /var/lib/telegram-bot-api
sudo chown $USER:$USER /var/lib/telegram-bot-api
```

### 6. Запуск:

```bash
telegram-bot-api \
  --api-id=20780149 \
  --api-hash=d7ea720267d033521347531410f0b4c2 \
  --local \
  --dir=/var/lib/telegram-bot-api \
  --http-port=8081
```

> Оставь окно открытым или заверни в systemd-сервис (подскажу, если нужно).

Доступ из Windows: `http://localhost:8081/...` (WSL пробрасывает порт).

---

# Подключение твоего Python-бота к локальному серверу

Важно: стандартные библиотеки идут на `https://api.telegram.org`. Нужно указать **свой base URL**.

## python-telegram-bot ≥20.x

```python
from telegram import Bot
from telegram.ext import ApplicationBuilder

BOT_TOKEN = "ТОКЕН_БОТА"
LOCAL_BASE = "http://localhost:8081/bot"  # без завершающего / после токена

bot = Bot(token=BOT_TOKEN, base_url=LOCAL_BASE)

app = ApplicationBuilder().token(BOT_TOKEN).base_url(LOCAL_BASE).build()
# далее обычные handlers...
```

> Обрати внимание: `base_url` = `"http://localhost:8081/bot"` (без токена внутри). lib сама добавит `/bot<token>`.

Если используешь старую версию, где `base_url` ожидает полный URL с токеном, то:

```python
LOCAL_BASE = f"http://localhost:8081/bot{BOT_TOKEN}"
bot = Bot(token=BOT_TOKEN, base_url=LOCAL_BASE)
```

Проверим док/версию — скажи, какая у тебя `python-telegram-bot`, я дам точный вариант.

---

## Проверка: getMe

```python
import asyncio
from telegram import Bot

BOT_TOKEN = "ТОКЕН"

async def test():
    bot = Bot(BOT_TOKEN, base_url="http://localhost:8081/bot")
    me = await bot.get_me()
    print(me)

asyncio.run(test())
```

---

# Работа с большими файлами

После перехода на локальный сервер в `--local`:

* Можно **получать** (скачивать) файлы без лимита (до 2 ГБ).
* Можно **присылать** большие архивы в бота (до 2 ГБ).
* Метод `get_file()` вернёт `file_path`, который может быть **абсолютным путём на локальном сервере**. Просто читай файл с диска — не нужно качать через HTTP.

## Пример обработки документа (адаптация твоего кода)

```python
import os
import asyncio
import tempfile
import patoolib
import paramiko

from pathlib import Path
from telegram import Update
from telegram.ext import ApplicationBuilder, MessageHandler, filters, ContextTypes

HOST = "5.23.51.236"
PORT = 22
USERNAME = "integrate3"
PASSWORD = "JKvub%#bksi90"
REMOTE_BASE_PATH = "/home/i/integrate3/yagis/public_html/YB"
ALLOWED_EXTENSIONS = (".zip", ".rar")

BOT_TOKEN = "ТОКЕН"
BASE_URL = "http://localhost:8081/bot"  # локальный Bot API сервер

async def process_archive(local_path: str, archive_name: str) -> str:
    remote_target_dir = f"{REMOTE_BASE_PATH}/{archive_name}"

    # распаковка
    with tempfile.TemporaryDirectory() as tmpdir:
        patoolib.extract_archive(local_path, outdir=tmpdir)

        transport = paramiko.Transport((HOST, PORT))
        transport.connect(username=USERNAME, password=PASSWORD)
        sftp = paramiko.SFTPClient.from_transport(transport)

        # гарантируем директории
        path = ""
        for part in remote_target_dir.strip("/").split("/"):
            path = f"{path}/{part}" if path else f"/{part}"
            try:
                sftp.stat(path)
            except IOError:
                sftp.mkdir(path)

        # загрузка
        for root, _, files in os.walk(tmpdir):
            rel = os.path.relpath(root, tmpdir)
            if rel == ".":
                rel = ""
            remote_dir = remote_target_dir if not rel else f"{remote_target_dir}/{rel}".replace("\\", "/")
            # создать дерево
            try:
                sftp.stat(remote_dir)
            except IOError:
                # рекурсивно
                parts = remote_dir.strip("/").split("/")
                subpath = ""
                for p in parts:
                    subpath = f"{subpath}/{p}" if subpath else f"/{p}"
                    try:
                        sftp.stat(subpath)
                    except IOError:
                        sftp.mkdir(subpath)
            # файлы
            for fn in files:
                lf = os.path.join(root, fn)
                rf = f"{remote_dir}/{fn}"
                sftp.put(lf, rf)

        sftp.close()
        transport.close()

    return f"[✓] Загрузка завершена: {remote_target_dir}"

async def handle_document(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not update.message or not update.message.document:
        return

    doc = update.message.document
    file_name = doc.file_name
    if not file_name.lower().endswith(ALLOWED_EXTENSIONS):
        await update.message.reply_text("❗ Отправьте архив .zip или .rar")
        return

    msg = await update.message.reply_text(f"⏳ Получен архив `{file_name}`. Загружаю...", parse_mode="Markdown")

    try:
        # получаем файл через локальный Bot API
        f = await doc.get_file()  # в локал-моде OK даже для >20МБ
        # file_path теперь абсолютный путь на локальном сервере Bot API
        local_botapi_path = f.file_path

        # копируем во временный файл (чтобы patool не трогал оригинал)
        with tempfile.NamedTemporaryFile(delete=False, suffix=Path(file_name).suffix) as tmp:
            tmp_path = tmp.name
        # простое копирование
        # если file_path это локальный путь, просто копируем
        import shutil
        shutil.copy(local_botapi_path, tmp_path)

        archive_name = Path(file_name).stem

        async def task():
            try:
                result = await asyncio.to_thread(process_archive, tmp_path, archive_name)
                await msg.edit_text(result)
            except Exception as e:
                await msg.edit_text(f"❌ Ошибка при загрузке `{file_name}`: {e}", parse_mode="Markdown")
            finally:
                if os.path.exists(tmp_path):
                    os.remove(tmp_path)

        asyncio.create_task(task())

    except Exception as e:
        await msg.edit_text(f"❌ Не удалось получить файл: {e}")

def main():
    app = ApplicationBuilder().token(BOT_TOKEN).base_url(BASE_URL).build()
    app.add_handler(MessageHandler(filters.Document.ALL, handle_document))
    print("✅ Бот запущен (локальный Bot API).")
    app.run_polling()

if __name__ == "__main__":
    main()
```

> Ключевое отличие: **в локальном режиме `file.file_path` — это путь на диске**, а не URL. Скопировал и работаешь.

---

# Проверка, что режим `--local` реально работает

1. Отправь в бота файл >20 МБ.
2. Логи Bot API сервера должны показать приём файла (сохраняет в свою `--dir` директорию).
3. В коде `get_file()` не должен падать с `File is too big`.
4. `file.file_size` покажет полный размер.

---

# Автозапуск Telegram Bot API сервера (Docker)

Создай PowerShell-скрипт `Start-TgBotApi.ps1`:

```powershell
cd C:\tgbotapi
docker compose up -d
```

Добавь в Планировщик заданий Windows (Task Scheduler) запуск при старте системы.

---

# Краткий чеклист

* [ ] Docker Desktop установлен?
* [ ] Контейнер поднят? (`docker ps`)
* [ ] curl getMe работает?
* [ ] Python-бот переведён на локальный `base_url`?
* [ ] Большие файлы проходят?

---

Готов помочь на следующем шаге: **какой путь выбираешь (Docker / WSL2)?** Напиши, и я дам точные команды под твой выбор.
