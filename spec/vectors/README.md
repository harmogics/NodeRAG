# Векторные алгоритмы в NodeRAG: Полный обзор

## Введение

Этот раздел документации описывает **векторные алгоритмы** в NodeRAG — методы работы с embeddings (плотными векторными представлениями текста) для семантического поиска и кластеризации. Векторные операции работают в тесной интеграции с [графовыми алгоритмами](../graph/) для создания **hybrid semantic intelligence**.

---

## Обзор векторных алгоритмов

### 1. [Embedding Generation](./embedding-generation.md)

**Назначение**: Преобразование текста в 1536-мерные векторы

**Модель**: OpenAI text-embedding-3-small

**Роль**: Создание **геометрических проекций** семантического содержания

```
Text → Embedding Model → Vector (ℝ¹⁵³⁶) → Semantic Space
```

**Ключевые характеристики**:
- **Модель**: text-embedding-3-small (1536D)
- **Стоимость**: $0.02 per 1M tokens
- **Скорость**: ~3,000 texts/minute (batch)
- **Качество**: State-of-the-art для embedding models

**Философия**: **Геометрическая проекция интенции** — переход от символьного к измеримому (см. [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)).

---

## Архитектура взаимодействия

### Векторы + Граф = Hybrid Intelligence

```
┌─────────────────────────────────────────────────────────┐
│              АНАЛОГОВЫЙ СЛОЙ (Vectors)                   │
│                                                          │
│  Text → Embeddings (1536D) → HNSW Index                 │
│                                   │                      │
│                                   ↓                      │
│              Cosine Similarity Search                    │
│                                   │                      │
│                                   ↓                      │
│                         Top-k Neighbors                  │
│                        (Analog Attractors)               │
└─────────────────────────────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────┐
│             СИМВОЛИЧЕСКИЙ СЛОЙ (Graph)                   │
│                                                          │
│  Entities + Relationships → Knowledge Graph              │
│                                   │                      │
│                                   ↓                      │
│                           PPR Expansion                  │
│                                   │                      │
│                                   ↓                      │
│                      Structural Navigation               │
│                       (Symbolic Paths)                   │
└─────────────────────────────────────────────────────────┘
                              │
                              ↓
                    ┌─────────────────┐
                    │  Unified Graph  │
                    │  (Concatenated) │
                    └─────────────────┘
                              │
                              ↓
                    ┌─────────────────┐
                    │   PPR with      │
                    │ Hybrid Personalization │
                    └─────────────────┘
```

### Индексирование (Indexing Pipeline)

```
1. Text Decomposition (LLM)
        ↓
   Semantic Units + Entities + Attributes
        ↓
2. [EMBEDDING GENERATION]
        ↓
   Vectors (1536D для каждого узла)
        ↓
3. [HNSW INDEXING]
        ↓
   HNSW Index + HNSW Graph (Layer 0)
        ↓
4. Graph Construction
        ↓
   Knowledge Graph
        ↓
5. [GRAPH CONCATENATION]
        ↓
   Unified Graph (Knowledge + HNSW)
```

### Поиск (Query Pipeline)

```
1. User Query
        ↓
2. [QUERY EMBEDDING]
        ↓
   Query Vector (1536D)
        ↓
3. [HNSW SEARCH — Cosine Similarity]
        ↓
   Top-50 Nearest Neighbors (аналоговые аттракторы)
        ↓
4. Query Decomposition (LLM)
        ↓
   Entities
        ↓
5. Accurate Search (regex)
        ↓
   Matched Entities (символические аттракторы)
        ↓
6. [DUAL PERSONALIZATION]
   HNSW results: weight = 1.0
   Accurate results: weight = 2.0
        ↓
7. PPR Expansion (graph algorithm)
        ↓
   Weighted Nodes with PPR scores
        ↓
8. Context Assembly
        ↓
9. LLM Answer Generation
```

---

## Embedding Space: Концептуальная модель

### Математическая структура

**Embedding space**: ℝ¹⁵³⁶ — 1536-мерное вещественное векторное пространство

**Свойства**:

1. **Семантическая близость → Геометрическая близость**
   ```
   cosine_similarity(emb("solar"), emb("renewable")) ≈ 0.75
   cosine_similarity(emb("solar"), emb("unrelated")) ≈ 0.10
   ```

2. **Кластерная структура**
   ```
   Тема "Energy":
     emb("solar power")     ]
     emb("wind turbines")   } → образуют кластер
     emb("hydroelectric")   ]
   ```

3. **Размерность как емкость**
   ```
   1536 dimensions → может различать ~2^1536 концепций
   ```

### Философская интерпретация

**Embeddings как онтологическое пространство** (см. [Question as Key Paradigm](../research/question-as-key-paradigm.md)):

```
Платоновская идея (концепция)
           ↓
Embedding (геометрическая проекция)
           ↓
Проявления (близкие векторы)
```

**Ключевая идея**: Концепция **измеряется** через расстояние в пространстве.

---

## Cosine Similarity: Мера семантической близости

### Формула

```
cosine_similarity(u, v) = (u · v) / (||u|| · ||v||)

где:
- u · v = скалярное произведение
- ||u|| = норма вектора u
- ||v|| = норма вектора v
```

**Диапазон**: [-1, 1]
- 1 = идентичные векторы
- 0 = ортогональные (несвязанные)
- -1 = противоположные

### Cosine Distance

В HNSW используется **cosine distance**:

```
cosine_distance(u, v) = 1 - cosine_similarity(u, v)
```

**Диапазон**: [0, 2]
- 0 = идентичные
- 1 = ортогональные
- 2 = противоположные

### Реализация в HNSW

**Файл**: `NodeRAG/utils/HNSW.py:93`

```python
self.hnsw = hnswlib_noderag.Index(
    space='cosine',  # Использует cosine distance
    dim=1536
)
```

**HNSW автоматически**:
1. Вычисляет cosine distance между query и всеми узлами
2. Находит k ближайших соседей (smallest distance)
3. Возвращает (distance, node_id) пары

См. [HNSW Algorithm](../graph/hnsw-algorithm.md)

---

## Взаимодействие с графовыми алгоритмами

### 1. Embeddings → HNSW → PPR

**Цепочка**:
```
Embeddings → HNSW Index → k-NN Search → Attractors → PPR Personalization
```

**Файл**: `NodeRAG/search/search.py:84-98`

```python
# 1. Query embedding
query_embedding = np.array(
    self.config.embedding_client.request(query),
    dtype=np.float32
)

# 2. HNSW search (uses cosine similarity internally)
HNSW_results = self.hnsw.search(
    query_embedding,
    HNSW_results=self.config.HNSW_results  # 50
)

# 3. Personalization для PPR
personalization = {
    ids: self.config.similarity_weight  # 1.0
    for ids in retrieval.HNSW_results
}

# 4. PPR expansion
weighted_nodes = self.graph_search(personalization)
```

### 2. HNSW Graph → Graph Concatenation

**HNSW Layer 0** экспортируется как NetworkX граф:

```python
# Экспорт HNSW graph
hnsw_graph = hnsw.nxgraphs  # Layer 0 connections

# Объединение с knowledge graph
unified_graph = GraphConcat(knowledge_graph).concat(hnsw_graph)

# PPR на unified graph
ppr_results = sparse_PPR(unified_graph).PPR(personalization)
```

См. [Graph Concatenation](../graph/graph-construction.md)

### 3. Embeddings → K-means → Community Edges

**Для больших сообществ**:

```python
# Embeddings high-level elements
hle_embeddings = np.array([he.embedding for he in high_level_elements])

# Embeddings community nodes
node_embeddings = np.array([mapper.get_embedding(node) for node in nodes])

# Concatenate
all_embeddings = np.vstack([hle_embeddings, node_embeddings])

# K-means clustering
kmeans = faiss.Kmeans(d=1536, k=centroids)
kmeans.train(all_embeddings)
_, labels = kmeans.assign(all_embeddings)

# Соединяем HLE только с узлами того же кластера
for i, he in enumerate(high_level_elements):
    for j, node in enumerate(nodes):
        if labels[i] == labels[len(hle) + j]:
            graph.add_edge(he.hash_id, node, weight=1)
```

См. [Leiden Algorithm](../graph/leiden-community-detection.md)

---

## Библиотеки и зависимости

### Core Dependencies

| Библиотека | Версия | Алгоритмы | Документация |
|------------|--------|-----------|--------------|
| **openai** | 1.66.3 | Embedding generation | [LLM and AI](../dependencies/llm-and-ai.md) |
| **numpy** | 1.26.4 | Vector operations | [Vector Search](../dependencies/vector-search.md) |
| **hnswlib-noderag** | 0.8.2 | HNSW, cosine similarity | [Vector Search](../dependencies/vector-search.md) |
| **faiss-cpu** | 1.10.0 | K-means clustering | [Vector Search](../dependencies/vector-search.md) |
| **pyarrow** | 19.0.1 | Parquet storage | [Data Processing](../dependencies/data-processing-and-utilities.md) |
| **scipy** | 1.12.0 | Sparse operations | [Graph Operations](../dependencies/graph-operations.md) |

### Почему именно эти?

1. **OpenAI**: State-of-the-art качество embeddings
2. **NumPy**: Стандарт де-факто для векторных операций
3. **hnswlib-noderag**: Fastest ANN library + custom fork
4. **FAISS**: Оптимизированная k-means реализация (Intel MKL)
5. **PyArrow**: Эффективное сжатие векторов в Parquet

---

## Производительность

### Сравнение операций

| Операция | Время | Throughput | Библиотека |
|----------|-------|------------|------------|
| Embedding generation (single) | ~100-200ms | ~5-10 QPS | OpenAI API |
| Embedding generation (batch of 100) | ~500-1000ms | ~100-200 texts/s | OpenAI API |
| HNSW search (k=50) | ~5-10ms | ~100-200 QPS | hnswlib-noderag |
| Cosine similarity (pairwise) | ~0.001ms | ~1M ops/s | NumPy |
| K-means clustering (10K vectors) | ~1-2s | - | FAISS |
| Parquet save (200K embeddings) | ~10-30s | ~20 MB/s | PyArrow |
| Parquet load (200K embeddings) | ~5-10s | ~50 MB/s | PyArrow |

### Memory Footprint

| Компонент | Память | Расчет |
|-----------|--------|--------|
| Single embedding (1536D) | ~6 KB | 1536 × 4 bytes (float32) |
| 200K embeddings (raw) | ~1.2 GB | 200K × 6 KB |
| 200K embeddings (Parquet) | ~600 MB | ~50% compression |
| HNSW index (200K nodes) | ~4-6 GB | Embeddings + graph structure |
| K-means (10K vectors) | ~60 MB | Temporary during clustering |

---

## Концептуальные связи

### Звёздная архитектура

Векторные алгоритмы реализуют **аналоговые аттракторы** в [звёздной архитектуре](../research/star-pattern-architecture.md):

```
                Query (центр звезды)
                       │
               [Query Embedding]
                       │
              ╔════════╩════════╗
              ║                 ║
      [HNSW Search]      [Query Decomposition]
    (Cosine Similarity)    (Exact Match)
              ║                 ║
        ╔═════╩═════╗     ╔═════╩═════╗
        ║     ║     ║     ║     ║     ║
      A₁   A₂   A₃ A₄   A₅   A₆   A₇
   (Analog)  (Analog)  (Symbolic) (Symbolic)
    w=1.0     w=1.0      w=2.0    w=2.0
        ║     ║     ║     ║     ║     ║
        ╚═════╩═════╩═════╩═════╩═════╝
                       │
              [PPR Personalization]
```

**Dual-mode attractors**:
- **HNSW (аналоговые)**: Найдены через embedding similarity
- **Decomposed (символические)**: Найдены через exact match

### Парадигма вопроса как ключа

Embeddings реализуют **геометрическую интенцию** (см. [Question as Key Paradigm](../research/question-as-key-paradigm.md)):

```
Вопрос (незнание, интенция)
       ↓ [Embedding Model]
Query Vector (геометрическая проекция)
       ↓ [Cosine Similarity]
Семантическое поле (близкие векторы)
       ↓ [HNSW k-NN]
Аттракторы (конкретные проявления)
       ↓ [PPR Expansion]
Контекст (структурное расширение)
       ↓ [LLM Synthesis]
Ответ (новое знание)
```

### Дуальность: Аналоговое vs Символическое

| Vectors (Аналоговое) | Graph (Символическое) |
|----------------------|----------------------|
| Непрерывное пространство (ℝ¹⁵³⁶) | Дискретные узлы и рёбра |
| Cosine similarity | Пути в графе |
| Имплицитные связи | Явные отношения |
| System 1 (быстрое) | System 2 (медленное) |
| Интуиция | Логика |
| HNSW search (~5-10ms) | PPR expansion (~100-200ms) |

**Hybrid Intelligence** = Vectors + Graph

---

## Оптимизация и trade-offs

### 1. Embedding Model

**Trade-off**: Quality vs. Cost vs. Speed

| Model | Dimensions | Cost (per 1M) | Quality | Speed |
|-------|------------|---------------|---------|-------|
| text-embedding-3-small | 1536 | $0.02 | ⭐⭐⭐⭐ | Fast |
| text-embedding-3-large | 3072 | $0.13 | ⭐⭐⭐⭐⭐ | Slow |
| ada-002 (legacy) | 1536 | $0.10 | ⭐⭐⭐ | Fast |
| Local (SBERT) | 384-768 | Free | ⭐⭐ | Fast (GPU) |

**NodeRAG**: text-embedding-3-small (баланс)

### 2. HNSW Parameters

**Trade-off**: Recall vs. Speed

См. [HNSW Algorithm](../graph/hnsw-algorithm.md) для деталей.

### 3. Batch Size

**Trade-off**: Throughput vs. Latency

| Batch Size | Latency | Throughput | Optimal для |
|------------|---------|------------|-------------|
| 1 | ~100ms | Low | Real-time query |
| 10 | ~200ms | Medium | Small batch |
| 100 | ~500ms | High | Indexing |
| 1000+ | Rate limit | - | Избегать |

**NodeRAG**: batch_size=100 (indexing)

---

## Расширения и будущие направления

### 1. Matryoshka Embeddings

**Идея**: Embeddings с переменной размерностью

```
1536D (full) → 768D → 384D → 192D
  ↑            ↑       ↑       ↑
Full quality  Good   OK    Fast
```

**Преимущество**: Можно truncate для экономии памяти.

### 2. Late Interaction

Вместо сравнения aggregated embeddings, сравниваем token-level:

```
query = [emb(token₁), emb(token₂), ...]
doc = [emb(token₁), emb(token₂), ...]

similarity = Σᵢ max_j cosine(query[i], doc[j])
```

**Преимущество**: Более точное multi-aspect similarity.

### 3. Hybrid Embeddings

Комбинировать dense (OpenAI) + sparse (BM25):

```
final_score = α × dense_similarity + β × sparse_similarity
```

**Преимущество**: Лучше для keyword-sensitive queries.

### 4. Domain Adaptation

Fine-tune embeddings для специфической домены:

```
Base model → Fine-tune на domain corpus → Domain-specific embeddings
```

**Ограничение**: OpenAI не поддерживает fine-tuning text-embedding-3-small.

---

## Практические рекомендации

### Для разработчиков

1. **Batch API requests**:
   ```python
   # Плохо
   for text in texts:
       emb = client.request(text)

   # Хорошо
   embeddings = client.request(texts)  # Batch
   ```

2. **Cache embeddings**:
   ```python
   # Сохранить
   storage(embeddings).save_parquet('embeddings.parquet')

   # Загрузить
   embeddings = storage.load('embeddings.parquet')
   ```

3. **Normalize vectors** (для cosine similarity):
   ```python
   import numpy as np

   emb = emb / np.linalg.norm(emb)
   ```

4. **Monitor token usage**:
   ```python
   import tiktoken

   enc = tiktoken.get_encoding("cl100k_base")
   tokens = sum(len(enc.encode(text)) for text in texts)
   cost = tokens * 0.00000002  # $0.02 per 1M
   print(f"Cost: ${cost:.4f}")
   ```

### Для исследователей

1. **Evaluate embedding quality**:
   ```python
   from sklearn.metrics import pairwise_distances

   # Semantic similarity matrix
   distances = pairwise_distances(embeddings, metric='cosine')

   # Average within-cluster distance (должен быть низким)
   # Average between-cluster distance (должен быть высоким)
   ```

2. **Visualize embedding space**:
   ```python
   from sklearn.manifold import TSNE

   tsne = TSNE(n_components=2)
   embeddings_2d = tsne.fit_transform(embeddings)

   plt.scatter(embeddings_2d[:, 0], embeddings_2d[:, 1])
   ```

3. **Analyze HNSW recall**:
   ```python
   # Ground truth: brute force k-NN
   true_neighbors = brute_force_knn(query, embeddings, k=50)

   # HNSW result
   hnsw_neighbors = hnsw.search(query, k=50)

   # Recall
   recall = len(set(true_neighbors) & set(hnsw_neighbors)) / 50
   print(f"Recall@50: {recall:.3f}")
   ```

---

## Часто задаваемые вопросы (FAQ)

### Q1: Почему 1536 dimensions?

**A**: OpenAI text-embedding-3-small использует 1536D архитектуру. Это баланс между:
- **Качество**: Достаточно для различения нюансов
- **Память**: Не слишком большой (6 KB per vector)
- **Скорость**: HNSW search быстрый даже для 1536D

### Q2: Можно ли использовать меньшую размерность?

**A**: Да, можно truncate:
```python
embedding_768d = embedding_1536d[:768]  # First 768 dims
```

Но качество снижается (~5-10%).

### Q3: Почему cosine similarity, а не Euclidean distance?

**A**: Cosine similarity **инвариантна** к магнитуде:
```
cosine(v, 2v) = 1.0  # Одинаковое направление
euclidean(v, 2v) ≠ 0 # Разное расстояние
```

Для текста важно **направление** (семантика), не магнитуда.

### Q4: Как обрабатывать multilingual тексты?

**A**: OpenAI embeddings multilingual:
```python
emb_en = embed("renewable energy")
emb_ru = embed("возобновляемая энергия")
cosine_similarity(emb_en, emb_ru) ≈ 0.70  # Хорошая близость!
```

Но качество варьируется по языкам.

### Q5: Можно ли использовать GPU для embeddings?

**A**: OpenAI API не требует GPU (серверная генерация). Для локальных моделей (SBERT):
```python
model = SentenceTransformer('all-MiniLM-L6-v2')
model = model.to('cuda')  # GPU acceleration
```

---

## Дополнительные ресурсы

### Документация

- [Embedding Generation](./embedding-generation.md)
- [HNSW Algorithm](../graph/hnsw-algorithm.md)
- [LLM and AI Dependencies](../dependencies/llm-and-ai.md)
- [Vector Search Dependencies](../dependencies/vector-search.md)

### Научные статьи

1. **Mikolov, T., et al. (2013)**. "Efficient Estimation of Word Representations in Vector Space."

2. **Devlin, J., et al. (2018)**. "BERT: Pre-training of Deep Bidirectional Transformers."

3. **Malkov, Y. A., & Yashunin, D. A. (2018)**. "Efficient and robust approximate nearest neighbor search using HNSW."

### Концептуальные исследования

- [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)
- [Звёздная архитектура](../research/star-pattern-architecture.md)
- [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)

---

## Заключение

Векторные алгоритмы в NodeRAG образуют **аналоговый слой** hybrid intelligence:

1. **Embedding Generation**: Проекция текста в геометрическое пространство
2. **Cosine Similarity**: Измерение семантической близости
3. **HNSW**: Быстрый approximate k-NN search
4. **Integration с Graph**: Hybrid personalization для PPR

Вместе с [графовыми алгоритмами](../graph/), они реализуют **философию NodeRAG** — дуальность аналогового (интуиция) и символического (логика) для intelligent semantic navigation.

---

**Последнее обновление**: 2025-11-13

**Версия документации**: 1.0

**См. также**: [Graph Algorithms](../graph/), [Dependencies](../dependencies/), [Research](../research/)
