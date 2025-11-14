# Vector API Reference

## Введение

**Vector API** предоставляет доступ к векторному пространству NodeRAG (HNSW index + embeddings). API позволяет выполнять векторный поиск, получать embeddings, исследовать HNSW структуру.

**Base Path**: `/api/v1/vectors`

---

## HNSW Index Structure

### Характеристики

| Параметр | Значение | Описание |
|----------|----------|----------|
| **Dimension** | 1536 | Размерность embeddings (OpenAI text-embedding-3-small) |
| **Metric** | cosine | Метрика расстояния |
| **M** | 64 | Max connections per node |
| **ef_construction** | 200 | Construction parameter |
| **Layers** | ~log(N) | Количество слоёв HNSW |

### Indexed Nodes

Embeddings доступны для:
- `semantic_unit` - Все semantic units
- `entity` - Entities с embeddings
- `attribute` - Generated attributes
- `high_level_element` - Community summaries

---

## Endpoints

### 1. Vector Search

```http
POST /vectors/search
```

**Request Body**:
```json
{
  "vector": [0.123, -0.456, ...],
  "k": 50,
  "ef": 200,
  "filter": {
    "node_types": ["semantic_unit", "entity"],
    "weight_min": 1
  },
  "include_distances": true,
  "include_content": true
}
```

**Alternative**: Text query
```json
{
  "text": "renewable energy research",
  "k": 50
}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "query": {
      "type": "vector",
      "dimension": 1536,
      "k": 50
    },
    "results": [
      {
        "node_id": "sem_unit_123",
        "type": "semantic_unit",
        "distance": 0.234,
        "similarity": 0.766,
        "content": "Dr. Emily Roberts presented research on renewable energy...",
        "metadata": {
          "weight": 3,
          "degree": 15
        }
      },
      {
        "node_id": "entity_456",
        "type": "entity",
        "distance": 0.312,
        "similarity": 0.688,
        "content": "EMILY ROBERTS",
        "metadata": {
          "weight": 12,
          "degree": 45
        }
      }
    ]
  },
  "metadata": {
    "execution_time_ms": 8,
    "index_size": 150000,
    "hnsw_ef": 200
  }
}
```

**Implementation**:
```python
# NodeRAG/utils/HNSW.py
class HNSW:
    def search(self, query: np.ndarray, k: int = 50):
        """Vector search через HNSW index"""
        idx, dist = self.hnsw.knn_query(query, k)
        idx = idx.flatten()
        dist = dist.flatten()

        results = []
        for i in range(len(idx)):
            node_id = self.id_map[idx[i]]
            results.append({
                'node_id': node_id,
                'distance': float(dist[i]),
                'similarity': 1.0 - float(dist[i])  # cosine distance → similarity
            })

        return results
```

---

### 2. Embed Text

```http
POST /vectors/embed
```

**Request Body**:
```json
{
  "text": "renewable energy research",
  "model": "text-embedding-3-small"
}
```

**Batch**:
```json
{
  "texts": [
    "renewable energy research",
    "solar panel efficiency"
  ],
  "model": "text-embedding-3-small"
}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "embeddings": [
      {
        "text": "renewable energy research",
        "embedding": [0.123, -0.456, ...],
        "model": "text-embedding-3-small",
        "dimension": 1536,
        "norm": 1.0
      }
    ]
  },
  "metadata": {
    "model": "text-embedding-3-small",
    "tokens": 3,
    "cost_usd": 0.0000075
  }
}
```

**Implementation**:
```python
# NodeRAG/LLM/LLM.py
class OpenAI_Embedding:
    def _create_embedding(self, input):
        """Generate embeddings via OpenAI API"""
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=input
        )
        return [res.embedding for res in response.data]
```

---

### 3. Get Node Embedding

```http
GET /vectors/node/{node_id}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "node_id": "sem_unit_123",
    "embedding": {
      "vector": [0.123, -0.456, ...],
      "dimension": 1536,
      "norm": 1.0,
      "model": "text-embedding-3-small"
    },
    "node_info": {
      "type": "semantic_unit",
      "content": "Dr. Emily Roberts presented...",
      "weight": 3
    }
  }
}
```

---

### 4. Similar Nodes (HNSW)

```http
GET /vectors/similar/{node_id}
```

**Query Parameters**:
- `k` (integer, default: 20) - Количество похожих узлов
- `node_types` (string, optional) - Фильтр по типу (comma-separated)

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "node_id": "sem_unit_123",
    "similar_nodes": [
      {
        "node_id": "sem_unit_456",
        "type": "semantic_unit",
        "distance": 0.145,
        "similarity": 0.855,
        "content": "Research on solar energy..."
      },
      {
        "node_id": "entity_789",
        "type": "entity",
        "distance": 0.278,
        "similarity": 0.722,
        "content": "RENEWABLE ENERGY"
      }
    ]
  }
}
```

**Implementation**:
```python
# NodeRAG/utils/HNSW.py
class HNSW:
    def get_similar(self, node_id, k=20):
        """Find similar nodes via HNSW"""
        # Get embedding
        embedding = self.mapper.embeddings[node_id]

        # HNSW search
        results = self.search(embedding, k + 1)  # +1 to exclude self

        # Filter out self
        return [r for r in results if r['node_id'] != node_id][:k]
```

---

### 5. HNSW Layer Structure

```http
GET /vectors/hnsw/layers
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "layers": [
      {
        "layer": 0,
        "num_nodes": 150000,
        "avg_degree": 64,
        "max_degree": 128
      },
      {
        "layer": 1,
        "num_nodes": 5000,
        "avg_degree": 32,
        "max_degree": 64
      },
      {
        "layer": 2,
        "num_nodes": 156,
        "avg_degree": 16,
        "max_degree": 32
      }
    ],
    "total_layers": 3,
    "entry_point": "node_xyz"
  },
  "metadata": {
    "M": 64,
    "ef_construction": 200,
    "index_size_bytes": 912000000
  }
}
```

**Implementation**:
```python
# NodeRAG/utils/HNSW.py
class HNSW:
    def get_layer_structure(self):
        """Get HNSW layer info"""
        layers = []
        layer_num = 0

        while True:
            layer_graph = self.hnsw.get_layer_graph(layer_num)
            if layer_graph is None:
                break

            layers.append({
                'layer': layer_num,
                'num_nodes': len(layer_graph),
                'avg_degree': sum(len(neighbors) for neighbors in layer_graph.values()) / len(layer_graph)
            })

            layer_num += 1

        return layers
```

---

### 6. HNSW Neighbors

```http
GET /vectors/hnsw/neighbors/{node_id}
```

**Query Parameters**:
- `layer` (integer, default: 0) - HNSW layer number

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "node_id": "sem_unit_123",
    "layer": 0,
    "hnsw_neighbors": [
      {
        "node_id": "sem_unit_456",
        "distance": 0.123,
        "in_graph": true
      },
      {
        "node_id": "entity_789",
        "distance": 0.234,
        "in_graph": false
      }
    ],
    "count": 64
  },
  "metadata": {
    "note": "HNSW neighbors may differ from knowledge graph neighbors"
  }
}
```

---

### 7. Compare Embeddings

```http
POST /vectors/compare
```

**Request Body**:
```json
{
  "nodes": ["sem_unit_123", "entity_456"],
  "metric": "cosine"
}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "nodes": [
      {
        "id": "sem_unit_123",
        "content": "Dr. Emily Roberts..."
      },
      {
        "id": "entity_456",
        "content": "EMILY ROBERTS"
      }
    ],
    "distance": 0.312,
    "similarity": 0.688,
    "metric": "cosine"
  }
}
```

---

### 8. Batch Vector Search

```http
POST /vectors/search/batch
```

**Request Body**:
```json
{
  "vectors": [
    [0.123, -0.456, ...],
    [0.789, 0.234, ...]
  ],
  "k": 20,
  "combine": "union"
}
```

**Combine methods**:
- `union` - Объединение всех результатов (remove duplicates)
- `intersection` - Пересечение результатов
- `separate` - Раздельные результаты для каждого вектора

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "combine_method": "union",
    "results": [
      {
        "node_id": "sem_unit_123",
        "min_distance": 0.234,
        "avg_distance": 0.312,
        "matched_queries": [0, 1]
      }
    ]
  }
}
```

**Implementation**:
```python
# NodeRAG/utils/HNSW.py
class HNSW:
    def search_list(self, query_list, k=20, combine='union'):
        """Batch vector search"""
        idx, dist = self.hnsw.knn_query(
            np.array(query_list).astype(np.float32),
            k
        )

        if combine == 'union':
            # Remove duplicates, keep min distance
            seen = {}
            for i in range(len(idx)):
                for j in range(k):
                    node_id = self.id_map[idx[i][j]]
                    if node_id not in seen or dist[i][j] < seen[node_id]:
                        seen[node_id] = dist[i][j]

            results = [
                {'node_id': node_id, 'distance': distance}
                for node_id, distance in sorted(seen.items(), key=lambda x: x[1])
            ][:k]

        return results
```

---

### 9. HNSW Graph Export

```http
GET /vectors/hnsw/graph
```

**Query Parameters**:
- `layer` (integer, default: 0) - Layer to export
- `format` (string, default: `networkx`) - Export format (`networkx`, `cytoscape`, `graphml`)

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "graph": {
      "nodes": [
        {
          "id": "sem_unit_123",
          "layer": 0
        }
      ],
      "edges": [
        {
          "source": "sem_unit_123",
          "target": "sem_unit_456",
          "distance": 0.145
        }
      ]
    },
    "format": "networkx",
    "layer": 0
  }
}
```

**Implementation**:
```python
# NodeRAG/utils/HNSW.py
class HNSW:
    @property
    def nxgraphs(self):
        """Convert HNSW layer 0 to NetworkX graph"""
        graph_layer_0 = self.hnsw.get_layer_graph(0)

        if self._nxgraphs is None:
            self._nxgraphs = nx.Graph()
            for id, neighbors in graph_layer_0.items():
                for neighbor in neighbors:
                    self._nxgraphs.add_edge(
                        self.id_map[id],
                        self.id_map[neighbor]
                    )

        return self._nxgraphs
```

---

## Advanced Features

### 10. Vector Analytics

```http
GET /vectors/analytics
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "index_stats": {
      "total_vectors": 150000,
      "dimension": 1536,
      "index_size_mb": 870,
      "avg_query_time_ms": 8
    },
    "distribution": {
      "by_type": {
        "semantic_unit": 100000,
        "entity": 40000,
        "attribute": 5000,
        "high_level_element": 5000
      }
    },
    "embedding_space": {
      "avg_norm": 1.0,
      "std_norm": 0.0,
      "centroid": [0.001, -0.002, ...]
    }
  }
}
```

---

### 11. Dimensionality Reduction

```http
POST /vectors/reduce
```

**Request Body**:
```json
{
  "node_ids": ["sem_unit_123", "entity_456", ...],
  "method": "umap",
  "target_dim": 2,
  "include_visualization": true
}
```

**Methods**:
- `umap` - UMAP reduction
- `tsne` - t-SNE reduction
- `pca` - PCA reduction

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "reduced_vectors": [
      {
        "node_id": "sem_unit_123",
        "original_dim": 1536,
        "reduced_dim": 2,
        "coordinates": [0.234, -0.567]
      }
    ],
    "visualization_data": {
      "type": "scatter",
      "points": [...]
    }
  },
  "metadata": {
    "method": "umap",
    "variance_explained": 0.68
  }
}
```

---

## Error Cases

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `VECTOR_INVALID_DIM` | 400 | Неверная размерность вектора |
| `VECTOR_NODE_NO_EMBEDDING` | 404 | У узла нет embedding |
| `HNSW_LAYER_NOT_FOUND` | 404 | HNSW layer не существует |
| `EMBEDDING_API_ERROR` | 502 | Ошибка OpenAI API |
| `VECTOR_SEARCH_TIMEOUT` | 504 | Search timeout |

---

## Implementation Mapping

| Endpoint | Component | Method |
|----------|-----------|--------|
| `/vectors/search` | `HNSW` | `search()` |
| `/vectors/embed` | `OpenAI_Embedding` | `_create_embedding()` |
| `/vectors/node/{id}` | `Mapper` | `embeddings[node_id]` |
| `/vectors/similar/{id}` | `HNSW` | `search()` with node embedding |
| `/vectors/hnsw/layers` | `HNSW` | `get_layer_graph()` |
| `/vectors/hnsw/graph` | `HNSW` | `nxgraphs` property |

---

## Performance

### Typical Latencies

| Operation | Time | Notes |
|-----------|------|-------|
| Vector search (k=50) | ~5-10ms | HNSW index |
| Embedding generation | ~150ms | OpenAI API |
| Batch embed (100 texts) | ~500ms | Parallel API calls |
| HNSW layer export | ~2s | Layer 0, 150K nodes |

### Caching

- **Embeddings**: Кэш в Redis (TTL: 24h)
- **Search results**: Кэш для популярных векторов (TTL: 1h)

---

## Связанные документы

- [Graph API Reference](./graph-api.md)
- [Search Trajectory API](./search-trajectory-api.md)
- [Introspection API](./introspection-api.md)

---

**Версия**: 1.0
**Последнее обновление**: 2025-11-14
