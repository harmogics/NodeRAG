# Implementation Mapping

## Введение

Этот документ описывает **mapping между API endpoints и существующими компонентами NodeRAG**, включая файлы, классы, методы и точки расширения для реализации REST API.

---

## Architecture Overview

```
REST API Layer (NEW)
    ↓
┌────────────────────────────────────────┐
│  Instrumentation Layer (NEW/MODIFIED)  │
│  - TrajectoryTracker                   │
│  - ReasoningChainCapture               │
│  - IntrospectionEngine                 │
└────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────┐
│   Existing NodeRAG Components          │
│  - NodeSearch                          │
│  - HNSW                                │
│  - sparse_PPR                          │
│  - Mapper                              │
│  - LLM clients                         │
└────────────────────────────────────────┘
```

---

## Component Mapping

### 1. Query & Search API

#### POST /query

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `NodeSearch` | `NodeRAG/search/search.py` | `answer()`, `answer_async()` |
| `Retrieval` | `NodeRAG/search/Answer_base.py` | Class initialization |
| `Answer` | `NodeRAG/search/Answer_base.py` | Class initialization |

**New Components Needed**:
- `TrajectoryTracker` (NEW) - Track search steps
- `QueryCache` (NEW) - Cache query results
- `AsyncQueryProcessor` (NEW) - Async job processing

**Implementation**:
```python
# api/endpoints/query.py (NEW)
from fastapi import APIRouter, BackgroundTasks
from NodeRAG.search.search import NodeSearch
from api.instrumentation.trajectory import TrajectoryTracker

router = APIRouter()

@router.post("/query")
async def execute_query(
    query: str,
    include_trajectory: bool = True,
    include_reasoning: bool = False
):
    # Initialize tracker
    tracker = TrajectoryTracker() if include_trajectory else None

    # Modified NodeSearch with tracking
    node_search = NodeSearch(config)
    result = await node_search.answer_with_trajectory(query, tracker)

    return {
        "status": "success",
        "data": {
            "query": query,
            "answer": result['answer'].response,
            "retrieval": result['retrieval'],
            "trajectory": tracker.get_trajectory() if tracker else None
        }
    }
```

**Modifications to Existing Code**:
```python
# NodeRAG/search/search.py (MODIFIED)
class NodeSearch:
    def answer_with_trajectory(self, query: str, tracker=None):
        """Modified answer() method with trajectory tracking"""

        # Step 1: Query Embedding
        if tracker:
            tracker.start_step('query_embedding', {'query': query})

        query_embedding = np.array(
            self.config.embedding_client.request(query),
            dtype=np.float32
        )

        if tracker:
            tracker.end_step({'embedding': query_embedding.tolist()})

        # ... остальные шаги с tracking
```

---

### 2. Graph API

#### GET /graph/nodes

**Existing Components**:
| Component | File | Method/Property |
|-----------|------|-----------------|
| `NodeSearch.G` | `NodeRAG/search/search.py` | `nx.Graph` instance |
| `Mapper` | `NodeRAG/storage/mapper.py` | `get()` method |

**Implementation**:
```python
# api/endpoints/graph.py (NEW)
from fastapi import APIRouter, Query
from NodeRAG.search.search import NodeSearch

router = APIRouter()

@router.get("/graph/nodes")
def get_nodes(
    type: str = None,
    weight_min: float = None,
    page: int = 1,
    page_size: int = 50
):
    node_search = NodeSearch(config)

    # Filter nodes
    nodes = []
    for node_id in node_search.G.nodes():
        node_data = node_search.G.nodes[node_id]

        # Apply filters
        if type and node_data.get('type') != type:
            continue
        if weight_min and node_data.get('weight', 0) < weight_min:
            continue

        nodes.append({
            'id': node_id,
            'type': node_data.get('type'),
            'content': node_search.mapper.get(node_id, 'context'),
            'weight': node_data.get('weight'),
            'degree': node_search.G.degree(node_id)
        })

    # Pagination
    start = (page - 1) * page_size
    end = start + page_size

    return {
        "status": "success",
        "data": {
            "nodes": nodes[start:end]
        },
        "pagination": {
            "page": page,
            "page_size": page_size,
            "total_items": len(nodes)
        }
    }
```

---

#### GET /graph/node/{node_id}

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `NodeSearch.G` | `NodeRAG/search/search.py` | `G.nodes[node_id]` |
| `Mapper` | `NodeRAG/storage/mapper.py` | `get()`, `embeddings` |

**Implementation**:
```python
@router.get("/graph/node/{node_id}")
def get_node(node_id: str, include_embedding: bool = False):
    node_search = NodeSearch(config)

    if node_id not in node_search.G:
        raise HTTPException(status_code=404, detail="Node not found")

    node_data = node_search.G.nodes[node_id]

    result = {
        'id': node_id,
        'type': node_data.get('type'),
        'content': node_search.mapper.get(node_id, 'context'),
        'weight': node_data.get('weight'),
        'degree': node_search.G.degree(node_id)
    }

    if include_embedding and node_id in node_search.mapper.embeddings:
        result['embedding'] = {
            'vector': node_search.mapper.embeddings[node_id].tolist(),
            'dimension': 1536
        }

    return {"status": "success", "data": {"node": result}}
```

---

#### POST /graph/subgraph

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `nx.ego_graph` | NetworkX | `nx.ego_graph()` |
| `NodeSearch.G` | `NodeRAG/search/search.py` | Graph instance |

**Implementation**:
```python
import networkx as nx

@router.post("/graph/subgraph")
def get_subgraph(request: SubgraphRequest):
    node_search = NodeSearch(config)

    if request.method == 'ego':
        subgraph = nx.ego_graph(
            node_search.G,
            request.center,
            radius=request.radius
        )
    elif request.method == 'nodes':
        subgraph = node_search.G.subgraph(request.nodes)

    # Build response
    return {
        "status": "success",
        "data": {
            "subgraph": {
                "nodes": [
                    {
                        'id': n,
                        'type': node_search.G.nodes[n].get('type'),
                        'content': node_search.mapper.get(n, 'context')
                    }
                    for n in subgraph.nodes()
                ],
                "edges": [
                    {
                        'source': u,
                        'target': v,
                        'weight': subgraph[u][v].get('weight', 1.0)
                    }
                    for u, v in subgraph.edges()
                ]
            }
        }
    }
```

---

### 3. Vector API

#### POST /vectors/search

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `HNSW` | `NodeRAG/utils/HNSW.py` | `search()` |

**Implementation**:
```python
# api/endpoints/vectors.py (NEW)
@router.post("/vectors/search")
def vector_search(request: VectorSearchRequest):
    node_search = NodeSearch(config)

    # Convert text to vector if needed
    if request.text:
        vector = node_search.config.embedding_client.request(request.text)
    else:
        vector = request.vector

    # HNSW search
    results = node_search.hnsw.search(
        np.array(vector, dtype=np.float32),
        HNSW_results=request.k
    )

    return {
        "status": "success",
        "data": {
            "results": [
                {
                    'node_id': node_id,
                    'distance': distance,
                    'similarity': 1.0 - distance,
                    'content': node_search.mapper.get(node_id, 'context')
                }
                for distance, node_id in results
            ]
        }
    }
```

---

#### POST /vectors/embed

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `OpenAI_Embedding` | `NodeRAG/LLM/LLM.py` | `_create_embedding()` |

**Implementation**:
```python
@router.post("/vectors/embed")
async def embed_text(request: EmbedRequest):
    embedding_client = config.embedding_client

    if isinstance(request.text, str):
        embeddings = [embedding_client.request(request.text)]
    else:
        embeddings = embedding_client.request(request.text)

    return {
        "status": "success",
        "data": {
            "embeddings": [
                {
                    'text': text,
                    'embedding': emb,
                    'dimension': 1536
                }
                for text, emb in zip(request.text, embeddings)
            ]
        }
    }
```

---

#### GET /vectors/hnsw/layers

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `HNSW` | `NodeRAG/utils/HNSW.py` | `get_layer_graph()` |

**Implementation**:
```python
@router.get("/vectors/hnsw/layers")
def get_hnsw_layers():
    node_search = NodeSearch(config)

    layers = []
    layer_num = 0

    while True:
        layer_graph = node_search.hnsw.get_layer_graph(layer_num)
        if layer_graph is None:
            break

        layers.append({
            'layer': layer_num,
            'num_nodes': len(layer_graph),
            'avg_degree': sum(len(neighbors) for neighbors in layer_graph.values()) / len(layer_graph) if layer_graph else 0
        })

        layer_num += 1

    return {
        "status": "success",
        "data": {
            "layers": layers,
            "total_layers": len(layers)
        }
    }
```

---

### 4. Search Trajectory API

#### GET /trajectory/{query_id}

**New Components Needed**:
```python
# api/instrumentation/trajectory.py (NEW)
import time
import uuid

class TrajectoryTracker:
    """Track search trajectory with timing"""

    def __init__(self):
        self.query_id = str(uuid.uuid4())
        self.steps = []
        self.current_step = None

    def start_step(self, name: str, input_data: dict):
        self.current_step = {
            'step_id': len(self.steps) + 1,
            'name': name,
            'timestamp_start': time.time(),
            'input': input_data
        }

    def end_step(self, output_data: dict, metadata: dict = None):
        if self.current_step:
            self.current_step['timestamp_end'] = time.time()
            self.current_step['duration_ms'] = int(
                (self.current_step['timestamp_end'] - self.current_step['timestamp_start']) * 1000
            )
            self.current_step['output'] = output_data
            if metadata:
                self.current_step['metadata'] = metadata

            self.steps.append(self.current_step)
            self.current_step = None

    def get_trajectory(self):
        return {
            'query_id': self.query_id,
            'total_duration_ms': sum(s['duration_ms'] for s in self.steps),
            'steps': self.steps
        }


# api/storage/trajectory_store.py (NEW)
class TrajectoryStore:
    """Store and retrieve trajectories"""

    def __init__(self):
        self.trajectories = {}  # In production: use Redis/DB

    def save(self, query_id: str, trajectory: dict):
        self.trajectories[query_id] = trajectory

    def get(self, query_id: str):
        return self.trajectories.get(query_id)
```

**Implementation**:
```python
# api/endpoints/trajectory.py (NEW)
@router.get("/trajectory/{query_id}")
def get_trajectory(query_id: str):
    trajectory_store = TrajectoryStore()
    trajectory = trajectory_store.get(query_id)

    if not trajectory:
        raise HTTPException(status_code=404, detail="Trajectory not found")

    return {
        "status": "success",
        "data": trajectory
    }
```

---

### 5. Introspection API

#### POST /introspection/attractors

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `HNSW.search()` | `NodeRAG/utils/HNSW.py` | Analog attractors |
| `decompose_query()` | `NodeRAG/search/search.py` | Symbolic attractors |
| `accurate_search()` | `NodeRAG/search/search.py` | Entity matching |

**New Components**:
```python
# api/introspection/engine.py (NEW)
class IntrospectionEngine:
    """Engine for introspection operations"""

    def __init__(self, node_search: NodeSearch):
        self.node_search = node_search

    def get_attractors(self, query: str, analog_k: int = 50):
        """Get dual-mode attractors"""

        # Analog attractors
        query_embedding = self.node_search.config.embedding_client.request(query)
        hnsw_results = self.node_search.hnsw.search(
            np.array(query_embedding, dtype=np.float32),
            HNSW_results=analog_k
        )

        analog = {
            'source': 'hnsw_search',
            'nodes': [
                {
                    'node_id': node_id,
                    'distance': distance,
                    'weight': 1.0
                }
                for distance, node_id in hnsw_results
            ]
        }

        # Symbolic attractors
        entities = self.node_search.decompose_query(query)
        matched_nodes = self.node_search.accurate_search(entities)

        symbolic = {
            'source': 'query_decomposition',
            'extracted_entities': entities,
            'nodes': [
                {
                    'node_id': node_id,
                    'weight': 2.0
                }
                for node_id in matched_nodes
            ]
        }

        # Combined personalization
        personalization = {}
        for node in analog['nodes']:
            personalization[node['node_id']] = node['weight']
        for node in symbolic['nodes']:
            personalization[node['node_id']] = personalization.get(node['node_id'], 0) + node['weight']

        return {
            'analog': analog,
            'symbolic': symbolic,
            'combined': {
                'personalization_vector': personalization
            }
        }
```

**Implementation**:
```python
# api/endpoints/introspection.py (NEW)
@router.post("/introspection/attractors")
def get_attractors(request: AttractorsRequest):
    node_search = NodeSearch(config)
    introspection = IntrospectionEngine(node_search)

    attractors = introspection.get_attractors(
        request.query,
        analog_k=request.analog_k
    )

    return {
        "status": "success",
        "data": {
            "query": request.query,
            "attractors": attractors
        }
    }
```

---

#### GET /introspection/concepts/{concept_id}

**Existing Components**:
| Component | File | Method |
|-----------|------|--------|
| `NodeSearch.G` | `NodeRAG/search/search.py` | Graph navigation |
| `Mapper` | `NodeRAG/storage/mapper.py` | Content/embeddings |

**New Components**:
```python
# api/introspection/engine.py (ADDITION)
class IntrospectionEngine:
    def get_concept_network(self, concept_id: str, depth: int = 2):
        """Get concept network"""

        # Concept info
        concept = {
            'id': concept_id,
            'type': self.node_search.G.nodes[concept_id].get('type'),
            'content': self.node_search.mapper.get(concept_id, 'context')
        }

        # Immediate neighbors
        neighbors = []
        for neighbor_id in self.node_search.G.neighbors(concept_id):
            neighbors.append({
                'node_id': neighbor_id,
                'type': self.node_search.G.nodes[neighbor_id].get('type'),
                'content': self.node_search.mapper.get(neighbor_id, 'context')
            })

        # Related concepts (via similarity + paths)
        related = self.find_related_concepts(concept_id, depth)

        return {
            'concept': concept,
            'network': {
                'immediate_neighbors': neighbors,
                'related_concepts': related
            }
        }

    def find_related_concepts(self, concept_id: str, depth: int):
        """Find related concepts"""
        import networkx as nx
        from scipy.spatial.distance import cosine

        related = []

        for node_id in self.node_search.G.nodes():
            if node_id == concept_id:
                continue

            # Only entities/HLE
            if self.node_search.G.nodes[node_id].get('type') not in ['entity', 'high_level_element_title']:
                continue

            try:
                path_length = nx.shortest_path_length(
                    self.node_search.G,
                    concept_id,
                    node_id
                )

                if path_length <= depth:
                    # Calculate similarity
                    similarity = None
                    if (concept_id in self.node_search.mapper.embeddings and
                        node_id in self.node_search.mapper.embeddings):
                        emb1 = self.node_search.mapper.embeddings[concept_id]
                        emb2 = self.node_search.mapper.embeddings[node_id]
                        similarity = 1.0 - cosine(emb1, emb2)

                    related.append({
                        'concept_id': node_id,
                        'content': self.node_search.mapper.get(node_id, 'context'),
                        'path_length': path_length,
                        'similarity': similarity
                    })
            except nx.NetworkXNoPath:
                pass

        # Sort by similarity
        related.sort(key=lambda x: x.get('similarity') or 0, reverse=True)

        return related[:20]
```

---

## File Structure

```
NodeRAG/
├── api/                           # NEW
│   ├── __init__.py
│   ├── main.py                    # FastAPI app
│   ├── endpoints/
│   │   ├── query.py               # Query & Search API
│   │   ├── graph.py               # Graph API
│   │   ├── vectors.py             # Vector API
│   │   ├── trajectory.py          # Trajectory API
│   │   └── introspection.py       # Introspection API
│   ├── instrumentation/
│   │   ├── trajectory.py          # TrajectoryTracker
│   │   └── reasoning.py           # Reasoning chain capture
│   ├── introspection/
│   │   └── engine.py              # IntrospectionEngine
│   ├── storage/
│   │   ├── trajectory_store.py    # Trajectory storage
│   │   └── query_cache.py         # Query cache
│   └── models/
│       ├── requests.py            # Pydantic request models
│       └── responses.py           # Pydantic response models
├── search/
│   ├── search.py                  # MODIFIED: add trajectory support
│   └── Answer_base.py             # Existing
├── utils/
│   ├── HNSW.py                    # Existing
│   └── PPR.py                     # Existing
└── ...
```

---

## Dependencies

### New Packages Needed

```toml
# requirements-api.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
redis==5.0.1          # For caching
prometheus-client==0.19.0  # For metrics
```

---

## Связанные документы

- [API Architecture](./api-architecture.md)
- [Graph API](./graph-api.md)
- [Vector API](./vector-api.md)
- [Search Trajectory API](./search-trajectory-api.md)
- [Introspection API](./introspection-api.md)

---

**Версия**: 1.0
**Последнее обновление**: 2025-11-14
