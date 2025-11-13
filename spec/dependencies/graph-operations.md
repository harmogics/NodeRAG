# Графовые операции

## Обзор

Эта категория включает библиотеки для построения, анализа и обработки knowledge graph — центрального компонента NodeRAG.

---

## 1. NetworkX

**Пакет**: `networkx==3.4.2`

**Официальный сайт**: https://networkx.org/

### Назначение в NodeRAG

NetworkX — это **основная библиотека** для представления и манипуляции knowledge graph в NodeRAG.

### Применение

**Файл**: `NodeRAG/build/pipeline/graph_pipeline.py:1-270`

```python
import networkx as nx

class Graph_pipeline:
    def load_graph(self) -> nx.Graph:
        if os.path.exists(self.config.graph_path):
            return storage.load_pickle(self.config.graph_path)
        return nx.Graph()

    def add_semantic_unit(self, semantic_unit: Dict, text_hash_id: str):
        semantic_unit = Semantic_unit(semantic_unit, text_hash_id)

        if self.G.has_node(semantic_unit.hash_id):
            # Weight accumulation
            self.G.nodes[semantic_unit.hash_id]['weight'] += 1
        else:
            # New node
            self.G.add_node(semantic_unit.hash_id, type='semantic_unit', weight=1)
            self.semantic_units.append(semantic_unit)

        return semantic_unit.hash_id

    def add_relationships(self, relationships: List[str], text_hash_id: str):
        for relationship in relationships:
            # Create relationship node
            relationship = Relationship(relationship, text_hash_id)

            # Add nodes and edges
            for node in [relationship.source, relationship.target, relationship]:
                if not self.G.has_node(node.hash_id):
                    self.G.add_node(
                        node.hash_id,
                        type='entity' if node in [relationship.source, relationship.target] else 'relationship',
                        weight=1
                    )

            # Add edges: source → relationship → target
            for edge in [(relationship.source.hash_id, relationship.hash_id),
                         (relationship.hash_id, relationship.target.hash_id)]:
                if not self.G.has_edge(*edge):
                    self.G.add_edge(*edge, weight=1)
                else:
                    self.G[edge[0]][edge[1]]['weight'] += 1
```

### Структура графа

#### Типы узлов

| Node Type | Описание | Атрибуты |
|-----------|----------|----------|
| `semantic_unit` | Атомарные факты | type, weight, context |
| `entity` | Именованные сущности | type, weight, context, attributes |
| `relationship` | Связи между entities | type, weight, context |
| `attribute` | Описания важных entities | type, weight, context |
| `high_level_element` | Тематические summaries | type, weight, context |
| `high_level_element_title` | Заголовки themes | type, weight, related_node |

#### Типы рёбер

```
Semantic Unit ←→ Entity (semantic belonging)
Entity ←→ Relationship ←→ Entity (relationship triplet)
Entity ←→ Attribute (attribute link)
Semantic Unit/Attribute ←→ High-level Element (community membership)
Node ←→ Node (HNSW neighbors)
```

### Используемые функции NetworkX

#### Построение графа

| Функция | Назначение | Использование |
|---------|-----------|---------------|
| `nx.Graph()` | Создание undirected graph | Graph initialization |
| `G.add_node(id, **attrs)` | Добавление узла с атрибутами | All node types |
| `G.add_edge(u, v, **attrs)` | Добавление ребра | All relationships |
| `G.has_node(id)` | Проверка существования узла | Weight accumulation |
| `G.has_edge(u, v)` | Проверка существования ребра | Weight accumulation |
| `G.nodes[id]` | Доступ к атрибутам узла | Read/write attributes |
| `G[u][v]` | Доступ к атрибутам ребра | Weight increment |

#### Анализ графа

**Файл**: `NodeRAG/build/pipeline/attribute_generation.py:27-64`

| Функция | Назначение | Использование |
|---------|-----------|---------------|
| `nx.core.k_core(G, k)` | K-core decomposition | Identify important nodes |
| `nx.betweenness_centrality(G)` | Betweenness centrality | Find bridge nodes |
| `G.degree()` | Node degrees | Average degree calculation |
| `G.neighbors(node)` | Get neighbors | Context collection |
| `G.number_of_nodes()` | Node count | Statistics |

#### Экспорт/Импорт

**Файл**: `NodeRAG/utils/graph_operator.py:1-48`

| Функция | Назначение | Использование |
|---------|-----------|---------------|
| `nx.adjacency_matrix(G)` | Export to sparse matrix | PPR calculation |

### Примеры использования

#### 1. K-Core Decomposition

```python
class NodeImportance:
    def K_core(self, k: int):
        """Find nodes in k-core (all nodes with degree >= k)"""
        self.k_subgraph = nx.core.k_core(self.G, k=k)

        for nodes in self.k_subgraph.nodes():
            if (self.G.nodes[nodes]['type'] == 'entity' and
                self.G.nodes[nodes]['weight'] > 1):
                self.important_nodes.append(nodes)
```

**Зачем**: Идентифицировать **центральные** entities для attribute generation.

#### 2. Betweenness Centrality

```python
def betweenness_centrality(self):
    """Find bridge nodes (nodes on many shortest paths)"""
    self.betweenness = nx.betweenness_centrality(self.G, k=10)
    average_betweenness = sum(self.betweenness.values()) / len(self.betweenness)

    for node in self.betweenness:
        if self.betweenness[node] > average_betweenness * scale:
            if (self.G.nodes[node]['type'] == 'entity' and
                self.G.nodes[node]['weight'] > 1):
                self.important_nodes.append(node)
```

**Зачем**: Найти **мостовые** entities, соединяющие разные части графа.

#### 3. Adjacency Matrix для PPR

```python
# Convert NetworkX graph to sparse adjacency matrix
adjacency_matrix = nx.adjacency_matrix(self.graph, weight='weight')
adjacency_matrix = (adjacency_matrix + adjacency_matrix.T) / 2  # Symmetrize
```

**Зачем**: Efficient PPR calculation через sparse matrix operations.

### Преимущества NetworkX

| Преимущество | Значение для NodeRAG |
|--------------|----------------------|
| **Pure Python** | Easy to install and debug |
| **Flexible attributes** | Store any metadata on nodes/edges |
| **Rich algorithms** | Centrality, clustering, etc. |
| **Visualization support** | Export to PyVis, Gephi |
| **Pickle-able** | Easy serialization |

### Недостатки и обходные пути

| Недостаток | Обходной путь |
|------------|--------------|
| **Slow for large graphs** (>100k nodes) | Use igraph for clustering |
| **No GPU support** | Use HNSW for vector search |
| **Memory intensive** | Store node data separately in Parquet |

---

## 2. iGraph + Leidenalg

**Пакеты**: `igraph==0.11.8`, `leidenalg==0.10.2`

**Официальные сайты**:
- igraph: https://igraph.org/
- leidenalg: https://leidenalg.readthedocs.io/

### Назначение в NodeRAG

iGraph используется исключительно для **community detection** через Leiden algorithm — это быстрее и точнее, чем NetworkX alternatives.

### Применение

**Файл**: `NodeRAG/build/pipeline/summary_generation.py:1-268`

```python
import leidenalg as la
from ...utils import IGraph

class SummaryGeneration:
    def partition(self):
        """Detect communities using Leiden algorithm"""
        # Convert NetworkX → igraph
        self.G_ig = IGraph(self.G).to_igraph()

        # Leiden community detection
        partition = la.find_partition(self.G_ig, la.ModularityVertexPartition)

        # Create Community_summary for each community
        for i, community in enumerate(partition):
            community_name = [
                self.G_ig.vs[node]['name']
                for node in community
                if self.G_ig.vs[node]['name'] in self.mapper.embeddings
            ]
            self.communities.append(
                Community_summary(community_name, self.mapper, self.G, self.config)
            )
```

### Конвертация NetworkX → iGraph

**Файл**: `NodeRAG/utils/IGraph.py` (referenced)

```python
class IGraph:
    def __init__(self, G: nx.Graph):
        self.G = G

    def to_igraph(self):
        """Convert NetworkX graph to igraph"""
        import igraph as ig

        # Create vertex list with names
        vertices = list(self.G.nodes())
        vertex_map = {v: i for i, v in enumerate(vertices)}

        # Create edge list
        edges = [(vertex_map[u], vertex_map[v]) for u, v in self.G.edges()]

        # Create igraph
        g = ig.Graph(n=len(vertices), edges=edges, directed=False)

        # Add vertex names
        g.vs['name'] = vertices

        # Add vertex attributes
        for v in vertices:
            for attr, value in self.G.nodes[v].items():
                if attr not in g.vs.attributes():
                    g.vs[attr] = None
                g.vs[vertex_map[v]][attr] = value

        # Add edge weights
        if self.G.edges():
            weights = [self.G[u][v].get('weight', 1) for u, v in self.G.edges()]
            g.es['weight'] = weights

        return g
```

### Leiden Algorithm

**Преимущества над Louvain**:
- ✅ Faster convergence
- ✅ Better-connected communities
- ✅ Deterministic results

**Файл**: `spec/research/star-pattern-architecture.md`

**Modularity Optimization**:
```
Q = (1/2m) * Σ [A_ij - (k_i * k_j)/(2m)] * δ(c_i, c_j)

где:
- A_ij = adjacency matrix
- k_i = degree of node i
- m = total edges
- δ(c_i, c_j) = 1 if nodes in same community
```

**Цель**: Максимизировать связи **внутри** communities, минимизировать **между** communities.

### Результат Community Detection

**Выход**:
```python
partition = [
    [node1, node2, node3, ...],  # Community 1
    [node10, node11, node12, ...],  # Community 2
    ...
]
```

**Использование**: Каждый community → LLM summarization → High-level theme

**Пример**:
```
Community 1 nodes: 50 semantic units о renewable energy
    ↓
LLM summarization
    ↓
High-level theme: "Advances in Solar Energy Technology"
```

### igraph vs. NetworkX

| Аспект | igraph | NetworkX |
|--------|--------|----------|
| **Performance** | Fast (C core) | Slower (Pure Python) |
| **Community Detection** | Excellent (Leiden, Infomap) | Basic (Louvain) |
| **Graph Construction** | More verbose | Pythonic |
| **Usage in NodeRAG** | Only for clustering | Everything else |

### Зависимости iGraph

| Зависимость | Назначение |
|-------------|-----------|
| `texttable==1.7.0` | Formatted output tables |

---

## 3. SciPy (Sparse Matrices)

**Пакет**: `scipy==1.12.0`

**Официальный сайт**: https://scipy.org/

### Назначение в NodeRAG

SciPy используется для **sparse matrix operations** в Personalized PageRank.

### Применение

**Файл**: `NodeRAG/utils/PPR.py:1-73`

```python
import scipy.sparse as sp
import numpy as np

class sparse_PPR():
    def __init__(self, graph: nx.Graph, modified=True, weight='weight'):
        self.graph = graph
        self.nodes = list(self.graph.nodes())
        self.trans_matrix = self.generate_sparse_trasition_matrix()

    def generate_sparse_trasition_matrix(self):
        """Generate sparse transition matrix from graph"""
        # Get adjacency matrix (sparse)
        adjaceny_matrix = nx.adjacency_matrix(self.graph, weight=self.weight)
        adjaceny_matrix = (adjaceny_matrix + adjaceny_matrix.T) / 2

        if self.modified:
            # Handle dangling nodes (degree = 0)
            out_degree = adjaceny_matrix.sum(1)
            adjaceny_matrix = sp.lil_matrix(adjaceny_matrix)
            adjaceny_matrix[out_degree == 0, :] = np.ones(self.n_nodes)
            adjaceny_matrix.setdiag(0)
            adjaceny_matrix = sp.csc_matrix(adjaceny_matrix)
            out_degree = adjaceny_matrix.sum(1)

        # Normalize to transition probabilities
        tansition_matrix = adjaceny_matrix.multiply(1 / out_degree)
        tansition_matrix = tansition_matrix.T  # Transpose

        return sp.csc_matrix(tansition_matrix)

    def PPR(self, personalization: dict[str, float], alpha=0.85, max_iter=100):
        """Compute Personalized PageRank via power iteration"""
        probs = np.zeros(len(self.nodes))
        for node, prob in personalization.items():
            probs[self.nodes.index(node)] = prob
        probs = probs / np.sum(probs)

        # Power iteration
        for i in range(max_iter):
            probs_old = probs.copy()
            # Sparse matrix-vector multiplication (O(E) instead of O(N²))
            probs = alpha * self.trans_matrix.dot(probs) + (1 - alpha) * probs

            if np.linalg.norm(probs - probs_old) < epsilons:
                break

        return sorted(zip(self.nodes, probs), key=itemgetter(1), reverse=True)
```

### Sparse Matrix Types

| Type | Описание | Использование |
|------|----------|---------------|
| `csr_matrix` | Compressed Sparse Row | Row operations |
| `csc_matrix` | Compressed Sparse Column | Column operations, matrix-vector multiply |
| `lil_matrix` | List of Lists | Incremental construction |

**В NodeRAG**:
- `lil_matrix` → construction (adding dangling node connections)
- `csc_matrix` → final format (efficient `.dot()` operation)

### Почему Sparse Matrices?

**Граф**: 100,000 nodes, 500,000 edges

**Dense matrix**:
```
Size: 100k × 100k = 10 billion elements
Memory: 10B × 8 bytes = 80 GB (float64)
```

**Sparse matrix (CSC)**:
```
Size: 500k edges × 2 (from/to) + overhead
Memory: ~4 MB
```

**Speedup**: Sparse matrix-vector multiply = O(E) вместо O(N²)

### Используемые функции SciPy

| Функция | Назначение |
|---------|-----------|
| `sp.csc_matrix()` | Create sparse matrix (CSC format) |
| `sp.lil_matrix()` | Create sparse matrix (LIL format) |
| `matrix.sum(axis)` | Sum along axis |
| `matrix.multiply(scalar)` | Element-wise multiply |
| `matrix.dot(vector)` | Matrix-vector product |
| `matrix.T` | Transpose |

### Интеграция с NetworkX

```python
# NetworkX → SciPy sparse
adjacency = nx.adjacency_matrix(graph, weight='weight')  # Returns scipy sparse matrix

# Process with SciPy
transition = adjacency.multiply(1 / adjacency.sum(1))

# Use in PPR
result = transition.dot(probabilities)
```

---

## 4. Graph Utilities

### SortedContainers

**Пакет**: `sortedcontainers==2.4.0`

**Официальный сайт**: https://grantjenks.com/docs/sortedcontainers/

**Применение**:

**Файл**: `NodeRAG/build/component/community.py:63-76`

```python
from sortedcontainers import SortedDict

def get_important_node_query(self):
    """Select most important neighbors when token limit exceeded"""
    weights_dict = SortedDict()

    # Calculate importance: sum of neighbor weights
    for name in self.used_unit:
        weight = 0
        for neighbour in self.graph.neighbors(name):
            weight += self.graph[neighbour]['weight']
        weights_dict[name] = weight

    # Iterate in descending order of importance
    weights_dict = reversed(weights_dict)
    for i in range(len(weights_dict) + 1):
        query = self.get_query(weights_dict.keys()[:i])
        if self.token_counter.token_limit(query):
            return query_old
        query_old = query
```

**Зачем**: Efficient sorting + iteration для priority-based selection.

### Graph Concatenation

**Файл**: `NodeRAG/utils/graph_operator.py:1-48`

```python
import networkx as nx

def concatenate_graphs(graphs: list[nx.Graph]) -> nx.Graph:
    """Merge multiple graphs with weight accumulation"""
    if not graphs:
        return nx.Graph()

    G = graphs[0].copy()

    for graph in graphs[1:]:
        # Add nodes
        for node, data in graph.nodes(data=True):
            if G.has_node(node):
                # Accumulate weight
                G.nodes[node]['weight'] = max(
                    G.nodes[node].get('weight', 1),
                    data.get('weight', 1)
                )
            else:
                G.add_node(node, **data)

        # Add edges
        for u, v, data in graph.edges(data=True):
            if G.has_edge(u, v):
                # Accumulate weight
                G[u][v]['weight'] += data.get('weight', 1)
            else:
                G.add_edge(u, v, **data)

    return G
```

**Использование**: Объединение base graph + HNSW graph.

---

## Итоговая роль в архитектуре

```
Documents
      ↓
[Text Decomposition]
      ↓
Semantic Units, Entities, Relationships
      ↓
┌──────────────────────────────────────┐
│       NetworkX Graph                 │
│                                      │
│  Nodes: SU, Entity, Rel, Attr, HLE   │
│  Edges: Semantic, Relationship       │
│  Attributes: type, weight, context   │
└──────────────────────────────────────┘
      ↓
┌─────────────────┬────────────────────┐
│                 │                    │
│ K-core +        │  Community         │
│ Betweenness     │  Detection         │
│ (NetworkX)      │  (igraph+Leiden)   │
│      ↓          │       ↓            │
│ Important       │  Communities       │
│ Entities        │  (50-200 nodes)    │
└─────────────────┴────────────────────┘
      ↓                   ↓
[Attribute Gen]    [LLM Summarization]
      ↓                   ↓
  Attributes      High-level Elements
      ↓                   ↓
      └───────┬───────────┘
              ↓
      Updated Graph
              ↓
  ┌───────────────────┐
  │  nx.adjacency_matrix
  │         ↓
  │  SciPy sparse matrix
  │         ↓
  │     PPR (search)
  └───────────────────┘
              ↓
        Retrieved nodes
              ↓
          Answer
```

### Ключевые принципы

1. **NetworkX as Primary**: All graph operations через NetworkX API
2. **igraph for Performance**: Only for community detection (faster)
3. **Sparse Matrices for Scale**: PPR через SciPy sparse для efficiency
4. **Weight Accumulation**: Frequency → importance
5. **Heterogeneous Graph**: Multiple node types, typed edges
