# NodeRAG REST API Architecture

## Введение

Данный документ описывает **архитектуру REST API** для NodeRAG, спроектированную для **максимального открытия внутренних структур и операций** системы внешним пользователям.

**Философия дизайна**: Transparency & Inspectability
- Открытый доступ к графовым и векторным представлениям
- Видимость траекторий семантического поиска
- Прозрачность reasoning chain системы
- Исследование концептуальных сетей и связей

---

## Архитектурные принципы

### 1. Layered Architecture

```
┌─────────────────────────────────────────────┐
│         REST API Layer                      │
│  (FastAPI / Flask endpoints)                │
└─────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────┐
│      Instrumentation Layer                  │
│  (Search trajectory tracking,               │
│   reasoning chain capture)                  │
└─────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────┐
│        Core NodeRAG Components              │
│  (NodeSearch, HNSW, PPR, Graph, LLM)       │
└─────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────┐
│          Storage Layer                      │
│  (Graph DB, Vector Index, Embeddings)       │
└─────────────────────────────────────────────┘
```

### 2. API Categories

API endpoints организованы в **6 логических категорий**:

| Категория | Назначение | Примеры endpoints |
|-----------|------------|-------------------|
| **Query API** | Основной поиск и QA | `/query`, `/search` |
| **Graph API** | Доступ к графу знаний | `/graph/nodes`, `/graph/neighbors` |
| **Vector API** | Векторное пространство | `/vectors/search`, `/vectors/embedding` |
| **Trajectory API** | Траектории поиска | `/trajectory/{query_id}` |
| **Reasoning API** | Цепочка рассуждений | `/reasoning/{query_id}` |
| **Introspection API** | Концепты и аттракторы | `/attractors`, `/concepts` |

### 3. Response Format

Все API возвращают **structured JSON** с метаданными:

```json
{
  "status": "success",
  "data": { ... },
  "metadata": {
    "query_id": "uuid",
    "timestamp": "ISO-8601",
    "execution_time_ms": 123,
    "model_info": { ... }
  },
  "links": {
    "self": "/api/v1/...",
    "related": ["/api/v1/..."]
  }
}
```

---

## API Endpoint Structure

### Base URL

```
https://api.noderag.example.com/api/v1
```

### Authentication

**Bearer Token** (JWT):

```http
Authorization: Bearer <token>
```

### Rate Limiting

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

---

## Core API Groups

### 1. Query & Search API (`/query`)

**Основной интерфейс** для пользовательских запросов с полным контекстом.

**Key endpoints**:
- `POST /query` - Выполнение query с полным reasoning
- `POST /search` - Только retrieval без LLM answer
- `GET /query/{query_id}` - Получение сохранённого результата

**Особенность**: Возвращает **не только ответ**, но и:
- Траекторию поиска (HNSW → Decomposition → PPR)
- Промежуточные результаты на каждом шаге
- Reasoning chain LLM
- Retrieved context с типами узлов

### 2. Graph Access API (`/graph`)

**Прямой доступ** к графу знаний (NetworkX graph).

**Key endpoints**:
- `GET /graph/nodes` - Список узлов с фильтрацией
- `GET /graph/node/{node_id}` - Детали узла
- `GET /graph/edges` - Список рёбер
- `GET /graph/neighbors/{node_id}` - Соседи узла
- `GET /graph/subgraph` - Извлечение подграфа
- `GET /graph/paths` - Кратчайшие пути
- `GET /graph/community/{community_id}` - Сообщество (Leiden)

**Особенность**: Поддержка **graph query language** для сложных запросов.

### 3. Vector Space API (`/vectors`)

**Доступ к векторному представлению** (HNSW + embeddings).

**Key endpoints**:
- `POST /vectors/search` - Векторный поиск по embedding
- `POST /vectors/embed` - Получение embedding для текста
- `GET /vectors/node/{node_id}` - Embedding узла
- `GET /vectors/similar/{node_id}` - Похожие узлы (HNSW)
- `GET /vectors/hnsw/layers` - Структура HNSW layers
- `GET /vectors/hnsw/neighbors/{node_id}` - HNSW соседи

**Особенность**: Доступ к **multilayer HNSW structure**.

### 4. Search Trajectory API (`/trajectory`)

**Трассировка процесса поиска** с временными метками.

**Key endpoints**:
- `GET /trajectory/{query_id}` - Полная траектория
- `GET /trajectory/{query_id}/steps` - Пошаговое выполнение
- `GET /trajectory/{query_id}/timeline` - Временная линия
- `GET /trajectory/{query_id}/visualization` - Данные для визуализации

**Траектория включает**:
1. Query embedding generation
2. HNSW k-NN search
3. Query decomposition (LLM)
4. Accurate search (regex matching)
5. Attractor construction (analog + symbolic)
6. PPR execution (iterations)
7. Node ranking & filtering
8. Context assembly
9. LLM answer generation

**Формат**:
```json
{
  "query_id": "uuid",
  "steps": [
    {
      "step": 1,
      "name": "query_embedding",
      "timestamp": "...",
      "duration_ms": 150,
      "input": { "query": "..." },
      "output": { "embedding": [...], "dim": 1536 },
      "metadata": { "model": "text-embedding-3-small" }
    },
    {
      "step": 2,
      "name": "hnsw_search",
      "timestamp": "...",
      "duration_ms": 8,
      "input": { "embedding": [...], "k": 50 },
      "output": {
        "results": [
          { "node_id": "...", "distance": 0.234, "type": "semantic_unit" }
        ]
      },
      "metadata": { "ef": 200, "M": 64 }
    }
    // ... остальные шаги
  ]
}
```

### 5. Reasoning Chain API (`/reasoning`)

**Цепочка рассуждений** системы и LLM.

**Key endpoints**:
- `GET /reasoning/{query_id}` - Полная reasoning chain
- `GET /reasoning/{query_id}/llm-calls` - Все LLM вызовы
- `GET /reasoning/{query_id}/decisions` - Ключевые решения
- `GET /reasoning/{query_id}/context-evolution` - Эволюция контекста

**Reasoning chain включает**:
1. **Query decomposition reasoning**: Почему извлечены эти entities?
2. **Attractor selection reasoning**: Почему выбраны эти аттракторы?
3. **PPR personalization reasoning**: Веса аттракторов
4. **Node filtering reasoning**: Критерии отбора узлов
5. **Context assembly reasoning**: Релевантность узлов
6. **Answer synthesis reasoning**: LLM chain-of-thought

**Формат**:
```json
{
  "query_id": "uuid",
  "reasoning_chain": [
    {
      "stage": "query_decomposition",
      "decision": "Extract entities from query",
      "input": { "query": "What did Emily Roberts research?" },
      "reasoning": {
        "llm_prompt": "...",
        "llm_response": {
          "elements": ["EMILY ROBERTS", "RESEARCH"]
        },
        "rationale": "Extracted key entities for accurate graph search"
      },
      "output": {
        "entities": ["EMILY ROBERTS", "RESEARCH"]
      }
    },
    {
      "stage": "attractor_construction",
      "decision": "Combine analog and symbolic attractors",
      "reasoning": {
        "analog_attractors": {
          "source": "HNSW k-NN",
          "count": 50,
          "weight": 1.0,
          "rationale": "Semantic similarity in embedding space"
        },
        "symbolic_attractors": {
          "source": "Query decomposition",
          "count": 2,
          "weight": 2.0,
          "rationale": "Exact entity matches in graph"
        },
        "combination": "Dual-mode attractors for PPR personalization"
      },
      "output": {
        "personalization": { "node_123": 1.0, "node_456": 2.0, ... }
      }
    }
    // ... остальные стадии
  ]
}
```

### 6. Introspection API (`/introspection`)

**Исследование концептуальных структур** и связей.

**Key endpoints**:

#### Attractors
- `POST /introspection/attractors` - Получение аттракторов для query
- `GET /introspection/attractors/{query_id}` - Сохранённые аттракторы
- `POST /introspection/attractors/compare` - Сравнение аттракторов

#### Concepts & Networks
- `GET /introspection/concepts/{concept_id}` - Концептуальная сеть
- `GET /introspection/concepts/similar/{concept_id}` - Похожие концепты
- `GET /introspection/concepts/path/{from}/{to}` - Концептуальный путь
- `POST /introspection/concepts/expand` - Расширение концептуальной сети

#### Related Queries
- `POST /introspection/related-queries` - Связанные вопросы
- `POST /introspection/query-suggestions` - Предложения вопросов
- `POST /introspection/query-refinements` - Уточнения query

**Формат attractors**:
```json
{
  "query": "What did Emily Roberts research?",
  "attractors": {
    "analog": {
      "source": "HNSW",
      "method": "cosine_similarity",
      "nodes": [
        {
          "node_id": "sem_unit_123",
          "type": "semantic_unit",
          "distance": 0.234,
          "content": "Dr. Emily Roberts presented research on...",
          "weight": 1.0
        }
      ]
    },
    "symbolic": {
      "source": "query_decomposition",
      "method": "exact_match",
      "nodes": [
        {
          "node_id": "entity_456",
          "type": "entity",
          "content": "EMILY ROBERTS",
          "matched_term": "Emily Roberts",
          "weight": 2.0
        }
      ]
    },
    "combined": {
      "personalization_vector": {
        "sem_unit_123": 1.0,
        "entity_456": 2.0,
        ...
      },
      "total_weight": 150.0,
      "normalization": "sum"
    }
  }
}
```

**Формат concept network**:
```json
{
  "concept_id": "entity_456",
  "concept": {
    "id": "entity_456",
    "type": "entity",
    "content": "EMILY ROBERTS",
    "embedding": [...],
    "attributes": [
      {
        "id": "attr_789",
        "content": "Dr. Emily Roberts is a leading researcher..."
      }
    ]
  },
  "network": {
    "immediate_neighbors": [
      {
        "node_id": "sem_unit_123",
        "type": "semantic_unit",
        "relation": "mentioned_in",
        "weight": 1.0,
        "content": "..."
      }
    ],
    "communities": [
      {
        "community_id": "comm_42",
        "high_level_elements": [
          {
            "id": "hle_1",
            "title": "Renewable Energy Research",
            "description": "..."
          }
        ]
      }
    ],
    "related_concepts": [
      {
        "concept_id": "entity_999",
        "content": "RENEWABLE ENERGY",
        "relation": "research_topic",
        "similarity": 0.78,
        "path_length": 2
      }
    ]
  }
}
```

---

## Request/Response Patterns

### Pagination

```http
GET /graph/nodes?page=1&page_size=50
```

Response:
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "page_size": 50,
    "total_items": 5000,
    "total_pages": 100
  },
  "links": {
    "self": "/graph/nodes?page=1",
    "next": "/graph/nodes?page=2",
    "prev": null,
    "first": "/graph/nodes?page=1",
    "last": "/graph/nodes?page=100"
  }
}
```

### Filtering

```http
GET /graph/nodes?type=entity&weight_min=5&limit=100
```

### Sorting

```http
GET /graph/nodes?sort_by=weight&order=desc
```

### Field Selection

```http
GET /graph/node/123?fields=id,type,content,embedding
```

---

## Advanced Features

### 1. Real-time Streaming

**Server-Sent Events (SSE)** для streaming ответов:

```http
POST /query/stream
Content-Type: application/json

{
  "query": "What did Emily Roberts research?",
  "stream": true
}
```

Response:
```
data: {"type": "trajectory_step", "step": "query_embedding", ...}

data: {"type": "trajectory_step", "step": "hnsw_search", ...}

data: {"type": "trajectory_step", "step": "ppr_execution", ...}

data: {"type": "answer_chunk", "chunk": "Dr. Emily Roberts"}

data: {"type": "answer_chunk", "chunk": " has conducted"}

data: {"type": "complete", "full_response": {...}}
```

### 2. Batch Operations

```http
POST /query/batch
Content-Type: application/json

{
  "queries": [
    { "id": "q1", "query": "..." },
    { "id": "q2", "query": "..." }
  ],
  "options": {
    "parallel": true,
    "include_trajectory": false
  }
}
```

### 3. GraphQL Support

**GraphQL endpoint** для гибких запросов:

```graphql
query {
  node(id: "entity_456") {
    id
    type
    content
    embedding
    neighbors(type: "semantic_unit", limit: 10) {
      id
      content
      relation
    }
    community {
      id
      high_level_elements {
        title
        description
      }
    }
  }
}
```

### 4. WebSocket для Interactive Sessions

```javascript
ws://api.noderag.example.com/ws/session

// Client sends
{
  "action": "query",
  "query": "What did Emily Roberts research?"
}

// Server streams
{ "type": "trajectory", "step": "hnsw_search", ... }
{ "type": "trajectory", "step": "ppr_execution", ... }
{ "type": "answer", "chunk": "Dr. Emily Roberts..." }
{ "type": "complete" }
```

---

## Performance Considerations

### Caching Strategy

```
┌──────────────┐
│ Query Cache  │  (Redis) - Кэш query results (TTL: 1h)
└──────────────┘
       ↓
┌──────────────┐
│Embedding Cache│ (Redis) - Кэш embeddings (TTL: 24h)
└──────────────┘
       ↓
┌──────────────┐
│ Graph Cache  │  (In-memory) - Горячие узлы
└──────────────┘
```

### Async Processing

Длительные операции (PPR, LLM) выполняются **асинхронно**:

```http
POST /query
```

Response (202 Accepted):
```json
{
  "status": "processing",
  "query_id": "uuid",
  "estimated_time_ms": 2000,
  "status_url": "/query/uuid/status",
  "result_url": "/query/uuid"
}
```

Polling:
```http
GET /query/uuid/status
```

Response:
```json
{
  "status": "completed",
  "progress": 100,
  "result_url": "/query/uuid"
}
```

---

## Security & Privacy

### 1. Data Access Control

**Role-Based Access Control (RBAC)**:
- `user` - Query, basic introspection
- `developer` - Full trajectory, reasoning chain
- `admin` - Graph modifications, system config

### 2. Query Logging

All queries logged для audit:

```json
{
  "query_id": "uuid",
  "user_id": "user123",
  "timestamp": "...",
  "query": "...",
  "ip_address": "...",
  "execution_time_ms": 1234
}
```

### 3. Privacy Controls

- **Anonymization**: Опция удаления PII из results
- **Data Retention**: Configurable TTL для query results
- **Access Logs**: Кто и когда обращался к data

---

## Error Handling

### Standard Error Response

```json
{
  "status": "error",
  "error": {
    "code": "GRAPH_NODE_NOT_FOUND",
    "message": "Node with id 'xyz' not found in graph",
    "details": {
      "node_id": "xyz",
      "suggestion": "Check node ID or use /graph/nodes to list available nodes"
    }
  },
  "metadata": {
    "request_id": "uuid",
    "timestamp": "..."
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_QUERY` | 400 | Malformed query |
| `NODE_NOT_FOUND` | 404 | Graph node not found |
| `EMBEDDING_FAILED` | 500 | Embedding generation failed |
| `PPR_TIMEOUT` | 504 | PPR execution timeout |
| `LLM_ERROR` | 502 | LLM API error |

---

## Monitoring & Observability

### Health Check

```http
GET /health
```

Response:
```json
{
  "status": "healthy",
  "components": {
    "api": "up",
    "graph_db": "up",
    "hnsw_index": "up",
    "llm_api": "up",
    "embedding_api": "up"
  },
  "version": "1.0.0"
}
```

### Metrics Endpoint

```http
GET /metrics
```

Prometheus format:
```
# HELP noderag_queries_total Total number of queries
# TYPE noderag_queries_total counter
noderag_queries_total 12345

# HELP noderag_query_duration_seconds Query execution time
# TYPE noderag_query_duration_seconds histogram
noderag_query_duration_seconds_bucket{le="0.5"} 100
noderag_query_duration_seconds_bucket{le="1.0"} 250
...
```

---

## OpenAPI Specification

Полная **OpenAPI 3.0 спецификация** доступна:

```http
GET /openapi.json
```

**Swagger UI**:
```
https://api.noderag.example.com/docs
```

**ReDoc**:
```
https://api.noderag.example.com/redoc
```

---

## Implementation Roadmap

### Phase 1: Core APIs (MVP)
- ✅ Query API (`/query`)
- ✅ Basic Graph API (`/graph/node`, `/graph/neighbors`)
- ✅ Basic Vector API (`/vectors/search`)
- ✅ Траектория поиска (basic)

### Phase 2: Advanced Introspection
- ⏳ Full Trajectory API с visualizations
- ⏳ Reasoning Chain API
- ⏳ Attractors API
- ⏳ Concepts & Networks API

### Phase 3: Real-time & Advanced Features
- ⏳ Streaming (SSE)
- ⏳ WebSocket sessions
- ⏳ GraphQL support
- ⏳ Batch operations

### Phase 4: Enterprise Features
- ⏳ Advanced caching
- ⏳ Multi-tenancy
- ⏳ Advanced security
- ⏳ Monitoring & alerts

---

## Связанные документы

- [Graph API Reference](./graph-api.md)
- [Vector API Reference](./vector-api.md)
- [Search Trajectory API](./search-trajectory-api.md)
- [Reasoning Chain API](./reasoning-chain-api.md)
- [Introspection API](./introspection-api.md)
- [Implementation Mapping](./implementation-mapping.md)

---

**Версия**: 1.0
**Последнее обновление**: 2025-11-14
**Статус**: Design Proposal
