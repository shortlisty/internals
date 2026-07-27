# Venue Intelligence Platform (IQ BENE) — Справочник по архитектуре

> **Аудитория:** Инженеры, архитекторы.
> **Назначение:** Единый источник истины для всех технических решений до начала разработки.

**Документы:** [Что такое IQ BENE?](what-is-vip.md) · [Бизнес-обзор](business-overview.md) · [Конкурентная среда](intelligence-and-competitive-landscape.md) · [Архитектура](architecture.md)

---

## 1. Контекст платформы

IQ BENE — это новый продуктовый сервис, построенный **поверх фундамента IQKV**. Он не заменяет и не форкает ни один существующий сервис. Он повторно использует:

| Фундаментальный сервис       | Что наследует IQ BENE                                                                                 |
| ---------------------------- | ----------------------------------------------------------------------------------------------------- |
| `foundation-gateway-service` | Валидация JWT, маршрутизация по тенантам, распространение заголовков — изменения не нужны             |
| `foundation-iam-service`     | Авторизация, мультитенантность, приглашения в команду, SSO, паттерн подписанных URL для загрузки в S3 |
| `foundation-billing-service` | Полномочия тарифа (`max_venues`, `ai_extraction_enabled` и т.д.), жизненный цикл подписки             |
| `foundation-audit-service`   | Журнал соответствия — пассивно потребляет события IQ BENE, изменения кода не нужны                    |
| `foundation-ui-app`          | Расширен (не форкнут) новыми маршрутами `/venues/*` в рамках архитектуры FSD                          |
| `foundation-tenancy`         | Библиотека изоляции схема-на-тенант используется напрямую                                             |

**Новые сервисы, вводимые IQ BENE:**

- `vip-venue-model` — общая библиотека (JAR). Каноническая доменная модель, контракты событий, перечисления и миграции Liquibase. Нет Spring-бин, нет бизнес-логики — чистая модель и схема. Импортируется обоими сервисами.
- `vip-venue-service` — основная доменная логика: площадки, активы, метаданные, поиск, применение правил тарифа. Только синхронные запрос/ответ.
- `vip-venue-ingestion-worker` — асинхронный сайдкар: ETL-конвейер документов, оркестрация извлечения, генерация эмбеддингов, плановые задания. Нет входящего HTTP — только событийно-управляемый. Использует ту же схему PostgreSQL, что и `vip-venue-service`.

**Новая инфраструктура, вводимая IQ BENE:**

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
| `extracted_text`           | text         | Сырой текст, извлечённый парсер                                              |
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

**Не корень агрегата — журнал событий, добавляемый только.**

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

### Схема метаданных (JSONB)

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
MANUAL_OVERRIDE     → всегда побеждает (пользователь явно это задал)
VERIFIED_EXTRACTION → администратор подтвердил результат ИИ
HIGH_CONFIDENCE_AI  → уверенность ≥ 0.9
MEDIUM_CONFIDENCE_AI→ уверенность 0.7–0.9
LOW_CONFIDENCE_AI   → уверенность < 0.7
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
                    │              vip-venue-service              │
                    │  площадки · активы · метаданные · поиск · api │
                    └────────┬─────────────────────────┬─────────┘
                             │ RabbitMQ: asset.uploaded │ чтение/запись
                    ┌────────▼─────────┐     ┌─────────▼────────┐
                    │ vip-ingestion-   │     │   PostgreSQL      │
                    │    worker        │────►│   + pgvector      │
                    │ (async sidecar)  │     │   + PostGIS       │
                    └──────────────────┘     └──────────────────┘
                             │
                    ┌────────▼─────────┐
                    │   S3 / MinIO     │  (существующий)
                    └──────────────────┘
```

### vip-venue-service

- **Обязанности:** CRUD площадок, поток загрузки активов (подписанный URL), чтение/запись метаданных, API поиска, применение полномочий тарифа
- **База данных:** владеет схемой IQ BENE в PostgreSQL (схема-на-тенант через `foundation-tenancy`). Общая с `vip-venue-ingestion-worker` — межсервисные вызовы API для данных нет.
- **Предоставляет:** REST API на `/api/v1/venues`
- **Публикует:** `venue.created`, `venue.updated`, `asset.uploaded`, `asset.deleted` (RabbitMQ)
- **Потребляет:** `extraction.completed`, `extraction.failed` (RabbitMQ) — запускает агрегацию метаданных

### vip-venue-ingestion-worker

- **Обязанности:** ETL-конвейер документов (парсинг → чанкинг → извлечение → эмбеддинг), жизненный цикл заданий извлечения, агрегация метаданных, плановые обслуживающие задания (обновление устаревшей агрегации, отчёты по стоимости)
- **Природа:** асинхронный сайдкар — нет входящего HTTP, нет REST API, нет записи в service discovery. Только событийно-управляемый.
- **База данных:** общая схема PostgreSQL с `vip-venue-service`. Читает `venue_assets`, пишет `extraction_jobs`, `venue_metadata_events`, `venue_vectors`, `ai_cost_tracking`.
- **Потребляет:** `asset.uploaded` (RabbitMQ) — запускает ETL-конвейер
- **Публикует:** `extraction.started`, `extraction.completed`, `extraction.failed` (RabbitMQ)
- **Внешние вызовы:** OpenAI API (GPT-4o, text-embedding-3-small), опционально сайдкар Docling (Фаза 2)
- **Масштабирование:** реплики масштабируются независимо на основе глубины очереди RabbitMQ — без влияния на `vip-venue-service`

### Владение таблицами

Оба сервиса используют одну схему PostgreSQL. Владение определяет, кто может писать в таблицу. Кросс-граничные чтения разрешены; кросс-граничные записи — нет.

| Таблица                 | Владелец                     | Другой сервис может…                                                                |
| ----------------------- | ---------------------------- | ----------------------------------------------------------------------------------- |
| `venues`                | `vip-venue-service`          | читать (ingestion-worker: только разрешать venue_id)                                |
| `venue_assets`          | `vip-venue-service`          | читать (ingestion-worker: получать актив для обработки)                             |
| `venue_metadata_events` | `vip-venue-service`          | писать через реакцию на событие (`extraction.completed` → venue-service агрегирует) |
| `extraction_jobs`       | `vip-venue-ingestion-worker` | читать (venue-service: показывать статус задания в API)                             |
| `venue_vectors`         | `vip-venue-ingestion-worker` | читать (venue-service: векторные поисковые запросы)                                 |
| `ai_cost_tracking`      | `vip-venue-ingestion-worker` | читать (venue-service: показывать сводку по стоимости в API)                        |

Единственный законный кросс-граничный чтение из `vip-venue-ingestion-worker` — это `SELECT` по `venue_assets` по `asset_id` (переданному в полезной нагрузке события `asset.uploaded`). Это поиск по внешнему ключу, не бизнес-логика — допустимо и намеренно.

---

## 4a. Общая библиотека — vip-venue-model

`vip-venue-model` — это обычная Java-библиотека (JAR, без Spring Boot, без `@SpringBootApplication`). И `vip-venue-service`, и `vip-venue-ingestion-worker` объявляют её как compile-зависимость. Это единый источник истины для всего, по чему оба сервиса должны договориться.

**Содержимое:**

```
vip-venue-model/
├── model/
│   ├── venue/
│   │   ├── Venue.java                  JPA-сущность (корень агрегата)
│   │   ├── VenueStatus.java            enum: DRAFT, ACTIVE, ARCHIVED
│   │   └── VenueAsset.java             JPA-сущность
│   ├── asset/
│   │   ├── AssetType.java              enum: PDF_DECK, FLOOR_PLAN, PHOTO, CAD_FILE…
│   │   └── ExtractionStatus.java       enum: PENDING, IN_PROGRESS, COMPLETED, FAILED
│   ├── extraction/
│   │   ├── ExtractionJob.java          JPA-сущность
│   │   ├── ExtractorType.java          enum: TIKA_TEXT, GPT4O_DOCUMENT, GPT4O_VISION
│   │   └── VenueMetadataEvent.java     JPA-сущность (журнал событий, добавляемый только)
│   ├── metadata/
│   │   ├── VenueMetadata.java          value object (зеркалирует venues.metadata JSONB)
│   │   ├── VenueCapacity.java          value object конфигураций вместимости
│   │   ├── MetadataSource.java         происхождение по каждому полю
│   │   └── MetadataEventType.java      enum: ASSET_EXTRACTED, MANUAL_OVERRIDE, BULK_IMPORT
│   └── events/                         RabbitMQ-контракты сообщений (POJO, без зависимостей от фреймворка)
│       ├── AssetUploadedEvent.java
│       ├── ExtractionStartedEvent.java
│       ├── ExtractionCompletedEvent.java
│       └── ExtractionFailedEvent.java
└── db/
    └── changelog/
        └── tenant/                     Миграции Liquibase — единый источник истины
            ├── 001-venues.sql
            ├── 002-venue-assets.sql
            ├── 003-extraction-jobs.sql
            ├── 004-metadata-events.sql
            ├── 005-venue-vectors.sql
            └── 006-ai-cost-tracking.sql
```

**Правила:**

- Никаких `@Service`, `@Repository`, `@Component` или любых аннотаций Spring-бин
- Никакой бизнес-логики — только сущности, value objects, перечисления, event POJO
- JPA-аннотации на сущностях допустимы (`@Entity`, `@Table`, `@Column` и т.д.)
- Миграции Liquibase живут здесь, так что изменения схемы — это повышение версии зависимости во время компиляции, а не упражнение по координации между сервисами
- Изменение поля в event POJO — это разрыв во время компиляции в обоих сервисах — намеренно, предотвращает тихое смещение контрактов

**Граф зависимостей:**

```
vip-venue-model  (библиотека, без среды выполнения)
      ├── vip-venue-service     (Spring Boot, импортирует model)
      └── vip-venue-ingestion-worker  (Spring Boot, импортирует model)
```

---

## 5. ETL-конвейер (vip-venue-ingestion-worker)

Построен на **ETL-фреймворке Spring AI**. Три компонуемых стадии:

```
DocumentReader  →  DocumentTransformer  →  DocumentWriter
  (парсинг)            (чанкинг + обогащение)        (эмбеддинг + сохранение)
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
3. **Агрегация** — публикует событие `extraction.completed` → `MetadataAggregationConsumer` обновляет `venues.metadata`.

### SLA обработки

| Тип актива                 | Целевая задержка |
| -------------------------- | ---------------- |
| PDF / DOCX (текст)         | < 30 с           |
| Изображения / планы этажей | < 60 с           |
| CAD-файлы                  | < 2 мин          |

Повтор при сбое: 3 попытки с экспоненциальной задержкой. После 3 сбоев → событие `extraction.failed` → уведомление пользователя.

---

## 6. Архитектура поиска

Весь поиск обслуживается `vip-venue-service`, делающим запросы напрямую в PostgreSQL. Нет отдельного поискового сервиса.

### Режимы поиска

| Режим                     | Реализация                                           | Сценарий использования                                |
| ------------------------- | ---------------------------------------------------- | ----------------------------------------------------- |
| Ключевой                  | `tsvector` / `to_tsquery` (GIN-индекс)               | "конференция центр AV"                                |
| Структурированные фильтры | Запросы JSONB (GIN-индекс)                           | вместимость ≥ 200, удобства включают WiFi             |
| Семантический             | косинусное расстояние pgvector (IVFFlat-индекс)      | "современное лофт-пространство с естественным светом" |
| Геопространственный       | PostGIS `ST_DWithin` (GIST-индекс)                   | площадки в радиусе 10 миль от почтового индекса       |
| Гибридный                 | Reciprocal Rank Fusion по ключевому + семантическому | Стандартная поисковая строка                          |

### Стратегия векторного индекса

- Тип индекса: IVFFlat (быстрое построение, хорошее восстановление до ~5M векторов)
- Расстояние: косинусное
- Измерения: 1536
- Область: схема на тенанта (нет межтенантных утечек)
- Путь обновления: переключиться на HNSW, когда у тенанта превысит ~500K площадок (редко)

---

## 7. Поверхность API (vip-venue-service)

Базовый путь: `/api/v1/venues`

### Площадки

| Метод    | Путь      | Авторизация  | Описание                                             |
| -------- | --------- | ------------ | ---------------------------------------------------- |
| `GET`    | `/`       | JWT Участник | Список площадок (с пагинацией, фильтруемый)          |
| `POST`   | `/`       | JWT Участник | Создать профиль площадки                             |
| `GET`    | `/{id}`   | JWT Участник | Получить площадку с объединёнными метаданными        |
| `PATCH`  | `/{id}`   | JWT Админ    | Обновить поля площадки                               |
| `DELETE` | `/{id}`   | JWT Владелец | Архивировать площадки                                |
| `GET`    | `/search` | JWT Участник | Гибридный поиск (ключевой + семантический + фильтры) |

### Активы

| Метод    | Путь                             | Авторизация  | Описание                                        |
| -------- | -------------------------------- | ------------ | ----------------------------------------------- |
| `POST`   | `/{venueId}/assets/initiate`     | JWT Участник | Начать загрузку — возвращает подписанный S3 URL |
| `POST`   | `/{venueId}/assets/{id}/confirm` | JWT Участник | Подтвердить загрузку, запустить извлечение      |
| `GET`    | `/{venueId}/assets`              | JWT Участник | Список активов для площадки                     |
| `DELETE` | `/{venueId}/assets/{id}`         | JWT Участник | Удалить актив и объект S3                       |

### Метаданные

| Метод  | Путь                                   | Авторизация  | Описание                                         |
| ------ | -------------------------------------- | ------------ | ------------------------------------------------ |
| `GET`  | `/{venueId}/metadata`                  | JWT Участник | Получить объединённые метаданные + происхождение |
| `POST` | `/{venueId}/metadata/{field}/override` | JWT Участник | Ручная перезапись для поля                       |
| `GET`  | `/{venueId}/metadata/{field}/history`  | JWT Участник | История событий извлечения для поля              |

### Задания извлечения (только чтение для клиентов)

| Метод | Путь                             | Авторизация  | Описание                               |
| ----- | -------------------------------- | ------------ | -------------------------------------- |
| `GET` | `/{venueId}/extractions`         | JWT Участник | Список заданий извлечения для площадки |
| `GET` | `/{venueId}/extractions/{jobId}` | JWT Участник | Получить статус и результат задания    |

---

## 8. Контракты событий (RabbitMQ)

Обменник: `iqkv.events` (Topic) — тот же обменник, что используют все фундаментальные сервисы.

### Публикуется vip-venue-service

| Ключ маршрутизации | Поля полезной нагрузки                                          | Описание                              |
| ------------------ | --------------------------------------------------------------- | ------------------------------------- |
| `venue.created`    | venue_id, tenant_id, created_by                                 | Новый профиль площадки создан         |
| `venue.updated`    | venue_id, tenant_id, changed_fields                             | Поля площадки обновлены               |
| `asset.uploaded`   | asset_id, venue_id, tenant_id, asset_type, s3_key, content_type | Актив подтверждён, готов к извлечению |
| `asset.deleted`    | asset_id, venue_id, tenant_id                                   | Актив удалён                          |

### Публикуется vip-venue-ingestion-worker

| Ключ маршрутизации     | Поля полезной нагрузки                        | Описание              |
| ---------------------- | --------------------------------------------- | --------------------- |
| `extraction.started`   | job_id, asset_id, venue_id, tenant_id         | Обработка началась    |
| `extraction.completed` | job_id, asset_id, venue_id, tenant_id         | Извлечение удалось    |
| `extraction.failed`    | job_id, asset_id, venue_id, tenant_id, reason | Все повторы исчерпаны |

### Потребляется vip-venue-ingestion-worker

| Ключ маршрутизации | Очередь                                | Действие                                     |
| ------------------ | -------------------------------------- | -------------------------------------------- |
| `asset.uploaded`   | `vip.extraction.priority` (Enterprise) | Запустить ETL-конвейер немедленно            |
| `asset.uploaded`   | `vip.extraction.standard` (Free/Pro)   | Запустить ETL-конвейер (стандартная очередь) |

### Потребляется vip-venue-service

| Ключ маршрутизации     | Очередь                    | Действие                                    |
| ---------------------- | -------------------------- | ------------------------------------------- |
| `extraction.completed` | `vip.metadata.aggregation` | Запустить агрегацию метаданных для площадки |
| `extraction.failed`    | `vip.extraction.dlq`       | Пометить extraction_status актива = FAILED  |

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

Миграции живут в `vip-venue-model` под `src/main/resources/db/changelog/tenant/` — общая библиотека это единый источник истины для схемы. И `vip-venue-service`, и `vip-venue-ingestion-worker` включают библиотеку в свой classpath; `vip-venue-service` запускает миграции при старте (или выделенный init-контейнер применяет их при предоставлении тенанта через слушатель `TenantProvisionedEvent`, тот же паттерн, что и в IAM).

```sql
-- расширения (применяются один раз на схему тенанта)
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS postgis;

-- venues
CREATE TABLE venues (
  id                      UUID PRIMARY KEY,
  name                    VARCHAR(255) NOT NULL,
  address                 TEXT,
  location                GEOGRAPHY(POINT, 4326),
  description             TEXT,
  description_embedding   VECTOR(1536),
  description_text        TSVECTOR,
  status                  VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
  metadata                JSONB NOT NULL DEFAULT '{}',
  metadata_sources        JSONB NOT NULL DEFAULT '{}',
  metadata_aggregated_at  TIMESTAMP,
  created_by              UUID NOT NULL,
  created_at              TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at              TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_venues_embedding  ON venues USING ivfflat (description_embedding vector_cosine_ops);
CREATE INDEX idx_venues_fts        ON venues USING GIN (description_text);
CREATE INDEX idx_venues_metadata   ON venues USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_venues_location   ON venues USING GIST (location);

CREATE TRIGGER trg_venues_tsvector BEFORE INSERT OR UPDATE ON venues
  FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(description_text, 'pg_catalog.english', name, description, address);

-- venue_assets
CREATE TABLE venue_assets (
  id                       UUID PRIMARY KEY,
  venue_id                 UUID NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
  asset_type               VARCHAR(50) NOT NULL,
  file_name                VARCHAR(255) NOT NULL,
  content_type             VARCHAR(100) NOT NULL,
  size_bytes               BIGINT NOT NULL,
  s3_key                   TEXT NOT NULL,
  extracted_text           TEXT,
  extracted_text_embedding VECTOR(1536),
  extraction_status        VARCHAR(20) NOT NULL DEFAULT 'PENDING',
  uploaded_by              UUID NOT NULL,
  uploaded_at              TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_assets_venue       ON venue_assets (venue_id);
CREATE INDEX idx_assets_type        ON venue_assets (asset_type);
CREATE INDEX idx_assets_embedding   ON venue_assets USING ivfflat (extracted_text_embedding vector_cosine_ops);

-- extraction_jobs
CREATE TABLE extraction_jobs (
  id               UUID PRIMARY KEY,
  asset_id         UUID NOT NULL REFERENCES venue_assets(id) ON DELETE CASCADE,
  status           VARCHAR(20) NOT NULL,
  extractor_type   VARCHAR(50) NOT NULL,
  extracted_data   JSONB,
  confidence_scores JSONB,
  started_at       TIMESTAMP,
  completed_at     TIMESTAMP,
  error_message    TEXT
);
CREATE INDEX idx_jobs_asset  ON extraction_jobs (asset_id);
CREATE INDEX idx_jobs_status ON extraction_jobs (status);

-- venue_metadata_events (добавляемые только)
CREATE TABLE venue_metadata_events (
  id           UUID PRIMARY KEY,
  venue_id     UUID NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
  event_type   VARCHAR(50) NOT NULL,
  source_type  VARCHAR(50) NOT NULL,
  source_id    UUID,
  event_data   JSONB NOT NULL,
  occurred_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  created_by   UUID
);
CREATE INDEX idx_metadata_events_venue  ON venue_metadata_events (venue_id, occurred_at DESC);
CREATE INDEX idx_metadata_events_source ON venue_metadata_events (source_id, source_type);

-- venue_vectors (таблица Spring AI PgVectorStore)
CREATE TABLE venue_vectors (
  id        UUID PRIMARY KEY,
  content   TEXT NOT NULL,
  metadata  JSONB,
  embedding VECTOR(1536)
);
CREATE INDEX idx_vectors_embedding ON venue_vectors USING ivfflat (embedding vector_cosine_ops);

-- ai_cost_tracking
CREATE TABLE ai_cost_tracking (
  id             UUID PRIMARY KEY,
  provider       VARCHAR(50) NOT NULL,
  operation_type VARCHAR(50) NOT NULL,
  model          VARCHAR(100) NOT NULL,
  tokens_used    INTEGER,
  cost_usd       NUMERIC(10, 6) NOT NULL,
  asset_id       UUID REFERENCES venue_assets(id),
  created_at     TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_ai_cost_month ON ai_cost_tracking (DATE_TRUNC('month', created_at));
```

---

## 11. UI-интеграция (foundation-ui-app)

Расширяем `foundation-ui-app` — **не** форкаем. Новые функции IQ BENE живут под:

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

Оба сервиса IQ BENE строго следуют фундаментальным паттернам.

**Добавляемые метрики Prometheus:**

| Метрика                           | Лейблы                            | Примечания                           |
| --------------------------------- | --------------------------------- | ------------------------------------ |
| `vip_venues_total`                | tenant_id, status                 | Количество площадок по статусу       |
| `vip_assets_uploaded_total`       | tenant_id, asset_type             | Объём загрузок                       |
| `vip_extractions_total`           | tenant_id, extractor_type, status | Уровни успеха/сбоя                   |
| `vip_extraction_duration_seconds` | extractor_type                    | Гистограмма задержки                 |
| `vip_ai_cost_usd_total`           | tenant_id, model                  | Отслеживание стоимости               |
| `vip_search_requests_total`       | search_mode                       | ключевой / семантический / гибридный |
| `vip_search_latency_seconds`      | search_mode                       | Задержка поиска                      |

Дашборд Grafana добавлен в `docker/grafana/provisioning/dashboards/VipService.json`.

---

## 13. Безопасность

| Аспект                   | Подход                                                                                                                                         |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Изоляция данных тенанта  | Схема-на-тенанта (PostgreSQL + pgvector); префикс ключа S3 на тенанта                                                                          |
| Доступ к активам         | Только подписанные S3 URL (15 мин на загрузку, 1 ч на скачивание). Без публичного бакета.                                                      |
| Обработка данных ИИ      | Документы отправляются в OpenAI API согласно их условиям обработки данных. Опция Enterprise: Azure OpenAI (данные остаются в регионе тенанта). |
| GDPR / право на удаление | `DELETE tenant` каскадно до venues → assets → S3-объекты → векторные эмбеддинги                                                                |
| Аудитный след            | Все события `venue.*`, `asset.*`, `extraction.*` пассивно потребляются Audit Service                                                           |
| ПДн в документах         | Предупреждение при загрузке. Не логировать извлечённый текст.                                                                                  |

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
| Файловое хранилище    | S3 / MinIO (существующий фундамент)              | Паттерн подписанных URL уже проверен в IAM                                                     |

Полное обоснование и анализ конкурентов: см. `venue-intelligence-platform-intelligence.md`.

---

## 15. Открытые решения (разрешить перед Спринтом 1)

- [x] **Один сервис или два?** ~~`vip-venue-service` + `vip-ai-service` vs. единый `vip-venue-service` с внутренним AI-модулем.~~ **Решено:** Две развёртывания — `vip-venue-service` (синхронный API, привязанный к данным) и `vip-venue-ingestion-worker` (асинхронный сайдкар, общая схема, нет входящего HTTP). Сервисы привязаны к данным; ингеста — это задача обработки, а не равноправный сервис.
- [x] **Соглашение об именовании.** Имена сервисов отражают предметную область/назначение, а не технологию реализации. `vip-venue-ingestion-worker` описывает, что он делает (ингеста и обработка активов), а не как (AI/ML).
- [ ] **Docling в Фазе 1?** Начать с чистого Tika (проще). Добавить сайдкар Docling в Фазе 2, когда понадобится точность планов этажей / таблиц. **Склонность: только Tika для Фазы 1.**
- [ ] **Таблица чанков** в отдельной схеме или той же, что и таблицы площадок? `PgVectorStore` Spring AI по умолчанию использует таблицу `vector_store`. IQ BENE использует `venue_vectors` для явности. Подтвердить именование перед первой миграцией.
- [ ] **Детализация отслеживания стоимости:** на актив или на тенанта-в-месяц? Оба варианта есть в схеме; решить, какой показывать в UI.

---

## 16. Следующие шаги и итерации дизайна

### Перед Спринтом 1 — Сначала разрешите это

- [ ] **Docling в Фазе 1?** Нет — только Tika для MVP. Добавить сайдкар Docling в Фазе 2 для точности планов этажей / таблиц.
- [ ] **Сокращение скоупа MVP** — Фаза 1: профили площадок, загрузка активов, базовое извлечение (только PDF), ключевой + семантический поиск, командное сотрудничество. Всё остальное — Фаза 2+.

### Дизайн Фазы 2 (сигнал после MVP)

- UX разрешения конфликтов — форма API + машина состояний для разрешения конкурирующих извлечённых значений
- Массовый импорт / CSV-ингеста — для консьерж-введения и миграций агентств
- Сохранённые поиски + оповещения — схема и механизм доставки
- Доставка уведомлений — выбрать WebSocket vs. опрос для статуса извлечения (можно повторно использовать WebSocket-инфраструктуру IAM)
- Визуальное извлечение CAD — конвертировать DWG/DXF в изображение, затем GPT-4o vision
- Видео-обзоры — извлечение ключевых кадров через ffmpeg, обнаружение удобств на основе зрения

### Дизайн Фазы 3

- Экспорт / общий доступ — общие брендированные ссылки, генерация отчётов PDF/Excel, вид сравнения площадок
- Дедупликация — обнаруживать и объединять дублирующие записи площадок между командами
- Рабочий процесс верификации — API + UX для перевода извлечённых ИИ полей в статус «проверено человеком»
- CRM-интеграции — коннекторы вебхуков Salesforce, HubSpot

### Сквозные аспекты (спроектировать перед сборкой Фазы 2)

- **Устойчивость ИИ** — поведение при недоступности OpenAI; изящная деградация (поставить в очередь на повтор, уведомить пользователя)
- **Контроли расходов ИИ на тенанта** — бюджетные лимиты, ежемесячные оповещения об использовании, что происходит при достижении лимита
- **Политика хранения / хранения** — правила жизненного цикла S3, старые версии эмбеддингов, хранение извлечённого текста (угол GDPR)
- **Стратегия тестирования** — тестовые дублёры для OpenAI, фикстуры документов для ETL-конвейера, контрактные тесты между сервисами

### Стратегические ставки для проверки первыми пользователями

- Используется ли семантический поиск («найди площадки, похожие на эту») на самом деле, или организаторы предпочитают структурированные фильтры?
- Какова реальная точность извлечения на реальных PDF площадок? Запустить бенчмарк на 50 образцах документов перед тем, как принимать на себя обязательства по точности.
- Момент «ага» — это результат извлечения или поиск, мгновенно находящий что-то? Формирует дизайн потока онбординга.
- Какой тип актива важнее всего загрузить первым — презентации площадок или планы этажей? Информирует приоритет парсеров.

---

**Документы:** [Что такое IQ BENE?](what-is-vip.md) · [Бизнес-обзор](business-overview.md) · [Конкурентная среда](intelligence-and-competitive-landscape.md) · [Архитектура](architecture.md)
