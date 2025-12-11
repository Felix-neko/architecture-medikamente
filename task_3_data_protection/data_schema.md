`Generated with Claude Opus 4.5`

# Схема данных сервисов TO-BE: тегирование и шифрование

Документ описывает таблицы и поля всех сервисов целевой архитектуры с указанием:
- тегов OpenMetadata для классификации
- необходимости и методов шифрования

---

## Теги OpenMetadata

| Тег | Описание | Применение |
|-----|----------|------------|
| `pii` | Персональные данные (ФИО, контакты, документы) | Поля класса C2 |
| `extra_protected` | Особая категория ПДн (ст. 10 152-ФЗ) | Поля класса C1 |
| `med_record` | Медицинские данные (врачебная тайна) | Диагнозы, анамнез, назначения |
| `dlp_needed` | Требуется DLP-обработка перед аналитикой | Все C1-C2 поля |
| `financial` | Финансовые данные | Платежи, счета |
| `consent_required` | Доступ только при наличии ИС | Медкарта пациента |
| `audit` | Требуется аудит доступа | Все C1-C3 данные |

---

## 1. CRM API (crm_db)

### Таблица `patients` — Пациенты

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `external_id` | VARCHAR(50) | — | Нет | ID для внешних систем (без ПДн) |
| `first_name` | VARCHAR(100) | `pii`, `dlp_needed`, `audit` | AES-256 (pgcrypto) | Имя |
| `last_name` | VARCHAR(100) | `pii`, `dlp_needed`, `audit` | AES-256 (pgcrypto) | Фамилия |
| `middle_name` | VARCHAR(100) | `pii`, `dlp_needed` | AES-256 (pgcrypto) | Отчество |
| `birth_date` | DATE | `pii`, `dlp_needed` | AES-256 (pgcrypto) | Дата рождения |
| `gender` | ENUM | — | Нет | Пол (м/ж) |
| `phone` | VARCHAR(20) | `pii`, `dlp_needed`, `audit` | AES-256 (pgcrypto) | Телефон |
| `email` | VARCHAR(255) | `pii`, `dlp_needed` | AES-256 (pgcrypto) | Email |
| `created_at` | TIMESTAMP | — | Нет | Дата регистрации |
| `updated_at` | TIMESTAMP | `audit` | Нет | Дата обновления |

### Таблица `patient_documents` — Документы пациентов

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `patient_id` | UUID | — | Нет | FK → patients |
| `document_type` | ENUM | — | Нет | Тип: passport, snils, polis |
| `document_series` | VARCHAR(10) | `pii`, `dlp_needed`, `audit` | AES-256 (pgcrypto) | Серия документа |
| `document_number` | VARCHAR(20) | `pii`, `dlp_needed`, `audit` | AES-256 (pgcrypto) | Номер документа |
| `issued_by` | TEXT | `pii`, `dlp_needed` | AES-256 (pgcrypto) | Кем выдан |
| `issued_date` | DATE | `pii` | AES-256 (pgcrypto) | Дата выдачи |
| `created_at` | TIMESTAMP | — | Нет | — |

### Таблица `patient_insurance` — Полисы ДМС

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `patient_id` | UUID | — | Нет | FK → patients |
| `insurance_company_id` | UUID | — | Нет | FK → справочник страховых |
| `policy_number` | VARCHAR(50) | `pii`, `dlp_needed` | AES-256 (pgcrypto) | Номер полиса |
| `valid_from` | DATE | — | Нет | Начало действия |
| `valid_to` | DATE | — | Нет | Окончание действия |
| `created_at` | TIMESTAMP | — | Нет | — |

### Таблица `appointments` — Записи на приём

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `patient_id` | UUID | `audit` | Нет | FK → patients |
| `doctor_id` | UUID | — | Нет | FK → staff |
| `appointment_date` | TIMESTAMP | `audit` | Нет | Дата/время приёма |
| `service_id` | UUID | — | Нет | FK → справочник услуг |
| `status` | ENUM | — | Нет | scheduled, completed, cancelled |
| `created_by` | UUID | `audit` | Нет | Кто создал запись |
| `created_at` | TIMESTAMP | — | Нет | — |

### Таблица `staff` — Сотрудники

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `ad_username` | VARCHAR(100) | — | Нет | Логин в AD |
| `first_name` | VARCHAR(100) | `pii` | AES-256 (pgcrypto) | Имя сотрудника |
| `last_name` | VARCHAR(100) | `pii` | AES-256 (pgcrypto) | Фамилия сотрудника |
| `role` | ENUM | — | Нет | doctor, nurse, receptionist, admin |
| `specialization_id` | UUID | — | Нет | FK → справочник специализаций |
| `is_active` | BOOLEAN | — | Нет | Активен ли |
| `created_at` | TIMESTAMP | — | Нет | — |

### Таблица `schedules` — Расписание

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `staff_id` | UUID | — | Нет | FK → staff |
| `work_date` | DATE | — | Нет | Дата |
| `start_time` | TIME | — | Нет | Начало смены |
| `end_time` | TIME | — | Нет | Конец смены |
| `clinic_id` | UUID | — | Нет | FK → справочник филиалов |

---

## 2. Med Record API (medrecord_db)

### Таблица `medical_records` — Медкарты

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `patient_id` | UUID | `consent_required`, `audit` | Нет | FK → crm_db.patients (cross-db ref) |
| `record_number` | VARCHAR(20) | — | Нет | Номер МК |
| `created_at` | TIMESTAMP | — | Нет | Дата создания МК |
| `minio_folder` | VARCHAR(255) | — | Нет | Путь `/patient_uuid/` в MinIO |

### Таблица `visits` — Визиты

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `medical_record_id` | UUID | `consent_required`, `audit` | Нет | FK → medical_records |
| `doctor_id` | UUID | `audit` | Нет | ID врача (из crm_db.staff) |
| `visit_date` | TIMESTAMP | `audit` | Нет | Дата визита |
| `visit_type` | ENUM | — | Нет | primary, follow_up, emergency |

### Таблица `diagnoses` — Диагнозы

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `visit_id` | UUID | `consent_required`, `audit` | Нет | FK → visits |
| `icd_code` | VARCHAR(10) | `med_record`, `extra_protected`, `dlp_needed`, `audit` | AES-256 (отдельный ключ) | Код МКБ-10 |
| `description` | TEXT | `med_record`, `extra_protected`, `dlp_needed`, `audit` | AES-256 (отдельный ключ) | Описание диагноза |
| `diagnosis_type` | ENUM | `med_record` | Нет | primary, secondary, complication |
| `diagnosed_at` | TIMESTAMP | `audit` | Нет | Дата постановки |
| `diagnosed_by` | UUID | `audit` | Нет | ID врача |

### Таблица `anamnesis` — Анамнез

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `visit_id` | UUID | `consent_required`, `audit` | Нет | FK → visits |
| `anamnesis_type` | ENUM | — | Нет | vitae, morbi |
| `content` | TEXT | `med_record`, `extra_protected`, `dlp_needed`, `audit` | AES-256 (отдельный ключ) | Текст анамнеза |
| `recorded_at` | TIMESTAMP | `audit` | Нет | — |
| `recorded_by` | UUID | `audit` | Нет | ID врача |

### Таблица `prescriptions` — Назначения

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `visit_id` | UUID | `consent_required`, `audit` | Нет | FK → visits |
| `medication_name` | VARCHAR(255) | `med_record`, `dlp_needed` | AES-256 (pgcrypto) | Название препарата |
| `dosage` | VARCHAR(100) | `med_record` | AES-256 (pgcrypto) | Дозировка |
| `frequency` | VARCHAR(100) | `med_record` | Нет | Частота приёма |
| `duration_days` | INT | — | Нет | Длительность курса |
| `prescribed_at` | TIMESTAMP | `audit` | Нет | — |
| `prescribed_by` | UUID | `audit` | Нет | ID врача |

### Таблица `lab_orders` — Направления на анализы

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `visit_id` | UUID | `consent_required`, `audit` | Нет | FK → visits |
| `test_codes` | JSONB | `med_record`, `dlp_needed` | Нет | Коды анализов (справочник) |
| `ordered_at` | TIMESTAMP | `audit` | Нет | — |
| `ordered_by` | UUID | `audit` | Нет | ID врача/медсестры |
| `status` | ENUM | — | Нет | pending, sent, completed |
| `external_order_id` | VARCHAR(100) | — | Нет | ID в системе лаборатории |

### Таблица `lab_results` — Результаты анализов

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `lab_order_id` | UUID | `consent_required`, `audit` | Нет | FK → lab_orders |
| `result_data` | JSONB | `med_record`, `extra_protected`, `dlp_needed`, `audit` | AES-256 (отдельный ключ) | Результаты в JSON |
| `minio_path` | VARCHAR(255) | `med_record`, `extra_protected` | Нет | Путь к PDF/скану в MinIO |
| `received_at` | TIMESTAMP | `audit` | Нет | Дата получения |

### Таблица `sensitive_data` — Особо защищённые данные

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `medical_record_id` | UUID | `consent_required`, `audit` | Нет | FK → medical_records |
| `data_type` | ENUM | `extra_protected`, `audit` | Нет | hiv_status, psychiatric, genetic |
| `data_value` | TEXT | `extra_protected`, `med_record`, `dlp_needed`, `audit` | AES-256 (ОТДЕЛЬНЫЙ ключ, доступ только лечащему врачу) | Значение |
| `recorded_at` | TIMESTAMP | `audit` | Нет | — |
| `recorded_by` | UUID | `audit` | Нет | ID врача |

---

## 3. Consent Manager API (consent_db)

### Таблица `consents` — Информированные согласия

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `patient_id` | UUID | `audit` | Нет | ID пациента (cross-db ref) |
| `consent_type` | ENUM | — | Нет | treatment, data_processing, lab_sharing |
| `granted_at` | TIMESTAMP | `audit` | Нет | Дата подписания |
| `valid_until` | TIMESTAMP | — | Нет | Срок действия |
| `granted_to_staff_id` | UUID | `audit` | Нет | Кому дано согласие |
| `scope` | JSONB | — | Нет | Область действия (услуги, данные) |
| `hmac_signature` | VARCHAR(64) | — | Нет | HMAC-подпись для целостности |
| `revoked_at` | TIMESTAMP | `audit` | Нет | Дата отзыва (NULL если действует) |

### Таблица `consent_audit_log` — Аудит согласий

| Поле | Тип | Теги | Шифрование | Комментарий |
|------|-----|------|------------|-------------|
| `id` | UUID | — | Нет | PK |
| `consent_id` | UUID | `audit` | Нет | FK → consents |
| `action` | ENUM | `audit` | Нет | granted, revoked, accessed, verified |
| `actor_id` | UUID | `audit` | Нет | Кто выполнил действие |
| `actor_ip` | INET | `audit` | Нет | IP-адрес |
| `timestamp` | TIMESTAMP | — | Нет | — |
| `details` | JSONB | — | Нет | Детали действия |

---

## 4. Keycloak (keycloak_db)

> Стандартная схема Keycloak. Ключевые таблицы:

| Таблица | Теги | Шифрование | Комментарий |
|---------|------|------------|-------------|
| `USER_ENTITY` | `pii` (username, email) | Стандартное Keycloak-шифрование | Пользователи |
| `CREDENTIAL` | `extra_protected` | bcrypt/Argon2 хеширование | Хеши паролей |
| `USER_ATTRIBUTE` | `pii` (если custom attrs) | Нет | Атрибуты пользователей |
| `USER_SESSION` | — | Нет | Активные сессии |

---

## 5. Redis (auth_proxy_cache, consent_cache)

### Структура ключей (Auth Proxy)

| Ключ | Теги | Шифрование | TTL | Комментарий |
|------|------|------------|-----|-------------|
| `session:{session_id}` | `audit` | FERNET (AES-128) | 30 мин | Данные сессии |
| `jwt_blacklist:{jti}` | — | Нет | до exp токена | Отозванные токены |
| `rate_limit:{user_id}:{endpoint}` | — | Нет | 1 мин | Rate limiting |

### Структура ключей (Consent Manager)

| Ключ | Теги | Шифрование | TTL | Комментарий |
|------|------|------------|-----|-------------|
| `consent:{patient_id}:{staff_id}` | `audit` | FERNET | 5 мин | Кэш проверки ИС |
| `consent_list:{patient_id}` | `audit` | FERNET | 10 мин | Список ИС пациента |

---

## 6. MinIO (S3 Storage)

### Структура бакетов

| Бакет | Теги | Шифрование | Access Policy |
|-------|------|------------|---------------|
| `medical-records` | `med_record`, `extra_protected`, `consent_required` | SSE-S3 (AES-256), ключ в Vault | Доступ только через Auth Proxy с проверкой ИС |
| `lab-results` | `med_record`, `extra_protected` | SSE-S3 (AES-256) | Доступ для doctor, nurse |
| `consent-scans` | `pii` | SSE-S3 (AES-256) | Доступ для receptionist, patient |
| `audit-exports` | `audit` | SSE-S3 (AES-256) | Только admin |

### Структура папок в `medical-records`

```
medical-records/
├── {patient_uuid}/
│   ├── scans/
│   │   ├── passport_scan.pdf          # pii, dlp_needed
│   │   └── xray_2024-01-15.dcm        # med_record, extra_protected
│   ├── documents/
│   │   └── consent_2024-01-10.pdf     # pii
│   └── reports/
│       └── lab_result_2024-01-20.pdf  # med_record, extra_protected
```

---

## 7. ClickHouse (analytics_db)

### Слой `raw_sources` — Сырые данные (ЗАШИФРОВАНО)

> Данные из Kafka/Debezium, доступ только для ETL-процессов

| Таблица | Теги | Шифрование | Комментарий |
|---------|------|------------|-------------|
| `cdc_patients` | `pii`, `dlp_needed` | AES-256 (column-level) | Реплика patients |
| `cdc_diagnoses` | `extra_protected`, `dlp_needed` | AES-256 (column-level) | Реплика diagnoses |
| `cdc_visits` | `audit` | Нет | Реплика visits |

### Слой `tokenized` — Токенизированные данные

> ФИО заменены на токены, ID сохранены

| Таблица | Теги | Шифрование | Комментарий |
|---------|------|------------|-------------|
| `visits_tokenized` | — | Нет | Визиты без ПДн |
| `diagnoses_tokenized` | `med_record` | Нет | Диагнозы с токеном вместо patient_id |

### Слой `dlp_approved` — Обезличенные данные

> Полностью обезличено, доступно для analyst

| Таблица | Теги | Шифрование | Комментарий |
|---------|------|------------|-------------|
| `visits_anonymized` | — | Нет | Статистика визитов (без ID) |
| `diagnoses_stats` | — | Нет | Агрегированная статистика по МКБ |
| `clinic_metrics` | — | Нет | Метрики филиалов |

---

## 8. ElasticSearch (logs)

### Индексы

| Индекс | Теги | Шифрование | Retention | Комментарий |
|--------|------|------------|-----------|-------------|
| `api-logs-*` | `audit` | Нет (данные маскируются на входе) | 90 дней | Логи API-запросов |
| `auth-logs-*` | `audit` | Нет | 180 дней | Логи аутентификации |
| `consent-audit-*` | `audit` | Нет | 365 дней | Логи проверок ИС |

### Маскирование полей в api-logs

| Поле | Маскирование |
|------|--------------|
| `request.body.phone` | `+7***###` |
| `request.body.email` | `***@***` |
| `request.body.first_name` | `***` |
| `request.body.passport_*` | `***` |
| `request.headers.Authorization` | `Bearer ***` |

---

## Сводная таблица: шифрование по сервисам

| Сервис | БД | Метод At Rest | Управление ключами |
|--------|-----|---------------|-------------------|
| CRM API | crm_db (PostgreSQL) | pgcrypto AES-256 | Vault Transit |
| Med Record API | medrecord_db (PostgreSQL) | pgcrypto AES-256 (отдельные ключи для C1) | Vault Transit |
| Consent Manager | consent_db (PostgreSQL) | HMAC для подписи | Vault KV |
| Auth Proxy | Redis | FERNET | Vault KV |
| MinIO | S3 | SSE-S3 (AES-256) | Vault Transit |
| ClickHouse | analytics_db | Column-level encryption (raw_sources) | Vault Transit |
| ElasticSearch | — | TLS in transit, маскирование | — |
| Keycloak | keycloak_db | Встроенное + bcrypt | Realm keys |
