# Approximate k-Nearest Neighbors (HNSW)

## Описание алгоритма

**HNSW (Hierarchical Navigable Small World)** — алгоритм для аппроксимативного поиска k ближайших соседей в высокомерных пространствах. Строит многослойный граф с "small world" свойством для эффективной навигации.

### Математическая модель

**Задача k-NN**:
```
Дано:
- Набор точек X = {x₁, x₂, ..., xₙ} в ℝᵈ
- Query point q ∈ ℝᵈ
- k (число соседей)

Найти: k точек из X с наименьшим distance(q, xᵢ)
```

**HNSW Solution**: Построить иерархический граф для O(log N) поиска вместо O(N).

### Структура алгоритма

**Многослойный граф**:
```
Layer 2:  • ────────────── •      (sparse, long-range)
          │                │
Layer 1:  • ── • ────── • ─•      (medium density)
          │    │        │  │
Layer 0:  • ── • ─• ─• ─• ─•      (dense, all nodes)
```

**Свойства**:
- Layer 0: Все узлы, M=64 связей на узел
- Layer l>0: Экспоненциально меньше узлов, M/2 связей
- Greedy search от верхнего слоя к нижнему

---

## Цель применения в NodeRAG

### 1. Быстрый семантический поиск

**Задача**: Найти top-50 узлов графа, наиболее близких к query embedding.

**Применение**:
```python
query_emb = embedding_model(query)  # ℝ^1536
results = hnsw.search(query_emb, k=50)  # ~5-10ms
# → [(distance₁, node_id₁), ..., (distance₅₀, node_id₅₀)]
```

**Альтернатива** (brute force):
```python
# O(N) - сравнить со всеми узлами
distances = [cosine_distance(query_emb, node_emb) for node_emb in all_embeddings]
top_k = nsmallest(50, zip(distances, node_ids))  # ~500-1000ms для 200K nodes
```

### 2. Аналоговые аттракторы для PPR

**Задача**: HNSW results становятся персонализацией для Personalized PageRank.

```python
# HNSW находит семантически близкие узлы
hnsw_results = hnsw.search(query_emb, k=50)

# Используем как аттракторы для PPR
personalization = {
    node_id: 1.0  # Вес аналоговых аттракторов
    for _, node_id in hnsw_results
}

# PPR расширяет контекст через граф
ppr_results = sparse_PPR(graph).PPR(personalization)
```

### 3. Экспорт HNSW графа для concatenation

**Задача**: Layer 0 HNSW экспортируется как NetworkX граф и объединяется с knowledge graph.

```python
# Экспорт Layer 0 connections
hnsw_graph = hnsw.nxgraphs  # NetworkX Graph

# Concatenation
unified_graph = GraphConcat(knowledge_graph).concat(hnsw_graph)

# PPR на unified graph
ppr_results = sparse_PPR(unified_graph).PPR(personalization)
```

---

## Реализация

### Тип реализации

**Сторонняя (Custom Fork)**: hnswlib-noderag

### Библиотека/Пакет

**Оригинал**: https://github.com/nmslib/hnswlib
**Fork**: https://github.com/Wannabeasmartguy/hnswlib
**Пакет**: `hnswlib-noderag==0.8.2`

**Отличия от оригинала**:
- Добавлен метод `export_graph()` для экспорта Layer 0
- Улучшенная интеграция с NumPy

### Класс в NodeRAG

**Файл**: `NodeRAG/utils/HNSW.py`

```python
import hnswlib_noderag

class HNSW:
    def __init__(self, config):
        self.hnsw = hnswlib_noderag.Index(
            space='cosine',  # Cosine distance metric
            dim=1536         # Embedding dimensions
        )

        self.hnsw.init_index(
            max_elements=len(self.id_map),
            ef_construction=200,  # Build-time parameter
            M=64                  # Max connections per node
        )

    def search(self, query: np.ndarray, HNSW_results: int = 50):
        # k-NN query
        idx, dist = self.hnsw.knn_query(query, HNSW_results)

        # Map internal IDs to node IDs
        node_list = [self.id_map[idx[i]] for i in range(len(idx))]
        dist_list = list(dist)

        return zip(dist_list, node_list)
```

---

## Параметры алгоритма

### Конструирование индекса

| Параметр | Значение | Описание |
|----------|----------|----------|
| **M** | 64 | Max число связей на узел в Layer 0 |
| **ef_construction** | 200 | Beam width при построении |
| **max_elements** | 200K | Максимум узлов в индексе |
| **space** | 'cosine' | Метрика расстояния |

**Trade-off M**:
- M↑ → Выше recall, но больше память и медленнее построение
- M↓ → Меньше память, но ниже recall

### Поиск

| Параметр | Значение | Описание |
|----------|----------|----------|
| **ef** | 50 | Beam width при поиске (обычно = k) |
| **k** | 50 | Число ближайших соседей |

**Trade-off ef**:
- ef↑ → Выше recall, но медленнее поиск
- ef↓ → Быстрее, но ниже recall

---

## Производительность

### Сложность

| Операция | Naive (Brute Force) | HNSW |
|----------|---------------------|------|
| Build Index | O(N²) | O(N log N) |
| k-NN Query | O(N · d) | O(log N · d) |
| Memory | O(N · d) | O(N · M · d) ≈ O(N · d) |

### Benchmarks (NodeRAG, 200K nodes, 1536D)

| Метрика | Значение |
|---------|----------|
| Build time | ~2-3 часа (с генерацией embeddings) |
| Index size (disk) | ~5-6 GB |
| Index size (RAM) | ~6-8 GB |
| Query latency (k=50) | ~5-10 ms |
| Query throughput | ~100-200 QPS (single thread) |
| Recall@50 | ~0.95-0.98 (vs brute force) |

### Сравнение с альтернативами

| Method | Query Time | Recall | Library |
|--------|------------|--------|---------|
| **HNSW** | ~5-10ms | 0.95-0.98 | hnswlib |
| **FAISS IVF** | ~10-20ms | 0.90-0.95 | faiss-cpu |
| **Annoy** | ~10-15ms | 0.90-0.95 | annoy |
| **Brute Force** | ~500ms | 1.00 | numpy |

---

## Математическая модель

### Greedy Search Algorithm

```python
def search(query, entry_point, ef, k):
    # Candidates priority queue (min-heap by distance)
    candidates = [(distance(query, entry_point), entry_point)]
    # Results priority queue (max-heap by distance, size k)
    results = []
    visited = {entry_point}

    while candidates:
        current_dist, current = heappop(candidates)

        # Если current дальше самого дальнего в results, останавливаемся
        if results and current_dist > results[0][0]:
            break

        # Исследуем соседей
        for neighbor in get_neighbors(current):
            if neighbor not in visited:
                visited.add(neighbor)
                dist = distance(query, neighbor)

                # Добавляем в candidates если лучше worst result
                if len(results) < ef:
                    heappush(candidates, (dist, neighbor))
                    heappush(results, (-dist, neighbor))  # Max-heap
                    if len(results) > k:
                        heappop(results)
                elif dist < -results[0][0]:
                    heappush(candidates, (dist, neighbor))
                    heappush(results, (-dist, neighbor))
                    if len(results) > k:
                        heappop(results)

    return [(-dist, node) for dist, node in results[:k]]
```

### Layer Assignment

**Вероятность узла попасть в Layer l**:

```
P(layer = l) = (1/m_L)^l × (1 - 1/m_L)

где m_L = 1/ln(M) ≈ 1/ln(64) ≈ 0.24
```

**Результат**: Экспоненциально меньше узлов на верхних слоях.

---

## Связь с другими ML алгоритмами

### 1. Transformer Embeddings → HNSW

```
Text → Transformer → Embedding (ℝ^1536) → HNSW Index → Fast Search
```

См. [Transformer Embeddings](./transformer-embeddings.md)

### 2. HNSW → Personalized PageRank

```
HNSW Results → Personalization Weights → PPR → Expanded Context
```

См. [Graph ML Algorithms](./graph-ml.md)

### 3. HNSW Graph → Graph Concatenation

```
HNSW Layer 0 → NetworkX Graph → Concatenate with Knowledge Graph
```

См. [Graph Construction](../graph/graph-construction.md)

---

## Преимущества и ограничения

### Преимущества

✅ **Скорость**: O(log N) вместо O(N)
✅ **Высокий recall**: ~0.95-0.98 vs brute force
✅ **Масштабируемость**: Эффективен для миллионов узлов
✅ **Простота**: Не требует обучения (в отличие от IVF)
✅ **Custom fork**: Метод `export_graph()` для integration

### Ограничения

❌ **Память**: ~6-8 GB для 200K nodes (vs ~1.2 GB raw embeddings)
❌ **Build time**: Дорогостоящее построение индекса
❌ **Approximate**: Не гарантирует exact top-k
❌ **Static**: Удаление узлов дорого (требует rebuild)

---

## Код примеры

### Построение индекса

```python
from NodeRAG.utils import HNSW

hnsw = HNSW(config)

# Добавление узлов (embeddings)
nodes = [
    ('SU-123', np.random.rand(1536)),
    ('ENT-456', np.random.rand(1536)),
    ('ATTR-789', np.random.rand(1536))
]

hnsw.add_nodes(nodes)

# Сохранение
hnsw.save_HNSW()  # → HNSW_path (binary), id_map_path (parquet)
```

### Поиск

```python
query_emb = embedding_model(query)
results = hnsw.search(query_emb, HNSW_results=50)

for distance, node_id in results:
    print(f"{node_id}: {distance:.4f}")
```

### Экспорт графа

```python
# Layer 0 как NetworkX graph
hnsw_graph = hnsw.nxgraphs

print(f"Nodes: {hnsw_graph.number_of_nodes()}")
print(f"Edges: {hnsw_graph.number_of_edges()}")
print(f"Avg degree: {sum(dict(hnsw_graph.degree()).values()) / hnsw_graph.number_of_nodes()}")
```

---

## Дополнительные ресурсы

### Документация

- [hnswlib GitHub](https://github.com/nmslib/hnswlib)
- [hnswlib-noderag Fork](https://github.com/Wannabeasmartguy/hnswlib)
- [Vector Search Dependencies](../dependencies/vector-search.md)

### Научные статьи

1. **Malkov, Y. A., & Yashunin, D. A. (2018)**. "Efficient and robust approximate nearest neighbor search using hierarchical navigable small world graphs." IEEE TPAMI.

### Связанные алгоритмы

- [Transformer Embeddings](./transformer-embeddings.md)
- [Graph ML Algorithms](./graph-ml.md)
- [Clustering](./clustering.md)

---

**Последнее обновление**: 2025-11-13
