# Relationship Nodes 🔗

## Обзор

**Relationship Nodes** представляют направленные отношения между сущностями в формате триплетов (source, relation, target). Они формируют структурированный слой knowledge graph, явно описывая связи между entities и обогащая семантическую навигацию.

---

## Характеристики

### Класс
```python
NodeRAG/build/component/relationship.py

class Relationship(Unit_base):
    - relationship_tuple: List[str]    # [source, relation, target]
    - source: Entity                    # Source entity
    - target: Entity                    # Target entity
    - unique_relationship: frozenset    # frozenset(source_hash, target_hash)
    - raw_context: str                  # "source relation target"
    - text_hash_id: str                 # Родительский Text Unit
    - hash_id: str                      # SHA256(unique_relationship)
    - human_readable_id: str            # R00001, R00002, ...
```

### Свойства

| Свойство               | Тип       | Описание                              |
|------------------------|-----------|---------------------------------------|
| `relationship_tuple`   | List[str] | [source, relation, target]            |
| `source`               | Entity    | Source entity объект                  |
| `target`               | Entity    | Target entity объект                  |
| `unique_relationship`  | frozenset | frozenset({source_hash, target_hash}) |
| `raw_context`          | str       | Конкатенация всех триплетов           |
| `text_hash_id`         | str       | Hash ID родительского Text Unit       |
| `hash_id`              | str       | SHA256 от unique_relationship         |
| `human_readable_id`    | str       | Последовательный ID (R00001, ...)     |

### Metadata

| Параметр             | Значение                       |
|----------------------|--------------------------------|
| Type                 | `relationship`                 |
| Has Embedding        | ❌ Нет                         |
| Has Weight           | ✅ Да (частота упоминания)     |
| Searchable           | ✅ Да (через graph navigation) |
| Storage              | `relationship.parquet`         |
| In Graph             | ✅ Да (as intermediate node)   |

---

## Создание

### Pipeline Stage
**Graph Pipeline** (Stage 3) - extraction из LLM decomposition

### Процесс

```python
# 1. LLM извлекает relationships из semantic unit
for output in text_decomposition_response['Output']:
    relationships = output['relationships']
    # ["DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE"]

    # 2. Парсинг relationship
    for rel_string in relationships:
        rel_parts = rel_string.split(',')
        rel_parts = [part.strip() for part in rel_parts]

        # 3. Валидация: должно быть 3 части
        if len(rel_parts) != 3:
            # Reconstruction через LLM
            rel_parts = await reconstruct_relationship(rel_parts)

        # 4. Создание Relationship объекта
        relationship = Relationship(rel_parts, text_hash_id)
        # Creates: source Entity, target Entity, relationship

        # 5. Дедупликация и добавление в граф
        rel_hash = relationship.hash_id
        if rel_hash in existing_relationships:
            # Уже существует - добавить context
            existing_relationships[rel_hash].add(relationship.raw_context)
        else:
            # Новое relationship
            add_to_graph(relationship)
```

### Graph Structure

```python
# 1. Добавить entities (если не существуют)
if not G.has_node(source.hash_id):
    G.add_node(source.hash_id, type='entity', weight=1)

if not G.has_node(target.hash_id):
    G.add_node(target.hash_id, type='entity', weight=1)

# 2. Добавить relationship node
if not G.has_node(relationship.hash_id):
    G.add_node(relationship.hash_id, type='relationship', weight=1)

# 3. Создать edges: source → relationship → target
G.add_edge(source.hash_id, relationship.hash_id, weight=1)
G.add_edge(relationship.hash_id, target.hash_id, weight=1)
```

---

## Роль в Графе

### Позиция
```
[Entity: SOURCE]
    ↓
[Relationship: "works at"] ← intermediate node
    ↓
[Entity: TARGET]
```

**Relationship как intermediate node** - ключевая особенность NodeRAG:
- Relationship это **узел**, не ребро
- Позволяет хранить context и weight
- Обеспечивает traversal через отношения

### Связи

**Входящие связи**:
- **От Source Entity**: Edge source→relationship

**Исходящие связи**:
- **К Target Entity**: Edge relationship→target

### Graph Pattern

```
[DR. EMILY ROBERTS]
    ↓ (edge)
[Relationship: "works at"]
    ↓ (edge)
[EUROPEAN RESEARCH INSTITUTE]
```

**Multi-hop example**:
```
[DR. EMILY ROBERTS]
    ↓ "works at"
[EUROPEAN RESEARCH INSTITUTE]
    ↓ "located in"
[EUROPE]
    ↓ "contains"
[PARIS]
```

---

## Storage Format

### relationship.parquet

```python
{
    'hash_id': 'abc123...',           # hash_id
    'human_readable_id': 'R00042',    # human ID
    'type': 'relationship',           # тип
    'unique_relationship': [          # frozenset as list
        'entity_source_hash',
        'entity_target_hash'
    ],
    'context': 'source relation target\tsource relation2 target',  # все варианты
    'text_hash_id': 'def456...',      # родительский Text Unit
    'weight': 3                       # частота упоминания
}
```

### Пример записи
```json
{
  "hash_id": "e7j2f6g9h1i3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a1b2c3",
  "human_readable_id": "R00042",
  "type": "relationship",
  "unique_relationship": [
    "d6i1e5f8g0h2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a1b2",
    "f8k3g6h9i2j4k5l6m7n8o9p0q1r2s3t4u5v6w7x8y9z0a1b2c3"
  ],
  "context": "DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE\tDR. EMILY ROBERTS employed by EUROPEAN RESEARCH INSTITUTE",
  "text_hash_id": "b4g9c3d5e6f7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7",
  "weight": 2
}
```

**Интерпретация**:
- Relationship между двумя entities
- 2 различных формулировки (через `\t`)
- Weight = 2 (упоминается дважды)

---

## Unique Relationship

### Концепция

**unique_relationship** = frozenset({source_hash, target_hash})

**Особенность**: Не направленное множество
```python
frozenset({"hash_A", "hash_B"}) == frozenset({"hash_B", "hash_A"})
```

### Дедупликация

```python
# Relationship 1: "A works at B"
rel1 = Relationship(["A", "works at", "B"])
rel1.unique_relationship = frozenset({hash_A, hash_B})

# Relationship 2: "A employed by B" (та же пара entities)
rel2 = Relationship(["A", "employed by", "B"])
rel2.unique_relationship = frozenset({hash_A, hash_B})

# rel1.hash_id == rel2.hash_id (одинаковый unique_relationship)

# При добавлении rel2
if rel2.hash_id in existing:
    # Добавить context к существующему
    existing[rel2.hash_id].add(rel2.raw_context)
    # context = "A works at B\tA employed by B"
```

### Преимущества

✅ **Automatic deduplication**: Разные формулировки одного отношения объединяются
✅ **Context accumulation**: Все варианты сохраняются через `\t`
✅ **Weight tracking**: Weight отражает частоту упоминания пары

---

## Типы Relationships

### По семантике

**1. Employment/Affiliation**:
```
DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE
JOHN SMITH, employed by, GOOGLE INC
```

**2. Location**:
```
EUROPEAN RESEARCH INSTITUTE, located in, EUROPE
PARIS, capital of, FRANCE
CONFERENCE, held in, PARIS
```

**3. Participation**:
```
DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE
CONFERENCE, focused on, RENEWABLE ENERGY
```

**4. Temporal**:
```
CONFERENCE, occurred in, 2024-09
DR. EMILY ROBERTS, presented in, SEPTEMBER 2024
```

**5. Research/Work**:
```
DR. EMILY ROBERTS, researches, SOLAR PANEL EFFICIENCY
STUDY, demonstrates, 15% IMPROVEMENT
```

**6. Hierarchy**:
```
SOLAR PANEL EFFICIENCY, part of, RENEWABLE ENERGY
PARIS, located in, FRANCE
```

---

## Weight и Context Accumulation

### Multiple Mentions

**Scenario**: Одно и то же relationship упоминается в разных semantic units

**First mention**:
```json
{
  "hash_id": "rel_123",
  "context": "DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE",
  "weight": 1
}
```

**Second mention** (другая формулировка):
```python
# Новое relationship с теми же entities
new_rel = "DR. EMILY ROBERTS employed by EUROPEAN RESEARCH INSTITUTE"

# unique_relationship совпадает → дедупликация
# Обновление context
```

**After accumulation**:
```json
{
  "hash_id": "rel_123",
  "context": "DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE\tDR. EMILY ROBERTS employed by EUROPEAN RESEARCH INSTITUTE",
  "weight": 2
}
```

### Parsing Context

```python
# Извлечь все варианты отношения
contexts = relationship['context'].split('\t')
# ["DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE",
#  "DR. EMILY ROBERTS employed by EUROPEAN RESEARCH INSTITUTE"]

# Каждый вариант = одно упоминание в корпусе
```

---

## Relationship Reconstruction

### Проблема

LLM иногда возвращает невалидные relationships:

**Невалидные форматы**:
```
"DR. EMILY ROBERTS, works at"  # Нет target
"works at, EUROPEAN RESEARCH INSTITUTE"  # Нет source
"DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE"  # Нет запятых
"A, B, C, D"  # Более 3 частей
```

### Решение: LLM Reconstruction

```python
async def reconstruct_relationship(parts: List[str]) -> List[str]:
    # Промпт для LLM
    prompt = f"""
    The following relationship is malformed: {parts}

    Please reconstruct it as a valid triplet in the format:
    [source, relation, target]

    Return JSON: {{"source": "...", "relationship": "...", "target": "..."}}
    """

    response = await LLM_client(prompt)

    return [
        response['source'],
        response['relationship'],
        response['target']
    ]
```

**Пример**:
```python
# Input: ["DR. EMILY ROBERTS", "works at"]
# Reconstruction: ["DR. EMILY ROBERTS", "works at", "EUROPEAN RESEARCH INSTITUTE"]
# (LLM infers target from context)
```

---

## Использование в Поиске

### Graph Navigation

Relationships используются для **multi-hop navigation**:

```python
# Query: "Where does Dr. Emily Roberts work?"

# 1. Find entity
entity_hash = find_entity("DR. EMILY ROBERTS")

# 2. Find relationships
for neighbor in G.neighbors(entity_hash):
    if G.nodes[neighbor]['type'] == 'relationship':
        rel_context = mapper.get(neighbor, 'context')

        # 3. Check if "works at" relationship
        if 'works at' in rel_context or 'employed by' in rel_context:
            # 4. Get target entity
            for target in G.neighbors(neighbor):
                if G.nodes[target]['type'] == 'entity':
                    print(f"Works at: {mapper.get(target, 'context')}")
```

### PPR with Relationships

```python
# Relationships учитываются в PPR
personalization = {
    entity_hash: 0.5  # Starting point
}

# PPR traverses через relationships
ppr_scores = sparse_PPR.PPR(personalization, alpha=0.85)

# Relationships распространяют вес от source к target
```

### Retrieval in Results

```python
# После PPR, relationships включаются в top-k
post_process_top_k():
    relationship_list = []

    for node in ppr_results:
        if G.nodes[node]['type'] == 'relationship':
            if len(relationship_list) < config.Rnode:  # default: 5
                relationship_list.append(node)

    # Relationships добавляются в context для answer generation
    retrieval.relationship_list = relationship_list
```

---

## Использование в Answer Generation

### Structured Context

```python
# Relationships предоставляют structured knowledge
context = {
    'entities': [...],
    'relationships': [
        {
            'source': 'DR. EMILY ROBERTS',
            'relation': 'works at',
            'target': 'EUROPEAN RESEARCH INSTITUTE'
        },
        {
            'source': 'DR. EMILY ROBERTS',
            'relation': 'attended',
            'target': 'INTERNATIONAL CONFERENCE'
        }
    ],
    'semantic_units': [...]
}
```

### Relationship-Based Reasoning

**Query**: "What is the connection between Dr. Roberts and renewable energy?"

**Retrieval**:
```
DR. EMILY ROBERTS
  → works at → EUROPEAN RESEARCH INSTITUTE
  → researches → SOLAR PANEL EFFICIENCY
SOLAR PANEL EFFICIENCY
  → part of → RENEWABLE ENERGY
```

**Generated Answer**:
```
Dr. Emily Roberts has a strong connection to renewable energy through
her work. She is employed at the European Research Institute, where
she conducts research on solar panel efficiency, which is a key
component of renewable energy technology.
```

### Advantages

✅ **Explicit connections**: Ясные связи между entities
✅ **Multi-hop reasoning**: Цепочки relationships для сложных queries
✅ **Context richness**: Relationships дополняют semantic units

---

## Примеры

### Пример 1: Simple Relationship

**Extraction**:
```json
{
  "relationships": [
    "DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE"
  ]
}
```

**Stored Relationship**:
```json
{
  "hash_id": "rel_001",
  "human_readable_id": "R00001",
  "unique_relationship": ["entity_hash_A", "entity_hash_B"],
  "context": "DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE",
  "weight": 1
}
```

**Graph**:
```
[E00001: DR. EMILY ROBERTS] → [R00001] → [E00002: EUROPEAN RESEARCH INSTITUTE]
```

### Пример 2: Multiple Formulations

**Extraction from different texts**:
```
Text 1: "DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE"
Text 2: "DR. EMILY ROBERTS, employed by, EUROPEAN RESEARCH INSTITUTE"
Text 3: "DR. EMILY ROBERTS, affiliated with, EUROPEAN RESEARCH INSTITUTE"
```

**After deduplication**:
```json
{
  "hash_id": "rel_001",
  "context": "DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE\tDR. EMILY ROBERTS employed by EUROPEAN RESEARCH INSTITUTE\tDR. EMILY ROBERTS affiliated with EUROPEAN RESEARCH INSTITUTE",
  "weight": 3
}
```

### Пример 3: Multi-hop Chain

**Relationships**:
```
R001: DR. EMILY ROBERTS → works at → EUROPEAN RESEARCH INSTITUTE
R002: EUROPEAN RESEARCH INSTITUTE → located in → EUROPE
R003: EUROPEAN RESEARCH INSTITUTE → researches → RENEWABLE ENERGY
R004: RENEWABLE ENERGY → includes → SOLAR PANEL EFFICIENCY
R005: DR. EMILY ROBERTS → researches → SOLAR PANEL EFFICIENCY
```

**Query**: "What is Dr. Roberts' research area and where is she based?"

**Traversal**: R001 → R002 (location), R001 → R003 → R004 (research)

---

## Метрики

### Distribution

**Typical metrics** (для 1000 relationships):
- Unique relationship pairs: 800-900
- Average weight: 1.2-1.5
- Weight > 1: 20-30%

### Context Accumulation

**Average formulations per relationship**: 1.2-2.0
- Weight = 1: 1 formulation
- Weight = 3: 2-3 formulations (some duplicates)

---

## Best Practices

### 1. Relationship Format

✅ **Хорошо**:
```
"SOURCE, relation, TARGET"
"DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE"
"PARIS, located in, FRANCE"
```

❌ **Плохо**:
```
"SOURCE relation TARGET"  # Нет запятых
"relation, TARGET"  # Нет source
"SOURCE, TARGET"  # Нет relation
```

### 2. Relation Clarity

✅ **Clear relations**:
```
"works at"
"located in"
"researches"
"attended"
```

❌ **Vague relations**:
```
"related to"
"associated with"
"connected to"
```

### 3. Consistency

**Standardize relation names**:
```
Use: "works at"
Not: "employed by", "works for", "works in"
(unless semantically different)
```

**LLM Prompting**:
```
"Use consistent relation names:
- Employment: 'works at'
- Location: 'located in'
- Research: 'researches'
- Participation: 'attended'"
```

---

## Диагностика

### Проверка relationships

```python
# Загрузить relationships
df = storage.load_parquet(config.relationship_path)

print(f"Total relationships: {len(df)}")
print(f"Unique pairs: {df['hash_id'].nunique()}")

# Weight distribution
print(df['weight'].value_counts())

# Relationships с множественными contexts
multi_context = df[df['context'].str.contains('\t')]
print(f"Relationships with multiple formulations: {len(multi_context)}")
```

### Проверка формата

```python
# Валидация triplets
for idx, row in df.iterrows():
    contexts = row['context'].split('\t')
    for context in contexts:
        parts = context.split()
        # Check if reasonable (at least 3 words: source, relation, target)
        if len(parts) < 3:
            print(f"Warning: Short relationship at {row['human_readable_id']}: {context}")
```

### Проверка graph structure

```python
# Relationships должны быть intermediate nodes
for rel_hash in df['hash_id']:
    if not G.has_node(rel_hash):
        print(f"Warning: Relationship {rel_hash} not in graph")
        continue

    neighbors = list(G.neighbors(rel_hash))
    if len(neighbors) != 2:
        print(f"Warning: Relationship {rel_hash} has {len(neighbors)} neighbors (expected 2)")

    # Check types
    types = [G.nodes[n]['type'] for n in neighbors]
    if types.count('entity') != 2:
        print(f"Warning: Relationship {rel_hash} not connected to 2 entities")
```

---

## FAQ

**Q: Почему relationships хранятся как nodes, а не edges?**
A: Nodes позволяют хранить weight, context, и делают traversal гибким. Можно легко найти все relationships для entity или включить relationships в retrieval.

**Q: Как обрабатываются направленные vs ненаправленные отношения?**
A: unique_relationship (frozenset) делает дедупликацию undirected, но raw_context сохраняет направленность. При retrieval можно учитывать направление из context.

**Q: Что делать с relationships, где weight очень высокий?**
A: Высокий weight = часто упоминаемая связь, вероятно важная. Используйте для ranking при retrieval.

**Q: Можно ли создать relationship между одной и той же entity?**
A: Технически да (self-loop), но обычно LLM не должен такое извлекать. Если происходит, это может быть ошибка extraction.

**Q: Как обрабатывать n-ary relationships (более 2 entities)?**
A: Разбивайте на binary relationships. Например: "A, B, C attended conference" → "A attended conference", "B attended conference", "C attended conference".

---

## Связанные Узлы

**Источники (созданы из)**:
- [Semantic Unit Nodes](./03-semantic-unit-nodes.md) - relationships извлекаются из фактов

**Связаны с**:
- [Entity Nodes](./04-entity-nodes.md) - source и target entities

**Используется в**:
- [Node Relationships](./08-node-relationships.md) - формируют структурированный граф
- [Search and Retrieval](./09-search-and-retrieval.md) - для multi-hop navigation
- [Answer Generation](./10-answer-generation.md) - для structured reasoning

---

[← Назад: Entity Nodes](./04-entity-nodes.md) | [Следующий: Attribute Nodes →](./06-attribute-nodes.md)
