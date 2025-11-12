# Search and Retrieval 🔍

## Обзор

Этот документ описывает стратегии поиска и retrieval в NodeRAG, объясняя как различные типы узлов используются на каждом этапе, multi-level retrieval подход, и примеры search flows.

---

## Search Architecture

### Hybrid Search Strategy

NodeRAG использует **triple hybrid search**:

1. **Vector Similarity Search** (HNSW) - semantic matching
2. **Exact Match Search** (accurate search) - keyword/entity matching
3. **Graph Traversal** (PPR) - структурированная навигация

```
Query
  ↓
  ├─→ [Vector Search] → Top-k similar nodes (HNSW)
  ├─→ [Exact Match] → Matched entities/titles
  └─→ [Graph Search] → PPR diffusion
         ↓
    [Combined Results] → Ranked retrieval
```

---

## Stage 1: Vector Similarity Search (HNSW)

### Процесс

```python
# 1. Query embedding
query_embedding = embedding_client.request(query)

# 2. HNSW search
hnsw_results = hnsw_index.search(
    query_embedding,
    k=config.HNSW_results  # default: 20-50
)

# Results: [(node_hash, cosine_distance), ...]
```

### Searchable Node Types

**С embeddings**:
- ✅ Semantic Units - primary target
- ✅ Attributes - rich context
- ✅ High-Level Elements - thematic search
- ✅ Text Units - broader context

**Без embeddings** (не участвуют в vector search):
- ❌ Entities - используются в accurate search
- ❌ Relationships - используются в graph traversal
- ❌ Documents - organizational metadata

### Example

**Query**: "What research is being done on solar panel efficiency?"

**HNSW Results**:
```python
[
    ('semantic_unit_hash_42', 0.15),  # High similarity
    ('attribute_hash_7', 0.18),
    ('semantic_unit_hash_103', 0.22),
    ('high_level_element_hash_5', 0.25),
    ...
]
```

**Interpretation**:
- Lower distance = higher similarity
- Semantic units доминируют (most numerous, focused)
- Attributes дают comprehensive context
- High-level elements для thematic matching

---

## Stage 2: Exact Match Search (Accurate Search)

### Процесс

```python
# 1. Query decomposition
decomposed_query = LLM_client({
    'query': decompose_prompt.format(query=query),
    'response_format': decomposed_text_json_schema
})

# decomposed_query = {'elements': ['SOLAR PANEL EFFICIENCY', 'RESEARCH']}

# 2. Exact/fuzzy match в entities и titles
accurate_results = []

for element in decomposed_query['elements']:
    # Regex pattern для whole-word match
    pattern = re.compile(r'\b' + re.escape(element.lower()) + r'\b')

    # Search in entities
    entity_matches = [
        hash_id for hash_id, context in entity_id_to_text.items()
        if pattern.search(context.lower())
    ]

    # Search in high-level element titles
    title_matches = [
        hash_id for hash_id, context in title_id_to_text.items()
        if pattern.search(context.lower())
    ]

    accurate_results.extend(entity_matches + title_matches)
```

### Query Decomposition

**Purpose**: Извлечь key entities/concepts из query

**Example**:
```
Query: "What did Dr. Emily Roberts research on renewable energy?"

Decomposed:
{
  "elements": [
    "DR. EMILY ROBERTS",
    "RENEWABLE ENERGY",
    "RESEARCH"
  ]
}
```

### Searchable Node Types

**Entities**:
- Имена людей, организаций
- Места, даты
- Концепты

**High-Level Element Titles**:
- Тематические названия
- Broad concepts

### Example

**Query**: "Dr. Emily Roberts renewable energy research"

**Decomposed**: `["DR. EMILY ROBERTS", "RENEWABLE ENERGY", "RESEARCH"]`

**Accurate Results**:
```python
[
    'entity_hash_DR_EMILY_ROBERTS',
    'entity_hash_RENEWABLE_ENERGY',
    'he_title_hash_RENEWABLE_ENERGY_RESEARCH'
]
```

---

## Stage 3: Graph Search (PPR)

### Personalized PageRank (PPR)

**Концепция**: Random walk с restart, starting от персонализированного набора узлов

```python
# Personalization vector
personalization = {}

# От HNSW results
for node_hash in hnsw_results:
    personalization[node_hash] = config.similarity_weight  # default: 1.0

# От accurate results
for node_hash in accurate_results:
    personalization[node_hash] = config.accuracy_weight  # default: 2.0

# Run PPR
ppr_scores = sparse_PPR.PPR(
    personalization,
    alpha=config.ppr_alpha,  # default: 0.85 (damping factor)
    max_iter=config.ppr_max_iter  # default: 100
)

# Sort by score
ranked_nodes = sorted(ppr_scores.items(), key=lambda x: x[1], reverse=True)
```

### PPR Parameters

**Alpha (damping factor)**:
- `α = 0.85`: Стандартное значение
- Higher α (0.9-0.95): Больше diffusion, wider exploration
- Lower α (0.7-0.8): Меньше diffusion, stay closer к starting points

**Personalization weights**:
- `similarity_weight`: Вес для HNSW results (semantic similarity)
- `accuracy_weight`: Вес для accurate results (exact matches)
- Обычно: accuracy > similarity (exact matches важнее)

### Graph Diffusion

**Как работает PPR**:

```
Starting nodes (from HNSW + accurate):
[SU42] (weight=1.0)  [Entity: DR. ROBERTS] (weight=2.0)

↓ Diffusion through graph

[SU42] → [Entity A] → [SU50] → [Entity B] → [Attribute]
                  ↓
         [Relationship] → [Entity C] → [SU51]

[Entity: DR. ROBERTS] → [SU43] → [Entity D] → [High-Level Element]
                     → [Attribute]
                     → [Relationship] → [Entity E]

Final scores (example):
- SU43: 0.25 (direct neighbor of DR. ROBERTS)
- SU50: 0.12 (2 hops from SU42)
- Attribute: 0.18 (neighbor of DR. ROBERTS)
- High-Level Element: 0.10 (thematic connection)
```

**Результат**: Узлы ближе к starting points получают higher scores

---

## Stage 4: Post-Processing и Ranking

### Node Type Filtering

```python
def post_process_top_k(ppr_results, config):
    entity_list = []
    relationship_list = []
    he_title_list = []
    additional_nodes = []

    for node_hash in ppr_results:
        node_type = G.nodes[node_hash]['type']

        match node_type:
            case 'entity':
                if len(entity_list) < config.Enode:  # default: 5
                    entity_list.append(node_hash)

                    # Add attributes if exists
                    if 'attributes' in G.nodes[node_hash]:
                        for attr_hash in G.nodes[node_hash]['attributes']:
                            additional_nodes.append(attr_hash)

            case 'relationship':
                if len(relationship_list) < config.Rnode:  # default: 5
                    relationship_list.append(node_hash)

            case 'high_level_element_title':
                if len(he_title_list) < config.Hnode:  # default: 3
                    he_title_list.append(node_hash)

                    # Add related high-level element
                    he_hash = G.nodes[node_hash]['related_node']
                    additional_nodes.append(he_hash)

            case _:  # semantic_unit, attribute, text_unit, high_level_element
                if len(additional_nodes) < config.cross_node:  # default: 20-30
                    additional_nodes.append(node_hash)

    return {
        'entities': entity_list,
        'relationships': relationship_list,
        'he_titles': he_title_list,
        'content_nodes': additional_nodes
    }
```

### Retrieval Quotas

**Default quotas** (configurable):

| Node Type                | Quota | Purpose |
|--------------------------|-------|---------|
| Content nodes (SU, Attr, HE, TU) | 20-30 | Main content для answer |
| Entities                 | 5     | Key entities для context |
| Relationships            | 5     | Structured knowledge |
| High-Level Element Titles| 3     | Thematic context |

**Attributes**: Automatically added для retrieved entities (если есть)

---

## Multi-Level Retrieval

### Level 1: Direct Search Results

**От HNSW**:
- Semantic units (most common)
- Attributes (comprehensive context)
- High-level elements (themes)

**От Accurate Search**:
- Entities (exact matches)
- High-level element titles (theme names)

### Level 2: Graph Expansion

**PPR Diffusion**:
- Neighbors semantic units
- Connected entities через relationships
- Related attributes
- Community members (high-level elements)

### Level 3: Targeted Expansion

**Entity Attributes**:
```python
for entity_hash in retrieved_entities:
    if 'attributes' in G.nodes[entity_hash]:
        # Automatically include attribute
        attributes.extend(G.nodes[entity_hash]['attributes'])
```

**High-Level Element Content**:
```python
for he_title_hash in retrieved_he_titles:
    he_hash = G.nodes[he_title_hash]['related_node']
    # Include the full high-level element description
    content_nodes.append(he_hash)
```

**Relationship Endpoints**:
- Relationships всегда include source и target entities

---

## Search Strategy by Query Type

### 1. Entity-Focused Queries

**Example**: "Who is Dr. Emily Roberts?"

**Strategy**:
```python
# Accurate search strong match
decomposed = ["DR. EMILY ROBERTS"]
accurate_results = [entity_hash_DR_ROBERTS]  # High weight в personalization

# PPR expands
ppr_results → attributes, semantic units, relationships связанные с entity

# Post-process
results = {
    'entities': [DR. EMILY ROBERTS],
    'attributes': [attribute описывающий DR. ROBERTS],
    'content': [semantic units упоминающие DR. ROBERTS]
}
```

**Key nodes**:
- ✅ Entity (exact match)
- ✅ Attribute (comprehensive description)
- ✅ Semantic Units (facts)
- ✅ Relationships (connections)

### 2. Concept-Focused Queries

**Example**: "What is renewable energy research?"

**Strategy**:
```python
# Vector search strong match
query_emb = embedding("renewable energy research")
hnsw_results = [high_level_element, semantic_units, attributes]

# Accurate search
accurate_results = [he_title_RENEWABLE_ENERGY_RESEARCH]

# PPR combines
ppr_results → thematic nodes, related semantic units

# Post-process
results = {
    'he_titles': [RENEWABLE ENERGY RESEARCH],
    'he_elements': [high-level element description],
    'content': [semantic units в community]
}
```

**Key nodes**:
- ✅ High-Level Elements (theme)
- ✅ Semantic Units (facts в theme)
- ✅ Attributes (important entities в theme)

### 3. Relationship Queries

**Example**: "Where does Dr. Emily Roberts work?"

**Strategy**:
```python
# Accurate search
accurate_results = [entity_hash_DR_ROBERTS]

# PPR expands через relationships
ppr_results → relationship nodes с "works at", target entities

# Post-process
results = {
    'entities': [DR. EMILY ROBERTS, EUROPEAN RESEARCH INSTITUTE],
    'relationships': [works at relationship],
    'content': [semantic units confirming relationship]
}
```

**Key nodes**:
- ✅ Entities (source, target)
- ✅ Relationships (structured connection)
- ✅ Semantic Units (context)

### 4. Broad Exploratory Queries

**Example**: "Tell me about solar panel technology"

**Strategy**:
```python
# Vector search broad matching
hnsw_results = [многие semantic units, attributes, high-level elements]

# Accurate search
accurate_results = [entity_SOLAR_PANEL_*, he_title_SOLAR_TECHNOLOGY]

# PPR aggregates
ppr_results → comprehensive set от multiple starting points

# Post-process
results = {
    'he_titles': [SOLAR TECHNOLOGY],
    'entities': [SOLAR PANEL EFFICIENCY, ...],
    'content': [diverse semantic units, attributes]
}
```

**Key nodes**:
- ✅ All types (comprehensive coverage)
- ✅ High diversity

---

## Role of Each Node Type in Search

### Semantic Units

**Role**: Primary content nodes для facts

**How used**:
- Direct retrieval от HNSW (most common result)
- Expansion через PPR от entities/high-level elements
- Provide specific, atomic facts

**Optimal for**:
- Detailed questions
- Fact verification
- Specific events/statements

### Entities

**Role**: Entry points для graph navigation

**How used**:
- Accurate search (exact match)
- Hub nodes в PPR (high degree → spread weight)
- Connect multiple semantic units

**Optimal for**:
- Entity-centric queries
- Finding all information about entity
- Relationship discovery

### Relationships

**Role**: Structured connections

**How used**:
- Retrieved в top-k от PPR
- Provide explicit (source, relation, target) info
- Multi-hop reasoning

**Optimal for**:
- "Who/what/where" questions
- Connection queries
- Inference chains

### Attributes

**Role**: Comprehensive entity descriptions

**How used**:
- Direct retrieval от HNSW (rich embeddings)
- Auto-added для retrieved entities
- Provide summary context

**Optimal for**:
- Entity overview queries
- Background information
- Comprehensive context

### High-Level Elements

**Role**: Thematic organization

**How used**:
- HNSW retrieval (theme matching)
- Entry points для community exploration
- Provide abstract context

**Optimal for**:
- Broad concept queries
- Thematic exploration
- "What is X about?" questions

### Text Units

**Role**: Broader context fallback

**How used**:
- HNSW retrieval (when semantic units insufficient)
- Provide original text chunks
- Preserve narrative flow

**Optimal for**:
- When details too fragmented
- Need original wording
- Broader paragraph-level context

---

## Example Search Flows

### Example 1: Simple Fact Query

**Query**: "When did Dr. Emily Roberts attend the Paris conference?"

**Flow**:
```
1. Query Embedding
   → embedding_model("When did Dr. Emily Roberts attend Paris conference")

2. HNSW Search
   → Top results: [
        SU: "Dr. Emily Roberts attended conference in Paris, Sept 2024" (distance=0.12),
        SU: "International Conference held in Paris" (distance=0.20),
        ...
     ]

3. Accurate Search
   → Decompose: ["DR. EMILY ROBERTS", "PARIS", "CONFERENCE"]
   → Matches: [entity_DR_ROBERTS, entity_PARIS, entity_CONFERENCE]

4. PPR
   → Personalization: {SU1: 1.0, entity_DR_ROBERTS: 2.0, entity_PARIS: 2.0}
   → Diffusion через граф
   → Top results: [SU1, SU_related_1, entity_DR_ROBERTS, entity_PARIS, ...]

5. Post-Process
   → Content: [SU1, SU_related_1, ...]
   → Entities: [DR. EMILY ROBERTS, PARIS, SEPTEMBER 2024, INTERNATIONAL CONFERENCE]
   → Relationships: [DR. EMILY ROBERTS attended CONFERENCE]

6. Results
   → Semantic unit directly contains answer
   → Entities provide structure
```

### Example 2: Complex Multi-Hop Query

**Query**: "What research does the institute where Dr. Roberts works focus on?"

**Flow**:
```
1. Query Embedding
   → embedding("research institute Dr. Roberts works focus")

2. HNSW Search
   → Top results: [
        Attribute: "European Research Institute focuses on..." (0.15),
        SU: "Dr. Roberts works at European Research Institute" (0.18),
        ...
     ]

3. Accurate Search
   → Decompose: ["DR. ROBERTS", "INSTITUTE", "RESEARCH"]
   → Matches: [entity_DR_ROBERTS, entity_EUROPEAN_RESEARCH_INSTITUTE]

4. PPR (Multi-Hop)
   → Start: entity_DR_ROBERTS (weight=2.0)
   → Hop 1: [works at] relationship → entity_EUROPEAN_RESEARCH_INSTITUTE
   → Hop 2: entity_EUROPEAN_RESEARCH_INSTITUTE → semantic units, attribute
   → Hop 3: semantic units → entity_RENEWABLE_ENERGY, entity_SOLAR_EFFICIENCY

5. Post-Process
   → Entities: [DR. ROBERTS, EUROPEAN RESEARCH INSTITUTE, RENEWABLE ENERGY]
   → Relationships: [DR. ROBERTS works at INSTITUTE, INSTITUTE researches RENEWABLE ENERGY]
   → Content: [Attribute of INSTITUTE, Semantic units about INSTITUTE research]

6. Results
   → Multi-hop reasoning successful
   → Complete answer from combined information
```

### Example 3: Thematic Query

**Query**: "What are the main themes in renewable energy research?"

**Flow**:
```
1. Query Embedding
   → embedding("main themes renewable energy research")

2. HNSW Search
   → Top results: [
        HE: "Renewable Energy Research" theme (0.10),
        HE: "Solar Technology Innovations" theme (0.14),
        SU: various semantic units (0.18-0.25),
        ...
     ]

3. Accurate Search
   → Decompose: ["RENEWABLE ENERGY", "RESEARCH"]
   → Matches: [he_title_RENEWABLE_ENERGY_RESEARCH, entity_RENEWABLE_ENERGY]

4. PPR
   → Start: HE1, HE2, entity_RENEWABLE_ENERGY
   → Diffusion captures community members
   → High-level elements и their content nodes score high

5. Post-Process
   → HE Titles: [RENEWABLE ENERGY RESEARCH, SOLAR TECHNOLOGY, ...]
   → HE Elements: [Full descriptions]
   → Content: [Representative semantic units from communities]
   → Entities: [Key entities in themes]

6. Results
   → Thematic structure clear
   → Multiple themes identified
   → Representative facts for each theme
```

---

## Search Configuration Parameters

### HNSW Parameters

```python
{
    'HNSW_results': 20,        # Top-k от vector search
    'ef_search': 50,           # HNSW search accuracy (higher = more accurate)
    'M': 16,                   # HNSW graph degree (higher = better recall)
}
```

### PPR Parameters

```python
{
    'ppr_alpha': 0.85,         # Damping factor (0-1)
    'ppr_max_iter': 100,       # Max iterations
    'similarity_weight': 1.0,  # Weight для HNSW results
    'accuracy_weight': 2.0,    # Weight для accurate results
}
```

### Retrieval Quotas

```python
{
    'cross_node': 25,          # Main content nodes (SU, Attr, HE, TU)
    'Enode': 5,                # Entity nodes
    'Rnode': 5,                # Relationship nodes
    'Hnode': 3,                # High-level element title nodes
}
```

### Optimization Flags

```python
{
    'unbalance_adjust': True,  # Adjust для unbalanced graph degrees
}
```

---

## Performance Considerations

### HNSW Efficiency

**Advantages**:
- O(log N) search time
- High recall на large datasets
- Constant memory per query

**Trade-offs**:
- Build time: O(N log N)
- Memory: O(N × M) для index
- Accuracy vs speed (ef_search parameter)

### PPR Scalability

**Advantages**:
- Captures global graph structure
- Handles multiple starting points
- Converges быстро (usually <50 iterations)

**Trade-offs**:
- O(E × iterations) complexity (E = edges)
- Требует sparse matrix operations
- Может быть slow на очень больших графах (>1M edges)

### Optimization Strategies

**1. Sparse Graph Representation**:
```python
# Use scipy.sparse для PPR
from scipy.sparse import csr_matrix

# Sparse adjacency matrix
A_sparse = csr_matrix(adjacency_matrix)
```

**2. Early Stopping для PPR**:
```python
# Stop если change < threshold
if np.linalg.norm(scores_new - scores_old) < 1e-6:
    break
```

**3. HNSW Index Caching**:
```python
# Load pre-built index
if os.exists(hnsw_index_path):
    hnsw.load_index(hnsw_index_path)
```

---

## Best Practices

### 1. Balance Hybrid Components

**Recommended weights**:
- Accurate search > HNSW search (exact matches важнее)
- `accuracy_weight = 2.0`, `similarity_weight = 1.0`

### 2. Tune Retrieval Quotas

**Guidelines**:
- More semantic units (20-30) для detailed answers
- Fewer high-level elements (2-3) для focus
- Moderate entities/relationships (5) для structure

### 3. Query Decomposition Quality

**Improve with**:
- Better prompts (examples, instructions)
- Validation (check decomposed elements не empty)
- Fallback (если decomposition fails, use whole query)

### 4. Monitor Search Quality

**Metrics**:
- Recall@k (are relevant nodes retrieved?)
- Precision@k (are retrieved nodes relevant?)
- Answer quality (final answers correct?)

---

## Диагностика Search Issues

### Low Recall

**Symptoms**: Relevant information не retrieved

**Potential causes**:
- HNSW k too small → increase `HNSW_results`
- PPR alpha too low → increase `ppr_alpha` (more diffusion)
- Retrieval quotas too small → increase `cross_node`, `Enode`, etc.

**Fixes**:
```python
config.HNSW_results = 50  # from 20
config.ppr_alpha = 0.90   # from 0.85
config.cross_node = 40    # from 25
```

### Low Precision

**Symptoms**: Many irrelevant nodes retrieved

**Potential causes**:
- HNSW k too large → decrease `HNSW_results`
- PPR alpha too high → decrease `ppr_alpha` (less diffusion)
- Query too broad → improve query decomposition

**Fixes**:
```python
config.HNSW_results = 15  # from 20
config.ppr_alpha = 0.80   # from 0.85
```

### Slow Performance

**Symptoms**: Search takes too long

**Potential causes**:
- PPR max_iter too high → decrease `ppr_max_iter`
- Graph too large → prune low-weight edges
- HNSW ef_search too high → decrease

**Fixes**:
```python
config.ppr_max_iter = 50  # from 100
# Prune graph
G_pruned = remove_low_weight_edges(G, threshold=1)
```

---

## FAQ

**Q: Почему используется PPR вместо BFS/DFS?**
A: PPR учитывает global graph structure и probability распространения. BFS/DFS могут explore irrelevant branches. PPR сбалансирован.

**Q: Можно ли использовать только vector search без PPR?**
A: Да, но теряется структурная информация. PPR улучшает recall через graph connections.

**Q: Как выбрать balance между HNSW и accurate search weights?**
A: Зависит от use case. Для entity-centric queries: higher accuracy weight. Для semantic queries: higher similarity weight.

**Q: Что делать, если query decomposition fails?**
A: Fallback на whole query для accurate search, или skip accurate search и rely на HNSW + PPR.

---

## Связанные Документы

**Node Types**:
- [Semantic Unit Nodes](./03-semantic-unit-nodes.md) - primary content
- [Entity Nodes](./04-entity-nodes.md) - entry points
- [Relationship Nodes](./05-relationship-nodes.md) - structured knowledge
- [Attribute Nodes](./06-attribute-nodes.md) - comprehensive descriptions
- [High-Level Element Nodes](./07-high-level-element-nodes.md) - themes

**Graph**:
- [Node Relationships](./08-node-relationships.md) - graph structure

**Usage**:
- [Answer Generation](./10-answer-generation.md) - using retrieval results

---

[← Назад: Node Relationships](./08-node-relationships.md) | [Следующий: Answer Generation →](./10-answer-generation.md)
