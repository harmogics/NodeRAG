# Introspection API Reference

## Введение

**Introspection API** предоставляет доступ к концептуальным структурам NodeRAG: attractors, concept networks, related queries. Это ключевой API для исследования внутренних reasoning механизмов.

**Base Path**: `/api/v1/introspection`

---

## Endpoints Categories

1. **Attractors** - Dual-mode attractors (analog + symbolic)
2. **Concepts & Networks** - Концептуальные сети и связи
3. **Related Queries** - Связанные вопросы и предложения

---

## 1. Attractors API

### 1.1 Get Attractors for Query

```http
POST /introspection/attractors
```

**Request Body**:
```json
{
  "query": "What did Emily Roberts research?",
  "include_weights": true,
  "include_content": true,
  "analog_k": 50,
  "decompose_query": true
}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "query": "What did Emily Roberts research?",
    "attractors": {
      "analog": {
        "source": "hnsw_search",
        "method": "cosine_similarity",
        "query_embedding": [0.123, -0.456, ...],
        "k": 50,
        "weight_per_node": 1.0,
        "nodes": [
          {
            "node_id": "sem_unit_123",
            "type": "semantic_unit",
            "rank": 1,
            "distance": 0.234,
            "similarity": 0.766,
            "weight": 1.0,
            "content": "Dr. Emily Roberts presented research on renewable energy..."
          },
          {
            "node_id": "sem_unit_456",
            "type": "semantic_unit",
            "rank": 2,
            "distance": 0.289,
            "similarity": 0.711,
            "weight": 1.0,
            "content": "Solar panel efficiency improvements through novel materials..."
          }
        ],
        "total_nodes": 50
      },
      "symbolic": {
        "source": "query_decomposition",
        "method": "exact_match",
        "extracted_entities": ["EMILY ROBERTS", "RESEARCH"],
        "weight_per_node": 2.0,
        "nodes": [
          {
            "node_id": "entity_456",
            "type": "entity",
            "content": "EMILY ROBERTS",
            "matched_entity": "EMILY ROBERTS",
            "match_type": "exact",
            "weight": 2.0,
            "metadata": {
              "degree": 45,
              "node_weight": 12,
              "has_attributes": true
            }
          }
        ],
        "total_nodes": 1,
        "unmatched_entities": ["RESEARCH"]
      },
      "combined": {
        "method": "union",
        "total_unique_nodes": 51,
        "personalization_vector": {
          "sem_unit_123": 1.0,
          "sem_unit_456": 1.0,
          "entity_456": 2.0,
          "_total_weight": 52.0,
          "_normalized": false
        },
        "weight_distribution": {
          "analog_total": 50.0,
          "symbolic_total": 2.0,
          "ratio_analog_symbolic": 25.0
        }
      }
    },
    "config": {
      "similarity_weight": 1.0,
      "accuracy_weight": 2.0,
      "dual_mode_enabled": true
    }
  },
  "metadata": {
    "execution_time_ms": 1165,
    "steps": {
      "embedding_ms": 150,
      "hnsw_search_ms": 8,
      "query_decomposition_ms": 1000,
      "accurate_search_ms": 7
    }
  }
}
```

**Implementation**:
```python
# NodeRAG/search/introspection.py (NEW)
class Introspection:
    def get_attractors(self, query, analog_k=50, decompose_query=True):
        """Get dual-mode attractors for query"""

        # Analog attractors (HNSW)
        query_embedding = self.config.embedding_client.request(query)
        hnsw_results = self.hnsw.search(query_embedding, k=analog_k)

        analog_attractors = {
            'source': 'hnsw_search',
            'method': 'cosine_similarity',
            'nodes': [
                {
                    'node_id': node_id,
                    'distance': distance,
                    'similarity': 1.0 - distance,
                    'weight': self.config.similarity_weight
                }
                for distance, node_id in hnsw_results
            ]
        }

        # Symbolic attractors (Query Decomposition)
        symbolic_attractors = {'nodes': []}
        if decompose_query:
            entities = self.decompose_query(query)
            matched_nodes = self.accurate_search(entities)

            symbolic_attractors = {
                'source': 'query_decomposition',
                'extracted_entities': entities,
                'nodes': [
                    {
                        'node_id': node_id,
                        'weight': self.config.accuracy_weight
                    }
                    for node_id in matched_nodes
                ]
            }

        # Combined personalization
        personalization = {}
        for node in analog_attractors['nodes']:
            personalization[node['node_id']] = node['weight']
        for node in symbolic_attractors['nodes']:
            personalization[node['node_id']] = node['weight']

        return {
            'analog': analog_attractors,
            'symbolic': symbolic_attractors,
            'combined': {
                'personalization_vector': personalization,
                'total_unique_nodes': len(personalization)
            }
        }
```

---

### 1.2 Compare Attractors

```http
POST /introspection/attractors/compare
```

**Request Body**:
```json
{
  "queries": [
    "What did Emily Roberts research?",
    "Tell me about renewable energy research"
  ],
  "show_overlap": true,
  "show_divergence": true
}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "queries": [...],
    "attractors": {
      "query_1": { ... },
      "query_2": { ... }
    },
    "comparison": {
      "overlap": {
        "common_nodes": ["sem_unit_456"],
        "overlap_count": 1,
        "overlap_percentage": 1.96,
        "jaccard_similarity": 0.019
      },
      "divergence": {
        "unique_to_query_1": ["entity_456", "sem_unit_123", ...],
        "unique_to_query_2": ["entity_999", ...]
      },
      "similarity_metrics": {
        "attractor_cosine_similarity": 0.45,
        "query_embedding_similarity": 0.67
      }
    }
  }
}
```

---

### 1.3 Attractor Visualization

```http
GET /introspection/attractors/{query_id}/visualization
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "visualization": {
      "type": "graph",
      "nodes": [
        {
          "id": "query_node",
          "label": "What did Emily Roberts research?",
          "type": "query",
          "size": 20,
          "color": "#FF5722"
        },
        {
          "id": "sem_unit_123",
          "label": "Dr. Emily Roberts presented...",
          "type": "analog_attractor",
          "size": 15,
          "color": "#2196F3",
          "weight": 1.0
        },
        {
          "id": "entity_456",
          "label": "EMILY ROBERTS",
          "type": "symbolic_attractor",
          "size": 18,
          "color": "#4CAF50",
          "weight": 2.0
        }
      ],
      "edges": [
        {
          "source": "query_node",
          "target": "sem_unit_123",
          "type": "analog",
          "similarity": 0.766
        },
        {
          "source": "query_node",
          "target": "entity_456",
          "type": "symbolic",
          "match": "exact"
        }
      ]
    },
    "layout": "force_directed"
  }
}
```

---

## 2. Concepts & Networks API

### 2.1 Get Concept Network

```http
GET /introspection/concepts/{concept_id}
```

**Query Parameters**:
- `depth` (integer, default: 2) - Глубина exploration
- `max_neighbors` (integer, default: 20) - Макс соседей
- `include_embeddings` (boolean, default: false)

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "concept": {
      "id": "entity_456",
      "type": "entity",
      "content": "EMILY ROBERTS",
      "weight": 12,
      "degree": 45,
      "embedding": [0.123, ...],
      "attributes": [
        {
          "id": "attr_789",
          "content": "Dr. Emily Roberts is a leading researcher in renewable energy..."
        }
      ]
    },
    "network": {
      "immediate_neighbors": [
        {
          "node_id": "sem_unit_123",
          "type": "semantic_unit",
          "relation": "mentioned_in",
          "edge_weight": 1.0,
          "distance_from_concept": 1,
          "content": "Dr. Emily Roberts presented research..."
        },
        {
          "node_id": "rel_789",
          "type": "relationship",
          "relation": "has_relationship",
          "edge_weight": 1.0,
          "distance_from_concept": 1,
          "content": "EMILY ROBERTS, attended, CONFERENCE"
        }
      ],
      "communities": [
        {
          "community_id": "comm_42",
          "modularity": 0.56,
          "size": 234,
          "concept_centrality_in_community": 0.89,
          "high_level_elements": [
            {
              "id": "hle_1",
              "title": "Renewable Energy Research",
              "description": "Recent breakthroughs in solar panel efficiency..."
            }
          ]
        }
      ],
      "related_concepts": [
        {
          "concept_id": "entity_999",
          "type": "entity",
          "content": "RENEWABLE ENERGY",
          "relation_type": "co_occurrence",
          "path_length": 2,
          "similarity": 0.78,
          "shared_neighbors": 12,
          "connection_strength": 0.85
        },
        {
          "concept_id": "entity_1234",
          "type": "entity",
          "content": "SOLAR PANEL EFFICIENCY",
          "relation_type": "research_topic",
          "path_length": 2,
          "similarity": 0.65,
          "shared_neighbors": 8,
          "connection_strength": 0.72
        }
      ]
    },
    "statistics": {
      "total_neighbors": 45,
      "neighbor_types": {
        "semantic_unit": 20,
        "relationship": 15,
        "attribute": 1,
        "high_level_element": 9
      },
      "avg_edge_weight": 1.05,
      "clustering_coefficient": 0.23
    }
  },
  "links": {
    "subgraph": "/graph/subgraph?method=ego&center=entity_456&radius=2",
    "similar_concepts": "/introspection/concepts/similar/entity_456"
  }
}
```

**Implementation**:
```python
# NodeRAG/search/introspection.py
class Introspection:
    def get_concept_network(self, concept_id, depth=2, max_neighbors=20):
        """Get concept network with related concepts"""

        # Basic concept info
        concept = {
            'id': concept_id,
            'type': self.G.nodes[concept_id].get('type'),
            'content': self.mapper.get(concept_id, 'context'),
            'weight': self.G.nodes[concept_id].get('weight'),
            'degree': self.G.degree(concept_id)
        }

        # Immediate neighbors
        neighbors = []
        for neighbor_id in list(self.G.neighbors(concept_id))[:max_neighbors]:
            neighbors.append({
                'node_id': neighbor_id,
                'type': self.G.nodes[neighbor_id].get('type'),
                'edge_weight': self.G[concept_id][neighbor_id].get('weight', 1.0),
                'content': self.mapper.get(neighbor_id, 'context')
            })

        # Related concepts (via paths and embeddings)
        related_concepts = self.find_related_concepts(concept_id, depth=depth)

        return {
            'concept': concept,
            'network': {
                'immediate_neighbors': neighbors,
                'related_concepts': related_concepts
            }
        }

    def find_related_concepts(self, concept_id, depth=2):
        """Find related concepts via paths and semantic similarity"""
        related = []

        # 1. Via graph paths
        for node_id in self.G.nodes():
            if node_id == concept_id:
                continue
            if self.G.nodes[node_id].get('type') not in ['entity', 'high_level_element_title']:
                continue

            try:
                path_length = nx.shortest_path_length(self.G, concept_id, node_id)
                if path_length <= depth:
                    # Calculate similarity if embeddings available
                    similarity = None
                    if concept_id in self.mapper.embeddings and node_id in self.mapper.embeddings:
                        emb1 = self.mapper.embeddings[concept_id]
                        emb2 = self.mapper.embeddings[node_id]
                        similarity = 1.0 - cosine(emb1, emb2)

                    related.append({
                        'concept_id': node_id,
                        'type': self.G.nodes[node_id].get('type'),
                        'content': self.mapper.get(node_id, 'context'),
                        'path_length': path_length,
                        'similarity': similarity
                    })
            except nx.NetworkXNoPath:
                pass

        # Sort by similarity
        related.sort(key=lambda x: x['similarity'] or 0, reverse=True)

        return related[:20]
```

---

### 2.2 Concept Path

```http
POST /introspection/concepts/path
```

**Request Body**:
```json
{
  "from_concept": "entity_456",
  "to_concept": "entity_999",
  "max_length": 5,
  "include_intermediate_concepts": true
}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "from": {
      "id": "entity_456",
      "content": "EMILY ROBERTS"
    },
    "to": {
      "id": "entity_999",
      "content": "RENEWABLE ENERGY"
    },
    "path": {
      "nodes": [
        {
          "id": "entity_456",
          "type": "entity",
          "content": "EMILY ROBERTS",
          "position": 0
        },
        {
          "id": "sem_unit_123",
          "type": "semantic_unit",
          "content": "Dr. Emily Roberts presented research on renewable energy...",
          "position": 1,
          "role": "connector"
        },
        {
          "id": "entity_999",
          "type": "entity",
          "content": "RENEWABLE ENERGY",
          "position": 2
        }
      ],
      "edges": [
        {
          "source": "entity_456",
          "target": "sem_unit_123",
          "weight": 1.0,
          "relation": "mentioned_in"
        },
        {
          "source": "sem_unit_123",
          "target": "entity_999",
          "weight": 1.0,
          "relation": "mentions"
        }
      ],
      "length": 2,
      "total_weight": 2.0
    },
    "semantic_journey": {
      "start_embedding": [0.123, ...],
      "end_embedding": [0.456, ...],
      "direct_similarity": 0.78,
      "path_coherence": 0.85,
      "intermediate_similarities": [0.82, 0.88]
    }
  }
}
```

---

### 2.3 Expand Concept Network

```http
POST /introspection/concepts/expand
```

**Request Body**:
```json
{
  "seed_concepts": ["entity_456", "entity_999"],
  "expansion_method": "similar",
  "k": 10,
  "filters": {
    "types": ["entity", "high_level_element_title"],
    "min_similarity": 0.6
  }
}
```

**Expansion Methods**:
- `similar` - По semantic similarity
- `connected` - По graph connectivity
- `community` - По community membership
- `hybrid` - Комбинация всех методов

**Response**: Expanded concept network

---

## 3. Related Queries API

### 3.1 Generate Related Questions

```http
POST /introspection/related-queries
```

**Request Body**:
```json
{
  "query": "What did Emily Roberts research?",
  "method": "llm",
  "k": 5,
  "include_attractors": true
}
```

**Methods**:
- `llm` - LLM generation на основе context
- `graph` - Graph-based (related concepts)
- `hybrid` - Комбинация LLM + graph

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "original_query": "What did Emily Roberts research?",
    "related_queries": [
      {
        "query": "What were the main findings of Emily Roberts' renewable energy research?",
        "relation": "refinement",
        "confidence": 0.92,
        "rationale": "Focuses on specific research outcomes",
        "predicted_attractors": {
          "analog_overlap": 0.78,
          "symbolic_overlap": 1.0,
          "new_symbolic": ["FINDINGS"]
        }
      },
      {
        "query": "Which conferences did Emily Roberts attend in 2024?",
        "relation": "tangent",
        "confidence": 0.85,
        "rationale": "Explores related aspect (conference attendance)",
        "predicted_attractors": {
          "analog_overlap": 0.65,
          "symbolic_overlap": 0.5,
          "new_symbolic": ["CONFERENCES", "2024"]
        }
      },
      {
        "query": "Who else has researched renewable energy efficiency?",
        "relation": "generalization",
        "confidence": 0.88,
        "rationale": "Broadens scope to other researchers",
        "predicted_attractors": {
          "analog_overlap": 0.72,
          "symbolic_overlap": 0.5,
          "new_symbolic": ["RESEARCHERS"]
        }
      },
      {
        "query": "What is solar panel efficiency?",
        "relation": "background",
        "confidence": 0.79,
        "rationale": "Provides context for research topic",
        "predicted_attractors": {
          "analog_overlap": 0.58,
          "symbolic_overlap": 0.0,
          "new_symbolic": ["SOLAR PANEL EFFICIENCY"]
        }
      },
      {
        "query": "What partnerships did Emily Roberts establish in Europe?",
        "relation": "specific_aspect",
        "confidence": 0.83,
        "rationale": "Focuses on collaboration aspect",
        "predicted_attractors": {
          "analog_overlap": 0.61,
          "symbolic_overlap": 0.5,
          "new_symbolic": ["PARTNERSHIPS", "EUROPE"]
        }
      }
    ],
    "generation_metadata": {
      "method": "llm",
      "model": "gpt-4o",
      "context_used": true,
      "attractors_analyzed": true
    }
  }
}
```

**Implementation**:
```python
# NodeRAG/search/introspection.py
class Introspection:
    async def generate_related_queries(self, query, method='llm', k=5):
        """Generate related queries"""

        if method == 'llm':
            # Get attractors and context
            attractors = self.get_attractors(query)
            context = self.assemble_context_from_attractors(attractors)

            # LLM prompt for related queries
            prompt = f"""
Given the query: "{query}"

And the following context from the knowledge base:
{context}

Generate {k} related questions that:
1. Refine the original query (more specific)
2. Explore tangential aspects
3. Generalize the query (broader scope)
4. Provide background context
5. Focus on specific aspects

For each question, explain the relation type and rationale.

Output as JSON array with structure:
[
  {{
    "query": "...",
    "relation": "refinement|tangent|generalization|background|specific_aspect",
    "rationale": "..."
  }}
]
"""

            response = await self.config.API_client({'query': prompt})

            return response['related_queries']

        elif method == 'graph':
            # Graph-based related queries
            # 1. Get concept network
            # 2. Find related concepts
            # 3. Generate queries from related concepts
            pass
```

---

### 3.2 Query Suggestions (Autocomplete)

```http
POST /introspection/query-suggestions
```

**Request Body**:
```json
{
  "partial_query": "What did Emily",
  "k": 5
}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "partial_query": "What did Emily",
    "suggestions": [
      {
        "suggestion": "What did Emily Roberts research?",
        "confidence": 0.95,
        "entity_matched": "EMILY ROBERTS",
        "common_pattern": "What did [ENTITY] [ACTION]?"
      },
      {
        "suggestion": "What did Emily Roberts present at the conference?",
        "confidence": 0.87,
        "entity_matched": "EMILY ROBERTS",
        "related_concepts": ["CONFERENCE"]
      }
    ]
  }
}
```

---

### 3.3 Query Refinements

```http
POST /introspection/query-refinements
```

**Request Body**:
```json
{
  "query": "Tell me about renewable energy",
  "refinement_type": "all"
}
```

**Refinement Types**:
- `temporal` - Добавить временные ограничения
- `spatial` - Добавить локационные ограничения
- `specificity` - Увеличить специфичность
- `all` - Все типы

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "original_query": "Tell me about renewable energy",
    "refinements": [
      {
        "refined_query": "Tell me about renewable energy research in 2024",
        "refinement_type": "temporal",
        "added_constraint": "2024",
        "expected_improvement": {
          "precision": "+15%",
          "results_reduction": "~30%"
        }
      },
      {
        "refined_query": "Tell me about solar renewable energy",
        "refinement_type": "specificity",
        "added_constraint": "solar",
        "expected_improvement": {
          "precision": "+25%",
          "results_reduction": "~50%"
        }
      }
    ]
  }
}
```

---

## Связанные документы

- [API Architecture](./api-architecture.md)
- [Graph API](./graph-api.md)
- [Vector API](./vector-api.md)
- [Search Trajectory API](./search-trajectory-api.md)

---

**Версия**: 1.0
**Последнее обновление**: 2025-11-14
