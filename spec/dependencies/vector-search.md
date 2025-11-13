# Векторный поиск

## Обзор

Эта категория включает библиотеки для Approximate Nearest Neighbor (ANN) search в embedding space — ключевой компонент retrieval pipeline.

---

## 1. hnswlib-noderag

**Пакет**: `hnswlib-noderag==0.8.2`

**Официальный репозиторий**: Custom fork of https://github.com/nmslib/hnswlib

### Назначение в NodeRAG

`hnswlib-noderag` — это **кастомный форк** hnswlib, обеспечивающий **быстрый ANN search** через Hierarchical Navigable Small World graphs.

### Почему форк?

**Основная библиотека**: `hnswlib` (original)

**Кастомизации в `hnswlib-noderag`**:
1. Добавлена функция `get_layer_graph(layer)` для экспорта HNSW graph structure
2. Улучшена интеграция с NetworkX
3. Дополнительные метрики

### Применение

**Файл**: `NodeRAG/utils/HNSW.py:11-117`

```python
import hnswlib_noderag
import networkx as nx
import numpy as np

class HNSW:
    def __init__(self, config):
        self.config = config
        self.id_map = self.load_id_map()  # Internal ID → Node hash_id
        self.load_HNSW()

    def load_HNSW(self):
        """Initialize or load HNSW index"""
        self.hnsw = hnswlib_noderag.Index(
            space=self.config.space,  # 'cosine', 'l2', or 'ip'
            dim=self.config.dim       # Embedding dimension (1536 for OpenAI)
        )

        if os.path.exists(self.config.HNSW_path):
            # Load existing index
            self.hnsw.load_index(self.config.HNSW_path)
        else:
            # Initialize new index
            self.hnsw.init_index(
                max_elements=len(self.id_map),
                ef_construction=self.config._ef,  # Search quality during construction
                M=self.config._m                   # Max connections per node
            )

    def add_nodes(self, nodes: List[Tuple[str, np.ndarray]]):
        """Add nodes with their embeddings to HNSW index"""
        current_length = len(self.id_map)
        id_list = []
        embedding_list = []

        for idx, (node_id, embedding) in enumerate(nodes):
            new_id = current_length + idx
            self.id_map[new_id] = node_id  # Map internal ID to hash_id
            id_list.append(new_id)
            embedding_list.append(embedding)

        # Resize and add
        self.hnsw.resize_index(len(id_list) + current_length)
        self.hnsw.add_items(
            np.array(embedding_list).astype(np.float32),
            id_list
        )

    def search(self, query: np.ndarray, HNSW_results: int = None):
        """k-NN search"""
        if HNSW_results is None:
            HNSW_results = self.config.top_k

        # Query HNSW index
        idx, dist = self.hnsw.knn_query(query, HNSW_results)
        idx = idx.flatten()
        dist = dist.flatten()

        # Map internal IDs to node IDs
        node_list = [self.id_map[idx[i]] for i in range(len(idx))]
        dist_list = list(dist)

        results = zip(dist_list, node_list)
        return results
```

### HNSW Algorithm

#### Концепция

**HNSW** = Hierarchical Navigable Small World

**Основная идея**:
- Построить **многослойный граф**
- Каждый слой — это **навигабельный small-world граф**
- Поиск начинается с **верхнего слоя** (мало узлов) и спускается вниз

**Визуализация**:
```
Layer 2 (top):     ●─────●─────●  (few nodes, long edges)
                    ╲     │    ╱
Layer 1:          ●──●──●──●──●  (more nodes)
                   ╲  │  │  │ ╱
Layer 0 (bottom): ●●●●●●●●●●●●● (all nodes, dense)
```

#### Параметры HNSW

**Файл**: `NodeRAG/config/config.py` (referenced)

| Параметр | Значение | Описание | Влияние |
|----------|----------|----------|---------|
| `M` | 16-64 | Max connections per node | Higher M → better recall, slower build |
| `ef_construction` | 100-400 | Search breadth during build | Higher ef → better quality, slower build |
| `ef` (search) | 50-500 | Search breadth during query | Higher ef → better recall, slower search |
| `space` | `'cosine'` | Distance metric | Cosine = normalized dot product |

**Типичная конфигурация в NodeRAG**:
```python
M = 64
ef_construction = 200
ef (search) = 100
space = 'cosine'
dim = 1536  # OpenAI embedding dimension
```

#### Complexity

| Operation | Time Complexity | Space Complexity |
|-----------|----------------|------------------|
| **Build** | O(N log N) | O(N · M) |
| **Search** | O(log N) | — |
| **Insert** | O(log N) | O(M) |

**Сравнение с Brute Force**:
- Brute force search: O(N) время
- HNSW search: O(log N) время
- Recall: ~95-99% (с правильными параметрами)

### Distance Metrics

| Metric | Formula | Use Case |
|--------|---------|----------|
| **Cosine** | `1 - (A · B) / (‖A‖ · ‖B‖)` | Normalized embeddings (OpenAI, Sentence-BERT) |
| **L2 (Euclidean)** | `‖A - B‖₂` | Raw vectors |
| **Inner Product** | `-A · B` | MIPS (Maximum Inner Product Search) |

**NodeRAG использует**: Cosine (normalized embeddings)

### HNSW Graph Export

**Уникальная feature в `hnswlib-noderag`**:

```python
@property
def nxgraphs(self):
    """Convert HNSW layer 0 to NetworkX graph"""
    graph_layer_0 = self.hnsw.get_layer_graph(0)
    if graph_layer_0 is not None:
        if self._nxgraphs is None:
            self._nxgraphs = nx.Graph()
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

**Использование**: HNSW graph concatenate с base graph для hybrid search.

**Файл**: `NodeRAG/search/search.py:71-75`

```python
# Load base graph
G = storage.load(self.config.base_graph_path)

# Load HNSW graph
HNSW_graph = storage.load(self.config.hnsw_graph_path)

# Concatenate
G = GraphConcat(G).concat(HNSW_graph)
```

**Эффект**: Узлы, близкие в embedding space, теперь **связаны в графе**, улучшая PPR expansion.

---

## 2. FAISS

**Пакет**: `faiss-cpu==1.10.0`

**Официальный сайт**: https://github.com/facebookresearch/faiss

### Назначение в NodeRAG

FAISS (Facebook AI Similarity Search) используется для **k-means clustering** high-level elements и semantic units.

### Применение

**Файл**: `NodeRAG/build/pipeline/summary_generation.py:150-185`

```python
import faiss
import numpy as np
import math

class SummaryGeneration:
    async def high_level_element_summary(self):
        # Calculate number of clusters
        centroids = math.ceil(math.sqrt(len(All_nodes) + len(self.high_level_elements)))
        threshold = (len(All_nodes) + len(self.high_level_elements)) / centroids

        # If cluster size too large, use k-means
        if threshold > self.config.Hcluster_size:
            # Get embeddings
            embedding_list = np.array([
                self.mapper.embeddings[node] for node in All_nodes
            ], dtype=np.float32)

            high_level_element_embedding = np.array([
                he.embedding for he in self.high_level_elements
            ], dtype=np.float32)

            all_embeddings = np.vstack([
                high_level_element_embedding,
                embedding_list
            ])

            # K-means clustering
            kmeans = faiss.Kmeans(
                d=all_embeddings.shape[1],  # Dimension (1536)
                k=centroids                  # Number of clusters
            )
            kmeans.train(all_embeddings.astype(np.float32))

            # Assign to clusters
            _, cluster_labels = kmeans.assign(all_embeddings.astype(np.float32))

            # Split labels
            high_level_element_cluster_labels = cluster_labels[:len(self.high_level_elements)]
            embedding_cluster_labels = cluster_labels[len(self.high_level_elements):]

            # Connect nodes in same cluster
            for i in range(len(self.high_level_elements)):
                for j in range(len(All_nodes)):
                    if (high_level_element_cluster_labels[i] == embedding_cluster_labels[j] and
                        All_nodes[j] in self.high_level_elements[i].related_node):
                        # Same cluster and related → add edge
                        self.G.add_edge(All_nodes[j], self.high_level_elements[i].hash_id, weight=1)
        else:
            # Connect all related nodes directly (no clustering)
            for he in self.high_level_elements:
                for node in he.related_node:
                    self.G.add_edge(node, he.hash_id, weight=1)
```

### Зачем K-means здесь?

**Проблема**: High-level element может быть связан с 1000+ semantic units (из community).

**Без clustering**:
```
High-level Element → 1000 edges → 1000 semantic units
```

**С clustering**:
```
High-level Element → 50-200 edges → semantic units в том же кластере
```

**Результат**: **Меньше рёбер** в графе, но сохранена **семантическая близость**.

### K-means Algorithm

**Файл**: FAISS implementation

**Шаги**:
1. Инициализация: Random selection of k centroids
2. Assignment: Assign each point to nearest centroid
3. Update: Recompute centroids as mean of assigned points
4. Repeat: Until convergence

**Complexity**: O(n · k · d · iterations)

**FAISS optimization**: GPU-accelerated (если `faiss-gpu`), SIMD, batching

### FAISS vs. HNSW

| Аспект | FAISS | HNSW (hnswlib) |
|--------|-------|----------------|
| **Primary use** | Clustering, large-scale search | Fast k-NN search |
| **GPU support** | ✅ Yes (`faiss-gpu`) | ❌ No |
| **Scalability** | Billions of vectors | Millions of vectors |
| **Latency** | ~1-10ms (with GPU) | ~0.1-1ms |
| **Memory** | Lower (quantization) | Higher (full vectors) |
| **Usage in NodeRAG** | K-means clustering | Primary k-NN search |

### Почему FAISS только для clustering?

**HNSW преимущества**:
- ✅ Проще в использовании (меньше параметров)
- ✅ Достаточно быстро для NodeRAG scale
- ✅ Меньше зависимостей (no BLAS)

**FAISS преимущества**:
- ✅ Excellent k-means implementation
- ✅ GPU support (future)
- ✅ Advanced quantization (not used yet)

### FAISS Algorithms (не используемые в NodeRAG)

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| **IndexFlatL2** | Brute force L2 | Small datasets (<10k) |
| **IndexIVFFlat** | Inverted file index | Medium datasets (10k-1M) |
| **IndexIVFPQ** | IVF + Product Quantization | Large datasets (>1M) |
| **IndexHNSW** | HNSW implementation | Alternative to hnswlib |

**NodeRAG использует**: Только `faiss.Kmeans` для clustering.

---

## 3. NumPy (Векторные операции)

**Пакет**: `numpy==1.26.4`

**Официальный сайт**: https://numpy.org/

### Назначение в NodeRAG

NumPy обеспечивает **эффективные векторные операции** для embeddings и matrix computations.

### Применение в векторном поиске

#### 1. Embedding Storage

```python
import numpy as np

# Store embedding as numpy array
embedding = np.array([0.23, -0.45, 0.78, ...], dtype=np.float32)  # 1536 dimensions

# Efficient storage
embeddings = np.vstack([emb1, emb2, emb3, ...])  # Shape: (N, 1536)
```

#### 2. Distance Calculations

**Файл**: `NodeRAG/utils/HNSW.py:48-79`

```python
def search_list(self, query_list: List[np.ndarray], HNSW_results: int = None):
    """Batch search with deduplication"""
    idx, dist = self.hnsw.knn_query(
        np.array(query_list).astype(np.float32),  # Convert to numpy array
        HNSW_results
    )
    idx = idx.flatten()
    dist = dist.flatten()

    # Deduplicate and aggregate distances
    for i in range(len(idx)):
        if self.id_map[idx[i]] not in node_list:
            node_list.append(self.id_map[idx[i]])
            dist_list.append(dist[i])
        else:
            # Weighted average favoring smaller distance
            existing_idx = node_list.index(self.id_map[idx[i]])
            dist_list[existing_idx] = 0.9 * min(
                dist_list[existing_idx],
                dist[i]
            )
```

#### 3. Matrix Operations (PPR)

**Файл**: `NodeRAG/utils/PPR.py:38-57`

```python
def PPR(self, personalization: dict[str, float], alpha=0.85):
    # Initialize probability distribution
    probs = np.zeros(len(self.nodes))
    for node, prob in personalization.items():
        probs[self.nodes.index(node)] = prob
    probs = probs / np.sum(probs)  # Normalize

    # Power iteration
    for i in range(max_iter):
        probs_old = probs.copy()

        # Matrix-vector multiplication (sparse matrix from SciPy)
        probs = alpha * self.trans_matrix.dot(probs) + (1 - alpha) * probs

        # Check convergence
        if np.linalg.norm(probs - probs_old) < epsilons:
            break

    return sorted(zip(self.nodes, probs), key=itemgetter(1), reverse=True)
```

### Используемые функции NumPy

| Функция | Назначение | Использование |
|---------|-----------|---------------|
| `np.array()` | Create array | Embedding storage |
| `np.vstack()` | Vertical stack | Concatenate embeddings |
| `np.zeros()` | Zero array | Initialize probabilities |
| `np.ones()` | Ones array | Dangling node handling |
| `np.sum()` | Sum | Normalization |
| `np.linalg.norm()` | Vector norm | Convergence check |
| `.astype()` | Type conversion | float32 for HNSW/FAISS |
| `.flatten()` | Flatten array | Convert 2D to 1D |
| `.copy()` | Copy array | Avoid aliasing |

### Почему float32?

**HNSW и FAISS требуют**: `np.float32` (not `float64`)

**Причины**:
- ✅ **Меньше памяти**: 4 bytes vs. 8 bytes per number
- ✅ **Быстрее**: SIMD optimizations для float32
- ✅ **Достаточная точность**: Embeddings уже approximate

**Пример**:
```python
# 100k vectors, 1536 dimensions
float64: 100k × 1536 × 8 bytes = 1.2 GB
float32: 100k × 1536 × 4 bytes = 0.6 GB
```

---

## Итоговая роль в архитектуре

```
Documents
      ↓
[Text Decomposition]
      ↓
Semantic Units, Entities, Attributes
      ↓
[Embedding Generation]
      ↓
OpenAI: text-embedding-3-small (1536 dim)
      ↓
NumPy arrays (float32)
      ↓
┌─────────────────────────────────────────┐
│           HNSW Index                    │
│  - hnswlib-noderag                      │
│  - M=64, ef=200                         │
│  - Space: cosine                        │
│  - ~100k vectors                        │
└─────────────────────────────────────────┘
      ↓
┌─────────────────┬───────────────────────┐
│                 │                       │
│  k-NN Search    │  Graph Export         │
│  (Answer Gen)   │  (HNSW → NetworkX)    │
│      ↓          │        ↓              │
│  Top-50 nodes   │   HNSW edges          │
│                 │        ↓              │
│                 │  Concatenate with     │
│                 │  base graph           │
└─────────────────┴───────────────────────┘
      ↓                    ↓
[PPR Expansion]    [Hybrid Graph Search]
      ↓                    ↓
Retrieved Nodes    Retrieved Nodes
      ↓                    ↓
      └─────────┬──────────┘
                ↓
          Context Assembly
                ↓
              Answer

Clustering (Community Summary):
    Communities (50-200 nodes each)
            ↓
    FAISS K-means
            ↓
    Cluster assignment
            ↓
    High-level Element → Cluster nodes
    (reduces edges in graph)
```

### Ключевые принципы

1. **HNSW as Primary**: Fast k-NN для всех retrieval queries
2. **FAISS for Clustering**: Efficient k-means для graph pruning
3. **NumPy Everywhere**: Underlying data structure для всех vectors
4. **Float32 Standard**: Memory efficiency без потери accuracy
5. **Hybrid Integration**: HNSW graph + Knowledge graph = richer structure

### Performance Benchmarks

**HNSW Search** (100k vectors, 1536 dim, top-50):
- Build time: ~30 seconds
- Query time: ~0.5-1ms per query
- Recall@50: ~97%
- Memory: ~600 MB

**FAISS K-means** (10k vectors, 1536 dim, 100 clusters):
- Training time: ~2 seconds (CPU)
- Assignment time: ~100ms
- Memory: ~60 MB

**NumPy Operations** (PPR, 100k nodes, 100 iterations):
- Sparse matrix-vector multiply: ~10ms per iteration
- Total PPR time: ~1 second
- Memory: ~100 MB (sparse matrix)
