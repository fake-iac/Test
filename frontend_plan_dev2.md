# frontend_plan_dev2.md

## Цели frontend‑части

Разработать интерфейс CRM для управления крупномасштабными базами телефонных номеров (миллионы записей) с поддержкой:
- Ролей Администратор и Менеджер с разграничением прав доступа.
- Работы с большими списками через серверную пагинацию и фильтрацию.
- Ключевых экранов: проекты, номера, конверсия, удалённые проекты, выгрузки/импорты.
- Интеграции с backend REST API.
- Локального запуска на одном ПК с возможностью переноса в облако.

## Выбранный стек frontend

### Рекомендуемый стек

- **Фреймворк**: React 18+ с TypeScript.
- **Управление состоянием**: Zustand или Redux Toolkit (предпочтительно Zustand для простоты).
- **Роутинг**: React Router v6.
- **UI библиотека**: Material-UI (MUI) или Ant Design для готовых компонентов таблиц, форм, модальных окон.
- **HTTP клиент**: Axios с интерцепторами для аутентификации и обработки ошибок.
- **Таблицы**: TanStack Table (react-table v8) для серверной пагинации и фильтрации.
- **Валидация форм**: React Hook Form + Zod.
- **Сборка**: Vite.

### Альтернативный стек

- **Фреймворк**: Vue 3 + TypeScript (Composition API).
- **UI библиотека**: Element Plus или Vuetify.
- **Таблицы**: AG Grid Community или встроенные компоненты UI библиотеки.

Для унификации с backend (Django REST) рекомендуется **React + TypeScript**.

## Проектирование интерфейсов для ролей

### Общая структура приложения

```
src/
├── api/              # API клиент и эндпоинты
├── components/       # Переиспользуемые компоненты
├── features/         # Модули по функциональности
│   ├── auth/        # Аутентификация
│   ├── projects/    # Управление проектами
│   ├── numbers/     # Управление номерами
│   ├── conversion/  # Конверсия проектов
│   ├── deleted/     # Удалённые проекты
│   ├── exports/     # Выгрузки/импорты
│   ├── audit/       # Аудит-лог (только Администратор)
│   └── settings/    # Настройки (только Администратор)
├── hooks/            # Custom hooks
├── layouts/          # Layout компоненты
├── store/            # Глобальное состояние
├── types/            # TypeScript типы и интерфейсы
├── utils/            # Утилиты
└── App.tsx
```

### Роль Администратор

**Доступные разделы (навигация)**:
- Проекты
- Номера
- Поставщики
- Гео
- Конверсия проектов
- Удалённые проекты
- Выгрузки/Импорты
- Отчёты
- Аудит-лог
- Настройки (роли, пользователи, SFTP, SoftPhone)

**Права**:
- Полный CRUD на все сущности.
- Доступ к системным настройкам и интеграциям.
- Просмотр всех записей аудит-лога.

### Роль Менеджер

**Доступные разделы**:
- Проекты (ограниченный список по фильтру).
- Номера (только в рамках доступных проектов).
- Конверсия проектов (базовая статистика).
- Выгрузки (ограниченный функционал).

**Права**:
- Просмотр и изменение статусов номеров.
- Создание проектов (по политике).
- Нет доступа к настройкам, аудит-логу, удалённым проектам, управлению ролями.

### Реализация разграничения прав

- **Route guards**: компонент `ProtectedRoute` проверяет роль пользователя перед рендером страницы.
- **Условный рендеринг**: кнопки и пункты меню отображаются в зависимости от прав.
- **API level**: backend всегда проверяет права, frontend только скрывает UI.

```typescript
// types/user.ts
export enum UserRole {
  ADMIN = 'ADMIN',
  MANAGER = 'MANAGER'
}

export interface User {
  id: string;
  username: string;
  role: UserRole;
  permissions: string[];
}

// hooks/usePermissions.ts
export const usePermissions = () => {
  const user = useAuthStore(state => state.user);
  
  const can = (permission: string) => {
    return user?.permissions.includes(permission) || false;
  };
  
  const isAdmin = user?.role === UserRole.ADMIN;
  const isManager = user?.role === UserRole.MANAGER;
  
  return { can, isAdmin, isManager };
};
```

## Реализация ключевых экранов

### 1. Управление проектами

**Путь**: `/projects`

**Компоненты**:
- `ProjectsTable`: таблица с серверной пагинацией, фильтрами (поставщик, гео, статус, дата).
- `ProjectForm`: форма создания/редактирования проекта с выбором поставщиков, гео, дней недели.
- `ProjectDetails`: детальная страница проекта с вкладками (номера, статистика, история).

**Функционал**:
- Список проектов с поиском и фильтрацией.
- Создание проекта (мультиселект поставщиков, гео; чекбоксы дней недели).
- Редактирование параметров проекта.
- Удаление проекта (мягкое удаление, перенос в «Удалённые проекты»).
- Массовые операции: экспорт выбранных проектов в CSV.

**API эндпоинты**:
- `GET /api/projects?page=1&per_page=50&provider=X&geo=Y&status=active`
- `POST /api/projects`
- `PUT /api/projects/:id`
- `DELETE /api/projects/:id`
- `GET /api/projects/:id`

### 2. Выбор поставщика/гео/дней недели

**Путь**: `/projects/:id/settings`

**Компоненты**:
- `ProviderSelector`: мультиселект с поиском поставщиков.
- `GeoSelector`: иерархический селектор (страна → регион → город) или flat list с поиском.
- `WeekdaySchedule`: grid из 7 чекбоксов (ПН-ВС) с опциональными временными интервалами.

**Функционал**:
- Выбор одного или нескольких поставщиков для проекта.
- Выбор гео (мультиселект).
- Настройка расписания по дням недели.
- Сохранение изменений с валидацией (минимум 1 поставщик, 1 гео).

**API эндпоинты**:
- `GET /api/providers?search=keyword`
- `GET /api/geo?parent_id=X`
- `PUT /api/projects/:id/settings`

### 3. Конверсия проектов

**Путь**: `/conversion`

**Компоненты**:
- `ConversionTable`: таблица проектов с метриками (всего номеров, обработано, успешно, конверсия %).
- `ConversionChart`: графики динамики конверсии по дням/неделям (опционально, библиотека Chart.js или Recharts).
- `ConversionFilters`: фильтры по датам, поставщикам, гео.

**Функционал**:
- Отображение агрегированных метрик по проектам.
- Фильтрация и сортировка.
- Экспорт отчёта в CSV/Excel.
- Drill-down: клик на проект → детальная статистика по номерам.

**API эндпоинты**:
- `GET /api/conversion?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD&provider=X`
- `GET /api/projects/:id/conversion-details`

### 4. Удалённые проекты

**Путь**: `/deleted-projects` (только Администратор)

**Компоненты**:
- `DeletedProjectsTable`: таблица с датой удаления, причиной, автором.
- `RestoreButton`: кнопка восстановления проекта.
- `PermanentDeleteButton`: кнопка окончательного удаления (с подтверждением).

**Функционал**:
- Просмотр удалённых проектов.
- Восстановление проекта (перенос обратно в активные).
- Окончательное удаление (hard delete).

**API эндпоинты**:
- `GET /api/projects/deleted?page=1`
- `POST /api/projects/:id/restore`
- `DELETE /api/projects/:id/permanent`

### 5. Страницы выгрузки и фильтрации номеров

**Путь**: `/numbers`, `/exports`

**Компоненты**:
- `NumbersTable`: таблица номеров с серверной пагинацией (по 100-500 записей на страницу).
- `NumberFilters`: фильтры (проект, статус, поставщик, гео, дата добавления).
- `ExportForm`: форма создания выгрузки (выбор формата CSV/XLS, фильтров, опция SFTP).
- `ExportJobsList`: список запущенных и завершённых выгрузок со статусами и ссылками на скачивание.

**Функционал**:
- Просмотр номеров с расширенными фильтрами.
- Массовое изменение статусов выбранных номеров.
- Создание задачи на выгрузку номеров с фильтрами.
- Мониторинг статуса выгрузки (pending, processing, completed, failed).
- Скачивание готового файла.
- Настройка автоматической отправки выгрузки по SFTP (для Администратора).

**API эндпоинты**:
- `GET /api/numbers?page=1&per_page=100&project=X&status=Y`
- `PATCH /api/numbers/bulk-update` (body: `{ids: [...], status: 'new_status'}`)
- `POST /api/exports` (body: `{filters: {...}, format: 'csv', sftp_enabled: true}`)
- `GET /api/exports?page=1`
- `GET /api/exports/:id/download`

### 6. Просмотр отчётов и аудит-лога

**Путь**: `/reports`, `/audit` (аудит только для Администратора)

**Компоненты**:
- `ReportsMenu`: список доступных отчётов (конверсия, активность по дням, топ поставщиков).
- `AuditLogTable`: таблица с записями (дата, пользователь, действие, сущность, изменения).
- `AuditLogFilters`: фильтры по пользователю, датам, типу действия.

**Функционал**:
- Просмотр предзаготовленных отчётов.
- Фильтрация и поиск в аудит-логе.
- Экспорт аудит-лога в CSV.

**API эндпоинты**:
- `GET /api/reports/conversion-summary`
- `GET /api/audit-log?page=1&user_id=X&start_date=Y`

## Подход к работе с большими списками

### Серверная пагинация

- Таблицы всегда используют серверную пагинацию (page, per_page).
- Backend возвращает метаданные: `{data: [...], total: 1000000, page: 1, per_page: 50, total_pages: 20000}`.
- Frontend отображает номер страницы, переключатель и информацию "Показано 1-50 из 1,000,000".

### Фильтры и поиск

- Фильтры применяются на сервере через query параметры.
- Debounce для поля поиска (300ms задержка перед запросом).
- Индикатор загрузки (skeleton или spinner) во время запроса.

### Массовые действия

- Выбор записей через чекбоксы (в заголовке таблицы — "выбрать все на текущей странице").
- Для операций с большим количеством записей (>1000) — создание фоновой задачи на backend.
- Frontend отображает уведомление: "Задача создана, вы получите результат через N минут" и ссылку на статус задачи.

### Виртуализация (опционально)

- Для списков с очень большим количеством элементов на одной странице (500+) можно использовать `react-window` или `react-virtualized`.
- В большинстве случаев достаточно пагинации по 50-100 записей.

## Интеграция с API backend

### HTTP клиент (Axios)

```typescript
// api/client.ts
import axios from 'axios';
import { getAuthToken } from '@/store/authStore';

const apiClient = axios.create({
  baseURL: process.env.VITE_API_BASE_URL || 'http://localhost:8000/api',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json'
  }
});

apiClient.interceptors.request.use(
  (config) => {
    const token = getAuthToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### API сервисы по сущностям

```typescript
// api/projects.ts
import apiClient from './client';
import { Project, ProjectFilters, PaginatedResponse } from '@/types';

export const projectsApi = {
  getAll: (filters: ProjectFilters) => 
    apiClient.get<PaginatedResponse<Project>>('/projects', { params: filters }),
  
  getById: (id: string) => 
    apiClient.get<Project>(`/projects/${id}`),
  
  create: (data: Partial<Project>) => 
    apiClient.post<Project>('/projects', data),
  
  update: (id: string, data: Partial<Project>) => 
    apiClient.put<Project>(`/projects/${id}`, data),
  
  delete: (id: string) => 
    apiClient.delete(`/projects/${id}`)
};
```

### Обработка ответов и ошибок

- Успешные ответы: отображение toast-уведомления "Проект создан успешно".
- Ошибки валидации (400): отображение ошибок под соответствующими полями формы.
- Ошибки сервера (500): глобальное уведомление "Произошла ошибка, попробуйте позже".
- Сетевые ошибки: "Нет связи с сервером".

### Управление состоянием загрузки

```typescript
// features/projects/useProjects.ts
import { useState, useEffect } from 'react';
import { projectsApi } from '@/api/projects';

export const useProjects = (filters: ProjectFilters) => {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [pagination, setPagination] = useState({ total: 0, page: 1, per_page: 50 });

  useEffect(() => {
    const fetchProjects = async () => {
      setLoading(true);
      setError(null);
      try {
        const response = await projectsApi.getAll(filters);
        setData(response.data.data);
        setPagination({
          total: response.data.total,
          page: response.data.page,
          per_page: response.data.per_page
        });
      } catch (err: any) {
        setError(err.message || 'Failed to load projects');
      } finally {
        setLoading(false);
      }
    };
    fetchProjects();
  }, [filters]);

  return { data, loading, error, pagination };
};
```

## Этапы/спринты и список технических задач

### Этап 1: Инфраструктура и аутентификация (Спринт 1, 1-2 недели)

**Задачи**:
1. Настроить проект (Vite + React + TypeScript).
2. Настроить структуру папок (features, api, components, store).
3. Установить и настроить UI библиотеку (MUI/Ant Design).
4. Создать API клиент (Axios с интерцепторами).
5. Реализовать страницы Login и Logout.
6. Реализовать хранилище аутентификации (Zustand/Redux).
7. Создать компонент `ProtectedRoute` с проверкой роли.
8. Настроить роутинг (React Router).
9. Создать базовый Layout с навигационным меню.
10. Реализовать условный рендеринг меню по ролям.

**Зависимости**: backend должен предоставить эндпоинты `/api/auth/login`, `/api/auth/logout`, `/api/auth/me`.

### Этап 2: Управление проектами (Спринт 2, 2 недели)

**Задачи**:
1. Создать типы TypeScript для сущностей (Project, Provider, Geo, Weekday).
2. Реализовать API сервисы для проектов (`api/projects.ts`).
3. Создать компонент `ProjectsTable` с серверной пагинацией.
4. Реализовать фильтры проектов (поставщик, гео, статус, даты).
5. Создать форму `ProjectForm` (создание/редактирование).
6. Реализовать мультиселект поставщиков (`ProviderSelector`).
7. Реализовать мультиселект гео (`GeoSelector`).
8. Реализовать компонент расписания дней недели (`WeekdaySchedule`).
9. Реализовать страницу детального просмотра проекта (`ProjectDetails`).
10. Добавить валидацию форм (React Hook Form + Zod).
11. Реализовать удаление проекта (мягкое).

**Зависимости**: backend эндпоинты `/api/projects`, `/api/providers`, `/api/geo`.

### Этап 3: Работа с номерами и выгрузки (Спринт 3, 2-3 недели)

**Задачи**:
1. Создать типы для номеров и выгрузок.
2. Реализовать API сервисы для номеров и выгрузок.
3. Создать таблицу `NumbersTable` с фильтрами и пагинацией.
4. Реализовать массовое изменение статусов номеров.
5. Создать форму выгрузки `ExportForm`.
6. Реализовать список задач выгрузки (`ExportJobsList`).
7. Добавить индикацию статуса задачи (pending, processing, completed, failed).
8. Реализовать скачивание файлов выгрузки.
9. Добавить уведомления о завершении/ошибках выгрузки.

**Зависимости**: backend эндпоинты `/api/numbers`, `/api/exports`.

### Этап 4: Конверсия и аналитика (Спринт 4, 1-2 недели)

**Задачи**:
1. Создать типы для метрик конверсии.
2. Реализовать API сервисы для конверсии.
3. Создать таблицу `ConversionTable` с метриками.
4. Реализовать фильтры по датам, поставщикам, гео.
5. Добавить графики динамики конверсии (Chart.js/Recharts).
6. Реализовать drill-down (клик на проект → детали).
7. Добавить экспорт отчёта в CSV.

**Зависимости**: backend эндпоинты `/api/conversion`, `/api/projects/:id/conversion-details`.

### Этап 5: Удалённые проекты и аудит-лог (Спринт 5, 1 неделя)

**Задачи**:
1. Создать страницу удалённых проектов (только для Администратора).
2. Реализовать таблицу `DeletedProjectsTable`.
3. Реализовать восстановление проекта.
4. Реализовать окончательное удаление проекта.
5. Создать страницу аудит-лога (только для Администратора).
6. Реализовать таблицу `AuditLogTable` с фильтрами.
7. Добавить экспорт аудит-лога в CSV.

**Зависимости**: backend эндпоинты `/api/projects/deleted`, `/api/audit-log`.

### Этап 6: Настройки и интеграции (Спринт 6, 1-2 недели)

**Задачи**:
1. Создать страницу настроек (только для Администратора).
2. Реализовать управление пользователями и ролями.
3. Создать форму настройки SFTP (хост, порт, логин, путь).
4. Реализовать тестирование подключения SFTP.
5. Создать раздел настройки интеграции SoftPhone.
6. Добавить страницу управления справочниками (поставщики, гео, статусы).

**Зависимости**: backend эндпоинты `/api/settings`, `/api/users`, `/api/sftp/test`.

### Этап 7: Тестирование и оптимизация (Спринт 7, 1-2 недели)

**Задачи**:
1. Провести end-to-end тестирование всех функций.
2. Оптимизировать производительность таблиц (lazy loading, мемоизация).
3. Провести тестирование с большими датасетами (>1 млн номеров).
4. Исправить найденные баги.
5. Провести code review и рефакторинг.
6. Написать юнит-тесты для критичных компонентов (Jest + React Testing Library).
7. Подготовить документацию для пользователей (user guide).
8. Настроить сборку для production (минификация, code splitting).
9. Подготовить Docker образ для frontend (nginx).
10. Провести финальное тестирование в production-like окружении.

## Использование ИИ‑помощников для frontend

### Генерация типового кода

- Генерация компонентов таблиц и форм на основе схемы данных.
- Создание API сервисов по OpenAPI спецификации.
- Генерация TypeScript типов из JSON schema или backend моделей.

Пример промпта:
```
Создай React компонент таблицы проектов с серверной пагинацией, 
фильтрами (поставщик, гео, статус) и сортировкой. 
Используй TanStack Table и TypeScript.
```

### Написание тестов

- Генерация тест-кейсов для компонентов (Jest + React Testing Library).
- Создание mock данных для тестирования.

Пример промпта:
```
Напиши юнит-тесты для компонента ProjectForm, 
проверяющие валидацию полей и отправку данных.
```

### Рефакторинг и оптимизация

- Анализ производительности компонентов и предложение оптимизаций (useMemo, useCallback).
- Рефакторинг дублирующегося кода.

Пример промпта:
```
Оптимизируй компонент NumbersTable для работы с большими списками, 
добавь мемоизацию и виртуализацию.
```

### Документация

- Генерация JSDoc комментариев для функций и компонентов.
- Создание Storybook stories для UI компонентов.
- Обновление README.md при добавлении новых фич.

### Важно

- Все сгенерированные ИИ решения проходят ручное ревью разработчиком.
- Код тестируется перед commit в репозиторий.
- ИИ используется как помощник, но не заменяет понимание архитектуры и бизнес-логики.

## Контракты с backend (зависимости)

Frontend разработчик должен получить от backend разработчика:

1. **OpenAPI спецификацию** (или аналог) с описанием всех эндпоинтов.
2. **Форматы запросов/ответов** для каждого API (JSON schema).
3. **Коды ошибок** и их значения.
4. **Механизм аутентификации** (JWT токены, refresh logic).
5. **WebSocket эндпоинты** (если планируется real-time уведомления).
6. **Mock сервер** или sandbox для независимой разработки frontend до готовности backend.

Подробное описание API контрактов см. в `api_contracts.md`.

## Локальный запуск и деплой

### Локальная разработка

```bash
# Установка зависимостей
npm install

# Запуск dev сервера
npm run dev

# Сборка для production
npm run build

# Предпросмотр production сборки
npm run preview
```

### Переменные окружения

```env
# .env.development
VITE_API_BASE_URL=http://localhost:8000/api

# .env.production
VITE_API_BASE_URL=https://api.crm.example.com/api
```

### Docker

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Перенос в облако

- Frontend статические файлы размещаются на CDN (Cloudflare, AWS S3 + CloudFront).
- Nginx конфигурируется для обслуживания SPA (fallback на index.html).
- CORS настраивается на backend для домена frontend.

## Критерии качества

- Все компоненты покрыты TypeScript типами без `any`.
- Код соответствует ESLint правилам (Airbnb style guide).
- Нет console.log и закомментированного кода в production сборке.
- Критичные компоненты имеют юнит-тесты (coverage >70%).
- Производительность: таблицы с 1000+ записями загружаются за <2 сек.
- Доступность: соответствие WCAG 2.1 AA (alt теги, keyboard navigation).
- Responsive design: корректное отображение на экранах >1280px (desktop only для MVP).

---

*Этот документ является техническим планом для frontend разработчика и должен синхронизироваться с `backend_plan_dev1.md` через `api_contracts.md`.*