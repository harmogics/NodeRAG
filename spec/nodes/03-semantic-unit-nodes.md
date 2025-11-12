# Semantic Unit Nodes 💡

## Обзор

**Semantic Unit Nodes** представляют атомарные семантически связные единицы информации - факты, события, утверждения. Они являются результатом LLM декомпозиции text units и составляют основу фактического слоя в графе знаний.

---

## Характеристики

### Класс
```python
NodeRAG/build/component/semantic_unit.py

class Semantic_unit(Unit_base):
    - raw_context: str       # Paraphrased semantic unit text
    - text_hash_id: str      # Родительский Text Unit
    - hash_id: str           # SHA256(raw_context)
    - human_readable_id: str # S00001, S00002, ...
```

### Свойства

| Свойство             | Тип       | Описание                              |
|----------------------|-----------|---------------------------------------|
| `raw_context`        | str       | Суммаризированный текст semantic unit |
| `text_hash_id`       | str       | Hash ID родительского Text Unit       |
| `hash_id`            | str       | SHA256 хеш от raw_context             |
| `human_readable_id`  | str       | Последовательный ID (S00001, ...)     |

### Metadata

| Параметр             | Значение                       |
|----------------------|--------------------------------|
| Type                 | `semantic_unit`                |
| Has Embedding        | ✅ Да                          |
| Has Weight           | ✅ Да (частота появления)      |
| Searchable           | ✅ Да (primary search target)  |
| Storage              | `semantic_units.parquet`       |
| In Graph             | ✅ Да                          |

---

## Создание

### Pipeline Stage
**Text Pipeline** (Stage 2) - extraction via LLM
**Graph Pipeline** (Stage 3) - добавление в граф

### Процесс LLM Extraction

```python
# 1. Text Unit отправляется в LLM
prompt = text_decomposition_prompt.format(text=text_unit.raw_context)

# 2. LLM decomposition
response = await LLM_client({
    'query': prompt,
    'response_format': text_decomposition_json_schema
})

# 3. Для каждого semantic unit в response
for output in response['Output']:
    semantic_unit_text = output['semantic_unit']

    # 4. Создание Semantic Unit узла
    su = Semantic_unit(semantic_unit_text, text_unit.hash_id)

    # 5. Добавление в граф
    if G.has_node(su.hash_id):
        # Дедупликация - инкремент веса
        G.nodes[su.hash_id]['weight'] += 1
    else:
        # Новый узел
        G.add_node(su.hash_id, type='semantic_unit', weight=1)
```

### Характеристики Extraction

**Input**: Text Unit (500-1048 tokens)
**Output**: Multiple Semantic Units (обычно 3-10)

**Каждый Semantic Unit**:
- Одно событие/факт/концепт
- Paraphrased (не дословная цитата)
- Сохраняет все ключевые детали
- Связан с entities

---

## Роль в Графе

### Позиция
```
[Text Unit]
    ↓ LLM decomposition
[Semantic Unit] ← центральный узел фактического слоя
    ↓
    ├── [Entity A]
    ├── [Entity B]
    └── [Entity C]
```

### Связи

**Входящие связи**:
- **От High-Level Elements**: Концепты связаны с фактами
- (Косвенно от Text Unit через text_hash_id)

**Исходящие связи**:
- **К Entities**: Множественные связи (упоминания сущностей)

### Graph Structure

```python
# Semantic Unit узел
G.add_node(
    semantic_unit.hash_id,
    type='semantic_unit',
    weight=1  # инкрементируется при дубликатах
)

# Связи с entities
for entity in extracted_entities:
    G.add_edge(
        semantic_unit.hash_id,
        entity.hash_id,
        weight=1
    )
```

### Centrality

Semantic Units - это **connecting nodes** в графе:

```
        [Entity A]
             ↑
             │
[SU1] ←→ [SU2] ←→ [SU3]
             │
             ↓
        [Entity B]
```

---

## Storage Format

### semantic_units.parquet

```python
{
    'hash_id': 'abc123...',           # hash_id
    'human_readable_id': 'S00001',    # human ID
    'type': 'semantic_unit',          # тип
    'context': 'paraphrased text...',  # raw_context
    'text_hash_id': 'def456...',      # родительский Text Unit
    'weight': 3,                      # частота появления
    'embedding': [0.1, 0.2, ...],     # vector
    'insert': None                    # metadata
}
```

### Пример записи
```json
{
  "hash_id": "c5h0d4e7f8g9h1i2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9",
  "human_readable_id": "S00042",
  "type": "semantic_unit",
  "context": "In September 2024, Dr. Emily Roberts attended the International Conference on Renewable Energy in Paris, where she presented research on solar panel efficiency improvements.",
  "text_hash_id": "b4g9c3d5e6f7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7",
  "weight": 1,
  "embedding": [0.123, -0.456, 0.789, ...],
  "insert": null
}
```

---

## Семантические Характеристики

### Atomicity

Каждый Semantic Unit = **один атомарный факт**.

**✅ Хорошо** (atomic):
```
"Dr. Emily Roberts attended the International Conference in Paris."
```

**❌ Плохо** (не atomic):
```
"Dr. Emily Roberts attended the International Conference in Paris,
where she presented research, met colleagues, and discussed
partnerships." (multiple facts)
```

### Paraphrasing

Semantic Units - это **paraphrases**, не дословные цитаты.

**Original Text Unit**:
```
"During her visit to Paris in September 2024, Dr. Emily Roberts,
a researcher at the European Research Institute, attended the
International Conference on Renewable Energy where she presented
her latest research findings on improving solar panel efficiency
by 15%."
```

**Semantic Unit** (paraphrased):
```
"In September 2024, Dr. Emily Roberts attended the International
Conference on Renewable Energy in Paris."
```

**Цель**:
- Сжатие без потери key info
- Улучшение clarity
- Стандартизация формулировок

### Completeness

Semantic Units сохраняют **все crucial details**.

**Критичные детали**:
- Who (Dr. Emily Roberts)
- What (attended conference)
- When (September 2024)
- Where (Paris)
- Why/How (если relevant)

---

## Weight: Частота Появления

### Механизм

```python
if G.has_node(semantic_unit.hash_id):
    # Семантически идентичный unit уже существует
    G.nodes[semantic_unit.hash_id]['weight'] += 1
else:
    # Новый unique unit
    G.add_node(semantic_unit.hash_id, type='semantic_unit', weight=1)
```

### Интерпретация

**Weight = 1**: Unique fact, упоминается один раз
**Weight = 5**: Повторяется в 5 разных контекстах
**Weight = 20**: Очень важный факт, многократно упоминается

### Использование Weight

**1. Ranking в Search**:
```python
# При равном cosine similarity, приоритет higher weight
results.sort(key=lambda x: (x['similarity'], x['weight']), reverse=True)
```

**2. Фильтрация по важности**:
```python
# Только часто упоминаемые факты
important_facts = [
    su for su in semantic_units
    if su['weight'] >= threshold
]
```

**3. Community Detection**:
```python
# Weight влияет на modularity в Leiden algorithm
# Высокий weight = stronger influence на community structure
```

---

## Embeddings

### Генерация

**Pipeline**: Embedding Pipeline (Stage 5)

```python
# Batch обработка
contexts = [su['context'] for su in semantic_units]
embeddings = await embedding_model(contexts)

# Сохранение
for su, emb in zip(semantic_units, embeddings):
    save_embedding(su['hash_id'], emb)
```

### Качество Embeddings

**Преимущества для Semantic Units**:
✅ Краткие и focused → лучшие embeddings
✅ Semantic coherence → высокое качество
✅ Paraphrased → нормализованные формулировки

**vs Text Units**:
- Text Units: broader, менее focused
- Semantic Units: precise, single-concept

---

## Использование в Поиске

### Primary Search Target

Semantic Units - это **основная цель** для semantic search.

**Почему**:
1. **Precision**: Один факт → точное совпадение
2. **Granularity**: Fine-grained information
3. **Quality**: Хорошие embeddings от focused content

### Search Flow

```python
# 1. Query embedding
query = "What did Dr. Emily Roberts research?"
query_emb = embedding_model(query)

# 2. Semantic Unit search
results = HNSW_index.search(
    query_emb,
    k=20,
    filter=lambda x: x['type'] == 'semantic_unit'
)

# 3. Ranking by similarity + weight
ranked_results = sorted(
    results,
    key=lambda x: (x['similarity'], x['weight']),
    reverse=True
)[:10]

# 4. Return top semantic units
for result in ranked_results:
    semantic_unit = load_semantic_unit(result['hash_id'])
    print(f"Fact: {semantic_unit.context}")
    print(f"Weight: {semantic_unit.weight}")
    print(f"Similarity: {result['similarity']:.3f}")
```

### Expansion через граф

После нахождения релевантных Semantic Units, expand для больше context:

```python
# Найдены релевантные semantic units
relevant_sus = search_results[:5]

expanded_context = []
for su in relevant_sus:
    # 1. Semantic unit сам
    expanded_context.append(su.context)

    # 2. Connected entities
    entities = [
        G.nodes[neighbor]
        for neighbor in G.neighbors(su.hash_id)
        if G.nodes[neighbor]['type'] == 'entity'
    ]

    # 3. Attributes этих entities (если есть)
    for entity in entities:
        if 'attributes' in entity:
            attr = load_attribute(entity['attributes'][0])
            expanded_context.append(attr.context)

    # 4. Related semantic units (через shared entities)
    for entity in entities:
        related_sus = [
            G.nodes[neighbor]
            for neighbor in G.neighbors(entity.hash_id)
            if G.nodes[neighbor]['type'] == 'semantic_unit'
        ]
        expanded_context.extend([su.context for su in related_sus[:2]])
```

---

## Использование в Answer Generation

### As Primary Facts

Semantic Units предоставляют **базовые факты** для ответа.

```python
# Retrieval
semantic_units = search(query, k=10)

# Context assembly
facts = "\n".join([
    f"- {su.context}"
    for su in semantic_units
])

# Answer generation
answer = await answer_llm({
    'query': user_query,
    'context': f"Based on these facts:\n{facts}\n\nAnswer the question."
})
```

### Advantages

✅ **Focused**: Только релевантные факты, не лишний текст
✅ **Clear**: Paraphrased формулировки легче для LLM
✅ **Comprehensive**: Multiple facts → полная картина

### Example

**Query**: "What research did Dr. Emily Roberts conduct?"

**Retrieved Semantic Units**:
1. "Dr. Emily Roberts presented research on solar panel efficiency improvements." (weight=3)
2. "Her research focuses on enhancing photovoltaic system performance." (weight=2)
3. "Dr. Emily Roberts published findings on 15% efficiency increase." (weight=1)

**Generated Answer**:
```
Dr. Emily Roberts conducts research in the field of renewable energy,
specifically focusing on solar panel technology. Her work aims to
enhance the efficiency of photovoltaic systems, and she has published
findings demonstrating a 15% increase in solar panel efficiency. She
has presented this research at international conferences, highlighting
its significance for sustainable energy development.
```

---

## Связь с Entities

### Множественные связи

```
[Semantic Unit]
    ├── [Entity: DR. EMILY ROBERTS]
    ├── [Entity: INTERNATIONAL CONFERENCE]
    ├── [Entity: PARIS]
    ├── [Entity: 2024-09]
    └── [Entity: SOLAR PANEL EFFICIENCY]
```

### Граф навигация

```python
# От Semantic Unit к Entities
su_hash = "abc123..."
entities = [
    G.nodes[neighbor]
    for neighbor in G.neighbors(su_hash)
    if G.nodes[neighbor]['type'] == 'entity'
]

# Получить контексты entities
entity_names = [mapper.get(e['hash_id'], 'context') for e in entities]
print(f"Entities in this fact: {entity_names}")
```

### Entity Frequency

```python
# Сколько раз entity упоминается в semantic units?
entity_hash = "def456..."
semantic_units_with_entity = [
    neighbor
    for neighbor in G.neighbors(entity_hash)
    if G.nodes[neighbor]['type'] == 'semantic_unit'
]

print(f"Entity mentioned in {len(semantic_units_with_entity)} facts")
```

---

## Связь с High-Level Elements

### Концепты → Факты

```
[High-Level Element: "Renewable Energy Research"]
    ├── [SU: "Dr. Emily Roberts researched solar panels"]
    ├── [SU: "Conference focused on renewable energy"]
    └── [SU: "15% efficiency improvement achieved"]
```

### Community Membership

Semantic Units в одном community = семантически близкие факты.

```python
# После community detection
community_nodes = partition[0]  # First community

semantic_units_in_community = [
    node for node in community_nodes
    if G.nodes[node]['type'] == 'semantic_unit'
]

# Эти semantic units семантически связаны
# Будет создан High-Level Element, обобщающий их
```

---

## Примеры

### Пример 1: Простой факт

**Text Unit**:
```
"Dr. Emily Roberts is a researcher at the European Research Institute."
```

**Semantic Unit**:
```
S00001: "Dr. Emily Roberts works at the European Research Institute."
```

**Connected Entities**:
- E00001: "DR. EMILY ROBERTS"
- E00002: "EUROPEAN RESEARCH INSTITUTE"

**Weight**: 1 (unique mention)

### Пример 2: Комплексный факт

**Text Unit**:
```
"In September 2024, Dr. Emily Roberts from the European Research
Institute attended the International Conference on Renewable Energy
in Paris, where she presented her research on improving solar panel
efficiency by 15%."
```

**Semantic Units** (decomposed):
```
S00010: "In September 2024, Dr. Emily Roberts attended the International
         Conference on Renewable Energy in Paris."

S00011: "Dr. Emily Roberts works at the European Research Institute."

S00012: "Dr. Emily Roberts presented research on solar panel efficiency
         improvements at the conference."

S00013: "The research demonstrated a 15% improvement in solar panel efficiency."
```

**Entities extracted**:
- DR. EMILY ROBERTS
- 2024-09
- INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY
- PARIS
- EUROPEAN RESEARCH INSTITUTE
- SOLAR PANEL EFFICIENCY

### Пример 3: Дубликат (высокий weight)

**Scenario**: Один и тот же факт упоминается в 5 разных документах.

**First occurrence**:
```python
S00100: {
    context: "Dr. Emily Roberts specializes in solar energy research.",
    weight: 1
}
```

**After processing 5 documents**:
```python
S00100: {
    context: "Dr. Emily Roberts specializes in solar energy research.",
    weight: 5  # Incremented 4 times
}
```

**Интерпретация**: Это **важный факт**, многократно подтвержденный.

---

## Метрики

### Extraction Rate

**Average**: 1 Text Unit → 5 Semantic Units

**Зависит от**:
- Information density текста
- Структурированность
- LLM prompt quality

### Weight Distribution

**Типичное распределение** (для 5000 semantic units):
- Weight = 1: 60% (unique facts)
- Weight = 2-3: 25% (moderately repeated)
- Weight = 4-10: 10% (important facts)
- Weight > 10: 5% (very important facts)

### Embedding Quality

**Metrics**:
- Cosine similarity между семантически схожими units: >0.85
- Cosine similarity между несвязанными units: <0.5

---

## Best Practices

### 1. Prompt Engineering

**Key instructions в промпте**:
- "Segment into multiple semantic units"
- "Each unit = one specific event/fact"
- "Retain all crucial details"
- "Paraphrase, don't copy verbatim"

### 2. Granularity

**Оптимальный уровень**:
✅ Один факт per unit
✅ Достаточно деталей для понимания
✅ Не слишком general, не слишком specific

**Примеры**:

❌ Слишком general:
```
"Dr. Emily Roberts is a researcher."
```

✅ Good:
```
"Dr. Emily Roberts conducts research in renewable energy."
```

❌ Слишком specific:
```
"Dr. Emily Roberts used Method X with Parameter Y to achieve Result Z
in Experiment A on Date B."
```

### 3. Дедупликация

- Используйте hash_id для автоматической дедупликации
- Weight отражает importance через repetition
- Не нужно manually удалять дубликаты

---

## Диагностика

### Проверка extraction

```python
# Загрузить semantic units
df = storage.load_parquet(config.semantic_units_path)

print(f"Total semantic units: {len(df)}")
print(f"Unique units (weight=1): {len(df[df['weight']==1])}")
print(f"Repeated units (weight>1): {len(df[df['weight']>1])}")

# Weight distribution
print(df['weight'].value_counts().sort_index())
```

### Проверка связей

```python
# Semantic units без entities (проблема)
for su_hash in G.nodes:
    if G.nodes[su_hash]['type'] == 'semantic_unit':
        entities = [
            n for n in G.neighbors(su_hash)
            if G.nodes[n]['type'] == 'entity'
        ]
        if len(entities) == 0:
            print(f"Warning: {su_hash} has no entities")
```

### Качество paraphrasing

```python
# Сравнение с original text
for su_hash, su_data in semantic_units_dict.items():
    text_hash = su_data['text_hash_id']
    original_text = text_units[text_hash]['context']

    # Проверка: semantic unit не должен быть копией
    similarity = text_similarity(su_data['context'], original_text)

    if similarity > 0.95:
        print(f"Warning: {su_hash} is too similar to original")
```

---

## FAQ

**Q: Почему semantic units paraphrased, а не дословные цитаты?**
A: Paraphrasing улучшает clarity, нормализует формулировки, и делает embeddings более эффективными.

**Q: Как определить оптимальную granularity для semantic units?**
A: One fact per unit. Если unit содержит "and" или "also", вероятно нужно разделить.

**Q: Что делать с высоким weight - это хорошо или плохо?**
A: Хорошо! Высокий weight = важная, многократно подтвержденная информация. Используйте для ranking.

**Q: Могут ли semantic units из разных documents быть идентичными?**
A: Да, через hash_id. Это нормально и показывает консенсус в информации.

---

## Связанные Узлы

**Родитель**:
- [Text Unit Nodes](./02-text-unit-nodes.md) - исходные chunks

**Дети/Связанные**:
- [Entity Nodes](./04-entity-nodes.md) - упомянутые сущности

**Связи**:
- [High-Level Element Nodes](./07-high-level-element-nodes.md) - концепты связаны с фактами

**Используется в**:
- [Search and Retrieval](./09-search-and-retrieval.md) - primary search target
- [Answer Generation](./10-answer-generation.md) - базовые факты для ответов

---

[← Назад: Text Unit Nodes](./02-text-unit-nodes.md) | [Следующий: Entity Nodes →](./04-entity-nodes.md)
