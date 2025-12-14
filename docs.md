# AICO Platform — Технический план MVP

## Executive Summary

**Цель проекта**: Создать MVP платформы AI-компаньонов для образования, которая трансформирует учебный контент в персонализированные траектории развития навыков.

**Оценка сроков MVP**: 4-5 месяцев (16-20 недель)

**Команда**: 1-2 Frontend, 2 Backend, 2 ML/AI, 1 Дизайнер, 1 Продакт

---

## 1. Архитектура системы

### 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND LAYER                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Web App    │  │  Mobile     │  │  AI Chat    │  │  Graph Visualizer   │ │
│  │  (React)    │  │  (PWA)      │  │  Widget     │  │  (D3.js/Cytoscape)  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY                                     │
│              (Kong / AWS API Gateway / nginx + rate limiting)                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        ▼                             ▼                             ▼
┌───────────────┐          ┌─────────────────┐          ┌───────────────────┐
│  Auth Service │          │  Core API       │          │  AI Service       │
│  (Keycloak/   │          │  (FastAPI)      │          │  (FastAPI)        │
│   Supertokens)│          │                 │          │                   │
└───────────────┘          │  - Courses      │          │  - LLM Gateway    │
                           │  - Users        │          │  - RAG Pipeline   │
                           │  - Analytics    │          │  - Personalization│
                           │  - Groups       │          │  - Graph Gen      │
                           └─────────────────┘          └───────────────────┘
                                      │                             │
        ┌─────────────────────────────┼─────────────────────────────┤
        ▼                             ▼                             ▼
┌───────────────┐          ┌─────────────────┐          ┌───────────────────┐
│  PostgreSQL   │          │  Redis          │          │  Vector DB        │
│  (Main DB)    │          │  (Cache/Queue)  │          │  (Qdrant/Pinecone)│
└───────────────┘          └─────────────────┘          └───────────────────┘
                                                                    │
                                                                    ▼
                                                        ┌───────────────────┐
                                                        │  Object Storage   │
                                                        │  (S3/MinIO)       │
                                                        │  Files, Media     │
                                                        └───────────────────┘
```

### 1.2 Доменная модель (Core Entities)

```
Course (Курс)
├── id, name, description, status (draft/published)
├── scenario_type: lecture_free | case_trainer | ai_assistant
├── author_id, created_at, updated_at
├── competencies: [Competency]
├── learning_outcomes: [LearningOutcome]
└── knowledge_graph: KnowledgeGraph

KnowledgeGraph (Граф знаний)
├── id, course_id, version
├── concepts: [Concept]
└── edges: [ConceptEdge]

Concept (Концепт/Тема)
├── id, graph_id, name, description, order
├── sections: [Section]
├── case_diagnostics: [CaseDiagnostic]
└── prerequisites: [Concept]

Section (Раздел/Подтема)
├── id, concept_id, name, order
├── content_essence (базовый контент для персонализации)
├── additional_content: [ContentBlock]  # видео, код, таблицы
├── diagnostics: [Diagnostic]
└── source_chunks: [SourceChunk]  # связь с исходными файлами

User
├── id, email, role: author | student
├── profile: UserProfile
└── preferences: UserPreferences (стиль Колба, интересы, сложность)

StudentProgress
├── user_id, course_id
├── concept_statuses: {concept_id: status}
├── diagnostic_results: [DiagnosticResult]
├── case_results: [CaseResult]
└── companion_interactions: [Interaction]
```

### 1.3 Рекомендуемый стек технологий

| Layer | Технология | Обоснование |
|-------|------------|-------------|
| **Frontend** | React + TypeScript + Vite | Быстрая разработка, типизация, HMR |
| **State Management** | Zustand / TanStack Query | Простота, отличный кэш для API |
| **UI Framework** | Tailwind + shadcn/ui | Быстрая кастомизация, accessibility |
| **Graph Viz** | Cytoscape.js / React Flow | Интерактивные графы знаний |
| **Rich Text Editor** | TipTap / Lexical | Markdown + кастомные блоки |
| **Backend API** | FastAPI (Python) | Async, автодокументация, ML-экосистема |
| **Auth** | Supertokens / Keycloak | OAuth2, RBAC, self-hosted |
| **Main DB** | PostgreSQL | JSONB, надёжность, масштабируемость |
| **Vector DB** | Qdrant (self-hosted) / Pinecone | Для RAG, семантический поиск |
| **Cache/Queue** | Redis + Celery/ARQ | Кэш, фоновые задачи (генерация) |
| **Object Storage** | MinIO / S3 | Файлы курсов, медиа |
| **LLM** | Claude API / OpenAI | Генерация, RAG, диалог |
| **Embeddings** | text-embedding-3-small / e5-large | Векторизация контента |
| **TTS** | ElevenLabs / Azure TTS | Аудиолектор |
| **Infrastructure** | Docker + Kubernetes / Render | Масштабирование, CI/CD |

---

## 2. Ключевые технические модули

### 2.1 RAG Pipeline (критичный модуль)

```
Ingestion Flow:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Upload  │───▶│  Parse   │───▶│  Chunk   │───▶│  Embed   │
│  Files   │    │  (docx,  │    │  (smart  │    │  (text-  │
│          │    │   pdf)   │    │  split)  │    │  embed)  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                     │
                                                     ▼
                                              ┌──────────┐
                                              │  Vector  │
                                              │    DB    │
                                              └──────────┘

Query Flow:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  User    │───▶│  Embed   │───▶│  Search  │───▶│  Rerank  │
│  Query   │    │  Query   │    │  Top-K   │    │  Filter  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                     │
                                                     ▼
                                              ┌──────────┐
                                              │  LLM +   │
                                              │  Context │
                                              └──────────┘
```

**Важные решения:**
- Chunk size: 512-1024 tokens с overlap 20%
- Metadata: source_file, page, concept_id, section_id
- Hybrid search: vector + BM25 для лучшего recall
- Обязательная ссылка на источник в ответах

### 2.2 Knowledge Graph Generation

```python
# Алгоритм генерации графа (упрощённо)
async def generate_knowledge_graph(course_id: str, files: list[File]) -> KnowledgeGraph:
    # 1. Парсинг и чанкинг
    chunks = await parse_and_chunk(files)
    
    # 2. Векторизация
    embeddings = await embed_chunks(chunks)
    await vector_db.upsert(course_id, chunks, embeddings)
    
    # 3. Извлечение концептов (LLM)
    concepts = await extract_concepts(chunks)  # промпт с методикой ВШЭ
    
    # 4. Сопоставление с РПД (если есть)
    if syllabus:
        concepts = await align_with_syllabus(concepts, syllabus)
    
    # 5. Построение связей
    edges = await build_concept_edges(concepts, embeddings)
    
    # 6. Генерация контент-эссенций
    for concept in concepts:
        concept.sections = await generate_sections(concept, chunks)
    
    return KnowledgeGraph(concepts=concepts, edges=edges)
```

### 2.3 Personalization Engine

```
Input Parameters:
├── kolb_style: accommodator | diverger | assimilator | converger
├── interests: [str]  # минимум 3
├── complexity_level: basic | intermediate | advanced
└── companion_style: informal | academic | custom_prompt

Personalization Points:
├── Text generation: стиль изложения по Колбу
├── Examples: генерация примеров по интересам
├── Complexity: адаптация сложности терминологии
└── Companion: тон и стиль диалога
```

### 2.4 Companion Dialog System

```
System Prompt Structure:
├── Base persona (роль тьютора/тренера)
├── Course context (название, цели)
├── Current page context (раздел, контент)
├── User profile (стиль, интересы, прогресс)
├── Conversation history (последние 20 сообщений)
└── RAG context (релевантные чанки из курса)

Special Modes:
├── Explain simpler: упрощение выделенного текста
├── Give example: пример с учётом интересов
├── Case dialog: структурированный кейс-опрос
└── Ask teacher: эскалация преподавателю
```

---

## 3. План разработки MVP

### 3.1 Фазы и спринты

```
Phase 0: Foundation (Недели 1-2)
├── Инфраструктура, CI/CD, dev environment
├── Базовая архитектура, API skeleton
└── Design system, UI kit

Phase 1: Author Core (Недели 3-6)
├── Sprint 1: Auth + Course CRUD
├── Sprint 2: File upload + Graph generation
└── Sprint 3: Graph editing + Content management

Phase 2: Diagnostics (Недели 7-9)
├── Sprint 4: Diagnostic creation (тесты)
├── Sprint 5: Case diagnostics + LLM evaluation

Phase 3: Student Experience (Недели 10-14)
├── Sprint 6: Onboarding + Personalization
├── Sprint 7: Course navigation + Longread mode
├── Sprint 8: Audio lecturer + TTS
├── Sprint 9: Companion dialog (RAG-based)

Phase 4: Analytics & Polish (Недели 15-18)
├── Sprint 10: Progress tracking + Analytics
├── Sprint 11: Teacher dashboard + Groups
├── Sprint 12: Polish, bugs, performance

Buffer: 2 weeks
```

### 3.2 Детальный план по спринтам

#### Phase 0: Foundation (2 недели)

| Задача | Ответственные | Deliverables |
|--------|---------------|--------------|
| Настройка репозитория, CI/CD | Backend | GitHub Actions, Docker compose |
| Dev environment (docker-compose) | Backend | Локальный стек: DB, Redis, MinIO |
| API skeleton (FastAPI) | Backend | OpenAPI spec, базовые endpoints |
| Frontend scaffolding | Frontend | Vite + React + Router + Auth flow |
| Design system | Дизайнер | Figma kit, Tailwind tokens |
| Архитектура ML pipeline | ML/AI | Документация RAG, схема промптов |

#### Phase 1: Author Core (4 недели)

**Sprint 1 (2 недели): Auth + Course CRUD**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Auth service (Supertokens) | 5 | Backend 1 |
| User management API | 3 | Backend 1 |
| Course CRUD API | 5 | Backend 2 |
| Competencies/Outcomes API | 3 | Backend 2 |
| Auth UI (login, register, onboarding) | 5 | Frontend 1 |
| Course list + creation UI | 5 | Frontend 1 |
| Дизайн: Author dashboard | - | Дизайнер |

**Sprint 2 (2 недели): File Upload + Graph Generation**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| File upload service (S3/MinIO) | 5 | Backend 1 |
| Document parsing (docx, pdf, txt) | 5 | ML/AI 1 |
| Chunking + Embedding pipeline | 8 | ML/AI 1 |
| Vector DB setup (Qdrant) | 3 | ML/AI 2 |
| Graph generation (LLM prompts) | 8 | ML/AI 2 |
| Upload UI + progress indicator | 5 | Frontend 1 |
| Дизайн: Graph editor | - | Дизайнер |

**Sprint 3 (2 недели): Graph Editing + Content**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Graph CRUD API | 5 | Backend 1 |
| Section content API | 5 | Backend 2 |
| Content generation (sections) | 8 | ML/AI 1 |
| Graph visualization (Cytoscape) | 8 | Frontend 1 |
| Graph editing UI | 8 | Frontend 2 |
| Section editor (TipTap) | 5 | Frontend 2 |

#### Phase 2: Diagnostics (3 недели)

**Sprint 4 (1.5 недели): Test Diagnostics**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Diagnostic types API | 5 | Backend 1 |
| Question generation (LLM) | 5 | ML/AI 1 |
| Diagnostic creation UI | 8 | Frontend 1 |
| Дизайн: Student diagnostics | - | Дизайнер |

**Sprint 5 (1.5 недели): Case Diagnostics**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Case generation API | 5 | ML/AI 2 |
| Case dialog evaluation | 8 | ML/AI 2 |
| Case creation UI | 5 | Frontend 2 |
| Result storage + scoring | 5 | Backend 2 |

#### Phase 3: Student Experience (5 недель)

**Sprint 6 (1 неделя): Onboarding + Personalization**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Kolb test API | 3 | Backend 1 |
| Personalization service | 5 | ML/AI 1 |
| Student onboarding UI | 5 | Frontend 1 |
| Preferences storage | 3 | Backend 2 |

**Sprint 7 (1.5 недели): Course Navigation + Longread**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Personalized content generation | 8 | ML/AI 1 |
| Student course API | 5 | Backend 1 |
| Course map UI (graph view) | 5 | Frontend 1 |
| Longread view | 8 | Frontend 2 |
| Inline actions (explain, example) | 5 | Frontend 2 + ML |

**Sprint 8 (1.5 недели): Audio Lecturer**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| TTS integration | 5 | ML/AI 2 |
| Slide generation | 5 | ML/AI 2 |
| Audio player UI | 5 | Frontend 1 |
| Slide viewer | 5 | Frontend 1 |

**Sprint 9 (1 неделя): Companion Dialog**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| RAG query API | 5 | ML/AI 1 |
| Dialog context management | 5 | Backend 1 |
| Chat UI (persistent sidebar) | 8 | Frontend 1 |
| Avatar emotions | 3 | Frontend 2 |

#### Phase 4: Analytics & Polish (4 недели)

**Sprint 10 (1.5 недели): Progress Tracking**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Progress tracking API | 5 | Backend 1 |
| Competency profile | 5 | Backend 2 |
| Student portfolio UI | 5 | Frontend 1 |

**Sprint 11 (1.5 недели): Teacher Dashboard**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Groups management API | 5 | Backend 1 |
| Analytics aggregation | 8 | Backend 2 |
| Status matrix UI | 8 | Frontend 1 |
| Student detail view | 5 | Frontend 2 |
| Teacher-student chat | 5 | Frontend 2 + Backend |

**Sprint 12 (1 неделя): Polish**

| Задача | Story Points | Команда |
|--------|--------------|---------|
| Bug fixes | - | All |
| Performance optimization | - | Backend + ML |
| UX improvements | - | Frontend + Design |
| Documentation | - | All |

---

## 4. Распределение ролей

### Backend разработчики (2 человека)

**Backend 1 (Senior)**
- Архитектура, API design
- Auth service
- Core API (courses, users, progress)
- File upload, storage
- WebSocket для чата

**Backend 2 (Middle+)**
- Analytics, groups
- Diagnostic results
- Integration endpoints
- Background jobs (Celery)

### ML/AI разработчики (2 человека)

**ML/AI 1 (Senior)**
- RAG pipeline architecture
- Content personalization
- Graph generation prompts
- Dialog context management

**ML/AI 2 (Middle+)**
- Document parsing
- Diagnostic/case generation
- TTS integration
- Answer evaluation

### Frontend разработчики (1-2 человека)

**Frontend 1 (Senior)**
- Архитектура, state management
- Auth flow, onboarding
- Course navigation, graph viz
- Chat interface

**Frontend 2 (Middle)** — если есть
- Content editors (TipTap)
- Diagnostic UI
- Audio player
- Analytics dashboards

### Дизайнер (1 человек)

- Design system (Figma)
- Author flows
- Student flows
- Avatar и эмоции компаньона
- Responsive/mobile

### Продакт (1 человек)

- Приоритизация фич
- User stories, acceptance criteria
- Тестирование сценариев
- Stakeholder communication
- Документация для пользователей

---

## 5. Риски и митигация

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| Качество генерации графа | Высокая | Высокое | Итеративное улучшение промптов, ручная корректировка |
| Стоимость LLM API | Средняя | Среднее | Кэширование, локальные модели для embedding |
| Сложность RAG | Средняя | Высокое | Начать с простого, итерировать |
| Персонализация не работает | Средняя | Среднее | A/B тесты, сбор обратной связи |
| Перегруз команды | Средняя | Высокое | Четкий scope MVP, отсечение nice-to-have |
| Интеграция TTS | Низкая | Низкое | Fallback на текст |

---

## 6. Что НЕ входит в MVP

❌ Интеграция с LMS (Moodle)
❌ Mobile native apps (только PWA)
❌ Сложная геймификация
❌ Видеозвонки с преподавателем
❌ Многоязычность
❌ Офлайн-режим
❌ Продвинутая аналитика (ML-based insights)
❌ Marketplace курсов

---

## 7. Метрики успеха MVP

**Технические:**
- Time to generate graph < 3 min
- RAG response latency < 3 sec
- Uptime > 99%

**Продуктовые:**
- Author: создание курса < 2 часов
- Student: completion rate > 60%
- NPS > 30

---

## 8. Следующие шаги

1. **Неделя 0**: Kickoff, финализация стека, setup инфры
2. **Неделя 1**: Design sprint (ключевые экраны)
3. **Неделя 1-2**: API contracts, DB schema
4. **Неделя 2+**: Начало разработки Phase 1

---

*Документ подготовлен для команды AICO Platform*
*Версия: 1.0 | Дата: декабрь 2024*
