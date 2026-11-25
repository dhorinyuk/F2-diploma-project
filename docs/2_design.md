# Розділ 2. Проєктування архітектури системи

## 2.1. Контекстне та контейнерне моделювання системи (C4 Model)

### Діаграма контексту системи (Level 1 - System Context)

Вхідні потоки даних та актори взаємодіють із платформою відповідно до меж, специфікованих у [SRS (Розділ 1: System Context)](./srs.md#1-system-context-контекст-системи).

```mermaid
C4Context
    title System Context Diagram - Vehicle Condition Analysis & Failure Prediction Platform

    Person(driver, "Vehicle Owner / Fleet Operator", "Користувач системи: моніторить технічний стан авто, ризики відмов та отримує рекомендації")
    Person(service_eng, "Service Engineer / Diagnostician", "Спеціаліст СТО: переглядає параметри датчиків, логи DTC та планує обслуговування")
    
    System(platform, "Vehicle Condition Analysis Platform", "Комплексна платформа: аналізує телеметрію OBD-II, виконує ML-прогнозування збоїв та агрегує стан вузлів")

    System_Ext(telematics_unit, "OBD-II / CAN Telematics Unit", "IoT-модуль / мобільний діагностичний сканер, що транслює кадри телеметрії авто")
    System_Ext(external_crm, "Auto Service CRM / Catalog API", "Сторонній постачальник каталогів запчастин та регламентів ТО")
    System_Ext(oauth_provider, "OAuth 2.0 / OIDC Identity Provider", "Централізований сервер автентифікації та емісії JWT-токенів")

    Rel(driver, platform, "Переглядає метрики зносу, графіки та отримує сповіщення", "HTTPS / Web Browser")
    Rel(service_eng, platform, "Аналізує глибокі логи дефектів та тренди деградації", "HTTPS / Web Browser")
    Rel(telematics_unit, platform, "Передає потокові дані сенсорів та коди помилок DTC", "HTTPS / REST API")
    Rel(platform, external_crm, "Запитує специфікації автокомпонентів та наявність деталей", "HTTPS / REST API")
    Rel(platform, oauth_provider, "Верифікує криптографічні підписи токенів користувачів", "HTTPS / OIDC")
```

#### Технічний опис зв'язків контекстного рівня

* **Клієнтські виклики:** Власники авто та інженери звертаються до платформи через зашифровані канали `HTTPS / TLS 1.3` відповідно до вимоги безпеки [SRS-NFR-SEC-02](./srs.md#3-non-functional-requirements-матриця-нефункціональних-вимог).
* **Телематичний потік:** Апаратні модулі OBD-II транслюють телеметрію по протоколу `HTTPS / REST API` із JSON-корисним навантаженням за контрактом [SRS-FUN-01](./srs.md#2-functional-requirements-функціональні-вимоги).
* **Зовнішні провайдери:** Інтеграція з `OAuth 2.0 / OIDC` виконується через стандартизовані протоколи автентифікації за вимогою [SRS-NFR-SEC-01](./srs.md#3-non-functional-requirements-матриця-нефункціональних-вимог).

---

### Діаграма контейнерів системи (Level 2 - Container Diagram)

Система розділена на окремі виконувані контейнери для ізоляції високонавантаженого потоку запису телеметрії від обчислювального аналізу даних.

```mermaid
C4Container
    title Container Diagram - Vehicle Condition Analysis & Failure Prediction Platform

    Person(user, "User / Service Engineer", "Кінцевий користувач платформи")
    System_Ext(telematics, "OBD-II Telematics Device", "Апаратний клієнт передачі телеметрії")
    System_Ext(crm, "Auto Service CRM API", "Зовнішня система каталогів і сервісних регламентів")

    Container_Boundary(c1, "Vehicle Condition Analysis & Prediction System") {
        Container(spa, "Single Page Application (SPA)", "React / TypeScript", "Інтерфейс користувача: дашборди параметрів сенсорів, діагностичні карти, алерти")
        Container(api_gateway, "API Gateway / Reverse Proxy", "Nginx / Traefik", "Маршрутизація, SSL Termination, Rate Limiting (100 req/min/IP), WAF")
        
        Container(auth_service, "Auth & Identity Service", "FastAPI / Python", "Керування сесіями, автентифікація, валідація JWT та ролей RBAC")
        Container(telemetry_service, "Telemetry Ingestion Service", "FastAPI / Python", "Високопродуктивний шлюз прийому пакетів телеметрії, первинна валідація схем DTO")
        Container(diagnostics_service, "Diagnostics & Prediction Engine", "Python / PyTorch / NumPy", "Сервіс інференсу ML-моделей, агрегації трендів та дерева рішень DTC")
        
        Container(msg_broker, "Message Broker", "RabbitMQ / AMQP", "Асинхронний буфер подій телеметрії між прийомом та важким ML-аналізом")
        ContainerDb(timeseries_db, "Telemetry & Metadata Storage", "PostgreSQL + TimescaleDB", "Зберігання реляційних сутностей (Users, Cars) та гіпертаблиць часових рядів сенсорів")
        ContainerDb(cache_db, "In-Memory Cache & States", "Redis", "Кешування останнього стану авто, сесійних ключів та лімітів Rate Limiter")
    }

    Rel(user, spa, "Взаємодіє з вебінтерфейсом", "HTTPS")
    Rel(telematics, api_gateway, "Надсилає пакети телеметрії [SRS-FUN-01]", "JSON / HTTPS")
    Rel(spa, api_gateway, "Виконує API-виклики", "JSON / HTTPS")

    Rel(api_gateway, auth_service, "Перенаправляє запити автентифікації", "REST / HTTPS")
    Rel(api_gateway, telemetry_service, "Проксує вхідні потоки датчиків", "REST / HTTPS")
    Rel(api_gateway, diagnostics_service, "Запитує аналітичні звіти та скоринг", "REST / HTTPS")

    Rel(telemetry_service, msg_broker, "Публікує сирі пакети телеметрії в чергу telemetry.raw", "AMQP / TCP")
    Rel(msg_broker, diagnostics_service, "Вичитує пакети пачками (Batch Consume)", "AMQP / TCP")
    
    Rel(telemetry_service, timeseries_db, "Записує часові ряди в гіпертаблиці [SRS-FUN-02]", "SQL / TCP")
    Rel(diagnostics_service, timeseries_db, "Вибирає історичні зрізи та записує результати ML [SRS-FUN-04]", "SQL / TCP")
    
    Rel(diagnostics_service, cache_db, "Оновлює актуальний скоринг авто та кешує аномалії", "RESP / TCP")
    Rel(auth_service, cache_db, "Зберігає статус відкликаних токенів (Blacklist)", "RESP / TCP")

    Rel(diagnostics_service, crm, "Отримує сервісні регламенти та коди заміни вузлів", "REST / HTTPS")
```

#### Технічний опис контейнерів та протоколів

* **API Gateway (Nginx/Traefik):** Периметровий вузол безпеки, що виконує SSL Termination, захист від DDoS через Rate Limiting (100 req/min/IP) та балансування навантаження, забезпечуючи виконання [SRS-NFR-PERF-01](./srs.md#3-non-functional-requirements-матриця-нефункціональних-вимог).
* **Telemetry Ingestion Service:** Асинхронний сервіс на базі `FastAPI`, призначений для високошвидкісного прийому пакетів телеметрії, валідації `JSON Schema` [SRS-FUN-01](./srs.md#2-functional-requirements-функціональні-вимоги) та паралельного запису в чергу `RabbitMQ`.
* **Message Broker (RabbitMQ):** Декуплює вхідний потік від обчислювального блоку аналітики за протоколом `AMQP / TCP`, що гарантує відсутність втрати телеметрії при сплесках трафіку.
* **База даних TimescaleDB:** Реалізує гібридне колоночне сховище за архітектурним рішенням [ADR-001](./adr/0001-choice-of-technology.md) та вимогою [SRS-FUN-02](./srs.md#2-functional-requirements-функціональні-вимоги) через протокол `SQL / TCP`.

---

## 2.2. Компонентне моделювання критичного сервісу системи (C4 Model: Component)

Для поглибленої декомпозиції обрано ключовий аналітичний сервіс бекенду: **Diagnostics & Failure Prediction Engine** (`FastAPI` / `Python` / `PyTorch`).

```mermaid
C4Component
    title Component Diagram - Diagnostics & Failure Prediction Engine

    Container(api_gw, "API Gateway", "Nginx / Reverse Proxy", "Шлюз вхідного трафіку системи")
    Container(broker, "Message Broker", "RabbitMQ", "Черга вхідних телематичних пакетів")
    ContainerDb(ts_db, "TimescaleDB", "PostgreSQL", "Сховище часових рядів та доменних сутностей")
    ContainerDb(redis_cache, "Redis Cache", "In-Memory", "Гарячий кеш станів вузлів")

    Container_Boundary(engine, "Diagnostics & Failure Prediction Engine") {
        Component(jwt_guard, "JWT & Scope Guard", "FastAPI Dependency / Security", "Перевіряє криптографічний підпис токена та права доступу ролі [SRS-NFR-SEC-01]")
        Component(err_handler, "Global Exception Handler", "FastAPI Middleware", "Перехоплює помилки домену та трансформує їх у RFC 7807 Problem Details")
        Component(metrics, "Prometheus Telemetry Collector", "Prometheus Client", "Збирає лічильники викликів, помилок та квантилі часу інференсу")
        
        Component(api_controller, "Diagnostics REST Controller", "FastAPI APIRouter", "Обробляє зовнішні запити на отримання звітів та скорингу авто")
        Component(amqp_consumer, "Telemetry Stream Consumer", "Aio-pika / Task Worker", "Вичитує пачки кадрів телеметрії з RabbitMQ у неблокуючому режимі")
        
        Component(dto_validator, "DTO Validation & Normalizer", "Pydantic v2", "Здійснює строгу типізацію, очищення та відсіювання некоректних сенсорних викидів")
        Component(prediction_coord, "Prediction Coordinator", "Domain Service", "Координує бізнес-сценарій: оркеструє завантаження вікна даних, ML-аналіз та генерацію рекомендацій")
        Component(dtc_analyzer, "DTC Fault Tree Engine", "Domain Service", "Зіставляє коди помилок за графом залежностей систем автомобіля (DTC Graph) [SRS-FUN-03]")
        Component(ml_inference, "Failure Prediction Model Runner", "PyTorch / ONNX Runtime", "Виконує прямий прохід (inference) моделі виявлення деградації за часовими вікнами [SRS-FUN-04]")
        
        Component(ts_repo, "Telemetry Data Repository", "SQLAlchemy 2.0 (Async)", "Шар доступу до часових рядів сенсорів та збереження розрахованого ризику")
        Component(cache_client, "State Cache Client", "Redis-py (Async)", "Адаптер збереження обчислених скоринг-станів авто у кеші")
    }

    Rel(api_gw, jwt_guard, "Передає HTTP-запити на аналітику", "REST / HTTPS")
    Rel(jwt_guard, api_controller, "Пропускає валідований контекст виклику", "Internal Call")
    Rel(api_controller, err_handler, "Охоплюється перехоплювачем збоїв", "AOP / Interceptor")
    Rel(api_controller, metrics, "Реєструє HTTP-метрики", "Internal In-Memory")

    Rel(broker, amqp_consumer, "Доставляє повідомлення telemetry.raw", "AMQP / TCP")
    Rel(amqp_consumer, dto_validator, "Передає сирі батчі даних", "Internal Call")
    Rel(dto_validator, prediction_coord, "Передає валідовані доменні сутності", "Internal Call")

    Rel(api_controller, prediction_coord, "Викликає розрахунок або читання стану", "Dependency Injection")
    
    Rel(prediction_coord, dtc_analyzer, "Ініціює аналіз ланцюгів несправностей", "Method Call")
    Rel(prediction_coord, ml_inference, "Передає нормалізований тензор часового вікна", "Memory Tensor Pointer")
    Rel(prediction_coord, ts_repo, "Отримує ретроспективне вікно сенсорів та пише прогноз", "Async Interface")
    Rel(prediction_coord, cache_client, "Кешує актуальний індекс здоров'я авто", "Async Interface")

    Rel(ts_repo, ts_db, "Виконує оптимізовані агрегаційні SQL-запити", "Asyncpg / TCP")
    Rel(cache_client, redis_cache, "Записує розраховані структури станів", "RESP / TCP")
```

### Технічний опис компонентів та принципів Clean Architecture

* **Вхідний шар (Inbound Layer):** `JWT & Scope Guard` перевіряє права доступу [SRS-NFR-SEC-01](./srs.md#3-non-functional-requirements-матриця-нефункціональних-вимог) до виклику бізнес-методів. `Telemetry Stream Consumer` вичитує події з черги за допомогою асинхронного клієнта `aio-pika`.
* **Шар доменної логіки (Domain Core):**
  * `DTO Validation & Normalizer` здійснює відсіювання шумів сенсорів через `Pydantic v2` під контракт [SRS-FUN-01](./srs.md#2-functional-requirements-функціональні-вимоги).
  * `Prediction Coordinator` оркеструє бізнес-процес без прямої прив'язки до баз даних (патерн *Dependency Inversion*).
  * `Failure Prediction Model Runner` оптимізовано через середовище `ONNX Runtime`, що забезпечує виконання вимоги інференсу $\le 50$ мс [SRS-NFR-PERF-03](./srs.md#3-non-functional-requirements-матриця-нефункціональних-вимог).
* **Шар інфраструктури (Outbound Layer):** `Telemetry Data Repository` реалізує патерн `Repository` через асинхронний драйвер `asyncpg`, ізолюючи формування SQL-запитів до TimescaleDB згідно з [ADR-001](./adr/0001-choice-of-technology.md).

---

## 2.3. Документування архітектурних рішень (Architecture Decision Records)

* [ADR-001: Вибір архітектури сховища даних для часових рядів телеметрії та діагностичних подій](./adr/0001-choice-of-technology.md) — **Status: Approved**.

---

## 2.4. Проєктування та моделювання баз даних

Схема зберігання даних узгоджена з [ADR-001](./adr/0001-choice-of-technology.md) та вимогами [SRS](./srs.md) для сховища `PostgreSQL + TimescaleDB`[cite: 3, 4, 5].

```mermaid
erDiagram
    USERS {
        uuid id PK
        varchar email UK
        varchar password_hash
        varchar role
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    VEHICLES {
        uuid id PK
        uuid user_id FK
        varchar vin UK
        varchar make
        varchar model
        smallint year
        int mileage_km
        int version
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    TELEMETRY_RECORDS {
        uuid id PK
        uuid vehicle_id FK
        varchar idempotency_key UK
        varchar dtc_code
        numeric rpm
        numeric coolant_temp_c
        jsonb sensor_payload
        timestamptz recorded_at
        timestamptz created_at
    }
    DIAGNOSTIC_PREDICTIONS {
        uuid id PK
        uuid vehicle_id FK
        uuid telemetry_id FK
        varchar status
        numeric anomaly_score
        numeric failure_risk_score
        jsonb model_meta
        timestamptz created_at
    }
    MAINTENANCE_LOGS {
        uuid id PK
        uuid vehicle_id FK
        varchar service_type
        int odometer_km
        bigint cost_minor
        char currency_code
        timestamptz performed_at
        timestamptz created_at
    }
    USERS ||--o{ VEHICLES : owns
    VEHICLES ||--o{ TELEMETRY_RECORDS : emits
    VEHICLES ||--o{ DIAGNOSTIC_PREDICTIONS : has
    TELEMETRY_RECORDS ||--o{ DIAGNOSTIC_PREDICTIONS : triggers
    VEHICLES ||--o{ MAINTENANCE_LOGS : logs
```

* **Індекси та цілісність:** складені `B-Tree` індекси на `TELEMETRY_RECORDS(vehicle_id, recorded_at DESC)` та `DIAGNOSTIC_PREDICTIONS(vehicle_id, status, created_at DESC)`, часткові індекси за `deleted_at IS NULL`, `GIN` на поля `jsonb` (`sensor_payload`, `model_meta`), унікальний `idempotency_key` для ідемпотентної обробки повідомлень.

## 2.5. Специфікація міжсервісних контрактів (API)

Контракт регламентує взаємодію клієнта / OBD-II пристрою через `API Gateway` (`Nginx`) та `API Service` (`FastAPI`) із `Telemetry Ingestion Service` і `Diagnostics & Failure Prediction Engine` (див. [2_design.md](./2_design.md), [SRS](./srs.md) та [ADR-001](./adr/0001-choice-of-technology.md))[cite: 3, 4, 5].

Деталізовану специфікацію ендпоінтів у форматі OpenAPI 3.0 зафіксовано у файлі [openapi.yaml](./api/openapi.yaml).

## 2.6. Моделювання логіки критичних вузлів

### Алгоритм 1. Прийом і перевірка масиву показників телеметрії Samples[i]

1. Вхід: масив `Samples[0..N-1]`, ліміт спроб `MaxRetries := 3`.
2. Старт: `i := 0`, `ValidCount := 0`, `Status := "PROCESSING"`, `HasError := FALSE`.
3. Якщо `(Samples = NULL) OR (N <= 0)`: `Status := "FAILED"`, кінець.
4. Поки `(i < N) AND (HasError = FALSE)`:
   * Якщо `(Samples[i].RPM < 0) OR (Samples[i].RPM > 10000) OR (Samples[i].CoolantTempC < -50) OR (Samples[i].CoolantTempC > 160)`: `HasError := TRUE`, `Status := "FAILED"`.
   * Інакше: `Retries := 0`, `Saved := FALSE`; поки `(Retries < MaxRetries) AND (NOT Saved)`: `Retries := Retries + 1`; якщо збережено або дублікат уже є за `IdempotencyKey`: `Saved := TRUE`, `ValidCount := ValidCount + 1`. Якщо `NOT Saved`: `HasError := TRUE`, `Status := "FAILED"`.
   * `i := i + 1`.
5. Якщо `HasError = FALSE`: `Status := "COMPLETED"`. Повернути `Status`, `ValidCount`.

```mermaid
graph TD
    A(["Початок"]) --> B["i := 0, ValidCount := 0, Status := PROCESSING, HasError := FALSE"]
    B --> C{"(Samples = NULL) OR (N <= 0)?"}
    C -- "ТАК" --> D["Status := FAILED"]
    D --> Z(["Кінець"])
    C -- "НІ" --> E{"(i < N) AND (HasError = FALSE)?"}
    E -- "НІ" --> F{"HasError = FALSE?"}
    F -- "ТАК" --> G["Status := COMPLETED"]
    F -- "НІ" --> D
    G --> Z
    E -- "ТАК" --> H{"(Samples[i].RPM < 0) OR (Samples[i].RPM > 10000) OR (Samples[i].CoolantTempC < -50) OR (Samples[i].CoolantTempC > 160)?"}
    H -- "ТАК" --> I["HasError := TRUE, Status := FAILED"]
    I --> J["i := i + 1"]
    J --> E
    H -- "НІ" --> K["Retries := 0, Saved := FALSE"]
    K --> L{"(Retries < MaxRetries) AND (NOT Saved)?"}
    L -- "ТАК" --> M["Retries := Retries + 1"]
    M --> N{"SavedOrDuplicateOK?"}
    N -- "ТАК" --> O["Saved := TRUE, ValidCount := ValidCount + 1"]
    O --> L
    N -- "НІ" --> L
    L -- "НІ" --> P{"Saved = FALSE?"}
    P -- "ТАК" --> I
    P -- "НІ" --> J
```

### Алгоритм 2. Підрахунок ризику несправності за масивом ознак Features[j]

1. Вхід: масив ознак `Features[0..M-1]`, поріг ризику `RiskThreshold := 0.75`, ліміт спроб `MaxAttempts := 3`.
2. Старт: `j := 0`, `Attempt := 0`, `ScoreSum := 0.0`, `ModelReady := FALSE`, `Status := "PROCESSING"`.
3. Якщо `(Features = NULL) OR (M <= 0)`: `Status := "FAILED"`, кінець.
4. Поки `(Attempt < MaxAttempts) AND (NOT ModelReady)`: `Attempt := Attempt + 1`; якщо сервіс аналітики доступний: `ModelReady := TRUE`.
5. Якщо `ModelReady = FALSE`: `Status := "FAILED"`, кінець.
6. Поки `j < M`: `ScoreSum := ScoreSum + Features[j].Weight * Features[j].NormalizedValue`; `j := j + 1`.
7. `RiskScore := ScoreSum / M`; якщо `RiskScore >= RiskThreshold`: `AlertRequired := TRUE`, інакше `AlertRequired := FALSE`; `Status := "COMPLETED"`. Повернути `Status`, `RiskScore`, `AlertRequired`.

```mermaid
graph TD
    M1(["Початок"]) --> M2["j := 0, Attempt := 0, ScoreSum := 0.0, ModelReady := FALSE, Status := PROCESSING"]
    M2 --> M3{"(Features = NULL) OR (M <= 0)?"}
    M3 -- "ТАК" --> M4["Status := FAILED"]
    M4 --> M13(["Кінець"])
    M3 -- "НІ" --> M5{"(Attempt < MaxAttempts) AND (NOT ModelReady)?"}
    M5 -- "ТАК" --> M6["Attempt := Attempt + 1"]
    M6 --> M7{"ModelServiceOK = TRUE?"}
    M7 -- "ТАК" --> M8["ModelReady := TRUE"]
    M8 --> M5
    M7 -- "НІ" --> M5
    M5 -- "НІ" --> M9{"ModelReady = FALSE?"}
    M9 -- "ТАК" --> M4
    M9 -- "НІ" --> M10{"j < M?"}
    M10 -- "ТАК" --> M11["ScoreSum := ScoreSum + Features[j].Weight * Features[j].NormalizedValue, j := j + 1"]
    M11 --> M10
    M10 -- "НІ" --> M12["RiskScore := ScoreSum / M, Status := COMPLETED"]
    M12 --> M13
```

## 2.7. Інтеграційний аудит та верифікація цілісності доменів

* Типи ідентифікаторів (`uuid`), часові мітки (`timestamptz` / `date-time`), ключі ідемпотентності (`idempotency_key` / `idempotencyKey`), параметри `rpm` та `coolant_temp_c` / `coolantTempC` і стани `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED` узгоджені між схемою БД, [openapi.yaml](./api/openapi.yaml) та алгоритмами з розділу 2.6.
* Критичних конфліктів інтеграції або блокерів не виявлено; специфікації готові до реалізації.
