# Entity Nodes 🏷️

## Обзор

**Entity Nodes** представляют именованные сущности (named entities), извлеченные из текста - люди, организации, места, даты, концепты и другие идентифицируемые объекты. Они являются ключевыми узлами связи в графе знаний, соединяя semantic units и relationships.

---

## Характеристики

### Класс
```python
NodeRAG/build/component/entity.py

class Entity(Unit_base):
    - raw_context: str       # Имя сущности (uppercase)
    - text_hash_id: str      # Родительский Text Unit
    - hash_id: str           # SHA256(raw_context)
    - human_readable_id: str # E00001, E00002, ...
```

### Свойства

| Свойство             | Тип       | Описание                              |
|----------------------|-----------|---------------------------------------|
| `raw_context`        | str       | Имя сущности (обычно UPPERCASE)       |
| `text_hash_id`       | str       | Hash ID родительского Text Unit       |
| `hash_id`            | str       | SHA256 хеш от raw_context             |
| `human_readable_id`  | str       | Последовательный ID (E00001, ...)     |

### Metadata

| Параметр             | Значение                       |
|----------------------|--------------------------------|
| Type                 | `entity`                       |
| Has Embedding        | ❌ Нет                         |
| Has Weight           | ✅ Да (частота упоминания)     |
| Searchable           | ✅ Да (через exact match)      |
| Storage              | `entities.parquet`             |
| In Graph             | ✅ Да                          |

---

## Создание

### Pipeline Stage
**Graph Pipeline** (Stage 3) - extraction из LLM decomposition

### Процесс

```python
# 1. LLM извлекает entities из semantic unit
for output in text_decomposition_response['Output']:
    semantic_unit = output['semantic_unit']
    entities = output['entities']  # ["DR. EMILY ROBERTS", "PARIS", ...]

    # 2. Создание Entity объектов
    for entity_name in entities:
        entity = Entity(entity_name, text_hash_id)

        # 3. Добавление в граф с дедупликацией
        if G.has_node(entity.hash_id):
            # Уже существует - инкремент веса
            G.nodes[entity.hash_id]['weight'] += 1
        else:
            # Новая entity
            G.add_node(entity.hash_id, type='entity', weight=1)
```

### Источники Entities

**Прямое извлечение**:
- Из поля `entities` в text decomposition

**Из Relationships**:
- Source и target в relationship tuple создают entity узлы
- Эти entities также добавляются с type='entity'

---

## Роль в Графе

### Позиция
```
[Semantic Unit]
    ↓
[Entity] ← связующий узел
    ↓
    ├── [Semantic Unit] (другой факт с этой entity)
    ├── [Relationship] (участие в отношении)
    └── [Attribute] (описание entity)
```

### Связи

**Входящие связи**:
- **От Semantic Units**: Множественные связи (упоминания в фактах)
- **От Relationships**: Связи source→entity и entity→target
- **От Attributes**: Связь entity→attribute (если important)

**Исходящие связи**:
- **К Semantic Units**: Обратные связи к фактам
- **К Relationships**: Участие в отношениях
- **К Attributes**: Описательные атрибуты

### Graph Structure

```python
# Entity узел
G.add_node(
    entity.hash_id,
    type='entity',
    weight=1  # инкрементируется при повторениях
)

# Связь с semantic unit
G.add_edge(
    semantic_unit.hash_id,
    entity.hash_id,
    weight=1
)

# Если entity важная - добавляется attribute
if entity_is_important:
    G.add_node(attribute.hash_id, type='attribute', weight=1)
    G.add_edge(entity.hash_id, attribute.hash_id, weight=1)
    G.nodes[entity.hash_id]['attributes'] = [attribute.hash_id]
```

### Hub Role

Entities служат **hub nodes** в графе:

```
    [SU1] ← "Dr. Roberts attended conference"
       ↓
   [Entity: DR. EMILY ROBERTS]
       ↓
    [SU2] ← "Dr. Roberts published research"
       ↓
   [Relationship: works at]
       ↓
   [Entity: EUROPEAN RESEARCH INSTITUTE]
```

---

## Storage Format

### entities.parquet

```python
{
    'hash_id': 'abc123...',           # hash_id
    'human_readable_id': 'E00042',    # human ID
    'type': 'entity',                 # тип
    'context': 'DR. EMILY ROBERTS',   # raw_context (имя)
    'text_hash_id': 'def456...',      # родительский Text Unit
    'weight': 7                       # частота упоминания
}
```

### Пример записи
```json
{
  "hash_id": "d6i1e5f8g0h2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a1b2",
  "human_readable_id": "E00042",
  "type": "entity",
  "context": "DR. EMILY ROBERTS",
  "text_hash_id": "b4g9c3d5e6f7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7",
  "weight": 7
}
```

**Интерпретация weight**:
- Weight = 7 означает, что "DR. EMILY ROBERTS" упоминается в 7 различных semantic units
- Высокий weight → важная entity в корпусе документов

---

## Типы Entities

### По категориям

**1. Персоны (People)**:
```
DR. EMILY ROBERTS
JOHN SMITH
PROFESSOR ANDERSON
```

**2. Организации (Organizations)**:
```
EUROPEAN RESEARCH INSTITUTE
UNITED NATIONS
GOOGLE INC
```

**3. Места (Locations)**:
```
PARIS
UNITED STATES
CALIFORNIA
```

**4. Даты и времена (Temporal)**:
```
2024-09
SEPTEMBER 2024
2024-09-15
```

**5. Концепты (Concepts)**:
```
RENEWABLE ENERGY
SOLAR PANEL EFFICIENCY
ARTIFICIAL INTELLIGENCE
```

**6. События (Events)**:
```
INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY
WORLD WAR II
PARIS SUMMIT
```

### Нормализация

**Case**: Обычно UPPERCASE для консистентности
```python
"Dr. Emily Roberts" → "DR. EMILY ROBERTS"
"paris" → "PARIS"
```

**Формат дат**: Стандартизация
```python
"September 2024" → "2024-09"
"15th of September 2024" → "2024-09-15"
```

---

## Weight: Частота Упоминания

### Механизм

```python
# При каждом упоминании entity
if G.has_node(entity.hash_id):
    G.nodes[entity.hash_id]['weight'] += 1
else:
    G.add_node(entity.hash_id, type='entity', weight=1)
```

### Интерпретация

| Weight | Значение |
|--------|----------|
| 1-2    | Редкое упоминание, возможно не центральная entity |
| 3-10   | Умеренно важная entity |
| 10-50  | Важная entity, центральная для некоторых тем |
| >50    | Очень важная entity, ключевая для всего корпуса |

### Использование Weight

**1. Определение Important Nodes**:
```python
# Для attribute generation
important_entities = [
    entity for entity in entities
    if entity['weight'] > 1  # Только entities с multiple mentions
]
```

**2. Ranking в результатах поиска**:
```python
# При равной релевантности, приоритет higher weight
results.sort(key=lambda x: x['weight'], reverse=True)
```

**3. Визуализация**:
```python
# Размер узла пропорционален weight
node_size = entity['weight'] * 10
```

---

## Important Entities и Attributes

### Критерии Important Entity

**K-Core Analysis**:
- Entity в k-core субграфе (k = log(n) * sqrt(avg_degree))
- Weight > 1

**Betweenness Centrality**:
- Высокая betweenness centrality
- Weight > 1

### Attribute Generation

```python
# Только для important entities
if entity in important_entities:
    # Собрать информацию о entity
    neighbor_info = []
    for neighbor in G.neighbors(entity.hash_id):
        if G.nodes[neighbor]['type'] == 'semantic_unit':
            neighbor_info.append(neighbor['context'])

    # LLM генерирует attribute (summary)
    attribute = generate_attribute(entity, neighbor_info)

    # Добавляем attribute в граф
    G.add_node(attribute.hash_id, type='attribute', weight=1)
    G.add_edge(entity.hash_id, attribute.hash_id, weight=1)
    G.nodes[entity.hash_id]['attributes'] = [attribute.hash_id]
```

**Пример**:
```
Entity: DR. EMILY ROBERTS (weight=15, important)

Attribute: "Dr. Emily Roberts is a leading researcher at the
European Research Institute, specializing in renewable energy and
solar panel technology. She has published multiple papers on
photovoltaic efficiency improvements and regularly presents at
international conferences."
```

---

## Использование в Поиске

### Accurate Search (Exact Match)

**Primary use case**: Точное совпадение имен и терминов

```python
# Query: "Dr. Emily Roberts"
query_entities = decompose_query(query)  # ["DR. EMILY ROBERTS"]

# Exact match в entities
accurate_results = []
for entity_name in query_entities:
    pattern = re.compile(r'\b' + re.escape(entity_name.lower()) + r'\b')
    matches = [
        entity_hash for entity_hash, entity_context in entities.items()
        if pattern.search(entity_context.lower())
    ]
    accurate_results.extend(matches)

# Используем как personalization для PPR
personalization = {entity_hash: accuracy_weight for entity_hash in accurate_results}
```

### Graph Navigation Entry Points

Entities служат **entry points** для graph traversal:

```python
# После нахождения entity
entity_hash = accurate_search("DR. EMILY ROBERTS")

# Expand к connected nodes
semantic_units = [
    neighbor for neighbor in G.neighbors(entity_hash)
    if G.nodes[neighbor]['type'] == 'semantic_unit'
]

relationships = [
    neighbor for neighbor in G.neighbors(entity_hash)
    if G.nodes[neighbor]['type'] == 'relationship'
]

attributes = G.nodes[entity_hash].get('attributes', [])
```

### PPR (Personalized PageRank)

```python
# Entities как starting points для graph diffusion
personalization = {
    entity1: 0.5,  # от accurate search
    entity2: 0.3,
    entity3: 0.2
}

# PPR распространяет вес по графу
ppr_scores = sparse_PPR.PPR(personalization, alpha=0.85)

# Top-k nodes по PPR score
top_nodes = sorted(ppr_scores.items(), key=lambda x: x[1], reverse=True)[:100]
```

---

## Использование в Answer Generation

### Entity Recognition в Context

```python
# Retrieval результаты включают entities
retrieval_results = {
    'entities': [
        {'name': 'DR. EMILY ROBERTS', 'weight': 15},
        {'name': 'PARIS', 'weight': 8},
        {'name': '2024-09', 'weight': 5}
    ],
    'semantic_units': [...],
    'relationships': [...]
}

# Context для LLM
context = format_context(retrieval_results)
```

### Entity-Centric Answers

**Query**: "Tell me about Dr. Emily Roberts"

**Retrieval**:
1. Exact match: Entity "DR. EMILY ROBERTS"
2. Expand: Attributes, connected semantic units
3. Relationships: "works at EUROPEAN RESEARCH INSTITUTE"

**Context assembly**:
```
Entity: DR. EMILY ROBERTS

Attribute: Dr. Emily Roberts is a leading researcher...

Related Facts:
- In September 2024, Dr. Emily Roberts attended conference in Paris
- Dr. Emily Roberts presented research on solar panel efficiency
- Research demonstrated 15% improvement in efficiency

Relationships:
- works at EUROPEAN RESEARCH INSTITUTE
- presented at INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY
```

**Generated Answer**:
```
Dr. Emily Roberts is a prominent researcher at the European Research
Institute, where she specializes in renewable energy and solar panel
technology. In September 2024, she attended the International Conference
on Renewable Energy in Paris, where she presented her research on
improving solar panel efficiency by 15%.
```

---

## Дедупликация Entities

### Automatic Deduplication

```python
# Entities с одинаковым именем автоматически дедуплицируются
entity1 = Entity("DR. EMILY ROBERTS", text_hash_1)
entity2 = Entity("DR. EMILY ROBERTS", text_hash_2)

# entity1.hash_id == entity2.hash_id (одинаковый raw_context)

# При добавлении в граф
if G.has_node(entity1.hash_id):
    G.nodes[entity1.hash_id]['weight'] += 1  # Только инкремент
else:
    G.add_node(entity1.hash_id, type='entity', weight=1)
```

### Challenges

**Вариации имен**:
```
"Dr. Emily Roberts" vs "Emily Roberts" vs "E. Roberts"
```

**Решение**: LLM должен нормализовать в extraction prompt
```
"Для персон используйте полное имя в формате: DR. EMILY ROBERTS"
```

**Homonyms** (омонимы):
```
"PARIS" (город) vs "PARIS" (имя)
"APPLE" (фрукт) vs "APPLE" (компания)
```

**Решение**: Context disambiguation через relationships и semantic units

---

## Примеры

### Пример 1: Персона

**Entity**:
```json
{
  "hash_id": "abc123",
  "human_readable_id": "E00001",
  "context": "DR. EMILY ROBERTS",
  "weight": 15
}
```

**Connected Nodes**:
- 8 Semantic Units (факты о ней)
- 3 Relationships (works at, presented at, researches)
- 1 Attribute (comprehensive summary)

### Пример 2: Место

**Entity**:
```json
{
  "hash_id": "def456",
  "human_readable_id": "E00025",
  "context": "PARIS",
  "weight": 12
}
```

**Connected Nodes**:
- 10 Semantic Units (события в Париже)
- 5 Relationships (located in, hosted, capital of)
- 0 Attributes (не достаточно important или не в k-core)

### Пример 3: Концепт

**Entity**:
```json
{
  "hash_id": "ghi789",
  "human_readable_id": "E00103",
  "context": "SOLAR PANEL EFFICIENCY",
  "weight": 23
}
```

**Connected Nodes**:
- 15 Semantic Units (факты о эффективности)
- 8 Relationships (improved by, measured by, depends on)
- 1 Attribute (technical summary)

### Пример 4: Редкая Entity

**Entity**:
```json
{
  "hash_id": "jkl012",
  "human_readable_id": "E00501",
  "context": "SPECIFIC CONFERENCE ROOM A-123",
  "weight": 1
}
```

**Connected Nodes**:
- 1 Semantic Unit (единственное упоминание)
- 0 Relationships
- 0 Attributes (weight=1, не important)

---

## Метрики

### Distribution

**Typical distribution** (для 1000 entities):
- Weight = 1: 40% (упоминаются один раз)
- Weight = 2-5: 35% (умеренные упоминания)
- Weight = 6-20: 20% (важные entities)
- Weight > 20: 5% (центральные entities)

### Degree

**Average degree**: 5-15 connections
- Semantic units: 3-10
- Relationships: 1-5
- Attributes: 0-1

**Hub entities**: >30 connections

---

## Best Practices

### 1. Naming Conventions

✅ **Хорошо**:
- Консистентный case (UPPERCASE)
- Полные имена: "DR. EMILY ROBERTS", не "E. Roberts"
- Стандартизированные даты: "2024-09"

❌ **Плохо**:
- Смешанный case: "Dr. Emily Roberts", "dr. emily roberts"
- Неполные имена: "Roberts", "Emily"
- Нестандартные форматы: "Sept 2024", "9/2024"

### 2. Granularity

**Оптимальный уровень**:
- Достаточно специфичный для идентификации
- Не слишком детальный

✅ **Good**: "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
❌ **Too general**: "CONFERENCE"
❌ **Too specific**: "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY ROOM A SESSION 3"

### 3. LLM Prompting

**Key instructions**:
```
- Extract named entities (people, organizations, places, dates, concepts)
- Use UPPERCASE for consistency
- Normalize dates to YYYY-MM or YYYY-MM-DD format
- Use full names for people (DR. FIRSTNAME LASTNAME)
- Be consistent in entity naming across all extractions
```

---

## Диагностика

### Проверка entities

```python
# Загрузить entities
df = storage.load_parquet(config.entities_path)

print(f"Total entities: {len(df)}")
print(f"Unique entities: {df['hash_id'].nunique()}")

# Weight distribution
print(df['weight'].describe())
print(df['weight'].value_counts().head(20))

# Top entities by weight
top_entities = df.nlargest(10, 'weight')
print(top_entities[['context', 'weight']])
```

### Проверка connections

```python
# Entities без связей (проблема)
for entity_hash in entities['hash_id']:
    if entity_hash not in G.nodes:
        print(f"Warning: Entity {entity_hash} not in graph")
        continue

    degree = G.degree(entity_hash)
    if degree == 0:
        print(f"Warning: Entity {entity_hash} has no connections")
```

### Проверка naming consistency

```python
# Найти похожие имена (возможные дубликаты)
from difflib import SequenceMatcher

entities_list = df['context'].tolist()
for i, entity1 in enumerate(entities_list):
    for entity2 in entities_list[i+1:]:
        similarity = SequenceMatcher(None, entity1.lower(), entity2.lower()).ratio()
        if 0.8 < similarity < 1.0:
            print(f"Similar entities: {entity1} ~ {entity2} ({similarity:.2f})")
```

---

## FAQ

**Q: Почему entities не имеют embeddings?**
A: Entities используются для exact match и graph navigation, не для semantic similarity. Их имена короткие и специфичные, поэтому exact match эффективнее.

**Q: Как различать entities с одинаковыми именами (homonyms)?**
A: Через context - connected semantic units и relationships должны disambiguate. В будущем можно добавить entity disambiguation через type tags.

**Q: Что делать с entities, которые имеют weight=1?**
A: Это нормально. Не все entities центральные. Они все равно полезны для связывания фактов и могут стать важными при добавлении новых документов.

**Q: Нужно ли вручную создавать entities?**
A: Нет, они автоматически извлекаются LLM в Text Pipeline. Ручное создание не поддерживается.

**Q: Как улучшить качество entity extraction?**
A: Через prompt engineering - добавьте examples, clear instructions, consistency rules в text decomposition prompt.

---

## Связанные Узлы

**Источники (созданы из)**:
- [Semantic Unit Nodes](./03-semantic-unit-nodes.md) - entities упоминаются в фактах
- [Relationship Nodes](./05-relationship-nodes.md) - source/target в relationships

**Связаны с**:
- [Attribute Nodes](./06-attribute-nodes.md) - описания important entities
- [Relationship Nodes](./05-relationship-nodes.md) - участие в отношениях

**Используется в**:
- [Search and Retrieval](./09-search-and-retrieval.md) - для accurate search и graph entry points
- [Answer Generation](./10-answer-generation.md) - для entity-centric contexts

---

[← Назад: Semantic Unit Nodes](./03-semantic-unit-nodes.md) | [Следующий: Relationship Nodes →](./05-relationship-nodes.md)
