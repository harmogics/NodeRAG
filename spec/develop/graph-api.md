# Graph API Reference

## Введение

**Graph API** предоставляет прямой доступ к графу знаний NodeRAG, построенному на основе **NetworkX**. API позволяет исследовать узлы, рёбра, сообщества, пути и подграфы.

**Base Path**: `/api/v1/graph`

---

## Структура графа

### Типы узлов (Node Types)

| Тип | Описание | Атрибуты |
|-----|----------|----------|
| `semantic_unit` | Семантическая единица текста | `content`, `hash_id`, `embedding` |
| `entity` | Извлечённая сущность | `content`, `weight`, `attributes` |
| `relationship` | Отношение между entities | `content`, `source`, `target` |
| `attribute` | Описание entity (LLM-generated) | `content`, `node` (related entity) |
| `high_level_element` | Абстракция из community | `context`, `title_hash_id`, `related_nodes` |
| `high_level_element_title` | Заголовок HLE | `context`, `related_node` (HLE id) |

### Атрибуты рёбер (Edge Attributes)

| Атрибут | Тип | Описание |
|---------|-----|----------|
| `weight` | float | Вес связи (может быть > 1 для множественных связей) |

### Граф Characteristics

- **Тип**: Undirected weighted graph
- **Размер**: ~500K nodes, ~2M edges (для 10K документов)
- **Компоненты**: Knowledge graph + HNSW graph (concatenated)
- **Balancing**: Unbalance adjustment applied

---

## Endpoints

### 1. Список узлов

```http
GET /graph/nodes
```

**Query Parameters**:

| Параметр | Тип | Default | Описание |
|----------|-----|---------|----------|
| `type` | string | null | Фильтр по типу узла |
| `weight_min` | float | null | Минимальный вес узла |
| `weight_max` | float | null | Максимальный вес узла |
| `has_embedding` | boolean | null | Только узлы с embeddings |
| `page` | integer | 1 | Номер страницы |
| `page_size` | integer | 50 | Размер страницы (max 1000) |
| `sort_by` | string | null | Поле для сортировки (`weight`, `degree`) |
| `order` | string | `asc` | Порядок сортировки (`asc`, `desc`) |
| `search` | string | null | Поиск по content (substring match) |

**Example Request**:
```http
GET /graph/nodes?type=entity&weight_min=5&page=1&page_size=100&sort_by=weight&order=desc
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "nodes": [
      {
        "id": "entity_456",
        "type": "entity",
        "content": "EMILY ROBERTS",
        "weight": 12,
        "degree": 45,
        "has_embedding": true,
        "attributes": ["attr_789"],
        "created_at": "2024-09-01T10:00:00Z"
      },
      {
        "id": "entity_123",
        "type": "entity",
        "content": "RENEWABLE ENERGY",
        "weight": 8,
        "degree": 38,
        "has_embedding": true,
        "attributes": null,
        "created_at": "2024-09-01T10:05:00Z"
      }
    ]
  },
  "pagination": {
    "page": 1,
    "page_size": 100,
    "total_items": 5432,
    "total_pages": 55
  },
  "metadata": {
    "graph_stats": {
      "total_nodes": 500000,
      "total_edges": 2000000,
      "node_types": {
        "semantic_unit": 100000,
        "entity": 50000,
        "relationship": 80000,
        "attribute": 5000,
        "high_level_element": 2000,
        "high_level_element_title": 2000
      }
    }
  },
  "links": {
    "self": "/graph/nodes?type=entity&weight_min=5&page=1",
    "next": "/graph/nodes?type=entity&weight_min=5&page=2"
  }
}
```

**Implementation**:
```python
# NodeRAG/search/search.py
class NodeSearch:
    def get_nodes(self, filters):
        nodes = []
        for node_id in self.G.nodes():
            node_data = self.G.nodes[node_id]

            # Apply filters
            if filters.get('type') and node_data.get('type') != filters['type']:
                continue
            if filters.get('weight_min') and node_data.get('weight', 0) < filters['weight_min']:
                continue

            # Build response
            nodes.append({
                'id': node_id,
                'type': node_data.get('type'),
                'content': self.mapper.get(node_id, 'context'),
                'weight': node_data.get('weight'),
                'degree': self.G.degree(node_id),
                'has_embedding': node_id in self.mapper.embeddings,
                'attributes': node_data.get('attributes')
            })

        return nodes
```

---

### 2. Детали узла

```http
GET /graph/node/{node_id}
```

**Path Parameters**:
- `node_id` (string, required) - ID узла

**Query Parameters**:

| Параметр | Тип | Default | Описание |
|----------|-----|---------|----------|
| `include_embedding` | boolean | false | Включить embedding vector |
| `include_neighbors` | boolean | false | Включить список соседей |
| `neighbors_limit` | integer | 10 | Лимит соседей |
| `fields` | string | null | Comma-separated список полей |

**Example Request**:
```http
GET /graph/node/entity_456?include_embedding=true&include_neighbors=true
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "node": {
      "id": "entity_456",
      "type": "entity",
      "content": "EMILY ROBERTS",
      "weight": 12,
      "degree": 45,
      "betweenness_centrality": 0.0234,
      "created_at": "2024-09-01T10:00:00Z",
      "embedding": {
        "model": "text-embedding-3-small",
        "dimension": 1536,
        "vector": [0.123, -0.456, ...],
        "norm": 1.0
      },
      "attributes": [
        {
          "id": "attr_789",
          "content": "Dr. Emily Roberts is a leading researcher in renewable energy..."
        }
      ],
      "neighbors": {
        "count": 45,
        "sample": [
          {
            "id": "sem_unit_123",
            "type": "semantic_unit",
            "edge_weight": 1.0,
            "content": "Dr. Emily Roberts attended..."
          },
          {
            "id": "rel_999",
            "type": "relationship",
            "edge_weight": 1.0,
            "content": "EMILY ROBERTS, attended, CONFERENCE"
          }
        ],
        "by_type": {
          "semantic_unit": 20,
          "relationship": 15,
          "attribute": 1,
          "high_level_element": 9
        }
      },
      "communities": [
        {
          "community_id": "comm_42",
          "modularity": 0.56,
          "size": 234
        }
      ]
    }
  },
  "links": {
    "self": "/graph/node/entity_456",
    "neighbors": "/graph/neighbors/entity_456",
    "embedding": "/vectors/node/entity_456",
    "subgraph": "/graph/subgraph?center=entity_456&radius=2"
  }
}
```

**Implementation**:
```python
# NodeRAG/search/search.py
class NodeSearch:
    def get_node_details(self, node_id, include_embedding=False, include_neighbors=False):
        if node_id not in self.G:
            raise NodeNotFoundError(node_id)

        node_data = self.G.nodes[node_id]
        result = {
            'id': node_id,
            'type': node_data.get('type'),
            'content': self.mapper.get(node_id, 'context'),
            'weight': node_data.get('weight'),
            'degree': self.G.degree(node_id)
        }

        if include_embedding and node_id in self.mapper.embeddings:
            result['embedding'] = {
                'model': 'text-embedding-3-small',
                'dimension': 1536,
                'vector': self.mapper.embeddings[node_id].tolist()
            }

        if include_neighbors:
            neighbors = list(self.G.neighbors(node_id))
            result['neighbors'] = {
                'count': len(neighbors),
                'sample': [
                    {
                        'id': n,
                        'type': self.G.nodes[n].get('type'),
                        'edge_weight': self.G[node_id][n].get('weight', 1.0)
                    }
                    for n in neighbors[:10]
                ]
            }

        return result
```

---

### 3. Соседи узла

```http
GET /graph/neighbors/{node_id}
```

**Query Parameters**:

| Параметр | Тип | Default | Описание |
|----------|-----|---------|----------|
| `type` | string | null | Фильтр по типу соседей |
| `min_weight` | float | null | Минимальный вес ребра |
| `limit` | integer | 100 | Максимум соседей |
| `sort_by` | string | `weight` | Сортировка (`weight`, `degree`) |
| `order` | string | `desc` | Порядок |

**Example Request**:
```http
GET /graph/neighbors/entity_456?type=semantic_unit&limit=20
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "node_id": "entity_456",
    "neighbors_count": 45,
    "filtered_count": 20,
    "neighbors": [
      {
        "id": "sem_unit_123",
        "type": "semantic_unit",
        "content": "Dr. Emily Roberts attended...",
        "edge_weight": 1.0,
        "neighbor_degree": 15,
        "neighbor_weight": 3
      },
      {
        "id": "sem_unit_789",
        "type": "semantic_unit",
        "content": "Research on renewable energy...",
        "edge_weight": 1.0,
        "neighbor_degree": 22,
        "neighbor_weight": 5
      }
    ],
    "statistics": {
      "total_neighbors": 45,
      "by_type": {
        "semantic_unit": 20,
        "relationship": 15,
        "attribute": 1,
        "high_level_element": 9
      },
      "avg_edge_weight": 1.05,
      "max_edge_weight": 2.0
    }
  }
}
```

**Implementation**:
```python
# NodeRAG/search/search.py
class NodeSearch:
    def get_neighbors(self, node_id, filters):
        if node_id not in self.G:
            raise NodeNotFoundError(node_id)

        neighbors = []
        for neighbor_id in self.G.neighbors(node_id):
            neighbor_data = self.G.nodes[neighbor_id]
            edge_data = self.G[node_id][neighbor_id]

            # Apply filters
            if filters.get('type') and neighbor_data.get('type') != filters['type']:
                continue
            if filters.get('min_weight') and edge_data.get('weight', 1.0) < filters['min_weight']:
                continue

            neighbors.append({
                'id': neighbor_id,
                'type': neighbor_data.get('type'),
                'content': self.mapper.get(neighbor_id, 'context'),
                'edge_weight': edge_data.get('weight', 1.0),
                'neighbor_degree': self.G.degree(neighbor_id),
                'neighbor_weight': neighbor_data.get('weight')
            })

        # Sort
        if filters.get('sort_by') == 'weight':
            neighbors.sort(key=lambda x: x['edge_weight'], reverse=(filters.get('order') == 'desc'))

        return neighbors[:filters.get('limit', 100)]
```

---

### 4. Список рёбер

```http
GET /graph/edges
```

**Query Parameters**:

| Параметр | Тип | Default | Описание |
|----------|-----|---------|----------|
| `source_type` | string | null | Тип source узла |
| `target_type` | string | null | Тип target узла |
| `weight_min` | float | null | Минимальный вес ребра |
| `page` | integer | 1 | Номер страницы |
| `page_size` | integer | 100 | Размер страницы |

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "edges": [
      {
        "source": "entity_456",
        "source_type": "entity",
        "target": "sem_unit_123",
        "target_type": "semantic_unit",
        "weight": 1.0
      }
    ]
  },
  "pagination": {...}
}
```

---

### 5. Подграф

```http
POST /graph/subgraph
```

**Request Body**:
```json
{
  "method": "ego",
  "center": "entity_456",
  "radius": 2,
  "filters": {
    "node_types": ["entity", "semantic_unit"],
    "min_edge_weight": 0.5
  },
  "include_stats": true
}
```

**Alternative methods**:
- `ego`: Ego graph (радиус вокруг центра)
- `nodes`: Induced subgraph (заданные узлы)
- `bfs`: BFS traversal с depth
- `community`: Community subgraph

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "subgraph": {
      "nodes": [
        {
          "id": "entity_456",
          "type": "entity",
          "content": "EMILY ROBERTS",
          "level": 0
        },
        {
          "id": "sem_unit_123",
          "type": "semantic_unit",
          "content": "...",
          "level": 1
        }
      ],
      "edges": [
        {
          "source": "entity_456",
          "target": "sem_unit_123",
          "weight": 1.0
        }
      ],
      "statistics": {
        "num_nodes": 45,
        "num_edges": 120,
        "density": 0.065,
        "avg_degree": 5.3,
        "node_types": {
          "entity": 5,
          "semantic_unit": 30,
          "relationship": 10
        }
      }
    },
    "visualization_data": {
      "format": "cytoscape",
      "elements": {
        "nodes": [...],
        "edges": [...]
      }
    }
  }
}
```

**Implementation**:
```python
# NodeRAG/search/search.py
import networkx as nx

class NodeSearch:
    def get_subgraph(self, method, params, filters):
        if method == 'ego':
            # Ego graph
            center = params['center']
            radius = params.get('radius', 1)
            subgraph = nx.ego_graph(self.G, center, radius=radius)

        elif method == 'nodes':
            # Induced subgraph
            nodes = params['nodes']
            subgraph = self.G.subgraph(nodes)

        # Apply filters
        if filters.get('node_types'):
            nodes_to_keep = [
                n for n in subgraph.nodes()
                if self.G.nodes[n].get('type') in filters['node_types']
            ]
            subgraph = subgraph.subgraph(nodes_to_keep)

        # Build response
        return {
            'nodes': [
                {
                    'id': n,
                    'type': self.G.nodes[n].get('type'),
                    'content': self.mapper.get(n, 'context')
                }
                for n in subgraph.nodes()
            ],
            'edges': [
                {
                    'source': u,
                    'target': v,
                    'weight': subgraph[u][v].get('weight', 1.0)
                }
                for u, v in subgraph.edges()
            ],
            'statistics': {
                'num_nodes': subgraph.number_of_nodes(),
                'num_edges': subgraph.number_of_edges(),
                'density': nx.density(subgraph)
            }
        }
```

---

### 6. Кратчайшие пути

```http
POST /graph/paths
```

**Request Body**:
```json
{
  "source": "entity_456",
  "target": "entity_789",
  "method": "shortest",
  "k": 3,
  "max_length": 5,
  "weight": "weight"
}
```

**Methods**:
- `shortest`: Единственный кратчайший путь
- `all_shortest`: Все кратчайшие пути
- `k_shortest`: K кратчайших путей

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "source": "entity_456",
    "target": "entity_789",
    "paths": [
      {
        "path": ["entity_456", "sem_unit_123", "entity_789"],
        "length": 2,
        "weight": 2.0,
        "nodes": [
          {
            "id": "entity_456",
            "type": "entity",
            "content": "EMILY ROBERTS"
          },
          {
            "id": "sem_unit_123",
            "type": "semantic_unit",
            "content": "..."
          },
          {
            "id": "entity_789",
            "type": "entity",
            "content": "RENEWABLE ENERGY"
          }
        ],
        "edges": [
          {
            "source": "entity_456",
            "target": "sem_unit_123",
            "weight": 1.0
          },
          {
            "source": "sem_unit_123",
            "target": "entity_789",
            "weight": 1.0
          }
        ]
      }
    ],
    "metadata": {
      "total_paths": 1,
      "shortest_length": 2,
      "avg_length": 2.0
    }
  }
}
```

**Implementation**:
```python
# NodeRAG/search/search.py
import networkx as nx

class NodeSearch:
    def get_paths(self, source, target, method, k=3, max_length=5):
        if method == 'shortest':
            path = nx.shortest_path(self.G, source, target, weight='weight')
            paths = [path]

        elif method == 'all_shortest':
            paths = list(nx.all_shortest_paths(self.G, source, target, weight='weight'))

        elif method == 'k_shortest':
            # K shortest paths (requires additional library or custom implementation)
            paths = self.get_k_shortest_paths(source, target, k)

        # Build response
        result_paths = []
        for path in paths:
            if len(path) - 1 > max_length:
                continue

            result_paths.append({
                'path': path,
                'length': len(path) - 1,
                'weight': self.calculate_path_weight(path),
                'nodes': [
                    {
                        'id': node,
                        'type': self.G.nodes[node].get('type'),
                        'content': self.mapper.get(node, 'context')
                    }
                    for node in path
                ],
                'edges': [
                    {
                        'source': path[i],
                        'target': path[i+1],
                        'weight': self.G[path[i]][path[i+1]].get('weight', 1.0)
                    }
                    for i in range(len(path) - 1)
                ]
            })

        return result_paths
```

---

### 7. Сообщества (Communities)

```http
GET /graph/communities
```

**Query Parameters**:

| Параметр | Тип | Default | Описание |
|----------|-----|---------|----------|
| `algorithm` | string | `leiden` | Алгоритм (`leiden`, `louvain`) |
| `min_size` | integer | 10 | Минимальный размер community |
| `page` | integer | 1 | Номер страницы |
| `page_size` | integer | 50 | Размер страницы |

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "communities": [
      {
        "id": "comm_42",
        "size": 234,
        "modularity": 0.56,
        "high_level_elements": [
          {
            "id": "hle_1",
            "title": "Renewable Energy Research",
            "description": "..."
          }
        ],
        "top_nodes": [
          {
            "id": "entity_456",
            "type": "entity",
            "content": "EMILY ROBERTS",
            "centrality": 0.89
          }
        ]
      }
    ]
  },
  "metadata": {
    "total_communities": 123,
    "algorithm": "leiden",
    "modularity": 0.58,
    "execution_time_ms": 12345
  }
}
```

---

### 8. Community Details

```http
GET /graph/community/{community_id}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "community": {
      "id": "comm_42",
      "size": 234,
      "modularity": 0.56,
      "nodes": [
        {
          "id": "entity_456",
          "type": "entity",
          "content": "EMILY ROBERTS",
          "degree_within_community": 23,
          "degree_total": 45,
          "centrality": 0.89
        }
      ],
      "high_level_elements": [
        {
          "id": "hle_1",
          "title": "Renewable Energy Research",
          "description": "Recent breakthroughs in solar panel efficiency...",
          "embedding": [0.123, ...],
          "related_nodes_count": 45
        }
      ],
      "statistics": {
        "density": 0.12,
        "avg_degree": 8.5,
        "node_types": {
          "entity": 30,
          "semantic_unit": 180,
          "relationship": 24
        }
      }
    }
  },
  "links": {
    "subgraph": "/graph/subgraph?method=community&community_id=comm_42"
  }
}
```

---

## Graph Modifications

### 9. Add Node

```http
POST /graph/node
```

**Request Body**:
```json
{
  "id": "custom_node_1",
  "type": "entity",
  "content": "CUSTOM ENTITY",
  "weight": 1,
  "attributes": {
    "custom_field": "value"
  },
  "embedding": [0.123, -0.456, ...]
}
```

**Response** (201 Created):
```json
{
  "status": "success",
  "data": {
    "node_id": "custom_node_1",
    "created": true
  },
  "links": {
    "node": "/graph/node/custom_node_1"
  }
}
```

### 10. Add Edge

```http
POST /graph/edge
```

**Request Body**:
```json
{
  "source": "entity_456",
  "target": "custom_node_1",
  "weight": 1.5
}
```

**Response** (201 Created):
```json
{
  "status": "success",
  "data": {
    "edge": {
      "source": "entity_456",
      "target": "custom_node_1",
      "weight": 1.5
    },
    "created": true
  }
}
```

---

## Advanced Queries

### 11. Graph Query Language

```http
POST /graph/query
```

**Request Body** (DSL):
```json
{
  "query": {
    "find": "nodes",
    "where": {
      "type": "entity",
      "weight": { "gte": 5 },
      "degree": { "gte": 20 }
    },
    "expand": {
      "neighbors": {
        "type": "semantic_unit",
        "limit": 5
      }
    },
    "return": {
      "fields": ["id", "content", "weight", "neighbors"]
    },
    "limit": 50
  }
}
```

**Response**:
```json
{
  "status": "success",
  "data": {
    "results": [
      {
        "id": "entity_456",
        "content": "EMILY ROBERTS",
        "weight": 12,
        "neighbors": [...]
      }
    ]
  }
}
```

---

## GraphQL Support

### 12. GraphQL Endpoint

```http
POST /graph/graphql
```

**Request Body**:
```graphql
{
  node(id: "entity_456") {
    id
    type
    content
    weight
    degree
    embedding {
      dimension
      vector
    }
    neighbors(type: "semantic_unit", limit: 10) {
      id
      content
      edgeWeight
    }
    community {
      id
      size
      highLevelElements {
        title
        description
      }
    }
    paths(target: "entity_789", maxLength: 3) {
      path
      length
      nodes {
        id
        content
      }
    }
  }
}
```

---

## Performance Considerations

### Caching

- **Hot nodes**: In-memory cache для frequently accessed nodes
- **Communities**: Кэш результатов Leiden algorithm (TTL: 24h)
- **Paths**: Кэш shortest paths между популярными узлами

### Pagination

Все list endpoints поддерживают pagination (max 1000 items per page)

### Timeouts

- Simple queries: 5s timeout
- Complex subgraph/path queries: 30s timeout
- Community detection: 60s timeout

---

## Error Cases

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `GRAPH_NODE_NOT_FOUND` | 404 | Node ID не найден |
| `GRAPH_EDGE_NOT_FOUND` | 404 | Edge не найдено |
| `GRAPH_INVALID_TYPE` | 400 | Неверный тип узла |
| `GRAPH_NO_PATH` | 404 | Путь не существует |
| `GRAPH_TIMEOUT` | 504 | Операция превысила timeout |
| `GRAPH_TOO_LARGE` | 413 | Subgraph слишком большой |

---

## Implementation Mapping

### Core Components

| Endpoint | NodeRAG Component | Method |
|----------|------------------|--------|
| `/graph/nodes` | `NodeSearch.G` | `G.nodes()` |
| `/graph/node/{id}` | `NodeSearch.G` | `G.nodes[node_id]` |
| `/graph/neighbors/{id}` | `NodeSearch.G` | `G.neighbors(node_id)` |
| `/graph/subgraph` | `nx.ego_graph()` | NetworkX |
| `/graph/paths` | `nx.shortest_path()` | NetworkX |
| `/graph/communities` | `SummaryGeneration` | `partition()` (Leiden) |

### Data Sources

- **Graph**: `NodeSearch.G` (NetworkX graph)
- **Node content**: `NodeSearch.mapper.get(node_id, 'context')`
- **Embeddings**: `NodeSearch.mapper.embeddings[node_id]`
- **Communities**: Cached from indexing pipeline

---

## Examples

### Example 1: Find Entity and Explore

```bash
# 1. Search entity by name
curl -X GET "https://api.noderag.com/api/v1/graph/nodes?type=entity&search=Emily"

# 2. Get details
curl -X GET "https://api.noderag.com/api/v1/graph/node/entity_456?include_neighbors=true"

# 3. Get ego subgraph
curl -X POST "https://api.noderag.com/api/v1/graph/subgraph" \
  -H "Content-Type: application/json" \
  -d '{
    "method": "ego",
    "center": "entity_456",
    "radius": 2
  }'
```

### Example 2: Explore Conceptual Path

```bash
# Find path between two concepts
curl -X POST "https://api.noderag.com/api/v1/graph/paths" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "entity_456",
    "target": "entity_789",
    "method": "k_shortest",
    "k": 3
  }'
```

---

## Связанные документы

- [API Architecture Overview](./api-architecture.md)
- [Vector API Reference](./vector-api.md)
- [Search Trajectory API](./search-trajectory-api.md)
- [Implementation Mapping](./implementation-mapping.md)

---

**Версия**: 1.0
**Последнее обновление**: 2025-11-14
