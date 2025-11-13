# Graph Patterns

## Обзор

Этот документ описывает паттерны работы с графами знаний, используемые в NodeRAG для построения, анализа и поиска информации в knowledge graph.

---

## 1. Graph Accumulation Pattern

### Назначение
Инкрементальное построение графа знаний с агрегацией весов при дублировании узлов и рёбер.

### Реализация

**Файл**: `NodeRAG/build/pipeline/graph_pipeline.py:107-143`

```python
class Graph_pipeline:
    def add_semantic_unit(self, semantic_unit: Dict, text_hash_id: str):
        """Add semantic unit node with weight accumulation"""
        semantic_unit = Semantic_unit(semantic_unit, text_hash_id)

        if self.G.has_node(semantic_unit.hash_id):
            # Узел уже существует → увеличить вес
            self.G.nodes[semantic_unit.hash_id]['weight'] += 1
        else:
            # Новый узел
            self.G.add_node(semantic_unit.hash_id, type='semantic_unit', weight=1)
            self.semantic_units.append(semantic_unit)

        return semantic_unit.hash_id

    def add_entities(self, entities: List[Dict], text_hash_id: str):
        """Add entity nodes with weight accumulation"""
        entities_hash_id = []

        for entity in entities:
            entity = Entity(entity, text_hash_id)
            entities_hash_id.append(entity.hash_id)

            if self.G.has_node(entity.hash_id):
                # Увеличить вес существующего узла
                self.G.nodes[entity.hash_id]['weight'] += 1
            else:
                # Добавить новый узел
                self.G.add_node(entity.hash_id, type='entity', weight=1)
                self.entities.append(entity)

        return entities_hash_id

    def add_semantic_belongings(self, semantic_unit_hash_id: str, hash_id: List[str]):
        """Add edges between semantic units and entities"""
        for entity_hash_id in hash_id:
            if self.G.has_edge(semantic_unit_hash_id, entity_hash_id):
                # Ребро существует → увеличить вес
                self.G[semantic_unit_hash_id][entity_hash_id]['weight'] += 1
            else:
                # Добавить новое ребро
                self.G.add_edge(semantic_unit_hash_id, entity_hash_id, weight=1)
```

### Ключевые характеристики

1. **Weight as Frequency**: Вес узла = количество упоминаний в документах
2. **Idempotent Operations**: Повторное добавление не создаёт дубликаты
3. **Hash-based Identity**: Узлы идентифицируются по hash_id (SHA256 от контента)
4. **Edge Weight Accumulation**: Вес ребра показывает силу связи

### Пример

```python
# Документ 1: "Dr. Roberts works at MIT"
add_entity("DR. ROBERTS")  # weight=1
add_entity("MIT")           # weight=1
add_edge("DR. ROBERTS", "MIT")  # weight=1

# Документ 2: "Dr. Roberts is a professor at MIT"
add_entity("DR. ROBERTS")  # weight=2 (увеличение)
add_entity("MIT")           # weight=2 (увеличение)
add_edge("DR. ROBERTS", "MIT")  # weight=2 (увеличение)
```

---

## 2. Graph Concatenation Pattern

### Назначение
Объединение нескольких графов в один с правильной агрегацией весов и атрибутов.

### Реализация

**Файл**: `NodeRAG/utils/graph_operator.py:1-48`

```python
import networkx as nx

def concatenate_graphs(graphs: list[nx.Graph]) -> nx.Graph:
    """Concatenate multiple graphs into one"""
    if not graphs:
        return nx.Graph()

    G = graphs[0].copy()

    for graph in graphs[1:]:
        # Добавить все узлы
        for node, data in graph.nodes(data=True):
            if G.has_node(node):
                # Узел существует → объединить веса
                G.nodes[node]['weight'] = max(
                    G.nodes[node].get('weight', 1),
                    data.get('weight', 1)
                )
            else:
                # Новый узел
                G.add_node(node, **data)

        # Добавить все рёбра
        for u, v, data in graph.edges(data=True):
            if G.has_edge(u, v):
                # Ребро существует → суммировать веса
                G[u][v]['weight'] += data.get('weight', 1)
            else:
                # Новое ребро
                G.add_edge(u, v, **data)

    return G
```

### Weight Adjustment

```python
def adjust_weights(graph: nx.Graph, weight_threshold: int = 1):
    """Remove low-weight nodes and edges"""
    # Удалить узлы с низким весом
    nodes_to_remove = [
        node for node, data in graph.nodes(data=True)
        if data.get('weight', 1) < weight_threshold
    ]
    graph.remove_nodes_from(nodes_to_remove)

    # Удалить рёбра с низким весом
    edges_to_remove = [
        (u, v) for u, v, data in graph.edges(data=True)
        if data.get('weight', 1) < weight_threshold
    ]
    graph.remove_edges_from(edges_to_remove)

    return graph
```

### Использование

```python
# Объединение графов из нескольких документов
graphs = [doc1_graph, doc2_graph, doc3_graph]
combined = concatenate_graphs(graphs)

# Фильтрация по весу
filtered = adjust_weights(combined, weight_threshold=2)
```

---

## 3. HNSW Index Pattern

### Назначение
Approximate Nearest Neighbor (ANN) search с использованием Hierarchical Navigable Small World graphs для быстрого поиска похожих embeddings.

### Реализация

**Файл**: `NodeRAG/utils/HNSW.py:11-116`

```python
import hnswlib_noderag
import networkx as nx
import numpy as np

class HNSW:
    """Wrapper around hnswlib with node ID mapping"""

    def __init__(self, config):
        self.config = config
        self.id_map = self.load_id_map()  # node_id -> hash_id mapping
        self.load_HNSW()
        self._nxgraphs = None

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

    def add_nodes(self, nodes: List[Tuple[str, np.ndarray]]):
        """Add nodes with their embeddings to HNSW index"""
        current_length = len(self.id_map)
        id_list = []
        embedding_list = []

        for idx, (node_id, embedding) in enumerate(nodes):
            new_id = current_length + idx
            self.id_map[new_id] = node_id
            id_list.append(new_id)
            embedding_list.append(embedding)

        # Resize index and add items
        self.hnsw.resize_index(len(id_list) + current_length)
        self.hnsw.add_items(
            np.array(embedding_list).astype(np.float32),
            id_list
        )

    def search(self, query: np.ndarray, HNSW_results: int = None):
        """Search for k-nearest neighbors"""
        if HNSW_results is None:
            HNSW_results = self.config.top_k

        # KNN query
        idx, dist = self.hnsw.knn_query(query, HNSW_results)
        idx = idx.flatten()
        dist = dist.flatten()

        # Map internal IDs to node IDs
        node_list = [self.id_map[idx[i]] for i in range(len(idx))]
        dist_list = list(dist)

        results = zip(dist_list, node_list)
        return results

    def search_list(self, query_list: List[np.ndarray], HNSW_results: int = None):
        """Batch search with deduplication"""
        if HNSW_results is None:
            HNSW_results = self.config.top_k

        idx, dist = self.hnsw.knn_query(
            np.array(query_list).astype(np.float32),
            HNSW_results
        )
        idx = idx.flatten()
        dist = dist.flatten()

        node_list = []
        dist_list = []

        # Deduplicate and aggregate distances
        for i in range(len(idx)):
            if self.id_map[idx[i]] not in node_list:
                node_list.append(self.id_map[idx[i]])
                dist_list.append(dist[i])
            else:
                # Update distance (weighted average favoring smaller distance)
                existing_idx = node_list.index(self.id_map[idx[i]])
                dist_list[existing_idx] = 0.9 * min(
                    dist_list[existing_idx],
                    dist[i]
                )

        results = zip(dist_list, node_list)
        return nsmallest(HNSW_results, results)

    def load_HNSW(self):
        """Load or initialize HNSW index"""
        self.hnsw = hnswlib_noderag.Index(
            space=self.config.space,
            dim=self.config.dim
        )

        if os.path.exists(self.config.HNSW_path):
            # Load existing index
            self.hnsw.load_index(self.config.HNSW_path)
        else:
            # Initialize new index
            self.hnsw.init_index(
                max_elements=len(self.id_map),
                ef_construction=self.config._ef,
                M=self.config._m
            )
```

### Параметры HNSW

**Конфигурация**:
```python
{
    "space": "cosine",       # Distance metric (cosine/l2/ip)
    "dim": 1536,             # Embedding dimension
    "_ef": 200,              # ef_construction (quality vs speed tradeoff)
    "_m": 64,                # M (max number of connections per node)
    "top_k": 50              # Number of results to return
}
```

**Метрики**:
- **Search Time**: ~0.1-1ms per query (зависит от размера графа)
- **Recall**: ~95-99% (с правильными параметрами)
- **Index Size**: ~300 bytes per vector (для cosine, dim=1536)

### Преимущества

1. **Fast Search**: O(log N) вместо O(N) для brute force
2. **Scalable**: Эффективен для миллионов векторов
3. **Incremental Updates**: Можно добавлять узлы без полной перестройки
4. **NetworkX Integration**: Конвертация в NetworkX для дальнейшего анализа

---

## 4. Personalized PageRank (PPR) Pattern

### Назначение
Ranking узлов графа на основе персонализированного PageRank для идентификации наиболее релевантных узлов относительно seed nodes.

### Реализация

**Файл**: `NodeRAG/utils/PPR.py:6-73`

```python
import networkx as nx
import numpy as np
import scipy.sparse as sp
from operator import itemgetter

class sparse_PPR():
    """Sparse matrix implementation of Personalized PageRank"""

    def __init__(self, graph: nx.Graph, modified=True, weight='weight'):
        self.graph = graph
        self.nodes = list(self.graph.nodes())
        self.modified = modified
        self.weight = weight
        self.n_nodes = len(self.nodes)
        self.trans_matrix = self.generate_sparse_trasition_matrix()

    def generate_sparse_trasition_matrix(self):
        """Generate sparse transition matrix from graph"""
        # Get adjacency matrix
        adjaceny_matrix = nx.adjacency_matrix(self.graph, weight=self.weight)
        adjaceny_matrix = (adjaceny_matrix + adjaceny_matrix.T) / 2

        if self.modified:
            # Handle nodes with zero out-degree (dangling nodes)
            out_degree = adjaceny_matrix.sum(1)
            adjaceny_matrix = sp.lil_matrix(adjaceny_matrix)

            # Dangling nodes connect to all nodes
            adjaceny_matrix[out_degree == 0, :] = np.ones(self.n_nodes)

            # Remove self-loops
            adjaceny_matrix.setdiag(0)
            adjaceny_matrix = sp.csc_matrix(adjaceny_matrix)
            out_degree = adjaceny_matrix.sum(1)

        # Normalize to get transition probabilities
        tansition_matrix = adjaceny_matrix.multiply(1 / out_degree)

        # Transpose: out_matrix transpose is in_matrix
        tansition_matrix = tansition_matrix.T

        return sp.csc_matrix(tansition_matrix)

    def PPR(self,
            perosnalization: dict[str, float],
            alpha: float = 0.85,
            max_iter: int = 100,
            epsilons: float = 1e-5):
        """
        Compute Personalized PageRank

        Args:
            perosnalization: Initial probability distribution {node: prob}
            alpha: Damping factor (0.85 = 85% follow edges, 15% teleport)
            max_iter: Maximum iterations
            epsilons: Convergence threshold

        Returns:
            List of (node, score) sorted by score descending
        """
        # Initialize probability distribution
        probs = np.zeros(len(self.nodes))

        for node, prob in perosnalization.items():
            probs[self.nodes.index(node)] = prob

        # Normalize
        probs = probs / np.sum(probs)

        # Power iteration
        for i in range(max_iter):
            probs_old = probs.copy()

            # PPR formula: probs = alpha * M * probs + (1-alpha) * personalization
            probs = alpha * self.trans_matrix.dot(probs) + (1 - alpha) * probs

            # Check convergence
            if np.linalg.norm(probs - probs_old) < epsilons:
                break

        return sorted(zip(self.nodes, probs), key=itemgetter(1), reverse=True)

    def PR(self,
           alpha: float = 0.1,
           max_iter: int = 100,
           epsilons: float = 1e-5):
        """Standard PageRank (uniform personalization)"""
        probs = np.ones(self.n_nodes) / self.n_nodes

        for i in range(max_iter):
            probs_old = probs.copy()
            probs = alpha * self.trans_matrix.dot(probs) + (1 - alpha) * probs

            if np.linalg.norm(probs - probs_old) < epsilons:
                break

        return sorted(zip(self.nodes, probs), key=itemgetter(1), reverse=True)
```

### Использование в Search

**Файл**: `NodeRAG/search/search.py` (referenced)

```python
# 1. HNSW retrieval → initial candidates
hnsw_results = hnsw.search(query_embedding, top_k=50)
seed_nodes = [node for dist, node in hnsw_results]

# 2. PPR для расширения контекста
personalization = {node: 1.0 / len(seed_nodes) for node in seed_nodes}
ppr_ranker = sparse_PPR(graph)
ppr_results = ppr_ranker.PPR(personalization, alpha=0.85)

# 3. Топ узлы по PPR score
top_nodes = ppr_results[:100]
```

### Преимущества

1. **Context Expansion**: Находит релевантные узлы не только среди seed nodes
2. **Graph Structure Awareness**: Учитывает структуру графа, не только embeddings
3. **Personalized Ranking**: Приоритизация относительно query-specific seeds
4. **Efficient Sparse Implementation**: O(E) на итерацию, где E = количество рёбер

---

## 5. K-Core Decomposition Pattern

### Назначение
Идентификация "важных" узлов через k-core decomposition и betweenness centrality для selective attribute generation.

### Реализация

**Файл**: `NodeRAG/build/pipeline/attribute_generation.py:20-64`

```python
class NodeImportance:
    """Identify important nodes in the graph"""

    def __init__(self, graph: nx.Graph, console: Console):
        self.G = graph
        self.important_nodes = []
        self.console = console

    def K_core(self, k: int | None = None):
        """
        K-core decomposition: subgraph where all nodes have degree >= k

        Узлы в k-core более "central" и connected
        """
        if k is None:
            k = self.defult_k()

        # Get k-core subgraph
        self.k_subgraph = nx.core.k_core(self.G, k=k)

        # Select only entities with weight > 1
        for nodes in self.k_subgraph.nodes():
            if self.G.nodes[nodes]['type'] == 'entity' and self.G.nodes[nodes]['weight'] > 1:
                self.important_nodes.append(nodes)

    def avarege_degree(self):
        """Calculate average degree of graph"""
        average_degree = sum(dict(self.G.degree()).values()) / self.G.number_of_nodes()
        return average_degree

    def defult_k(self):
        """Heuristic for choosing k"""
        k = round(np.log(self.G.number_of_nodes()) * self.avarege_degree() ** (1/2))
        return k

    def betweenness_centrality(self):
        """
        Betweenness centrality: measures how often a node lies on shortest paths

        High betweenness → important "bridge" node
        """
        # Sample 10 random nodes for approximation (faster)
        self.betweenness = nx.betweenness_centrality(self.G, k=10)

        average_betweenness = sum(self.betweenness.values()) / len(self.betweenness)
        scale = round(math.log10(len(self.betweenness)))

        for node in self.betweenness:
            if self.betweenness[node] > average_betweenness * scale:
                if self.G.nodes[node]['type'] == 'entity' and self.G.nodes[node]['weight'] > 1:
                    self.important_nodes.append(node)

    def main(self):
        """Combine K-core and betweenness"""
        self.K_core()
        self.console.print('[bold green]K_core done[/bold green]')

        self.betweenness_centrality()
        self.console.print('[bold green]Betweenness done[/bold green]')

        # Union of both methods
        self.important_nodes = list(set(self.important_nodes))
        return self.important_nodes
```

### Критерии Important Nodes

**Узел считается "important" если**:
1. **В k-core subgraph**: `core_number(node) >= k`
2. **High betweenness**: `betweenness(node) > average * log10(N)`
3. **Is entity**: `type == 'entity'`
4. **Multiple mentions**: `weight > 1`

**Результат**: 10-15% entities выбираются для attribute generation

### Использование

```python
# Identify important nodes
node_importance = NodeImportance(graph, console)
important_nodes = node_importance.main()

# Generate attributes only for important nodes
for node in important_nodes:
    attribute = await generate_attribute(node)
    graph.add_node(attribute.hash_id, type='attribute')
    graph.add_edge(node, attribute.hash_id)
```

---

## 6. Community Detection Pattern (Leiden Algorithm)

### Назначение
Обнаружение сообществ (кластеров) в графе для последующей генерации high-level summaries.

### Реализация

**Файл**: `NodeRAG/build/pipeline/summary_generation.py:49-56`

```python
import leidenalg as la
from ...utils import IGraph

class SummaryGeneration:
    def partition(self):
        """Detect communities using Leiden algorithm"""
        # Convert NetworkX graph to igraph
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

### Leiden Algorithm Properties

**Преимущества над Louvain**:
- Гарантирует well-connected communities
- Быстрее для больших графов
- Более стабильные результаты

**Modularity Optimization**:
```
Q = (1/2m) * Σ[A_ij - (k_i * k_j)/(2m)] * δ(c_i, c_j)

где:
- A_ij = adjacency matrix
- k_i = degree of node i
- m = total edges
- δ(c_i, c_j) = 1 if nodes i,j in same community, else 0
```

### Использование

```python
# 1. Detect communities
summary_gen = SummaryGeneration(config)
summary_gen.partition()

# 2. Generate summary for each community
for community in summary_gen.communities:
    summary = await community.generate_community_summary()

# 3. Create high-level element nodes
for he in high_level_elements:
    graph.add_node(he.hash_id, type='high_level_element')
```

---

## 7. Clustering-based Graph Pruning Pattern

### Назначение
Использование k-means clustering для интеллектуального сокращения связей между high-level elements и semantic units.

### Реализация

**Файл**: `NodeRAG/build/pipeline/summary_generation.py:150-185`

```python
import faiss
import numpy as np

class SummaryGeneration:
    async def high_level_element_summary(self):
        """Connect high-level elements to semantic units via clustering"""
        # ...

        # Calculate number of clusters
        centroids = math.ceil(math.sqrt(len(All_nodes) + len(self.high_level_elements)))
        threshold = (len(All_nodes) + len(self.high_level_elements)) / centroids

        # If cluster size would be too large, use k-means
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
            kmeans = faiss.Kmeans(d=all_embeddings.shape[1], k=centroids)
            kmeans.train(all_embeddings.astype(np.float32))
            _, cluster_labels = kmeans.assign(all_embeddings.astype(np.float32))

            # Split labels
            high_level_element_cluster_labels = cluster_labels[:len(self.high_level_elements)]
            embedding_cluster_labels = cluster_labels[len(self.high_level_elements):]

            self.config.console.print(f'[bold green]KMeans Clustering with {centroids} centroids[/bold green]')

            # Connect nodes in same cluster
            self.config.tracker.set(len(self.high_level_elements), 'Adding High Level Element Summary')
            for i in range(len(self.high_level_elements)):
                for j in range(len(All_nodes)):
                    if (high_level_element_cluster_labels[i] == embedding_cluster_labels[j] and
                        All_nodes[j] in self.high_level_elements[i].related_node):
                        # Same cluster and related → add edge
                        self.G.add_edge(All_nodes[j], self.high_level_elements[i].hash_id, weight=1)
                        n += 1
                self.config.tracker.update()
        else:
            # Connect all related nodes directly (no clustering needed)
            self.config.tracker.set(len(self.high_level_elements), 'Adding High Level Element Summary')
            for he in self.high_level_elements:
                for node in he.related_node:
                    self.G.add_edge(node, he.hash_id, weight=1)
                    n += 1
                self.config.tracker.update()
```

### Strategy

**Condition**: Если `avg_cluster_size > Hcluster_size` (например, 100)

**Without Clustering**:
```
High-level element → ALL related semantic units (может быть 1000+)
```

**With Clustering**:
```
High-level element → semantic units in SAME cluster (обычно 50-200)
```

**Benefits**:
1. **Reduced Graph Size**: Меньше рёбер без потери информации
2. **Semantic Coherence**: Кластеры семантически связанных узлов
3. **Retrieval Efficiency**: Меньше узлов для traversal

### Hyperparameters

```python
centroids = ceil(sqrt(n_nodes))  # Количество кластеров
Hcluster_size = 100              # Порог для активации clustering
```

---

## 8. Incremental Graph Update Pattern

### Назначение
Эффективное обновление графа при добавлении новых документов без полной перестройки.

### Реализация

**Файл**: `NodeRAG/build/pipeline/graph_pipeline.py:37-46`

```python
class Graph_pipeline:
    def check_processed(self, data: Dict) -> bool:
        """Check if text unit is already processed"""
        if data.get('processed'):
            return False
        return True

    def load_data(self) -> List[LLMOutput]:
        """Load only unprocessed data"""
        data_list = []
        processed_data = []

        with open(self.config.text_decomposition_path, 'r', encoding='utf-8') as f:
            for line in f:
                data = json.loads(line)
                if self.check_processed(data):
                    # Not processed yet
                    data_list.append(data)
                else:
                    # Already processed
                    processed_data.append(data)

        return data_list, processed_data
```

**Процесс**:

```
1. Load existing graph from disk
   ↓
2. Load text decomposition results
   ↓
3. Filter: keep only unprocessed (processed=False)
   ↓
4. Build graph from unprocessed data
   ↓
5. Merge with existing graph (weight accumulation)
   ↓
6. Mark processed texts (processed=True)
   ↓
7. Save updated graph
```

### Преимущества

1. **Efficient Updates**: Обрабатываем только новые данные
2. **No Data Loss**: Существующий граф сохраняется
3. **Weight Accumulation**: Новые mentions увеличивают веса
4. **Atomic Operations**: Каждый text unit помечается как processed

---

## 9. Mapper Pattern для Graph Data Access

### Назначение
Unified interface для доступа к данным узлов графа из различных источников (Parquet files, embeddings).

### Реализация

**Файл**: `NodeRAG/storage/storage.py` (referenced)

```python
from typing import Dict, List
import pandas as pd

class Mapper:
    """Unified data access layer for graph nodes"""

    def __init__(self, parquet_paths: List[str]):
        self.data = {}
        self.embeddings = None

        # Load all parquet files
        for path in parquet_paths:
            if os.path.exists(path):
                df = pd.read_parquet(path)
                for idx, row in df.iterrows():
                    self.data[row['hash_id']] = row.to_dict()

    def add_embedding(self, embedding_path: str):
        """Load embeddings separately"""
        if os.path.exists(embedding_path):
            df = pd.read_parquet(embedding_path)
            self.embeddings = {row['hash_id']: row['embedding'] for _, row in df.iterrows()}

    def get(self, node_id: str, attribute: str):
        """Get attribute of node"""
        if node_id in self.data:
            return self.data[node_id].get(attribute)
        return None

    def find_non_HNSW(self):
        """Find nodes without HNSW embedding"""
        if self.embeddings is None:
            return []

        non_hnsw = []
        for hash_id, embedding in self.embeddings.items():
            if self.data[hash_id].get('embedding') != 'HNSW':
                non_hnsw.append((hash_id, embedding))

        return non_hnsw
```

### Использование

```python
# Initialize mapper with multiple data sources
mapper = Mapper([
    config.semantic_units_path,
    config.entities_path,
    config.attributes_path
])

# Add embeddings
mapper.add_embedding(config.embedding_path)

# Access node data
context = mapper.get(node_id, 'context')
weight = mapper.get(node_id, 'weight')
embedding = mapper.embeddings[node_id]
```

### Преимущества

1. **Separation of Concerns**: Граф хранит только структуру, данные — отдельно
2. **Memory Efficient**: Не загружаем все данные в граф
3. **Flexible Data Sources**: Можно добавлять новые источники данных
4. **Lazy Loading**: Данные загружаются только при необходимости

---

## 10. Graph Persistence Pattern

### Назначение
Эффективное сохранение и загрузка графов с использованием Pickle и Parquet.

### Реализация

**Файл**: `NodeRAG/storage/storage.py` (referenced)

```python
import pickle
import pandas as pd
import networkx as nx

class storage:
    """Unified storage interface"""

    def __init__(self, data):
        self.data = data

    def save_pickle(self, path: str):
        """Save graph as pickle"""
        with open(path, 'wb') as f:
            pickle.dump(self.data, f)

    @staticmethod
    def load_pickle(path: str) -> nx.Graph:
        """Load graph from pickle"""
        with open(path, 'rb') as f:
            return pickle.load(f)

    def save_parquet(self, path: str, append: bool = False):
        """Save node/edge data as Parquet"""
        df = pd.DataFrame(self.data)

        if append and os.path.exists(path):
            existing = pd.read_parquet(path)
            df = pd.concat([existing, df], ignore_index=True)

        df.to_parquet(path, index=False)

    @staticmethod
    def load_parquet(path: str) -> pd.DataFrame:
        """Load node/edge data from Parquet"""
        return pd.read_parquet(path)
```

### Storage Strategy

**Graph Structure**: Pickle
```python
# Сохраняет весь NetworkX граф (узлы, рёбра, атрибуты)
storage(graph).save_pickle('graph.pkl')

# Быстрая загрузка
graph = storage.load_pickle('graph.pkl')
```

**Node Data**: Parquet
```python
# Сохраняет node attributes в columnar format
nodes = [
    {'hash_id': 'abc123', 'context': '...', 'weight': 5},
    {'hash_id': 'def456', 'context': '...', 'weight': 3}
]
storage(nodes).save_parquet('nodes.parquet')

# Efficient querying с Pandas/Polars
df = storage.load_parquet('nodes.parquet')
high_weight = df[df['weight'] > 3]
```

### Преимущества

1. **Pickle для Graphs**: Быстрая сериализация структуры графа
2. **Parquet для Data**: Эффективное хранение больших dataframes
3. **Compression**: Parquet автоматически сжимает данные
4. **Incremental Append**: Поддержка append mode для Parquet

---

## 11. Weighted Graph Traversal Pattern

### Назначение
Traversal графа с учётом весов для сбора контекста важных узлов.

### Реализация

**Файл**: `NodeRAG/build/pipeline/attribute_generation.py:98-136`

```python
class Attribution_generation_pipeline:
    def get_neighbours_material(self, node: str):
        """Collect all neighbor context"""
        entity = self.mapper.get(node, 'context')
        semantic_neighbours = '' + '\n'
        relationship_neighbours = '' + '\n'

        # Traverse all neighbors
        for neighbour in self.G.neighbors(node):
            if self.G.nodes[neighbour]['type'] == 'semantic_unit':
                semantic_neighbours += f'{self.mapper.get(neighbour, "context")}\n'
            elif self.G.nodes[neighbour]['type'] == 'relationship':
                relationship_neighbours += f'{self.mapper.get(neighbour, "context")}\n'

        query = self.prompt_manager.attribute_generation.format(
            entity=entity,
            semantic_units=semantic_neighbours,
            relationships=relationship_neighbours
        )
        return query

    def get_important_neibours_material(self, node: str):
        """Collect only important neighbor context (when token limit exceeded)"""
        entity = self.mapper.get(node, 'context')
        semantic_neighbours = '' + '\n'
        relationship_neighbours = '' + '\n'
        sorted_neighbours = SortedDict()

        # Calculate importance: sum of neighbor weights
        for neighbour in self.G.neighbors(node):
            value = 0
            for neighbour_neighbour in self.G.neighbors(neighbour):
                value += self.G.nodes[neighbour_neighbour]['weight']
            sorted_neighbours[neighbour] = value

        query = ''
        # Add neighbors in descending order of importance until token limit
        for neighbour in reversed(sorted_neighbours):
            while not self.token_counter.token_limit(query):
                query = self.prompt_manager.attribute_generation.format(
                    entity=entity,
                    semantic_units=semantic_neighbours,
                    relationships=relationship_neighbours
                )
                if self.G.nodes[neighbour]['type'] == 'semantic_unit':
                    semantic_neighbours += f'{self.mapper.get(neighbour, "context")}\n'
                elif self.G.nodes[neighbour]['type'] == 'relationship':
                    relationship_neighbours += f'{self.mapper.get(neighbour, "context")}\n'

        return query
```

### Importance Metric

```python
importance(node) = Σ weight(neighbor_of_neighbor)
                   for all neighbors of node
```

**Rationale**: Узел важен, если его соседи связаны с другими важными узлами

### Fallback Strategy

```
1. Try full context (all neighbors)
   ↓
2. Check token limit
   ↓
3. If exceeded → use only important neighbors
   ↓
4. Sort by importance
   ↓
5. Add incrementally until token limit
```

---

## Заключение

Graph patterns в NodeRAG обеспечивают:

- ✅ **Efficient Construction**: Incremental updates, weight accumulation
- ✅ **Fast Search**: HNSW для ANN, PPR для context expansion
- ✅ **Intelligent Selection**: K-core + betweenness для важных узлов
- ✅ **Community Detection**: Leiden algorithm для high-level summaries
- ✅ **Scalable Storage**: Pickle для структуры, Parquet для данных
- ✅ **Smart Traversal**: Weight-based prioritization для context collection
- ✅ **Clustering Optimization**: K-means для graph pruning

Эти паттерны делают knowledge graph масштабируемым, эффективным и точным инструментом для RAG.
