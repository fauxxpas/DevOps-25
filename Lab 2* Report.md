1. “Плохой” docker-compose.yml
version: "3.9"

services:
  app:
    image: python:latest              # ❌ latest
    container_name: my_app_container   # ❌ жёсткое имя
    volumes:
      - .:/app                         # ❌ монтируем всё подряд
    working_dir: /app
    command: python app.py
    ports:
      - "8000:8000"                    # ❌ без необходимости
    depends_on:
      - db

  db:
    image: postgres:latest             # ❌ latest
    environment:
      POSTGRES_PASSWORD: password      # ❌ пароль в открытом виде
    ports:
      - "5432:5432"                    # ❌ БД торчит наружу
Минимально, но уже куча грехов.
2. “Хороший” docker-compose.yml
version: "3.9"

services:
  app:
    image: python:3.12-slim
    volumes:
      - ./app:/app:ro                  # только нужная папка, read-only
    working_dir: /app
    command: python app.py
    depends_on:
      - db
    networks:
      - app_net

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    networks:
      - db_net

secrets:
  db_password:
    file: ./secrets/db_password.txt

networks:
  app_net:
    internal: true
  db_net:
    internal: true
    
3. Изоляция контейнеров друг от друга
Ключевой момент задания со звездочкой:
контейнеры поднимаются вместе, но не видят друг друга по сети.
Что сделано:
app и db подключены к разным сетям
обе сети объявлены как internal: true
контейнер видит только те сервисы, которые в той же сети
app не может обратиться к db по DNS, потому что они в разных сетях
Это тупо, но эффективно. Docker делает ровно то, что ему сказали, и ни миллиметром больше.
4. README (затычка, но приличная)
Bad practices в плохом docker-compose
Использование latest
Почему плохо: непредсказуемые обновления, нестабильные сборки
Исправление: зафиксированы версии образов
Результат: воспроизводимые сборки
Жёстко заданные container_name
Почему плохо: конфликты имён при запуске нескольких окружений
Исправление: Docker сам управляет именами
Результат: compose-проект стал масштабируемым
Открытые порты без необходимости
Почему плохо: лишняя поверхность атаки
Исправление: порты убраны
Результат: сервисы доступны только внутри Docker
Секреты в открытом виде
Почему плохо: утечки, коммиты паролей в репозиторий
Исправление: Docker secrets
Результат: секреты вынесены из compose-файла
Изоляция сервисов
В хорошем файле сервисы подключены к разным internal-сетям.
Docker создаёт отдельные bridge-сети, между которыми нет маршрутизации.
Таким образом контейнеры стартуют в одном compose-проекте, но не могут
обмениваться трафиком.
