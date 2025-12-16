# AICO Platform — Design Doc (6 недель)

## Executive Summary

**Цель**: MVP платформы AI-компаньонов для образования за 6 недель.

**Ключевое ограничение**: Радикальное сужение scope. Делаем только то, что доказывает ценность продукта.

**Команда**: 1-2 Frontend, 2 Backend, 2 ML/AI, Дизайнер, Продакт

**AI-инструменты**: Cursor/Copilot, Claude, v0.dev, Lovable, Bolt.new

---

## 1. Философия 6-недельного MVP

### Что мы доказываем?

> "AI-компаньон может трансформировать учебный контент в персонализированный опыт обучения"

### Минимальный путь пользователя

**Автор:**
```
Загрузил файлы → Получил граф → Подправил → Добавил тесты → Опубликовал
```

**Студент:**
```
Прошёл мини-онбординг → Изучает персонализированный контент → Общается с компаньоном → Проходит тесты
```

### Что УБИРАЕМ из MVP

| Фича | Причина отказа | Когда вернём |
|------|----------------|--------------|
| Аудиолектор (TTS) | Сложная интеграция, не core value | v1.1 |
| Кейсовые диагностики | Сложная LLM-оценка | v1.1 |
| Тест Колба (30 вопросов) | Упрощаем до 5 вопросов | v1.1 |
| Группы преподавателя | Минимальная аналитика без групп | v1.1 |
| Генерация обложек | Косметика | v1.2 |
| Эмоции аватара | Косметика | v1.2 |
| Справочник компетенций ФГОС | Свободный ввод пока | v1.1 |
| Вопросы преподавателю | Сложный flow | v1.1 |
| История чата (папка) | Минимум | v1.1 |

### Что ОСТАВЛЯЕМ (Core MVP)

| Фича | Почему критична |
|------|-----------------|
| Загрузка файлов + парсинг | Вход данных |
| Генерация графа знаний | Core AI value |
| Редактирование графа | Контроль качества |
| Персонализация контента | Core AI value |
| Longread режим | Основной формат |
| Inline actions (объяснить, пример) | Интерактивность |
| Чат с компаньоном (RAG) | Core AI value |
| Тестовые диагностики | Оценка прогресса |
| Базовый прогресс студента | Метрики |

---

## 2. Архитектура (упрощённая)

### 2.1 Минимальная архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND                                 │
│                    React + TypeScript                            │
│            (сгенерировано через v0/Lovable + доработка)          │
└────────────────────────────┬────────────────────────────────────┘
                             │ REST + WebSocket
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MONOLITH API                                │
│                        FastAPI                                   │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────────┐ │
│  │  Auth   │  │ Courses │  │   AI    │  │    File Upload      │ │
│  │ (simple)│  │  CRUD   │  │ (LLM+RAG)│  │    (S3/MinIO)       │ │
│  └─────────┘  └─────────┘  └─────────┘  └─────────────────────┘ │
└───────┬─────────────┬─────────────┬─────────────┬───────────────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
   PostgreSQL      Redis        Qdrant      MinIO/S3
   (main DB)      (cache)     (vectors)     (files)
```

**Ключевое решение: Монолит вместо микросервисов**

Почему:
- Быстрее разрабатывать
- Проще деплоить
- Меньше инфраструктуры
- Достаточно для MVP нагрузки

### 2.2 Упрощённая модель данных

```python
# Минимальные сущности

User:
  id, email, password_hash, role (author|student)
  preferences: JSON  # стиль, интересы, сложность — всё в одном поле

Course:
  id, author_id, name, description, status (draft|published)
  scenario_type: str  # lecture_free | case_trainer
  
KnowledgeGraph:
  id, course_id
  data: JSON  # весь граф в JSON (concepts, sections, edges)
  
SourceFile:
  id, course_id, filename, s3_key, status (processing|ready|error)

ContentChunk:
  id, course_id, content, embedding (vector), metadata: JSON

Diagnostic:
  id, section_id, type, question, options: JSON, correct_answer

StudentProgress:
  id, user_id, course_id
  completed_sections: JSON  # список ID
  diagnostic_results: JSON  # {diagnostic_id: score}
  
ChatHistory:
  id, user_id, course_id
  messages: JSON  # последние 20
```

### 2.3 Стек технологий (финальный)

| Компонент | Технология | Комментарий |
|-----------|------------|-------------|
| Frontend | React + Vite + TypeScript | shadcn/ui для UI |
| UI Generation | v0.dev / Lovable | Генерация компонентов |
| State | Zustand + TanStack Query | Простота |
| Graph Viz | React Flow | Проще Cytoscape |
| Rich Editor | TipTap | Для редактирования контента |
| Backend | FastAPI (монолит) | Один сервис |
| Auth | FastAPI + JWT | Без внешних сервисов |
| DB | PostgreSQL + pgvector | Можно без Qdrant |
| Vector DB | pgvector ИЛИ Qdrant | pgvector проще |
| Cache | Redis | Кэш + очереди |
| Queue | ARQ (async Redis queue) | Проще Celery |
| Storage | MinIO (dev) / S3 (prod) | |
| LLM | Claude API | Для всего AI |
| Embeddings | text-embedding-3-small | OpenAI, дёшево |
| Deploy | Railway / Render | Простой деплой |

---

## 3. AI-инструменты в разработке

### 3.1 Стратегия использования AI

| Инструмент | Применение | Ожидаемый буст |
|------------|------------|----------------|
| **Cursor / Copilot** | Написание кода, автокомплит | 30-50% ускорение |
| **Claude (chat)** | Архитектурные решения, промпты, дебаг | 20-30% ускорение |
| **v0.dev** | Генерация UI компонентов | 50-70% ускорение UI |
| **Lovable / Bolt** | Прототипы целых страниц | Быстрые прототипы |
| **Claude Artifacts** | Генерация схем, документации | |

### 3.2 Конкретные применения

**Frontend (v0.dev + Cursor):**
```
1. Сгенерировать в v0: "Dashboard with sidebar, course cards grid, 
   create course modal with form fields for name, description, 
   file upload dropzone"
   
2. Экспортировать код → доработать в Cursor

3. Cursor: "Add TanStack Query hooks for fetching courses from 
   /api/courses endpoint"
```

**Backend (Cursor + Claude):**
```
1. Claude: сгенерировать OpenAPI spec для всех endpoints
2. Cursor: "Generate FastAPI routes from this OpenAPI spec"
3. Cursor: "Add SQLAlchemy models for these entities"
```

**ML/AI (Claude):**
```
1. Claude: написать и отладить промпты для:
   - Извлечения концептов из текста
   - Генерации персонализированного контента
   - RAG-ответов компаньона
   
2. Cursor: обернуть в Python код с retry, caching
```

### 3.3 Что НЕ делегировать AI

- Архитектурные решения (review человеком)
- Промпты для LLM (требуют итераций и понимания)
- Сложная бизнес-логика (персонализация)
- Security (auth, validation)
- Тестирование edge cases

---

## 4. План разработки (6 недель)

### Обзор фаз

```
Неделя 1: Foundation + Auth + File Upload
Неделя 2: Graph Generation + Editing
Неделя 3: Student Onboarding + Content View
Неделя 4: Companion Chat (RAG) + Inline Actions
Неделя 5: Diagnostics + Progress
Неделя 6: Integration + Polish + Deploy
```

### Детальный план

#### Неделя 1: Foundation

| День | Backend | Frontend | ML/AI | Design |
|------|---------|----------|-------|--------|
| 1-2 | Docker compose, DB schema, базовый FastAPI | Vite setup, роутинг, shadcn/ui | Qdrant/pgvector setup | Дизайн auth + dashboard |
| 3-4 | Auth endpoints (register, login, JWT) | Auth pages (v0 → доработка) | Document parsing (docx, pdf) | Дизайн course creation |
| 5 | File upload endpoint + S3 | Upload UI + progress | Chunking pipeline | |

**Deliverables недели 1:**
- [ ] Работающий auth flow
- [ ] Загрузка файлов в S3
- [ ] Парсинг документов в чанки
- [ ] Базовый dashboard автора

#### Неделя 2: Graph Generation

| День | Backend | Frontend | ML/AI | Design |
|------|---------|----------|-------|--------|
| 1-2 | Course CRUD API, Graph storage | Course creation form | Embedding pipeline | Дизайн graph editor |
| 3-4 | Background job для генерации | Progress indicator | Graph generation prompts | Дизайн content view |
| 5 | Graph CRUD API | React Flow интеграция | Тестирование генерации | |

**Deliverables недели 2:**
- [ ] Создание курса с метаданными
- [ ] Автогенерация графа из файлов
- [ ] Визуализация графа
- [ ] Редактирование структуры

#### Неделя 3: Student Experience (базовый)

| День | Backend | Frontend | ML/AI | Design |
|------|---------|----------|-------|--------|
| 1-2 | Student preferences API | Мини-онбординг (5 вопросов) | Персонализация промптов | Дизайн longread |
| 3-4 | Section content API | Longread view | Content personalization | Дизайн chat |
| 5 | Course enrollment | Course navigation + map | | |

**Deliverables недели 3:**
- [ ] Онбординг студента (стиль, интересы, сложность)
- [ ] Навигация по курсу (граф + список)
- [ ] Отображение персонализированного контента

#### Неделя 4: Companion Chat

| День | Backend | Frontend | ML/AI | Design |
|------|---------|----------|-------|--------|
| 1-2 | WebSocket endpoint | Chat UI sidebar | RAG pipeline | |
| 3-4 | Chat history storage | Inline actions UI | Context management | Дизайн diagnostics |
| 5 | | Интеграция inline → chat | Промпты для explain/example | |

**Deliverables недели 4:**
- [ ] Работающий чат с компаньоном
- [ ] RAG по материалам курса
- [ ] "Объяснить проще" / "Привести пример"
- [ ] Контекст текущей страницы в чате

#### Неделя 5: Diagnostics + Progress

| День | Backend | Frontend | ML/AI | Design |
|------|---------|----------|-------|--------|
| 1-2 | Diagnostic CRUD API | Diagnostic creation UI | Question generation | |
| 3-4 | Progress tracking API | Diagnostic passing UI | | Дизайн analytics |
| 5 | Basic analytics endpoint | Progress display | | |

**Deliverables недели 5:**
- [ ] Создание тестовых диагностик
- [ ] Прохождение диагностик студентом
- [ ] Трекинг прогресса
- [ ] Базовая аналитика для автора

#### Неделя 6: Polish + Deploy

| День | Backend | Frontend | ML/AI | Design |
|------|---------|----------|-------|--------|
| 1-2 | Bug fixes, API hardening | Bug fixes, UX polish | Prompt tuning | |
| 3-4 | Deploy setup (Railway/Render) | Performance, loading states | | |
| 5 | Monitoring, logging | Final testing | | |

**Deliverables недели 6:**
- [ ] Продакшен деплой
- [ ] Исправлены критичные баги
- [ ] Базовый мониторинг
- [ ] Готово к демо

---

## 5. Распределение по ролям (6 недель)

### Backend 1 (Senior)
- Архитектура монолита
- Auth + JWT
- Core API (courses, users)
- WebSocket для чата
- Deploy инфраструктура

### Backend 2 
- File upload + S3
- Background jobs (ARQ)
- Diagnostics API
- Progress tracking
- Analytics

### ML/AI 1 (Senior)
- RAG pipeline architecture
- Graph generation prompts
- Content personalization
- Chat context management

### ML/AI 2
- Document parsing
- Embedding pipeline
- Diagnostic generation
- Prompt engineering assist

### Frontend 1 (Senior)
- Архитектура фронта
- Auth flow
- Course navigation
- Chat interface
- Graph visualization

### Frontend 2 (если есть)
- Content editor
- Diagnostic UI
- Онбординг
- Вспомогательные экраны

### Дизайнер
- Неделя 1: Auth, Dashboard, Course creation
- Неделя 2: Graph editor, Content view
- Неделя 3: Longread, Chat
- Неделя 4-5: Diagnostics, Analytics
- Неделя 6: Polish

### Продакт
- Daily standups
- Приоритизация при срезании scope
- Acceptance testing
- Demo подготовка

---

## 6. Риски и митигация

### 6.1 Критичные риски (Red)

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| **Качество генерации графа неприемлемо** | Высокая | Критичное | Ручное редактирование как fallback. Итерации промптов с первого дня. Готовность показать "полу-автоматический" режим. |
| **RAG даёт нерелевантные ответы** | Высокая | Критичное | Начать с простого retrieval. Добавить reranking. Fallback: "Я не нашёл информацию в курсе". |
| **Не успеваем за 6 недель** | Высокая | Критичное | Scope cutting plan готов заранее (см. раздел 6.4). Фокус на demo-path. |
| **LLM API недоступен / дорого** | Средняя | Критичное | Rate limiting. Кэширование ответов. Бюджет на API заложен. |

### 6.2 Серьёзные риски (Yellow)

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| **Персонализация не чувствуется** | Средняя | Высокое | Явные маркеры ("Пример для тебя как любителя X"). A/B между персонализированным и базовым. |
| **Парсинг документов ломается** | Средняя | Высокое | Ограничить форматы (только docx, pdf). Ручная проверка после загрузки. |
| **Frontend не успевает** | Средняя | Высокое | v0/Lovable для генерации UI. Минимальный polish, функциональность важнее. |
| **Сложная интеграция компонентов** | Средняя | Среднее | Интеграционные тесты с недели 3. Ежедневные деплои на staging. |
| **Команда выгорает** | Средняя | Высокое | Реалистичные ожидания. Scope cutting без вины. Celebration milestones. |

### 6.3 Умеренные риски (Green)

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| **Проблемы с деплоем** | Низкая | Среднее | Railway/Render — простой деплой. Docker с первого дня. |
| **Производительность** | Низкая | Среднее | Для MVP достаточно. Кэширование Redis. |
| **Security issues** | Низкая | Высокое | Базовые практики: JWT, input validation, CORS. Не MVP для банка. |

### 6.4 План срезания scope (если не успеваем)

**Уровень 1 (срезаем первым):**
- Редактирование связей в графе (только структура)
- Inline action "задать вопрос" (оставить только explain/example)
- Аналитика автора (только список студентов)
- Выбор сценария курса (только Lecture-Free)

**Уровень 2 (срезаем если совсем плохо):**
- Генерация диагностик (только ручное создание)
- Персонализация по интересам (только по сложности)
- История чата
- Прогресс на графе (только список)

**Уровень 3 (минимальное демо):**
- Загрузка → Граф → Просмотр контента → Чат
- Без диагностик
- Без онбординга (захардкоженные настройки)

### 6.5 Риски использования AI-инструментов

| Риск | Митигация |
|------|-----------|
| **Сгенерированный код низкого качества** | Code review обязателен. Не мержить без проверки. |
| **Vendor lock-in на v0/Lovable** | Генерируем компоненты, не целые приложения. |
| **AI галлюцинирует архитектуру** | Архитектурные решения — только с senior review. |
| **Переоценка скорости от AI** | Закладывать 30% buffer, AI не серебряная пуля. |
| **Несовместимость сгенерированного кода** | Единый style guide. Линтеры с первого дня. |

---

## 7. Definition of Done для MVP

### Must Have (без этого не релизим)

- [ ] Автор может загрузить файлы и получить граф
- [ ] Автор может отредактировать структуру графа
- [ ] Автор может опубликовать курс
- [ ] Студент может пройти онбординг
- [ ] Студент видит персонализированный контент
- [ ] Студент может общаться с компаньоном по курсу
- [ ] Компаньон отвечает на основе материалов курса (RAG)
- [ ] Работает на продакшене (не localhost)

### Should Have (желательно)

- [ ] Диагностики (хотя бы ручное создание + прохождение)
- [ ] Inline actions (объяснить проще, пример)
- [ ] Прогресс студента виден автору
- [ ] Редактирование контента разделов

### Could Have (бонус)

- [ ] Генерация диагностик
- [ ] История чата
- [ ] Аналитика

---

## 8. Технические решения для скорости

### 8.1 Shortcuts (осознанный tech debt)

| Решение | Нормально было бы | Почему ОК для MVP |
|---------|-------------------|-------------------|
| Граф в JSON поле | Отдельные таблицы с связями | Проще работать, миграция позже |
| JWT без refresh | Access + Refresh tokens | 7 дней expiry достаточно |
| pgvector вместо Qdrant | Dedicated vector DB | Меньше инфраструктуры |
| Монолит | Микросервисы | Быстрее разрабатывать |
| ARQ вместо Celery | Celery + Redis/RabbitMQ | Проще настроить |
| Railway/Render | Kubernetes | Zero DevOps overhead |

### 8.2 Что НЕ упрощаем

- **Auth security** — JWT правильно, validation, CORS
- **Input validation** — Pydantic везде
- **Error handling** — Graceful degradation
- **Logging** — Structured logs с первого дня
- **RAG quality** — Это core value, нельзя резать

---

## 9. Ежедневный ритм

```
09:00 - Standup (15 min)
        - Что сделал вчера
        - Что делаю сегодня
        - Блокеры

12:00 - Sync ML + Backend (если нужно)

17:00 - Деплой на staging (ежедневно!)

18:00 - Quick demo того, что готово (по пятницам)
```

**Правила:**
- Блокеры эскалируются немедленно
- Scope cutting решается в тот же день
- Мержим только работающий код
- Staging всегда должен работать

---

## 10. Метрики успеха

**Технические (к концу недели 6):**
- Graph generation: < 5 min для 100 страниц
- RAG response: < 5 sec
- Uptime: > 95%
- Критичных багов: 0

**Продуктовые:**
- Автор создаёт курс за < 1 час
- Студент понимает как пользоваться за < 5 min
- Компаньон даёт релевантные ответы в > 70% случаев

---

## Приложение A: Промпты для генерации (стартовые)

### Graph Generation Prompt

```
Ты — эксперт по структурированию образовательного контента.

На входе: текст учебных материалов.

Задача: выдели основные концепты (темы) и разделы (подтемы).

Формат ответа (JSON):
{
  "concepts": [
    {
      "name": "Название концепта",
      "description": "Краткое описание",
      "sections": [
        {"name": "Раздел 1", "key_points": ["тезис 1", "тезис 2"]},
        ...
      ]
    }
  ],
  "edges": [
    {"from": "Концепт A", "to": "Концепт B", "relation": "prerequisite"}
  ]
}

Правила:
- 5-15 концептов на курс
- 2-5 разделов на концепт
- Связи только если есть явная зависимость
```

### Personalization Prompt

```
Адаптируй текст под студента:
- Стиль обучения: {kolb_style}
- Интересы: {interests}
- Уровень сложности: {complexity}

Исходный текст:
{content}

Правила адаптации:
- Для Accommodator: больше практических примеров
- Для Diverger: разные точки зрения
- Для Assimilator: теория и модели
- Для Converger: применение и решения

Примеры должны использовать интересы студента.
```

### RAG Companion Prompt

```
Ты — AI-компаньон курса "{course_name}".

Контекст страницы:
{page_content}

Релевантные материалы курса:
{rag_context}

Вопрос студента: {question}

Правила:
- Отвечай ТОЛЬКО на основе материалов курса
- Если информации нет — честно скажи
- Ссылайся на конкретные разделы
- Стиль общения: {companion_style}
```

---

*Design Doc v2.0 — 6-недельный MVP*
*Команда AICO | Декабрь 2024*
