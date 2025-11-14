# Search Trajectory API Reference

## Введение

**Search Trajectory API** предоставляет полную трассировку процесса выполнения query в NodeRAG, включая все промежуточные шаги, временные метки и результаты каждой операции.

**Цель**: Полная transparency поиска для debugging, analysis, visualization.

**Base Path**: `/api/v1/trajectory`

---

## Search Pipeline Steps

### Полная траектория query

```
1. Query Embedding Generation (OpenAI API)
   ↓
2. HNSW k-NN Search (Analog Attractors)
   ↓
3. Query Decomposition (LLM → Entities)
   ↓
4. Accurate Search (Regex Matching → Symbolic Attractors)
   ↓
5. Attractor Construction (Analog + Symbolic)
   ↓
6. PPR Personalization Vector
   ↓
7. PPR Execution (Power Iteration)
   ↓
8. Node Ranking (PPR scores)
   ↓
9. Post-processing & Filtering (Entity, Relationship, HLE)
   ↓
10. Attribute Expansion
   ↓
11. Context Assembly
   ↓
12. LLM Answer Generation
```

---

## Endpoints

### 1. Get Full Trajectory

```http
GET /trajectory/{query_id}
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "query_id": "uuid-1234",
    "query": "What did Emily Roberts research?",
    "total_duration_ms": 2345,
    "steps": [
      {
        "step_id": 1,
        "name": "query_embedding",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:00.000Z",
        "timestamp_end": "2024-11-14T10:00:00.150Z",
        "duration_ms": 150,
        "input": {
          "query": "What did Emily Roberts research?"
        },
        "output": {
          "embedding": [0.123, -0.456, ...],
          "dimension": 1536,
          "norm": 1.0
        },
        "metadata": {
          "model": "text-embedding-3-small",
          "tokens": 6,
          "cost_usd": 0.000015
        }
      },
      {
        "step_id": 2,
        "name": "hnsw_search",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:00.150Z",
        "timestamp_end": "2024-11-14T10:00:00.158Z",
        "duration_ms": 8,
        "input": {
          "embedding": [0.123, ...],
          "k": 50,
          "ef": 200
        },
        "output": {
          "results": [
            {
              "rank": 1,
              "node_id": "sem_unit_123",
              "type": "semantic_unit",
              "distance": 0.234,
              "similarity": 0.766,
              "content": "Dr. Emily Roberts presented research..."
            },
            {
              "rank": 2,
              "node_id": "sem_unit_456",
              "type": "semantic_unit",
              "distance": 0.289,
              "similarity": 0.711,
              "content": "Renewable energy efficiency improvements..."
            }
          ],
          "total_results": 50
        },
        "metadata": {
          "hnsw_M": 64,
          "hnsw_ef": 200,
          "index_size": 150000
        }
      },
      {
        "step_id": 3,
        "name": "query_decomposition",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:00.158Z",
        "timestamp_end": "2024-11-14T10:00:01.158Z",
        "duration_ms": 1000,
        "input": {
          "query": "What did Emily Roberts research?",
          "prompt": "Please break down the following query..."
        },
        "output": {
          "entities": ["EMILY ROBERTS", "RESEARCH"]
        },
        "metadata": {
          "model": "gpt-4o",
          "tokens_input": 45,
          "tokens_output": 8,
          "cost_usd": 0.00023
        }
      },
      {
        "step_id": 4,
        "name": "accurate_search",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:01.158Z",
        "timestamp_end": "2024-11-14T10:00:01.165Z",
        "duration_ms": 7,
        "input": {
          "entities": ["EMILY ROBERTS", "RESEARCH"]
        },
        "output": {
          "matched_nodes": [
            {
              "entity": "EMILY ROBERTS",
              "matched_node_id": "entity_456",
              "type": "entity",
              "content": "EMILY ROBERTS",
              "match_method": "exact"
            }
          ],
          "total_matches": 1
        },
        "metadata": {
          "search_method": "regex",
          "searchable_nodes": 50000
        }
      },
      {
        "step_id": 5,
        "name": "attractor_construction",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:01.165Z",
        "timestamp_end": "2024-11-14T10:00:01.167Z",
        "duration_ms": 2,
        "input": {
          "hnsw_results": ["sem_unit_123", "sem_unit_456", ...],
          "accurate_results": ["entity_456"]
        },
        "output": {
          "attractors": {
            "analog": {
              "source": "hnsw",
              "nodes": ["sem_unit_123", ...],
              "count": 50,
              "weight_per_node": 1.0
            },
            "symbolic": {
              "source": "query_decomposition",
              "nodes": ["entity_456"],
              "count": 1,
              "weight_per_node": 2.0
            }
          },
          "personalization_vector": {
            "sem_unit_123": 1.0,
            "sem_unit_456": 1.0,
            "entity_456": 2.0,
            "_total_nodes": 51,
            "_total_weight": 52.0
          }
        },
        "metadata": {
          "similarity_weight": 1.0,
          "accuracy_weight": 2.0,
          "dual_mode": true
        }
      },
      {
        "step_id": 6,
        "name": "ppr_execution",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:01.167Z",
        "timestamp_end": "2024-11-14T10:00:01.287Z",
        "duration_ms": 120,
        "input": {
          "personalization": {"sem_unit_123": 1.0, ...},
          "alpha": 0.85,
          "max_iter": 100,
          "epsilon": 0.00001
        },
        "output": {
          "iterations": 12,
          "converged": true,
          "top_nodes": [
            {
              "rank": 1,
              "node_id": "entity_456",
              "type": "entity",
              "ppr_score": 0.0234,
              "initial_weight": 2.0
            },
            {
              "rank": 2,
              "node_id": "sem_unit_123",
              "type": "semantic_unit",
              "ppr_score": 0.0189,
              "initial_weight": 1.0
            }
          ],
          "total_ranked_nodes": 500000
        },
        "metadata": {
          "graph_nodes": 500000,
          "graph_edges": 2000000,
          "alpha": 0.85,
          "convergence_iterations": 12
        }
      },
      {
        "step_id": 7,
        "name": "node_filtering",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:01.287Z",
        "timestamp_end": "2024-11-14T10:00:01.295Z",
        "duration_ms": 8,
        "input": {
          "ranked_nodes": ["entity_456", "sem_unit_123", ...],
          "filters": {
            "entity_limit": 10,
            "relationship_limit": 10,
            "hle_limit": 5,
            "cross_node_limit": 50
          }
        },
        "output": {
          "selected_nodes": {
            "entities": [
              {"node_id": "entity_456", "ppr_score": 0.0234}
            ],
            "relationships": [
              {"node_id": "rel_789", "ppr_score": 0.0123}
            ],
            "high_level_elements": [
              {"node_id": "hle_title_1", "ppr_score": 0.0156}
            ],
            "other": [
              {"node_id": "sem_unit_123", "ppr_score": 0.0189}
            ]
          },
          "total_selected": 67
        },
        "metadata": {
          "config": {
            "Enode": 10,
            "Rnode": 10,
            "Hnode": 5,
            "cross_node": 50
          }
        }
      },
      {
        "step_id": 8,
        "name": "attribute_expansion",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:01.295Z",
        "timestamp_end": "2024-11-14T10:00:01.298Z",
        "duration_ms": 3,
        "input": {
          "entities_with_attributes": ["entity_456"]
        },
        "output": {
          "added_attributes": [
            {"entity": "entity_456", "attribute": "attr_789"}
          ],
          "total_attributes_added": 1
        }
      },
      {
        "step_id": 9,
        "name": "context_assembly",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:01.298Z",
        "timestamp_end": "2024-11-14T10:00:01.302Z",
        "duration_ms": 4,
        "input": {
          "selected_nodes": ["entity_456", "attr_789", "sem_unit_123", ...]
        },
        "output": {
          "context": {
            "structured": "------------entity-------------\\n1. EMILY ROBERTS\\n\\n------------attribute-------------\\n1. Dr. Emily Roberts is a leading researcher...\\n\\n",
            "unstructured": "EMILY ROBERTS\\nDr. Emily Roberts is a leading researcher...\\n",
            "node_count": 67,
            "estimated_tokens": 1234
          }
        }
      },
      {
        "step_id": 10,
        "name": "llm_answer_generation",
        "status": "completed",
        "timestamp_start": "2024-11-14T10:00:01.302Z",
        "timestamp_end": "2024-11-14T10:00:03.302Z",
        "duration_ms": 2000,
        "input": {
          "query": "What did Emily Roberts research?",
          "context": "------------entity-------------...",
          "prompt_template": "---Role---\\nYou are a thorough assistant..."
        },
        "output": {
          "answer": "Dr. Emily Roberts has conducted extensive research in renewable energy, particularly focusing on solar panel efficiency improvements. In September 2024, she presented her findings at the International Conference on Renewable Energy in Paris, where she showcased innovative approaches that achieved up to 20% better performance in solar panel systems...",
          "tokens_generated": 156
        },
        "metadata": {
          "model": "gpt-4o",
          "tokens_input": 1280,
          "tokens_output": 156,
          "cost_usd": 0.0048
        }
      }
    ],
    "summary": {
      "total_steps": 10,
      "completed_steps": 10,
      "failed_steps": 0,
      "total_duration_ms": 2345,
      "bottlenecks": [
        {"step": "llm_answer_generation", "duration_ms": 2000, "percentage": 85.3},
        {"step": "query_decomposition", "duration_ms": 1000, "percentage": 42.6},
        {"step": "query_embedding", "duration_ms": 150, "percentage": 6.4}
      ],
      "costs": {
        "embedding_usd": 0.000015,
        "query_decomposition_usd": 0.00023,
        "answer_generation_usd": 0.0048,
        "total_usd": 0.005045
      }
    }
  },
  "links": {
    "visualization": "/trajectory/uuid-1234/visualization",
    "timeline": "/trajectory/uuid-1234/timeline"
  }
}
```

---

### 2. Get Step Details

```http
GET /trajectory/{query_id}/step/{step_id}
```

**Response**: Детали одного шага (из structure выше)

---

### 3. Timeline Visualization

```http
GET /trajectory/{query_id}/timeline
```

**Response** (200 OK):
```json
{
  "status": "success",
  "data": {
    "timeline": {
      "start": "2024-11-14T10:00:00.000Z",
      "end": "2024-11-14T10:00:03.302Z",
      "duration_ms": 3302,
      "events": [
        {
          "time": 0,
          "event": "query_start",
          "label": "Query received"
        },
        {
          "time": 150,
          "event": "embedding_complete",
          "label": "Embedding generated"
        },
        {
          "time": 158,
          "event": "hnsw_complete",
          "label": "HNSW search (50 results)"
        },
        {
          "time": 1158,
          "event": "decomposition_complete",
          "label": "Query decomposed (2 entities)"
        },
        {
          "time": 1287,
          "event": "ppr_complete",
          "label": "PPR converged (12 iterations)"
        },
        {
          "time": 3302,
          "event": "answer_complete",
          "label": "Answer generated"
        }
      ]
    },
    "visualization": {
      "type": "gantt",
      "bars": [
        {
          "name": "Query Embedding",
          "start": 0,
          "end": 150,
          "color": "#4CAF50"
        },
        {
          "name": "HNSW Search",
          "start": 150,
          "end": 158,
          "color": "#2196F3"
        },
        {
          "name": "Query Decomposition (LLM)",
          "start": 158,
          "end": 1158,
          "color": "#FF9800"
        },
        {
          "name": "PPR Execution",
          "start": 1167,
          "end": 1287,
          "color": "#9C27B0"
        },
        {
          "name": "Answer Generation (LLM)",
          "start": 1302,
          "end": 3302,
          "color": "#FF9800"
        }
      ]
    }
  }
}
```

---

### 4. Comparison of Trajectories

```http
POST /trajectory/compare
```

**Request Body**:
```json
{
  "query_ids": ["uuid-1234", "uuid-5678"]
}
```

**Response**: Side-by-side comparison траекторий

---

## Implementation

### Instrumentation Layer

```python
# NodeRAG/search/trajectory_tracker.py (NEW)
import time
import uuid
from typing import Dict, List, Any

class TrajectoryTracker:
    def __init__(self):
        self.query_id = str(uuid.uuid4())
        self.steps = []
        self.current_step = None

    def start_step(self, name: str, input_data: Dict):
        """Start tracking a step"""
        self.current_step = {
            'step_id': len(self.steps) + 1,
            'name': name,
            'status': 'running',
            'timestamp_start': time.time(),
            'input': input_data
        }

    def end_step(self, output_data: Dict, metadata: Dict = None):
        """End tracking current step"""
        if self.current_step:
            self.current_step['timestamp_end'] = time.time()
            self.current_step['duration_ms'] = int(
                (self.current_step['timestamp_end'] - self.current_step['timestamp_start']) * 1000
            )
            self.current_step['status'] = 'completed'
            self.current_step['output'] = output_data
            if metadata:
                self.current_step['metadata'] = metadata

            self.steps.append(self.current_step)
            self.current_step = None

    def get_trajectory(self):
        """Get full trajectory"""
        return {
            'query_id': self.query_id,
            'total_duration_ms': sum(s['duration_ms'] for s in self.steps),
            'steps': self.steps
        }


# Modified NodeSearch with trajectory tracking
class NodeSearch:
    def search_with_trajectory(self, query: str):
        """Search with trajectory tracking"""
        tracker = TrajectoryTracker()

        # Step 1: Query Embedding
        tracker.start_step('query_embedding', {'query': query})
        query_embedding = np.array(
            self.config.embedding_client.request(query),
            dtype=np.float32
        )
        tracker.end_step(
            {'embedding': query_embedding.tolist(), 'dimension': 1536},
            {'model': 'text-embedding-3-small'}
        )

        # Step 2: HNSW Search
        tracker.start_step('hnsw_search', {
            'embedding': query_embedding.tolist(),
            'k': self.config.HNSW_results
        })
        HNSW_results = self.hnsw.search(query_embedding, HNSW_results=self.config.HNSW_results)
        tracker.end_step(
            {'results': list(HNSW_results)[:10]},  # Top 10 for brevity
            {'hnsw_M': 64, 'hnsw_ef': 200}
        )

        # Step 3: Query Decomposition
        tracker.start_step('query_decomposition', {'query': query})
        decomposed_entities = self.decompose_query(query)
        tracker.end_step(
            {'entities': decomposed_entities},
            {'model': 'gpt-4o'}
        )

        # ... остальные шаги

        # Return retrieval + trajectory
        return {
            'retrieval': retrieval,
            'trajectory': tracker.get_trajectory()
        }
```

---

## Связанные документы

- [API Architecture](./api-architecture.md)
- [Reasoning Chain API](./reasoning-chain-api.md)
- [Implementation Mapping](./implementation-mapping.md)

---

**Версия**: 1.0
**Последнее обновление**: 2025-11-14
