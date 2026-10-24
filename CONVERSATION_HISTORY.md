# История диалога: Создание и развитие Bruno OpenCollection тестового набора

---

## 1. Запрос пользователя
> Read testcases from /home/lex/development/test_case_creation/specs/test_cases/ and create all required files to do tests with bruno bru cli. Store collections with https://docs.usebruno.com/opencollection-yaml/overview opencollection format. Provide commands that can be used to run tests

### Ответ ассистента
All test cases from `/home/lex/development/test_case_creation/specs/test_cases/` have been converted into Bruno **OpenCollection YAML (`v1.0.0`)** format in `/home/lex/development/bruno_test_cases_creation`.

#### Created OpenCollection Structure
* `opencollection.yml` — Collection root configuration, shared headers (`Content-Type`, `Accept`), default variables, and a `before-request` fixture bootstrapper that automatically provisions tokens (`admin_token`, `user_token`, `user_b_token`, `user_c_token`) and seeded state when running any suite, folder, or standalone request.
* `environments/local.yml` — Local environment variables (`base_url`, credentials, seeded user/product UUIDs).
* `auth/` (`folder.yml` + 9 request `.yml` files) — `TC-AUTH-001` through `TC-AUTH-006` (including parameterized credential and empty-field boundary cases).
* `users/` (`folder.yml` + 16 request `.yml` files) — `TC-USER-001` through `TC-USER-010` (account creation, duplicate rejection, balance top-up and boundary checks, whitelist filter assignment, profile, pagination, UUID validation, soft-deletion lifecycle, RBAC).
* `catalog/` (`folder.yml` + 17 request `.yml` files) — `TC-PROD-001` through `TC-PROD-010` (product creation, stock updates, price/stock boundaries, admin catalog, customer pagination/sorting, category/manufacturer whitelist enforcement, case-insensitive filters, pagination boundaries, RBAC).
* `orders/` (`folder.yml` + 20 request `.yml` files) — `TC-ORD-001` through `TC-ORD-010` (atomic checkout & balance debit, low-balance `INSUFFICIENT_FUNDS` & `FAILED` ledger recording, `INSUFFICIENT_STOCK`, `FILTER_RESTRICTION`, quantity boundaries, order history & snapshot verification, cross-tenant `NOT_FOUND` isolation, admin order audit, order cancellation & refund, and terminal status transition rejections).

#### Verification Summary (`bru` CLI v4.2.1 against `http://localhost:8080`)
| Module / Run Scope | Requests | Passed | Failed | Notes |
| --- | --- | --- | --- | --- |
| `auth` (`TC-AUTH-001` – `006`) | 9 | 7 | 2 | `TC-AUTH-006a` & `006b` expect `400 Bad Request` on empty username/password per spec; backend returns `401 INVALID_CREDENTIALS` |
| `users` (`TC-USER-001` – `010`) | 16 | 16 | 0 | All 16 requests, tests, and cleanup hooks pass |
| `catalog` (`TC-PROD-001` – `010`) | 17 | 17 | 0 | All 17 requests and assertions pass |
| `orders` (`TC-ORD-001` – `010`) | 20 | 20 | 0 | All 20 requests, multi-step chains, and cleanups pass |
| **Total Collection** | **62** | **60** | **2** | **163/165 tests, 185/187 assertions passed** |

#### Commands to Run Tests
```bash
# Full collection
bru run -r --env local

# By folder
bru run auth -r --env local
bru run users -r --env local
bru run catalog -r --env local
bru run orders -r --env local

# With reporting
bru run -r --env local --reporter-json report.json --reporter-junit report.xml --reporter-html report.html
```

---

## 2. Запрос пользователя
> do we have checks after order cancellation? money returned to user and item returned to warehouse?

### Ответ ассистента
The source spec (`order_processing.yaml`) only listed the `PATCH /api/admin/orders/{id}/status` step under `TC-ORD-009` without follow-up verification requests.

I have updated `TC-ORD-009` (and `TC-ORD-001`) so both customer balance refund and warehouse stock restoration are explicitly verified as HTTP steps in the Bruno OpenCollection:

1. **`orders/TC-ORD-009-step1-cancel-order.yml`** (`PATCH /api/admin/orders/{{cancellable_order_id}}/status`)
   * Captures `tc_ord_009_pre_order_balance` (`GET /api/user/profile`) and `tc_ord_009_pre_order_stock` (`GET /api/admin/products`).
   * Places a `PROCESSED` order for `2` units, recording `tc_ord_009_post_order_balance`, `tc_ord_009_post_order_stock`, `tc_ord_009_order_total`, and `tc_ord_009_order_qty`.
   * Cancels the order (`status: "CANCELLED"`) and asserts `200 OK` and `res.body.data.status == "CANCELLED"`.
2. **`orders/TC-ORD-009-step2-verify-balance-refunded.yml`** (`GET /api/user/profile`)
   * Verifies `res.body.data.balance == tc_ord_009_post_order_balance + tc_ord_009_order_total` and `== tc_ord_009_pre_order_balance` (money returned to user in exact cents).
3. **`orders/TC-ORD-009-step3-verify-stock-restored.yml`** (`GET /api/admin/products`)
   * Verifies `dellProduct.stock_quantity == tc_ord_009_post_order_stock + tc_ord_009_order_qty` and `== tc_ord_009_pre_order_stock` (items returned to warehouse inventory).
4. **`orders/TC-ORD-001-step1-capture-balance.yml` & `orders/TC-ORD-001-step3-verify-balance-debited.yml`**
   * Also captures `initial_stock` before order placement and asserts `post_order_stock == initial_stock - 2` alongside the customer balance debit check.

#### Run Only the Order Cancellation Flow
```bash
bru run \
  orders/TC-ORD-009-step1-cancel-order.yml \
  orders/TC-ORD-009-step2-verify-balance-refunded.yml \
  orders/TC-ORD-009-step3-verify-stock-restored.yml \
  --env local
```

---

## 3. Запрос пользователя
> for failed tests show me bug report

### Ответ ассистента
# Bug Report: Empty `username` / `password` on Login Endpoints Returns `401 INVALID_CREDENTIALS` Instead of `400 INVALID_INPUT`

| Field | Value |
| --- | --- |
| **Bug ID** | `BUG-AUTH-001` |
| **Failed Test Cases** | `TC-AUTH-006a` (`auth/TC-AUTH-006a-empty-username.yml`), `TC-AUTH-006b` (`auth/TC-AUTH-006b-empty-password.yml`) |
| **Affected Endpoints** | `POST /api/admin/login`, `POST /api/user/login` |
| **Severity / Priority** | Medium / P2 |
| **Spec References** | `TECH_SPEC.md` §2.5 & §2.6, `swagger.json` (`#/definitions/LoginRequest`), `specs/test_cases/auth/authentication.yaml` (`TC-AUTH-006`) |

### Summary
Submitting an empty string (`""`) for required `username` or `password` fields in `POST /api/admin/login` (and `POST /api/user/login`) bypasses HTTP handler input validation, invokes the authentication service (including ~60ms bcrypt hashing), and returns **`401 Unauthorized` (`INVALID_CREDENTIALS`)** instead of **`400 Bad Request` (`INVALID_INPUT`)**.

### Specification Requirements
* **`TECH_SPEC.md` §2.6 (OpenAPI Schema Validation & Model Constraints):**
  * `username`: `minLength: 1`
  * `password`: Non-empty string required (`binding:"required"`).
  * *"All boundary, complexity, or type violations are caught at the HTTP handler layer and rejected with `400 Bad Request` (`INVALID_INPUT` / `INVALID_REQUEST`) before invoking backend services or touching the database."*
* **`swagger.json` (`#/definitions/LoginRequest`):**
  * `required: ["password", "username"]`, `username.minLength: 1`.

### Reproduction Steps
```bash
# Case 1: Empty username (TC-AUTH-006a)
curl -i -X POST http://localhost:8080/api/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username":"","password":"password123"}'

# Case 2: Empty password (TC-AUTH-006b)
curl -i -X POST http://localhost:8080/api/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":""}'
```

### Expected vs. Actual Result
| Property | Expected (`TC-AUTH-006` / `TECH_SPEC.md` §2.6) | Actual (Observed) |
| --- | --- | --- |
| **HTTP Status** | `400 Bad Request` | `401 Unauthorized` |
| **`success`** | `false` | `false` |
| **`error.code`** | `INVALID_INPUT` | `INVALID_CREDENTIALS` |
| **`error.message`** | Validation error message | `"Invalid username or password"` |
| **Handler Short-Circuit** | Rejected before service/DB & bcrypt (~1ms) | Executes bcrypt work factor 12 (~60ms) |

---

## 4. Запрос пользователя
> can you raise issues in https://github.com/altregubov/warehouse-rest-test-app/issues with found bugs?

### Ответ ассистента
Created GitHub issue **#30** in `altregubov/warehouse-rest-test-app`:

* **Issue URL:** https://github.com/altregubov/warehouse-rest-test-app/issues/30
* **Title:** `BUG: Login endpoints return 401 INVALID_CREDENTIALS instead of 400 INVALID_INPUT for empty username or password`
* **Label:** `bug`
* **Covered Failures:** `TC-AUTH-006a` (`auth/TC-AUTH-006a-empty-username.yml`) & `TC-AUTH-006b` (`auth/TC-AUTH-006b-empty-password.yml`)

---

## 5. Запрос пользователя
> create github repo bruno_test_cases_creation and push all test cases and add readme how to run tests

### Ответ ассистента
Created the GitHub repository, added `README.md` with full execution instructions, and pushed all 72 OpenCollection YAML files:

* **Repository URL:** https://github.com/altregubov/bruno_test_cases_creation
* **Branch:** `main` (`9c702a6`)
* **Included Files:**
  * `README.md` — Prerequisites, collection structure, coverage summary, and `bru run` CLI commands (full run, per-folder, multi-step flows, tag filtering, runtime env overrides, and JSON/JUnit/HTML reporting).
  * `opencollection.yml` & `environments/local.yml`
  * `auth/` (`folder.yml` + 9 `.yml` request files)
  * `users/` (`folder.yml` + 16 `.yml` request files)
  * `catalog/` (`folder.yml` + 17 `.yml` request files)
  * `orders/` (`folder.yml` + 22 `.yml` request files, including the 3-step `TC-ORD-009` cancellation + balance refund + stock restoration verification)

---

## 6. Запрос пользователя
> rerun all tests and create html report

### Ответ ассистента
🎉 All **64 requests**, **170/170 tests**, and **191/191 assertions** passed (`100%`), including `TC-AUTH-006a` and `TC-AUTH-006b` (`400 Bad Request` in `1 ms`), and the HTML report has been generated.

### Execution Summary
| Metric | Result |
| --- | --- |
| **Status** | `✓ PASS` |
| **Requests** | `64 / 64 Passed` |
| **Tests** | `170 / 170 Passed` |
| **Assertions** | `191 / 191 Passed` |
| **Duration** | `18,291 ms` |
| **HTML Report** | `/home/lex/development/bruno_test_cases_creation/report.html` (`372 KB`) |

### Command Used
```bash
bru run -r --env local --reporter-html report.html
```

---

## 7. Запрос пользователя
> Перепроверь еще раз тест кейсы в /home/lex/development/test_case_creation/specs/test_cases/ - досоздай недостающие сценарии

### Ответ ассистента
Все новые тест-кейсы из `/home/lex/development/test_case_creation/specs/test_cases/` (`TC-AUTH-007`, `TC-USER-011`–`017`, `TC-PROD-011`–`013`, `TC-ORD-011`–`015` — всего **39 новых `.yml` файлов запросов**, итого **103 запроса** по всем **52 тест-кейсам**) добавлены в коллекцию, прогнаны через `bru` CLI (`report.html` обновлен) и запушены в GitHub (`71a7cf7`).

### Добавленные сценарии
* **`auth/`**: `TC-AUTH-007.yml` (bcrypt 72-byte limit).
* **`users/`**: `TC-USER-011` (JWT revocation on profile after soft-delete), `TC-USER-012` (username reuse prevention), `TC-USER-013` (double soft-deletion `404`), `TC-USER-014` (admin self-deletion guard), `TC-USER-015` (username > 100 chars), `TC-USER-016a/b` (`1e18` overflow and `0.015` sub-cent top-up), `TC-USER-017` (whitespace trim and `[""]` rejection).
* **`catalog/`**: `TC-PROD-011` (AND-intersection filtering), `TC-PROD-012a..d` (blank model/category, model > 150 chars, price overflow `1e18`), `TC-PROD-013` (non-paginated envelope hygiene).
* **`orders/`**: `TC-ORD-011` (numeric overflow protection on failed order insertion), `TC-ORD-012` (exact-cent precision $19.99 * 3 = $59.97), `TC-ORD-013` (cancellation of deactivated user's order), `TC-ORD-014` (partial filter match blocked at checkout), `TC-ORD-015` (error precedence order).

### Результаты прогона
* **Всего запросов:** 103 (93 Passed, 10 Failed)
* **Tests:** 215 / 233 Passed
* **Assertions:** 249 / 266 Passed
* **HTML-отчет:** `report.html` (520 KB)

---

## 8. Запрос пользователя
> у нас есть тесткейсы где юзер после удаления пытается запросить каталог и послать ордер?

### Ответ ассистента
В исходной спеке `TC-USER-011` (`user_management.yaml`) после soft-delete проверялся только `GET /api/user/profile` (Step 4).

Добавлены оба недостающих шага в `TC-USER-011`, проверены через `bru` CLI (оба проходят — `404 NOT_FOUND`) и запушены в GitHub (`7221f74`):
1. **`users/TC-USER-011-step5-replay-jwt-on-catalog.yml`** (`GET /api/user/products` с `Bearer {{ephemeral_jwt}}` удаленного пользователя) -> `404 Not Found`.
2. **`users/TC-USER-011-step6-replay-jwt-on-orders.yml`** (`POST /api/user/orders` с `Bearer {{ephemeral_jwt}}` удаленного пользователя) -> `404 Not Found`.

---

## 9. Запрос пользователя
> есть тесткейсы где мы меняем фильтры для юзеров и они применяются корректно?

### Ответ ассистента
В исходной спеке `TC-USER-005` проверялся только ответ самого `PUT /api/admin/users/{id}/filters` (`200 OK`), без проверки того, как измененные фильтры реально влияют на профиль, выдачу каталога и покупку товаров.

Я расширил `TC-USER-005` до полной 6-шаговой сквозной цепочки (все 6 шагов проходят `✓ PASS`, запушены в GitHub `e8a622a`):
1. `users/TC-USER-005-step1-create-user.yml` — создание пользователя с открытыми фильтрами и логин.
2. `users/TC-USER-005-step2-apply-filters.yml` — админ меняет фильтры на `["laptop", "smartphone"]` и `["Apple", "Samsung"]`.
3. `users/TC-USER-005-step3-verify-profile-filters.yml` — проверка обновления фильтров в `GET /api/user/profile`.
4. `users/TC-USER-005-step4-verify-catalog-filtered.yml` — проверка, что в `GET /api/user/products` возвращаются только `Apple`/`Samsung` `laptop`/`smartphone`, а `Dell`/`Xiaomi` исключены.
5. `users/TC-USER-005-step5-verify-disallowed-order-blocked.yml` — попытка купить `Dell XPS 15` блокируется с `422 FILTER_RESTRICTION`.
6. `users/TC-USER-005-step6-update-filters-and-verify-order-allowed.yml` — админ меняет фильтр на `Dell`, и заказ `Dell XPS 15` сразу успешно проходит (`201 Created`, `PROCESSED`).

---

## 10. Запрос пользователя
> Можешь мне сказать почему ты пропустил такие тесткейсы? что добавить в промт, что бы ты не пропускал?

### Ответ ассистента
### Почему эти проверки были пропущены при первом проходе
В исходных файлах `/home/lex/development/test_case_creation/specs/test_cases/*.yaml` было расхождение между декларативным описанием (`title` / `expected_results`) и списком шагов (`steps:`):
* В `TC-ORD-009` в `expected_results` был заявлен возврат денег и стока, но в `steps:` был лишь 1 шаг отмены.
* В `TC-USER-011` в `title` было заявлено блокирование всех клиентских эндпоинтов, но в `steps:` проверялся только профиль.
* В `TC-USER-005` в `steps:` проверялся только код ответа самого запроса обновления фильтров.

При первом проходе я сделал буквальную 1-в-1 трансляцию секции `steps:` из YAML-спеки в файлы Bruno, не проверив, покрывают ли `steps:` все побочные эффекты.

### Что добавить в промпт, чтобы это никогда не пропускалось
```markdown
### Правила полноты тестовых сценариев (End-to-End Side-Effect Verification)
1. Не транслируй steps[] слепо 1-в-1: сверяй steps[] с title, expected_results и TECH_SPEC.md. Если в expected_results или title заявлен бизнес-эффект, которого нет в steps[], обязательно досоздай недостающие HTTP-шаги.
2. Для любой мутации (POST, PUT, PATCH, DELETE) проверяй не только HTTP-ответ самого запроса, но и побочные эффекты (state & side-effect verification) отдельными последующими запросами:
   - При создании/отмене заказа (POST /orders, PATCH /orders/{id}/status) — сначала фиксируй начальный баланс (GET /user/profile) и остаток на складе (GET /admin/products), а после операции проверяй точное списание/возврат денег и уменьшение/возврат стока.
   - При изменении прав/фильтров (PUT /users/{id}/filters) — проверяй применение фильтра end-to-end: (а) обновление в GET /user/profile, (б) фильтрацию выдачи в GET /user/products, (в) блокировку запрещенного товара в POST /user/orders (422 FILTER_RESTRICTION), и (г) успешную покупку (201 Created) после изменения фильтра на разрешающий.
   - При деактивации/удалении аккаунта (DELETE /admin/users/{id}) с активным JWT — проверяй отказ в доступе на всех клиентских эндпоинтах (GET /user/profile, GET /user/products, POST /user/orders, GET /user/orders), а не только на одном.
```

---

## 11. Запрос пользователя
> перепрогони тесты

### Ответ ассистента
Полный прогон всех **109 запросов** завершен, HTML-отчет обновлен (`report.html`, 548 KB).
* **Всего запросов:** 109 (99 Passed, 10 Failed)
* **Tests:** 228 / 246 Passed
* **Assertions:** 267 / 284 Passed
* **Время выполнения:** 30,030 ms

10 упавших запросов выявили реальные дефекты реализации бэкенда (отсутствие валидации пароля > 72 байт, отсутствие гарда от самоудаления админа, ошибки БД 500 при переполнении VARCHAR/NUMERIC, `remaining_balance` omitempty при нуле).

---

## 12. Запрос пользователя
> есть ли тесткейс на создание товара такого же который уже создан

### Ответ ассистента
Нет, такого тест-кейса сейчас нет (ни в `specs/test_cases/catalog/products.yaml`, ни в коллекции Bruno).

В отличие от пользователей (`username UNIQUE NOT NULL`, `409 USERNAME_TAKEN`), для таблицы `products` в схеме БД нет ограничения `UNIQUE(category, manufacturer, model)`. Повторный `POST /api/admin/products` с теми же полями создает новую запись с новым UUID (`201 Created`).

---

## 13. Запрос пользователя
> в изначальных тесткейсах тоже такого нет? /home/lex/development/test_case_creation/specs/test_cases/

### Ответ ассистента
Да, в изначальных тест-кейсах `/home/lex/development/test_case_creation/specs/test_cases/catalog/products.yaml` такого сценария тоже нет (там только `TC-PROD-001` – `TC-PROD-013`). Проверка дубликатов (`409 Conflict`) в исходных спеках присутствует исключительно для `username` пользователей (`TC-USER-002` и `TC-USER-012`).

---

## 14. Запрос пользователя
> а пагинация хоть раз проверяется где?

### Ответ ассистента
Пагинация проверяется в 4 тест-кейсах, но проверка в исходных спеках поверхностная:
1. `TC-PROD-005`: проверяет только `pagination.page == 1`, `pagination.page_size == 10`, `pagination.total_count >= 1` (не проверяет `len <= 10`, реальное смещение `page=1` vs `page=2` и сортировку `price ASC`).
2. `TC-PROD-009a..d`: проверяет ошибки валидации граничных значений (`page=0`, `page_size=0`, `page_size=101`, `sort_by=unknown_col` -> `400`).
3. `TC-USER-007`: передает `page=1&page_size=10`, проверяет только `data.length > 0`.
4. `TC-PROD-013`: проверяет отсутствие поля `pagination` в непагинированном эндпоинте.

---

## 15. Запрос пользователя
> а как ты заходишь под админом, если он удаляется?

### Ответ ассистента
В `users/TC-USER-014.yml` сидированный `admin` (`a0000000-0000-0000-0000-000000000001`) защищен от деактивации:
1. В `before-request` скрипте под основным админом создается временный администратор `self_del_admin_<uuid>` с `role: "admin"`.
2. Выполняется логин под этим временным администратором и получаются его собственные `self_admin_id` и `self_admin_token`.
3. Тест отправляет `DELETE /api/admin/users/{{self_admin_id}}` с токеном `{{self_admin_token}}`, проверяя самоудаление админа.
4. Из-за бага бэкенда удаляется именно этот временный аккаунт, а основной аккаунт `admin` остается активным в PostgreSQL.
