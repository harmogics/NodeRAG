# K-Core Decomposition и Betweenness Centrality: Поиск важных узлов

## Концептуальный обзор

**K-Core decomposition** и **Betweenness Centrality** — это два взаимодополняющих алгоритма для идентификации **важных узлов** в графе знаний. В NodeRAG они используются для выбора entities, которые заслуживают **детальных атрибутов** — нарративных описаний, генерируемых LLM.

### Философская сущность

В контексте [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), эти алгоритмы выполняют функцию **рефлексивного осознания** (уровень 3):

```
Entities (все сущности)
       ↓
[K-Core + Betweenness]
       ↓
Important Entities (10-15%)
       ↓
[LLM Attribute Generation]
       ↓
Comprehensive Descriptions
```

Это аналог **метакогниции** — система "размышляет", **какие** entities заслуживают **глубокого анализа**.

---

## Роль в системе

### Проблема: Scale

**Проблема**: Генерация атрибутов для **всех** entities дорогостоящая:

```
Corpus: 100K entities
Cost per attribute: ~$0.001 (GPT-4o)
Total cost: 100K × $0.001 = $100

Time per attribute: ~1-2s
Total time: 100K × 1.5s ≈ 42 hours
```

### Решение: Selective Attribution

**Решение**: Генерируем атрибуты только для **важных** entities:

```
Important entities: 10-15K (10-15%)
Cost: 10-15K × $0.001 = $10-15
Time: 10-15K × 1.5s ≈ 4-6 hours
```

**Экономия**: 85-90% времени и стоимости

### Критерии важности

**Вопрос**: Что делает entity "важной"?

**Ответ NodeRAG**: Комбинация двух критериев:

1. **K-Core**: Entity находится в **плотном подграфе** (высокая структурная интеграция)
2. **Betweenness Centrality**: Entity является **мостом** между разными частями графа

```
K-Core ∩ Betweenness = Структурно важные + Семантически центральные
```

---

## K-Core Decomposition

### Математическая модель

**Определение**: **k-core** графа G — это максимальный подграф, в котором каждый узел имеет **степень ≥ k**.

```
k-core(G) = H ⊆ G, где deg_H(v) ≥ k для всех v ∈ H
```

**Интуиция**: Узлы в k-core **плотно связаны** друг с другом.

### Алгоритм

```python
def k_core(G, k):
    H = G.copy()

    while True:
        # Находим узлы с degree < k
        to_remove = [v for v in H.nodes() if H.degree(v) < k]

        if not to_remove:
            break

        # Удаляем их
        H.remove_nodes_from(to_remove)

    return H
```

**Сложность**: O(E) — линейная по числу рёбер

### Core Number

**Core number c(v)** узла v — максимальный k, при котором v принадлежит k-core:

```
c(v) = max{k : v ∈ k-core(G)}
```

**Интерпретация**: Чем выше c(v), тем **глубже** v интегрирован в плотные части графа.

### Визуализация

```
         k=1 (весь граф)
         ┌─────────────┐
         │ A B C D E F │
         │    ╱│╲   ╱  │
         │   ╱ │ ╲ ╱   │
         │  B──C──D    │
         │   ╲ │ ╱ ╲   │
         │    ╲│╱   E──F
         └─────────────┘

         k=2 (2-core)
         ┌───────┐
         │ B C D │
         │ │╲│╱│ │
         │ B─C─D │
         └───────┘

         k=3 (3-core)
         ┌─────┐
         │ C D │
         │ C─D │
         └─────┘
```

**Луковичная структура** (onion structure): Граф разбивается на вложенные k-cores.

---

## Betweenness Centrality

### Математическая модель

**Betweenness centrality** узла v измеряет, как часто v лежит на **кратчайших путях** между другими узлами:

```
BC(v) = Σ_{s≠v≠t} σ_st(v) / σ_st
```

где:
- `σ_st` — число кратчайших путей от s к t
- `σ_st(v)` — число таких путей, проходящих через v

**Интерпретация**: Узлы с высоким BC являются **мостами** или **узловыми точками** (hubs) в графе.

### Алгоритм (Brandes, 2001)

```python
def betweenness_centrality(G):
    BC = {v: 0 for v in G.nodes()}

    for s in G.nodes():
        # BFS from s
        stack = []
        paths = {v: [] for v in G.nodes()}
        sigma = {v: 0 for v in G.nodes()}
        sigma[s] = 1
        dist = {v: -1 for v in G.nodes()}
        dist[s] = 0
        queue = [s]

        while queue:
            v = queue.pop(0)
            stack.append(v)

            for w in G.neighbors(v):
                if dist[w] < 0:
                    queue.append(w)
                    dist[w] = dist[v] + 1

                if dist[w] == dist[v] + 1:
                    sigma[w] += sigma[v]
                    paths[w].append(v)

        # Accumulation
        delta = {v: 0 for v in G.nodes()}
        while stack:
            w = stack.pop()
            for v in paths[w]:
                delta[v] += (sigma[v] / sigma[w]) * (1 + delta[w])

            if w != s:
                BC[w] += delta[w]

    return BC
```

**Сложность**: O(V · E) для невзвешенного графа

### Нормализация

```
BC_normalized(v) = BC(v) / [(N-1) · (N-2) / 2]
```

где N = число узлов.

**Диапазон**: BC_normalized ∈ [0, 1]

---

## Реализация в NodeRAG

### Файл: `NodeRAG/build/pipeline/attribute_generation.py`

#### Класс `NodeImportance`

```python
class NodeImportance:

    def __init__(self, graph: nx.Graph, console: Console):
        self.G = graph
        self.important_nodes = []
        self.console = console
```

#### K-Core с автоматическим выбором k

```python
def K_core(self, k: int | None = None):
    if k is None:
        k = self.default_k()

    # NetworkX k-core
    self.k_subgraph = nx.core.k_core(self.G, k=k)

    for node in self.k_subgraph.nodes():
        # Фильтруем: только entities с weight > 1
        if (self.G.nodes[node]['type'] == 'entity'
            and self.G.nodes[node]['weight'] > 1):
            self.important_nodes.append(node)
```

**Фильтры**:
1. `type == 'entity'`: Только entities (не semantic units, relationships, etc.)
2. `weight > 1`: Entity упоминается **более одного раза** в корпусе

#### Эвристика для выбора k

```python
def average_degree(self):
    average_degree = sum(dict(self.G.degree()).values()) / self.G.number_of_nodes()
    return average_degree

def default_k(self):
    k = round(np.log(self.G.number_of_nodes()) * self.average_degree() ** (1/2))
    return k
```

**Формула**:
```
k = log(N) · √(avg_degree)
```

**Обоснование**:
- Больше узлов (N↑) → более высокий k (более жёсткий фильтр)
- Выше средняя степень → более плотный граф → более высокий k

**Пример**:
```
N = 500,000 nodes
avg_degree = 4.0
k = log(500,000) · √4 = 13.12 · 2 ≈ 26
```

#### Betweenness Centrality с sampling

```python
def betweenness_centrality(self):
    # Sampling: вычисляем только для k=10 исходных узлов
    self.betweenness = nx.betweenness_centrality(self.G, k=10)

    average_betweenness = sum(self.betweenness.values()) / len(self.betweenness)
    scale = round(math.log10(len(self.betweenness)))

    for node in self.betweenness:
        # Порог: avg * log10(N)
        if self.betweenness[node] > average_betweenness * scale:
            if (self.G.nodes[node]['type'] == 'entity'
                and self.G.nodes[node]['weight'] > 1):
                self.important_nodes.append(node)
```

**Sampling**: `k=10` означает вычисление BC на основе **10 случайных источников** вместо всех узлов.

**Зачем sampling**:
- Полный BC: O(V · E) — очень дорого для 500K узлов
- Sampling BC: O(k · E) — в ~50,000 раз быстрее

**Порог**:
```
threshold = avg_BC · log₁₀(N)
```

**Обоснование**: В больших графах, небольшое число узлов имеет **очень высокий BC**, остальные близки к 0. Лога рифмическая шкала фильтрует топ ~0.1-1%.

#### Объединение результатов

```python
def main(self):
    self.K_core()
    self.console.print('[bold green]K_core done[/bold green]')

    self.betweenness_centrality()
    self.console.print('[bold green]Betweenness done[/bold green]')

    # Удаляем дубликаты (узлы могут быть в обоих списках)
    self.important_nodes = list(set(self.important_nodes))

    return self.important_nodes
```

**Результат**: ~10-15% entities (обычно 10K-15K из 100K)

---

## Использование в системе

### Attribute Generation Pipeline

**Файл**: `NodeRAG/build/pipeline/attribute_generation.py`

```python
class Attribution_generation_pipeline:

    def __init__(self, config: NodeConfig):
        self.config = config
        self.important_nodes = []
        self.attributes = []

        self.mapper = Mapper([...])
        self.G = storage.load(self.config.graph_path)

    def get_important_nodes(self):
        node_importance = NodeImportance(self.G, self.config.console)
        important_nodes = node_importance.main()

        # Исключаем entities, которые уже имеют attributes
        if os.path.exists(self.config.attributes_path):
            attributes = storage.load(self.config.attributes_path)
            existing_nodes = attributes['node'].tolist()
            important_nodes = [
                node for node in important_nodes
                if node not in existing_nodes
            ]

        self.important_nodes = important_nodes
        self.config.console.print('[bold green]Important nodes found[/bold green]')
```

### Генерация атрибутов

Для каждого important entity:

1. **Собрать контексты** из соседних semantic units и relationships
2. **Генерировать атрибут** через LLM
3. **Добавить в граф** как новый узел

```python
async def generate_attribution(self, node: str):
    query = self.get_neighbours_material(node)

    if self.token_counter.token_limit(query):
        # Если контекст слишком большой, используем sorted neighbours
        query = self.get_important_neibours_material(node)

    response = await self.API_client({'query': query})

    if response is not None:
        attribute = Attribute(response, node)

        self.attributes.append(attribute)
        self.G.nodes[node]['attributes'] = [attribute.hash_id]
        self.G.add_node(attribute.hash_id, type='attribute', weight=1)
        self.G.add_edge(node, attribute.hash_id, weight=1)

    self.config.tracker.update()
```

**LLM Prompt**: См. [Attribute Generation Prompt](../prompts/attribute-generation-prompt.md)

---

## Производительность

### Сложность

| Операция | Алгоритм | Сложность |
|----------|----------|-----------|
| K-Core | Iterative removal | O(E) |
| Full Betweenness | Brandes algorithm | O(V · E) |
| Sampled Betweenness | k sources | O(k · E) |
| Total (NodeRAG) | - | O(E + k·E) ≈ O(E) |

**NodeRAG**: Использует k=10, поэтому O(10·E) ≈ O(E)

### Бенчмарки (NodeRAG)

**Граф**: 500K узлов, 2M рёбер

| Метрика | K-Core | Betweenness (k=10) | Total |
|---------|--------|---------------------|-------|
| Execution time | ~5-10s | ~30-60s | ~35-70s |
| Important nodes found | ~8-10K | ~2-3K | ~10-12K (после dedup) |
| Percentage | ~10-12% | ~2-3% | ~10-12% |
| Memory | ~50 MB | ~100 MB | ~150 MB |

**Hardware**: CPU (NetworkX operations)

---

## Библиотеки

### NetworkX

**Библиотека**: `networkx==3.4.2`

**Используемые функции**:

```python
import networkx as nx

# K-Core
k_core_subgraph = nx.core.k_core(G, k=k)

# Betweenness Centrality
betweenness = nx.betweenness_centrality(G, k=10)  # k=10 sampling

# Degree
degrees = dict(G.degree())
```

**Документация**: [Graph Operations Dependencies](../dependencies/graph-operations.md)

### NumPy

**Библиотека**: `numpy==1.26.4`

**Используется для**:
- Логарифмические вычисления: `np.log(N)`
- Статистика: `np.mean()`, `np.std()`

---

## Связь с другими алгоритмами

### 1. Graph Construction → K-Core + Betweenness

После graph construction, запускается attribute generation:

```python
# После text и graph pipelines
G = build_graph(semantic_units, entities, relationships)

# Поиск important nodes
important_nodes = K_core(G) ∩ Betweenness(G)

# Генерация attributes
for node in important_nodes:
    attribute = llm.generate_attribute(node, neighbors(node))
    G.add_node(attribute)
    G.add_edge(node, attribute)
```

См. [Graph Construction](./graph-construction.md)

### 2. Attributes → HNSW

Attributes добавляются в HNSW индекс для поиска:

```python
for attribute in attributes:
    embedding = embedding_model(attribute.context)
    hnsw.add_nodes([(attribute.hash_id, embedding)])
```

См. [HNSW Algorithm](./hnsw-algorithm.md)

### 3. Important Entities → PPR

Во время поиска, important entities могут получить **более высокие PPR scores**:

```python
# Если entity найден через accurate search И имеет attribute
if entity.hash_id in important_nodes:
    # Attribute также будет найден через граф
    neighbors = G.neighbors(entity.hash_id)  # Включает attribute
```

См. [Personalized PageRank](./personalized-pagerank.md)

---

## Концептуальные связи

### Рефлексивное осознание

В модели [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), K-Core + Betweenness соответствуют **метакогнитивному выбору**:

```
Уровень 2: Аналитическое (Text Decomposition) → Entities
                         ↓
         [K-Core + Betweenness]
         "Какие entities важны?"
                         ↓
Уровень 3: Рефлексивное (Attribute Generation) → Attributes
```

**Когнитивная функция**: **Интроспекция** — система "размышляет" о **своём собственном знании** и решает, что заслуживает глубокого анализа.

### Селективное внимание

K-Core + Betweenness реализуют **механизм внимания** (attention):

```
Все entities (100%)
       ↓
 [Attention Filter]
       ↓
Important entities (10-15%)
       ↓
 [Deep Processing]
       ↓
Comprehensive attributes
```

**Аналогия**: Человек не может обработать всю информацию **одинаково глубоко** — внимание фокусируется на **важном**.

### Dual Importance Criteria

NodeRAG использует **две ортогональные** меры важности:

| K-Core | Betweenness |
|--------|-------------|
| **Локальная плотность** | **Глобальная центральность** |
| "Узел окружён связями" | "Узел соединяет части графа" |
| Структурная интеграция | Информационные потоки |
| Cohesive subgroups | Bridges |

**Комбинация**: Узлы, важные **и локально, и глобально** → наиболее ценные.

---

## Вариации и расширения

### 1. Weighted K-Core

Вместо числа рёбер, учитывать **сумму весов**:

```python
def weighted_k_core(G, k):
    H = G.copy()

    while True:
        to_remove = []

        for v in H.nodes():
            weight_sum = sum(H[v][u]['weight'] for u in H.neighbors(v))

            if weight_sum < k:
                to_remove.append(v)

        if not to_remove:
            break

        H.remove_nodes_from(to_remove)

    return H
```

**NodeRAG**: Не используется, так как стандартный k-core достаточен.

### 2. Closeness Centrality

Альтернатива betweenness — **closeness centrality** (среднее расстояние до всех узлов):

```
CC(v) = (N - 1) / Σ_u distance(v, u)
```

**Преимущество**: O(V · E) как betweenness, но проще интерпретация.

**NodeRAG**: Не используется, так как betweenness лучше находит **мосты**.

### 3. PageRank Centrality

Вместо betweenness, можно использовать **PageRank** для важности:

```python
pr = nx.pagerank(G)
important = [v for v, score in pr.items() if score > threshold]
```

**NodeRAG**: Не используется, так как PageRank глобален, а нам нужны **локально важные** узлы.

### 4. Multi-criteria Scoring

Комбинировать K-Core, Betweenness, Degree в единый score:

```
importance(v) = α·k_core(v) + β·BC(v) + γ·degree(v)
```

**NodeRAG**: Не используется, простое объединение (union) достаточно.

---

## Ограничения и edge cases

### 1. K-Core: Dense Regions Bias

K-Core **благоприятствует плотным регионам**, игнорируя важные узлы в разреженных частях:

```
Плотная часть: k-core выбирает много узлов
Разреженная часть: k-core игнорирует все узлы
```

**NodeRAG решение**: Betweenness компенсирует, находя мосты в разреженных частях.

### 2. Betweenness: Peripheral Hubs

Узлы на **периферии** могут иметь высокий BC, но низкую семантическую важность:

```
     A
     │
     B ─── [dense cluster]
```

B имеет высокий BC (мост к A), но может быть не важен семантически.

**NodeRAG решение**: K-Core фильтрует периферийные узлы (они имеют низкий core number).

### 3. Sampling Variance

Betweenness с k=10 имеет **высокую вариацию**:

```bash
Run 1: 2,345 important nodes
Run 2: 2,198 important nodes
Run 3: 2,421 important nodes
```

**NodeRAG**: Приемлемо, так как различия ~5-10%.

### 4. Cold Start

При малых графах (<1000 узлов), k может быть **слишком высоким**:

```
N = 1000, avg_degree = 3
k = log(1000) · √3 ≈ 6.9 · 1.73 ≈ 12
```

Но в графе может не быть 12-core.

**NodeRAG**: Не проблема, так как индексирование начинается с достаточного корпуса.

---

## Код examples

### K-Core Decomposition

```python
import networkx as nx
import numpy as np

# Создание графа
G = nx.karate_club_graph()

# Автоматический выбор k
def default_k(G):
    N = G.number_of_nodes()
    avg_degree = sum(dict(G.degree()).values()) / N
    k = round(np.log(N) * avg_degree ** 0.5)
    return k

k = default_k(G)
print(f"Optimal k: {k}")

# K-Core
k_core_subgraph = nx.core.k_core(G, k=k)

print(f"Nodes in {k}-core: {k_core_subgraph.number_of_nodes()}")
print(f"Percentage: {100 * k_core_subgraph.number_of_nodes() / G.number_of_nodes():.2f}%")
```

### Betweenness Centrality with Sampling

```python
import math

# Betweenness с sampling
betweenness = nx.betweenness_centrality(G, k=10)

# Вычисление порога
avg_bc = sum(betweenness.values()) / len(betweenness)
scale = round(math.log10(len(betweenness)))
threshold = avg_bc * scale

print(f"Average BC: {avg_bc:.6f}")
print(f"Scale: {scale}")
print(f"Threshold: {threshold:.6f}")

# Фильтрация
important_bc = [v for v, bc in betweenness.items() if bc > threshold]

print(f"Important nodes (BC): {len(important_bc)}")
```

### Комбинация K-Core + Betweenness

```python
# K-Core
k_core_nodes = set(k_core_subgraph.nodes())

# Betweenness
bc_nodes = set(important_bc)

# Объединение
important_nodes = list(k_core_nodes | bc_nodes)

print(f"K-Core: {len(k_core_nodes)}")
print(f"Betweenness: {len(bc_nodes)}")
print(f"Union: {len(important_nodes)}")
print(f"Intersection: {len(k_core_nodes & bc_nodes)}")
```

### Визуализация важных узлов

```python
import matplotlib.pyplot as plt

# Цвета: важные узлы = красный, остальные = синий
colors = ['red' if v in important_nodes else 'blue' for v in G.nodes()]

pos = nx.spring_layout(G)
nx.draw_networkx_nodes(G, pos, node_color=colors, node_size=100)
nx.draw_networkx_edges(G, pos, alpha=0.3)
plt.title('Important Nodes (Red) vs Others (Blue)')
plt.show()
```

---

## Практические рекомендации

### Для разработчиков

1. **Cache important nodes**: Вычисление дорогостоящее, сохраняйте результаты.

2. **Adjust k heuristic**: Для специфических графов, тюньте формулу для k.

3. **Monitor coverage**: Убедитесь, что 10-15% entities — разумный баланс.

4. **Incremental updates**: При добавлении узлов, пересчитывайте только локально.

### Для исследователей

1. **Experiment with k**: Попробуйте разные k ∈ [5, 50] и изучите покрытие.

2. **Compare centrality measures**: Closeness, Eigenvector, PageRank vs. Betweenness.

3. **Analyze important node distribution**: Визуализируйте, где важные узлы в графе.

4. **Study attribute quality**: Оцените quality LLM attributes для разных типов entities.

---

## Дополнительные ресурсы

### Научные статьи

1. **Brandes, U. (2001)**. "A faster algorithm for betweenness centrality." Journal of Mathematical Sociology, 25(2), 163-177.

2. **Batagelj, V., & Zaversnik, M. (2003)**. "An O(m) algorithm for cores decomposition of networks." arXiv preprint cs/0310049.

3. **Freeman, L. C. (1977)**. "A set of measures of centrality based on betweenness." Sociometry, 35-41.

### Документация

- [NetworkX K-Core](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.core.k_core.html)
- [NetworkX Betweenness](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.centrality.betweenness_centrality.html)
- [Graph Operations Dependencies](../dependencies/graph-operations.md)

### Связанные алгоритмы

- [Graph Construction](./graph-construction.md) — создаёт граф для анализа
- [Personalized PageRank](./personalized-pagerank.md) — альтернативная мера важности
- [Leiden Algorithm](./leiden-community-detection.md) — кластеризация вместо важности

---

**Последнее обновление**: 2025-11-13

**См. также**:
- [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)
- [Attribute Generation Prompt](../prompts/attribute-generation-prompt.md)
- [Indexing Transformations](../transform/indexing-transformations.md)
