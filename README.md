# restful_booker_API

Автотесты (Postman + Newman) для публичного API [RestfulBooker](https://restful-booker.herokuapp.com/apidoc/index.html).
**36 кейсов** в 8 группах: `POST /auth`, CRUD `/booking`, фильтры списка, `GET /ping`.

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

Логин/пароль RestfulBooker (`admin`/`password123`) в репозитории не хранятся —
задаются локально при запуске или через секреты CI.

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
даже при падении тестов.

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

| Наблюдение (проверено на прогонах) | Ожидалось | Кейсы |
|---|---|---|
| `POST /auth` с неверными кредами → `200` + `{"reason":"Bad credentials"}` | 401/400 | TC-AUTH-002…005 |
| `POST /booking` без `firstname` → `500` | 400/422 | TC-BOOK-004 |
| `POST /booking` с пустым телом `{}` → `500` | 400/422 | TC-BOOK-007 |
| `POST /booking` c `totalprice:"two thousand"` → принято, сохранено `null` | 400/422 | TC-BOOK-005 |
| `GET /booking` с невалидной датой → `500` | 400 | TC-LIST-007 |
| `PUT/PATCH/DELETE /booking/999999999` с токеном → `405` | 404 | TC-PUT-003, TC-PATCH-004, TC-DELETE-003 |
| `DELETE` уже удалённой брони → `405` | 404 | TC-DELETE-004 |
| `GET /ping` → `201` | 200 | TC-PING-001 |

## Особенности

- **Порядок важен**: `token`/`bookingId` создают Setup-запросы (01 Auth → 02 Create);
  без них кейсы 04–07 не работают.
- Баги выше — умышленные «кривые» сценарии тренировочного API; при изменении
  сервиса тесты нужно актуализировать.
- Ассерты времени (`< 3000 ms`) могут флакать на холодном старте Heroku.