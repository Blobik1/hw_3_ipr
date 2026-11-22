# Docker: HTTP-сервис со счётчиком

Домашнее задание по «Инструментам промышленной разработки», уровни 4–8.
Приложение на Python упаковано в Docker. Основная конфигурация запускает
два сервиса: `app` и `redis`. Код в `app/` не изменён.

[Условие задания](https://github.com/Nikfilk2030/missing_semester_hse_cs/blob/main/2026/amcs/year_1/hw_3/README.md)
· [Описание приложения](app/README.md)

## Требования

- Docker Engine или Docker Desktop с поддержкой Linux-контейнеров.
- Docker Compose с поддержкой `up --wait`.
- `curl` для проверок HTTP-запросов.

Команды ниже выполняются в Bash из корня репозитория. На Windows работа
проверялась через Ubuntu в WSL 2 с интеграцией Docker Desktop.
Устанавливать Python, SQLite или сервер Redis на компьютер не требуется.

## Быстрый запуск: приложение и Redis

```bash
git clone https://github.com/Blobik1/hw_3_ipr.git
cd hw_3_ipr
cp -n .env.example .env
docker compose config -q
docker compose up -d --build --wait --wait-timeout 90
docker compose ps
```

Если репозиторий уже скачан, начните с перехода в его папку.
`cp -n` сохраняет существующий `.env`. При первом скачивании команда
создаёт его из примера. Значения можно изменить в любом текстовом редакторе.

```bash
curl -fsS --max-time 5 -w '\n' http://localhost:8080/
curl -fsS --max-time 5 -w '\n' -X POST http://localhost:8080/increment
```

В ответе должно быть `"storage": "redis"`. В новом пустом хранилище первый
GET вернёт `count: 0`, первый POST — `count: 1`. При повторном запуске
используется сохранённое значение. GET не увеличивает счётчик.

## Настройки

Пример настроек находится в [`.env.example`](.env.example).
Compose подставляет их из локального `.env`, который исключён из Git.
Переменные, экспортированные в оболочке, имеют приоритет над `.env`.

| Переменная | Значение при отсутствии настройки | Назначение |
| --- | --- | --- |
| `HOST_PORT` | `8080` | Порт, на котором приложение доступно на компьютере |
| `GREETING` | `Hello, Docker!` | Поле `greeting` в JSON-ответе |

Например, для другого порта и приветствия измените `.env`:

```dotenv
HOST_PORT=8081
GREETING=Другое приветствие
```

Примените настройки и используйте выбранный порт в запросах:

```bash
docker compose up -d --wait --wait-timeout 90
curl -fsS --max-time 5 -w '\n' http://localhost:8081/
```

В остальных примерах используется `HOST_PORT=8080`.
Внутренний порт приложения остаётся `8000`.

В основном `compose.yaml` фиксированы `STORAGE=redis` и
`REDIS_URL=redis://redis:6379/0`. Имя сервиса `redis` разрешается внутри
сети Compose. `localhost` внутри `app` обозначал бы само приложение.
У Redis нет секции `ports`: на компьютер опубликован только порт `app`,
привязанный к `127.0.0.1`. Строка `6379/tcp` в `docker compose ps`
обозначает внутренний порт Redis, а не публикацию порта на компьютере.

## Файлы и хранение данных

| Файл | Назначение |
| --- | --- |
| `Dockerfile` | Окружение Python, установка зависимостей и запуск приложения |
| `.dockerignore` | Исключение Git, локальных настроек, кешей и отчётов из контекста сборки |
| `compose.yaml` | Основной запуск: приложение и Redis |
| `compose.sqlite.yaml` | Отдельная конфигурация уровня 6: приложение и SQLite |
| `.env.example` | Пример пользовательских настроек |
| `reports/` | Сохранённые ответы HTTP и логи проверок |

Redis запущен с `--appendonly yes`: изменения записываются в AOF.
Именованный том `redis_data` подключён к `/data` сервиса Redis.
У приложения в режиме Redis собственного тома данных нет.
SQLite-конфигурация использует отдельный том `counter_data` и файл
`/data/counter.db`. Перенос значения между SQLite и Redis не выполняется.

Имена ресурсов по умолчанию содержат префикс Compose-проекта, например
`hw_3_ipr_redis_data`. Фиксированные `container_name` и глобальные имена
томов не задаются.

## Уровень 4: запуск одного контейнера

Для этого примера используется порт `8090`, чтобы не занимать порт
основного Compose-проекта.

```bash
docker build -t ipr-counter .
docker run -d --rm --name counter-single -p 127.0.0.1:8090:8000 -e GREETING="Привет, Амир!" ipr-counter
curl -fsS --retry 5 --retry-all-errors --retry-delay 1 --max-time 5 -w '\n' http://localhost:8090/
curl -fsS --max-time 5 -w '\n' -X POST http://localhost:8090/increment
curl -fsS --max-time 5 -w '\n' http://localhost:8090/
docker logs counter-single
docker stop counter-single
```

Ожидается последовательность счётчика `0 → 1 → 1`. После остановки
контейнер удаляется благодаря `--rm`. Именованный том в этом примере
не подключён, поэтому его локальная база не сохраняется после удаления.

Сохранённый вывод первоначального запуска: [reports/level4.log](reports/level4.log).
При этой проверке использовался контейнер `counter-demo` и порт `8080`;
ответы показали `0 → 1 → 1`, запросы приложения записаны в логах.
Запрос браузера к отсутствующей иконке `/favicon.ico` вернул 404 и не
влияет на работу счётчика.

## Уровень 6: Compose и SQLite

Чтобы перейти с основной конфигурации на SQLite, сначала остановите
основной проект без удаления томов:

```bash
docker compose down
docker compose -f compose.sqlite.yaml up -d --build
curl -fsS --retry 5 --retry-all-errors --retry-delay 1 --max-time 5 -w '\n' http://localhost:8080/
curl -fsS --max-time 5 -w '\n' -X POST http://localhost:8080/increment
curl -fsS --max-time 5 -w '\n' http://localhost:8080/
docker compose -f compose.sqlite.yaml down
docker compose -f compose.sqlite.yaml up -d
curl -fsS --retry 5 --retry-all-errors --retry-delay 1 --max-time 5 -w '\n' http://localhost:8080/
```

Значение счётчика до и после пересоздания должно совпасть. В выполненной
проверке оно осталось `1`, а `hostname` изменился:

- До: `2afead867771`, [ответ](reports/level6-before.json).
- После: `310abc46f182`, [ответ](reports/level6-after.json).
- После настройки приветствия через `.env`: [ответ](reports/level6-config.json).

Возврат к Redis:

```bash
docker compose -f compose.sqlite.yaml down
docker compose up -d --build --wait --wait-timeout 90
```

## Уровень 8: готовность и проверки Redis

У Redis healthcheck выполняет `redis-cli ping`. У приложения Python
запрашивает `http://127.0.0.1:8000/readyz`. Проверки выполняются каждые
5 секунд, тайм-аут — 3 секунды, `retries: 5`, `start_period: 5s`.
Приложение запускается после готовности Redis благодаря
`depends_on` с условием `service_healthy`.

### Сохранение после удаления контейнеров

```bash
mkdir -p reports
curl -fsS --max-time 5 -w '\n' -X POST http://localhost:8080/increment
curl -fsS --max-time 5 -w '\n' -X POST http://localhost:8080/increment
curl -fsS --max-time 5 -w '\n' -X POST http://localhost:8080/increment
curl -fsS --max-time 5 -w '\n' http://localhost:8080/ | tee reports/level8-before.json
docker compose down
docker compose up -d --wait --wait-timeout 90
curl -fsS --max-time 5 -w '\n' http://localhost:8080/ | tee reports/level8-after.json
```

Запишите значение `N` перед `down`. После нового `up` ожидается то же
`N` и новый `hostname`. Если исходный счётчик был нулевым, `N = 3`.
Команда `down` удаляет контейнеры и сеть, но оставляет именованный том.
**Не используйте `down -v` в этой проверке:** флаг `-v` удаляет тома
вместе с данными.

Подтверждённый результат проверки:

| Состояние | `count` | `hostname` | Сохранённый ответ |
| --- | --- | --- | --- |
| До удаления контейнеров | `3` | `54126c456805` | [level8-before.json](reports/level8-before.json) |
| После нового запуска | `3` | `5d076d6a5e9a` | [level8-after.json](reports/level8-after.json) |

### Отключение Redis

```bash
docker compose stop redis
curl -sS --max-time 5 -w '\nHTTP %{http_code}\n' http://localhost:8080/healthz | tee reports/level8-healthz.txt
curl -sS --max-time 5 -w '\nHTTP %{http_code}\n' http://localhost:8080/readyz | tee reports/level8-readyz.txt
curl -sS --max-time 5 -w '\nHTTP %{http_code}\n' -X POST http://localhost:8080/increment | tee reports/level8-unavailable.txt
```

Подтверждённые результаты проверки:

| Запрос при остановленном Redis | HTTP | Ответ |
| --- | --- | --- |
| `GET /healthz` | `200` | `{"status": "alive"}` |
| `GET /readyz` | `503` | `{"error": "storage_unavailable"}` |
| `POST /increment` | `503` | `{"error": "storage_unavailable"}` |

Для ожидаемых ответов 503 используется `curl` без `-f`, чтобы сохранить
тело ответа. HTTP-сервер продолжает работать; healthcheck приложения,
который обращается к `/readyz`, со временем покажет `unhealthy`.

### Восстановление Redis без перезапуска приложения

```bash
docker compose start redis
curl -fsS --retry 10 --retry-all-errors --retry-delay 1 --max-time 5 -w '\n' http://localhost:8080/readyz
curl -fsS --max-time 5 -w '\n' http://localhost:8080/ | tee reports/level8-recovered.json
curl -fsS --max-time 5 -w '\n' -X POST http://localhost:8080/increment | tee reports/level8-next.json
```

В выполненной проверке `/readyz` снова вернул
`{"status": "ready", "storage": "redis"}`. GET вернул прежнее значение
`3`, следующий POST — `4`. В обоих ответах `hostname` равен
`5d076d6a5e9a`; приложение вручную не перезапускалось.
Общий ожидаемый результат — `N`, затем `N + 1`.

## Логи и остановка

```bash
docker compose logs --tail=100 app redis
docker compose down
```

При повторном запуске сохранённые данные снова доступны:

```bash
docker compose up -d --wait --wait-timeout 90
```

Если `up --wait` завершился ошибкой, посмотрите состояния всех
контейнеров и логи:

```bash
docker compose ps -a
docker compose logs --tail=100 app redis
```

Ограничения процесса и автоматический сценарий уровня 10 в этот
контрольный вариант не входят.
