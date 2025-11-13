# Graph Construction and Concatenation: Построение графа знаний

## Концептуальный обзор

**Graph Construction** — это процесс преобразования текстового корпуса в **гетерогенный knowledge graph** с узлами разных типов (entities, semantic units, relationships, attributes, high-level elements) и взвешенными рёбрами. **Graph Concatenation** объединяет этот граф с **HNSW graph** для создания **унифицированного графа** для Personalized PageRank.

### Философская сущность

В контексте [парадигмы вопроса как ключа](../research/question-as-key-paradigm.md), граф представляет **структуру знания** — онтологическую сеть концепций и их связей:

```
Неструктурированный текст (хаос)
            ↓
    [Text Decomposition]
            ↓
Семантические единицы (атомы знания)
            ↓
   [Graph Construction]
            ↓
Knowledge Graph (структура знания)
```

Это переход от **линейной** репрезентации (текст) к **сетевой** репрезентации (граф) — от последовательности к структуре.

---

## Роль в системе

### Зачем граф?

**Проблема 1: Текст — линейная структура**
```
"Dr. Roberts presented findings at the conference on solar energy."
```

**Граф — многомерная структура**:
```
      [DR. ROBERTS]
         /    \
     presented  is_a
       /          \
 [FINDINGS]      [RESEARCHER]
      |              |
    about         works_on
      |              |
 [SOLAR ENERGY] ← [RENEWABLE ENERGY]
      |
  discussed_at
      |
 [CONFERENCE]
```

**Преимущество**: Можно **навигировать** в разных направлениях, находя связи.

**Проблема 2: Embedding similarity — аналоговая**

```
"solar energy" близко к "renewable energy" в embedding space
```

**Граф — символическая структура**:
```
[SOLAR ENERGY] ---is_type_of---> [RENEWABLE ENERGY]
```

**Преимущество**: **Явные** отношения вместо имплицитных расстояний.

**Решение NodeRAG**: **Hybrid approach** — комбинирование graph structure + embeddings.

---

## Типы узлов в графе

### 1. Semantic Units (Семантические единицы)

**Определение**: Атомарные факты или утверждения, извлечённые из текста.

**Пример**:
```
"Dr. Emily Roberts, a leading researcher in renewable energy,
 presented groundbreaking findings on solar panel efficiency
 improvements at the International Renewable Energy Conference."
```

**Характеристики**:
- **Размер**: 1-3 предложения (~50-200 токенов)
- **Тип узла**: `semantic_unit`
- **Embedding**: ✓ (используется в HNSW)
- **Вес**: Число упоминаний в корпусе

### 2. Entities (Сущности)

**Определение**: Именованные объекты, люди, организации, концепции.

**Примеры**:
```
- "DR. EMILY ROBERTS" (person)
- "SOLAR PANEL EFFICIENCY" (concept)
- "INTERNATIONAL RENEWABLE ENERGY CONFERENCE" (event)
```

**Характеристики**:
- **Извлечение**: LLM text decomposition
- **Тип узла**: `entity`
- **Embedding**: ✓ (используется в HNSW)
- **Вес**: Число упоминаний

### 3. Relationships (Отношения)

**Определение**: Семантические связи между entities.

**Примеры**:
```
- [DR. ROBERTS] ---presented---> [FINDINGS]
- [SOLAR PANELS] ---is_type_of---> [RENEWABLE ENERGY]
- [CONFERENCE] ---located_in---> [COPENHAGEN]
```

**Характеристики**:
- **Формат**: "(entity1, relation, entity2)"
- **Тип узла**: `relationship`
- **Embedding**: ✗ (не индексируется в HNSW)
- **Вес**: Число упоминаний

### 4. Attributes (Атрибуты)

**Определение**: Детальные нарративные описания важных entities.

**Пример**:
```
Entity: "DR. EMILY ROBERTS"

Attribute: "Dr. Emily Roberts is a distinguished researcher
specializing in renewable energy technologies, with particular
expertise in solar panel efficiency. She has published over 50
papers in leading journals and is recognized internationally
for her contributions to sustainable energy solutions."
```

**Характеристики**:
- **Генерация**: LLM attribute generation (только для important entities)
- **Тип узла**: `attribute`
- **Embedding**: ✓ (используется в HNSW)
- **Вес**: 1

### 5. High-level Elements (Элементы высокого уровня)

**Определение**: Абстрактные темы, извлечённые из сообществ узлов.

**Пример**:
```
Community: {SU-1, SU-2, ..., SU-50} (все о возобновляемой энергии)

High-level Element (Title): "RENEWABLE ENERGY RESEARCH AND INNOVATIONS"

High-level Element (Description): "This cluster encompasses research
on various renewable energy technologies, including solar, wind, and
hydroelectric power, with emphasis on efficiency improvements and
practical applications."
```

**Характеристики**:
- **Генерация**: Leiden community detection + LLM summarization
- **Тип узла**: `high_level_element` и `high_level_element_title`
- **Embedding**: ✓ (используется в HNSW)
- **Вес**: Число сообществ с данной темой

### 6. Texts (Исходные тексты)

**Определение**: Оригинальные текстовые фрагменты (chunks) из корпуса.

**Характеристики**:
- **Тип узла**: `text`
- **Embedding**: ✗
- **Связь**: text → semantic_units (один text порождает несколько SU)

---

## Структура графа

### Граф как мультиграф

NodeRAG использует **неориентированный мультиграф**:

```python
G = nx.Graph()  # Неориентированный граф
```

**Свойства**:
- **Неориентированный**: Рёбра двунаправленные (A↔B, не A→B)
- **Взвешенный**: Каждое ребро имеет атрибут `weight`
- **Гетерогенный**: Узлы разных типов (`type` attribute)
- **Мультиграф** (концептуально): Несколько рёбер между узлами объединяются (суммирование весов)

### Типы рёбер

| Ребро | Значение | Вес |
|-------|----------|-----|
| `semantic_unit ↔ entity` | SU упоминает entity | 1 |
| `semantic_unit ↔ relationship` | SU содержит relationship | 1 |
| `entity ↔ relationship` | Entity участвует в relation | 1 |
| `entity ↔ attribute` | Attribute описывает entity | 1 |
| `semantic_unit ↔ high_level_element` | SU в сообществе HLE | 1 |
| `high_level_element ↔ high_level_element_title` | Title для description | 1 |
| `node ↔ node` (HNSW) | HNSW Layer 0 соседи | 1 |

### Пример подграфа

```
          [TEXT-001]
               │
       ┌───────┼───────┐
       │       │       │
    [SU-1]  [SU-2]  [SU-3]
       │       │       │
       ├───────┼───────┤
       │       │       │
    [ENT-A] [REL-X] [ENT-B]
       │               │
   [ATTR-A]        [ATTR-B]
       │
    [HLE-T]
       │
    [HLE-D]
```

**Интерпретация**:
- TEXT-001 разбит на 3 semantic units
- SU-1 и SU-2 упоминают ENT-A
- REL-X связывает ENT-A и ENT-B
- ENT-A важная (имеет ATTR-A)
- SU-1, SU-2, SU-3 в одном сообществе → HLE (title + description)

---

## Алгоритм построения графа

### Фаза 1: Инициализация

```python
G = nx.Graph()
```

### Фаза 2: Добавление semantic units

```python
for semantic_unit in semantic_units:
    if G.has_node(semantic_unit.hash_id):
        # Узел уже существует (дубликат), увеличиваем вес
        G.nodes[semantic_unit.hash_id]['weight'] += 1
    else:
        # Новый узел
        G.add_node(
            semantic_unit.hash_id,
            type='semantic_unit',
            weight=1
        )

    # Соединяем с text
    if not G.has_edge(text.hash_id, semantic_unit.hash_id):
        G.add_edge(text.hash_id, semantic_unit.hash_id, weight=1)
```

**Дедупликация**:
- Semantic units идентифицируются по **hash_id** (SHA256 хэш контента)
- Дубликаты увеличивают `weight` вместо создания нового узла

### Фаза 3: Добавление entities

```python
for entity in entities:
    if G.has_node(entity.hash_id):
        G.nodes[entity.hash_id]['weight'] += 1
    else:
        G.add_node(entity.hash_id, type='entity', weight=1)

    # Соединяем entity с semantic unit
    if not G.has_edge(semantic_unit.hash_id, entity.hash_id):
        G.add_edge(semantic_unit.hash_id, entity.hash_id, weight=1)
    else:
        # Ребро уже существует, увеличиваем вес
        G[semantic_unit.hash_id][entity.hash_id]['weight'] += 1
```

### Фаза 4: Добавление relationships

```python
for relationship in relationships:
    if G.has_node(relationship.hash_id):
        G.nodes[relationship.hash_id]['weight'] += 1
    else:
        G.add_node(relationship.hash_id, type='relationship', weight=1)

    # Соединяем relationship с semantic unit
    if not G.has_edge(semantic_unit.hash_id, relationship.hash_id):
        G.add_edge(semantic_unit.hash_id, relationship.hash_id, weight=1)
    else:
        G[semantic_unit.hash_id][relationship.hash_id]['weight'] += 1

    # Соединяем relationship с entities
    for entity in relationship.entities:
        if G.has_node(entity):
            if not G.has_edge(relationship.hash_id, entity):
                G.add_edge(relationship.hash_id, entity, weight=1)
            else:
                G[relationship.hash_id][entity]['weight'] += 1
```

**Парсинг relationships**:
```python
# Формат: "(entity1, relation, entity2)"
# Пример: "(DR. ROBERTS, presented, FINDINGS)"

import re

match = re.search(r'\((.*?),\s*(.*?),\s*(.*?)\)', relationship_str)
if match:
    entity1, relation, entity2 = match.groups()
```

### Фаза 5: Добавление attributes

```python
for attribute in attributes:
    G.add_node(attribute.hash_id, type='attribute', weight=1)

    # Соединяем с entity
    entity = attribute.node
    G.nodes[entity]['attributes'] = [attribute.hash_id]
    G.add_edge(entity, attribute.hash_id, weight=1)
```

**Только для important entities**: См. [K-Core and Centrality](./k-core-centrality.md)

### Фаза 6: Добавление high-level elements

```python
for high_level_element in high_level_elements:
    # Description node
    G.add_node(
        high_level_element.hash_id,
        type='high_level_element',
        weight=1
    )

    # Title node
    G.add_node(
        high_level_element.title_hash_id,
        type='high_level_element_title',
        weight=1,
        related_node=high_level_element.hash_id
    )

    # Title ↔ Description
    G.add_edge(
        high_level_element.hash_id,
        high_level_element.title_hash_id,
        weight=1
    )

    # HLE ↔ Community nodes
    for node in high_level_element.related_node:
        G.add_edge(node, high_level_element.hash_id, weight=1)
```

**Community clustering**: Для больших сообществ используется k-means (см. [Leiden Algorithm](./leiden-community-detection.md))

---

## Graph Concatenation: Объединение с HNSW

### Зачем объединение?

**Проблема**: HNSW graфоказывает **embedding similarity**, knowledge graph — **structural relationships**.

**Решение**: Объединить оба графа для **hybrid navigation**:

```
Unified Graph = Knowledge Graph ∪ HNSW Graph (Layer 0)
```

**Преимущество**: PPR может распространяться через **оба типа связей** одновременно.

### Класс `GraphConcat`

**Файл**: `NodeRAG/utils/graph_operator.py:50-97`

```python
class GraphConcat():

    def __init__(self, base_graph: nx.Graph = None):
        if base_graph is None:
            raise Exception('Base graph is None')

        self.graph = base_graph

    def concat(self, hnsw_graph: nx.Graph):
        if hnsw_graph is None:
            raise Exception('HNSW graph is None')

        # Добавляем узлы из HNSW graph
        for node, data in hnsw_graph.nodes(data=True):
            if node not in self.graph:
                self.graph.add_node(node, **data)

        # Добавляем рёбра из HNSW graph
        for u, v, data in hnsw_graph.edges(data=True):
            if self.graph.has_edge(u, v):
                # Ребро уже существует, увеличиваем вес
                self.graph[u][v]['weight'] += 1
            else:
                # Новое ребро
                self.graph.add_edge(u, v, weight=1)

        return self.graph
```

**Логика**:
1. **Узлы**: Если узел из HNSW уже в knowledge graph, **не дублируем**
2. **Рёбра**: Если ребро уже существует (например, через semantic unit), **увеличиваем вес**

### HNSW Graph Export

**Файл**: `NodeRAG/utils/HNSW.py:24-34`

```python
@property
def nxgraphs(self):
    graph_layer_0 = self.hnsw.get_layer_graph(0)

    if graph_layer_0 is not None:
        if self._nxgraphs is None:
            self._nxgraphs = nx.Graph()

            # get_layer_graph возвращает {node_id: [neighbor_ids]}
            for id, neighbors in graph_layer_0.items():
                for neighbor in neighbors:
                    self._nxgraphs.add_edge(
                        self.id_map[id],
                        self.id_map[neighbor]
                    )

        return self._nxgraphs
    else:
        return None
```

**HNSW Layer 0**: Самый плотный слой, содержит **все узлы** и их M ближайших соседей (M=64).

### Unbalance Adjust

**Проблема**: После concatenation, некоторые узлы имеют **очень высокую степень** (degree):

```
node A: degree = 200 (100 HNSW + 100 knowledge graph)
node B: degree = 5
```

**Эффект на PPR**: Узлы с высокой степенью **разбавляют** PPR score соседей.

**Решение**: `unbalance_adjust` — ограничиваем веса рёбер:

```python
@staticmethod
def unbalance_adjust(graph: nx.Graph):
    for node in graph.nodes():
        degree = graph.degree(node)

        if degree > 0:
            weight_factor = 1 / degree

            for neighbor in graph.neighbors(node):
                # Если вес ребра > 1/degree, урезаем до 1/degree
                if graph[node][neighbor]['weight'] > weight_factor:
                    graph[node][neighbor]['weight'] = weight_factor

    return graph
```

**Интерпретация**:
```
degree = 100 → max edge weight = 1/100 = 0.01
degree = 10  → max edge weight = 1/10 = 0.1
```

Узлы с высокой степенью имеют **более слабые рёбра**.

**Эффект**: PPR распределяется **более равномерно**, high-degree узлы не доминируют.

---

## Реализация в NodeRAG

### Файл: `NodeRAG/search/search.py:59-75`

```python
def load_graph(self):
    # Загружаем base knowledge graph
    if os.path.exists(self.config.base_graph_path):
        G = storage.load(self.config.base_graph_path)
    else:
        raise Exception('No base graph found.')

    # Загружаем HNSW graph
    if os.path.exists(self.config.hnsw_graph_path):
        HNSW_graph = storage.load(self.config.hnsw_graph_path)
    else:
        raise Exception('No HNSW graph found.')

    # Объединение
    if self.config.unbalance_adjust:
        G = GraphConcat(G).concat(HNSW_graph)
        return GraphConcat.unbalance_adjust(G)

    return GraphConcat(G).concat(HNSW_graph)
```

**Config**: `unbalance_adjust=True` (default)

---

## Статистика графа

### Типичные размеры (NodeRAG)

**Корпус**: 10K documents, ~1M tokens

| Тип узла | Число | % |
|----------|-------|---|
| Semantic Units | ~200K | 40% |
| Entities | ~100K | 20% |
| Relationships | ~150K | 30% |
| Attributes | ~10K | 2% |
| High-level Elements (titles) | ~500 | 0.1% |
| High-level Elements (descriptions) | ~500 | 0.1% |
| Texts | ~10K | 2% |
| **Total** | ~471K | 100% |

### Рёбра

| Тип рёбер | Число | % |
|-----------|-------|---|
| SU ↔ Entity | ~400K | 30% |
| SU ↔ Relationship | ~300K | 22% |
| Entity ↔ Relationship | ~200K | 15% |
| Entity ↔ Attribute | ~10K | 1% |
| SU ↔ HLE | ~100K | 7% |
| HLE ↔ HLE_title | ~500 | 0.04% |
| HNSW edges (Layer 0) | ~300K | 22% |
| Text ↔ SU | ~50K | 4% |
| **Total** | ~1.36M | 100% |

### Средняя степень

```
avg_degree = (2 × E) / V = (2 × 1.36M) / 471K ≈ 5.8
```

**Интерпретация**: Граф достаточно разреженный (avg_degree << N).

---

## Производительность

### Сложность построения

| Операция | Сложность | NodeRAG |
|----------|-----------|---------|
| Add node | O(1) | ~471K nodes |
| Add edge | O(1) | ~1.36M edges |
| Check node exists | O(1) (hash) | ~471K checks |
| Check edge exists | O(1) (hash) | ~1.36M checks |
| **Total** | O(V + E) | O(1.8M) |

### Бенчмарки

**Корпус**: 10K documents

| Этап | Время | Операции |
|------|-------|----------|
| Text decomposition | ~30-60 min | LLM calls (~10K) |
| Graph construction | ~2-5 min | Add nodes/edges |
| Attribute generation | ~15-30 min | LLM calls (~10K) |
| Community detection | ~30-60 sec | Leiden algorithm |
| Community summarization | ~10-20 min | LLM calls (~500) |
| HNSW concatenation | ~10-30 sec | Graph merge |
| **Total** | ~60-120 min | - |

**Hardware**: CPU (graph operations), API (LLM calls)

---

## Библиотеки

### NetworkX

**Библиотека**: `networkx==3.4.2`

**Основные операции**:

```python
import networkx as nx

# Создание графа
G = nx.Graph()

# Добавление узлов с атрибутами
G.add_node('node1', type='entity', weight=1)

# Добавление рёбер с весами
G.add_edge('node1', 'node2', weight=1)

# Проверка существования
if G.has_node('node1'):
    ...

if G.has_edge('node1', 'node2'):
    G['node1']['node2']['weight'] += 1

# Статистика
print(f"Nodes: {G.number_of_nodes()}")
print(f"Edges: {G.number_of_edges()}")
print(f"Avg degree: {sum(dict(G.degree()).values()) / G.number_of_nodes()}")

# Сохранение/загрузка
import pickle

with open('graph.pkl', 'wb') as f:
    pickle.dump(G, f)

with open('graph.pkl', 'rb') as f:
    G = pickle.load(f)
```

**Документация**: [Graph Operations Dependencies](../dependencies/graph-operations.md)

---

## Связь с другими алгоритмами

### 1. Text Decomposition → Graph Construction

LLM decomposition создаёт **входы** для graph construction:

```python
# Text decomposition
decomposed = llm.decompose(text)

# Извлекаем компоненты
semantic_units = decomposed['semantic_units']
entities = decomposed['entities']
relationships = decomposed['relationships']

# Строим граф
for su in semantic_units:
    G.add_node(su.hash_id, type='semantic_unit', weight=1)

for entity in entities:
    G.add_node(entity.hash_id, type='entity', weight=1)
    G.add_edge(su.hash_id, entity.hash_id, weight=1)
```

См. [Text Decomposition](../transform/indexing-transformations.md)

### 2. Graph Construction → K-Core + Betweenness

После построения графа, выявляем important nodes:

```python
G = build_graph(...)

important_nodes = K_core(G) ∪ Betweenness(G)

# Генерируем attributes
for node in important_nodes:
    attribute = llm.generate_attribute(node, neighbors(node))
    G.add_node(attribute.hash_id, type='attribute', weight=1)
    G.add_edge(node, attribute.hash_id, weight=1)
```

См. [K-Core and Centrality](./k-core-centrality.md)

### 3. Graph Construction → Leiden

После графа (с attributes), детектируем сообщества:

```python
G = build_graph(...)

communities = leiden_algorithm(G)

# Суммируем каждое сообщество
for community in communities:
    hle = llm.summarize_community(community_nodes)
    G.add_node(hle.hash_id, type='high_level_element', weight=1)
    for node in community_nodes:
        G.add_edge(node, hle.hash_id, weight=1)
```

См. [Leiden Algorithm](./leiden-community-detection.md)

### 4. Graph Concatenation → PPR

Объединённый граф используется для PPR:

```python
unified_graph = GraphConcat(knowledge_graph).concat(hnsw_graph)
unified_graph = GraphConcat.unbalance_adjust(unified_graph)

# PPR на unified graph
ppr_results = sparse_PPR(unified_graph).PPR(personalization)
```

См. [Personalized PageRank](./personalized-pagerank.md)

### 5. Graph → HNSW

Embeddings узлов индексируются в HNSW:

```python
for node in G.nodes():
    if node.type in ['semantic_unit', 'entity', 'attribute', 'high_level_element']:
        embedding = embedding_model(node.context)
        hnsw.add_nodes([(node.hash_id, embedding)])
```

См. [HNSW Algorithm](./hnsw-algorithm.md)

---

## Концептуальные связи

### От текста к структуре

```
НЕСТРУКТУРИРОВАННОЕ ЗНАНИЕ (Текст)
               ↓
    [Semantic Segmentation]
               ↓
АТОМАРНОЕ ЗНАНИЕ (Semantic Units)
               ↓
    [Entity & Relation Extraction]
               ↓
ОНТОЛОГИЧЕСКОЕ ЗНАНИЕ (Entities + Relationships)
               ↓
    [Graph Construction]
               ↓
СТРУКТУРНОЕ ЗНАНИЕ (Knowledge Graph)
               ↓
    [HNSW Concatenation]
               ↓
HYBRID ЗНАНИЕ (Graph + Embeddings)
```

### Дуальность аналогового и символического

Knowledge graph + HNSW graph реализуют **dual-process theory**:

| Knowledge Graph | HNSW Graph |
|-----------------|------------|
| **Символическое** | **Аналоговое** |
| Дискретные отношения | Непрерывные расстояния |
| Явные связи | Имплицитная близость |
| Логика | Интуиция |
| System 2 (медленное) | System 1 (быстрое) |

**Объединение**: Hybrid intelligence = логика + интуиция.

### Многоуровневая иерархия

```
АБСТРАКЦИЯ ↑
           │
High-level Elements ← Leiden + LLM
           │
Attributes ← K-Core + Betweenness + LLM
           │
Relationships ← LLM extraction
           │
Entities ← LLM extraction
           │
Semantic Units ← LLM decomposition
           │
Tokens ← Text chunking
           │
КОНКРЕТИКА ↓
```

Graph construction создаёт **все уровни иерархии**.

---

## Ограничения и edge cases

### 1. Hash Collisions

**Проблема**: SHA256 хэши могут (теоретически) коллидировать.

**Вероятность**: ~2^-256 (пренебрежимо мала)

**NodeRAG**: Не проблема на практике.

### 2. Duplicate Edges

**Проблема**: Несколько semantic units могут упоминать одну и ту же пару (entity, relationship).

**Решение**: Увеличиваем вес ребра вместо создания дубликата.

### 3. Disconnected Components

**Проблема**: Если корпус содержит несвязанные темы, граф может быть несвязным.

**NodeRAG решение**: HNSW concatenation соединяет компоненты через embedding similarity.

### 4. Memory Consumption

**Проблема**: Для очень больших графов (>10M узлов), NetworkX может потребовать >100 GB RAM.

**NodeRAG**: Для типичных корпусов (<1M узлов), это не проблема.

**Альтернатива**: Использовать graph databases (Neo4j, JanusGraph) для >10M узлов.

---

## Код examples

### Построение простого графа

```python
import networkx as nx

G = nx.Graph()

# Semantic unit
G.add_node('SU-1', type='semantic_unit', weight=1)

# Entities
G.add_node('ENT-A', type='entity', weight=1)
G.add_node('ENT-B', type='entity', weight=1)

# Relationship
G.add_node('REL-X', type='relationship', weight=1)

# Рёбра
G.add_edge('SU-1', 'ENT-A', weight=1)
G.add_edge('SU-1', 'ENT-B', weight=1)
G.add_edge('SU-1', 'REL-X', weight=1)
G.add_edge('ENT-A', 'REL-X', weight=1)
G.add_edge('ENT-B', 'REL-X', weight=1)

# Статистика
print(f"Nodes: {G.number_of_nodes()}")  # 4
print(f"Edges: {G.number_of_edges()}")  # 5
```

### Graph Concatenation

```python
from NodeRAG.utils import GraphConcat

# Knowledge graph
knowledge_graph = nx.Graph()
knowledge_graph.add_edge('A', 'B', weight=1)
knowledge_graph.add_edge('B', 'C', weight=2)

# HNSW graph
hnsw_graph = nx.Graph()
hnsw_graph.add_edge('A', 'D', weight=1)
hnsw_graph.add_edge('B', 'C', weight=1)  # Дублирует ребро из knowledge graph

# Объединение
unified = GraphConcat(knowledge_graph).concat(hnsw_graph)

print(unified.edges(data=True))
# [('A', 'B', {'weight': 1}),
#  ('A', 'D', {'weight': 1}),
#  ('B', 'C', {'weight': 3})]  # 2 + 1 = 3

# Unbalance adjust
unified = GraphConcat.unbalance_adjust(unified)

print(unified['B']['C']['weight'])  # Урезан до 1/degree(B) или 1/degree(C)
```

### Визуализация графа

```python
import matplotlib.pyplot as plt

# Цвета по типу узла
color_map = {
    'semantic_unit': 'blue',
    'entity': 'red',
    'relationship': 'green',
    'attribute': 'orange',
    'high_level_element': 'purple'
}

colors = [color_map[G.nodes[node]['type']] for node in G.nodes()]

pos = nx.spring_layout(G)
nx.draw_networkx_nodes(G, pos, node_color=colors, node_size=300)
nx.draw_networkx_edges(G, pos, alpha=0.5)
nx.draw_networkx_labels(G, pos, font_size=8)
plt.title('Knowledge Graph')
plt.show()
```

---

## Практические рекомендации

### Для разработчиков

1. **Use hash-based deduplication**: Avoid duplicate nodes and edges.

2. **Batch operations**: Group multiple add_node/add_edge calls for efficiency.

3. **Monitor graph statistics**: Track nodes, edges, avg_degree during construction.

4. **Save intermediate graphs**: Checkpoint after each pipeline stage.

### Для исследователей

1. **Analyze graph structure**: Degree distribution, clustering coefficient, path lengths.

2. **Study node type distribution**: Balance of entities, semantic units, etc.

3. **Compare with/without HNSW concatenation**: PPR quality difference.

4. **Experiment with unbalance_adjust**: Compare PPR with/without adjustment.

---

## Дополнительные ресурсы

### Научные статьи

1. **Knowledge Graph Construction**: Ji, S., et al. (2021). "A Survey on Knowledge Graphs: Representation, Acquisition, and Applications." IEEE Transactions on Neural Networks and Learning Systems.

2. **Graph Embeddings**: Hamilton, W., Ying, Z., & Leskovec, J. (2017). "Inductive Representation Learning on Large Graphs." NeurIPS 2017.

### Документация

- [NetworkX Documentation](https://networkx.org/documentation/stable/)
- [Graph Operations Dependencies](../dependencies/graph-operations.md)
- [Storage Dependencies](../dependencies/data-processing-and-utilities.md)

### Связанные алгоритмы

- [HNSW Algorithm](./hnsw-algorithm.md) — генерирует граф для concatenation
- [Personalized PageRank](./personalized-pagerank.md) — использует unified graph
- [Leiden Algorithm](./leiden-community-detection.md) — работает на knowledge graph
- [K-Core and Centrality](./k-core-centrality.md) — анализирует knowledge graph

---

**Последнее обновление**: 2025-11-13

**См. также**:
- [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)
- [Indexing Transformations](../transform/indexing-transformations.md)
- [Text Decomposition Prompt](../prompts/text-decomposition-prompt.md)
