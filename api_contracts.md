# API Contracts

## 1. Общие принципы API

### 1.1 Стиль архитектуры

- REST API с JSON форматом запросов и ответов
- Версионирование через префикс пути: `/api/v1/`
- HTTP методы: GET (чтение), POST (создание), PUT/PATCH (обновление), DELETE (удаление)

### 1.2 Аутентификация и авторизация

- JWT токены (access + refresh)
- Header: `Authorization: Bearer <token>`
- Роли передаются в payload токена и проверяются на backend

### 1.3 Стандартные коды ответов

- `200 OK` — успешный запрос
- `201 Created` — успешное создание ресурса
- `400 Bad Request` — ошибка валидации входных данных
- `401 Unauthorized` — отсутствует или невалидный токен
- `403 Forbidden` — нет прав доступа
- `404 Not Found` — ресурс не найден
- `409 Conflict` — конфликт данных (дубликат, нарушение ограничений)
- `500 Internal Server Error` — серверная ошибка

### 1.4 Формат ошибок

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "phone_number",
        "issue": "must be valid international format"
      }
    ]
  }
}
```

### 1.5 Пагинация

Для списков используется cursor-based или offset-based пагинация:

**Запрос:**
```
GET /api/v1/phone_numbers?limit=100&offset=0
```

**Ответ:**
```json
{
  "data": [...],
  "pagination": {
    "total": 5000000,
    "limit": 100,
    "offset": 0,
    "has_next": true
  }
}
```

---

## 2. Аутентификация

### 2.1 POST /api/v1/auth/login

Вход в систему.

**Запрос:**
```json
{
  "username": "admin",
  "password": "securepassword"
}
```

**Ответ (200 OK):**
```json
{
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "expires_in": 3600,
  "user": {
    "id": 1,
    "username": "admin",
    "role": "administrator"
  }
}
```

### 2.2 POST /api/v1/auth/refresh

Обновление access токена.

**Запрос:**
```json
{
  "refresh_token": "eyJhbGc..."
}
```

**Ответ (200 OK):**
```json
{
  "access_token": "eyJhbGc...",
  "expires_in": 3600
}
```

### 2.3 POST /api/v1/auth/logout

Выход из системы (инвалидация токена).

**Запрос:** пустое тело или refresh_token  
**Ответ (200 OK):**
```json
{
  "message": "Logged out successfully"
}
```

---

## 3. Проекты (Projects)

### 3.1 GET /api/v1/projects

Получение списка проектов.

**Query параметры:**
- `limit` (int, default 50)
- `offset` (int, default 0)
- `status` (string, optional: active, deleted, all)
- `provider_id` (int, optional)
- `geo_id` (int, optional)

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Project Alpha",
      "status": "active",
      "created_at": "2026-01-15T10:00:00Z",
      "updated_at": "2026-02-10T12:30:00Z",
      "providers": [1, 3],
      "geo": [10, 20],
      "weekdays": [1, 2, 3, 4, 5]
    }
  ],
  "pagination": {
    "total": 250,
    "limit": 50,
    "offset": 0,
    "has_next": true
  }
}
```

### 3.2 GET /api/v1/projects/{id}

Получение детальной информации о проекте.

**Ответ (200 OK):**
```json
{
  "id": 1,
  "name": "Project Alpha",
  "status": "active",
  "description": "Main project for region A",
  "created_at": "2026-01-15T10:00:00Z",
  "updated_at": "2026-02-10T12:30:00Z",
  "providers": [
    {"id": 1, "name": "Provider A"},
    {"id": 3, "name": "Provider C"}
  ],
  "geo": [
    {"id": 10, "name": "Moscow", "country": "RU"},
    {"id": 20, "name": "Saint Petersburg", "country": "RU"}
  ],
  "schedules": [
    {"weekday": 1, "start_time": "09:00", "end_time": "18:00"},
    {"weekday": 2, "start_time": "09:00", "end_time": "18:00"}
  ],
  "conversion_stats": {
    "total_numbers": 15000,
    "processed": 12000,
    "successful": 3500,
    "failed": 8500,
    "conversion_rate": 0.233
  }
}
```

### 3.3 POST /api/v1/projects

Создание нового проекта.

**Запрос:**
```json
{
  "name": "Project Beta",
  "description": "New project for testing",
  "provider_ids": [1, 2],
  "geo_ids": [10],
  "schedules": [
    {"weekday": 1, "start_time": "10:00", "end_time": "17:00"}
  ]
}
```

**Ответ (201 Created):**
```json
{
  "id": 2,
  "name": "Project Beta",
  "status": "active",
  "created_at": "2026-02-25T14:00:00Z"
}
```

### 3.4 PATCH /api/v1/projects/{id}

Обновление проекта.

**Запрос:**
```json
{
  "name": "Project Beta Updated",
  "status": "active",
  "geo_ids": [10, 20]
}
```

**Ответ (200 OK):**
```json
{
  "id": 2,
  "name": "Project Beta Updated",
  "updated_at": "2026-02-25T15:00:00Z"
}
```

### 3.5 DELETE /api/v1/projects/{id}

Удаление проекта (мягкое удаление, перенос в deleted_projects).

**Ответ (200 OK):**
```json
{
  "message": "Project moved to deleted",
  "id": 2
}
```

### 3.6 POST /api/v1/projects/{id}/restore

Восстановление удалённого проекта.

**Ответ (200 OK):**
```json
{
  "message": "Project restored",
  "id": 2
}
```

---

## 4. Номера (Phone Numbers)

### 4.1 GET /api/v1/phone_numbers

Получение списка номеров с фильтрами.

**Query параметры:**
- `limit` (int, default 100)
- `offset` (int, default 0)
- `project_id` (int, optional)
- `provider_id` (int, optional)
- `geo_id` (int, optional)
- `status_id` (int, optional)
- `phone_number` (string, optional — поиск по номеру)

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1001,
      "phone_number": "+79161234567",
      "project_id": 1,
      "provider_id": 1,
      "geo_id": 10,
      "status_id": 3,
      "status_name": "successful",
      "created_at": "2026-01-20T08:00:00Z",
      "updated_at": "2026-02-15T12:00:00Z"
    }
  ],
  "pagination": {
    "total": 5000000,
    "limit": 100,
    "offset": 0,
    "has_next": true
  }
}
```

### 4.2 GET /api/v1/phone_numbers/{id}

Получение информации о конкретном номере.

**Ответ (200 OK):**
```json
{
  "id": 1001,
  "phone_number": "+79161234567",
  "project": {"id": 1, "name": "Project Alpha"},
  "provider": {"id": 1, "name": "Provider A"},
  "geo": {"id": 10, "name": "Moscow", "country": "RU"},
  "status": {"id": 3, "name": "successful"},
  "call_history": [
    {
      "call_id": "abc123",
      "timestamp": "2026-02-15T12:00:00Z",
      "duration": 120,
      "result": "answered"
    }
  ],
  "created_at": "2026-01-20T08:00:00Z",
  "updated_at": "2026-02-15T12:00:00Z"
}
```

### 4.3 POST /api/v1/phone_numbers

Добавление нового номера.

**Запрос:**
```json
{
  "phone_number": "+79167654321",
  "project_id": 1,
  "provider_id": 1,
  "geo_id": 10
}
```

**Ответ (201 Created):**
```json
{
  "id": 1002,
  "phone_number": "+79167654321",
  "created_at": "2026-02-25T14:30:00Z"
}
```

### 4.4 PATCH /api/v1/phone_numbers/{id}

Обновление статуса или атрибутов номера.

**Запрос:**
```json
{
  "status_id": 4,
  "notes": "Customer declined"
}
```

**Ответ (200 OK):**
```json
{
  "id": 1002,
  "status_id": 4,
  "updated_at": "2026-02-25T15:00:00Z"
}
```

### 4.5 POST /api/v1/phone_numbers/bulk_update

Массовое обновление статусов номеров.

**Запрос:**
```json
{
  "phone_number_ids": [1001, 1002, 1003],
  "status_id": 5
}
```

**Ответ (200 OK):**
```json
{
  "updated_count": 3,
  "failed_ids": []
}
```

---

## 5. Поставщики (Providers)

### 5.1 GET /api/v1/providers

Список поставщиков.

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Provider A",
      "type": "telecom",
      "status": "active",
      "contact_email": "info@provider-a.com"
    }
  ]
}
```

### 5.2 POST /api/v1/providers

Создание нового поставщика.

**Запрос:**
```json
{
  "name": "Provider D",
  "type": "telecom",
  "contact_email": "contact@provider-d.com"
}
```

**Ответ (201 Created):**
```json
{
  "id": 4,
  "name": "Provider D",
  "created_at": "2026-02-25T14:00:00Z"
}
```

---

## 6. Гео (Geo)

### 6.1 GET /api/v1/geo

Список географических объектов.

**Query параметры:**
- `country` (string, optional)
- `region` (string, optional)

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 10,
      "name": "Moscow",
      "country": "RU",
      "region": "Central",
      "timezone": "Europe/Moscow"
    }
  ]
}
```

### 6.2 POST /api/v1/geo

Добавление нового гео.

**Запрос:**
```json
{
  "name": "Novosibirsk",
  "country": "RU",
  "region": "Siberia",
  "timezone": "Asia/Novosibirsk"
}
```

**Ответ (201 Created):**
```json
{
  "id": 30,
  "name": "Novosibirsk",
  "created_at": "2026-02-25T14:00:00Z"
}
```

---

## 7. Конверсия проектов (Project Conversion)

### 7.1 GET /api/v1/projects/{id}/conversion

Получение статистики конверсии проекта.

**Ответ (200 OK):**
```json
{
  "project_id": 1,
  "total_numbers": 15000,
  "processed": 12000,
  "successful": 3500,
  "failed": 8500,
  "conversion_rate": 0.233,
  "breakdown_by_status": [
    {"status_id": 1, "status_name": "new", "count": 3000},
    {"status_id": 2, "status_name": "in_progress", "count": 0},
    {"status_id": 3, "status_name": "successful", "count": 3500},
    {"status_id": 4, "status_name": "failed", "count": 8500}
  ],
  "updated_at": "2026-02-25T13:00:00Z"
}
```

### 7.2 GET /api/v1/conversion/summary

Общая сводка конверсии по всем проектам.

**Query параметры:**
- `date_from` (date, optional)
- `date_to` (date, optional)

**Ответ (200 OK):**
```json
{
  "projects": [
    {
      "project_id": 1,
      "project_name": "Project Alpha",
      "conversion_rate": 0.233
    },
    {
      "project_id": 2,
      "project_name": "Project Beta",
      "conversion_rate": 0.145
    }
  ],
  "total_numbers": 25000,
  "total_successful": 4725,
  "average_conversion_rate": 0.189
}
```

---

## 8. Удалённые проекты (Deleted Projects)

### 8.1 GET /api/v1/projects/deleted

Список удалённых проектов.

**Query параметры:**
- `limit` (int, default 50)
- `offset` (int, default 0)

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 5,
      "name": "Old Project",
      "deleted_at": "2026-02-01T10:00:00Z",
      "deleted_by": "admin"
    }
  ],
  "pagination": {
    "total": 10,
    "limit": 50,
    "offset": 0,
    "has_next": false
  }
}
```

---

## 9. Импорт/Экспорт номеров (Imports/Exports)

### 9.1 GET /api/v1/exports

Список задач экспорта.

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "filename": "export_2026-02-25.csv",
      "status": "completed",
      "created_at": "2026-02-25T10:00:00Z",
      "completed_at": "2026-02-25T10:05:00Z",
      "download_url": "/api/v1/exports/1/download"
    }
  ]
}
```

### 9.2 POST /api/v1/exports

Запуск экспорта номеров.

**Запрос:**
```json
{
  "project_id": 1,
  "format": "csv",
  "filters": {
    "status_id": 3,
    "date_from": "2026-02-01",
    "date_to": "2026-02-25"
  }
}
```

**Ответ (201 Created):**
```json
{
  "id": 2,
  "status": "processing",
  "created_at": "2026-02-25T14:00:00Z"
}
```

### 9.3 GET /api/v1/exports/{id}/download

Скачивание файла экспорта.

**Ответ (200 OK):**  
Файл в формате CSV/XLS.

### 9.4 POST /api/v1/imports

Запуск импорта номеров.

**Запрос (multipart/form-data):**
- `file`: CSV/XLS файл
- `project_id`: ID проекта
- `provider_id`: ID поставщика
- `geo_id`: ID гео

**Ответ (201 Created):**
```json
{
  "id": 3,
  "status": "processing",
  "filename": "import_numbers.csv",
  "created_at": "2026-02-25T14:00:00Z"
}
```

### 9.5 GET /api/v1/imports/{id}

Статус задачи импорта.

**Ответ (200 OK):**
```json
{
  "id": 3,
  "status": "completed",
  "filename": "import_numbers.csv",
  "total_rows": 10000,
  "imported_rows": 9850,
  "failed_rows": 150,
  "errors": [
    {"row": 5, "error": "invalid phone format"},
    {"row": 123, "error": "duplicate number"}
  ],
  "created_at": "2026-02-25T14:00:00Z",
  "completed_at": "2026-02-25T14:10:00Z"
}
```

---

## 10. Интеграция SFTP (SFTP Jobs)

### 10.1 GET /api/v1/sftp/jobs

Список SFTP задач.

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "type": "export",
      "sftp_host": "sftp.partner.com",
      "remote_path": "/uploads/export_20260225.csv",
      "status": "completed",
      "started_at": "2026-02-25T12:00:00Z",
      "completed_at": "2026-02-25T12:05:00Z"
    }
  ]
}
```

### 10.2 POST /api/v1/sftp/jobs

Ручной запуск SFTP задачи.

**Запрос:**
```json
{
  "type": "import",
  "sftp_config_id": 1,
  "remote_path": "/downloads/numbers.csv",
  "project_id": 1
}
```

**Ответ (201 Created):**
```json
{
  "id": 2,
  "status": "scheduled",
  "created_at": "2026-02-25T14:00:00Z"
}
```

### 10.3 GET /api/v1/sftp/configs

Список SFTP конфигураций.

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Partner A SFTP",
      "host": "sftp.partner-a.com",
      "port": 22,
      "username": "crm_user",
      "auth_method": "key",
      "remote_directory": "/data/exchange",
      "status": "active"
    }
  ]
}
```

### 10.4 POST /api/v1/sftp/configs

Создание новой SFTP конфигурации.

**Запрос:**
```json
{
  "name": "Partner B SFTP",
  "host": "sftp.partner-b.com",
  "port": 22,
  "username": "crm_user",
  "auth_method": "password",
  "password": "securepass",
  "remote_directory": "/exchange"
}
```

**Ответ (201 Created):**
```json
{
  "id": 2,
  "name": "Partner B SFTP",
  "created_at": "2026-02-25T14:00:00Z"
}
```

---

## 11. Интеграция SoftPhone

### 11.1 POST /api/v1/softphone/events

Получение события от SoftPhone (webhook).

**Запрос:**
```json
{
  "event_type": "call_started",
  "call_id": "abc123",
  "phone_number": "+79161234567",
  "direction": "outbound",
  "timestamp": "2026-02-25T14:00:00Z"
}
```

**Ответ (200 OK):**
```json
{
  "status": "event_received",
  "phone_number_id": 1001,
  "project_id": 1
}
```

### 11.2 POST /api/v1/softphone/call_result

Запись результата звонка.

**Запрос:**
```json
{
  "call_id": "abc123",
  "phone_number_id": 1001,
  "result": "answered",
  "duration": 120,
  "notes": "Customer interested"
}
```

**Ответ (200 OK):**
```json
{
  "message": "Call result saved",
  "phone_number_id": 1001,
  "updated_status_id": 3
}
```

---

## 12. Аудит-лог (Audit Log)

### 12.1 GET /api/v1/audit_log

Просмотр аудит-лога (только для Администратора).

**Query параметры:**
- `limit` (int, default 100)
- `offset` (int, default 0)
- `user_id` (int, optional)
- `action` (string, optional: create, update, delete, export, import)
- `date_from` (date, optional)
- `date_to` (date, optional)

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1001,
      "user_id": 1,
      "username": "admin",
      "action": "update",
      "entity_type": "project",
      "entity_id": 1,
      "changes": {
        "name": {"old": "Project Alpha", "new": "Project Alpha Updated"}
      },
      "ip_address": "192.168.1.100",
      "timestamp": "2026-02-25T13:00:00Z"
    }
  ],
  "pagination": {
    "total": 50000,
    "limit": 100,
    "offset": 0,
    "has_next": true
  }
}
```

---

## 13. Пользователи и роли (Users & Roles)

### 13.1 GET /api/v1/users

Список пользователей (только Администратор).

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "username": "admin",
      "email": "admin@crm.local",
      "role": "administrator",
      "status": "active",
      "created_at": "2026-01-01T00:00:00Z"
    },
    {
      "id": 2,
      "username": "manager1",
      "email": "manager1@crm.local",
      "role": "manager",
      "status": "active",
      "created_at": "2026-01-15T10:00:00Z"
    }
  ]
}
```

### 13.2 POST /api/v1/users

Создание пользователя.

**Запрос:**
```json
{
  "username": "manager2",
  "email": "manager2@crm.local",
  "password": "securepass123",
  "role": "manager"
}
```

**Ответ (201 Created):**
```json
{
  "id": 3,
  "username": "manager2",
  "created_at": "2026-02-25T14:00:00Z"
}
```

### 13.3 GET /api/v1/roles

Список доступных ролей.

**Ответ (200 OK):**
```json
{
  "data": [
    {
      "id": 1,
      "name": "administrator",
      "permissions": ["all"]
    },
    {
      "id": 2,
      "name": "manager",
      "permissions": [
        "projects.read",
        "projects.update",
        "phone_numbers.read",
        "phone_numbers.update",
        "exports.create"
      ]
    }
  ]
}
```

---

## 14. Использование ИИ-помощников для работы с API

- ИИ может генерировать клиентский код для вызова API (axios/fetch для frontend, requests для Python)
- ИИ может создавать mock-данные для тестирования API эндпоинтов
- ИИ может генерировать OpenAPI/Swagger спецификацию на основе этого документа
- Все сгенерированные интеграции проходят ручное тестирование перед использованием в продакшене
