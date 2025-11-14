# ML Алгоритмы в NodeRAG: Обзор

## Введение

NodeRAG использует комбинацию **классических ML алгоритмов** и **deep learning моделей** для семантической обработки текста и навигации по графу знаний. Эта документация описывает все ML алгоритмы, их цели, реализации и взаимосвязи.

---

## Классификация алгоритмов

### По типу задачи

| Категория | Алгоритмы | Цель |
|-----------|-----------|------|
| **Deep Learning** | Transformer Embeddings, LLM (GPT-4o, Gemini) | Семантическое понимание текста |
| **Approximate Search** | HNSW | Быстрый k-NN в высоких размерностях |
| **Graph ML** | Personalized PageRank, Leiden, K-Core, Betweenness | Анализ и навигация графа |
| **Clustering** | K-means (FAISS) | Группировка векторов |
| **Text Processing** | BPE Tokenization, Semantic Chunking | Обработка текста |

### По реализации

| Тип | Алгоритмы | Пакеты |
|-----|-----------|--------|
| **Сторонние API** | OpenAI Embeddings, GPT-4o, Gemini | `openai`, `google-genai` |
| **Сторонние библиотеки** | HNSW, Leiden, K-means | `hnswlib-noderag`, `leidenalg`, `faiss` |
| **Open Source (NetworkX)** | K-Core, Betweenness | `networkx` |
| **Open Source (SciPy)** | Sparse PPR | `scipy` |
| **Собственные** | Semantic Chunking, Graph Construction | NodeRAG codebase |

---

## Обзор алгоритмов

### 1. [Transformer-based Text Embeddings](./transformer-embeddings.md)

**Тип**: Deep Learning (Transformer архитектура)
**Модель**: OpenAI text-embedding-3-small
**Размерность**: 1536D

**Цель**:
- Преобразование текста в векторы для semantic similarity
- Создание embeddings для узлов графа (SU, entities, attributes, HLE)
- Query embedding для поиска

**Реализация**: Сторонняя API (OpenAI)
**Пакет**: `openai==1.66.3`

**Производительность**:
- Стоимость: $0.02 per 1M tokens
- Скорость: ~3,000 texts/min (batch)
- Качество: MTEB score 62.3

### 2. [Approximate k-Nearest Neighbors (HNSW)](./approximate-knn.md)

**Тип**: Graph-based Approximate Search
**Алгоритм**: Hierarchical Navigable Small World

**Цель**:
- Быстрый поиск top-k ближайших векторов к query
- Генерация аналоговых аттракторов для PPR
- Экспорт HNSW графа для concatenation

**Реализация**: Сторонняя библиотека (custom fork)
**Пакет**: `hnswlib-noderag==0.8.2`

**Производительность**:
- Сложность: O(log N) vs O(N) brute force
- Query time: ~5-10ms для 200K nodes
- Recall: ~0.95-0.98

### 3. [Graph ML Algorithms](./graph-ml.md)

#### 3.1 Personalized PageRank (PPR)

**Тип**: Graph Ranking Algorithm

**Цель**:
- Распространение семантического влияния от аттракторов
- Расширение контекста через структуру графа
- Ранжирование узлов по релевантности

**Реализация**: Собственная (через SciPy sparse)
**Пакет**: `scipy==1.12.0`

**Производительность**:
- Сложность: O(E · T), T ~10-20 итераций
- Query time: ~100-200ms для 500K nodes

#### 3.2 Leiden Algorithm

**Тип**: Community Detection
**Алгоритм**: Modularity optimization

**Цель**:
- Детекция тематических кластеров в графе
- Группировка узлов для community summarization
- Создание high-level elements

**Реализация**: Сторонняя библиотека
**Пакет**: `igraph==0.11.8` + `leidenalg==0.10.2`

**Производительность**:
- Сложность: O(N log N)
- Execution time: ~30-60s для 500K nodes
- Modularity Q: ~0.4-0.6

#### 3.3 K-Core Decomposition

**Тип**: Graph Centrality Measure

**Цель**:
- Идентификация плотных подграфов
- Поиск важных entities для attribute generation
- Фильтрация ~10-15% entities

**Реализация**: Сторонняя библиотека
**Пакет**: `networkx==3.4.2`

**Производительность**:
- Сложность: O(E)
- Execution time: ~5-10s для 500K nodes

#### 3.4 Betweenness Centrality

**Тип**: Graph Centrality Measure

**Цель**:
- Идентификация мостов в графе
- Поиск важных entities (дополняет K-Core)
- Sampling для производительности

**Реализация**: Сторонняя библиотека
**Пакет**: `networkx==3.4.2`

**Производительность**:
- Сложность: O(k · E) с sampling (k=10)
- Execution time: ~30-60s для 500K nodes

### 4. [K-means Clustering](./clustering.md)

**Тип**: Unsupervised Clustering
**Алгоритм**: Lloyd's algorithm (optimized)

**Цель**:
- Кластеризация community embeddings
- Определение связей high-level elements с узлами
- Graph pruning для больших сообществ

**Реализация**: Сторонняя библиотека (optimized)
**Пакет**: `faiss-cpu==1.10.0`

**Производительность**:
- Сложность: O(k · N · d · i)
- Execution time: ~1-2s для 10K vectors
- Оптимизация: Intel MKL, SIMD

### 5. [LLM-based Algorithms](./llm-algorithms.md)

#### 5.1 Text Decomposition

**Модель**: GPT-4o
**Тип**: Structured Output Generation

**Цель**:
- Разбиение текста на semantic units, entities, relationships
- Извлечение структурированной информации
- Создание узлов для графа

**Реализация**: Сторонняя API (OpenAI)
**Пакет**: `openai==1.66.3`

#### 5.2 Query Decomposition

**Модель**: GPT-4o
**Тип**: Entity Extraction

**Цель**:
- Извлечение entities из query
- Генерация символических аттракторов
- Дополнение HNSW search

#### 5.3 Attribute Generation

**Модель**: GPT-4o
**Тип**: Text Generation (Summarization)

**Цель**:
- Создание детальных описаний для important entities
- Обогащение графа нарративным контекстом
- Улучшение семантического поиска

#### 5.4 Community Summarization

**Модель**: GPT-4o
**Тип**: Abstract Summarization

**Цель**:
- Создание high-level themes из сообществ
- Генерация абстракций высокого уровня
- Иерархия репрезентаций

#### 5.5 Answer Generation

**Модель**: GPT-4o
**Тип**: Question Answering

**Цель**:
- Синтез финального ответа из контекста
- Генерация связного текста
- Креативный синтез (temperature=0.7-1.0)

---

## Архитектура взаимодействия

### Indexing Pipeline

```
1. Text Input
        ↓
2. [Semantic Chunking] (собственный)
        ↓
3. [Text Decomposition] (GPT-4o LLM)
        ↓
   Semantic Units + Entities + Relationships
        ↓
4. [Graph Construction] (собственный)
        ↓
5. [Transformer Embeddings] (OpenAI API)
        ↓
6. [HNSW Indexing] (hnswlib-noderag)
        ↓
7. [K-Core + Betweenness] (NetworkX)
        ↓
8. [Attribute Generation] (GPT-4o LLM)
        ↓
9. [Leiden Community Detection] (leidenalg)
        ↓
10. [Community Summarization] (GPT-4o LLM)
        ↓
11. [K-means Clustering] (FAISS, если нужно)
        ↓
12. [Graph Concatenation] (собственный)
```

### Query Pipeline

```
1. User Query
        ↓
2. [Query Embedding] (OpenAI API)
        ↓
3. [HNSW k-NN Search] (hnswlib-noderag)
        ↓
   Analog Attractors
        ↓
4. [Query Decomposition] (GPT-4o LLM)
        ↓
   Symbolic Attractors
        ↓
5. [Personalized PageRank] (SciPy sparse)
        ↓
   Weighted Nodes
        ↓
6. [Context Assembly] (собственный)
        ↓
7. [Answer Generation] (GPT-4o LLM)
```

---

## Сравнительная таблица

| Алгоритм | Тип | Сложность | Реализация | Пакет |
|----------|-----|-----------|------------|-------|
| **Transformer Embeddings** | Deep Learning | O(L²·d) | Сторонний API | openai |
| **HNSW** | Approximate Search | O(log N) | Сторонний | hnswlib-noderag |
| **PPR** | Graph Ranking | O(E·T) | Собственный | scipy |
| **Leiden** | Community Detection | O(N log N) | Сторонний | leidenalg |
| **K-Core** | Graph Centrality | O(E) | Сторонний | networkx |
| **Betweenness** | Graph Centrality | O(k·E) | Сторонний | networkx |
| **K-means** | Clustering | O(k·N·d·i) | Сторонний | faiss |
| **GPT-4o (all)** | LLM | Varies | Сторонний API | openai |

---

## Производительность (типичный корпус 10K docs)

| Этап | Алгоритмы | Время | Стоимость |
|------|-----------|-------|-----------|
| **Text Decomposition** | GPT-4o | ~30-60 min | ~$5-10 |
| **Embeddings** | OpenAI API | ~1-2 min | ~$0.40 |
| **HNSW Build** | hnswlib | ~10-30 sec | Free |
| **K-Core + Betweenness** | NetworkX | ~30-60 sec | Free |
| **Attribute Generation** | GPT-4o | ~15-30 min | ~$10-15 |
| **Leiden** | leidenalg | ~30-60 sec | Free |
| **Community Summary** | GPT-4o | ~10-20 min | ~$5-10 |
| **K-means** | FAISS | ~1-2 sec | Free |
| **Total Indexing** | All | ~60-120 min | ~$20-40 |
| **Query** | HNSW + PPR + GPT-4o | ~1-2 sec | ~$0.01-0.02 |

---

## Концептуальные связи

### Hybrid Intelligence

**Dual-Process Theory**:

| System 1 (Fast, Intuitive) | System 2 (Slow, Deliberate) |
|-----------------------------|------------------------------|
| HNSW (~5-10ms) | PPR (~100-200ms) |
| Transformer Embeddings | LLM Reasoning |
| Analog (vectors) | Symbolic (graph) |
| Cosine Similarity | Graph Paths |

### Иерархия абстракций

```
КОНКРЕТИКА ↓
Tokens (BPE Tokenization)
    ↓
Semantic Units (LLM Text Decomposition)
    ↓
Entities + Relationships (LLM Extraction)
    ↓
Attributes (LLM + K-Core/Betweenness)
    ↓
Communities (Leiden Algorithm)
    ↓
High-level Elements (LLM Community Summary)
    ↓
АБСТРАКЦИЯ ↑
```

### ML Pipeline Stages

```
┌─────────────────────────────────────────┐
│   Deep Learning (Embeddings + LLM)      │
│   - Semantic understanding              │
│   - Structured extraction               │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│   Classical ML (HNSW, K-means, Graph)   │
│   - Fast search                         │
│   - Clustering                          │
│   - Graph analysis                      │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│   Hybrid Integration                    │
│   - Graph + Embeddings                  │
│   - Analog + Symbolic                   │
└─────────────────────────────────────────┘
```

---

## Зависимости

### Core ML Libraries

| Библиотека | Версия | Алгоритмы |
|------------|--------|-----------|
| `openai` | 1.66.3 | Embeddings, GPT-4o |
| `hnswlib-noderag` | 0.8.2 | HNSW |
| `scipy` | 1.12.0 | Sparse PPR |
| `networkx` | 3.4.2 | K-Core, Betweenness |
| `igraph` | 0.11.8 | Leiden (backend) |
| `leidenalg` | 0.10.2 | Leiden |
| `faiss-cpu` | 1.10.0 | K-means |
| `numpy` | 1.26.4 | Vector operations |

### Документация зависимостей

- [LLM and AI Dependencies](../dependencies/llm-and-ai.md)
- [Graph Operations Dependencies](../dependencies/graph-operations.md)
- [Vector Search Dependencies](../dependencies/vector-search.md)

---

## Trade-offs и оптимизации

### 1. Качество vs Стоимость

**Embeddings**:
- OpenAI text-embedding-3-small: Высокое качество, $0.02/1M
- Локальные модели (SBERT): Среднее качество, бесплатно

**Выбор NodeRAG**: OpenAI (баланс)

### 2. Точность vs Скорость

**k-NN Search**:
- Brute Force: 100% recall, ~500ms
- HNSW: ~95-98% recall, ~5-10ms

**Выбор NodeRAG**: HNSW (аппроксимация приемлема)

### 3. Exploration vs Exploitation

**PPR alpha**:
- α=0.50: Больше exploitation (близкие узлы)
- α=0.95: Больше exploration (далёкие узлы)

**Выбор NodeRAG**: α=0.85 (стандартный PageRank)

---

## Рекомендации

### Для разработчиков

1. **Batch processing**: Используйте batch API для embeddings и LLM calls
2. **Caching**: Кэшируйте embeddings, HNSW index, graph
3. **Async operations**: Параллельные LLM requests
4. **Monitor costs**: Логируйте token usage для API calls

### Для исследователей

1. **Evaluate**: Benchmark recall (HNSW), modularity (Leiden), precision (PPR)
2. **Ablation studies**: Отключайте компоненты для анализа вклада
3. **Hyperparameter tuning**: Grid search для M, ef, alpha, k
4. **Visualize**: t-SNE для embeddings, community structure для graphs

---

## Дополнительные ресурсы

### Детальная документация

- [Transformer Embeddings](./transformer-embeddings.md)
- [Approximate k-NN (HNSW)](./approximate-knn.md)
- [Graph ML Algorithms](./graph-ml.md)
- [K-means Clustering](./clustering.md)
- [LLM Algorithms](./llm-algorithms.md)

### Концептуальные исследования

- [Question as Key Paradigm](../research/question-as-key-paradigm.md)
- [Star Architecture](../research/star-pattern-architecture.md)
- [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)

### Связанные разделы

- [Graph Algorithms](../graph/)
- [Vector Algorithms](../vectors/)
- [Dependencies](../dependencies/)

---

**Последнее обновление**: 2025-11-13

**Версия документации**: 1.0

**Для вопросов**: См. README в корне проекта
