# Attribute Nodes 📋

## Обзор

**Attribute Nodes** представляют LLM-генерированные суммаризации важных entities. Они агрегируют информацию из всех связанных semantic units и relationships, предоставляя comprehensive описание entity в одном компактном тексте.

---

## Характеристики

### Класс
```python
NodeRAG/build/component/attribute.py

class Attribute(Unit_base):
    - raw_context: str       # LLM-сгенерированное описание entity
    - node: str              # Hash ID родительской entity
    - hash_id: str           # SHA256(raw_context)
    - human_readable_id: str # A00001, A00002, ...
```

### Свойства

| Свойство             | Тип       | Описание                              |
|----------------------|-----------|---------------------------------------|
| `raw_context`        | str       | Comprehensive описание entity         |
| `node`               | str       | Hash ID entity, к которой относится   |
| `hash_id`            | str       | SHA256 хеш от raw_context             |
| `human_readable_id`  | str       | Последовательный ID (A00001, ...)     |

### Metadata

| Параметр             | Значение                       |
|----------------------|--------------------------------|
| Type                 | `attribute`                    |
| Has Embedding        | ✅ Да                          |
| Has Weight           | ✅ Да (наследуется от entity)  |
| Searchable           | ✅ Да (through embedding)      |
| Storage              | `attributes.parquet`           |
| In Graph             | ✅ Да                          |

---

## Создание

### Pipeline Stage
**Attribute Generation Pipeline** (Stage 4)

### Процесс

```python
# 1. Определить important entities
node_importance = NodeImportance(G)
important_nodes = node_importance.main()  # K-core + Betweenness

# Критерии:
# - K-core: entity в k-core субграфе
# - Betweenness: high betweenness centrality
# - Weight > 1 (упоминается multiple times)

# 2. Для каждой important entity
for entity_hash in important_nodes:
    # 3. Собрать информацию от соседей
    entity_name = mapper.get(entity_hash, 'context')
    semantic_units = []
    relationships = []

    for neighbor in G.neighbors(entity_hash):
        if G.nodes[neighbor]['type'] == 'semantic_unit':
            semantic_units.append(mapper.get(neighbor, 'context'))
        elif G.nodes[neighbor]['type'] == 'relationship':
            relationships.append(mapper.get(neighbor, 'context'))

    # 4. Формирование промпта
    prompt = attribute_generation_prompt.format(
        entity=entity_name,
        semantic_units='\n'.join(semantic_units),
        relationships='\n'.join(relationships)
    )

    # 5. LLM генерация
    attribute_text = await LLM_client(prompt)

    # 6. Создание Attribute узла
    attribute = Attribute(attribute_text, entity_hash)

    # 7. Добавление в граф
    G.add_node(attribute.hash_id, type='attribute', weight=1)
    G.add_edge(entity_hash, attribute.hash_id, weight=1)
    G.nodes[entity_hash]['attributes'] = [attribute.hash_id]
```

---

## Роль в Графе

### Позиция
```
[Entity: Important] ← только для important entities
    ↓
[Attribute] ← comprehensive summary
```

### Связи

**Входящие связи**:
- **От Entity**: Единственная связь (one-to-one)

**Исходящие связи**:
- Нет (leaf node)

### Graph Properties

```python
# Attribute узел
G.add_node(
    attribute.hash_id,
    type='attribute',
    weight=1  # или weight от parent entity
)

# Связь с entity
G.add_edge(
    entity.hash_id,
    attribute.hash_id,
    weight=1
)

# Маркер в entity node
G.nodes[entity.hash_id]['attributes'] = [attribute.hash_id]
```

---

## Storage Format

### attributes.parquet

```python
{
    'node': 'entity_hash_id',         # родительская entity
    'type': 'attribute',              # тип
    'context': 'LLM summary...',      # raw_context
    'hash_id': 'abc123...',           # hash_id
    'human_readable_id': 'A00042',    # human ID
    'weight': 7,                      # weight от parent entity
    'embedding': [0.1, 0.2, ...]      # vector (после embedding)
}
```

### Пример записи
```json
{
  "node": "d6i1e5f8g0h2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a1b2",
  "type": "attribute",
  "context": "Dr. Emily Roberts is a leading researcher at the European Research Institute, specializing in renewable energy technologies. Her primary focus is on improving solar panel efficiency, where she has achieved significant breakthroughs demonstrating up to 15% performance improvements. She is an active participant in international conferences, including the International Conference on Renewable Energy held in Paris in September 2024, where she presented her latest findings. Her work is widely recognized in the field of sustainable energy research.",
  "hash_id": "f8k3g6h9i2j4k5l6m7n8o9p0q1r2s3t4u5v6w7x8y9z0a1b2c3d4",
  "human_readable_id": "A00042",
  "weight": 15,
  "embedding": [0.123, -0.456, 0.789, ...]
}
```

---

## Important Entities Selection

### K-Core Analysis

**Концепция**: k-core = максимальный субграф, где каждый узел имеет степень ≥ k

```python
k = round(log(n_nodes) * sqrt(avg_degree))

k_core_subgraph = nx.k_core(G, k=k)

important_nodes = [
    node for node in k_core_subgraph.nodes()
    if G.nodes[node]['type'] == 'entity'
    and G.nodes[node]['weight'] > 1
]
```

**Интуиция**: Entities в k-core - это хорошо связанные узлы, вероятно центральные для графа

### Betweenness Centrality

**Концепция**: Мера того, как часто узел лежит на shortest paths между другими узлами

```python
betweenness = nx.betweenness_centrality(G, k=10)
avg_betweenness = mean(betweenness.values())
scale = round(log10(len(betweenness)))

important_nodes_bc = [
    node for node in betweenness
    if betweenness[node] > avg_betweenness * scale
    and G.nodes[node]['type'] == 'entity'
    and G.nodes[node]['weight'] > 1
]
```

**Интуиция**: High betweenness = entity служит "мостом" между разными частями графа

### Combined Selection

```python
important_nodes = list(set(important_nodes_k_core + important_nodes_bc))
```

**Типичный результат**: 5-15% всех entities получают attributes

---

## Attribute Generation Process

### Material Collection

```python
def get_neighbours_material(entity_hash):
    entity_name = mapper.get(entity_hash, 'context')

    # Semantic units, упоминающие entity
    semantic_units = []
    for neighbor in G.neighbors(entity_hash):
        if G.nodes[neighbor]['type'] == 'semantic_unit':
            semantic_units.append(mapper.get(neighbor, 'context'))

    # Relationships, в которых участвует entity
    relationships = []
    for neighbor in G.neighbors(entity_hash):
        if G.nodes[neighbor]['type'] == 'relationship':
            relationships.append(mapper.get(neighbor, 'context'))

    return entity_name, semantic_units, relationships
```

### Prompt Formation

```python
prompt = f"""
Entity: {entity_name}

Semantic Units (facts about this entity):
{'\n'.join(semantic_units)}

Relationships (connections to other entities):
{'\n'.join(relationships)}

Based on the above information, generate a comprehensive summary
of this entity. Include:
- Key characteristics and attributes
- Main activities and roles
- Important relationships
- Significant facts and achievements

Summary:
"""
```

### LLM Generation

```python
attribute_text = await LLM_client(prompt)

# Example output:
"""
Dr. Emily Roberts is a leading researcher at the European Research
Institute, specializing in renewable energy technologies. Her primary
focus is on improving solar panel efficiency, where she has achieved
significant breakthroughs demonstrating up to 15% performance
improvements. She is an active participant in international conferences,
including the International Conference on Renewable Energy held in Paris
in September 2024, where she presented her latest findings. Her work is
widely recognized in the field of sustainable energy research.
"""
```

### Token Limit Handling

Если материал слишком большой:

```python
def get_important_neighbours_material(entity_hash):
    # Ранжировать соседей по importance
    sorted_neighbours = SortedDict()

    for neighbor in G.neighbors(entity_hash):
        # Weight = сумма весов соседей neighbor'а
        weight = sum(
            G.nodes[nn]['weight']
            for nn in G.neighbors(neighbor)
        )
        sorted_neighbours[neighbor] = weight

    # Добавлять по убыванию важности, пока не достигнут token limit
    material = []
    for neighbor in reversed(sorted_neighbours):
        context = mapper.get(neighbor, 'context')
        if not exceeds_token_limit(material + [context]):
            material.append(context)
        else:
            break

    return format_prompt(entity, material)
```

---

## Embeddings

### Генерация

**Pipeline**: Embedding Pipeline (Stage 5)

```python
# Batch processing
for i in range(0, len(attributes), batch_size):
    batch = attributes[i:i+batch_size]
    contexts = [attr['context'] for attr in batch]

    # Embedding model
    embeddings = await embedding_client(contexts)

    # Сохранение
    for attr, emb in zip(batch, embeddings):
        save_embedding(attr['hash_id'], emb)
```

### Особенности

**Длина context**: Attributes обычно длиннее semantic units (100-300 tokens)

**Quality**: Comprehensive summaries → high-quality, information-rich embeddings

---

## Использование в Поиске

### Vector Search

Attributes используются как **rich context nodes** для поиска:

```python
# Query: "Tell me about renewable energy researchers"
query_embedding = embedding_model(query)

# Search in attributes (наряду с semantic units)
results = HNSW_index.search(
    query_embedding,
    k=20
)

# Attributes могут быть в top results
for result in results:
    if result['type'] == 'attribute':
        entity_hash = result['node']
        entity_name = mapper.get(entity_hash, 'context')
        print(f"Found: {entity_name}")
        print(f"Summary: {result['context']}")
```

### Advantages для Search

✅ **Comprehensive**: Одно описание агрегирует множество фактов
✅ **Rich embeddings**: Длинный, detailed текст → better semantic matching
✅ **Entity-centric**: Прямой доступ к полной информации о entity

### Expansion через Graph

```python
# Нашли attribute в search
attribute_hash = top_result['hash_id']

# Получить родительскую entity
entity_hash = G.nodes[attribute_hash]['node']  # или через edge

# Expand к другим связанным узлам
semantic_units = [
    n for n in G.neighbors(entity_hash)
    if G.nodes[n]['type'] == 'semantic_unit'
]

relationships = [
    n for n in G.neighbors(entity_hash)
    if G.nodes[n]['type'] == 'relationship'
]
```

---

## Использование в Answer Generation

### As Primary Context

Attributes предоставляют **compact, comprehensive summaries** для ответов:

```python
# Query: "What does Dr. Emily Roberts research?"

# Retrieval включает attribute
retrieval_results = {
    'entities': ['DR. EMILY ROBERTS'],
    'attributes': [
        {
            'entity': 'DR. EMILY ROBERTS',
            'summary': 'Dr. Emily Roberts is a leading researcher...'
        }
    ],
    'semantic_units': [...]
}

# Context для LLM
context = f"""
About Dr. Emily Roberts:
{attribute_summary}

Additional facts:
{semantic_units}
"""

answer = generate_answer(query, context)
```

### Advantages

✅ **Efficiency**: Один attribute заменяет множество semantic units
✅ **Coherence**: LLM-generated summary уже coherent и well-structured
✅ **Completeness**: Включает all relevant information о entity

### Example

**Query**: "Who is Dr. Emily Roberts and what does she work on?"

**Without Attribute**:
- Retrieval: 10 semantic units о Dr. Roberts
- Context: Фрагментированные факты
- Answer: LLM должен сам агрегировать

**With Attribute**:
- Retrieval: 1 attribute + 2-3 semantic units для деталей
- Context: Coherent summary + specific facts
- Answer: More comprehensive и structured

**Generated Answer**:
```
Dr. Emily Roberts is a leading researcher at the European Research
Institute, where she specializes in renewable energy technologies,
particularly solar panel efficiency. Her work has achieved significant
breakthroughs, demonstrating up to 15% improvements in photovoltaic
performance. In September 2024, she presented her latest findings at
the International Conference on Renewable Energy in Paris, showcasing
her contributions to the field of sustainable energy research.
```

---

## Incrementality

### Проверка существующих Attributes

```python
# При incremental processing
if os.path.exists(config.attributes_path):
    existing_attributes = storage.load_parquet(config.attributes_path)
    existing_entity_hashes = existing_attributes['node'].tolist()

    # Не генерировать attribute повторно
    new_important_nodes = [
        node for node in important_nodes
        if node not in existing_entity_hashes
    ]
```

### Update Strategy

**Current**: Attributes не обновляются, только создаются новые

**Potential future**: При значительных изменениях entity (новые semantic units) - регенерация attribute

---

## Примеры

### Пример 1: Researcher Entity

**Entity**: DR. EMILY ROBERTS (weight=15, important)

**Connected Nodes**:
- 8 Semantic Units
- 3 Relationships
- 1 Attribute

**Attribute**:
```
Dr. Emily Roberts is a leading researcher at the European Research
Institute, specializing in renewable energy technologies. Her primary
focus is on improving solar panel efficiency, where she has achieved
significant breakthroughs demonstrating up to 15% performance
improvements. She is an active participant in international conferences,
including the International Conference on Renewable Energy held in Paris
in September 2024, where she presented her latest findings. Her work is
widely recognized in the field of sustainable energy research.
```

**Usage**:
- Query: "Tell me about Dr. Roberts" → Attribute retrieved directly
- Provides comprehensive answer without fragmenting across multiple semantic units

### Пример 2: Organization Entity

**Entity**: EUROPEAN RESEARCH INSTITUTE (weight=22, important)

**Connected Nodes**:
- 15 Semantic Units
- 7 Relationships
- 1 Attribute

**Attribute**:
```
The European Research Institute is a prominent research organization based
in Europe, focusing on renewable energy and sustainable technologies. The
institute employs leading researchers including Dr. Emily Roberts, and
conducts cutting-edge research in solar panel efficiency and photovoltaic
systems. It actively collaborates with international organizations and
hosts conferences on renewable energy topics.
```

### Пример 3: Concept Entity (No Attribute)

**Entity**: SPECIFIC TECHNICAL TERM (weight=2, not important)

**Connected Nodes**:
- 2 Semantic Units
- 1 Relationship
- 0 Attributes (не достаточно important)

**Reason**: Weight слишком низкий, не в k-core, low betweenness

---

## Метрики

### Coverage

**Typical metrics**:
- Total entities: 1000
- Important entities: 100-150 (10-15%)
- Entities with attributes: 100-150

### Attribute Length

**Average length**: 100-300 tokens
**Range**: 50-500 tokens

**Distribution**:
- Short (<100 tokens): 20%
- Medium (100-200 tokens): 50%
- Long (>200 tokens): 30%

### Retrieval Impact

**With attributes**: ~20-30% reduction в количестве semantic units needed для comprehensive answers

---

## Best Practices

### 1. Prompt Engineering

**Key instructions**:
```
- Generate a comprehensive summary (100-300 words)
- Include key characteristics, activities, relationships
- Be factual and objective
- Synthesize information coherently
- Avoid speculation or information not in the provided context
```

### 2. Important Node Selection

**Balance**:
- Too few: Miss important entities
- Too many: Unnecessary overhead, longer processing

**Recommended**: 10-15% of entities
- K-core + Betweenness обеспечивает good coverage

### 3. Token Management

- Monitor token limits для LLM calls
- Use importance-based filtering для больших entities
- Prioritize high-weight semantic units и relationships

---

## Диагностика

### Проверка attributes

```python
# Загрузить attributes
df = storage.load_parquet(config.attributes_path)

print(f"Total attributes: {len(df)}")
print(f"Unique entities with attributes: {df['node'].nunique()}")

# Length distribution
lengths = df['context'].apply(lambda x: len(x.split()))
print(f"Average length: {lengths.mean():.1f} words")
print(f"Min: {lengths.min()}, Max: {lengths.max()}")
```

### Проверка важности entities

```python
# Entities с attributes vs total entities
entities_df = storage.load_parquet(config.entities_path)
attribute_entity_hashes = set(df['node'].tolist())

total_entities = len(entities_df)
attributed_entities = len(attribute_entity_hashes)

print(f"Attributed entities: {attributed_entities}/{total_entities} ({100*attributed_entities/total_entities:.1f}%)")

# Проверка weight distribution
attributed_weights = entities_df[
    entities_df['hash_id'].isin(attribute_entity_hashes)
]['weight']

print(f"Average weight of attributed entities: {attributed_weights.mean():.1f}")
```

### Проверка graph connections

```python
# Attributes должны быть связаны с entities
for attr_hash in df['hash_id']:
    if not G.has_node(attr_hash):
        print(f"Warning: Attribute {attr_hash} not in graph")
        continue

    neighbors = list(G.neighbors(attr_hash))
    if len(neighbors) != 1:
        print(f"Warning: Attribute {attr_hash} has {len(neighbors)} neighbors (expected 1)")

    # Parent должен быть entity
    parent = neighbors[0]
    if G.nodes[parent]['type'] != 'entity':
        print(f"Warning: Attribute parent is not entity: {G.nodes[parent]['type']}")
```

---

## FAQ

**Q: Почему attributes только для important entities?**
A: Генерация attributes требует LLM calls, что дорого. Important entities дают наибольшую пользу. Для остальных достаточно semantic units.

**Q: Могут ли attributes быть outdated при добавлении новых документов?**
A: Да, потенциально. В будущем можно добавить regeneration при значительных изменениях. Сейчас attributes статичны после создания.

**Q: Как attributes влияют на retrieval quality?**
A: Они улучшают recall для entity-centric queries и reduce fragmentation. LLM получает coherent summaries вместо множества фрагментов.

**Q: Можно ли создать attribute вручную?**
A: Технически да, через прямое добавление в parquet и граф, но не рекомендуется. Лучше улучшить prompts или important node selection.

---

## Связанные Узлы

**Родитель**:
- [Entity Nodes](./04-entity-nodes.md) - attributes описывают important entities

**Используется в**:
- [High-Level Element Nodes](./07-high-level-element-nodes.md) - attributes входят в community summaries
- [Search and Retrieval](./09-search-and-retrieval.md) - как rich context для поиска
- [Answer Generation](./10-answer-generation.md) - как comprehensive summaries

---

[← Назад: Relationship Nodes](./05-relationship-nodes.md) | [Следующий: High-Level Element Nodes →](./07-high-level-element-nodes.md)
