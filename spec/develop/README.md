# NodeRAG REST API Documentation

## Введение

Эта документация описывает **comprehensive REST API** для NodeRAG, спроектированный для **максимального открытия внутренних структур и операций** системы RAG внешним пользователям.

**Философия**: Полная transparency, inspectability, и extensibility.

---

## Что предоставляет API?

### 1. **Доступ к данным**
- ✅ **Graph Knowledge Base** - Полный доступ к графу знаний (узлы, рёбра, сообщества)
- ✅ **Vector Space** - HNSW index, embeddings, similarity search
- ✅ **Structured Context** - Все типы узлов с метаданными

### 2. **Прозрачность процессов**
- ✅ **Search Trajectory** - Пошаговая трассировка выполнения query
- ✅ **Reasoning Chain** - LLM reasoning и decision-making процесс
- ✅ **Performance Metrics** - Timing, costs, bottlenecks каждого шага

### 3. **Концептуальные структуры**
- ✅ **Dual-Mode Attractors** - Analog (HNSW) + Symbolic (entities)
- ✅ **Concept Networks** - Related concepts, paths, communities
- ✅ **Related Queries** - Генерация связанных вопросов

### 4. **Интроспекция и анализ**
- ✅ **PPR Personalization** - Веса аттракторов, convergence iterations
- ✅ **Node Rankings** - PPR scores, centrality measures
- ✅ **Semantic Journeys** - Paths между концептами в embedding space

---

## Структура документации

| Документ | Описание |
|----------|----------|
| [api-architecture.md](./api-architecture.md) | Архитектура API, layered design, принципы |
| [graph-api.md](./graph-api.md) | Graph API endpoints (nodes, edges, paths, communities) |
| [vector-api.md](./vector-api.md) | Vector API endpoints (HNSW search, embeddings) |
| [search-trajectory-api.md](./search-trajectory-api.md) | Траектории поиска, timeline, bottlenecks |
| [introspection-api.md](./introspection-api.md) | Attractors, concepts, related queries |
| [implementation-mapping.md](./implementation-mapping.md) | Mapping API → NodeRAG components |

---

## Quick Start

### 1. Основной query с полной траекторией

```http
POST /api/v1/query
Content-Type: application/json

{
  "query": "What did Emily Roberts research?",
  "include_trajectory": true,
  "include_reasoning": true
}
```

**Response**:
```json
{
  "status": "success",
  "data": {
    "query": "What did Emily Roberts research?",
    "answer": "Dr. Emily Roberts has conducted extensive research...",
    "retrieval": {
      "hnsw_results": [...],
      "accurate_results": [...],
      "ppr_scores": [...]
    },
    "trajectory": {
      "query_id": "uuid",
      "total_duration_ms": 2345,
      "steps": [
        {
          "step": "query_embedding",
          "duration_ms": 150,
          "output": {...}
        },
        {
          "step": "hnsw_search",
          "duration_ms": 8,
          "output": {
            "results": [...]
          }
        },
        ...
      ]
    },
    "reasoning": {
      "attractors": {
        "analog": [...],
        "symbolic": [...]
      },
      "ppr_personalization": {...},
      "node_selection": {...}
    }
  }
}
```

### 2. Исследование графа

```http
# Получить entity
GET /api/v1/graph/node/entity_456?include_neighbors=true

# Получить ego subgraph
POST /api/v1/graph/subgraph
{
  "method": "ego",
  "center": "entity_456",
  "radius": 2
}

# Найти путь между концептами
POST /api/v1/graph/paths
{
  "source": "entity_456",
  "target": "entity_789",
  "method": "shortest"
}
```

### 3. Vector search

```http
# Текстовый поиск
POST /api/v1/vectors/search
{
  "text": "renewable energy research",
  "k": 50
}

# Получить похожие узлы
GET /api/v1/vectors/similar/sem_unit_123?k=20

# Исследовать HNSW структуру
GET /api/v1/vectors/hnsw/layers
```

### 4. Получение attractors

```http
POST /api/v1/introspection/attractors
{
  "query": "What did Emily Roberts research?",
  "analog_k": 50,
  "include_content": true
}
```

**Response показывает**:
- Analog attractors (HNSW top-k) с distances
- Symbolic attractors (extracted entities) с matches
- Combined personalization vector для PPR

### 5. Концептуальные сети

```http
# Получить concept network
GET /api/v1/introspection/concepts/entity_456?depth=2

# Найти связанные концепты
GET /api/v1/introspection/concepts/similar/entity_456

# Генерировать related queries
POST /api/v1/introspection/related-queries
{
  "query": "What did Emily Roberts research?",
  "k": 5
}
```

---

## Ключевые возможности

### 1. Search Transparency

**Проблема**: Black-box RAG systems
**Решение**: Полная visibility траектории поиска

```
Query → Embedding → HNSW → Decomposition → PPR → Filtering → LLM
  ↓         ↓          ↓          ↓           ↓         ↓        ↓
150ms     8ms      1000ms      120ms       8ms    2000ms   (times)
```

**Вы видите**:
- Все промежуточные результаты
- Timing каждого шага
- Bottlenecks (обычно LLM calls)
- Cost breakdown

### 2. Dual-Mode Attractors

**Концепция**: Hybrid analog + symbolic search

**Analog Attractors** (HNSW):
- Semantic similarity в embedding space (ℝ¹⁵³⁶)
- Top-k nearest neighbors
- Weight: 1.0 per node

**Symbolic Attractors** (Query Decomposition):
- Extracted entities via LLM
- Exact matches в графе
- Weight: 2.0 per node (higher importance)

**Combined**: PPR personalization vector

```python
personalization = {
    "sem_unit_123": 1.0,  # Analog
    "sem_unit_456": 1.0,  # Analog
    "entity_456": 2.0,    # Symbolic (EMILY ROBERTS)
    ...
}
```

### 3. Concept Networks

**Проблема**: Isolated node access
**Решение**: Contextual networks

Для каждого концепта вы видите:
- **Immediate neighbors** - Direct connections в графе
- **Communities** - Leiden community membership
- **Related concepts** - Via paths + semantic similarity
- **Attributes** - LLM-generated descriptions
- **High-level elements** - Abstract themes

### 4. Reasoning Chain

**LLM Decision Points**:
1. **Query Decomposition**: Почему extracted эти entities?
2. **Attractor Selection**: Rationale для dual-mode
3. **PPR Personalization**: Почему эти веса?
4. **Node Filtering**: Criteria для selection
5. **Answer Synthesis**: Chain-of-thought

### 5. Performance Analytics

**Metrics доступны** для каждого query:
- **Execution Time**: Total + per step
- **Cost**: Embeddings + LLM calls ($USD)
- **Throughput**: Nodes/second, iterations/second
- **Cache Hit Rate**: Query cache, embedding cache

---

## Use Cases

### 1. Research & Analysis

**Scenario**: Understand how NodeRAG processes complex queries

```bash
# Execute query with full introspection
curl -X POST /api/v1/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the latest developments in renewable energy?",
    "include_trajectory": true,
    "include_reasoning": true
  }'

# Analyze attractors
curl -X POST /api/v1/introspection/attractors \
  -d '{"query": "...same query..."}'

# Get related queries
curl -X POST /api/v1/introspection/related-queries \
  -d '{"query": "...same query...", "k": 10}'
```

**Insights**:
- Почему system выбрала эти узлы?
- Какие концепты semantically related?
- Какие alternative questions можно задать?

### 2. Debugging & Optimization

**Scenario**: Query returns poor results

```bash
# Get trajectory
curl /api/v1/trajectory/{query_id}

# Analyze bottleneck steps
# → LLM answer generation: 2000ms (85%)
# → PPR execution: 120ms (5%)

# Inspect attractors
curl -X POST /api/v1/introspection/attractors/compare \
  -d '{
    "queries": [
      "Current poor query",
      "Known good query"
    ]
  }'

# Check concept networks
curl /api/v1/introspection/concepts/{concept_id}
```

**Optimizations**:
- Reduce LLM latency (caching, smaller model)
- Improve attractors (better decomposition)
- Add missing graph connections

### 3. Interactive Exploration

**Scenario**: User explores knowledge base interactively

```bash
# Start with entity
curl /api/v1/graph/node/entity_456?include_neighbors=true

# Get related concepts
curl /api/v1/introspection/concepts/entity_456?depth=2

# Explore concept path
curl -X POST /api/v1/graph/paths \
  -d '{
    "source": "entity_456",
    "target": "entity_789",
    "k": 3
  }'

# Get similar nodes in vector space
curl /api/v1/vectors/similar/entity_456?k=20
```

**Result**: Rich contextual understanding

### 4. Visualization & UI

**Scenario**: Build interactive UI на основе API

```javascript
// Frontend code
async function visualizeQuery(query) {
  // 1. Execute query
  const result = await fetch('/api/v1/query', {
    method: 'POST',
    body: JSON.stringify({ query, include_trajectory: true })
  });

  // 2. Get trajectory for timeline
  const trajectory = result.trajectory;
  renderTimeline(trajectory.steps);

  // 3. Get attractors for graph visualization
  const attractors = await fetch('/api/v1/introspection/attractors', {
    method: 'POST',
    body: JSON.stringify({ query })
  });

  renderAttractorGraph(attractors.data);

  // 4. Get concept networks for exploration
  const conceptId = attractors.data.symbolic.nodes[0].node_id;
  const network = await fetch(`/api/v1/introspection/concepts/${conceptId}`);

  renderConceptNetwork(network.data);
}
```

---

## Architecture Highlights

### Layered Design

```
┌─────────────────────────────────────┐
│      REST API Layer (FastAPI)       │  ← Public interface
├─────────────────────────────────────┤
│   Instrumentation Layer             │  ← Trajectory tracking,
│   - TrajectoryTracker               │    reasoning capture
│   - IntrospectionEngine             │
├─────────────────────────────────────┤
│   Core NodeRAG Components           │  ← Existing system
│   - NodeSearch, HNSW, PPR, LLM     │
├─────────────────────────────────────┤
│   Storage Layer                     │  ← Data persistence
│   - Graph DB, Vector Index, Cache  │
└─────────────────────────────────────┘
```

### Response Format

All endpoints return **structured JSON**:

```json
{
  "status": "success" | "error",
  "data": { ... },
  "metadata": {
    "query_id": "uuid",
    "timestamp": "ISO-8601",
    "execution_time_ms": 123
  },
  "links": {
    "self": "/api/v1/...",
    "related": [...]
  }
}
```

### Error Handling

Standardized error codes:

| Code | Status | Description |
|------|--------|-------------|
| `GRAPH_NODE_NOT_FOUND` | 404 | Graph node not found |
| `VECTOR_INVALID_DIM` | 400 | Invalid vector dimension |
| `PPR_TIMEOUT` | 504 | PPR execution timeout |
| `LLM_ERROR` | 502 | LLM API error |

---

## Implementation Status

### Phase 1: Core APIs (MVP) ✅ Designed

- ✅ Query API design
- ✅ Graph API design
- ✅ Vector API design
- ✅ Trajectory API design
- ✅ Introspection API design

### Phase 2: Implementation 🚧

- ⏳ FastAPI application setup
- ⏳ Instrumentation layer (TrajectoryTracker)
- ⏳ IntrospectionEngine implementation
- ⏳ Modified NodeSearch with tracking
- ⏳ API endpoints implementation

### Phase 3: Advanced Features

- ⏳ Streaming (SSE) for real-time answers
- ⏳ WebSocket for interactive sessions
- ⏳ GraphQL support
- ⏳ Batch operations
- ⏳ Advanced caching (Redis)

### Phase 4: Production

- ⏳ Authentication & Authorization
- ⏳ Rate limiting
- ⏳ Monitoring & Metrics (Prometheus)
- ⏳ Logging & Tracing
- ⏳ API documentation (Swagger/ReDoc)

---

## Technology Stack

### API Framework
- **FastAPI** 0.104.1 - Modern, fast, OpenAPI compliant
- **Uvicorn** 0.24.0 - ASGI server
- **Pydantic** 2.5.0 - Data validation

### Storage & Caching
- **Redis** 5.0.1 - Query cache, trajectory storage
- **NetworkX** 3.4.2 - Graph operations (existing)
- **HNSW** noderag fork - Vector index (existing)

### Monitoring
- **Prometheus Client** 0.19.0 - Metrics
- **OpenTelemetry** - Tracing (optional)

### Documentation
- **Swagger UI** - Interactive API docs
- **ReDoc** - Alternative API docs

---

## Example Workflows

### Workflow 1: Deep Query Analysis

```bash
# 1. Execute query
QUERY_RESULT=$(curl -X POST /api/v1/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What did Emily Roberts research?", "include_trajectory": true}')

QUERY_ID=$(echo $QUERY_RESULT | jq -r '.data.trajectory.query_id')

# 2. Get detailed trajectory
curl /api/v1/trajectory/$QUERY_ID > trajectory.json

# 3. Analyze attractors
curl -X POST /api/v1/introspection/attractors \
  -d '{"query": "What did Emily Roberts research?"}' > attractors.json

# 4. Explore top concept
CONCEPT_ID=$(jq -r '.data.attractors.symbolic.nodes[0].node_id' attractors.json)
curl /api/v1/introspection/concepts/$CONCEPT_ID > concept_network.json

# 5. Get related queries
curl -X POST /api/v1/introspection/related-queries \
  -d '{"query": "What did Emily Roberts research?", "k": 5}' > related_queries.json
```

### Workflow 2: Comparative Analysis

```bash
# Compare two queries
curl -X POST /api/v1/introspection/attractors/compare \
  -d '{
    "queries": [
      "What did Emily Roberts research?",
      "Tell me about renewable energy"
    ],
    "show_overlap": true
  }' > comparison.json

# Analyze overlap
jq '.data.comparison.overlap' comparison.json

# Get divergence
jq '.data.comparison.divergence' comparison.json
```

---

## Performance Considerations

### Typical Latencies

| Operation | Time | Notes |
|-----------|------|-------|
| Query (full) | ~2-3s | Including LLM |
| Query (no LLM) | ~200-300ms | Retrieval only |
| HNSW search | ~5-10ms | k=50 |
| PPR execution | ~100-200ms | 500K nodes |
| LLM answer | ~1-2s | GPT-4o |

### Caching Strategy

- **Query results**: Redis, TTL 1h
- **Embeddings**: Redis, TTL 24h
- **Trajectories**: Redis, TTL 6h
- **Graph stats**: In-memory, TTL 30min

### Scalability

- **Horizontal**: Multiple API instances behind load balancer
- **Vertical**: Larger HNSW index, graph sharding
- **Cache**: Redis cluster для distributed caching

---

## Security & Privacy

### Authentication

**Bearer Token (JWT)**:
```http
Authorization: Bearer <token>
```

### Role-Based Access Control

| Role | Permissions |
|------|-------------|
| `user` | Query, basic introspection |
| `developer` | Full trajectory, reasoning chain |
| `admin` | Graph modifications, system config |

### Data Privacy

- **Query logging**: Optional, configurable retention
- **PII filtering**: Automatic detection and masking
- **Access audit**: Who accessed what and when

---

## Next Steps

### For Developers

1. **Read** [api-architecture.md](./api-architecture.md) для understanding design
2. **Review** [implementation-mapping.md](./implementation-mapping.md) для code mapping
3. **Implement** instrumentation layer (TrajectoryTracker)
4. **Test** with existing NodeRAG codebase

### For Researchers

1. **Explore** [introspection-api.md](./introspection-api.md) для research capabilities
2. **Analyze** attractors и concept networks
3. **Experiment** with different queries
4. **Publish** findings на основе API insights

### For Product Teams

1. **Design** UI/UX на основе API capabilities
2. **Build** interactive visualizations
3. **Create** user-friendly query interface
4. **Integrate** with existing tools

---

## FAQ

**Q: Почему REST API а не GraphQL?**
A: REST проще для начала, но GraphQL endpoint планируется (см. api-architecture.md)

**Q: Как handle large graphs (>1M nodes)?**
A: Pagination на всех list endpoints (max 1000 items/page), graph sharding планируется

**Q: Можно ли modify graph через API?**
A: Да, POST /graph/node и POST /graph/edge (требует admin role)

**Q: Что если PPR не converges?**
A: Timeout 30s, partial results returned с warning

**Q: Как cache invalidation работает?**
A: TTL-based + manual invalidation при graph modifications

---

## Contributing

### Reporting Issues

```bash
# Template
- **API Endpoint**: /api/v1/...
- **Request**: {...}
- **Expected**: ...
- **Actual**: ...
- **Error**: ...
```

### Suggesting Features

```bash
# Template
- **Feature**: ...
- **Use Case**: ...
- **API Design**: ...
- **Implementation**: ...
```

---

## Связанные ресурсы

### Документация
- [API Architecture](./api-architecture.md)
- [Graph API Reference](./graph-api.md)
- [Vector API Reference](./vector-api.md)
- [Search Trajectory API](./search-trajectory-api.md)
- [Introspection API](./introspection-api.md)
- [Implementation Mapping](./implementation-mapping.md)

### NodeRAG Core
- [ML Algorithms Documentation](../algo/)
- [Graph Algorithms](../graph/)
- [Vector Algorithms](../vectors/)
- [Conceptual Research](../research/)

---

**Версия**: 1.0
**Последнее обновление**: 2025-11-14
**Статус**: Design Proposal - Ready for Implementation
**Авторы**: Claude (Anthropic) + NodeRAG Team

---

## License

MIT License (same as NodeRAG core)

Copyright (c) 2024 NodeRAG Contributors

---

**END OF README**
