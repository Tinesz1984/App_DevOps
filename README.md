# App_DevOps
Создание девопс окружения для приложения 

Так, сначала была создана папка `habitquest`, куда мы поместили папку исходного проекта. Сначала поменяем `requirements.txt`. 

Теперь он выглядит как-то так: 
```
flask>=3.0.0
requests>=2.31.0
gunicorn>=22.0.0
```

Дело в том, что flask run нельзя использовать в production. Gunicorn же production-ready, стабильнее, лучше работает под нагрузкой, умеет worker-процессы. 

Далее, в корневой папке создаем `Dockerfile` с содержимым: 
```
FROM python:3.12-slim 

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY app/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app/ .

EXPOSE 5001

CMD ["gunicorn", "--bind", "0.0.0.0:5001", "app:app"]
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

Очевидно, что так как мы собираем какие-никакие данные пользователей, то их хранить в коде мы не можем. Git сохраняет историю, ключ может утеч, а это немного-нимало протворечит политике безопасноти. Поэтому добавляем еще и файл `.env` 
```
SECRET_KEY=super_secret_prod_key_change_me
```
Теперь переходим в корень проекта и приступаем непосредственно к сборке контейнера 
