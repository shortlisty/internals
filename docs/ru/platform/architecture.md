# Venue Intelligence Platform (BENE) — Справочник по архитектуре

> **Аудитория:** Инженеры, архитекторы.
> **Назначение:** Единый источник истины для всех технических решений до начала разработки.

**Документы:** [Что такое BENE?](../README.md) · [Бизнес-обзор](../business/proposal.md) · [Конкурентная среда](../business/comparison.md) · [Архитектура](architecture.md)

---

## 1. Контекст платформы

BENE Intelligence — это новый продуктовый сервис, построенный **поверх фундамента IQ Key Value**. Он не заменяет и не форкает ни один существующий сервис. Он повторно использует:

| Фундаментальный сервис       | Что наследует BENE                                                                                    |
| ---------------------------- | ----------------------------------------------------------------------------------------------------- |
| `foundation-gateway-service` | Валидация JWT, маршрутизация по тенантам, распространение заголовков — изменения не нужны             |
| `foundation-iam-service`     | Авторизация, мультитенантность, приглашения в команду, SSO, паттерн подписанных URL для загрузки в S3 |
| `foundation-billing-service` | Полномочия тарифа (`max_venues`, `ai_extraction_enabled` и т.д.), жизненный цикл подписки             |
| `foundation-audit-service`   | Журнал соответствия — пассивно потребляет события BENE, изменения кода не нужны                       |
| `foundation-ui-app`          | Расширен (не форкнут) новыми маршрутами `/venues/*` в рамках архитектуры FSD                          |
| `foundation-tenancy`         | Библиотека изоляции схема-на-тенант используется напрямую                                             |

**Новые сервисы, вводимые BENE Intelligence:**

- `iqbene-venue-model` — общая библиотека (JAR). Каноническая доменная модель, контракты событий, перечисления и миграции Liquibase. Нет Spring-бин, нет бизнес-логики — чистая модель и схема. Импортируется обоими сервисами.
- `iqbene-venue-service` — основная доменная логика: площадки, активы, метаданные, поиск, применение правил тарифа, поиск в реестре площадок. Только синхронные запрос/ответ.
- `iqbene-venue-ingestion-worker` — асинхронный сайдкар: ETL-конвейер документов, оркестрация извлечения, генерация эмбеддингов, сопоставление с реестром, плановые задания. Нет входящего HTTP — только событийно-управляемый. Использует ту же схему PostgreSQL, что и `iqbene-venue-service`.

**Новая инфраструктура, вводимая BENE Intelligence:**

- Расширение pgvector в существующей PostgreSQL (не новый сервис)
- Расширение PostGIS в существующей PostgreSQL (не новый сервис)
- IBM Docling (опциональный самохостящийся контейнер, только Фаза 2)

---

## 2. Доменная модель

### Ограниченные контексты

#### `venue/` — Основной профиль

**Корень агрегата: `Venue`**

| Поле                     | Тип              | Примечания                                          |
| ------------------------ | ---------------- | --------------------------------------------------- |
| `id`                     | UUID             | PK                                                  |
| `name`                   | varchar(255)     |                                                     |
| `address`                | text             |                                                     |
| `location`               | geography(point) | PostGIS, широта/долгота                             |
| `description`            | text             | Написано человеком или составлено ИИ                |
| `status`                 | enum             | `DRAFT`, `ACTIVE`, `ARCHIVED`                       |
| `metadata`               | jsonb            | Объединённые извлечённые + ручные поля              |
| `metadata_sources`       | jsonb            | Происхождение по каждому полю (см. §4)              |
| `metadata_aggregated_at` | timestamp        | Когда последний раз запускалось объединение         |
| `description_embedding`  | vector(1536)     | pgvector, для семантического поиска                 |
| `description_text`       | tsvector         | Автообновляется через триггер, полнотекстовый поиск |
| `created_by`             | UUID             | Идентификатор пользователя IAM                      |
| `created_at`             | timestamp        |                                                     |
| `updated_at`             | timestamp        |                                                     |

**Операции:** создать, обновить, архивировать, восстановить.

---

#### `asset/` — Файловые вложения

**Корень агрегата: `VenueAsset`**

| Поле                       | Тип          | Примечания                                                                   |
| -------------------------- | ------------ | ---------------------------------------------------------------------------- |
| `id`                       | UUID         | PK                                                                           |
| `venue_id`                 | UUID         | FK → venues                                                                  |
| `asset_type`               | enum         | `PDF_DECK`, `FLOOR_PLAN`, `PHOTO`, `VIDEO`, `CAD_FILE`, `SPEC_SHEET`, `MISC` |
| `file_name`                | varchar(255) |                                                                              |
| `content_type`             | varchar(100) | MIME-тип                                                                     |
| `size_bytes`               | bigint       |                                                                              |
| `s3_key`                   | text         | Путь хранения                                                                |
| `extracted_text`           | text         | Сырой текст, извлечённый парсером                                            |
| `extracted_text_embedding` | vector(1536) | pgvector, поиск на уровне чанков                                             |
| `extraction_status`        | enum         | `PENDING`, `IN_PROGRESS`, `COMPLETED`, `FAILED`                              |
| `uploaded_by`              | UUID         |                                                                              |
| `uploaded_at`              | timestamp    |                                                                              |

**Поток загрузки:** двухфазный по подписанному URL (тот же паттерн, что и загрузка аватара в IAM).

1. `POST /assets/initiate` → возвращает подписанный S3 PUT URL + `asset_id`
2. Клиент загружает напрямую в S3
3. `POST /assets/{id}/confirm` → помечает актив готовым, публикует событие `asset.uploaded`

---

#### `extraction/` — Задания обработки ИИ

**Корень агрегата: `ExtractionJob`**

| Поле                | Тип       | Примечания                                    |
| ------------------- | --------- | --------------------------------------------- |
| `id`                | UUID      | PK                                            |
| `asset_id`          | UUID      | FK → venue_assets                             |
| `status`            | enum      | `QUEUED`, `PROCESSING`, `COMPLETED`, `FAILED` |
| `extractor_type`    | enum      | `TIKA_TEXT`, `GPT4O_DOCUMENT`, `GPT4O_VISION` |
| `extracted_data`    | jsonb     | Сырой результат извлечения                    |
| `confidence_scores` | jsonb     | Уверенность по каждому полю (0.0–1.0)         |
| `started_at`        | timestamp |                                               |
| `completed_at`      | timestamp |                                               |
| `error_message`     | text      | При сбое                                      |

---

#### `metadata_events/` — Журнал событий (для агрегации)

**Не корень агрегата — журнал событий только на добавление.**

| Поле          | Тип       | Примечания                                          |
| ------------- | --------- | --------------------------------------------------- |
| `id`          | UUID      | PK                                                  |
| `venue_id`    | UUID      | FK → venues                                         |
| `event_type`  | enum      | `ASSET_EXTRACTED`, `MANUAL_OVERRIDE`, `BULK_IMPORT` |
| `source_type` | enum      | `PDF_DECK`, `FLOOR_PLAN`, `PHOTO`, `USER_INPUT`     |
| `source_id`   | UUID      | asset_id или user_id                                |
| `event_data`  | jsonb     | Поля со значениями и оценками уверенности           |
| `occurred_at` | timestamp |                                                     |
| `created_by`  | UUID      |                                                     |

---

#### `registry/` — Реестр площадок платформы

**Не принадлежит тенанту. Живёт в схеме `public`. Только для чтения тенантами.**

Реестр площадок — это справочный датасет уровня платформы, растущий каталог известных площадок, засеянный на этапе разработки и обогащаемый со временем. Это не источник истины; это отправная точка. Данные тенанта всегда побеждают данные реестра.

**`VenueRegistryEntry`**

| Поле           | Тип              | Примечания                                    |
| -------------- | ---------------- | --------------------------------------------- |
| `id`           | UUID             | PK                                            |
| `name`         | varchar(255)     |                                               |
| `address`      | text             |                                               |
| `city`         | varchar(100)     |                                               |
| `country_code` | varchar(2)       | ISO 3166-1 alpha-2                            |
| `location`     | geography(point) | PostGIS, широта/долгота                       |
| `metadata`     | jsonb            | Та же форма полей, что и `venues.metadata`    |
| `confidence`   | numeric(3,2)     | Общая оценка качества 0.0–1.0                 |
| `source`       | varchar(50)      | `platform_seed`, `web_scrape`, `admin_import` |
| `created_at`   | timestamp        |                                               |
| `updated_at`   | timestamp        |                                               |

**`VenueRegistryAlias`** — альтернативные названия одной и той же площадки (например, "The Bowery Hotel" / "Bowery Hotel NYC"):

| Поле                      | Тип          | Примечания          |
| ------------------------- | ------------ | ------------------- |
| `id`                      | UUID         | PK                  |
| `venue_registry_entry_id` | UUID         | FK → venue_registry |
| `alias`                   | varchar(255) |                     |

---

#### `venue_groups/` — Организация библиотеки тенанта (Фаза 2)

**Принадлежит тенанту. Живёт в схеме `t_{tenantKey}`.**

Организаторы мероприятий группируют свою библиотеку площадок по городу, типу мероприятия, клиенту, сезону или любой другой таксономии. Группы — это исключительно UI/навигационная концепция; они не влияют на поиск, извлечение или метаданные.

Площадки привязываются к группам через связующую таблицу `venue_group_members(venue_id, group_id)`. Площадка может принадлежать нескольким группам. Это функция Фазы 2 — таблица `venue_groups` не включена в миграции Liquibase Фазы 1 (MVP).

---

Столбец `venues.metadata` хранит **объединённое представление** всех извлечённых и введённых вручную данных. Столбец `venues.metadata_sources` хранит **происхождение** по каждому полю.

**Канонический набор полей:**

```
capacity
  └─ max_total (int)
  └─ configurations: banquet, theater, classroom, cocktail, conference (int each)

venue_type (string[])         например ["conference_center", "hotel_ballroom"]

location_notes (string)       например "в 3 кварталах от Grand Central"

catering
  └─ policy (enum)            in_house_exclusive | in_house_preferred | outside_allowed | no_catering
  └─ kosher_available (bool)
  └─ halal_available (bool)

av_tech
  └─ built_in_av (bool)
  └─ projector_lumens (int)
  └─ screens (int)
  └─ rigging_points (bool)
  └─ internet_bandwidth_mbps (int)

accessibility
  └─ ada_compliant (bool)
  └─ elevator_access (bool)
  └─ accessible_restrooms (bool)
  └─ wheelchair_stage_access (bool)

logistics
  └─ load_in_access (string)
  └─ parking_spaces (int)
  └─ valet_available (bool)
  └─ curfew_time (time)

restrictions (string[])       например ["no_open_flame", "no_confetti"]

amenities (string[])          например ["WiFi", "AV_equipment", "parking"]

contacts (object[])
  └─ name, role, email, phone

pricing
  └─ minimum_spend (int)
  └─ currency (string)
  └─ rental_fee_indicative (int)
```

**Происхождение по каждому полю** (хранится в `metadata_sources`):

```json
"capacity.max_total": {
  "value": 500,
  "confidence": 0.94,
  "source_type": "PDF_DECK",
  "source_id": "<asset-uuid>",
  "updated_at": "...",
  "alternatives": [
    { "value": 480, "confidence": 0.65, "source_type": "FLOOR_PLAN" }
  ]
}
```

---

## 3. Агрегация метаданных

Несколько активов на площадку порождают несколько событий извлечения, потенциально с конфликтующими значениями. Сервис агрегации разрешает конфликты и поддерживает объединённый столбец `metadata`.

### Приоритет разрешения конфликтов

```
MANUAL_OVERRIDE     → всегда побеждает (пользователь явно задал значение)
VERIFIED_EXTRACTION → администратор подтвердил результат ИИ
HIGH_CONFIDENCE_AI  → уверенность ≥ 0.9
MEDIUM_CONFIDENCE_AI→ уверенность 0.7–0.9
LOW_CONFIDENCE_AI   → уверенность < 0.7
REGISTRY            → данные реестра платформы (наименьший приоритет — только заполняет пробелы)
```

### Поля-массивы (удобства, ограничения)

Объединение множеств по всем источникам. Запись включается, если хотя бы один источник сообщает о ней с уверенностью ≥ 0.6.

### Точки запуска

Агрегация запускается (асинхронно, через RabbitMQ), когда:

- Событие `asset.uploaded` запускает извлечение → извлечение завершается → `extraction.completed` запускает агрегацию
- Пользователь отправляет ручную перезапись → немедленная повторная агрегация
- Плановое задание обрабатывает устаревшие площадки (24 ч без повторной агрегации)

Агрегация дебаунсится (5 с) для пакетной обработки быстрых последовательных событий.

---

## 4. Архитектура сервисов

```
                    ┌──────────────────┐
  Браузер/Прилож ──► │  Gateway Service │  (существующий)
                    └────────┬─────────┘
                             │ JWT проверен, установлен X-Tenant-ID
                    ┌────────▼─────────┐     ┌──────────────────┐
                    │   IAM Service    │     │  Billing Service │  (существующий)
                    └────────┬─────────┘     └────────┬─────────┘
                             │                        │ полномочия тарифа
                    ┌────────▼────────────────────────▼─────────┐
                    │            iqbene-venue-service             │
                    │  площадки · активы · метаданные · поиск · api │
                    └────────┬──────────────┬──────────┬─────────┘
                             │ RabbitMQ:    │ ч/з       │ подписанный URL
                             │ asset.uploa- │           │ выдача + удаление
                    ┌────────▼─────────┐  ┌─▼──────────▼──────┐
                    │ iqbene-ingestion- │  │   PostgreSQL        │
                    │    worker        │──►│   t_{tenant}        │
                    │ (async sidecar)  │  │   + pgvector        │
                    └────────┬─────────┘  │   + PostGIS         │
                             │ match      └─────────────────────┘
                             │ реестра    ┌─────────────────────┐
                             ├───────────►│   public schema      │
                             │            │   venue_registry     │
                             │            └─────────────────────┘
                             │ чтение     ┌─────────────────────┐
                             └───────────►│   S3 / MinIO         │◄── клиент (прямой PUT)
                                          │   vip/tenants/{key}/ │
                                          │   vip/registry/      │
                                          └─────────────────────┘
```

### iqbene-venue-service

- **Обязанности:** CRUD площадок, поток загрузки активов (подписанный URL), чтение/запись метаданных, API поиска, применение полномочий тарифа, поиск в реестре площадок
- **База данных:** владеет схемой BENE в PostgreSQL. Тенантность на уровне схем через `foundation-tenancy` — каждый тенант получает свою схему `t_{tenantKey}`. Нет столбца `tenant_id` ни в одной таблице; маршрутизация по схеме обрабатывается `MyBatisSchemaInterceptor`. Общая с `iqbene-venue-ingestion-worker` — без кросс-сервисных API-вызовов для данных.
- **Предоставляет:** REST API на `/api/v1/venues`
- **Публикует:** `venue.created`, `venue.updated`, `asset.uploaded`, `asset.deleted` (RabbitMQ)
- **Потребляет:** `extraction.completed`, `extraction.failed` (RabbitMQ) — запускает агрегацию метаданных

### iqbene-venue-ingestion-worker

- **Обязанности:** ETL-конвейер документов (парсинг → чанкинг → извлечение → эмбеддинг), жизненный цикл заданий извлечения, сопоставление с реестром и заполнение пробелов, агрегация метаданных, плановые обслуживающие задания (обновление устаревшей агрегации, отчёты по стоимости)
- **Природа:** асинхронный сайдкар — нет входящего HTTP, нет REST API, нет записи в service discovery. Только событийно-управляемый.
- **База данных:** общая схема PostgreSQL с `iqbene-venue-service`. Читает `venue_assets`, пишет `extraction_jobs`, `venue_metadata_events`, `venue_vectors`, `ai_cost_tracking`. Также читает `public.venue_registry` для шага сопоставления с реестром.
- **Потребляет:** `asset.uploaded` (RabbitMQ) — запускает ETL-конвейер
- **Публикует:** `extraction.started`, `extraction.completed`, `extraction.failed` (RabbitMQ)
- **Внешние вызовы:** OpenAI API (GPT-4o, text-embedding-3-small), опционально сайдкар Docling (Фаза 2)
- **Масштабирование:** реплики масштабируются независимо на основе глубины очереди RabbitMQ — без влияния на `iqbene-venue-service`

### Владение таблицами

Оба сервиса используют одну схему PostgreSQL. Владение определяет, кто может писать в таблицу. Кросс-граничные чтения разрешены; кросс-граничные записи — нет.

| Таблица                 | Владелец                        | Другой сервис может…                                                                |
| ----------------------- | ------------------------------- | ----------------------------------------------------------------------------------- |
| `venues`                | `iqbene-venue-service`          | читать (ingestion-worker: только разрешать venue_id)                                |
| `venue_assets`          | `iqbene-venue-service`          | читать (ingestion-worker: получать актив для обработки)                             |
| `venue_metadata_events` | `iqbene-venue-service`          | писать через реакцию на событие (`extraction.completed` → venue-service агрегирует) |
| `extraction_jobs`       | `iqbene-venue-ingestion-worker` | читать (venue-service: показывать статус задания в API)                             |
| `venue_vectors`         | `iqbene-venue-ingestion-worker` | читать (venue-service: векторные поисковые запросы)                                 |
| `ai_cost_tracking`      | `iqbene-venue-ingestion-worker` | читать (venue-service: показывать сводку по стоимости в API)                        |

Единственное законное кросс-граничное чтение из `iqbene-venue-ingestion-worker` — это `SELECT` по `venue_assets` по `asset_id` (переданному в полезной нагрузке события `asset.uploaded`). Это поиск по внешнему ключу, не бизнес-логика — допустимо и намеренно.

---

## 4a. Общая библиотека — iqbene-venue-model

`iqbene-venue-model` — это обычная Java-библиотека (JAR, без Spring Boot, без `@SpringBootApplication`). И `iqbene-venue-service`, и `iqbene-venue-ingestion-worker` объявляют её как compile-зависимость. Это единый источник истины для всего, по чему оба сервиса должны договориться.

**Содержимое:**

```
iqbene-venue-model/
├── model/
│   ├── venue/
│   │   ├── Venue.java                  Простой POJO (корень агрегата, без JPA-аннотаций)
│   │   ├── VenueStatus.java            enum: DRAFT, ACTIVE, ARCHIVED
│   │   └── VenueAsset.java             Простой POJO
│   ├── asset/
│   │   ├── AssetType.java              enum: PDF_DECK, FLOOR_PLAN, PHOTO, CAD_FILE…
│   │   └── ExtractionStatus.java       enum: PENDING, IN_PROGRESS, COMPLETED, FAILED
│   ├── extraction/
│   │   ├── ExtractionJob.java          Простой POJO
│   │   ├── ExtractorType.java          enum: TIKA_TEXT, GPT4O_DOCUMENT, GPT4O_VISION
│   │   └── VenueMetadataEvent.java     Простой POJO (журнал событий только на добавление)
│   ├── metadata/
│   │   ├── VenueMetadata.java          Value object (зеркалирует venues.metadata JSONB)
│   │   ├── VenueCapacity.java          Value object конфигураций вместимости
│   │   ├── MetadataSource.java         Происхождение по каждому полю
│   │   └── MetadataEventType.java      enum: ASSET_EXTRACTED, MANUAL_OVERRIDE, BULK_IMPORT, REGISTRY
│   ├── registry/
│   │   ├── VenueRegistryEntry.java     Простой POJO — запись реестра платформы (public schema)
│   │   └── VenueRegistryAlias.java     Простой POJO — альтернативные названия для записей реестра
│   └── events/                         RabbitMQ-контракты сообщений (POJO, без зависимостей от фреймворка)
│       ├── AssetUploadedEvent.java
│       ├── ExtractionStartedEvent.java
│       ├── ExtractionCompletedEvent.java
│       └── ExtractionFailedEvent.java
└── db/
    └── changelog/
        ├── system/                     Миграции системной (public) схемы
        │   ├── master.xml
        │   └── 20250101000000-create-venue-registry.xml
        └── tenant/                     Миграции схемы тенанта — единый источник истины
            ├── master.xml
            ├── 20250101000001-create-venues.xml
            ├── 20250101000002-create-venue-assets.xml
            ├── 20250101000003-create-extraction-jobs.xml
            ├── 20250101000004-create-metadata-events.xml
            ├── 20250101000005-create-venue-vectors.xml
            └── 20250101000006-create-ai-cost-tracking.xml
```

**Правила:**

- Никаких `@Service`, `@Repository`, `@Component` или любых аннотаций Spring-бин
- Никакой бизнес-логики — только простые доменные классы, value objects, перечисления, event POJO
- Никаких JPA-аннотаций — платформа использует MyBatis, не JPA. Сущности — простые POJO, не `@Entity`-классы. ORM-аннотации в этой библиотеке недопустимы.
- Миграции Liquibase живут здесь, поэтому изменения схемы — это повышение версии compile-зависимости, а не упражнение по координации между сервисами
- Изменение поля в event POJO — это разрыв во время компиляции в обоих сервисах — намеренно, предотвращает тихое смещение контрактов

**Граф зависимостей:**

```
iqbene-venue-model  (библиотека, без среды выполнения)
      ├── iqbene-venue-service     (Spring Boot, импортирует model)
      └── iqbene-venue-ingestion-worker  (Spring Boot, импортирует model)
```

---

## 4б. Структура хранилища S3

S3 (MinIO для локальной разработки) уже входит в стек IQ Key Value. BENE добавляет собственное пространство имён с префиксом внутри общего бакета (`iqkv-files`) — новый бакет в dev/staging не нужен. В production можно выделить отдельный бакет (`iqkv-vip-files`) одним изменением конфигурации; структура ключей при этом не меняется.

---

### Стратегия бакета

| Среда      | Бакет            | Примечания                                                                                               |
| ---------- | ---------------- | -------------------------------------------------------------------------------------------------------- |
| Dev / CI   | `iqkv-files`     | Общий с сервисами фундамента, MinIO по умолчанию. VIP-объекты под префиксом `vip/`.                      |
| Staging    | `iqkv-files`     | Тот же общий бакет, та же схема префиксов. Изоляция только по префиксу.                                  |
| Production | `iqkv-vip-files` | Отдельный бакет. Отдельная IAM-политика, отдельные правила жизненного цикла. Структура ключей идентична. |

MinIO в локальной разработке настраивается в `docker-compose.yml` через `MINIO_DEFAULT_BUCKETS=iqkv-files`. Бакет `iqkv-vip-files` не нужен до введения production-конфигурации развёртывания.

---

### Соглашение об именовании ключей

Все VIP-объекты следуют детерминированной иерархической структуре ключей. Каждый сегмент — строчными буквами, без пробелов.

#### Файлы активов тенантов (загружаются пользователями тенанта)

```
vip/tenants/{tenantKey}/venues/{venueId}/assets/{assetId}/{fileName}
```

| Сегмент       | Значение                                                       | Пример                             |
| ------------- | -------------------------------------------------------------- | ---------------------------------- |
| `vip/`        | Пространство имён VIP — отделяет от других объектов фундамента | (литерал)                          |
| `tenants/`    | Корень поддерева тенантов                                      | (литерал)                          |
| `{tenantKey}` | 8-символьный nanoid из JWT-клейма `tenant_id`                  | `acme0001`                         |
| `venues/`     | Поддерево площадок                                             | (литерал)                          |
| `{venueId}`   | UUID площадки (без дефисов — компактная форма)                 | `550e8400e29b41d4a716446655440000` |
| `assets/`     | Поддерево активов                                              | (литерал)                          |
| `{assetId}`   | UUID актива (без дефисов)                                      | `6ba7b8109dad11d180b400c04fd430c8` |
| `{fileName}`  | Оригинальное имя файла, URL-безопасное, макс. 255 символов     | `grand-ballroom-deck.pdf`          |

Полный пример:

```
vip/tenants/acme0001/venues/550e8400e29b41d4a716446655440000/assets/6ba7b8109dad11d180b400c04fd430c8/grand-ballroom-deck.pdf
```

**Правила именования ключей:**

- `{fileName}` — оригинальное имя файла, поступившее от клиента, очищенное от разделителей путей (`/`, `\`) и URL-кодированное при необходимости. Хранится как есть после санитизации — без замены UUID — чтобы ключ оставался человекочитаемым в консоли MinIO / S3 CLI.
- `{venueId}` и `{assetId}` — UUID без дефисов (компактный 32-символьный hex). Это сокращает длину ключей и избегает проблем двойного кодирования.
- `asset_id` генерируется на стороне сервера при `POST /assets/initiate` и записывается в `venue_assets.id` до выдачи подписанного URL. S3-ключ вычисляется из этого ID и сохраняется в `venue_assets.s3_key`. При подтверждении повторное вычисление ключа не происходит — используется сохранённый `s3_key` напрямую.

#### Выдача подписанных URL

Ответ `POST /assets/initiate`:

```json
{
  "asset_id": "<uuid>",
  "upload_url": "https://minio.local/iqkv-files/vip/tenants/acme0001/venues/.../grand-ballroom-deck.pdf?X-Amz-Signature=...",
  "expires_at": "2025-06-01T12:15:00Z"
}
```

Подписанный PUT URL формируется на стороне сервера с точным ключом. Клиент загружает напрямую. Сервис никогда не проксирует тело файла.

Подписанные URL для скачивания (TTL 1 ч) генерируются по запросу в `GET /assets/{id}/download-url` — не сохраняются. Ключ всегда берётся из `venue_assets.s3_key`.

---

#### Файлы импорта реестра платформы (администраторские / посевные)

Посевные данные реестра и файлы массового импорта не принадлежат тенантам. Они живут в отдельном поддереве:

```
vip/registry/imports/{importId}/{fileName}
vip/registry/exports/{date}/{snapshot}.jsonl.gz
```

| Путь                               | Назначение                                                                             |
| ---------------------------------- | -------------------------------------------------------------------------------------- |
| `vip/registry/imports/{importId}/` | Одна папка на батч импорта (инициируется администратором). Содержит исходные CSV/JSON. |
| `vip/registry/exports/{date}/`     | Ночные компактированные снапшоты `public.venue_registry` для downstream-потребителей.  |

`{importId}` — UUID, генерируемый при инициации импорта. `{date}` — `YYYY-MM-DD`.

Файлы импорта реестра обрабатываются плановым административным заданием (без входящего HTTP — администратор загружает файлы через Registry Admin API, Фаза 2). Задание импорта читает из S3, наполняет `public.venue_registry` и `public.venue_registry_aliases`, затем архивирует исходный объект, перемещая его в `vip/registry/imports/processed/{importId}/`.

---

### Изоляция тенантов

Изоляция данных тенантов в S3 зеркалирует подход схема-на-тенанта в PostgreSQL:

- Все объекты тенанта ограничены областью `vip/tenants/{tenantKey}/`. Кросс-тенантное чтение структурно невозможно без знания ключа другого тенанта.
- Сервисный аккаунт, используемый `iqbene-venue-service` и `iqbene-venue-ingestion-worker`, имеет единственную S3 IAM-политику, разрешающую `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` на полный префикс `vip/*`. Подписанные URL ограничены точным ключом объекта — клиент не может перечислять или получать доступ к любому другому ключу.
- Пути реестра (`vip/registry/*`) недоступны через подписанные URL, выданные тенантам. В них пишет только сервисный аккаунт внутреннего планового задания платформы.

---

### Правила жизненного цикла и компактирование

Правила жизненного цикла S3 настраиваются на бакете (не в коде приложения). Применяются три правила:

| Правило                               | Префикс                            | Действие                                                                                                                                                                                                          |
| ------------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Истечение срока артефактов извлечения | `vip/tenants/*/venues/*/assets/*/` | Переход в Glacier/IA после 90 дней, если `extraction_status = COMPLETED` и нет ожидающего повторного извлечения. Управляется через теги объекта, установленные при подтверждении (`extraction_status=completed`). |
| Очистка обработанных импортов         | `vip/registry/imports/processed/`  | Удалить через 30 дней.                                                                                                                                                                                            |
| Ротация снапшотов реестра             | `vip/registry/exports/`            | Хранить последние 14 ежедневных снапшотов; удалять старые.                                                                                                                                                        |

Теги объектов устанавливаются `iqbene-venue-service` при `POST /assets/confirm` через `PutObjectTagging`:

| Ключ тега           | Значения                                        | Устанавливается                            |
| ------------------- | ----------------------------------------------- | ------------------------------------------ |
| `extraction_status` | `pending`, `completed`, `failed`                | venue-service при подтверждении/обновлении |
| `asset_type`        | `pdf_deck`, `floor_plan`, `photo`, `cad_file` … | venue-service при инициации                |
| `tenant_key`        | 8-символьный nanoid                             | venue-service при инициации                |

Теги позволяют формировать отчёты по распределению затрат по тенанту и типу актива в AWS Cost Explorer / MinIO billing.

---

### Каскад удаления

Когда тенант удаляет актив (`DELETE /assets/{id}`) или когда аккаунт тенанта закрывается:

1. `iqbene-venue-service` удаляет запись `venue_assets` (каскад БД удаляет связанные extraction_jobs и metadata_events).
2. `iqbene-venue-service` выполняет `s3:DeleteObject` для `venue_assets.s3_key`.
3. Публикуется событие `asset.deleted` → `iqbene-venue-ingestion-worker` удаляет все строки `venue_vectors`, где `metadata->>'asset_id' = :assetId`.

При полном удалении тенанта (право на удаление по GDPR):

1. `DELETE FROM t_{tenantKey}.venues` каскадно удаляет все строки активов.
2. Отдельное событие `tenant.deleted` запускает фоновую очистку S3: `s3:DeleteObjects` для всех ключей по маске `vip/tenants/{tenantKey}/*` (батчами по 1000 объектов в соответствии с лимитами S3 API).
3. Очистка pgvector удаляет все строки `venue_vectors` для схемы тенанта (сброс схемы обрабатывает это неявно при удалении схемы).

---

### Стратегия наполнения реестра

Таблица `public.venue_registry` — это канонический справочник площадок платформы. Она никогда не наполняется загрузками тенантов. Пути наполнения:

| Путь                              | Механизм                                                                                                                                                           | Источник в `venue_registry.source` |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------- |
| **Холодный старт (посев)**        | Загрузка CSV администратором → S3 `vip/registry/imports/{id}/` → задание импорта → INSERT                                                                          | `platform_seed`                    |
| **Импорт результатов скрейпинга** | Конвейер скрейпинга создаёт JSONL → S3 `vip/registry/imports/{id}/` → задание импорта                                                                              | `web_scrape`                       |
| **Ручной импорт администратором** | Registry Admin API (Фаза 2): `POST /admin/registry/import` → подписанный S3 PUT → задание импорта                                                                  | `admin_import`                     |
| **Обогащение сигналами тенантов** | После extraction.completed, если данные тенанта имеют поля с высокой уверенностью, отсутствующие в реестре → событие-кандидат (Фаза 3, обратного потока в MVP нет) | — (не в MVP)                       |

**Шаги задания импорта** (выполняется `iqbene-venue-ingestion-worker` по плановому триггеру или по административному событию RabbitMQ):

1. Перечислить объекты в `vip/registry/imports/{importId}/`.
2. Разобрать каждый файл (CSV или JSONL). Каждая строка должна содержать минимум: `name`, `address`, `country_code`.
3. Для каждой строки: попытаться дедублировать по имени + PostGIS-близости относительно существующих записей `venue_registry`. Порог дедубликации: сходство имён ≥ 0.85 (триграммы) И местоположение в радиусе 100 м (если предоставлены широта/долгота). При совпадении → UPDATE (слияние непустых полей, повышение `confidence`, добавление новых алиасов). При отсутствии совпадения → INSERT.
4. Нормализация алиасов: убрать артикли ("The ", "A "), привести к нижнему регистру, убрать пунктуацию. Сохранить и оригинальное название, и нормализованную форму как строки `venue_registry_aliases`. Это позволяет делать нечёткое совпадение, когда тенант загружает документ со ссылкой "Bowery Hotel NYC", а запись в реестре — "The Bowery Hotel".
5. Переместить обработанный исходный файл в `vip/registry/imports/processed/{importId}/`.
6. Добавить запись в очередь ночного экспорта. Ночное задание создания снапшотов компактирует все строки реестра в `vip/registry/exports/{date}/venue_registry_snapshot.jsonl.gz` для внешних потребителей (партнёрский API Фазы 3).

**Стратегия алиасов для сопоставления с реестром при извлечении:**

Когда `VenueRegistryMatcher` запускается после извлечения документа тенанта:

1. Первичное совпадение: `venues.name` против `venue_registry.name` + `venue_registry_aliases.alias` (сходство триграмм, GIN-индекс по `alias`).
2. Вторичное совпадение: PostGIS `ST_DWithin(venues.location, venue_registry.location, 200)` — радиус 200 м.
3. Комбинированная уверенность: `0.6 * name_similarity + 0.4 * (1 если geo_match иначе 0)`. Порог: ≥ 0.7 запускает заполнение пробелов.
4. При совпадении: скопировать поля реестра в `venues.metadata` для любого поля, ещё не заполненного при извлечении. Каждое скопированное поле помечается `source_type = REGISTRY` в `metadata_sources`. Это одноразовое копирование — запись тенанта с этого момента полностью независима.

---

## 5. ETL-конвейер (iqbene-venue-ingestion-worker)

Построен на **ETL-фреймворке Spring AI**. Три компонуемых стадии:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
  (парсинг)       (чанкинг + обогащение)   (эмбеддинг + сохранение)
```

### Стадия 1 — Парсинг (по типу актива)

| Тип актива                     | Ридер                                        | Примечания                                      |
| ------------------------------ | -------------------------------------------- | ----------------------------------------------- |
| PDF (текстовый)                | `TikaDocumentReader`                         | Apache Tika, поставляется с Spring AI           |
| PDF (отсканированный)          | `TikaDocumentReader` + Tesseract OCR         | Tika включает OCR                               |
| PDF (сложная вёрстка, таблицы) | Сайдкар Docling → кастомный `DocumentReader` | Фаза 2; лучшая точность таблиц                  |
| DOCX / XLSX / PPTX             | `TikaDocumentReader`                         | Тот же ридер, 1000+ форматов                    |
| Изображения (JPG, PNG)         | Прямой GPT-4o vision                         | Текстовый ридер не нужен                        |
| План этажа (PDF/изображение)   | Docling layout-aware → GPT-4o vision         | Фаза 2                                          |
| DWG / DXF (CAD)                | `TikaDocumentReader` (AutoCAD-парсер)        | Извлекает метаданные; визуально в Фазе 2        |
| Видео                          | Вне скоупа Фазы 1                            | Фаза 2: извлечение ключевых кадров через ffmpeg |

### Стадия 2 — Преобразование

1. **Чанкинг** — `TokenTextSplitter` (512 токенов, перекрытие 50 токенов). Для таблиц спецификаций используем чанки по 256 токенов для сохранения точности строк.
2. **Тегирование** — прикрепляем `venue_id`, `asset_id`, `asset_type`, `tenant_id` как метаданные Document.
3. **Извлечение** — `VenueMetadataEnricher` (кастомный `DocumentTransformer`): вызывает GPT-4o со схемой структурированного вывода, возвращает POJO `VenueMetadata` с оценками уверенности по каждому полю.

### Стадия 3 — Загрузка

1. **Эмбеддинг** — `EmbeddingModel` (`text-embedding-3-small`, 1536 измерений).
2. **Сохранение** — `TenantAwarePgVectorStore` пишет чанки + эмбеддинги в таблицу `venue_vectors` в схеме тенанта.
3. **Сопоставление с реестром** — `VenueRegistryMatcher` запрашивает `public.venue_registry` по схожести имени + PostGIS-близости. При совпадении выше порога уверенности, поля метаданных реестра копируются в запись тенанта для незаполненных полей. Источник помечается как `REGISTRY` в `metadata_sources`. Копирование, не связывание — запись тенанта с этого момента независима.
4. **Агрегация** — публикует событие `extraction.completed` → `MetadataAggregationConsumer` обновляет `venues.metadata`.

### SLA обработки

| Тип актива                 | Целевая задержка |
| -------------------------- | ---------------- |
| PDF / DOCX (текст)         | < 30 с           |
| Изображения / планы этажей | < 60 с           |
| CAD-файлы                  | < 2 мин          |

Повтор при сбое: 3 попытки с экспоненциальной задержкой. После 3 сбоев → событие `extraction.failed` → уведомление пользователя.

---

## 6. Архитектура поиска

Весь поиск обслуживается `iqbene-venue-service`, делающим запросы напрямую в PostgreSQL. Нет отдельного поискового сервиса.

### Режимы поиска

| Режим                     | Реализация                                           | Сценарий использования                                |
| ------------------------- | ---------------------------------------------------- | ----------------------------------------------------- |
| Ключевой                  | `tsvector` / `to_tsquery` (GIN-индекс)               | "конференция центр AV"                                |
| Структурированные фильтры | Запросы JSONB (GIN-индекс)                           | вместимость ≥ 200, удобства включают WiFi             |
| Семантический             | Косинусное расстояние pgvector (IVFFlat-индекс)      | "современное лофт-пространство с естественным светом" |
| Геопространственный       | PostGIS `ST_DWithin` (GIST-индекс)                   | площадки в радиусе 10 миль от почтового индекса       |
| Гибридный                 | Reciprocal Rank Fusion по ключевому + семантическому | Стандартная поисковая строка                          |

### Стратегия векторного индекса

- Тип индекса: IVFFlat (быстрое построение, хорошее восстановление до ~5M векторов)
- Расстояние: косинусное
- Измерения: 1536
- Область: схема на тенанта (нет межтенантных утечек)
- Путь обновления: переключиться на HNSW, когда у тенанта превысит ~500K площадок (редко)

---

## 7. Поверхность API (iqbene-venue-service)

Все эндпоинты следуют платформенным соглашениям на основе фактической реализации в `foundation-cms-service` и `foundation-iam-service`:

- Базовый путь: `/api/v1/venues` — всегда версионирован
- Все эндпоинты требуют Bearer JWT — публичных маршрутов нет
- `POST` (создание) → `201 Created` с телом ресурса
- `GET` (чтение) → `200 OK`
- `PUT` (полная замена) → `200 OK`
- `PATCH` (частичное обновление) → `200 OK`
- `DELETE` → `204 No Content`
- Постраничные ответы: кастомные record-обёртки (например, `VenueSummaryListResponse(items, totalElements)`) — не Spring `Page<T>`
- Ответы об ошибках: RFC 7807 `ProblemDetail`, `type` = `about:blank`, включает расширенные свойства `correlationId` и `requestId`
- Строки полномочий — без префикса: `USER`, `ADMIN`, `TENANT_OWNER` (никогда не с префиксом `ROLE_`)
- Контекст тенанта устанавливается автоматически через `TenantExtractionFilter` из JWT-клейма `tenant_id` — переменная пути тенанта в обычных эндпоинтах не нужна

---

### Площадки

Базовый путь: `/api/v1/venues`

| Метод    | Путь    | Полномочие              | Статус | Запрос / Ответ                                                                     | Примечания                                                                           |
| -------- | ------- | ----------------------- | ------ | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `GET`    | `/`     | `MEMBER`                | 200    | Параметры: `page`, `size`, `sort`, `status`, `search` → `VenueSummaryListResponse` | Гибридный поиск + фильтр при наличии параметра `search`                              |
| `POST`   | `/`     | `MEMBER`                | 201    | `CreateVenueRequest` → `VenueResponse`                                             | Применяет лимит тарифа `max_venues` перед вставкой                                   |
| `GET`    | `/{id}` | `MEMBER`                | 200    | → `VenueResponse` (с объединёнными метаданными)                                    | 404 если не найдено или принадлежит другому тенанту                                  |
| `PATCH`  | `/{id}` | `MEMBER`                | 200    | `UpdateVenueRequest` → `VenueResponse`                                             | Частичное обновление — игнорирует null-поля                                          |
| `DELETE` | `/{id}` | `ADMIN`, `TENANT_OWNER` | 204    | → пустое тело                                                                      | Мягкое удаление: `status = ARCHIVED`. Жёсткое удаление — отдельный эндпоинт (Фаза 3) |

#### Поиск

Поиск предоставляется через эндпоинт списка (`GET /`) с параметрами запроса — отдельный `POST /search` при данном масштабе не нужен.

| Параметр     | Тип            | Описание                                                                |
| ------------ | -------------- | ----------------------------------------------------------------------- |
| `search`     | string         | Запрос на естественном языке или по ключевому слову — гибридный режим   |
| `status`     | enum           | `DRAFT`, `ACTIVE`, `ARCHIVED`                                           |
| `capacity`   | integer        | Минимальная общая вместимость                                           |
| `lat`, `lng` | decimal        | Центральная точка для геопространственного поиска                       |
| `radius_km`  | decimal        | Радиус от lat/lng (требует задания обоих)                               |
| `amenities`  | string[]       | Коды обязательных удобств (через запятую)                               |
| `catering`   | enum           | Фильтр по политике кейтеринга                                           |
| `page`       | integer (от 0) | По умолчанию: 0                                                         |
| `size`       | integer        | По умолчанию: 20, макс.: 100                                            |
| `sort`       | string         | например `name,asc` или `relevance` (по умолчанию при наличии `search`) |

#### DTO

```
CreateVenueRequest  — name (обязательно), address, description, tags
UpdateVenueRequest  — все поля опциональны; null-поля игнорируются (PATCH-семантика)
VenueResponse       — id, name, address, location, status, metadata (объединённые),
                       metadata_aggregated_at, asset_count, created_by, created_at, updated_at
```

---

### Активы

Базовый путь: `/api/v1/venues/{venueId}/assets`

Загрузка использует двухфазный паттерн подписанного URL (тот же, что и загрузка аватара в IAM — без multipart на сервис).

| Метод    | Путь            | Полномочие              | Статус | Запрос / Ответ                                     | Примечания                                                    |
| -------- | --------------- | ----------------------- | ------ | -------------------------------------------------- | ------------------------------------------------------------- |
| `GET`    | `/`             | `MEMBER`                | 200    | → `List<AssetResponse>`                            | Все активы площадки; без пагинации (разумная верхняя граница) |
| `POST`   | `/initiate`     | `MEMBER`                | 201    | `InitiateUploadRequest` → `InitiateUploadResponse` | Возвращает `asset_id` + подписанный S3 PUT URL (TTL 15 мин)   |
| `POST`   | `/{id}/confirm` | `MEMBER`                | 200    | → `AssetResponse`                                  | Помечает актив готовым, публикует событие `asset.uploaded`    |
| `DELETE` | `/{id}`         | `ADMIN`, `TENANT_OWNER` | 204    | → пустое тело                                      | Удаляет запись актива + S3-объект + связанные векторы         |

Применяет лимит тарифа `max_assets_per_venue` при `POST /initiate`.

#### DTO

```
InitiateUploadRequest  — file_name (обязательно), content_type (обязательно), size_bytes (обязательно),
                          asset_type (обязательно: PDF_DECK | FLOOR_PLAN | PHOTO | VIDEO | CAD_FILE | SPEC_SHEET | MISC)
InitiateUploadResponse — asset_id (UUID), upload_url (подписанный S3 PUT, 15 мин), expires_at
AssetResponse          — id, venue_id, asset_type, file_name, content_type, size_bytes,
                          extraction_status, uploaded_by, uploaded_at
```

Проверка плана: функция `cad_support` проверяется при `POST /initiate`, когда `asset_type = CAD_FILE`. Возвращает `403 Forbidden` с `ProblemDetail` (функция недоступна в текущем тарифе).

---

### Метаданные

Базовый путь: `/api/v1/venues/{venueId}/metadata`

| Метод   | Путь               | Полномочие | Статус | Запрос / Ответ                                      | Примечания                                                                                  |
| ------- | ------------------ | ---------- | ------ | --------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `GET`   | `/`                | `MEMBER`   | 200    | → `MetadataResponse` (объединённые + происхождение) | Включает оценки уверенности и атрибуцию источника                                           |
| `PATCH` | `/{field}`         | `MEMBER`   | 200    | `MetadataOverrideRequest` → `MetadataFieldResponse` | Ручная перезапись — устанавливает source = `MANUAL_OVERRIDE`, запускает повторную агрегацию |
| `GET`   | `/{field}/history` | `MEMBER`   | 200    | → `List<MetadataEventResponse>`                     | Полная история извлечений + перезаписей для одного поля                                     |

#### DTO

```
MetadataResponse        — fields (маппинг поле → значение + уверенность + источник), aggregated_at,
                           conflict_count (int: поля с неразрешёнными конкурирующими значениями)
MetadataOverrideRequest — value (обязательно), reason (опциональный свободный текст)
MetadataFieldResponse   — field, value, confidence, source_type, source_id, updated_at
MetadataEventResponse   — event_type, source_type, source_id, value, confidence, occurred_at, created_by
```

---

### Задания извлечения

Базовый путь: `/api/v1/venues/{venueId}/extractions`

Только для чтения клиентами API. Задания создаются внутри при потреблении `asset.uploaded`.

| Метод | Путь       | Полномочие | Статус | Ответ                         | Примечания                                             |
| ----- | ---------- | ---------- | ------ | ----------------------------- | ------------------------------------------------------ |
| `GET` | `/`        | `MEMBER`   | 200    | `List<ExtractionJobResponse>` | Все задания площадки, упорядочены по `started_at DESC` |
| `GET` | `/{jobId}` | `MEMBER`   | 200    | `ExtractionJobResponse`       | Статус задания + извлечённые данные (если завершено)   |

#### DTO

```
ExtractionJobResponse — id, asset_id, status, extractor_type, confidence_scores (маппинг),
                         started_at, completed_at, error_message
```

---

### Ответы об ошибках

Все ошибки используют Spring `ProblemDetail` (RFC 7807). Поле `type` — `about:blank` для всех ошибок. Каждый ответ об ошибке включает `correlationId` и `requestId` как расширенные свойства.

| Сценарий                                 | Статус | Title                   |
| ---------------------------------------- | ------ | ----------------------- |
| Площадка не найдена                      | 404    | `Not Found`             |
| Актив не найден                          | 404    | `Not Found`             |
| Достигнута квота площадок (лимит тарифа) | 402    | `Plan upgrade required` |
| Достигнута квота активов (лимит тарифа)  | 402    | `Plan upgrade required` |
| Функция недоступна в тарифе (напр. CAD)  | 403    | `Plan upgrade required` |
| Ошибка валидации                         | 400    | `Validation Failed`     |
| Отсутствует контекст тенанта             | 400    | `Bad Request`           |
| Доступ запрещён                          | 403    | `Forbidden`             |
| Токен отозван / истёк                    | 401    | `Unauthorized`          |

---

## 8. Контракты событий (RabbitMQ)

Обменник: `iqkv.events` (Topic) — тот же обменник, что используют все фундаментальные сервисы.

### Публикуется iqbene-venue-service

| Ключ маршрутизации | Поля полезной нагрузки                                          | Описание                              |
| ------------------ | --------------------------------------------------------------- | ------------------------------------- |
| `venue.created`    | venue_id, tenant_id, created_by                                 | Новый профиль площадки создан         |
| `venue.updated`    | venue_id, tenant_id, changed_fields                             | Поля площадки обновлены               |
| `asset.uploaded`   | asset_id, venue_id, tenant_id, asset_type, s3_key, content_type | Актив подтверждён, готов к извлечению |
| `asset.deleted`    | asset_id, venue_id, tenant_id                                   | Актив удалён                          |

### Публикуется iqbene-venue-ingestion-worker

| Ключ маршрутизации     | Поля полезной нагрузки                        | Описание              |
| ---------------------- | --------------------------------------------- | --------------------- |
| `extraction.started`   | job_id, asset_id, venue_id, tenant_id         | Обработка началась    |
| `extraction.completed` | job_id, asset_id, venue_id, tenant_id         | Извлечение удалось    |
| `extraction.failed`    | job_id, asset_id, venue_id, tenant_id, reason | Все повторы исчерпаны |

### Потребляется iqbene-venue-ingestion-worker

| Ключ маршрутизации | Очередь                                   | Действие                                     |
| ------------------ | ----------------------------------------- | -------------------------------------------- |
| `asset.uploaded`   | `iqbene.extraction.priority` (Enterprise) | Запустить ETL-конвейер немедленно            |
| `asset.uploaded`   | `iqbene.extraction.standard` (Free/Pro)   | Запустить ETL-конвейер (стандартная очередь) |

### Потребляется iqbene-venue-service

| Ключ маршрутизации     | Очередь                       | Действие                                    |
| ---------------------- | ----------------------------- | ------------------------------------------- |
| `extraction.completed` | `iqbene.metadata.aggregation` | Запустить агрегацию метаданных для площадки |
| `extraction.failed`    | `iqbene.extraction.dlq`       | Пометить extraction_status актива = FAILED  |

### Потребляется foundation-audit-service (пассивно, без изменений)

| Ключ маршрутизации                   | Примечания                                                    |
| ------------------------------------ | ------------------------------------------------------------- |
| `venue.#`, `asset.#`, `extraction.#` | Автоматически захватывается wildcard-привязкой сервиса аудита |

---

## 9. Сопоставление полномочий тарифа

Коды функций, используемые в конфигурации планов `foundation-billing-service`:

| Код функции              | Free | Pro | Enterprise | Точка применения                                  |
| ------------------------ | ---- | --- | ---------- | ------------------------------------------------- |
| `max_venues`             | 10   | 500 | unlimited  | venue-service: перед созданием                    |
| `max_assets_per_venue`   | 20   | 100 | unlimited  | venue-service: перед загрузкой                    |
| `basic_extraction`       | ✅   | ✅  | ✅         | ai-service: только текст PDF                      |
| `advanced_extraction`    | ⛔   | ✅  | ✅         | ai-service: все типы активов                      |
| `cad_support`            | ⛔   | ✅  | ✅         | venue-service: отклонять загрузку DWG/DXF         |
| `semantic_search`        | ⛔   | ✅  | ✅         | venue-service: поисковая конечная точка           |
| `priority_ai_processing` | ⛔   | ⛔  | ✅         | RabbitMQ: маршрутизировать в приоритетную очередь |
| `api_access`             | ⛔   | ✅  | ✅         | gateway: маршрут API-ключа                        |
| `white_label`            | ⛔   | ⛔  | ✅         | ui-app: конфигурация брендинга                    |

Применение через `PlanFeatureGuard` (тот же паттерн, что и существующая реализация сервиса IAM).

---

## 10. Схема базы данных (Liquibase, схема тенанта)

Миграции живут в `iqbene-venue-model` под `src/main/resources/db/changelog/tenant/` — общая библиотека является единым источником истины для схемы. И `iqbene-venue-service`, и `iqbene-venue-ingestion-worker` включают библиотеку в свой classpath; `iqbene-venue-service` запускает миграции при старте (или выделенный init-контейнер применяет их при предоставлении тенанта через слушатель `TenantProvisionedEvent`, тот же паттерн, что и в IAM).

### Соглашения об именовании и формате

Следует руководству по кодированию платформы (§16):

- Формат: **только XML** — никаких SQL-скриптов, никакого YAML
- Именование файлов: `YYYYMMDDhhmmss-description.xml` (временно́й префикс + kebab-case описание)
- ID changeset: совпадает с именем файла без расширения `.xml`
- Автор: `iqkv`
- Каждый changeset должен включать блок `<rollback>`
- Никогда не изменять существующий changeset — добавлять новый

### Файлы миграций

```
db/changelog/system/
├── master.xml
└── 20250101000000-create-venue-registry.xml   ← public-схема, принадлежит платформе

db/changelog/tenant/
├── master.xml
├── 20250101000001-create-venues.xml
├── 20250101000002-create-venue-assets.xml
├── 20250101000003-create-extraction-jobs.xml
├── 20250101000004-create-metadata-events.xml
├── 20250101000005-create-venue-vectors.xml
└── 20250101000006-create-ai-cost-tracking.xml
```

`TenantLiquibaseRunner` из `foundation-tenancy` применяет `system/master.xml` первым (к схеме `public`), затем `tenant/master.xml` к каждой схеме `t_{tenantKey}`. Таблицы `venue_registry` и `venue_registry_aliases` создаются в `public` один раз, не per-tenant.

### Структура changeset (пример: таблица venues)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.33.xsd">

  <changeSet id="20250101000001-create-venues" author="iqkv">

    <!-- Расширения применяются один раз на схему тенанта -->
    <sql>CREATE EXTENSION IF NOT EXISTS vector;</sql>
    <sql>CREATE EXTENSION IF NOT EXISTS postgis;</sql>

    <createTable tableName="venues">
      <column name="id" type="UUID">
        <constraints primaryKey="true" nullable="false"/>
      </column>
      <column name="name" type="VARCHAR(255)">
        <constraints nullable="false"/>
      </column>
      <column name="address" type="TEXT"/>
      <column name="location" type="GEOGRAPHY(POINT, 4326)"/>
      <column name="description" type="TEXT"/>
      <column name="description_embedding" type="VECTOR(1536)"/>
      <column name="description_text" type="TSVECTOR"/>
      <column name="status" type="VARCHAR(20)" defaultValue="DRAFT">
        <constraints nullable="false"/>
      </column>
      <column name="metadata" type="JSONB" defaultValue="{}">
        <constraints nullable="false"/>
      </column>
      <column name="metadata_sources" type="JSONB" defaultValue="{}">
        <constraints nullable="false"/>
      </column>
      <column name="metadata_aggregated_at" type="TIMESTAMP"/>
      <column name="created_by" type="UUID">
        <constraints nullable="false"/>
      </column>
      <column name="created_at" type="TIMESTAMP" defaultValueComputed="NOW()">
        <constraints nullable="false"/>
      </column>
      <column name="updated_at" type="TIMESTAMP" defaultValueComputed="NOW()">
        <constraints nullable="false"/>
      </column>
    </createTable>

    <createIndex tableName="venues" indexName="idx_venues_embedding" using="ivfflat">
      <column name="description_embedding vector_cosine_ops"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_fts" using="gin">
      <column name="description_text"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_metadata" using="gin">
      <column name="metadata jsonb_path_ops"/>
    </createIndex>
    <createIndex tableName="venues" indexName="idx_venues_location" using="gist">
      <column name="location"/>
    </createIndex>

    <sql>
      CREATE TRIGGER trg_venues_tsvector BEFORE INSERT OR UPDATE ON venues
        FOR EACH ROW EXECUTE FUNCTION
        tsvector_update_trigger(description_text, 'pg_catalog.english', name, description, address);
    </sql>

    <rollback>
      <sql>DROP TRIGGER IF EXISTS trg_venues_tsvector ON venues;</sql>
      <dropIndex tableName="venues" indexName="idx_venues_location"/>
      <dropIndex tableName="venues" indexName="idx_venues_metadata"/>
      <dropIndex tableName="venues" indexName="idx_venues_fts"/>
      <dropIndex tableName="venues" indexName="idx_venues_embedding"/>
      <dropTable tableName="venues"/>
    </rollback>

  </changeSet>
</databaseChangeLog>
```

### Обзор схемы (все таблицы)

**Public-схема (принадлежит платформе, не привязана к тенантам):**

| Таблица                  | Ключевые столбцы                                                                                                                                                | Примечания                                              |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `venue_registry`         | `id` UUID PK, `name` VARCHAR(255), `address` TEXT, `city` VARCHAR(100), `location` GEOGRAPHY, `metadata` JSONB, `confidence` NUMERIC(3,2), `source` VARCHAR(50) | Посевные данные платформы. Только для чтения тенантами. |
| `venue_registry_aliases` | `id` UUID PK, `venue_registry_entry_id` UUID FK, `alias` VARCHAR(255)                                                                                           | Альтернативные названия для дедупликации реестра        |

**Схема тенанта `t_{tenantKey}` (принадлежит тенанту):**

| Таблица                 | Ключевые столбцы                                                                                                                        | Владелец                        |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- |
| `venues`                | `id` UUID PK, `status` VARCHAR(20), `metadata` JSONB, `description_embedding` VECTOR(1536), `location` GEOGRAPHY                        | `iqbene-venue-service`          |
| `venue_assets`          | `id` UUID PK, `venue_id` UUID FK, `asset_type` VARCHAR(50), `extraction_status` VARCHAR(20), `extracted_text_embedding` VECTOR(1536)    | `iqbene-venue-service`          |
| `extraction_jobs`       | `id` UUID PK, `asset_id` UUID FK, `status` VARCHAR(20), `extractor_type` VARCHAR(50), `extracted_data` JSONB, `confidence_scores` JSONB | `iqbene-venue-ingestion-worker` |
| `venue_metadata_events` | `id` UUID PK, `venue_id` UUID FK, `event_type` VARCHAR(50), `event_data` JSONB — только на добавление                                   | `iqbene-venue-service`          |
| `venue_vectors`         | `id` UUID PK, `content` TEXT, `metadata` JSONB, `embedding` VECTOR(1536) — таблица Spring AI PgVectorStore                              | `iqbene-venue-ingestion-worker` |
| `ai_cost_tracking`      | `id` UUID PK, `provider` VARCHAR(50), `model` VARCHAR(100), `tokens_used` INTEGER, `cost_usd` NUMERIC(10,6)                             | `iqbene-venue-ingestion-worker` |

### Сводка по стратегии индексов

| Таблица            | Имя индекса                  | Тип     | Столбец(цы)                       | Назначение                       |
| ------------------ | ---------------------------- | ------- | --------------------------------- | -------------------------------- |
| `venues`           | `idx_venues_embedding`       | IVFFlat | `description_embedding`           | Семантический поиск              |
| `venues`           | `idx_venues_fts`             | GIN     | `description_text`                | Полнотекстовый поиск             |
| `venues`           | `idx_venues_metadata`        | GIN     | `metadata jsonb_path_ops`         | Фильтры атрибутов JSONB          |
| `venues`           | `idx_venues_location`        | GIST    | `location`                        | Геопространственные запросы      |
| `venue_assets`     | `idx_assets_venue`           | btree   | `venue_id`                        | Поиск по FK                      |
| `venue_assets`     | `idx_assets_type`            | btree   | `asset_type`                      | Фильтр по типу                   |
| `venue_assets`     | `idx_assets_embedding`       | IVFFlat | `extracted_text_embedding`        | Векторный поиск на уровне чанков |
| `extraction_jobs`  | `idx_jobs_asset`             | btree   | `asset_id`                        | Поиск по FK                      |
| `extraction_jobs`  | `idx_jobs_status`            | btree   | `status`                          | Опрос очереди                    |
| `metadata_events`  | `idx_metadata_events_venue`  | btree   | `venue_id, occurred_at DESC`      | Временны́е запросы                |
| `metadata_events`  | `idx_metadata_events_source` | btree   | `source_id, source_type`          | Поиск по происхождению           |
| `venue_vectors`    | `idx_vectors_embedding`      | IVFFlat | `embedding`                       | Поиск по сходству векторов       |
| `ai_cost_tracking` | `idx_ai_cost_month`          | btree   | `DATE_TRUNC('month', created_at)` | Ежемесячный свод по затратам     |

---

## 11. UI-интеграция (foundation-ui-app)

Расширяем `foundation-ui-app` — **не** форкаем. Новые функции BENE живут под:

```
src/features/venue-management/
├── create-venue/
├── upload-asset/
├── view-venue/          (профиль + карточка метаданных + галерея активов)
├── edit-metadata/       (ручная перезапись, бейджи уверенности, оповещения о конфликтах)
└── search-venues/       (поисковая строка, фильтры, семантические результаты)
```

Новые маршруты, добавленные в TanStack Router:

| Путь                   | Авторизация | Описание                     |
| ---------------------- | ----------- | ---------------------------- |
| `/venues`              | Участник    | Список / поиск площадок      |
| `/venues/new`          | Участник    | Создать площадку             |
| `/venues/:id`          | Участник    | Профиль площадки             |
| `/venues/:id/assets`   | Участник    | Галерея активов              |
| `/venues/:id/metadata` | Участник    | Просмотр + правка метаданных |

Повторно используем без изменений:

- Потоки авторизации, управление сессиями, обновление токенов
- Управление командой (`/team`)
- Биллинг / полномочия (`FeatureGate`, `useEntitlements`)
- Значок уведомлений (подписаться на уведомления `extraction.completed`)

---

## 12. Наблюдаемость

Оба сервиса BENE строго следуют фундаментальным паттернам.

**Добавляемые метрики Prometheus:**

| Метрика                              | Лейблы                            | Примечания                           |
| ------------------------------------ | --------------------------------- | ------------------------------------ |
| `iqbene_venues_total`                | tenant_id, status                 | Количество площадок по статусу       |
| `iqbene_assets_uploaded_total`       | tenant_id, asset_type             | Объём загрузок                       |
| `iqbene_extractions_total`           | tenant_id, extractor_type, status | Уровни успеха/сбоя                   |
| `iqbene_extraction_duration_seconds` | extractor_type                    | Гистограмма задержки                 |
| `iqbene_ai_cost_usd_total`           | tenant_id, model                  | Отслеживание стоимости               |
| `iqbene_search_requests_total`       | search_mode                       | ключевой / семантический / гибридный |
| `iqbene_search_latency_seconds`      | search_mode                       | Задержка поиска                      |

Дашборд Grafana добавлен в `docker/grafana/provisioning/dashboards/VipService.json`.

---

## 13. Безопасность

| Аспект                   | Подход                                                                                                                                                               |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Изоляция данных тенанта  | Схема-на-тенанта (PostgreSQL + pgvector); S3-ключ с префиксом `vip/tenants/{tenantKey}/` на тенанта — см. §4б для полной структуры ключей                            |
| Доступ к активам         | Только подписанные S3 URL (15 мин на загрузку, 1 ч на скачивание). Без публичного бакета. Пути реестра (`vip/registry/*`) недоступны через подписанные URL тенантов. |
| Обработка данных ИИ      | Документы отправляются в OpenAI API согласно их условиям обработки данных. Опция Enterprise: Azure OpenAI (данные остаются в регионе тенанта).                       |
| GDPR / право на удаление | `DELETE tenant` каскадно до venues → assets → S3-объекты → векторные эмбеддинги                                                                                      |
| Аудитный след            | Все события `venue.*`, `asset.*`, `extraction.*` пассивно потребляются Audit Service                                                                                 |
| ПДн в документах         | Предупреждение при загрузке. Не логировать извлечённый текст.                                                                                                        |

---

## 14. Технологические решения (сводка)

| Аспект                | Решение                                          | Обоснование                                                                                    |
| --------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Парсинг документов    | Apache Tika через Spring AI `TikaDocumentReader` | 1000+ форматов, поддержка DWG, отказоустойчивые Tika Pipes, ноль дополнительной инфраструктуры |
| Вёрстка / таблицы PDF | IBM Docling (Фаза 2, самохостинг)                | Передовое восстановление таблиц, лицензия MIT, ноль стоимости за страницу                      |
| AI-фреймворк          | Spring AI 1.0                                    | Нативный для Java, независимый от провайдера, встроенный ETL-конвейер, интеграция Micrometer   |
| LLM (извлечение)      | OpenAI GPT-4o                                    | Лучший структурированный вывод + мультимодальный (зрение для изображений/планов этажей)        |
| Эмбеддинги            | OpenAI text-embedding-3-small                    | 1536 измерений, $0.02/1M токенов, хорошее соотношение качество/стоимость                       |
| Векторное хранилище   | pgvector (расширение PostgreSQL)                 | Нет нового сервиса, транзакционный, изоляция по тенантам через схему                           |
| Полнотекстовый поиск  | PostgreSQL tsvector                              | Унифицирован с реляционными данными, нет нового сервиса                                        |
| Геопоиск              | PostGIS (расширение PostgreSQL)                  | Нет нового сервиса                                                                             |
| Асинхронная обработка | RabbitMQ (существующий фундамент)                | Приоритетные очереди, DLQ, уже в платформе                                                     |
| Файловое хранилище    | S3 / MinIO (существующий фундамент)              | Паттерн подписанных URL уже проверен в IAM; структура ключей VIP — см. §4б                     |

Полное обоснование и анализ конкурентов: см. `../business/comparison.md`.
