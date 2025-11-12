# Node Relationships и Graph Topology 🕸️

## Обзор

Этот документ описывает структуру connections между различными типами узлов в NodeRAG knowledge graph, patterns связей, graph topology, и стратегии traversal.

---

## Иерархия Узлов

### Полная Структура

```
[Document] (не в графе, только storage)
    ↓
[Text Unit] (добавляется позже, Stage 7)
    ↓
[Semantic Unit] ← базовый слой фактов
    ↓
    ├── [Entity] ← сущности
    │   ↓
    │   └── [Attribute] ← описания important entities
    │
    └── [Relationship] ← structured relationships
            ↓
        [Entity] (source/target)

[High-Level Element] ← abstract themes
    ↓
    ├── [Semantic Unit]
    ├── [Attribute]
    └── [High-Level Element Title] ← для exact match
```

---

## Типы Edges

### 1. Parent-Child Relationships

**Document → Text Unit** (не в графе):
- Связь через поле `doc_hash_id` в Text Unit
- Отслеживается в storage, не в graph

**Text Unit → Semantic Unit** (не в графе):
- Связь через поле `text_hash_id` в Semantic Unit
- Отслеживается в storage, не в graph

### 2. Semantic Containment

**Semantic Unit → Entity**:
```python
# Semantic unit упоминает entity
G.add_edge(semantic_unit.hash_id, entity.hash_id, weight=1)
```

**Direction**: Semantic Unit → Entity
**Meaning**: Entity упоминается в semantic unit
**Weight**: Количество упоминаний

### 3. Structured Relationships

**Entity → Relationship → Entity**:
```python
# Source entity
G.add_edge(source_entity.hash_id, relationship.hash_id, weight=1)

# Target entity
G.add_edge(relationship.hash_id, target_entity.hash_id, weight=1)
```

**Pattern**: Source → Relationship (node) → Target
**Relationship как intermediate node** - ключевая особенность
**Weight**: Частота этой связи

### 4. Descriptive Attributes

**Entity → Attribute**:
```python
G.add_edge(entity.hash_id, attribute.hash_id, weight=1)
G.nodes[entity.hash_id]['attributes'] = [attribute.hash_id]
```

**Direction**: Entity → Attribute (one-way)
**Meaning**: Attribute описывает entity
**Only for**: Important entities (k-core, high betweenness)

### 5. Thematic Grouping

**High-Level Element → Semantic Unit/Attribute**:
```python
G.add_edge(semantic_unit.hash_id, he.hash_id, weight=1)
G.add_edge(attribute.hash_id, he.hash_id, weight=1)
```

**Direction**: Content node → High-Level Element
**Meaning**: Node принадлежит к thematic community
**Weight**: Strength of association

**High-Level Element ↔ Title**:
```python
G.add_edge(he.hash_id, he.title_hash_id, weight=1)
G.nodes[he.title_hash_id]['related_node'] = he.hash_id
```

**Direction**: Bidirectional
**Meaning**: Title узел связан с element node

### 6. HNSW Graph Edges

**Similar Nodes Connection** (Stage 8):
```python
# Добавляются в отдельный HNSW граф
# Затем объединяются с основным графом
```

**Between**: Узлы с embeddings (semantic units, attributes, high-level elements, text units)
**Based on**: Vector similarity (cosine distance)
**Purpose**: Улучшение navigation через similarity

---

## Graph Layers

### Layer 1: Document Organization (Storage Only)

```
[Document]
    │
    ├── [Text Unit 1]
    ├── [Text Unit 2]
    └── [Text Unit 3]
```

**Характеристики**:
- Не в NetworkX графе
- Отслеживается через parquet storage
- Служит для tracking источников

### Layer 2: Factual Layer

```
[Text Unit]
    │
[Semantic Unit A]   [Semantic Unit B]   [Semantic Unit C]
    ↓                   ↓                   ↓
[Entities]          [Entities]          [Entities]
```

**Характеристики**:
- Semantic Units - базовые факты
- Entities - извлеченные сущности
- Dense connections между semantic units и entities

### Layer 3: Structured Knowledge

```
[Entity A]
    ↓
[Relationship: "works at"]
    ↓
[Entity B]
```

**Характеристики**:
- Relationships как intermediate nodes
- Формируют structured knowledge graph
- Triple format: (source, relation, target)

### Layer 4: Enhanced Context

```
[Important Entity]
    ↓
[Attribute]
```

**Характеристики**:
- Attributes только для important entities
- LLM-generated summaries
- Rich contextual information

### Layer 5: Thematic Abstraction

```
[High-Level Element: "Theme"]
    │
    ├── [Semantic Unit]
    ├── [Semantic Unit]
    ├── [Attribute]
    └── [Attribute]
```

**Характеристики**:
- Abstract themes из community detection
- Группируют семантически связанные узлы
- Top-level navigation layer

### Layer 6: Similarity Network (HNSW)

```
[Node A] ←→ [Node B] ←→ [Node C]
   ↕           ↕
[Node D]    [Node E]
```

**Характеристики**:
- Based на embedding similarity
- Navigable small-world graph
- Enables efficient ANN search

---

## Connection Patterns

### Pattern 1: Hub-and-Spoke (Entities)

```
        [SU1]
          ↓
[SU2] → [Entity] ← [SU3]
          ↓
        [SU4]
```

**Entities как hubs**:
- High degree nodes
- Connect multiple semantic units
- Центральная роль в navigation

**Metrics**:
- Average degree: 5-15
- Hub entities: >30 connections
- Weight correlates с degree

### Pattern 2: Chain (Relationships)

```
[Entity A] → [Rel: "works at"] → [Entity B] → [Rel: "located in"] → [Entity C]
```

**Multi-hop reasoning**:
- Follow relationship chains
- Transitive connections
- Path-based inference

### Pattern 3: Star (High-Level Elements)

```
            [High-Level Element]
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
    [SU1]       [Attr1]     [SU2]
```

**Thematic grouping**:
- Central theme node
- Multiple content nodes
- Community-based structure

### Pattern 4: Bridge (Important Entities with Attributes)

```
[SU1] → [Important Entity] → [SU2]
              ↓
          [Attribute]
```

**Enhanced nodes**:
- Entity connects semantic units (hub)
- Attribute provides comprehensive context
- Enriched navigation paths

---

## Graph Properties

### Node Types Distribution

**Typical distribution** (для 100 documents, ~500K tokens):

| Type                     | Count    | Percentage |
|--------------------------|----------|------------|
| Semantic Units           | 2000     | 50%        |
| Entities                 | 1000     | 25%        |
| Relationships            | 500      | 12.5%      |
| Text Units               | 200      | 5%         |
| Attributes               | 150      | 3.75%      |
| High-Level Elements      | 100      | 2.5%       |
| High-Level Element Titles| 100      | 2.5%       |

**Total**: ~4000 nodes

### Edge Distribution

**By type**:

| Edge Type                          | Count | Percentage |
|------------------------------------|-------|------------|
| Semantic Unit → Entity             | 8000  | 60%        |
| Entity → Relationship → Entity     | 1000  | 7.5%       |
| Entity → Attribute                 | 150   | 1%         |
| Content → High-Level Element       | 2000  | 15%        |
| High-Level Element ↔ Title         | 100   | 0.75%      |
| HNSW similarity edges              | 2000  | 15%        |

**Total**: ~13000 edges

### Degree Distribution

**Average degrees by type**:

| Node Type              | Average Degree |
|------------------------|----------------|
| Semantic Unit          | 5-8            |
| Entity                 | 8-15           |
| Relationship           | 2 (fixed)      |
| Attribute              | 1 (leaf)       |
| High-Level Element     | 10-50          |
| Text Unit              | 5-10           |

### Graph Metrics

**Global metrics**:
- **Diameter**: 6-10 (small-world property)
- **Average path length**: 3-5
- **Clustering coefficient**: 0.3-0.5 (moderate clustering)
- **Modularity**: 0.4-0.7 (strong community structure)

---

## Traversal Strategies

### Strategy 1: Entity-Centric Traversal

**Use case**: Find all information about specific entity

```python
def entity_centric_traversal(entity_hash, max_hops=2):
    # Hop 1: Direct neighbors
    neighbors_h1 = G.neighbors(entity_hash)

    semantic_units = [
        n for n in neighbors_h1
        if G.nodes[n]['type'] == 'semantic_unit'
    ]

    relationships = [
        n for n in neighbors_h1
        if G.nodes[n]['type'] == 'relationship'
    ]

    attributes = [
        n for n in neighbors_h1
        if G.nodes[n]['type'] == 'attribute'
    ]

    # Hop 2: Extended neighbors (через relationships)
    related_entities = []
    for rel in relationships:
        rel_neighbors = G.neighbors(rel)
        for n in rel_neighbors:
            if G.nodes[n]['type'] == 'entity' and n != entity_hash:
                related_entities.append(n)

    return {
        'facts': semantic_units,
        'relationships': relationships,
        'related_entities': related_entities,
        'summary': attributes
    }
```

### Strategy 2: Multi-Hop Relationship Traversal

**Use case**: Follow relationship chains

```python
def relationship_chain_traversal(start_entity, max_hops=3):
    visited = set()
    current_level = [start_entity]
    chain = []

    for hop in range(max_hops):
        next_level = []

        for entity in current_level:
            if entity in visited:
                continue

            visited.add(entity)

            # Find relationships
            for neighbor in G.neighbors(entity):
                if G.nodes[neighbor]['type'] == 'relationship':
                    rel_context = mapper.get(neighbor, 'context')

                    # Find target entity
                    for target in G.neighbors(neighbor):
                        if G.nodes[target]['type'] == 'entity' and target != entity:
                            chain.append({
                                'source': entity,
                                'relationship': neighbor,
                                'target': target,
                                'context': rel_context
                            })
                            next_level.append(target)

        current_level = next_level

    return chain
```

### Strategy 3: Theme-Based Traversal

**Use case**: Explore thematic communities

```python
def theme_traversal(query):
    # 1. Find relevant high-level elements
    query_emb = embedding_model(query)
    he_results = search_high_level_elements(query_emb, k=3)

    all_content = []

    for he_hash in he_results:
        # 2. Get content nodes in community
        content_nodes = [
            n for n in G.neighbors(he_hash)
            if G.nodes[n]['type'] in ['semantic_unit', 'attribute']
        ]

        # 3. Rank by relevance
        ranked_content = rank_by_similarity(content_nodes, query_emb)

        all_content.extend(ranked_content[:10])

    return all_content
```

### Strategy 4: PPR (Personalized PageRank)

**Use case**: Global graph diffusion от starting points

```python
def ppr_traversal(start_nodes, weights):
    # Personalization vector
    personalization = {
        node: weight
        for node, weight in zip(start_nodes, weights)
    }

    # Run PPR
    ppr_scores = sparse_PPR.PPR(
        personalization,
        alpha=0.85,  # damping factor
        max_iter=100
    )

    # Sort by score
    ranked_nodes = sorted(
        ppr_scores.items(),
        key=lambda x: x[1],
        reverse=True
    )

    return ranked_nodes
```

**Advantages**:
- Global view (не только local neighbors)
- Учитывает graph structure (degree, paths)
- Balances множественные starting points

---

## Graph Topology Patterns

### Small-World Property

**Характеристика**: Короткие paths между узлами, high clustering

**Implications**:
- Efficient navigation (few hops)
- Information быстро распространяется (PPR)
- Localized communities существуют

### Scale-Free Network

**Характеристика**: Power-law degree distribution (несколько hubs, много low-degree nodes)

**Hub nodes**:
- Common entities (DR. EMILY ROBERTS, RENEWABLE ENERGY)
- High-level elements с большими communities
- Important entities с many connections

**Implications**:
- Hub-based navigation efficient
- Vulnerability к hub removal
- Natural importance hierarchy

### Community Structure

**Характеристика**: Dense connections внутри communities, sparse между communities

**Communities в NodeRAG**:
- Detected через Leiden algorithm
- Представлены high-level elements
- Semantic coherence внутри community

**Implications**:
- Thematic organization
- Efficient theme-based retrieval
- Модульная структура графа

---

## Graph Construction Sequence

### Stage 3: Graph Pipeline

```python
# 1. Create semantic units
for su in semantic_units:
    G.add_node(su.hash_id, type='semantic_unit', weight=1)

# 2. Create entities
for entity in entities:
    G.add_node(entity.hash_id, type='entity', weight=1)

# 3. Connect semantic units → entities
for su, entities in su_entity_pairs:
    for entity in entities:
        G.add_edge(su.hash_id, entity.hash_id, weight=1)

# 4. Create relationships (as nodes)
for rel in relationships:
    G.add_node(rel.hash_id, type='relationship', weight=1)
    G.add_edge(source.hash_id, rel.hash_id, weight=1)
    G.add_edge(rel.hash_id, target.hash_id, weight=1)
```

### Stage 4: Attribute Generation

```python
# Add attributes для important entities
for entity in important_entities:
    attribute = generate_attribute(entity)
    G.add_node(attribute.hash_id, type='attribute', weight=1)
    G.add_edge(entity.hash_id, attribute.hash_id, weight=1)
    G.nodes[entity.hash_id]['attributes'] = [attribute.hash_id]
```

### Stage 6: Summary Generation

```python
# Community detection
partition = leiden_community_detection(G)

# Create high-level elements
for community in partition:
    he_list = generate_high_level_elements(community)

    for he in he_list:
        # Element node
        G.add_node(he.hash_id, type='high_level_element', weight=1)

        # Title node
        G.add_node(he.title_hash_id, type='high_level_element_title', weight=1, related_node=he.hash_id)

        # Connect element ↔ title
        G.add_edge(he.hash_id, he.title_hash_id, weight=1)

        # Connect к community nodes
        for node in community:
            G.add_edge(node, he.hash_id, weight=1)
```

### Stage 7: Insert Text Units

```python
# Add text units к графу
for text_unit in text_units:
    G.add_node(text_unit.hash_id, type='text_unit', weight=1)

    # Connect к semantic units
    for su in semantic_units_from_text:
        G.add_edge(text_unit.hash_id, su.hash_id, weight=1)
```

### Stage 8: HNSW Graph

```python
# Build HNSW index
hnsw_index = build_hnsw(all_embeddings)

# Create similarity graph
G_hnsw = nx.Graph()

for node in nodes_with_embeddings:
    neighbors = hnsw_index.knn_query(node.embedding, k=5)

    for neighbor in neighbors:
        G_hnsw.add_edge(node.hash_id, neighbor.hash_id, weight=similarity)

# Merge с основным графом
G_combined = merge_graphs(G, G_hnsw)
```

---

## Best Practices

### 1. Graph Balance

**Avoid**:
- Слишком dense: Performance issues, noise
- Слишком sparse: Disconnected components, poor navigation

**Recommended**:
- Average degree: 5-15
- Diameter: <10
- Largest component: >95% nodes

### 2. Weight Management

**Use weights для**:
- Ranking при traversal
- PPR importance
- Filtering low-weight edges

**Update weights**:
- Инкремент при дедупликации
- Reflect frequency/importance

### 3. Type-Aware Navigation

**Учитывайте типы узлов**:
```python
# Не все edges одинаковы
if neighbor_type == 'attribute':
    # Leaf node, не продолжать traversal
    pass
elif neighbor_type == 'relationship':
    # Intermediate node, продолжить к target
    pass
```

### 4. Pruning Low-Value Edges

**Для efficiency**:
```python
# Remove low-weight edges
threshold = 1  # или динамический
G_pruned = G.copy()
edges_to_remove = [
    (u, v) for u, v, data in G.edges(data=True)
    if data.get('weight', 1) < threshold
]
G_pruned.remove_edges_from(edges_to_remove)
```

---

## Диагностика Graph Structure

### Проверка connectivity

```python
# Largest connected component
components = nx.connected_components(G)
largest_cc = max(components, key=len)

print(f"Largest component: {len(largest_cc)}/{G.number_of_nodes()} nodes ({100*len(largest_cc)/G.number_of_nodes():.1f}%)")

# Должно быть >95%
```

### Проверка degree distribution

```python
degrees = dict(G.degree())
avg_degree = sum(degrees.values()) / len(degrees)

print(f"Average degree: {avg_degree:.2f}")

# Degree distribution by type
for node_type in ['semantic_unit', 'entity', 'relationship']:
    type_nodes = [n for n in G.nodes if G.nodes[n]['type'] == node_type]
    type_degrees = [G.degree(n) for n in type_nodes]

    print(f"{node_type}: avg degree = {np.mean(type_degrees):.2f}")
```

### Проверка edge types

```python
edge_types = {}

for u, v in G.edges():
    u_type = G.nodes[u]['type']
    v_type = G.nodes[v]['type']
    edge_type = f"{u_type} → {v_type}"

    edge_types[edge_type] = edge_types.get(edge_type, 0) + 1

for edge_type, count in sorted(edge_types.items(), key=lambda x: x[1], reverse=True):
    print(f"{edge_type}: {count}")
```

---

## FAQ

**Q: Почему relationships хранятся как nodes, а не edges?**
A: Это позволяет хранить context, weight, и делает их traversable. Можно найти все relationships для entity или включить в retrieval.

**Q: Как graph масштабируется с количеством документов?**
A: Linearly для semantic units и entities. Relationships и high-level elements растут медленнее (дедупликация, communities).

**Q: Можно ли иметь disconnected components?**
A: Теоретически да (documents без shared entities), но редко на практике. Community detection работает лучше на connected графах.

**Q: Как HNSW edges влияют на graph structure?**
A: Добавляют similarity-based connections, improve navigation, но могут increase density. Используйте с осторожностью (k=5-10 neighbors).

---

## Связанные Документы

**Node Types**:
- [Semantic Unit Nodes](./03-semantic-unit-nodes.md)
- [Entity Nodes](./04-entity-nodes.md)
- [Relationship Nodes](./05-relationship-nodes.md)
- [Attribute Nodes](./06-attribute-nodes.md)
- [High-Level Element Nodes](./07-high-level-element-nodes.md)

**Usage**:
- [Search and Retrieval](./09-search-and-retrieval.md) - использует graph structure
- [Answer Generation](./10-answer-generation.md) - использует traversal strategies

---

[← Назад: High-Level Element Nodes](./07-high-level-element-nodes.md) | [Следующий: Search and Retrieval →](./09-search-and-retrieval.md)
