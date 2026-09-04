# RestfulBooker API — автотесты (Postman + Newman)

Автотесты для публичного API [RestfulBooker](https://restful-booker.herokuapp.com/apidoc/index.html).
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

Локальные значения `<LOGIN>`/`<PASSWORD>` (`admin`/`password123`) берутся из
[документации сервиса](https://restful-booker.herokuapp.com/apidoc/index.html) и
в репозитории не хранятся.

## Переменные окружения

| Переменная | Значение/источник | Назначение |
|---|---|---|
| `base_url` | `https://restful-booker.herokuapp.com/` (env) | базовый URL всех запросов |
| `valid_login`, `valid_password` | `--env-var` / секреты CI | позитивная авторизация (TC-AUTH-001) |
| `invalid_login`, `invalid_password` | `wrong_user`, `wrong_pass` (env) | негативные кейсы (TC-AUTH-002/-003) |
| `token`, `bookingId` | рантайм Setup-запросов | сквозное состояние коллекции |

## CI (GitHub Actions)

Workflow: `.github/workflows/restful-booker.yml` — запуск на push в `main` и PR.
Использует Docker-action `postman/newman@v6`, установка зависимостей не нужна.
Креды подставляются из секретов:

- `Settings → Secrets and variables → Actions` → создать `API_USERNAME`, `API_PASSWORD`.

## Файлы

```
├── .github/workflows/restful-booker.yml   # CI
├── restful-booker.herokuapp.com.postman_collection.json   # тесты
├── restful-booker.herokuapp.com.postman_environment.json  # переменные
└── README.md
```

## Особенности

- **Порядок важен**: `token`/`bookingId` создают Setup-запросы (01 Auth → 02 Create). Без них кейсы 04–07 не работают.
- **Баги API зафиксированы в тестах**: ожидаемые коды могут включать `500` (`// BUG:` в TC-BOOK-007, TC-PATCH-004) — это осознанно.
- **Тайминги**: ассерты `< 3000 ms` могут флакать на холодном старте Heroku.
- **Сервис публичный** — данные общие; тесты не зависят от чужих данных.