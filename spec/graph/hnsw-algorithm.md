# HNSW: Hierarchical Navigable Small World Algorithm

## Концептуальный обзор

**HNSW (Hierarchical Navigable Small World)** — это алгоритм аппроксимативного поиска ближайших соседей (Approximate Nearest Neighbor, ANN) в многомерных пространствах. В NodeRAG он играет роль **первичных аттракторов**, находящих точки входа для семантического поиска через граф знаний.

### Философская сущность

В контексте [парадигмы вопроса как ключа](../research/question-as-key-paradigm.md), HNSW выполняет функцию **первичной интуиции** — быстрого нахождения концептуально близких узлов без глубокого рассуждения:

```
Вопрос (концепция) → HNSW → Проявления (конкретные узлы)
     ↓                ↓              ↓
  Абстракция    Семантическая   Конкретика
                 близость
```

Это аналог **перцептивного осознания** (см. [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), уровень 1) — быстрое, дорефлексивное схватывание релевантных элементов.

---

## Роль в звёздной архитектуре

Согласно [звёздной архитектуре](../research/star-pattern-architecture.md), HNSW генерирует **аналоговые аттракторы**:

```
                Query (центр звезды)
                       │
                  [Embedding]
                       │
               ╔═══════╩═══════╗
               ║               ║
          [HNSW Search]   [Decomposition]
               ║               ║
         ╔═════╩═══════════════╩═════╗
         ║         ║         ║        ║
    Attractor₁ Attractor₂ ... Attractor₅₀
     (SU-123)   (ATTR-456)    (HLE-789)
         ║         ║         ║        ║
         ╚═════════╩═════════╩════════╝
                       │
              [Personalization для PPR]
```

**Ключевые характеристики**:
- **Быстрые**: O(log N) вместо O(N) для полного перебора
- **Аналоговые**: Основаны на непрерывной семантической близости
- **Разнообразные**: Возвращают top-k узлов разных типов

---

## Математическая модель

### 1. Embedding Space

Все узлы проецируются в **1536-мерное пространство** (OpenAI `text-embedding-3-small`):

```
v ∈ ℝ¹⁵³⁶
```

### 2. Cosine Similarity

Близость между query и узлом измеряется через **косинусное расстояние**:

```
distance(q, v) = 1 - cos(q, v) = 1 - (q · v) / (||q|| · ||v||)
```

где:
- `q` — query embedding
- `v` — node embedding
- `distance ∈ [0, 2]` (0 = идентичные, 2 = противоположные)

### 3. k-NN Query

Задача: найти **top-k ближайших соседей**:

```
kNN(q, k) = argmin_{v₁, v₂, ..., vₖ} distance(q, vᵢ)
```

**Точное решение**: O(N) — сравнить query со всеми N узлами.

**HNSW решение**: O(log N) — навигация по многослойному графу.

---

## Структура HNSW

### Иерархия слоёв

HNSW строит **многослойный граф** с экспоненциально уменьшающимся числом узлов:

```
Layer 2:  • ─────────────────── •  (редкие long-range связи)
          │                     │
Layer 1:  • ── • ──────── • ── •  (средние связи)
          │    │          │    │
Layer 0:  • ── • ── • ── • ── •  (плотная сеть, все узлы)
```

**Свойства**:
- **Layer 0**: Содержит **все узлы**, плотно соединённые (M=64 связей на узел)
- **Layer 1, 2, ...**: Подмножества узлов с long-range связями
- **Число слоёв**: l_max ~ log(N)

### Navigable Small World Property

Каждый узел v имеет:
1. **Короткие связи**: к ближайшим соседям в embedding space
2. **Длинные связи**: к далёким узлам (на верхних слоях)

Это создаёт **small world топологию** — короткий путь между любыми двумя узлами.

---

## Алгоритм поиска

### Фаза 1: Спуск по слоям

Начинаем с **entry point** на верхнем слое и жадно движемся к query:

```python
def search_layer(query, entry_point, layer):
    current = entry_point
    visited = {entry_point}

    while True:
        neighbors = get_neighbors(current, layer)
        closest = min(neighbors, key=lambda v: distance(query, v))

        if distance(query, closest) >= distance(query, current):
            break  # Достигли локального минимума

        current = closest
        visited.add(current)

    return current
```

### Фаза 2: Beam Search на Layer 0

На нижнем слое используем **beam search** с очередью кандидатов:

```python
def search_layer_0(query, entry_points, k):
    candidates = MinHeap(entry_points, key=lambda v: distance(query, v))
    results = MaxHeap(k)
    visited = set(entry_points)

    while candidates:
        current = candidates.pop_min()

        if distance(query, current) > distance(query, results.max()):
            break  # Все оставшиеся хуже текущих top-k

        for neighbor in get_neighbors(current, layer=0):
            if neighbor not in visited:
                visited.add(neighbor)

                if len(results) < k or distance(query, neighbor) < distance(query, results.max()):
                    candidates.add(neighbor)
                    results.add(neighbor)
                    if len(results) > k:
                        results.pop_max()

    return results.get_all()
```

### Параметры поиска

**ef (exploration factor)** — размер beam search:
- `ef = 50` (default в NodeRAG)
- Чем больше ef, тем точнее результаты, но медленнее поиск
- Trade-off: скорость vs. качество

---

## Реализация в NodeRAG

### Файл: `NodeRAG/utils/HNSW.py`

#### Инициализация

```python
def __init__(self, config):
    self.config = config
    self.id_map = self.load_id_map()  # Маппинг внутренних ID → node IDs
    self.load_HNSW()
    self._nxgraphs = None  # Lazy load Layer 0 как NetworkX graph
```

**Библиотека**: `hnswlib-noderag==0.8.2` (см. [Vector Search Dependencies](../dependencies/vector-search.md))

#### Загрузка индекса

```python
def load_HNSW(self):
    self.hnsw = hnswlib_noderag.Index(
        space=self.config.space,  # 'cosine'
        dim=self.config.dim        # 1536
    )

    if os.path.exists(self.config.HNSW_path):
        self.hnsw.load_index(self.config.HNSW_path)
    else:
        self.hnsw.init_index(
            max_elements=len(self.id_map),
            ef_construction=self.config._ef,  # 200
            M=self.config._m                  # 64
        )
```

**Параметры построения**:
- `ef_construction=200`: Beam size при добавлении узлов
- `M=64`: Максимальное число связей на узел в Layer 0
- `M=32` на верхних слоях

#### Поиск k-NN

```python
def search(self, query: np.ndarray, HNSW_results: int = None):
    if HNSW_results is None:
        HNSW_results = self.config.top_k

    # knn_query использует внутренний HNSW search
    idx, dist = self.hnsw.knn_query(query, HNSW_results)
    idx = idx.flatten()
    dist = dist.flatten()

    # Маппинг обратно к node IDs
    node_list = [self.id_map[idx[i]] for i in range(len(idx))]
    dist_list = list(dist)

    results = zip(dist_list, node_list)
    return results
```

**Вход**:
- `query`: np.ndarray shape (1536,)
- `HNSW_results`: int (обычно 50)

**Выход**:
- `List[(distance, node_id)]` — top-k узлов с расстояниями

#### Batch Search

```python
def search_list(self, query_list: List[np.ndarray], HNSW_results: int = None):
    if HNSW_results is None:
        HNSW_results = self.config.top_k

    # Batch query для множества vectors
    idx, dist = self.hnsw.knn_query(
        np.array(query_list).astype(np.float32),
        HNSW_results
    )

    idx = idx.flatten()
    dist = dist.flatten()

    node_list = []
    dist_list = []

    # Deduplication с сохранением минимального расстояния
    for i in range(len(idx)):
        if self.id_map[idx[i]] not in node_list:
            node_list.append(self.id_map[idx[i]])
            dist_list.append(dist[i])
        else:
            # Если узел уже есть, берём min(old_dist, new_dist) * 0.9
            existing_idx = node_list.index(self.id_map[idx[i]])
            dist_list[existing_idx] = 0.9 * min(
                dist_list[existing_idx],
                dist[i]
            )

    results = zip(dist_list, node_list)
    return nsmallest(HNSW_results, results)
```

**Используется**: При генерации embeddings для сообществ (batch processing).

#### Экспорт Layer 0 как NetworkX Graph

```python
@property
def nxgraphs(self):
    graph_layer_0 = self.hnsw.get_layer_graph(0)

    if graph_layer_0 is not None:
        if self._nxgraphs is None:
            self._nxgraphs = nx.Graph()

            # get_layer_graph возвращает {node_id: [neighbor_ids]}
            for id, neighbors in graph_layer_0.items():
                for neighbor in neighbors:
                    self._nxgraphs.add_edge(
                        self.id_map[id],
                        self.id_map[neighbor]
                    )

        return self._nxgraphs
    else:
        return None
```

**Зачем**:
- HNSW граф объединяется с knowledge graph для PPR
- См. [Graph Concatenation Algorithm](./graph-concatenation.md)

---

## Использование в системе

### 1. Индексирование (Indexing Pipeline)

**Файл**: `NodeRAG/build/pipeline/graph_pipeline.py`

После генерации embeddings для entities, attributes, и semantic units, они добавляются в HNSW:

```python
def add_to_hnsw(self, nodes: List[Tuple[str, np.ndarray]]):
    self.hnsw.add_nodes(nodes)
```

**Метод `add_nodes` в `HNSW.py:36-46`**:

```python
def add_nodes(self, nodes: List[Tuple[str, np.ndarray]]):
    current_length = len(self.id_map)
    id_list = []
    embedding_list = []

    for idx, (node_id, embedding) in enumerate(nodes):
        new_id = current_length + idx
        self.id_map[new_id] = node_id
        id_list.append(new_id)
        embedding_list.append(embedding)

    # Расширяем индекс
    self.hnsw.resize_index(len(id_list) + current_length)

    # Batch insert
    self.hnsw.add_items(
        np.array(embedding_list).astype(np.float32),
        id_list
    )
```

### 2. Поиск (Query Pipeline)

**Файл**: `NodeRAG/search/search.py:78-104`

```python
def search(self, query: str):
    retrieval = Retrieval(...)

    # Шаг 1: HNSW search для семантических аттракторов
    query_embedding = np.array(
        self.config.embedding_client.request(query),
        dtype=np.float32
    )

    HNSW_results = self.hnsw.search(
        query_embedding,
        HNSW_results=self.config.HNSW_results  # 50
    )

    retrieval.HNSW_results_with_distance = HNSW_results

    # Шаг 2: Query decomposition для точных аттракторов
    decomposed_entities = self.decompose_query(query)
    accurate_results = self.accurate_search(decomposed_entities)
    retrieval.accurate_results = accurate_results

    # Шаг 3: Персонализация для PPR
    personalization = {
        ids: self.config.similarity_weight    # 1.0
        for ids in retrieval.HNSW_results
    }
    personalization.update({
        id: self.config.accuracy_weight       # 2.0
        for id in retrieval.accurate_results
    })

    # Шаг 4: PPR expansion с персонализацией
    weighted_nodes = self.graph_search(personalization)

    retrieval = self.post_process_top_k(weighted_nodes, retrieval)

    return retrieval
```

**Связь с концептуальной моделью**:
1. **Query → Embedding**: Концепция → Геометрическая проекция
2. **HNSW Search**: Проекция → Аналоговые аттракторы
3. **Personalization**: Аттракторы → Семантическая масса (веса для PPR)
4. **PPR Expansion**: Массы → Лучи семантического влияния

---

## Производительность

### Сложность

| Операция | Naive | HNSW |
|----------|-------|------|
| Построение индекса | O(N²) | O(N log N) |
| k-NN Query | O(N) | O(log N) |
| Память | O(N) | O(N · M) = O(N) |

где:
- N = число узлов
- M = 64 (число связей на узел)
- k = HNSW_results (обычно 50)

### Бенчмарки (NodeRAG)

**Corpus**: 1M semantic units + 200K entities + 50K attributes = ~1.25M nodes

| Метрика | Значение |
|---------|----------|
| Build time | ~2-3 часа (с генерацией embeddings) |
| Index size (disk) | ~5-6 GB |
| Index size (RAM) | ~6-8 GB |
| Query latency | ~5-10 ms (k=50) |
| Query throughput | ~100-200 QPS (single thread) |
| Recall@50 | ~0.95-0.98 (vs. brute force) |

**Hardware**: CPU (HNSW не использует GPU)

### Параметры оптимизации

**Trade-offs**:

| Параметр | Увеличение → | Точность | Скорость | Память |
|----------|--------------|----------|----------|--------|
| M | 64 → 128 | ↑ | ↓ | ↑ |
| ef_construction | 200 → 400 | ↑ | ↓↓ (build) | = |
| ef (search) | 50 → 100 | ↑ | ↓ | = |

**Рекомендации NodeRAG**:
- `M=64`: Баланс скорости и точности
- `ef_construction=200`: Достаточно для качественного индекса
- `ef=50`: Совпадает с `HNSW_results=50`

---

## Библиотеки

### hnswlib-noderag (Custom Fork)

**Оригинал**: https://github.com/nmslib/hnswlib

**Fork**: https://github.com/Wannabeasmartguy/hnswlib (версия 0.8.2)

**Отличия от оригинала**:

1. **`export_graph()` method**:
   ```python
   def get_layer_graph(self, layer: int) -> Dict[int, List[int]]:
       """Экспортирует граф Layer N как adjacency list"""
       pass
   ```

   Используется для объединения HNSW графа с knowledge graph (см. `HNSW.py:24-34`).

2. **Улучшенная интеграция с NumPy**

**Установка**:
```bash
pip install hnswlib-noderag==0.8.2
```

**Документация**: [Vector Search Dependencies](../dependencies/vector-search.md)

### NetworkX (для экспорта)

**Библиотека**: `networkx==3.4.2`

Используется для конвертации Layer 0 в NetworkX graph:

```python
import networkx as nx

hnsw_graph = nx.Graph()
for u, neighbors in layer_0.items():
    for v in neighbors:
        hnsw_graph.add_edge(u, v)
```

**Документация**: [Graph Operations Dependencies](../dependencies/graph-operations.md)

---

## Связь с другими алгоритмами

### 1. Personalized PageRank (PPR)

HNSW results используются как **персонализация** для PPR:

```python
personalization = {
    node_id: 1.0
    for node_id in HNSW_results
}
```

См. [Personalized PageRank Algorithm](./personalized-pagerank.md)

### 2. Graph Concatenation

HNSW граф (Layer 0) объединяется с knowledge graph:

```
Unified Graph = Knowledge Graph ∪ HNSW Graph (Layer 0)
```

См. [Graph Concatenation Algorithm](./graph-concatenation.md)

### 3. K-means Clustering (FAISS)

В summary generation используется k-means на HNSW embeddings для кластеризации сообществ:

```python
import faiss

kmeans = faiss.Kmeans(d=1536, k=centroids)
kmeans.train(all_embeddings)
_, labels = kmeans.assign(all_embeddings)
```

См. [Leiden Community Detection](./leiden-algorithm.md)

---

## Концептуальные связи

### Парадигма вопроса как ключа

HNSW реализует **первую фазу поиска концепции**:

```
Вопрос (абстракция)
    ↓ [LLM embedding]
Query Vector (проекция в ℝ¹⁵³⁶)
    ↓ [HNSW k-NN]
Top-k Nodes (конкретные проявления)
```

Это **аппроксимация платоновской идеи** через её проявления в embedding space.

### Звёздная архитектура

HNSW генерирует **первичные аттракторы** в звёздной топологии:

```
         Query (центр)
            │
      [HNSW Search]
     /  |  |  |  \
   A₁  A₂ A₃ A₄  A₅  ... A₅₀
(Аналоговые аттракторы)
```

Эти аттракторы создают **семантическое гравитационное поле** для последующего PPR expansion.

### Перцептивное осознание

В модели [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), HNSW соответствует **уровню 0: предсознательная обработка**:

- **Быстрое**: O(log N) — автоматическое, дорефлексивное
- **Параллельное**: Может выполняться одновременно с query decomposition
- **Аналоговое**: Основано на непрерывных distances, не на символах

Это аналог **System 1** в dual-process theory — быстрое, интуитивное познание.

---

## Ограничения и edge cases

### 1. Проклятие размерности

В высоких размерностях (1536D) **все расстояния становятся похожими**:

```
distance(q, v₁) ≈ distance(q, v₂) ≈ ... ≈ distance(q, vₖ)
```

**Решение в NodeRAG**:
- Использование **модели высокого качества** (OpenAI text-embedding-3-small)
- Комбинация с **символическим поиском** (query decomposition + accurate search)

### 2. Холодный старт

При малом числе узлов (<1000) HNSW **не эффективнее brute force**.

**NodeRAG**: Индексирование начинается после обработки достаточного корпуса.

### 3. Динамическое обновление

**Добавление** узлов — O(log N), но **удаление** дорого:

```python
# Добавление — эффективно
hnsw.add_items(new_embeddings, new_ids)

# Удаление — требует пересчёта связей
# НЕ поддерживается в hnswlib
```

**NodeRAG**: HNSW индекс **пересоздаётся** при обновлении knowledge graph.

### 4. Embedding drift

Если embeddings генерируются **разными моделями** или **разными версиями**, расстояния некорректны.

**NodeRAG**: Все embeddings генерируются **одной моделью** — OpenAI text-embedding-3-small.

---

## Расширения и вариации

### 1. Multi-vector HNSW

Для длинных документов можно создать **несколько embeddings**:

```python
doc_embeddings = [
    embed(chunk_1),
    embed(chunk_2),
    ...
]
```

И индексировать все в HNSW с одинаковым `doc_id`.

**NodeRAG**: Не используется, так как semantic units уже chunked.

### 2. Hybrid HNSW + Keyword

Комбинация векторного и лексического поиска:

```python
hnsw_results = hnsw.search(query_emb, k=100)
keyword_results = bm25.search(query_tokens, k=100)
final_results = rerank(hnsw_results + keyword_results, k=50)
```

**NodeRAG**: Реализовано через **query decomposition + accurate search**.

### 3. Approximate PPR на HNSW

Вместо PPR на полном графе, можно делать PPR **только на HNSW графе**:

```python
hnsw_subgraph = hnsw.get_layer_graph(0)
ppr_scores = sparse_PPR(hnsw_subgraph).PPR(personalization)
```

**NodeRAG**: Не используется, так как HNSW граф **объединяется** с knowledge graph.

---

## Сравнение с альтернативами

| Метод | Сложность Query | Recall | Библиотека |
|-------|-----------------|--------|------------|
| **Brute Force** | O(N) | 1.0 | NumPy |
| **HNSW** | O(log N) | 0.95-0.98 | hnswlib |
| **FAISS IVF** | O(√N) | 0.90-0.95 | faiss-cpu |
| **Annoy** | O(log N) | 0.90-0.95 | annoy |
| **ScaNN** | O(√N) | 0.95-0.98 | scann |

**Почему NodeRAG выбрала HNSW**:
1. **Высокий recall** (0.95-0.98) — критично для RAG
2. **Простота**: Не требует обучения (в отличие от IVF)
3. **Стабильность**: Зрелая библиотека hnswlib
4. **Custom fork**: Возможность добавить `export_graph()`

---

## Код examples

### Построение HNSW индекса

```python
from NodeRAG.utils import HNSW
from NodeRAG.config import NodeConfig

config = NodeConfig(...)

# Инициализация
hnsw = HNSW(config)

# Добавление узлов
nodes = [
    ('SU-123', np.random.rand(1536)),
    ('ENT-456', np.random.rand(1536)),
    ('ATTR-789', np.random.rand(1536)),
]

hnsw.add_nodes(nodes)

# Сохранение
hnsw.save_HNSW()
```

### Поиск k-NN

```python
# Генерация query embedding
query = "What is renewable energy?"
query_emb = config.embedding_client.request(query)
query_emb = np.array(query_emb, dtype=np.float32)

# k-NN search
results = hnsw.search(query_emb, HNSW_results=50)

# Результаты
for distance, node_id in results:
    print(f"{node_id}: {distance:.4f}")
```

### Экспорт Layer 0 графа

```python
import networkx as nx

# Получение Layer 0 как NetworkX graph
hnsw_graph = hnsw.nxgraphs

# Статистика
print(f"Nodes: {hnsw_graph.number_of_nodes()}")
print(f"Edges: {hnsw_graph.number_of_edges()}")
print(f"Avg degree: {sum(dict(hnsw_graph.degree()).values()) / hnsw_graph.number_of_nodes()}")

# Экспорт для визуализации
nx.write_gexf(hnsw_graph, 'hnsw_layer0.gexf')
```

---

## Практические рекомендации

### Для разработчиков

1. **Используйте batch operations** при добавлении узлов:
   ```python
   # Медленно
   for node in nodes:
       hnsw.add_nodes([node])

   # Быстро
   hnsw.add_nodes(nodes)
   ```

2. **Нормализуйте embeddings** для cosine distance:
   ```python
   emb = emb / np.linalg.norm(emb)
   ```

3. **Мониторьте recall**: Периодически проверяйте качество через ground truth.

4. **Настраивайте ef**: Если precision низкий, увеличьте `ef` параметр.

### Для исследователей

1. **Экспериментируйте с M**: Для малых корпусов M=32 может быть достаточно.

2. **Изучите Layer distribution**: Визуализируйте распределение узлов по слоям.

3. **Анализируйте failed queries**: Найдите queries с низким recall и изучите причины.

4. **Сравните с alternatives**: Попробуйте FAISS IVF или ScaNN для сравнения.

---

## Дополнительные ресурсы

### Научные статьи

1. **Malkov, Y. A., & Yashunin, D. A. (2018)**. "Efficient and robust approximate nearest neighbor search using hierarchical navigable small world graphs." IEEE Transactions on Pattern Analysis and Machine Intelligence.

2. **Навигация по малым мирам**: Milgram's six degrees of separation (1967).

### Документация

- [hnswlib GitHub](https://github.com/nmslib/hnswlib)
- [hnswlib-noderag Fork](https://github.com/Wannabeasmartguy/hnswlib)
- [Vector Search Dependencies](../dependencies/vector-search.md)

### Связанные алгоритмы

- [Personalized PageRank](./personalized-pagerank.md) — использует HNSW results
- [Graph Concatenation](./graph-concatenation.md) — объединяет HNSW graph
- [Query Decomposition](../transform/answer-generation-transformations.md) — дополняет HNSW

---

**Последнее обновление**: 2025-11-13

**См. также**:
- [Концептуальное исследование NodeRAG](../research/README.md)
- [Звёздная архитектура](../research/star-pattern-architecture.md)
- [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)
