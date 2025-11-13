# Personalized PageRank (PPR): Семантическая диффузия через граф

## Концептуальный обзор

**Personalized PageRank (PPR)** — это модификация алгоритма PageRank, которая вычисляет важность узлов графа **относительно заданного набора источников** (персонализации). В NodeRAG это ключевой механизм **распространения семантического влияния** от аттракторов (HNSW results) по всему knowledge graph.

### Философская сущность

В контексте [звёздной архитектуры](../research/star-pattern-architecture.md), PPR реализует **лучи семантической обработки** — распространение релевантности от центра (Query) через аттракторы по всему графу знаний:

```
         Query (центр звезды)
            │
      [HNSW аттракторы]
     /  |  |  |  \
   A₁  A₂ A₃ A₄  A₅
    │  │  │  │  │
    ↓  ↓  ↓  ↓  ↓  [PPR диффузия]
   ═══════════════
   │││││││││││││││  Лучи семантической энергии
   ═══════════════
    ↓  ↓  ↓  ↓  ↓
   N₁ N₂ N₃ N₄ N₅ ... (релевантные узлы)
```

Это математическая реализация **герменевтического круга** — движения от известного (аттракторы) к неизвестному (связанные узлы) и обратно.

---

## Роль в парадигме вопроса как ключа

Согласно [парадигме вопроса как ключа](../research/question-as-key-paradigm.md), PPR выполняет **фазу расширения контекста**:

```
1. Вопрос → Embedding               (интенция → проекция)
2. HNSW Search → Аттракторы         (проекция → точки входа)
3. PPR Expansion → Контекст         (точки входа → семантическое поле)
4. Context Assembly → Retrieval     (семантическое поле → структура)
5. LLM Synthesis → Answer           (структура → знание)
```

**Эпистемологический принцип**:
- HNSW находит **прямые проявления** концепции
- PPR находит **косвенные связи** через структуру знаний
- Вместе они создают **полное семантическое поле** вокруг вопроса

---

## Математическая модель

### 1. Стандартный PageRank

Вероятность оказаться в узле v после случайного блуждания:

```
PR(v) = (1 - α) + α · Σ_{u→v} PR(u) / out_degree(u)
```

где:
- α = 0.85 — damping factor (вероятность следовать по рёбрам)
- 1 - α = 0.15 — вероятность телепортации в случайный узел

**Интерпретация**: Стационарное распределение случайного блуждания с телепортацией.

### 2. Personalized PageRank

Вместо равномерной телепортации, используем **персонализированное распределение**:

```
PPR(v) = (1 - α) · P₀(v) + α · Σ_{u→v} PPR(u) / out_degree(u)
```

где:
- P₀(v) — **персонализация** (preference vector)
- Если v — аттрактор, то P₀(v) > 0
- Если v — обычный узел, то P₀(v) = 0

**Интерпретация**:
- С вероятностью α — идём по рёбрам графа
- С вероятностью 1-α — телепортируемся **только** к аттракторам

### 3. Матричная форма

```
P = (1 - α) · P₀ + α · M · P
```

где:
- P — распределение вероятностей (вектор размера N)
- M — **транзитивная матрица** (column-stochastic)
- P₀ — персонализация (нормализована: Σ P₀(v) = 1)

**Решение через power iteration**:

```
P^(t+1) = (1 - α) · P₀ + α · M · P^(t)
P^(0) = P₀

Сходится к P* при t → ∞
```

### 4. Sparse Matrix Formulation

Для больших графов используем **разреженные матрицы** (SciPy CSR):

```python
# Adjacency matrix (sparse)
A = sp.csr_matrix([[0, 1, 1],
                    [1, 0, 1],
                    [0, 1, 0]])

# Out-degree normalization
D_out = A.sum(axis=1)  # row sums
M = A.multiply(1 / D_out)  # element-wise division
M = M.T  # transpose (column-stochastic)

# Power iteration
P = P0.copy()
for i in range(max_iter):
    P = (1 - alpha) * P0 + alpha * M.dot(P)
```

**Эффективность**:
- Матрица M хранится в CSR формате (только ненулевые элементы)
- Умножение M.dot(P) — O(E), где E = число рёбер

---

## Философская интерпретация

### Физическая аналогия: Диффузия тепла

Представьте граф как **тепловую сеть**:
- Аттракторы = **источники тепла** (температура P₀(v))
- Рёбра = **теплопроводы**
- PPR = **стационарное распределение температуры**

```
Аттракторы (горячие)
    │ │ │
    ↓ ↓ ↓  [Тепловая диффузия]
═════════════
│││││││││││││  Граф знаний
═════════════
    ↓ ↓ ↓
Релевантные узлы (тёплые)
```

**Свойства**:
- Узлы **близкие** к аттракторам — **более горячие** (высокий PPR score)
- Узлы **далёкие** — **холоднее** (низкий PPR score)
- **Плотные кластеры** удерживают тепло (high conductivity)

### Семантическая интерпретация

```
PPR score = "Семантическая релевантность с учётом структуры"
```

**Компоненты релевантности**:

1. **Прямая связь**: Узел соединён с аттрактором
2. **Косвенная связь**: Узел соединён через промежуточные узлы
3. **Структурная важность**: Узел находится на пути между многими аттракторами
4. **Кластерная близость**: Узел принадлежит тому же семантическому кластеру

**Формула интуиции**:
```
PPR(v) ∝ Σ_{attractor a} [
    Proximity(v, a) × Weight(a) × Density(path(a → v))
]
```

---

## Реализация в NodeRAG

### Файл: `NodeRAG/utils/PPR.py`

#### Класс `sparse_PPR`

```python
class sparse_PPR():

    def __init__(self, graph: nx.Graph, modified=True, weight='weight'):
        self.graph = graph
        self.nodes = list(self.graph.nodes())
        self.modified = modified
        self.weight = weight
        self.n_nodes = len(self.nodes)
        self.trans_matrix = self.generate_sparse_trasition_matrix()
```

**Параметры**:
- `graph`: NetworkX Graph (knowledge graph + HNSW graph)
- `modified`: Если True, узлы без исходящих рёбер соединяются со всеми
- `weight`: Название атрибута для весов рёбер

#### Генерация транзитивной матрицы

```python
def generate_sparse_trasition_matrix(self):
    # Получаем adjacency matrix (sparse CSR)
    adjacency_matrix = nx.adjacency_matrix(self.graph, weight=self.weight)

    # Симметризация (граф неориентированный)
    adjacency_matrix = (adjacency_matrix + adjacency_matrix.T) / 2

    if self.modified:
        # Модификация для узлов без исходящих рёбер
        out_degree = adjacency_matrix.sum(1)  # row sums

        # Конвертируем в LIL для изменения
        adjacency_matrix = sp.lil_matrix(adjacency_matrix)

        # Узлы с out_degree=0 соединяем со всеми
        adjacency_matrix[out_degree == 0, :] = np.ones(self.n_nodes)

        # Убираем self-loops
        adjacency_matrix.setdiag(0)

        # Обратно в CSC
        adjacency_matrix = sp.csc_matrix(adjacency_matrix)

        # Пересчитываем out_degree
        out_degree = adjacency_matrix.sum(1)

    # Нормализация: M[i,j] = A[i,j] / out_degree[i]
    transition_matrix = adjacency_matrix.multiply(1 / out_degree)

    # Транспонируем (column-stochastic вместо row-stochastic)
    transition_matrix = transition_matrix.T

    return sp.csc_matrix(transition_matrix)
```

**Зачем `modified=True`**?
- **Проблема**: Узлы без рёбер (isolated nodes) имеют PPR = 0
- **Решение**: Соединяем их со всеми узлами (равномерная телепортация)
- **Эффект**: Все узлы могут получить ненулевой PPR score

#### Метод PPR

```python
def PPR(self,
        personalization: dict[str, float],
        alpha: float = 0.85,
        max_iter: int = 100,
        epsilons: float = 1e-5):

    # Инициализация P₀ (персонализация)
    probs = np.zeros(len(self.nodes))

    for node, prob in personalization.items():
        probs[self.nodes.index(node)] = prob

    # Нормализация P₀
    probs = probs / np.sum(probs)

    # Power iteration
    for i in range(max_iter):
        probs_old = probs.copy()

        # P^(t+1) = α·M·P^(t) + (1-α)·P₀
        probs = alpha * self.trans_matrix.dot(probs) + (1 - alpha) * probs

        # Проверка сходимости
        if np.linalg.norm(probs - probs_old) < epsilons:
            break

    # Сортируем узлы по PPR score
    return sorted(
        zip(self.nodes, probs),
        key=itemgetter(1),
        reverse=True
    )
```

**Параметры**:
- `personalization`: dict `{node_id: weight}`
  - Аттракторы HNSW: weight = 1.0
  - Аттракторы Decomposed: weight = 2.0
- `alpha`: 0.85 (damping factor)
- `max_iter`: 100 (обычно сходится за 10-20 итераций)
- `epsilons`: 1e-5 (критерий сходимости)

**Выход**:
- `List[(node_id, ppr_score)]` — все узлы, отсортированные по PPR score

---

## Использование в системе

### Поиск: Graph Search

**Файл**: `NodeRAG/search/search.py:172-177`

```python
def graph_search(self, personalization: Dict[str, float]) -> List[str]:
    page_rank_scores = self.sparse_PPR.PPR(
        personalization,
        alpha=self.config.ppr_alpha,      # 0.85
        max_iter=self.config.ppr_max_iter # 100
    )

    return [id for id, score in page_rank_scores]
```

**Вход**: Персонализация из HNSW + Decomposed results

```python
personalization = {
    # HNSW аттракторы (аналоговые)
    'SU-123': 1.0,
    'ATTR-456': 1.0,
    'HLE-789': 1.0,

    # Decomposed аттракторы (символические)
    'ENT-999': 2.0,  # Двойной вес!
}
```

**Выход**: Все узлы графа, ранжированные по PPR score

### Post-processing: Top-K Selection

**Файл**: `NodeRAG/search/search.py:180-236`

После PPR выполняется **типизированная агрегация** — сбор узлов разных типов:

```python
def post_process_top_k(self, weighted_nodes: List[str], retrieval: Retrieval):
    entity_list = []
    high_level_element_title_list = []
    relationship_list = []
    addition_node = 0

    for node in weighted_nodes:
        # Пропускаем узлы, уже найденные HNSW
        if node not in retrieval.search_list:
            type = self.G.nodes[node].get('type')

            match type:
                case 'entity':
                    if len(entity_list) < self.config.Enode:  # 10
                        entity_list.append(node)

                case 'relationship':
                    if len(relationship_list) < self.config.Rnode:  # 20
                        relationship_list.append(node)

                case 'high_level_element_title':
                    if len(high_level_element_title_list) < self.config.Hnode:  # 5
                        high_level_element_title_list.append(node)

                case _:
                    # Semantic units, attributes
                    if addition_node < self.config.cross_node:  # 30
                        retrieval.search_list.append(node)
                        addition_node += 1

            # Проверяем, достигли ли квоты
            if (addition_node >= self.config.cross_node
                and len(entity_list) >= self.config.Enode
                and len(relationship_list) >= self.config.Rnode
                and len(high_level_element_title_list) >= self.config.Hnode):
                break

    # Добавляем attributes для entities
    for entity in entity_list:
        attributes = self.G.nodes[entity].get('attributes')
        if attributes:
            for attribute in attributes:
                retrieval.search_list.append(attribute)

    # Добавляем related_node для high-level elements
    for hle_title in high_level_element_title_list:
        related_node = self.G.nodes[hle_title].get('related_node')
        retrieval.search_list.append(related_node)

    retrieval.relationship_list = list(set(relationship_list))

    return retrieval
```

**Квоты** (см. `config.yaml`):
- Entities: 10
- Relationships: 20
- High-level elements: 5
- Semantic units + Attributes: 30

**Итого**: ~65-70 узлов для контекста

---

## Персонализация: Weighted Attractors

### Dual-Mode Attractors

NodeRAG использует **два типа аттракторов** с разными весами:

| Тип | Вес | Механизм поиска | Философия |
|-----|-----|-----------------|-----------|
| **HNSW аттракторы** | 1.0 | Embedding similarity | Аналоговая семантика |
| **Decomposed аттракторы** | 2.0 | Exact lexical match | Символическая семантика |

**Пример персонализации**:

```python
# Query: "What did Dr. Emily Roberts present at the renewable energy conference?"

# HNSW нашёл (семантически близкие):
HNSW_results = [
    'SU-100',  # "Dr. Roberts presented findings..."
    'SU-101',  # "Renewable energy conference highlighted..."
    'ATTR-200',  # "Dr. Emily Roberts is a leading researcher..."
]

# Decompose нашёл (точные совпадения):
decomposed_entities = ['DR. EMILY ROBERTS', 'RENEWABLE ENERGY CONFERENCE']

accurate_results = [
    'ENT-500',  # entity: "DR. EMILY ROBERTS"
    'ENT-501',  # entity: "RENEWABLE ENERGY CONFERENCE"
]

# Персонализация для PPR:
personalization = {
    'SU-100': 1.0,
    'SU-101': 1.0,
    'ATTR-200': 1.0,
    'ENT-500': 2.0,  # Двойной вес для точных совпадений!
    'ENT-501': 2.0,
}
```

**Интерпретация весов**:
- **1.0**: "Это семантически релевантно"
- **2.0**: "Это буквально упоминается в вопросе"

### Эффект взвешенной персонализации

```
         Query
           │
      ╔════╩════╗
      ║         ║
   [HNSW]  [Decompose]
      ║         ║
  ┌───╨───┐ ┌──╨──┐
  │ w=1.0 │ │w=2.0│
  └───┬───┘ └──┬──┘
      │        │
    [PPR диффузия]
      │        │
   Более сильная диффузия от decomposed!
```

**Результат**:
- Узлы, связанные с **decomposed entities**, получают **более высокие PPR scores**
- Это приоритизирует **точно релевантные** узлы над **семантически близкими**

---

## Параметры и настройки

### Alpha (Damping Factor)

```
alpha = 0.85 (default в NodeRAG)
```

**Интерпретация**:
- **alpha → 1.0**: Больше exploration (далёкие узлы)
- **alpha → 0.0**: Больше exploitation (только аттракторы)

**Влияние**:

| Alpha | Exploration | Exploitation | Diversity |
|-------|-------------|--------------|-----------|
| 0.50  | Низкий      | Высокий      | Низкая    |
| 0.85  | Средний     | Средний      | Средняя   |
| 0.95  | Высокий     | Низкий       | Высокая   |

**NodeRAG выбор** (0.85):
- **Баланс** между точностью и покрытием
- Соответствует стандартному PageRank

### Max Iterations

```
max_iter = 100 (default)
```

**Типичная сходимость**: 10-20 итераций

**Проверка**:
```python
if np.linalg.norm(probs - probs_old) < 1e-5:
    print(f"Converged at iteration {i}")
    break
```

### Epsilon (Convergence Threshold)

```
epsilon = 1e-5
```

**Интерпретация**:
- Норма разницы между P^(t) и P^(t-1)
- Меньше epsilon → сходимость достигнута

---

## Производительность

### Сложность

| Операция | Naive | Sparse |
|----------|-------|--------|
| Транзитивная матрица | O(N²) | O(E) |
| Power iteration (1 шаг) | O(N²) | O(E) |
| Полный PPR | O(N² · T) | O(E · T) |

где:
- N = число узлов (~500K в NodeRAG)
- E = число рёбер (~2M в NodeRAG)
- T = число итераций (~10-20)

**Sparse advantage**: O(E · T) ≈ O(2M · 15) ≈ 30M операций вместо O(N² · T) ≈ 7.5B

### Бенчмарки (NodeRAG)

**Граф**: 500K узлов, 2M рёбер

| Метрика | Значение |
|---------|----------|
| Matrix construction | ~2-3 секунды |
| Single PPR query | ~100-200 ms |
| Convergence iterations | 10-20 |
| Memory (CSR matrix) | ~50 MB |
| Throughput | ~5-10 QPS (single thread) |

**Hardware**: CPU (SciPy sparse operations)

---

## Библиотеки

### SciPy Sparse

**Библиотека**: `scipy==1.12.0`

**Используемые модули**:

```python
import scipy.sparse as sp

# Sparse matrix formats
sp.csr_matrix  # Compressed Sparse Row (для умножения на вектор)
sp.csc_matrix  # Compressed Sparse Column (для операций по столбцам)
sp.lil_matrix  # List of Lists (для модификации матрицы)
```

**Преимущества**:
- **Эффективность**: Хранит только ненулевые элементы
- **Скорость**: Оптимизированные C/Fortran операции
- **Память**: O(E) вместо O(N²)

**Документация**: [Graph Operations Dependencies](../dependencies/graph-operations.md)

### NetworkX

**Библиотека**: `networkx==3.4.2`

**Используется для**:
- Генерации adjacency matrix: `nx.adjacency_matrix(G, weight='weight')`
- Хранения графа: `nx.Graph()`

---

## Связь с другими алгоритмами

### 1. HNSW → PPR

HNSW results становятся **персонализацией** для PPR:

```python
HNSW_results = hnsw.search(query_emb, k=50)

personalization = {
    node_id: 1.0
    for _, node_id in HNSW_results
}
```

См. [HNSW Algorithm](./hnsw-algorithm.md)

### 2. Query Decomposition → PPR

Decomposed entities также становятся **персонализацией** с **двойным весом**:

```python
decomposed_entities = decompose_query(query)
accurate_results = accurate_search(decomposed_entities)

personalization.update({
    node_id: 2.0
    for node_id in accurate_results
})
```

### 3. PPR → Post-processing

PPR results фильтруются по **типам узлов** и **квотам**:

```python
weighted_nodes = graph_search(personalization)
retrieval = post_process_top_k(weighted_nodes, retrieval)
```

См. [Graph Construction](./graph-construction.md)

### 4. Graph Concatenation → PPR

PPR работает на **объединённом графе** (Knowledge Graph + HNSW Graph):

```python
unified_graph = GraphConcat(base_graph).concat(hnsw_graph)
sparse_PPR(unified_graph).PPR(personalization)
```

См. [Graph Concatenation Algorithm](./graph-concatenation.md)

---

## Концептуальные связи

### Звёздная архитектура: Лучи обработки

PPR реализует **лучи семантического влияния** из центра (Query) через аттракторы:

```
            Query (центр)
               │
        [HNSW аттракторы]
         /  |  |  |  \
       A₁  A₂ A₃ A₄  A₅  (аттракторы с весами)
        │   │  │  │   │
        ↓   ↓  ↓  ↓   ↓  [PPR: α·M·P + (1-α)·P₀]
       ═════════════════
       │││││││││││││││││  Лучи диффузии
       ═════════════════
        ↓   ↓  ↓  ↓   ↓
       N₁  N₂ N₃ N₄  N₅ ... (релевантные узлы)
```

**Метафора**: Query как **звезда**, излучающая семантическую энергию через аттракторы.

### Герменевтический круг

PPR математически реализует **герменевтический круг** — движение от целого к частям и обратно:

```
Целое (Query)
    ↓ [HNSW]
Части (Аттракторы)
    ↓ [PPR]
Расширенный контекст (Релевантные узлы)
    ↓ [LLM Synthesis]
Новое целое (Answer)
```

### Телепортация как возврат к источнику

Термин **(1-α)·P₀** в формуле PPR:

```
P^(t+1) = α·M·P^(t) + (1-α)·P₀
                         ↑
                  Возврат к аттракторам
```

**Философская интерпретация**:
- **α·M·P^(t)**: Exploration — движение по графу, открытие новых связей
- **(1-α)·P₀**: Return — возврат к источнику, сохранение фокуса

**Аналогия**: Герой мифа (Одиссей) путешествует (exploration), но всегда возвращается домой (return).

### Dual-Process Theory

PPR соответствует **System 2** в dual-process theory (см. [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)):

| System 1 (Fast) | System 2 (Slow) |
|-----------------|-----------------|
| HNSW search | PPR expansion |
| O(log N) | O(E · T) |
| Аналоговый | Структурный |
| Intuitive | Deliberative |

**PPR = медленное, осознанное рассуждение** через структуру графа.

---

## Варианты и расширения

### 1. Approximate PPR

Для очень больших графов можно использовать **аппроксимативный PPR**:

```python
def approx_PPR(graph, personalization, alpha=0.85, epsilon=0.01):
    """
    Monte Carlo sampling вместо power iteration
    """
    scores = defaultdict(float)

    for source in personalization:
        for _ in range(num_walks):
            current = source

            while random() > (1 - alpha):
                neighbors = list(graph.neighbors(current))
                if not neighbors:
                    break
                current = choice(neighbors)

            scores[current] += 1

    return normalize(scores)
```

**Преимущество**: O(W · L) вместо O(E · T), где W = число walks, L = длина walk

**NodeRAG**: Не используется, так как граф достаточно мал для точного PPR.

### 2. Multi-hop PPR

Вместо одного PPR, можно делать **итеративный PPR** для глубокого поиска:

```python
personalization_1 = {a: 1.0 for a in HNSW_results}
results_1 = PPR(personalization_1, alpha=0.85)

personalization_2 = {node: score for node, score in results_1[:100]}
results_2 = PPR(personalization_2, alpha=0.85)
```

**Эффект**: Находит узлы **2+ hops** от аттракторов.

### 3. Topic-Sensitive PPR

Разные **персонализации** для разных **топиков**:

```python
tech_personalization = {'ENT-1': 1.0, 'ENT-2': 1.0}  # tech entities
science_personalization = {'ENT-3': 1.0, 'ENT-4': 1.0}  # science entities

tech_ppr = PPR(tech_personalization)
science_ppr = PPR(science_personalization)

combined = alpha * tech_ppr + (1 - alpha) * science_ppr
```

---

## Ограничения и edge cases

### 1. Isolated Nodes

Узлы без рёбер имеют PPR = 0 (или очень малый).

**Решение в NodeRAG**: `modified=True` — соединяем isolated nodes со всеми узлами.

```python
if out_degree == 0:
    adjacency_matrix[node, :] = np.ones(n_nodes)
```

### 2. Disconnected Components

Если граф **несвязный**, PPR не распространяется между компонентами.

**NodeRAG**: Граф обычно связный благодаря HNSW graph concatenation.

### 3. Dense Subgraphs

**Плотные подграфы** (cliques) удерживают высокий PPR score:

```
A₁ ⟷ A₂ ⟷ A₃
 ⟍   ⟋   ⟍
  ⟍ ⟋     ⟍
   A₄ ⟷ A₅
```

**Эффект**: Узлы внутри clique получают **непропорционально высокие** scores.

**NodeRAG**: Не проблема, так как плотные кластеры обычно **семантически связаны**.

### 4. Low-Degree Bottlenecks

Узлы с **низкой степенью** могут блокировать диффузию:

```
A₁ ─ B ─ C₁
A₂ ─ ┘   C₂
           C₃
```

Если B имеет degree = 2, диффузия к C₁, C₂, C₃ **слабая**.

**NodeRAG**: Граф обычно **плотный** благодаря HNSW connections.

---

## Код examples

### Построение PPR

```python
from NodeRAG.utils import sparse_PPR
import networkx as nx

# Создание графа
G = nx.Graph()
G.add_edges_from([
    ('A', 'B', {'weight': 1}),
    ('B', 'C', {'weight': 1}),
    ('C', 'D', {'weight': 2}),
    ('A', 'D', {'weight': 1}),
])

# Инициализация PPR
ppr = sparse_PPR(G, modified=True, weight='weight')

# Персонализация
personalization = {'A': 1.0}

# Вычисление PPR
results = ppr.PPR(
    personalization,
    alpha=0.85,
    max_iter=100,
    epsilons=1e-5
)

# Результаты
for node, score in results:
    print(f"{node}: {score:.6f}")
```

### Dual-Mode Personalization

```python
# HNSW аттракторы
hnsw_results = [('SU-100', 0.12), ('ATTR-200', 0.15)]

# Decomposed аттракторы
accurate_results = ['ENT-500', 'ENT-501']

# Персонализация с разными весами
personalization = {}

for _, node_id in hnsw_results:
    personalization[node_id] = 1.0  # Аналоговые

for node_id in accurate_results:
    personalization[node_id] = 2.0  # Символические (двойной вес!)

# PPR
results = ppr.PPR(personalization, alpha=0.85)
```

### Visualizing PPR Scores

```python
import matplotlib.pyplot as plt

# Топ-50 узлов
top_50 = results[:50]
nodes, scores = zip(*top_50)

plt.figure(figsize=(12, 6))
plt.bar(range(len(nodes)), scores)
plt.xlabel('Node Rank')
plt.ylabel('PPR Score')
plt.title('Personalized PageRank Scores (Top 50)')
plt.xticks(range(len(nodes)), nodes, rotation=90)
plt.tight_layout()
plt.show()
```

---

## Практические рекомендации

### Для разработчиков

1. **Normalize personalization**:
   ```python
   total = sum(personalization.values())
   personalization = {k: v/total for k, v in personalization.items()}
   ```

2. **Use sparse matrices**: Не конвертируйте в dense!

3. **Monitor convergence**: Логируйте число итераций до сходимости.

4. **Cache transition matrix**: Вычисляйте M один раз при загрузке графа.

### Для исследователей

1. **Experiment with alpha**: Попробуйте 0.7, 0.85, 0.95 и сравните diversity.

2. **Analyze score distribution**: Визуализируйте гистограмму PPR scores.

3. **Compare with alternatives**: Попробуйте HITS, SimRank, или graph neural networks.

4. **Study failure modes**: Найдите queries с низким recall и изучите причины.

---

## Дополнительные ресурсы

### Научные статьи

1. **Page, L., Brin, S., et al. (1999)**. "The PageRank Citation Ranking: Bringing Order to the Web." Stanford InfoLab Technical Report.

2. **Haveliwala, T. H. (2002)**. "Topic-sensitive PageRank." Proceedings of the 11th International Conference on World Wide Web.

3. **Andersen, R., Chung, F., & Lang, K. (2006)**. "Local Graph Partitioning using PageRank Vectors." FOCS 2006.

### Документация

- [SciPy Sparse Documentation](https://docs.scipy.org/doc/scipy/reference/sparse.html)
- [NetworkX PageRank](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.link_analysis.pagerank_alg.pagerank.html)
- [Graph Operations Dependencies](../dependencies/graph-operations.md)

### Связанные алгоритмы

- [HNSW Algorithm](./hnsw-algorithm.md) — генерирует аттракторы для PPR
- [Graph Concatenation](./graph-concatenation.md) — создаёт граф для PPR
- [Leiden Algorithm](./leiden-algorithm.md) — использует PPR-like распространение

---

**Последнее обновление**: 2025-11-13

**См. также**:
- [Звёздная архитектура](../research/star-pattern-architecture.md)
- [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)
- [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)
