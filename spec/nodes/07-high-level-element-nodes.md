# High-Level Element Nodes 🌐

## Обзор

**High-Level Element Nodes** представляют тематические концепты высокого уровня, извлеченные из communities в графе. Они обеспечивают абстрактный слой для навигации и поиска, объединяя семантически связанные semantic units и attributes под общими темами.

---

## Характеристики

### Классы
```python
NodeRAG/build/component/community.py

class Community_summary(Unit_base):
    - community_node: list[str]      # Узлы в community
    - response: dict                 # LLM response с high-level elements
    - hash_id: str                   # SHA256(community_node)
    - human_readable_id: str         # C00001, C00002, ...

class High_level_elements(Unit_base):
    - context: str                   # Описание концепта
    - title: str                     # Название концепта
    - title_hash_id: str             # SHA256(title)
    - hash_id: str                   # SHA256(context)
    - human_readable_id: str         # H00001, H00002, ...
    - embedding: list[float]         # Vector embedding
    - related_node: list[str]        # Узлы из community
```

### Свойства High-Level Element

| Свойство             | Тип         | Описание                              |
|----------------------|-------------|---------------------------------------|
| `context`            | str         | Detailed описание концепта            |
| `title`              | str         | Короткое название темы                |
| `title_hash_id`      | str         | Hash ID названия (для поиска)         |
| `hash_id`            | str         | Hash ID описания                      |
| `human_readable_id`  | str         | Последовательный ID (H00001, ...)     |
| `embedding`          | list[float] | Vector embedding описания             |
| `related_node`       | list[str]   | Hash IDs узлов в community            |

### Metadata

| Параметр             | Значение                                |
|----------------------|-----------------------------------------|
| Type                 | `high_level_element` и `high_level_element_title` |
| Has Embedding        | ✅ Да (для описания)                    |
| Has Weight           | ✅ Да (частота встречаемости)           |
| Searchable           | ✅ Да (semantic + title match)          |
| Storage              | `high_level_elements.parquet`, `high_level_elements_titles.parquet` |
| In Graph             | ✅ Да (оба узла: element + title)       |

---

## Создание

### Pipeline Stage
**Summary Generation Pipeline** (Stage 6)

### Процесс

```python
# 1. Community Detection (Leiden Algorithm)
partition = leiden.find_partition(G, ModularityVertexPartition)

# partition = [[node1, node2, ...], [node10, node11, ...], ...]
# Каждый элемент = community

# 2. Для каждого community
for community in partition:
    # Фильтровать узлы с embeddings (semantic units, attributes)
    community_nodes = [
        node for node in community
        if node in mapper.embeddings
    ]

    # 3. Собрать контексты
    contexts = [
        mapper.get(node, 'context')
        for node in community_nodes
    ]

    # 4. Формирование промпта
    prompt = community_summary_prompt.format(
        content='\n'.join(contexts)
    )

    # 5. LLM генерация
    response = await LLM_client({
        'query': prompt,
        'response_format': high_level_element_json_schema
    })

    # response = {
    #     'high_level_elements': [
    #         {'title': 'Renewable Energy Research', 'description': '...'},
    #         {'title': 'Solar Technology', 'description': '...'}
    #     ]
    # }

    # 6. Создание High-Level Element узлов
    for he_data in response['high_level_elements']:
        he = High_level_elements(
            context=he_data['description'],
            title=he_data['title'],
            config=config
        )
        he.related_node(community_nodes)

        # 7. Дедупликация
        if G.has_node(he.hash_id):
            G.nodes[he.hash_id]['weight'] += 1
            G.nodes[he.title_hash_id]['weight'] += 1
        else:
            # Добавить element узел
            G.add_node(he.hash_id, type='high_level_element', weight=1)

            # Добавить title узел (для exact match)
            G.add_node(
                he.title_hash_id,
                type='high_level_element_title',
                weight=1,
                related_node=he.hash_id
            )

            # Связь element ↔ title
            G.add_edge(he.hash_id, he.title_hash_id, weight=1)

    # 8. Embedding generation
    embedding = await embedding_client(he.context)
    he.store_embedding(embedding)

    # 9. Связь с community nodes
    # KMeans clustering для оптимизации (если много узлов)
    if len(community_nodes) > threshold:
        # Clustering для определения наиболее связанных узлов
        cluster_assignment = kmeans_clustering(
            he_embeddings, node_embeddings
        )
        # Связываем только с узлами в том же кластере
    else:
        # Прямые связи со всеми узлами community
        for node in community_nodes:
            G.add_edge(node, he.hash_id, weight=1)
```

---

## Роль в Графе

### Позиция
```
[High-Level Element: "Renewable Energy Research"] ← abstract concept
    ↓
    ├── [Semantic Unit 1]
    ├── [Semantic Unit 2]
    ├── [Attribute 1]
    └── ...
        ↕
[High-Level Element Title: "RENEWABLE ENERGY RESEARCH"] ← для exact match
```

### Два узла на концепт

**1. High-Level Element Node** (type='high_level_element'):
- Содержит detailed описание
- Имеет embedding для semantic search
- Связан с community nodes

**2. Title Node** (type='high_level_element_title'):
- Содержит короткое название
- Используется для exact/fuzzy match
- Связан с element node

### Связи

**High-Level Element**:
- **К Title Node**: Двунаправленная связь
- **К Community Nodes**: Множественные связи (semantic units, attributes)

**Title Node**:
- **От High-Level Element**: Связь к описанию
- `related_node` attribute: Hash ID основного element

### Graph Structure

```python
# Element node
G.add_node(
    he.hash_id,
    type='high_level_element',
    weight=1
)

# Title node
G.add_node(
    he.title_hash_id,
    type='high_level_element_title',
    weight=1,
    related_node=he.hash_id  # Ссылка на element
)

# Edge element ↔ title
G.add_edge(he.hash_id, he.title_hash_id, weight=1)

# Edges к community nodes
for node in community_nodes:
    G.add_edge(node, he.hash_id, weight=1)
```

---

## Storage Format

### high_level_elements.parquet

```python
{
    'type': 'high_level_element',
    'title_hash_id': 'title_hash...',    # Hash ID title node
    'context': 'Detailed description...', # Описание концепта
    'hash_id': 'abc123...',              # Hash ID element
    'human_readable_id': 'H00042',       # Human ID
    'related_nodes': [                   # Community nodes
        'semantic_unit_hash1',
        'attribute_hash2',
        ...
    ],
    'embedding': 'done'                  # Marker (actual в embedding.parquet)
}
```

### high_level_elements_titles.parquet

```python
{
    'type': 'high_level_element_title',
    'hash_id': 'title_hash...',         # Hash ID title
    'context': 'Renewable Energy Research',  # Название
    'human_readable_id': 'H00042'       # Тот же Human ID
}
```

### Пример записи

**Element**:
```json
{
  "type": "high_level_element",
  "title_hash_id": "t1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0",
  "context": "This high-level element encompasses research and developments in renewable energy, particularly focusing on solar panel technology, photovoltaic systems, and efficiency improvements. It includes various studies on enhancing energy conversion rates, innovations in solar cell materials, and practical applications in sustainable energy systems. The element also covers international collaborations and conferences dedicated to advancing renewable energy solutions.",
  "hash_id": "h1i2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a0b1c2d3e4f5",
  "human_readable_id": "H00001",
  "related_nodes": [
    "semantic_unit_hash_1",
    "semantic_unit_hash_5",
    "attribute_hash_7",
    ...
  ],
  "embedding": "done"
}
```

**Title**:
```json
{
  "type": "high_level_element_title",
  "hash_id": "t1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0",
  "context": "Renewable Energy Research",
  "human_readable_id": "H00001"
}
```

---

## Community Detection

### Leiden Algorithm

**Алгоритм**: Leiden community detection (оптимизация modularity)

```python
import leidenalg as la

# Конвертация в igraph
G_ig = igraph_from_networkx(G)

# Partition
partition = la.find_partition(G_ig, la.ModularityVertexPartition)

# partition = список communities
# Каждая community = список узлов, семантически связанных
```

**Результат**: Communities с высоким modularity - узлы внутри community более связаны между собой, чем с узлами других communities

### Used Nodes

Из каждой community используются только узлы с embeddings:

```python
used_nodes = []
for node in community:
    node_type = G.nodes[node]['type']
    if node_type == 'semantic_unit':
        used_nodes.append(node)
    elif node_type == 'attribute':
        used_nodes.append(node)
    elif node_type == 'entity' and G.nodes[node].get('attribute'):
        # Entity с attribute - добавить attribute
        for neighbor in G.neighbors(node):
            if G.nodes[neighbor]['type'] == 'attribute':
                used_nodes.append(neighbor)
```

**Обоснование**: Semantic units и attributes содержат actual content для summarization

---

## LLM Summarization

### Prompt Structure

```python
prompt = f"""
Below is a collection of information units from a knowledge graph community:

{context}

Based on these units, identify and describe the main high-level themes and
concepts. For each theme:
1. Provide a clear, descriptive title (2-5 words)
2. Write a comprehensive description (100-200 words) explaining the theme,
   its key aspects, and how it relates to the information provided

Return as JSON:
{{
  "high_level_elements": [
    {{"title": "Theme Title", "description": "Detailed description..."}},
    ...
  ]
}}
"""
```

### Response Format

```json
{
  "high_level_elements": [
    {
      "title": "Renewable Energy Research",
      "description": "This theme covers research and developments..."
    },
    {
      "title": "Solar Technology Innovations",
      "description": "Focus on solar panel efficiency..."
    },
    {
      "title": "International Collaboration",
      "description": "Conferences and partnerships..."
    }
  ]
}
```

**Typical output**: 1-5 high-level elements per community

---

## Дедупликация

### По Hash ID

```python
# Если high-level element с таким же context уже существует
if G.has_node(he.hash_id):
    # Инкремент weight
    G.nodes[he.hash_id]['weight'] += 1

    # Также для title
    if G.has_node(he.title_hash_id):
        G.nodes[he.title_hash_id]['weight'] += 1
    else:
        # Title node не существует (странно, но handle)
        continue
else:
    # Новый high-level element
    add_high_level_element(he)
```

**Интерпретация weight**:
- Weight = 1: Уникальная тема, встречается в одной community
- Weight > 1: Тема пересекается с несколькими communities

---

## KMeans Clustering для Связей

### Проблема

Для больших communities (>100 узлов) создание edge к каждому узлу неэффективно

### Решение

```python
threshold = config.Hcluster_size  # default: ~50

if len(community_nodes) > threshold:
    # 1. Embeddings всех узлов + high-level elements
    node_embeddings = [mapper.embeddings[node] for node in community_nodes]
    he_embeddings = [he.embedding for he in high_level_elements]
    all_embeddings = np.vstack([he_embeddings, node_embeddings])

    # 2. KMeans clustering
    centroids = ceil(sqrt(len(all_embeddings)))
    kmeans = faiss.Kmeans(d=embedding_dim, k=centroids)
    kmeans.train(all_embeddings)
    _, cluster_labels = kmeans.assign(all_embeddings)

    # 3. Связываем только узлы в том же кластере
    he_cluster_labels = cluster_labels[:len(high_level_elements)]
    node_cluster_labels = cluster_labels[len(high_level_elements):]

    for i, he in enumerate(high_level_elements):
        for j, node in enumerate(community_nodes):
            if he_cluster_labels[i] == node_cluster_labels[j]:
                # Тот же кластер - создать edge
                if node in he.related_node:
                    G.add_edge(node, he.hash_id, weight=1)
else:
    # Малая community - прямые связи со всеми
    for he in high_level_elements:
        for node in he.related_node:
            G.add_edge(node, he.hash_id, weight=1)
```

**Результат**: Сокращение числа edges при сохранении semantic relationships

---

## Embeddings

### Генерация

```python
# Batch processing
for i in range(0, len(high_level_elements), batch_size):
    batch = high_level_elements[i:i+batch_size]
    contexts = [he.context for he in batch]

    # Embedding model
    embeddings = await embedding_client(contexts)

    # Store
    for he, emb in zip(batch, embeddings):
        he.store_embedding(emb)
```

### Особенности

**Длина**: 100-200 tokens (shorter than attributes, longer than semantic units)

**Quality**: Abstract, conceptual descriptions → broad semantic matching

---

## Использование в Поиске

### Dual Search Strategy

**1. Semantic Search** (через element node):
```python
query_embedding = embedding_model(query)

# Search in high-level elements
results = HNSW_index.search(
    query_embedding,
    k=20,
    filter=lambda x: x['type'] == 'high_level_element'
)
```

**2. Title Match** (через title node):
```python
# Query: "renewable energy"
query_entities = decompose_query(query)

# Exact/fuzzy match в titles
for entity in query_entities:
    pattern = re.compile(r'\b' + re.escape(entity.lower()) + r'\b')
    matches = [
        title_hash for title_hash, title_text in titles.items()
        if pattern.search(title_text.lower())
    ]

    # Получить related high-level element
    for title_hash in matches:
        element_hash = G.nodes[title_hash]['related_node']
        accurate_results.append(element_hash)
```

### Graph Navigation Entry Point

High-level elements служат **theme-based entry points**:

```python
# После нахождения high-level element
he_hash = search_result['hash_id']

# Expand к connected nodes
community_nodes = [
    neighbor for neighbor in G.neighbors(he_hash)
    if G.nodes[neighbor]['type'] in ['semantic_unit', 'attribute']
]

# Получить detailed information
detailed_facts = [mapper.get(node, 'context') for node in community_nodes[:10]]
```

### PPR (Personalized PageRank)

```python
# High-level elements в personalization
personalization = {
    he_hash_1: 0.3,  # от semantic search
    he_hash_2: 0.2,
    entity_hash_1: 0.5  # от entity accurate search
}

# PPR diffuses вес через граф
ppr_scores = sparse_PPR.PPR(personalization, alpha=0.85)

# Retrieval top-k
top_nodes = sorted(ppr_scores.items(), key=lambda x: x[1], reverse=True)[:100]
```

### Post-Processing

```python
def post_process_top_k(ppr_results):
    he_title_list = []

    for node in ppr_results:
        if G.nodes[node]['type'] == 'high_level_element_title':
            if len(he_title_list) < config.Hnode:  # default: 3
                he_title_list.append(node)

    # Для каждого title, добавить related element
    for title_hash in he_title_list:
        element_hash = G.nodes[title_hash]['related_node']
        if element_hash not in retrieval.unique_search_list:
            retrieval.search_list.append(element_hash)
```

---

## Использование в Answer Generation

### Thematic Context

High-level elements предоставляют **thematic framing** для ответов:

```python
context = {
    'themes': [
        {
            'title': 'Renewable Energy Research',
            'description': 'This theme covers...'
        }
    ],
    'entities': [...],
    'facts': [...]
}

prompt = f"""
Based on the following themes and information:

Themes:
{format_themes(context['themes'])}

Facts:
{format_facts(context['facts'])}

Answer the question: {query}
"""
```

### Advantages

✅ **Context setting**: LLM понимает broader theme
✅ **Coherence**: Ответ структурирован вокруг темы
✅ **Completeness**: Theme description дополняет specific facts

### Example

**Query**: "What research is being done in renewable energy?"

**Retrieval**:
- High-level element: "Renewable Energy Research" (theme)
- Semantic units: Specific facts о исследованиях
- Entities: DR. EMILY ROBERTS, SOLAR PANEL EFFICIENCY

**Context**:
```
Theme: Renewable Energy Research
This theme covers research and developments in renewable energy, particularly
focusing on solar panel technology and efficiency improvements.

Specific facts:
- Dr. Emily Roberts researches solar panel efficiency at European Research Institute
- Research demonstrated 15% improvement in photovoltaic performance
- International Conference on Renewable Energy held in Paris, September 2024
```

**Generated Answer**:
```
Research in renewable energy is actively progressing, with a particular focus
on solar panel technology and efficiency improvements. Dr. Emily Roberts, a
researcher at the European Research Institute, is conducting groundbreaking
work in this area. Her recent research has demonstrated a 15% improvement in
photovoltaic performance. This work was presented at the International Conference
on Renewable Energy in Paris in September 2024, highlighting the ongoing
international collaboration in advancing renewable energy solutions.
```

---

## Примеры

### Пример 1: Technical Theme

**Community**: 25 semantic units о solar technology

**High-Level Element**:
```json
{
  "title": "Solar Panel Efficiency Improvements",
  "description": "This theme focuses on technological advancements in solar panel efficiency, including research on photovoltaic materials, energy conversion optimization, and practical applications. Key developments include 15% efficiency improvements and innovations in solar cell design.",
  "related_nodes": [25 semantic unit hashes]
}
```

**Weight**: 1 (unique theme)

### Пример 2: Cross-Cutting Theme

**Community 1**: Semantic units о renewable energy
**Community 2**: Semantic units о climate change solutions

**High-Level Element** (appears in both):
```json
{
  "title": "Sustainable Energy Solutions",
  "description": "Broad theme covering sustainable energy approaches...",
  "weight": 2
}
```

**Weight**: 2 (appears in 2 communities)

### Пример 3: Event Theme

**Community**: Semantic units о конференции

**High-Level Element**:
```json
{
  "title": "International Conference on Renewable Energy",
  "description": "This element represents the International Conference on Renewable Energy held in Paris in September 2024. The conference brought together researchers and practitioners to discuss advances in solar technology, wind energy, and sustainable systems.",
  "related_nodes": [15 semantic unit hashes]
}
```

---

## Метрики

### Coverage

**Typical metrics**:
- Communities detected: 50-100
- High-level elements created: 100-200
- Average elements per community: 1-3

### Distribution

**Elements by weight**:
- Weight = 1: 85% (unique themes)
- Weight = 2-3: 12% (cross-cutting themes)
- Weight > 3: 3% (very broad themes)

### Retrieval Impact

**With high-level elements**: ~30% improvement в theme-based queries
**Average nodes per element**: 10-50

---

## Best Practices

### 1. Community Detection

- Use Leiden algorithm для better modularity
- Monitor community sizes (too large → oversummarization)
- Consider resolution parameter tuning

### 2. Summarization Prompts

**Key instructions**:
```
- Identify 1-5 main themes
- Each title: clear, concise (2-5 words)
- Each description: comprehensive (100-200 words)
- Be abstract but grounded in provided context
- Avoid overgeneralization
```

### 3. Clustering Optimization

- Use KMeans для больших communities (>50 nodes)
- Balance precision (fewer edges) vs recall (coverage)
- Monitor edge count growth

---

## Диагностика

### Проверка high-level elements

```python
# Загрузить high-level elements
he_df = storage.load_parquet(config.high_level_elements_path)
titles_df = storage.load_parquet(config.high_level_elements_titles_path)

print(f"Total high-level elements: {len(he_df)}")
print(f"Total titles: {len(titles_df)}")

# Should be equal
assert len(he_df) == len(titles_df)

# Weight distribution
print(he_df['weight'] if 'weight' in he_df.columns else "No weight column")
```

### Проверка graph structure

```python
# Каждый element должен иметь title node
for he_hash in he_df['hash_id']:
    if not G.has_node(he_hash):
        print(f"Warning: Element {he_hash} not in graph")
        continue

    # Найти title node
    title_hash = he_df[he_df['hash_id'] == he_hash]['title_hash_id'].iloc[0]

    if not G.has_node(title_hash):
        print(f"Warning: Title {title_hash} not in graph")
        continue

    # Проверить связь element ↔ title
    if not G.has_edge(he_hash, title_hash):
        print(f"Warning: No edge between element {he_hash} and title {title_hash}")
```

### Проверка community coverage

```python
# Сколько узлов связаны с high-level elements?
connected_nodes = set()
for he_hash in he_df['hash_id']:
    neighbors = [
        n for n in G.neighbors(he_hash)
        if G.nodes[n]['type'] in ['semantic_unit', 'attribute']
    ]
    connected_nodes.update(neighbors)

all_semantic_units = [
    n for n in G.nodes
    if G.nodes[n]['type'] == 'semantic_unit'
]

coverage = len(connected_nodes) / len(all_semantic_units)
print(f"Community coverage: {coverage*100:.1f}%")
```

---

## FAQ

**Q: Почему используются два узла (element + title)?**
A: Element имеет detailed описание для semantic search, title - короткое название для exact match. Это обеспечивает hybrid search (semantic + keyword).

**Q: Как high-level elements отличаются от attributes?**
A: Attributes описывают конкретные entities. High-level elements описывают abstract themes, объединяющие multiple entities и facts.

**Q: Могут ли разные communities иметь одинаковые high-level elements?**
A: Да, через дедупликацию (hash_id). Weight отражает, сколько communities содержат эту тему.

**Q: Как выбрать количество high-level elements для community?**
A: LLM определяет автоматически (обычно 1-5). Зависит от разнообразия контента в community.

**Q: Нужно ли перегенерировать high-level elements при добавлении документов?**
A: В текущей реализации - да, полная регенерация. Incremental update high-level elements - future feature.

---

## Связанные Узлы

**Источники (созданы из)**:
- [Semantic Unit Nodes](./03-semantic-unit-nodes.md) - входят в communities
- [Attribute Nodes](./06-attribute-nodes.md) - входят в communities

**Используется в**:
- [Search and Retrieval](./09-search-and-retrieval.md) - для theme-based search
- [Answer Generation](./10-answer-generation.md) - для thematic framing

**Связи**:
- [Node Relationships](./08-node-relationships.md) - формируют top layer графа

---

[← Назад: Attribute Nodes](./06-attribute-nodes.md) | [Следующий: Node Relationships →](./08-node-relationships.md)
