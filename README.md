# App_DevOps
Создание девопс окружения для приложения 

Поместим в папку `habitquest` проект с нашим приложением. Сначала поменяем `requirements.txt`. 

Теперь он выглядит как-то так: 
```
flask>=3.0.0
requests>=2.31.0
gunicorn>=22.0.0
```

Дело в том, что flask run нельзя использовать в production. Gunicorn же production-ready, стабильнее, лучше работает под нагрузкой и умеет worker-процессы. 

Далее, в корневой папке создаем `Dockerfile` с содержимым: 
```
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y curl gcc && \
    rm -rf /var/lib/apt/lists/*

COPY app/requirements.txt .

RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY app/ .

EXPOSE 5001

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:5001 || exit 1

CMD ["gunicorn", "--workers", "2", "--bind", "0.0.0.0:5001", "app:app"]
```

Фиксируем версию Python 3.12 - у нее маленький размер, быстрый ci/cd и меньше уязвимостей. 
`WORKDIR /app` обеспечит выполнение команд внутри контейнера из `/app`. 
`COPY app/requirements.txt .` ускорит сборку Docker. Он хэширует слой с зависимостями, и тогда если код меняется, а зависимости нет, `pip install` не запускается заново, что ускоряет сборку. 

Далее создаем docker-compose.yml: 
```
version: '3.9'

services:
  habitquest:
    build: .
    container_name: habitquest_app

    ports:
      - "5001:5001"

    env_file:
      - .env

    volumes:
      - ./app/database.db:/app/database.db

    restart: unless-stopped
```

`5001:5001` - левая часть - это порт компьютера, правая - порт контейнера. 
Очень важно указать `- ./app/database.db:/app/database.db`, потому что SQLite хранится как файл, без volume при удалении контейнера удалится и база, что приведет к потере данных пользователей. 
`restart: unless-stopped` - обеспечивает автоматическое поднятие контейнера после reboot сервера, падения docker или рестарта машины. 


Создадим `.dockerignore`: 
```
venv
__pycache__
*.pyc
.git
.gitignore
.DS_Store
__MACOSX
```
Без этого фйла в Docker попадет virtualenv, pycache, мусор компьютера и git история, что замедлит сборку, увеличит размер image и будет ломать переносимость.

По такому же принципу создадим файл `.gitignore`:

```
venv/
__pycache__/
*.pyc
.env
.DS_Store
__MACOSX/
```

Очевидно, что так как мы собираем какие-никакие данные пользователей, то их просто так мы не можем. Git сохраняет историю, ключ может утеч, а это немного-немало протворечит политике безопасноти. Поэтому добавляем еще и файл `.env` 
```
SECRET_KEY=super_secret_prod_key_change_me
```
Теперь переходим в корень проекта и приступаем непосредственно к сборке контейнера. Запускаем `docker compose build`:
<img width="640" height="590" alt="image" src="https://github.com/user-attachments/assets/3e797136-b91e-4179-b066-26fd5fd353b7" />

И поднимаем приложение `docker compose up -d`: 
<img width="638" height="114" alt="image" src="https://github.com/user-attachments/assets/5b65fdcc-ea10-4c6e-92a9-534100f0bcd4" 

Переходим на `http://localhost:5001`. Наше приложение работает и запущено внутри контейнера!

<img width="736" height="710" alt="image" src="https://github.com/user-attachments/assets/a4e48b27-27a2-4120-bf21-b29c10c0d845" />

Чтобы остановить работу можно воспользоваться командой `docker compose down`.

---

# GitLab CI/CD

Теперь реализуем автоматическую сборку. Создаем файол `.gitlab-ci.yml`. Разберем этот pipeline по частям: 

`stage: build` --- Собирает Docker image, пушит image в GitLab Container Registry. 

`stage: deploy` --- подключается к серверу по SSH, скачивает новый Docker image, останавливает контейнер, запускает новый. 

На сайте GitLab создаем репозиторий, а в папку с приложением инициализируем git через команду `git init`, добавляем файлы, делаем первый коммит `git commit -m "Initial DevOps setup"`, затем подключаем GitLab remote `git remote add origin https://gitlab.com/tinesz1984-group/habitquest.git`, переключаемся на main `git branch -M main` и отправляем проект в GitLab `git push -u origin main`. Собственно, теперь наш проект на GitLab. 
<img width="1140" height="679" alt="image" src="https://github.com/user-attachments/assets/3ce2b4b2-2584-42e4-8761-29dc46ae9090" />


Но, как мы видим, с деплоем, все таки, что-то не так. Кажется, я забыла что у нашего проекта нет сервера. поправим немного наш ci/cd


