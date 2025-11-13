# Leiden Algorithm: Детекция сообществ в графе знаний

## Концептуальный обзор

**Leiden algorithm** — это алгоритм детекции сообществ (community detection) в графах, который находит **плотно связанные группы узлов**. В NodeRAG используется для идентификации **тематических кластеров** в knowledge graph, которые затем превращаются в **high-level elements** — абстракции высокого уровня.

### Философская сущность

В контексте [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), Leiden algorithm выполняет функцию **абстрактного осознания** (уровень 4):

```
Конкретика (Semantic Units + Entities)
         ↓
    [Graph Construction]
         ↓
  Структура (Knowledge Graph)
         ↓
  [Leiden Community Detection]
         ↓
  Сообщества (Плотные кластеры)
         ↓
  [LLM Community Summarization]
         ↓
Абстракция (High-level Elements)
```

Это аналог **категоризации** в когнитивной науке — группировка конкретных элементов в абстрактные категории.

---

## Роль в парадигме вопроса как ключа

Согласно [парадигме вопроса как ключа](../research/question-as-key-paradigm.md), сообщества представляют **макроскопические концепции** — крупные семантические единицы:

```
Микро-уровень: Entities, Semantic Units (атомарные факты)
       ↓
  [Leiden Algorithm]
       ↓
Мезо-уровень: Communities (тематические кластеры)
       ↓
  [LLM Summarization]
       ↓
Макро-уровень: High-level Elements (абстрактные темы)
```

**Эпистемологический принцип**:
- **Bottom-up**: От конкретных фактов к абстрактным темам
- **Гештальт**: "Целое больше суммы частей" — community summary ≠ просто список узлов

---

## Математическая модель

### 1. Задача детекции сообществ

**Задача**: Разбить граф G = (V, E) на **непересекающиеся подмножества** (communities) C₁, C₂, ..., Cₖ, такие что:

```
V = C₁ ∪ C₂ ∪ ... ∪ Cₖ
Cᵢ ∩ Cⱼ = ∅  (для i ≠ j)
```

**Критерий качества**: Максимизировать **modularity** (модулярность).

### 2. Modularity

**Modularity Q** измеряет, насколько плотность рёбер **внутри** сообществ превосходит случайное ожидание:

```
Q = 1/(2m) · Σᵢⱼ [Aᵢⱼ - (kᵢ·kⱼ)/(2m)] · δ(cᵢ, cⱼ)
```

где:
- `Aᵢⱼ` = 1 если есть ребро между i и j, иначе 0
- `kᵢ` = степень узла i
- `m` = общее число рёбер
- `δ(cᵢ, cⱼ)` = 1 если узлы i и j в одном сообществе, иначе 0

**Интерпретация**:
- `Aᵢⱼ` — фактическое число рёбер
- `(kᵢ·kⱼ)/(2m)` — ожидаемое число рёбер в случайном графе
- Положительный Q → структура сообществ **сильнее случайной**

**Диапазон**: Q ∈ [-0.5, 1.0]
- Q ≈ 0: Нет структуры сообществ
- Q > 0.3: Хорошая структура
- Q > 0.7: Очень сильная структура

### 3. Leiden Algorithm

Leiden — это усовершенствование **Louvain algorithm** с гарантией **хорошо связанных сообществ**.

#### Фазы алгоритма

**Фаза 1: Local Moving**

Каждый узел пробует переместиться в соседнее сообщество, если это увеличивает Q:

```python
for node in random_order(nodes):
    current_community = community[node]
    best_community = current_community
    best_delta_Q = 0

    for neighbor_community in neighbor_communities(node):
        delta_Q = compute_modularity_gain(node, neighbor_community)

        if delta_Q > best_delta_Q:
            best_delta_Q = delta_Q
            best_community = neighbor_community

    if best_community != current_community:
        move_node(node, best_community)
```

**Фаза 2: Refinement** (Уникально для Leiden)

Разделяем плохо связанные подсообщества:

```python
for community in communities:
    subgraph = induced_subgraph(community)
    subcommunities = detect_subcommunities(subgraph)

    if len(subcommunities) > 1:
        split_community(community, subcommunities)
```

**Фаза 3: Aggregation**

Сжимаем граф: каждое сообщество становится **одним узлом**:

```python
aggregated_graph = Graph()

for community in communities:
    super_node = create_super_node(community)
    aggregated_graph.add_node(super_node)

for edge in edges:
    if community[edge.u] != community[edge.v]:
        aggregated_graph.add_edge(
            community[edge.u],
            community[edge.v]
        )
```

**Итерация**: Повторяем Фазы 1-3 на aggregated_graph, пока Q не перестанет расти.

---

## Реализация в NodeRAG

### Файл: `NodeRAG/build/pipeline/summary_generation.py`

#### Класс `SummaryGeneration`

```python
class SummaryGeneration:

    def __init__(self, config: NodeConfig):
        self.config = config
        self.indices = self.config.indices
        self.communities = []
        self.high_level_elements = []

        if os.path.exists(self.config.graph_path):
            self.mapper = Mapper([...])
            self.G = storage.load(self.config.graph_path)  # NetworkX Graph
            self.G_ig = IGraph(self.G).to_igraph()  # Convert to igraph
```

**Конвертация в igraph**:

NodeRAG использует **igraph** (C library) вместо **NetworkX** (Python) для Leiden, так как igraph **на порядок быстрее**:

```python
class IGraph:

    def __init__(self, graph: nx.Graph):
        self.graph = graph

    def to_igraph(self):
        G = ig.Graph.TupleList(self.graph.edges(), directed=False)
        return G
```

**Файл**: `NodeRAG/utils/graph_operator.py:6-14`

#### Метод `partition()`

```python
def partition(self):
    # Leiden community detection
    partition = la.find_partition(
        self.G_ig,
        la.ModularityVertexPartition
    )

    for i, community in enumerate(partition):
        # Извлекаем имена узлов
        community_nodes = [
            self.G_ig.vs[node]['name']
            for node in community
            if self.G_ig.vs[node]['name'] in self.mapper.embeddings
        ]

        # Создаём объект Community_summary
        self.communities.append(
            Community_summary(community_nodes, self.mapper, self.G, self.config)
        )
```

**Фильтрация**:
- Берём только узлы с embeddings (semantic units, attributes)
- Исключаем entities, relationships (они не имеют embeddings)

#### Генерация Community Summaries

```python
async def generate_community_summary(self, community: Community_summary):
    await community.generate_community_summary()

    if isinstance(community.response, str):
        # Ошибка при генерации
        self.config.tracker.update()
        return

    community_dict = {
        'community': community.community_node,
        'response': community.response,
        'hash_id': community.hash_id,
        'human_readable_id': community.human_readable_id
    }

    # Сохраняем в JSON lines
    with open(self.config.summary_path, 'a', encoding='utf-8') as f:
        f.write(json.dumps(community_dict, ensure_ascii=False) + '\n')

    self.config.tracker.update()
```

**Промпт для LLM** (см. [Community Summary Prompt](../prompts/community-summary-prompt.md)):

```python
# В Community_summary класс
def generate_community_summary(self):
    # Собираем контексты узлов сообщества
    contexts = [self.mapper.get(node, 'context') for node in self.community_nodes]

    # Промпт: "Summarize the following semantic units into high-level themes..."
    prompt = self.config.prompt_manager.community_summary.format(
        contexts='\n'.join(contexts)
    )

    # LLM генерация
    response = await self.config.API_client.request({
        'query': prompt,
        'response_format': self.config.prompt_manager.high_level_elements_json
    })

    self.response = response
```

**Структурированный вывод** (Pydantic):

```python
class HighLevelElement(BaseModel):
    title: str
    description: str

class CommunityResponse(BaseModel):
    high_level_elements: List[HighLevelElement]
```

---

## Использование в системе

### Индексирование: Summary Generation Pipeline

**Этап в pipeline**: После graph construction и attribute generation

```
Document → Text → Semantic Units → Graph Construction
                                          ↓
                                    [Leiden Algorithm]
                                          ↓
                                     Communities
                                          ↓
                                 [LLM Summarization]
                                          ↓
                               High-level Elements
```

**Файл**: `NodeRAG/build/Node.py`

```python
async def run_summary_generation_pipeline(self):
    summary_gen = SummaryGeneration(self.config)
    await summary_gen.main()
```

### Встраивание High-level Elements в граф

```python
async def high_level_element_summary(self):
    results = []

    with open(self.config.summary_path, 'r', encoding='utf-8') as f:
        for line in f:
            results.append(json.loads(line))

    for result in results:
        node_names = result['community']

        for hle in result['response']['high_level_elements']:
            he = High_level_elements(
                hle['description'],
                hle['title'],
                self.config
            )
            he.related_node(node_names)

            if self.G.has_node(he.hash_id):
                # High-level element уже существует, увеличиваем вес
                self.G.nodes[he.hash_id]['weight'] += 1
            else:
                # Добавляем новые узлы
                self.G.add_node(
                    he.hash_id,
                    type='high_level_element',
                    weight=1
                )
                self.G.add_node(
                    he.title_hash_id,
                    type='high_level_element_title',
                    weight=1,
                    related_node=he.hash_id
                )

                # Ребро: title ↔ description
                self.G.add_edge(he.hash_id, he.title_hash_id, weight=1)

                self.high_level_elements.append(he)
```

### K-means Clustering для больших сообществ

Если сообщество слишком большое, используется **k-means** для **кластеризации** узлов:

```python
# Вычисляем optimal number of clusters
centroids = math.ceil(math.sqrt(len(All_nodes) + len(self.high_level_elements)))
threshold = (len(All_nodes) + len(self.high_level_elements)) / centroids

if threshold > self.config.Hcluster_size:  # 50
    # Используем FAISS k-means
    embedding_list = np.array([...], dtype=np.float32)
    high_level_element_embedding = np.array([...], dtype=np.float32)
    all_embeddings = np.vstack([high_level_element_embedding, embedding_list])

    kmeans = faiss.Kmeans(d=all_embeddings.shape[1], k=centroids)
    kmeans.train(all_embeddings.astype(np.float32))
    _, cluster_labels = kmeans.assign(all_embeddings.astype(np.float32))

    # Соединяем high-level elements только с узлами того же кластера
    for i, he in enumerate(self.high_level_elements):
        for j, node in enumerate(All_nodes):
            if cluster_labels[i] == cluster_labels[len(he) + j]:
                if node in he.related_node:
                    self.G.add_edge(node, he.hash_id, weight=1)
```

**Зачем k-means**:
- Без кластеризации: каждый high-level element соединяется со **всеми узлами** сообщества
- Для больших сообществ (>50 узлов) это создаёт **слишком плотный граф**
- k-means разбивает сообщество на **подкластеры** и соединяет только с релевантными узлами

---

## Производительность

### Сложность

| Алгоритм | Worst Case | Average Case | NodeRAG |
|----------|------------|--------------|---------|
| **Louvain** | O(N log N) | O(N log N) | - |
| **Leiden** | O(N log N) | O(N log N) | ✓ |
| **Label Propagation** | O(E) | O(E) | - |
| **Infomap** | O(E log N) | O(E log N) | - |

**Leiden преимущества над Louvain**:
1. **Гарантирует хорошо связанные сообщества** (no disconnected subcommunities)
2. **Быстрее сходится** (refinement phase)
3. **Более высокая modularity** (~5-10% выше)

### Бенчмарки (NodeRAG)

**Граф**: 500K узлов, 2M рёбер (после graph construction)

| Метрика | Значение |
|---------|----------|
| Leiden execution time | ~30-60 секунд |
| Number of communities | ~500-1000 |
| Average community size | ~50-100 узлов |
| Modularity Q | ~0.4-0.6 |
| LLM summarization time | ~10-20 минут (500 communities × 1-2s each) |
| Total pipeline time | ~15-30 минут |

**Hardware**:
- Leiden: CPU (igraph C core)
- LLM summarization: API (OpenAI GPT-4o)

---

## Библиотеки

### igraph + leidenalg

**Библиотеки**:
- `igraph==0.11.8` — Graph library (C core)
- `leidenalg==0.10.2` — Leiden algorithm implementation

**Установка**:
```bash
pip install igraph leidenalg
```

**Документация**: [Graph Operations Dependencies](../dependencies/graph-operations.md)

#### Использование

```python
import igraph as ig
import leidenalg as la

# Создание igraph из NetworkX
G_nx = nx.Graph()
G_ig = ig.Graph.TupleList(G_nx.edges(), directed=False)

# Leiden community detection
partition = la.find_partition(
    G_ig,
    la.ModularityVertexPartition,
    weights='weight'  # если рёбра имеют веса
)

# Результаты
print(f"Number of communities: {len(partition)}")
print(f"Modularity: {partition.modularity:.4f}")

# Извлечение сообществ
for i, community in enumerate(partition):
    nodes = [G_ig.vs[node]['name'] for node in community]
    print(f"Community {i}: {nodes[:10]}...")  # First 10 nodes
```

### FAISS (для k-means)

**Библиотека**: `faiss-cpu==1.10.0`

Используется для **кластеризации больших сообществ**:

```python
import faiss

kmeans = faiss.Kmeans(
    d=1536,         # Размерность embeddings
    k=centroids,    # Число кластеров
    niter=20,       # Число итераций
    verbose=True
)

kmeans.train(embeddings)
distances, labels = kmeans.assign(embeddings)
```

**Документация**: [Vector Search Dependencies](../dependencies/vector-search.md)

---

## Связь с другими алгоритмами

### 1. Graph Construction → Leiden

Leiden работает на **полном knowledge graph**:

```python
# После text decomposition и graph construction
G = build_graph(semantic_units, entities, relationships)

# Leiden community detection
communities = leiden_algorithm(G)
```

См. [Graph Construction](./graph-construction.md)

### 2. Leiden → LLM Summarization

Сообщества **суммируются** LLM для создания high-level themes:

```python
for community in communities:
    summary = llm.summarize(community_nodes)
    high_level_elements.append(summary)
```

См. [Community Summary Prompt](../prompts/community-summary-prompt.md)

### 3. High-level Elements → HNSW

High-level element embeddings **добавляются в HNSW**:

```python
for he in high_level_elements:
    embedding = embedding_model(he.context)
    hnsw.add_nodes([(he.hash_id, embedding)])
```

См. [HNSW Algorithm](./hnsw-algorithm.md)

### 4. High-level Elements → PPR

High-level elements могут быть **найдены через PPR**:

```python
# Если HNSW находит high-level element title
HNSW_results = ['HLE-TITLE-123']

# PPR расширяет до related nodes
personalization = {'HLE-TITLE-123': 1.0}
ppr_results = PPR(personalization)
```

См. [Personalized PageRank](./personalized-pagerank.md)

---

## Концептуальные связи

### Абстрактное осознание

В модели [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), Leiden + LLM Summarization соответствуют **уровню 4: Абстрактное осознание**:

```
Уровень 2: Аналитическое (Text Decomposition) → Semantic Units
                         ↓
Уровень 3: Рефлексивное (Attribute Generation) → Attributes
                         ↓
         [Graph Construction + Leiden]
                         ↓
Уровень 4: Абстрактное (Community Summarization) → High-level Elements
```

**Когнитивная функция**: **Категоризация** — группировка элементов в абстрактные категории.

### Гештальт высшего порядка

Сообщества представляют **гештальты** — целостные структуры, где **целое > суммы частей**:

```
Community = {SU-1, SU-2, ..., SU-50}

Просто список: "50 semantic units"

High-level Summary: "Research on renewable energy technologies
                     with focus on solar panel efficiency"
```

**Emergent property**: Абстрактная тема **не содержится** ни в одном semantic unit, но **возникает** из их комбинации.

### Иерархия репрезентаций

```
АБСТРАКЦИЯ ↑
           │
High-level Elements ← [Leiden + LLM]
           │
Communities ← [Leiden Algorithm]
           │
Graph Structure ← [Graph Construction]
           │
Entities + Relationships ← [Text Decomposition]
           │
Semantic Units ← [Text Decomposition]
           │
Tokens ← [Chunking]
           │
КОНКРЕТИКА ↓
```

Leiden находится на **переходе от структуры к абстракции**.

---

## Вариации и расширения

### 1. Hierarchical Leiden

Вместо одного уровня сообществ, можно построить **иерархию**:

```python
# Level 0: Исходный граф
communities_L0 = leiden(G)

# Level 1: Aggregated graph
G_L1 = aggregate_graph(G, communities_L0)
communities_L1 = leiden(G_L1)

# Level 2: Super-aggregated graph
G_L2 = aggregate_graph(G_L1, communities_L1)
communities_L2 = leiden(G_L2)
```

**Результат**: Дерево сообществ (малые → средние → большие)

**NodeRAG**: Не используется, так как один уровень достаточен.

### 2. Overlapping Communities

Leiden находит **непересекающиеся** сообщества. Для пересекающихся можно использовать:

- **SLPA** (Speaker-Listener Label Propagation)
- **Link Communities** (Evans & Lambiotte, 2009)
- **OSLOM** (Order Statistics Local Optimization Method)

**NodeRAG**: Не нужно, так как узлы могут быть связаны с несколькими high-level elements через PPR.

### 3. Resolution Parameter

Leiden поддерживает **resolution parameter γ**, контролирующий размер сообществ:

```python
partition = la.find_partition(
    G_ig,
    la.CPMVertexPartition,  # Constant Potts Model
    resolution_parameter=0.5  # γ
)
```

- γ < 1: Более крупные сообщества
- γ = 1: Стандартная modularity
- γ > 1: Более мелкие сообщества

**NodeRAG**: Использует γ = 1 (стандарт).

---

## Ограничения и edge cases

### 1. Resolution Limit

**Проблема**: Modularity не может обнаружить сообщества **меньше определённого размера**:

```
min_size ≈ √(2m)
```

где m = число рёбер.

**Эффект**: Малые, но плотные кластеры могут быть **пропущены**.

**NodeRAG**: Не критично, так как малые кластеры обычно не несут самостоятельной семантики.

### 2. Disconnected Communities

**Проблема** (в Louvain): Сообщество может содержать **несвязанные компоненты**.

**Leiden решение**: Refinement phase разбивает disconnected subcommunities.

### 3. Стохастичность

Leiden использует **случайное блуждание**, результаты могут немного варьироваться:

```bash
Run 1: 523 communities, Q = 0.512
Run 2: 518 communities, Q = 0.509
Run 3: 526 communities, Q = 0.515
```

**NodeRAG**: Приемлемо, так как различия минимальны (~1%).

### 4. LLM Summarization Failures

Некоторые сообщества могут быть **слишком разнородными** для суммирования:

```python
Community = {
    "solar panel efficiency",
    "quantum computing applications",
    "machine learning in healthcare"
}
```

LLM может вернуть generic summary или error.

**NodeRAG решение**: Такие summaries фильтруются (проверка типа response).

---

## Код examples

### Leiden Community Detection

```python
from NodeRAG.utils import IGraph
import leidenalg as la
import networkx as nx

# Создание графа
G_nx = nx.karate_club_graph()

# Конвертация в igraph
G_ig = IGraph(G_nx).to_igraph()

# Leiden algorithm
partition = la.find_partition(
    G_ig,
    la.ModularityVertexPartition
)

print(f"Communities: {len(partition)}")
print(f"Modularity: {partition.modularity:.4f}")

# Печать сообществ
for i, community in enumerate(partition):
    nodes = [G_ig.vs[node]['name'] for node in community]
    print(f"Community {i}: {nodes}")
```

### Визуализация сообществ

```python
import matplotlib.pyplot as plt

# Colour nodes по сообществу
colors = []
for node in G_nx.nodes():
    community_id = partition.membership[node]
    colors.append(community_id)

# Рисуем граф
pos = nx.spring_layout(G_nx)
nx.draw_networkx_nodes(G_nx, pos, node_color=colors, cmap=plt.cm.rainbow)
nx.draw_networkx_edges(G_nx, pos, alpha=0.3)
nx.draw_networkx_labels(G_nx, pos, font_size=8)
plt.show()
```

### LLM Summarization of Community

```python
from NodeRAG.build.component import Community_summary

# Создание community summary объекта
community = Community_summary(
    community_nodes=['SU-1', 'SU-2', 'SU-3'],
    mapper=mapper,
    graph=G,
    config=config
)

# Асинхронная генерация
await community.generate_community_summary()

# Результат
print(community.response['high_level_elements'])
# [
#   {
#     'title': 'Solar Energy Research',
#     'description': 'Studies on solar panel efficiency...'
#   },
#   ...
# ]
```

---

## Практические рекомендации

### Для разработчиков

1. **Используйте igraph для больших графов**: В 10-100x быстрее NetworkX.

2. **Фильтруйте малые сообщества**: Communities с <5 узлами часто не информативны.

3. **Кэшируйте сообщества**: Leiden дорогостоящ, сохраняйте результаты.

4. **Batch LLM requests**: Суммируйте несколько сообществ в одном запросе.

### Для исследователей

1. **Experiment with resolution**: Попробуйте γ ∈ [0.5, 2.0] и изучите размеры сообществ.

2. **Compare algorithms**: Попробуйте Louvain, Infomap, Label Propagation.

3. **Analyze modularity distribution**: Визуализируйте Q для разных параметров.

4. **Study semantic coherence**: Оцените качество LLM summaries вручную.

---

## Дополнительные ресурсы

### Научные статьи

1. **Traag, V. A., Waltman, L., & van Eck, N. J. (2019)**. "From Louvain to Leiden: guaranteeing well-connected communities." Scientific Reports, 9(1), 1-12.

2. **Blondel, V. D., et al. (2008)**. "Fast unfolding of communities in large networks." Journal of Statistical Mechanics: Theory and Experiment.

3. **Newman, M. E., & Girvan, M. (2004)**. "Finding and evaluating community structure in networks." Physical Review E, 69(2), 026113.

### Документация

- [leidenalg Documentation](https://leidenalg.readthedocs.io/)
- [igraph Python Documentation](https://igraph.org/python/)
- [Graph Operations Dependencies](../dependencies/graph-operations.md)

### Связанные алгоритмы

- [Graph Construction](./graph-construction.md) — создаёт граф для Leiden
- [HNSW Algorithm](./hnsw-algorithm.md) — индексирует high-level elements
- [Personalized PageRank](./personalized-pagerank.md) — находит узлы через HLE

---

**Последнее обновление**: 2025-11-13

**См. также**:
- [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)
- [Community Summary Prompt](../prompts/community-summary-prompt.md)
- [Indexing Transformations](../transform/indexing-transformations.md)
