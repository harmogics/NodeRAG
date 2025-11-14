# K-means Clustering в NodeRAG

## Введение

NodeRAG использует **K-means clustering** для кластеризации embeddings в высокоразмерном пространстве (ℝ¹⁵³⁶). Алгоритм применяется для **graph pruning** — сокращения количества рёбер между high-level elements и узлами графа в больших сообществах.

**Ключевая идея**: Вместо создания рёбер между HLE и всеми узлами сообщества, создаём рёбра только с узлами в том же кластере (близкими по семантике).

---

## Описание алгоритма

### K-means: Lloyd's Algorithm

**K-means** — это классический алгоритм unsupervised clustering, который разбивает N точек на K кластеров, минимизируя **within-cluster sum of squares** (WCSS).

**Objective Function**:

```
WCSS = ∑ᵏ₌₁ᴷ ∑ₓ∈Cₖ ‖x - μₖ‖²
```

Где:
- `Cₖ` — кластер k
- `μₖ` — центроид кластера k
- `x` — точка (embedding) в кластере

**Цель**: Минимизировать WCSS

### Lloyd's Algorithm

**Итерационный процесс**:

1. **Initialization**: Случайно выбрать K центроидов
2. **Assignment Step**: Назначить каждую точку ближайшему центроиду
   ```
   c(x) = argminₖ ‖x - μₖ‖²
   ```
3. **Update Step**: Обновить центроиды как среднее точек кластера
   ```
   μₖ = (1/|Cₖ|) · ∑ₓ∈Cₖ x
   ```
4. **Convergence Check**: Повторять шаги 2-3 до сходимости

**Сходимость**: Обычно за 10-50 итераций

---

## Цель применения в NodeRAG

K-means используется для **graph pruning** при создании high-level elements:

### Проблема: Quadratic Growth

При создании HLE из больших сообществ возникает проблема **квадратичного роста** рёбер:

```
Community с N узлов → H high-level elements
Наивный подход: H · N рёбер (all-to-all)
```

**Пример**: Community с 10,000 узлов и 100 HLE → 1,000,000 рёбер

### Решение: Clustering-based Pruning

**Adaptive Approach**:

```python
centroids = ceil(√(N_nodes + N_HLE))
threshold = (N_nodes + N_HLE) / centroids

if threshold > Hcluster_size:  # e.g., 100
    # Use K-means clustering
    Apply K-means with K=centroids
    Create edges only within same cluster
else:
    # Small community: all-to-all
    Create all edges
```

**Эффект**:
- Квадратичный рост O(N²) → Линейный рост O(N)
- Сохраняет семантическую близость (HLE связаны с близкими узлами)

---

## Реализация

### Тип и пакет

**Тип**: Сторонняя библиотека (optimized)
**Пакет**: `faiss-cpu==1.10.0` (Facebook AI Similarity Search)
**Файл**: `NodeRAG/build/pipeline/summary_generation.py`

### Почему FAISS?

**FAISS (Facebook AI Similarity Search)**:
- Оптимизация для **высоких размерностей** (1536D)
- Использует **Intel MKL**, **SIMD** для ускорения
- **GPU support** (в NodeRAG используется CPU версия)
- ~10-100x быстрее чем scikit-learn для больших датасетов

### Код реализации

```python
import faiss
import numpy as np
import math

class SummaryGeneration:
    async def high_level_element_summary(self):
        """
        Создание high-level elements с clustering-based pruning
        """
        # Загрузка embeddings для всех узлов и HLE
        embedding_list = np.array(
            [self.mapper.embeddings[node] for node in All_nodes],
            dtype=np.float32
        )
        high_level_element_embedding = np.array(
            [he.embedding for he in self.high_level_elements],
            dtype=np.float32
        )

        # Concatenate: HLE embeddings + node embeddings
        all_embeddings = np.vstack([
            high_level_element_embedding,
            embedding_list
        ])

        # Определение количества кластеров
        centroids = math.ceil(math.sqrt(len(All_nodes) + len(self.high_level_elements)))
        threshold = (len(All_nodes) + len(self.high_level_elements)) / centroids

        if threshold > self.config.Hcluster_size:
            # K-means clustering через FAISS
            kmeans = faiss.Kmeans(
                d=all_embeddings.shape[1],  # Размерность (1536)
                k=centroids                  # Количество кластеров
            )

            # Training: Lloyd's algorithm
            kmeans.train(all_embeddings.astype(np.float32))

            # Assignment: assign each point to nearest centroid
            _, cluster_labels = kmeans.assign(all_embeddings.astype(np.float32))

            # Разделение labels
            high_level_element_cluster_labels = cluster_labels[:len(self.high_level_elements)]
            embedding_cluster_labels = cluster_labels[len(self.high_level_elements):]

            self.config.console.print(
                f'[bold green]KMeans Clustering with {centroids} centroids[/bold green]'
            )

            # Создание рёбер только внутри кластеров
            self.config.tracker.set(
                len(self.high_level_elements),
                'Adding High Level Element Summary'
            )

            for i in range(len(self.high_level_elements)):
                for j in range(len(All_nodes)):
                    # Проверка: same cluster AND related node
                    if (high_level_element_cluster_labels[i] == embedding_cluster_labels[j] and
                        All_nodes[j] in self.high_level_elements[i].related_node):
                        # Создание ребра
                        self.G.add_edge(
                            All_nodes[j],
                            self.high_level_elements[i].hash_id,
                            weight=1
                        )

        else:
            # Малые сообщества: all-to-all edges
            self.config.tracker.set(
                len(self.high_level_elements),
                'Adding High Level Element Summary'
            )

            for he in self.high_level_elements:
                for node in he.related_node:
                    self.G.add_edge(node, he.hash_id, weight=1)
```

### FAISS K-means API

```python
# Инициализация K-means
kmeans = faiss.Kmeans(
    d=1536,           # Размерность embeddings
    k=100,            # Количество кластеров
    niter=25,         # Число итераций (default 25)
    verbose=False,    # Логирование
    spherical=False   # Spherical K-means (cosine distance)
)

# Training (Lloyd's algorithm)
kmeans.train(embeddings)  # embeddings: np.array (N, 1536), dtype=float32

# Assignment (получение cluster labels)
distances, labels = kmeans.assign(embeddings)
# distances: (N,) расстояния до центроидов
# labels: (N,) cluster assignments
```

---

## Производительность

### Вычислительная сложность

**Lloyd's Algorithm**:
- **Time**: `O(k · N · d · i)`
  - k — количество кластеров
  - N — количество точек
  - d — размерность (1536)
  - i — число итераций (~25)
- **Space**: `O((N + k) · d)`

**Практическая сложность** с FAISS оптимизациями:
- Intel MKL: Vectorized distance computations
- SIMD: Parallel centroid updates
- ~10x faster than scikit-learn

### Практические метрики

**Типичный сценарий**:
- **Input**: 10,000 embeddings (1536D)
- **K**: ~100 clusters
- **Execution time**: ~1-2 seconds (FAISS CPU)
- **Iterations**: ~15-25 до сходимости

**Сравнение с альтернативами**:

| Реализация | Время (10K points) | Оптимизации |
|------------|-------------------|-------------|
| scikit-learn | ~10-20s | Базовая NumPy |
| FAISS CPU | ~1-2s | Intel MKL, SIMD |
| FAISS GPU | ~0.1-0.5s | CUDA |

**NodeRAG использует**: FAISS CPU (баланс производительности и deployment)

---

## Гиперпараметры

### Количество кластеров K

**Formula**:
```python
K = ceil(√(N_nodes + N_HLE))
```

**Обоснование**:
- Балансирует cluster size и количество кластеров
- Для N=10,000: K ≈ 100 (average cluster size ~100)

**Threshold для применения**:
```python
threshold = (N_nodes + N_HLE) / K

if threshold > Hcluster_size:  # default 100
    apply_clustering()
```

### FAISS параметры

| Параметр | Значение | Описание |
|----------|----------|----------|
| `d` | 1536 | Размерность embeddings |
| `k` | `ceil(√N)` | Количество кластеров |
| `niter` | 25 (default) | Число итераций Lloyd's algorithm |
| `spherical` | False | Euclidean distance (не cosine) |

**Примечание**: NodeRAG использует **Euclidean distance** для K-means, хотя embeddings нормализованы. Cosine distance (spherical K-means) может улучшить качество.

---

## Математические детали

### Distance Metric

**FAISS K-means по умолчанию использует Euclidean distance**:

```
d(x, μ) = ‖x - μ‖² = ∑ᵢ (xᵢ - μᵢ)²
```

**Альтернатива: Spherical K-means (cosine distance)**:

```python
kmeans = faiss.Kmeans(d=1536, k=100, spherical=True)
# Использует cosine distance: d(x, μ) = 1 - (x · μ) / (‖x‖ · ‖μ‖)
```

**Trade-off**:
- Euclidean: Учитывает magnitude embeddings
- Cosine: Только direction (лучше для normalized embeddings)

### Convergence Criteria

**FAISS stopping condition**:

```
Δcentroids < ε  или  iterations ≥ niter
```

Где:
```
Δcentroids = ‖μₖ⁽ᵗ⁺¹⁾ - μₖ⁽ᵗ⁾‖
```

**Практика**: Сходимость обычно за 15-25 итераций

---

## Связь с другими компонентами

### Input

1. **High-Level Element Embeddings** (из LLM community summarization + OpenAI embeddings)
2. **Node Embeddings** (из mapper: semantic units, entities, attributes)

### Output

**Pruned Graph**:
- Рёбра `(node, HLE)` только если:
  - `cluster(node) == cluster(HLE)` (same cluster)
  - `node` в `HLE.related_nodes` (Leiden community membership)

### Pipeline

```
Leiden Communities
       ↓
LLM Community Summarization
       ↓
High-Level Elements (HLE)
       ↓
Embedding Generation (OpenAI)
       ↓
┌──────┴──────┐
│ Small       │ Large
│ Community   │ Community
└──────┬──────┘
       │           ↓
All-to-All    K-means Clustering
Edges              ↓
       │     Cluster-based Pruning
       └──────┬──────┘
              ↓
       Graph with HLE
```

---

## Эффективность graph pruning

### Edge Reduction

**Без K-means (all-to-all)**:
```
N_edges = H · N
```

**С K-means clustering**:
```
N_edges ≈ H · (N / K)
```

**Пример**:
- Community: 10,000 узлов
- HLE: 100 elements
- K: 100 clusters

**Без clustering**: 100 · 10,000 = 1,000,000 edges
**С clustering**: 100 · (10,000 / 100) = 100,000 edges

**Reduction**: 10x меньше рёбер

### Semantic Preservation

**Гарантия**: HLE связан только с семантически близкими узлами (same cluster)

**Qualitative Impact**:
- PPR propagation: Более целевая (focused on relevant nodes)
- Graph size: Меньше памяти, быстрее operations
- Interpretability: HLE связан с coherent группой узлов

---

## Trade-offs и оптимизации

### 1. Euclidean vs Cosine Distance

**Текущий подход**: Euclidean distance (FAISS default)

**Альтернатива**:
```python
kmeans = faiss.Kmeans(d=1536, k=100, spherical=True)
```

**Trade-off**:
- Euclidean: Проще, быстрее
- Cosine: Лучше для normalized embeddings (OpenAI embeddings нормализованы)

**Рекомендация**: Попробовать spherical K-means для embeddings

### 2. K Selection: √N vs Elbow Method

**Текущий подход**: `K = ceil(√N)`

**Альтернатива**: Elbow method (найти optimal K)

**Trade-off**:
- √N: Простой, детерминированный, O(1)
- Elbow: Optimal K, но требует multiple runs O(k_max)

**NodeRAG выбор**: √N (производительность)

### 3. Threshold Parameter

**Текущий**: `Hcluster_size = 100`

**Влияние**:
- Low threshold (50): Чаще применяется K-means
- High threshold (200): Реже, больше all-to-all edges

**Trade-off**: Производительность vs simplicity

---

## Альтернативные реализации

### scikit-learn K-means

```python
from sklearn.cluster import KMeans

kmeans = KMeans(n_clusters=100, random_state=42)
labels = kmeans.fit_predict(embeddings)
```

**Pros**: Простой API, stable
**Cons**: ~10x медленнее FAISS для больших датасетов

### FAISS GPU K-means

```python
import faiss

# GPU resources
res = faiss.StandardGpuResources()

# GPU K-means
kmeans = faiss.Kmeans(d=1536, k=100, gpu=True)
kmeans.train(embeddings)
```

**Pros**: ~10-50x быстрее CPU
**Cons**: Требует GPU, deployment сложнее

**NodeRAG выбор**: FAISS CPU (баланс)

---

## Концептуальные связи

### Иерархия абстракций

K-means участвует в создании **иерархии абстракций**:

```
Semantic Units (ℝ¹⁵³⁶)
         ↓
   Leiden Clustering (Graph structure)
         ↓
   Communities (Semantic clusters)
         ↓
LLM Summarization
         ↓
High-Level Elements (ℝ¹⁵³⁶)
         ↓
K-means Clustering (Vector space clusters)
         ↓
Graph Pruning (Structure + Semantics)
```

### Hybrid Intelligence

**K-means как мост**:
- **Input**: Embeddings (analog, ℝ¹⁵³⁶)
- **Process**: Clustering (discrete grouping)
- **Output**: Graph structure (symbolic)

**Комбинация**:
- Vector space proximity (embeddings)
- Graph structure (Leiden communities)
- Semantic coherence (LLM summarization)

---

## Зависимости

### Core Package

| Библиотека | Версия | Алгоритм | Лицензия |
|------------|--------|----------|----------|
| `faiss-cpu` | 1.10.0 | K-means (optimized) | MIT |
| `numpy` | 1.26.4 | Array operations | BSD-3 |

### FAISS Installation

```bash
pip install faiss-cpu==1.10.0
```

**GPU версия** (опционально):
```bash
pip install faiss-gpu==1.10.0
```

### Связанные документы

- [Vector Search Dependencies](../dependencies/vector-search.md)
- [Transformer Embeddings](./transformer-embeddings.md)
- [Graph ML Algorithms](./graph-ml.md) (Leiden clustering)

---

## Рекомендации

### Для разработчиков

1. **Spherical K-means**: Попробовать `spherical=True` для normalized embeddings
2. **GPU acceleration**: Рассмотреть FAISS GPU для очень больших сообществ
3. **Caching**: Кэшировать cluster assignments для одинаковых сообществ
4. **Monitoring**: Логировать cluster sizes, edge reduction ratio

### Для исследователей

1. **Optimal K selection**:
   - Elbow method
   - Silhouette score
   - Davies-Bouldin index

2. **Alternative clustering**:
   - HDBSCAN (density-based, automatic K)
   - Agglomerative clustering (hierarchical)
   - Spectral clustering (graph-based)

3. **Evaluation metrics**:
   - Cluster coherence (cosine similarity within cluster)
   - Edge reduction ratio
   - PPR performance с/без pruning

---

## Дополнительные ресурсы

### Papers

- **K-means**: Lloyd, S. (1982). "Least squares quantization in PCM"
- **FAISS**: Johnson, J., Douze, M., & Jégou, H. (2019). "Billion-scale similarity search with GPUs"

### Связанные разделы

- [Transformer Embeddings](./transformer-embeddings.md)
- [Graph ML Algorithms](./graph-ml.md)
- [LLM Algorithms](./llm-algorithms.md)
- [Approximate k-NN (HNSW)](./approximate-knn.md)

---

**Последнее обновление**: 2025-11-14
**Версия документации**: 1.0
**Для вопросов**: См. README в корне проекта
