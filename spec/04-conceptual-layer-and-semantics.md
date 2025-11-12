# Концептуальный Слой и Семантическая Обработка

## Обзор

Концептуальный слой NodeRAG представляет собой высокоуровневую абстракцию знаний, извлеченных из документов. Он создается через многоэтапную семантическую обработку, которая трансформирует сырой текст в структурированное представление знаний.

---

## Семантическая Иерархия

### Уровень 0: Raw Documents
```
Неструктурированный текст документов
```

### Уровень 1: Text Units
```
Семантически целостные chunks текста
- Размер: ~1048 tokens
- Граница: Параграфы, предложения
```

### Уровень 2: Semantic Units + Entities
```
Извлеченные факты и сущности
- Semantic Units: События, процессы, утверждения
- Entities: Именованные объекты
- Relationships: Связи между сущностями
```

### Уровень 3: Attributes
```
Богатые описания ключевых сущностей
- Только для важных узлов
- Агрегация контекста из графа
```

### Уровень 4: High-Level Elements (Концептуальный слой)
```
Абстрактные концепты и темы
- Темы
- Идеи
- Теории
- Паттерны
```

---

## Формирование Концептуального Слоя

### 1. Community Detection

**Алгоритм**: Leiden Algorithm

**Файл**: `NodeRAG/build/pipeline/summary_generation.py`

#### Preprocessing

1. **Конвертация графа**:
   ```python
   G_ig = IGraph(NetworkX_graph).to_igraph()
   ```
   - NetworkX → igraph
   - igraph оптимизирован для community detection

2. **Фильтрация узлов**:
   - Отбор узлов с embeddings
   - Semantic units и attributes в приоритете

#### Leiden Algorithm

```python
partition = la.find_partition(G_ig, la.ModularityVertexPartition)
```

**Особенности**:
- **Objective**: Максимизация modularity
- **Метод**: Iterative refinement
- **Преимущества**:
  - Лучше чем Louvain
  - Гарантия well-connected communities
  - Быстрая конвергенция

**Modularity**:
```
Q = 1/(2m) Σ[A_ij - (k_i × k_j)/(2m)] × δ(c_i, c_j)

где:
- m: количество ребер
- A_ij: adjacency matrix
- k_i, k_j: degrees узлов
- c_i, c_j: communities узлов
- δ: 1 если в одном community, 0 иначе
```

#### Результат

```python
partition = [
    [node1, node2, node5],      # Community 0
    [node3, node7, node8],      # Community 1
    [node4, node6, node9],      # Community 2
    ...
]
```

Каждый community = кластер семантически близких узлов.

---

### 2. Community Characterization

Для каждого community:

#### a) Сбор Контекста

**Типы узлов для анализа**:
1. **Semantic units** в community
2. **Attributes** сущностей в community
3. **Attribute neighbors** ключевых сущностей

```python
used_unit = []
for node in community_nodes:
    if G.nodes[node]['type'] == 'semantic_unit':
        used_unit.append(node)
    elif G.nodes[node]['type'] == 'attribute':
        used_unit.append(node)
    elif G.nodes[node].get('attribute'):
        # Добавить attribute соседей
        for neighbour in G.neighbors(node):
            if G.nodes[neighbour]['type'] == 'attribute':
                used_unit.append(neighbour)
```

**Обоснование**:
- Semantic units: факты и события
- Attributes: детальные описания
- Комбинация дает полную картину community

#### b) Token Management

**Проблема**: Большие communities → превышение context window

**Решение**: Приоритизация по важности

```python
def get_important_node_query():
    weights_dict = SortedDict()

    # Вычисление важности = сумма весов соседей
    for node in used_unit:
        weight = sum(
            G.nodes[neighbour]['weight']
            for neighbour in G.neighbors(node)
        )
        weights_dict[node] = weight

    # Добавление в порядке убывания
    content = ''
    for node in reversed(weights_dict):
        temp_content = content + mapper.get(node, 'context') + '\n'
        if token_counter.token_limit(temp_content):
            break
        content = temp_content

    return content
```

**Критерий важности**: Узлы с высоковесными соседями важнее.

#### c) LLM Extraction

**Промпт**: `community_summary`

**Задача**:
- Извлечь distinct categories высокоуровневой информации
- Концепты, темы, теории, воздействия, инсайты
- Избегать redundancy
- Обеспечить diversity

**Output**:
```json
{
  "high_level_elements": [
    {"title": "...", "description": "..."},
    {"title": "...", "description": "..."}
  ]
}
```

#### d) Создание High-Level Element Nodes

Для каждого элемента:

```python
he = High_level_elements(description, title, config)

# Два узла: content и title
G.add_node(he.hash_id, type='high_level_element', weight=1)
G.add_node(he.title_hash_id, type='high_level_element_title', weight=1,
           related_node=he.hash_id)

# Связь title ↔ content
G.add_edge(he.hash_id, he.title_hash_id, weight=1)
```

**Обоснование двух узлов**:
- **Title node**: Быстрый поиск по ключевым словам
- **Content node**: Детальная информация для retrieval

---

### 3. Embedding Generation

```python
# Для каждого high-level element
context = high_level_element.context  # Description
embedding = await embedding_client([context])
high_level_element.store_embedding(embedding)
```

**Batch processing**:
```python
batch_size = config.embedding_batch_size

for i in range(0, len(high_level_elements), batch_size):
    batch = high_level_elements[i:i+batch_size]
    contexts = [he.context for he in batch]
    embeddings = await embedding_client(contexts)

    for he, emb in zip(batch, embeddings):
        he.store_embedding(emb)
```

---

### 4. Linking Strategy

Связывание High-Level Elements с исходными узлами community.

#### Режим 1: Малые Communities

**Условие**:
```python
threshold = (len(all_nodes) + len(high_level_elements)) / centroids
if threshold <= config.Hcluster_size:
    # Прямое связывание
```

**Действие**:
```python
for he in high_level_elements:
    for node in he.related_nodes:
        G.add_edge(node, he.hash_id, weight=1)
```

**Обоснование**: При малом количестве узлов прямое связывание эффективно.

#### Режим 2: Большие Communities

**Условие**:
```python
if threshold > config.Hcluster_size:
    # KMeans кластеризация
```

**Процесс**:

1. **Объединение embeddings**:
   ```python
   # Embeddings semantic units и entities из communities
   node_embeddings = [mapper.embeddings[node] for node in all_nodes]

   # Embeddings high-level elements
   he_embeddings = [he.embedding for he in high_level_elements]

   # Concatenate
   all_embeddings = np.vstack([he_embeddings, node_embeddings])
   ```

2. **KMeans**:
   ```python
   centroids = ceil(sqrt(len(all_nodes) + len(high_level_elements)))

   kmeans = faiss.Kmeans(d=embedding_dim, k=centroids)
   kmeans.train(all_embeddings.astype(np.float32))
   _, cluster_labels = kmeans.assign(all_embeddings)
   ```

3. **Связывание**:
   ```python
   he_labels = cluster_labels[:len(high_level_elements)]
   node_labels = cluster_labels[len(high_level_elements):]

   for i, he in enumerate(high_level_elements):
       for j, node in enumerate(all_nodes):
           # Только если в одном кластере
           if he_labels[i] == node_labels[j]:
               if node in he.related_nodes:
                   G.add_edge(node, he.hash_id, weight=1)
   ```

**Обоснование**:
- Уменьшает количество ребер
- Связывает только семантически близкие узлы
- Эффективнее для больших графов

**Вычисление centroids**:
```python
centroids = ceil(sqrt(total_nodes))
```
Heuristic для баланса между granularity и performance.

---

## Семантическая Декомпозиция

### Цели

1. **Segmentation**: Разбить текст на semantic units
2. **Extraction**: Извлечь entities и relationships
3. **Normalization**: Привести к стандартному формату

### Принципы

#### 1. Event-Based Segmentation

Каждый semantic unit = одно событие, факт или концепт.

**Пример**:
```
Original: "Dr. Emily attended a conference. She presented research.
           She also explored partnerships."

Semantic Units:
1. "Dr. Emily attended a conference."
2. "Dr. Emily presented research at the conference."
3. "Dr. Emily explored partnerships."
```

**Обоснование**: Atomic facts легче для retrieval и reasoning.

#### 2. Paraphrasing

Semantic unit = paraphrase, сохраняющий все детали.

**Цель**:
- Сжатие без потери информации
- Улучшение clarity
- Стандартизация формулировок

**Пример**:
```
Original: "During her visit to Paris in September 2024, Dr. Emily Roberts,
           who is a researcher, attended the International Conference on
           Renewable Energy."

Paraphrase: "In September 2024, Dr. Emily Roberts attended the International
             Conference on Renewable Energy in Paris."
```

#### 3. Entity Extraction from Original

**Критично**: Entities извлекаются из ORIGINAL text, не из paraphrase.

**Обоснование**:
- Сохранение точных формулировок
- Избежание hallucinations от LLM
- Consistency в именовании

#### 4. UPPERCASE Normalization

Все entities → UPPERCASE.

**Обоснование**:
- Дедупликация: "Emily Roberts" = "EMILY ROBERTS"
- Визуальное выделение
- Стандарт для NER систем

#### 5. Temporal Entity Format

**Правило**: Не заполнять missing parts.

**Примеры**:
- "September 2024" → "2024-09"
- "2024" → "2024"
- "September" → "09" (если год не указан, используем месяц)
- "15th" → "15" (только день)

**Обоснование**: Избежание false precision.

#### 6. Descriptive Relationships

Relationship type = descriptive sentence, не просто глагол.

**Примеры**:

❌ Плохо:
```
"DR. EMILY, attends, CONFERENCE"
```

✅ Хорошо:
```
"DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
"DR. EMILY ROBERTS, presented research on solar panel efficiency at, CONFERENCE"
```

**Обоснование**: Больше контекста → лучше retrieval.

---

## Формирование Концептов

### Абстракция

High-Level Elements = абстракция от конкретных фактов к общим концептам.

**Пример**:

**Конкретные факты** (Semantic Units):
- "Dr. Emily Roberts attended renewable energy conference"
- "Dr. Emily Roberts presented solar panel research"
- "Dr. John Miller documented species in Amazon"
- "Both contribute to environmental conservation"

**Концепт** (High-Level Element):
```
Title: "Environmental Conservation Through Scientific Research"

Description: "This concept encompasses interdisciplinary scientific efforts
aimed at environmental protection. It includes renewable energy research
focused on solar technology improvements, biodiversity conservation through
field documentation, and international collaboration via academic conferences.
The work demonstrates how diverse scientific approaches contribute to the
overarching goal of environmental sustainability."
```

### Характеристики Концептов

#### 1. Abstraction Level

Концепты находятся на более высоком уровне абстракции чем факты.

**Semantic Unit**: "Dr. Emily presented solar panel research"
**Concept**: "Renewable Energy Research and Innovation"

#### 2. Multi-faceted

Концепт объединяет multiple perspectives.

**Facets**:
- Technological innovation
- Academic collaboration
- Environmental impact
- Policy implications

#### 3. Non-redundant

Похожие концепты объединяются в один comprehensive concept.

**Merge Example**:
```
"Solar Energy Research" + "Renewable Technologies" →
"Renewable Energy Research and Innovation"
```

#### 4. Significance-based

Отбираются концепты с наибольшей significance и diversity.

**Критерии**:
- Частота упоминаний в community
- Связь с множественными узлами
- Уникальность perspective

---

## Семантическая Близость

### В Пространстве Embeddings

**Метрика**: Cosine similarity

```python
similarity = dot(embedding1, embedding2) / (norm(embedding1) × norm(embedding2))
```

**Интерпретация**:
- similarity > 0.9: Очень близкие концепты
- 0.7 < similarity < 0.9: Связанные концепты
- similarity < 0.7: Далекие концепты

### В Графе

**Метрики**:

1. **Path distance**: Длина кратчайшего пути
2. **Common neighbors**: Количество общих соседей
3. **Community membership**: Принадлежность к community

**Комбинация**:
```python
semantic_similarity = (
    α × embedding_similarity +
    β × graph_proximity +
    γ × community_overlap
)
```

---

## Концептуальный Граф

### Структура

```
High-Level Element (Concept)
    ├─→ Semantic Unit 1
    ├─→ Semantic Unit 2
    ├─→ Entity A
    ├─→ Entity B
    └─→ Attribute C

High-Level Element Title
    └─→ High-Level Element (Content)
```

### Свойства

**High-Level Element узлы**:
- `type`: 'high_level_element'
- `weight`: 1 (может инкрементироваться)
- `embedding`: ✓
- `related_nodes`: List связанных узлов

**High-Level Element Title узлы**:
- `type`: 'high_level_element_title'
- `weight`: 1
- `related_node`: hash_id content узла

### Traversal

**От концепта к фактам**:
```python
concept_node = "H00001"
related_nodes = list(G.neighbors(concept_node))

for node in related_nodes:
    if G.nodes[node]['type'] == 'semantic_unit':
        fact = mapper.get(node, 'context')
```

**От факта к концептам**:
```python
semantic_unit = "S00042"
related_concepts = [
    node for node in G.neighbors(semantic_unit)
    if G.nodes[node]['type'] == 'high_level_element'
]
```

---

## Retrieval Strategy

### Многоуровневый Retrieval

#### Уровень 1: Концептуальный поиск

```python
# Embedding query
query_embedding = embedding_model(user_query)

# Поиск похожих концептов
similar_concepts = HNSW_index.search(query_embedding, k=5)

# Это high-level elements с наибольшим semantic similarity
```

#### Уровень 2: Expansion через граф

```python
relevant_nodes = set()

for concept in similar_concepts:
    # Получить все связанные узлы
    neighbors = G.neighbors(concept)

    for neighbor in neighbors:
        if G.nodes[neighbor]['type'] in ['semantic_unit', 'attribute']:
            relevant_nodes.add(neighbor)
```

#### Уровень 3: Detailed retrieval

```python
# Embedding search на semantic units и attributes
detailed_results = []

for node in relevant_nodes:
    node_embedding = mapper.embeddings[node]
    similarity = cosine_similarity(query_embedding, node_embedding)

    if similarity > threshold:
        detailed_results.append({
            'node': node,
            'content': mapper.get(node, 'context'),
            'similarity': similarity
        })

# Сортировка по similarity
detailed_results.sort(key=lambda x: x['similarity'], reverse=True)
```

### Преимущества Многоуровневого Подхода

1. **Efficiency**: Сначала фильтрация на концептуальном уровне
2. **Coverage**: Expansion через граф находит related info
3. **Precision**: Финальный ranking на детальном уровне
4. **Explainability**: Можно показать цепочку: Query → Concept → Facts

---

## Пример: Полный Flow

### Входные данные

**Документ**:
```
Renewable energy research is crucial for addressing climate change. In
September 2024, Dr. Emily Roberts from the European Research Institute
attended the International Conference on Renewable Energy in Paris. She
presented groundbreaking research on improving solar panel efficiency by
15%. The conference brought together over 500 researchers from 40 countries.
Dr. Roberts also explored partnerships with European companies interested
in commercializing her innovations. Meanwhile, her colleague Dr. John
Miller was conducting field research in the Amazon Rainforest, documenting
new species and studying deforestation impacts. Both researchers contribute
significantly to environmental conservation efforts through their distinct
yet complementary approaches.
```

### Семантическая Обработка

**Text Units** (1 unit, так как < 1048 tokens):
```
T00001: [весь текст]
```

**Semantic Units** (LLM decomposition):
```
S00001: "Renewable energy research is crucial for addressing climate change."
S00002: "In September 2024, Dr. Emily Roberts attended the International
         Conference on Renewable Energy in Paris."
S00003: "Dr. Emily Roberts presented research on improving solar panel
         efficiency by 15%."
S00004: "The conference brought together over 500 researchers from 40 countries."
S00005: "Dr. Emily Roberts explored partnerships with European companies for
         commercializing her innovations."
S00006: "Dr. John Miller conducted field research in the Amazon Rainforest."
S00007: "Dr. John Miller documented new species and studied deforestation impacts."
S00008: "Both Dr. Roberts and Dr. Miller contribute to environmental conservation
         through complementary approaches."
```

**Entities**:
```
E00001: "RENEWABLE ENERGY RESEARCH"
E00002: "CLIMATE CHANGE"
E00003: "DR. EMILY ROBERTS"
E00004: "EUROPEAN RESEARCH INSTITUTE"
E00005: "2024-09"
E00006: "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
E00007: "PARIS"
E00008: "SOLAR PANEL EFFICIENCY"
E00009: "EUROPEAN COMPANIES"
E00010: "DR. JOHN MILLER"
E00011: "AMAZON RAINFOREST"
E00012: "NEW SPECIES"
E00013: "DEFORESTATION"
E00014: "ENVIRONMENTAL CONSERVATION"
```

**Relationships**:
```
R00001: "RENEWABLE ENERGY RESEARCH, is crucial for addressing, CLIMATE CHANGE"
R00002: "DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE"
R00003: "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE"
R00004: "DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY"
R00005: "DR. EMILY ROBERTS, explored partnerships with, EUROPEAN COMPANIES"
R00006: "DR. JOHN MILLER, conducted field research in, AMAZON RAINFOREST"
R00007: "DR. JOHN MILLER, documented, NEW SPECIES"
R00008: "DR. JOHN MILLER, studied impacts of, DEFORESTATION"
R00009: "DR. EMILY ROBERTS, contributes to, ENVIRONMENTAL CONSERVATION"
R00010: "DR. JOHN MILLER, contributes to, ENVIRONMENTAL CONSERVATION"
```

### Graph Construction

Граф с узлами:
- 8 Semantic Units
- 14 Entities
- 10 Relationships

Ребра:
- Semantic Units → Entities (множественные)
- Relationships → Entities (source/target)

### Community Detection

**Leiden algorithm** находит communities:

**Community 1**: Renewable energy research
- S00001, S00002, S00003, S00004, S00005
- E00001, E00003, E00006, E00008, E00009
- R00001, R00003, R00004, R00005

**Community 2**: Biodiversity research
- S00006, S00007
- E00010, E00011, E00012, E00013
- R00006, R00007, R00008

**Community 3**: Environmental conservation (overlap)
- S00008
- E00003, E00010, E00014
- R00009, R00010

### Концептуальный Слой

**High-Level Elements**:

```
H00001:
  Title: "Renewable Energy Research and Innovation"
  Description: "Cutting-edge research in sustainable energy technologies,
                focusing on solar panel efficiency improvements and
                international scientific collaboration. This includes
                participation in global conferences, presentation of
                breakthrough findings, and partnerships with industry for
                commercialization."
  Related: Community 1 узлы

H00002:
  Title: "Biodiversity Conservation and Ecosystem Research"
  Description: "Field research in critical ecosystems like the Amazon
                Rainforest aimed at documenting species diversity and
                understanding environmental impacts of deforestation. This
                work is essential for informing conservation strategies."
  Related: Community 2 узлы

H00003:
  Title: "Interdisciplinary Environmental Science"
  Description: "The integration of various scientific disciplines working
                towards environmental protection. This concept encompasses
                both technological innovation in renewable energy and
                ecological research in biodiversity, demonstrating how
                diverse approaches contribute to the common goal of
                environmental sustainability."
  Related: Community 3 узлы + overlap с 1 и 2
```

### Retrieval Example

**Query**: "How are researchers addressing environmental challenges?"

**Step 1**: Концептуальный поиск
```
Query embedding → [0.12, 0.45, -0.23, ...]
Similarity с H00003: 0.92 (highest)
Similarity с H00001: 0.78
Similarity с H00002: 0.81
```

**Step 2**: Graph expansion
```
H00003 neighbors:
- S00008 (environmental conservation)
- E00003 (Dr. Emily Roberts)
- E00010 (Dr. John Miller)
- E00014 (environmental conservation)

Expansion через E00003 и E00010:
- S00003 (solar panel research)
- S00007 (deforestation research)
- A00001 (attribute для Dr. Emily Roberts)
- A00002 (attribute для Dr. John Miller)
```

**Step 3**: Ranking и return
```
1. H00003 + description (0.92)
2. S00008 context (0.89)
3. A00001 attribute (0.87)
4. A00002 attribute (0.86)
5. S00003 context (0.82)
```

**Answer Construction**:
```
"Researchers are addressing environmental challenges through interdisciplinary
approaches. Dr. Emily Roberts focuses on renewable energy innovation,
particularly solar panel efficiency improvements, collaborating with industry
partners. Dr. John Miller conducts field research in ecosystems like the Amazon,
documenting biodiversity and deforestation impacts. Both contribute to
environmental conservation through their complementary scientific work."
```

---

## Заключение

Концептуальный слой NodeRAG:

1. **Автоматически генерируется** через community detection + LLM
2. **Представляет высокоуровневые темы** и концепты
3. **Связан с исходными фактами** через граф
4. **Оптимизирован для retrieval** через embeddings и HNSW
5. **Обеспечивает explainability** через multi-hop reasoning

Это позволяет системе отвечать как на конкретные (fact-based), так и на абстрактные (concept-based) вопросы.
