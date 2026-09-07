# restful_booker_API

Автотесты (Postman + Newman) для публичного API [RestfulBooker](https://restful-booker.herokuapp.com/apidoc/index.html).
**39 кейсов** в 8 группах (`POST /auth`, CRUD `/booking`, фильтры списка, `GET /ping`)
и **289 `pm.test`-проверок**.

## Быстрый старт

```bash
npm i -g newman newman-reporter-htmlextra

newman run restful-booker.herokuapp.com.postman_collection.json \
  -e restful-booker.herokuapp.com.postman_environment.json \
  --env-var base_url=https://restful-booker.herokuapp.com/ \
  --env-var valid_login=<LOGIN> \
  --env-var valid_password=<PASSWORD> \
  -r cli,htmlextra
```

Логин/пароль RestfulBooker в репозитории и документации не хранятся (значения по
умолчанию — в официальной документации сервиса); задаются локально при запуске
или через секреты CI.

## Переменные окружения

| Переменная | Источник | Назначение |
|---|---|---|
| `base_url` | env-файл | базовый URL (с завершающим `/`) |
| `valid_login`, `valid_password` | `--env-var` / секреты CI | позитивная авторизация (TC-AUTH-001) |
| `invalid_login`, `invalid_password` | env-файл | негативные auth-кейсы (TC-AUTH-002/-003) |
| `token`, `bookingId` | рантайм Setup-запросов | сквозное состояние коллекции |

## CI (GitHub Actions)

Workflow `.github/workflows/restful-booker.yml` — запуск на push в `main` и PR.
Устанавливает Newman, прогоняет коллекцию, загружает HTML-отчёт артефактом
даже при падении тестов. Отчёт собирается без заголовков и тел запросов/ответов
(`omitHeaders`, `omitRequestBodies`, `omitResponseBodies`) — токен и креды
не попадают в артефакт (GitHub маскирует секреты только в логах, но не в артефактах).

Секреты (*Settings → Secrets and variables → Actions*):
`API_USERNAME`, `API_PASSWORD` — креды API, подставляются в `--env-var`.

## Файлы

```
├── .github/workflows/restful-booker.yml
├── restful-booker.herokuapp.com.postman_collection.json
├── restful-booker.herokuapp.com.postman_environment.json
├── RestfulBooker_Тест-кейсы_Баг-репорты.xlsx
└── README.md
```

## Документированные баги API

Тесты фиксируют **фактическое** поведение сервиса (не ожидаемое), поэтому
пайплайн остаётся зелёным. Отклонения помечены `// BUG:` в коллекции:

| Наблюдение (проверено на прогонах) | Ожидалось | Кейсы | Баг |
|---|---|---|---|
| `POST /auth` с неверными кредами → `200` + `{"reason":"Bad credentials"}` | 401/400 | TC-AUTH-002…005 | BUG-001 |
| `POST /booking` без `firstname` → `500` | 400/422 | TC-BOOK-007 | BUG-005 |
| `POST /booking` без `lastname` → `500` | 400/422 | TC-BOOK-008 | BUG-005 |
| `POST /booking` с пустым телом `{}` → `500` | 400/422 | TC-BOOK-004 | BUG-005 |
| `POST /booking` c `totalprice:"two thousand"` → принято, сохранено `null` | 400/422 | TC-BOOK-005 | BUG-003 |
| `POST /booking` с пустым `firstname` → `200`, бронь создаётся | 400/422 | TC-BOOK-003 | BUG-002 |
| `POST /booking` c `checkout` раньше `checkin` → `200`, бронь создаётся | 400/422 | TC-BOOK-006 | BUG-004 |
| `GET /booking` с невалидной датой → `500` | 400 | TC-LIST-007 | BUG-011 |
| Фильтр `/booking` по точным датам существующей брони → пустой массив | бронь должна попадать в выборку | TC-LIST-004, TC-LIST-005 | BUG-007 |
| `DELETE` успешной брони → `201 Created` (с телом) | 204 (без тела) | TC-DELETE-001 | BUG-008 |
| `PATCH /booking/:id` с пустым телом `{}` → `200`, бронь не меняется | 400/422 | TC-PATCH-005 | BUG-009 |
| `GET /booking/:id` с не-числовым ID → `404 "Not Found"` | 400 | TC-GET-003 | BUG-010 |
| `PUT/PATCH/DELETE` несуществующего ID → `405` | 404 | TC-PUT-003, TC-PATCH-004, TC-DELETE-003 | BUG-006 |
| `DELETE` уже удалённой брони → `405` | 404 | TC-DELETE-004 | BUG-006 |
| `GET /ping` → `201 Created` | 200 | TC-PING-001 | BUG-012 |

## Особенности

- **Порядок важен**: `token`/`bookingId` создают Setup-запросы (01 Auth → 02 Create);
  без них кейсы 04–07 не работают.
- Баги выше — умышленные «кривые» сценарии тренировочного API; при изменении
  сервиса тесты нужно актуализировать.
- Ассерты времени (`< 3000 ms`) могут флакать на холодном старте Heroku.
- Типовой прогон: ~266 зелёных из 289 ассертов; красные — только намеренные
  «Contract:»-проверки на задокументированные отклонения (см. таблицу выше).
- Не-баг шум в прогоне: `pm.expect(...).to.include(payload)` с вложенным объектом
  (TC-BOOK-002) и async-проверки персистентности через `pm.sendRequest` (TC-PUT-001,
  TC-PATCH-005) могут флакать сами по себе — прямые вызовы API подтверждают, что
  данные при этом персистятся корректно.
- В прогонах 07.09.2026 `DELETE` несуществующей и уже удалённой брони возвращал `403`
  вместо ранее фиксируемого `405` (TC-DELETE-003/004) — вероятное изменение поведения
  сервиса, требует мониторинга при актуализации BUG-006.
- Middle-уровень проверок: deep-equal контракта (echo при создании, read-back при чтении),
  персистентность и неизменность данных через повторный GET (`pm.sendRequest`),
  уникальность `bookingid`, проверка `Content-Type` и точных тел ошибок
  (500 → `Internal Server Error`, 400 → `Bad Request`).
- Нумерация TC коллекции синхронизирована с xlsx; `TC-BOOK-009`, `TC-LIST-004/005/007`, `TC-PATCH-005` — дополнения к базовому дизайну.