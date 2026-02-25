# backend_plan_dev1.md

## Цели backend‑части

Реализовать stateless REST API для управления многомиллионными базами номеров, проектами, поставщиками, гео и связанными сущностями, с поддержкой аутентификации, ролей, интеграций SFTP/SoftPhone и аудит‑логирования.

## Выбранный стек backend + БД

### Рекомендуемый стек

- **Язык**: Python 3.11+
- **Фреймворк**: Django 4.2+ с Django REST Framework (DRF) 3.14+
- **БД**: PostgreSQL 15+
- **Очереди**: Celery + Redis (для фоновых задач SFTP, массовых операций)
- **Аутентификация**: JWT (djangorestframework‑simplejwt или аналог)
- **SFTP**: paramiko или pysftp
- **Тесты**: pytest + pytest‑django

### Альтернатива

- **Язык**: Node.js (TypeScript)
- **Фреймворк**: NestJS 10+
- **БД**: PostgreSQL 15+
- **Очереди**: BullMQ + Redis
- **ORM**: TypeORM или Prisma
- **Тесты**: Jest

## Проектирование схемы БД под большие объемы номеров

### Основные таблицы

#### 1. users
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(150) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'manager', -- 'admin', 'manager'
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_is_active ON users(is_active);
```

#### 2. projects
```sql
CREATE TABLE projects (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(50) DEFAULT 'active', -- 'active', 'paused', 'deleted'
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP,
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_is_deleted ON projects(is_deleted);
CREATE INDEX idx_projects_created_at ON projects(created_at DESC);
```

#### 3. providers
```sql
CREATE TABLE providers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(100), -- тип поставщика
    contact_info JSONB, -- email, phone, etc.
    status VARCHAR(50) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_providers_status ON providers(status);
```

#### 4. geo
```sql
CREATE TABLE geo (
    id BIGSERIAL PRIMARY KEY,
    country_code VARCHAR(10),
    region VARCHAR(255),
    city VARCHAR(255),
    timezone VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_geo_country_code ON geo(country_code);
```

#### 5. phone_numbers
```sql
CREATE TABLE phone_numbers (
    id BIGSERIAL PRIMARY KEY,
    phone_number VARCHAR(20) NOT NULL,
    project_id BIGINT REFERENCES projects(id) ON DELETE SET NULL,
    provider_id BIGINT REFERENCES providers(id) ON DELETE SET NULL,
    geo_id BIGINT REFERENCES geo(id) ON DELETE SET NULL,
    status VARCHAR(50) DEFAULT 'new', -- 'new', 'in_progress', 'success', 'failed', 'no_answer'
    call_attempts INT DEFAULT 0,
    last_call_at TIMESTAMP,
    result_data JSONB, -- результаты звонков
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_phone_numbers_phone ON phone_numbers(phone_number);
CREATE INDEX idx_phone_numbers_project_id ON phone_numbers(project_id);
CREATE INDEX idx_phone_numbers_status ON phone_numbers(status);
CREATE INDEX idx_phone_numbers_provider_id ON phone_numbers(provider_id);
CREATE INDEX idx_phone_numbers_geo_id ON phone_numbers(geo_id);
CREATE INDEX idx_phone_numbers_created_at ON phone_numbers(created_at DESC);
```

#### 6. project_providers (связь many‑to‑many)
```sql
CREATE TABLE project_providers (
    id BIGSERIAL PRIMARY KEY,
    project_id BIGINT REFERENCES projects(id) ON DELETE CASCADE,
    provider_id BIGINT REFERENCES providers(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, provider_id)
);
CREATE INDEX idx_project_providers_project ON project_providers(project_id);
CREATE INDEX idx_project_providers_provider ON project_providers(provider_id);
```

#### 7. project_geo (связь many‑to‑many)
```sql
CREATE TABLE project_geo (
    id BIGSERIAL PRIMARY KEY,
    project_id BIGINT REFERENCES projects(id) ON DELETE CASCADE,
    geo_id BIGINT REFERENCES geo(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, geo_id)
);
CREATE INDEX idx_project_geo_project ON project_geo(project_id);
CREATE INDEX idx_project_geo_geo ON project_geo(geo_id);
```

#### 8. project_schedules
```sql
CREATE TABLE project_schedules (
    id BIGSERIAL PRIMARY KEY,
    project_id BIGINT REFERENCES projects(id) ON DELETE CASCADE,
    weekday INT NOT NULL, -- 0=Пн, 6=Вс
    start_time TIME,
    end_time TIME,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_project_schedules_project ON project_schedules(project_id);
CREATE INDEX idx_project_schedules_weekday ON project_schedules(weekday);
```

#### 9. project_conversions (предрасчитанные метрики)
```sql
CREATE TABLE project_conversions (
    id BIGSERIAL PRIMARY KEY,
    project_id BIGINT REFERENCES projects(id) ON DELETE CASCADE,
    total_numbers INT DEFAULT 0,
    processed_numbers INT DEFAULT 0,
    success_numbers INT DEFAULT 0,
    failed_numbers INT DEFAULT 0,
    conversion_rate NUMERIC(5,2), -- %
    calculated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, calculated_at)
);
CREATE INDEX idx_project_conversions_project ON project_conversions(project_id);
CREATE INDEX idx_project_conversions_calculated_at ON project_conversions(calculated_at DESC);
```

#### 10. sftp_jobs
```sql
CREATE TABLE sftp_jobs (
    id BIGSERIAL PRIMARY KEY,
    job_type VARCHAR(50) NOT NULL, -- 'import', 'export'
    project_id BIGINT REFERENCES projects(id) ON DELETE SET NULL,
    file_name VARCHAR(255),
    file_path TEXT,
    status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'running', 'completed', 'failed'
    error_log TEXT,
    records_count INT DEFAULT 0,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_sftp_jobs_status ON sftp_jobs(status);
CREATE INDEX idx_sftp_jobs_created_at ON sftp_jobs(created_at DESC);
```

#### 11. audit_log
```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL, -- 'create_project', 'update_number', etc.
    entity_type VARCHAR(50), -- 'project', 'phone_number', etc.
    entity_id BIGINT,
    changes JSONB, -- изменения (old_value, new_value)
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_audit_log_user_id ON audit_log(user_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_created_at ON audit_log(created_at DESC);
```

### Оптимизация для больших объемов

- **Индексы**: по всем ключевым полям (phone_number, project_id, status, created_at)
- **Партиционирование**: таблица phone_numbers может быть партиционирована по project_id или created_at для больших объемов
- **Batched операции**: bulk_create, bulk_update в Django ORM
- **Connection pooling**: pgbouncer для PostgreSQL

## Реализация основных модулей

### Модуль 1: Сущности и бизнес‑логика

**Модели Django (models.py)**:
- User (расширение AbstractUser с полем role)
- Project
- Provider
- Geo
- PhoneNumber
- ProjectProvider (many‑to‑many)
- ProjectGeo (many‑to‑many)
- ProjectSchedule
- ProjectConversion
- SftpJob
- AuditLog

**Serializers (serializers.py)**:
- Для каждой модели: чтение (read) и запись (write) serializers
- Валидация входных данных (phone_number format, unique constraints)

**ViewSets (views.py)**:
- ProjectViewSet (CRUD, фильтры, пагинация)
- PhoneNumberViewSet (с оптимизацией запросов для больших таблиц)
- ProviderViewSet
- GeoViewSet
- ProjectConversionViewSet (read‑only)

### Модуль 2: SFTP‑интеграция

**Компоненты**:
- `sftp_client.py`: класс SFTPClient с методами connect, upload, download, list_files
- `sftp_tasks.py`: Celery‑задачи для импорта и экспорта
- `sftp_parsers.py`: парсеры CSV/XLS файлов с номерами
- `sftp_config.py`: конфигурация SFTP‑хостов (чтение из env или БД)

**API**:
- `POST /api/sftp/import/`: запуск импорта номеров из SFTP
- `POST /api/sftp/export/`: запуск экспорта номеров на SFTP
- `GET /api/sftp/jobs/`: просмотр статусов задач

### Модуль 3: Точка расширения для SoftPhone

**Компоненты**:
- `softphone_webhook.py`: endpoint для приема событий звонков
- `call_events.py`: модель CallEvent для хранения событий
- `phone_number_matcher.py`: поиск номера в БД по phone_number из события

**API**:
- `POST /api/softphone/webhook/`: прием событий (call_started, call_ended, call_result)
- `GET /api/softphone/number/{phone}/`: получение информации о номере для pop‑up

### Модуль 4: Система ролей и прав

**Компоненты**:
- `permissions.py`: кастомные DRF permissions (IsAdmin, IsManager)
- `role_decorators.py`: декораторы для проверки ролей
- `access_control.py`: RBAC‑логика (проверка доступа к проектам/номерам)

**Логика**:
- Администратор: полный доступ
- Менеджер: доступ к проектам/номерам по фильтру (queryset filtering в ViewSet)

### Модуль 5: Аудит‑логирование

**Компоненты**:
- `audit_middleware.py`: middleware для автоматической записи действий
- `audit_logger.py`: утилита для записи в audit_log
- `audit_signals.py`: Django signals для логирования изменений моделей

**Действия для логирования**:
- Создание/обновление/удаление проектов, номеров, поставщиков
- Изменение статусов номеров
- Запуск SFTP‑задач
- Вход/выход пользователей

## Структура API (основные эндпоинты)

### Аутентификация
- `POST /api/auth/login/`: вход (получение JWT)
- `POST /api/auth/refresh/`: обновление токена
- `POST /api/auth/logout/`: выход

### Проекты
- `GET /api/projects/`: список проектов (с фильтрами, пагинацией)
- `POST /api/projects/`: создание проекта
- `GET /api/projects/{id}/`: детали проекта
- `PUT/PATCH /api/projects/{id}/`: обновление
- `DELETE /api/projects/{id}/`: удаление (мягкое, is_deleted=True)
- `GET /api/projects/deleted/`: список удаленных проектов
- `POST /api/projects/{id}/restore/`: восстановление

### Номера
- `GET /api/phone-numbers/`: список номеров (фильтры: project_id, status, provider_id, geo_id)
- `POST /api/phone-numbers/`: создание номера
- `POST /api/phone-numbers/bulk/`: массовое создание
- `PATCH /api/phone-numbers/{id}/`: обновление статуса
- `POST /api/phone-numbers/bulk-update/`: массовое обновление

### Поставщики и Гео
- `GET /api/providers/`: список поставщиков
- `POST /api/providers/`: создание
- `GET /api/geo/`: список гео
- `POST /api/geo/`: создание

### Конверсия
- `GET /api/projects/{id}/conversion/`: метрики конверсии проекта
- `POST /api/projects/{id}/recalculate-conversion/`: пересчет метрик

### SFTP
- `POST /api/sftp/import/`: импорт номеров
- `POST /api/sftp/export/`: экспорт номеров
- `GET /api/sftp/jobs/`: статусы задач

### SoftPhone
- `POST /api/softphone/webhook/`: webhook для событий звонков
- `GET /api/softphone/number/{phone}/`: информация о номере

### Аудит
- `GET /api/audit-log/`: просмотр аудит‑лога (только для Администратора)

## Этапы/спринты (структура технических задач)

### Спринт 1: Базовая инфраструктура (1–2 недели)

1. Настройка Django‑проекта, структура приложения
2. Настройка PostgreSQL, подключение через docker‑compose
3. Создание моделей Django (User, Project, Provider, Geo, PhoneNumber, и др.)
4. Создание миграций и применение к БД
5. Настройка Django REST Framework
6. Реализация JWT‑аутентификации
7. Создание базовых serializers и ViewSets

### Спринт 2: Ключевые сущности (2 недели)

1. Реализация ProjectViewSet с CRUD и фильтрацией
2. Реализация PhoneNumberViewSet с оптимизацией запросов
3. Массовые операции (bulk_create, bulk_update) для номеров
4. Реализация удаленных проектов (мягкое удаление, восстановление)
5. Реализация ProviderViewSet и GeoViewSet
6. Создание связей many‑to‑many (project_providers, project_geo)
7. Реализация расписаний проектов (ProjectSchedule)

### Спринт 3: Конверсия и роли (1–2 недели)

1. Реализация модели ProjectConversion
2. Расчет метрик конверсии (фоновая задача или ручной запуск)
3. API для просмотра конверсии проекта
4. Реализация кастомных permissions (IsAdmin, IsManager)
5. Разграничение доступа по ролям в ViewSets (queryset filtering)
6. Тестирование ролевой модели

### Спринт 4: Интеграции (2 недели)

1. Настройка Celery + Redis
2. Реализация SFTP‑клиента (SFTPClient)
3. Реализация парсеров CSV/XLS
4. Celery‑задачи для импорта/экспорта через SFTP
5. API для запуска SFTP‑задач и просмотра статусов
6. Реализация webhook для SoftPhone
7. Логика привязки событий звонков к номерам

### Спринт 5: Аудит и финализация (1 неделя)

1. Реализация audit_middleware и audit_logger
2. Настройка Django signals для логирования изменений
3. API для просмотра аудит‑лога
4. Написание тестов (pytest) для ключевых модулей
5. Оптимизация запросов (select_related, prefetch_related)
6. Настройка docker‑compose для локального запуска
7. Документирование API (OpenAPI/Swagger)

## Зависимости от frontend и требования к интеграции

### Контракты API

- Все API отвечают в формате JSON
- Стандартные HTTP‑коды (200, 201, 400, 401, 403, 404, 500)
- Пагинация: `{"count": 1000, "next": "...", "previous": "...", "results": [...]}`
- Ошибки: `{"detail": "..."}` или `{"field": ["error message"]}`

### Требования к frontend

- Отправка JWT‑токена в заголовке `Authorization: Bearer <token>`
- Обработка пагинации и фильтров на клиенте
- Обработка ошибок (401 → перенаправление на логин, 403 → сообщение о нехватке прав)
- Поддержка массовых операций (bulk‑эндпоинты)

## Использование ИИ‑помощников

### Генерация кода

- Используйте ИИ для генерации типовых serializers, ViewSets, фильтров
- Пример промпта: "Создай Django REST serializer для модели PhoneNumber с полями phone_number, project, status, с валидацией формата номера"

### Генерация миграций

- Django автоматически создает миграции через `python manage.py makemigrations`
- ИИ может помочь с написанием custom SQL‑миграций (для индексов, партиционирования)

### Написание тестов

- Пример промпта: "Создай pytest‑тест для ProjectViewSet, проверяющий создание, обновление и удаление проекта"
- Все генерируемые тесты должны быть проверены вручную

### Обновление .md‑документации

- При изменении API используйте ИИ для автоматического обновления api_contracts.md
- Пример промпта: "Обнови api_contracts.md, добавив новый эндпоинт POST /api/phone‑numbers/bulk/ с параметрами..."

### Важно

Любые решения и код, предложенные ИИ, проходят ручное ревью и тестирование перед включением в основной код.
