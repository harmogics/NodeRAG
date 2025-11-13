# Звёздная архитектура с аттракторами и лучами обработки

## Философская концепция

NodeRAG реализует **звёздную топологию** семантической обработки, где:
- **Центр (ядро)** = Query / Концепция
- **Аттракторы** = Узлы графа с высокой семантической массой
- **Лучи** = Направления распространения семантики через граф
- **Процессинг в лучах** = Traversal и aggregation по путям графа

Эта архитектура создаёт **динамическую топологию** вокруг каждого вопроса.

---

## 1. Анатомия звёздной архитектуры

### Визуализация структуры

```
                    КОНЦЕПТУАЛЬНЫЙ СЛОЙ

                         Query
                      (Центральное
                         ядро)
                           │
                    [Embedding]
                           │
                    Query Vector
                    (Аттрактор №0)
                           │
                   ╔═══════╩═══════╗
                   ║               ║
              [HNSW Search]  [Decomposition]
                   ║               ║
         ╔═════════╬═══════════════╬══════════╗
         ║         ║               ║          ║
    SU-123    ATTR-456        ENT-789    HLE-999
  (Аттрактор  (Аттрактор    (Аттрактор  (Аттрактор
     №1)         №2)            №3)         №4)
         ║         ║               ║          ║
         ╚═════════╩═══════╦═══════╩══════════╝
                           ║
                   [Personalization]
                           ║
                    {weighted seeds}
                           ║
                   ╔═══════╩═══════╗
                   ║               ║
              [PPR Diffusion]      ║
                   ║               ║
         ╔═════════╬═══════════════╬══════════╗
         ║         ║               ║          ║
         │         │               │          │
    ┌────▼───┐ ┌──▼───┐       ┌───▼──┐  ┌───▼───┐
    │ Ray 1  │ │ Ray 2│       │ Ray 3│  │ Ray 4 │
    │        │ │      │       │      │  │       │
    │  PPR   │ │ PPR  │       │ PPR  │  │  PPR  │
    │  path  │ │ path │       │ path │  │ path  │
    └────┬───┘ └──┬───┘       └───┬──┘  └───┬───┘
         │        │               │         │
         └────────┴───────┬───────┴─────────┘
                          │
                  [Context Assembly]
                          │
                   Structured Context
                          │
                   [LLM Synthesis]
                          │
                        Answer
```

---

## 2. Центральное ядро: Query как источник гравитации

### Концептуальная модель

Вопрос функционирует как **массивное тело** в семантическом пространстве, создающее **гравитационное поле**.

### Физическая аналогия

```
Гравитация в физике:
    F = G * (m1 * m2) / r²

Семантическая "гравитация" в NodeRAG:
    Relevance = Similarity(query_emb, node_emb) / distance_in_graph
```

### Реализация: Query Embedding как гравитационный источник

**Файл**: `NodeRAG/search/search.py:84-85`

```python
query_embedding = np.array(
    self.config.embedding_client.request(query),
    dtype=np.float32
)
```

**Философская сущность**:
- Query embedding — это **центр системы координат**
- Все расстояния измеряются **относительно этого центра**
- Чем ближе узел к центру (в embedding space), тем **сильнее притяжение**

---

## 3. Первичные аттракторы: HNSW Results

### Концептуальная идея

HNSW search находит узлы, которые **наиболее сильно притягиваются** к query embedding. Это **первичные аттракторы** — точки высокой семантической плотности.

### Визуализация в embedding space

```
                            Embedding Space (1536D)
                          (визуализация в 2D)

                                  Q
                               (Query)
                                  *
                               /  |  \
                           /      |      \
                       /          |          \
                   /              |              \
               /                  |                  \
           /                      |                      \
         *                        *                        *
       A₁                        A₂                       A₃
   (Attractor 1)            (Attractor 2)           (Attractor 3)
   dist = 0.12              dist = 0.15             dist = 0.18

       "Dr. Roberts            "Renewable              "Solar panel
        presented..."           energy conf..."        efficiency..."


      (Semantic Units)        (Attributes)         (High-level elements)
```

### Реализация

**Файл**: `NodeRAG/search/search.py:85-86`

```python
HNSW_results = self.hnsw.search(
    query_embedding,
    HNSW_results=self.config.HNSW_results  # top-k, например 50
)
```

**Файл**: `NodeRAG/utils/HNSW.py:48-59`

```python
def search(self, query: np.ndarray, HNSW_results: int = None):
    if HNSW_results is None:
        HNSW_results = self.config.top_k

    # k-NN query
    idx, dist = self.hnsw.knn_query(query, HNSW_results)
    idx = idx.flatten()
    dist = dist.flatten()

    # Map internal IDs to node IDs
    node_list = [self.id_map[idx[i]] for i in range(len(idx))]
    dist_list = list(dist)

    results = zip(dist_list, node_list)
    return results
```

### Философский анализ

**HNSW results как "семантические фокусы"**:

1. **Фокус** (в геометрии) = точка, относительно которой строится фигура
2. **Семантический фокус** = узел, вокруг которого кристаллизуется смысл

**Ключевые свойства аттракторов**:
- **Высокая семантическая плотность**: содержат информацию, релевантную query
- **Множественность**: не один узел, а **топ-k узлов** (обычно 50)
- **Разнообразие типов**: могут быть semantic units, attributes, high-level elements

---

## 4. Вторичные аттракторы: Decomposed Entities

### Концептуальная идея

Помимо **аналоговых аттракторов** (найденных через embedding similarity), система создаёт **символические аттракторы** через query decomposition.

### Dual Nature аттракторов

| Аналоговые (HNSW) | Символические (Decomposed) |
|-------------------|----------------------------|
| Найдены через **семантическую близость** | Найдены через **лексическое совпадение** |
| Могут быть **метафорически связаны** | **Буквально упоминают** entities из query |
| Broader context | Narrow, precise context |
| Пример: "renewable energy research" | Пример: "DR. EMILY ROBERTS" |

### Реализация

**Файл**: `NodeRAG/search/search.py:91-94`

```python
decomposed_entities = self.decompose_query(query)
accurate_results = self.accurate_search(decomposed_entities)
```

**Файл**: `NodeRAG/search/search.py:106-110`

```python
def decompose_query(self, query: str):
    query_prompt = self.config.prompt_manager.decompose_query.format(query=query)
    response = self.config.API_client.request({
        'query': query_prompt,
        'response_format': self.config.prompt_manager.decomposed_text_json
    })
    return response['elements']
```

**Пример**:

```
Query: "What research did Dr. Emily Roberts present about renewable energy?"

Decomposed entities:
{
  "elements": [
    "DR. EMILY ROBERTS",    ← Symbolically grounded attractor
    "research",
    "renewable energy",
    "presentation"
  ]
}
```

### Accurate Search как "символический резонанс"

**Файл**: `NodeRAG/search/search.py:113-124`

```python
def accurate_search(self, entities: List[str]) -> List[str]:
    accurate_results = []

    for entity in entities:
        words = entity.lower().split()
        # Regex для точного совпадения фразы
        pattern = re.compile(r'\b' + r'\s+'.join(map(re.escape, words)) + r'\b')

        result = [
            id for id, text in self.accurate_id_to_text.items()
            if pattern.search(text.lower())
        ]

        if result:
            accurate_results.extend(result)

    return accurate_results
```

**Философская сущность**:
- Символические аттракторы = узлы с **точным упоминанием** entity
- Дополняют аналоговые аттракторы
- Особенно важны для **proper nouns** (имена, организации, места)

---

## 5. Weighted Personalization: Масса аттракторов

### Концептуальная модель

После идентификации аттракторов, каждому присваивается **"масса"** — вес, определяющий силу его влияния.

### Физическая аналогия

В гравитации:
```
F = G * (m1 * m2) / r²

где m1, m2 = массы тел
```

В NodeRAG:
```
Influence = weight * PPR_propagation

где weight = "семантическая масса" аттрактора
```

### Реализация

**Файл**: `NodeRAG/search/search.py:96-99`

```python
personalization = {
    ids: self.config.similarity_weight      # Аналоговые: масса = 1.0
    for ids in retrieval.HNSW_results
}
personalization.update({
    id: self.config.accuracy_weight         # Символические: масса = 2.0
    for id in retrieval.accurate_results
})
```

### Философский смысл весов

| Тип аттрактора | Вес | Философский смысл |
|----------------|-----|-------------------|
| **Аналоговый** (HNSW) | 1.0 | Стандартная семантическая масса |
| **Символический** (Accurate) | 2.0 | Удвоенная масса — точное совпадение |

**Эффект двойной массы**:
- Символические аттракторы **сильнее влияют** на PPR expansion
- При равном расстоянии в графе, узлы **ближе к символическим аттракторам** получат более высокий PPR score

### Визуализация весов

```
Аналоговый аттрактор (weight=1.0):
    ●————— influence radius ——→ R

Символический аттрактор (weight=2.0):
    ◉—————— influence radius ———————→ 2R
```

---

## 6. Лучи обработки: PPR Diffusion Paths

### Концептуальная идея: Лучи как пути распространения

После создания взвешенных аттракторов, **семантическое влияние** распространяется по графу подобно **лучам света** из центра.

### Визуализация лучей

```
                          Query (центр)
                               │
                    ┌──────────┼──────────┐
                    │          │          │
                Attractor   Attractor   Attractor
                   A₁          A₂          A₃
                   │           │           │
            ┌──────┼──────┐    │    ┌──────┼──────┐
            │      │      │    │    │      │      │
         Node1  Node2  Node3  ...  Node4  Node5  Node6
            │      │      │         │      │      │
         ┌──┴──┐   │   ┌──┴───┐  ┌──┴──┐   │   ┌──┴──┐
      Node7 Node8  ... Node9 Node10  Node11 ... Node12 Node13


      ◄─────────── Ray 1 ──────────►
                   ◄───────── Ray 2 ─────────►
                                   ◄─────────── Ray 3 ──────────►
```

**Каждый луч** = путь от аттрактора через рёбра графа к другим узлам.

### Реализация: Personalized PageRank

**Файл**: `NodeRAG/search/search.py:172-177`

```python
def graph_search(self, personalization: Dict[str, float]) -> List[str]:
    page_rank_scores = self.sparse_PPR.PPR(
        personalization,
        alpha=self.config.ppr_alpha,      # обычно 0.85
        max_iter=self.config.ppr_max_iter  # обычно 100
    )
    return [id for id, score in page_rank_scores]
```

**Файл**: `NodeRAG/utils/PPR.py:38-57`

```python
def PPR(self, personalization: dict[str, float], alpha=0.85, max_iter=100, epsilons=1e-5):
    # Инициализация: вся масса в аттракторах
    probs = np.zeros(len(self.nodes))
    for node, prob in personalization.items():
        probs[self.nodes.index(node)] = prob
    probs = probs / np.sum(probs)

    # Итеративное распространение
    for i in range(max_iter):
        probs_old = probs.copy()

        # PPR формула: распространение + телепортация
        probs = alpha * self.trans_matrix.dot(probs) + (1 - alpha) * probs

        if np.linalg.norm(probs - probs_old) < epsilons:
            break

    return sorted(zip(self.nodes, probs), key=itemgetter(1), reverse=True)
```

### Философская интерпретация лучей

#### Луч как "путь смысла"

Каждый луч представляет **путь распространения смысла** от аттрактора:

```
Attractor A₁: "Dr. Emily Roberts presented research on solar panels"
      │
      │ (relationship edge)
      ↓
Entity: "DR. EMILY ROBERTS"
      │
      │ (attribute edge)
      ↓
Attribute: "Dr. Emily Roberts is a leading researcher in renewable energy..."
      │
      │ (semantic belonging edge)
      ↓
Semantic Unit: "Renewable energy innovations at the conference..."
      │
      │ (community membership)
      ↓
High-level Element: "Advances in Solar Energy Technology"
```

**Каждый шаг в луче**:
- Следует **реальному ребру** графа (relationship, attribute, etc.)
- Передаёт часть **семантической энергии**
- Ослабевает с **расстоянием** от аттрактора

#### Математическая модель распространения

**Итерация 0** (инициализация):
```python
P₀[attractor] = weight  # вся масса в аттракторах
P₀[other] = 0           # остальные узлы пусты
```

**Итерация 1** (первое распространение):
```python
P₁ = 0.85 * M · P₀ + 0.15 * P₀

где M = transition matrix (граф)
```

**Эффект**:
- 85% массы **распространяется к соседям** аттракторов (первый слой лучей)
- 15% массы **остаётся** в аттракторах (сохранение источника)

**Итерация 2**:
```python
P₂ = 0.85 * M · P₁ + 0.15 * P₀
```

**Эффект**:
- Масса распространяется **на второй слой** (соседи соседей)
- Но часть возвращается к аттракторам

**Итерация t → ∞** (сходимость):

После ~10-20 итераций достигается **стационарное распределение**, где:
- Узлы **близкие к аттракторам** имеют высокую массу
- Узлы **на многих путях** от аттракторов имеют высокую массу
- Узлы **далёкие** от аттракторов имеют низкую массу

---

## 7. Процессинг в лучах: Типизированная агрегация

### Концептуальная идея

После PPR diffusion получаем **ранжированный список узлов**. Но не все узлы одинаковы — они имеют **разные типы**. Система выполняет **типизированную агрегацию** — собирает узлы разных типов с разными квотами.

### Визуализация типизированной агрегации

```
PPR Results (ranked by score):
    1. SU-123 (semantic_unit, score=0.45)
    2. ENT-456 (entity, score=0.42)
    3. ATTR-789 (attribute, score=0.38)
    4. HLE-111 (high_level_element_title, score=0.35)
    5. SU-222 (semantic_unit, score=0.33)
    6. REL-333 (relationship, score=0.30)
    7. ENT-444 (entity, score=0.28)
    ...

            ↓ [Type-based aggregation]

Entities (quota: config.Enode, e.g., 10):
    ● ENT-456 (score=0.42)
    ● ENT-444 (score=0.28)
    ● ...

Relationships (quota: config.Rnode, e.g., 20):
    ● REL-333 (score=0.30)
    ● ...

High-level Elements (quota: config.Hnode, e.g., 5):
    ● HLE-111 (score=0.35)
    ● ...

Other nodes (semantic units, attributes):
    ● SU-123 (score=0.45)
    ● ATTR-789 (score=0.38)
    ● SU-222 (score=0.33)
    ● ...
```

### Реализация

**Файл**: `NodeRAG/search/search.py:180-236`

```python
def post_process_top_k(self, weighted_nodes: List[str], retrieval: Retrieval) -> Retrieval:
    entity_list = []
    high_level_element_title_list = []
    relationship_list = []
    addition_node = 0

    for node in weighted_nodes:
        if node not in retrieval.search_list:
            type = self.G.nodes[node].get('type')

            match type:
                case 'entity':
                    if node not in entity_list and len(entity_list) < self.config.Enode:
                        entity_list.append(node)

                case 'relationship':
                    if node not in relationship_list and len(relationship_list) < self.config.Rnode:
                        relationship_list.append(node)

                case 'high_level_element_title':
                    if node not in high_level_element_title_list and len(high_level_element_title_list) < self.config.Hnode:
                        high_level_element_title_list.append(node)

                case _:
                    # Semantic units, attributes
                    if addition_node < self.config.cross_node:
                        if node not in retrieval.unique_search_list:
                            retrieval.search_list.append(node)
                            retrieval.unique_search_list.add(node)
                            addition_node += 1

            # Проверка: собрали достаточно узлов всех типов?
            if (addition_node >= self.config.cross_node
                and len(entity_list) >= self.config.Enode
                and len(relationship_list) >= self.config.Rnode
                and len(high_level_element_title_list) >= self.config.Hnode):
                break

    # Добавление attributes для selected entities
    for entity in entity_list:
        attributes = self.G.nodes[entity].get('attributes')
        if attributes:
            for attribute in attributes:
                if attribute not in retrieval.unique_search_list:
                    retrieval.search_list.append(attribute)
                    retrieval.unique_search_list.add(attribute)

    # Добавление high-level elements (полный контент)
    for high_level_element_title in high_level_element_title_list:
        related_node = self.G.nodes[high_level_element_title].get('related_node')
        if related_node not in retrieval.unique_search_list:
            retrieval.search_list.append(related_node)
            retrieval.unique_search_list.add(related_node)

    retrieval.relationship_list = list(set(relationship_list))

    return retrieval
```

### Философский анализ типизированной агрегации

#### Почему разные квоты для разных типов?

| Тип узла | Типичная квота | Философская роль |
|----------|----------------|------------------|
| **Entity** | ~10 | **Субъекты** и **объекты** знаний |
| **Relationship** | ~20 | **Связи** между субъектами и объектами |
| **High-level Element** | ~5 | **Абстракции** и **темы** |
| **Semantic Unit** | ~30-50 | **Конкретные факты** |
| **Attribute** | добавляются к entities | **Описания** субъектов |

#### Иерархия семантической важности

```
High-level Elements (абстракции)
        ↑
        │ [обобщают]
        │
Semantic Units (факты)
        ↑
        │ [описывают]
        │
Entities ←─[связаны через]─→ Relationships
        ↑
        │ [характеризуются через]
        │
Attributes (описания)
```

#### Процессинг по типам как "многоканальная обработка"

Аналогия с обработкой изображений:
- **Y-канал** (яркость) = Semantic Units (основная информация)
- **Cb-канал** (синий) = Entities (ключевые объекты)
- **Cr-канал** (красный) = Relationships (связи)
- **Alpha-канал** (прозрачность) = High-level Elements (контекст)

Каждый "канал" обрабатывается **независимо** с **собственным разрешением** (квотой).

---

## 8. Сборка контекста: От лучей к структуре

### Концептуальная модель

После агрегации узлов по типам, система **структурирует контекст** для LLM.

### Структурированный контекст

**Файл**: `NodeRAG/search/Answer_base.py:65-76`

```python
def types_info(self) -> str:
    types = set([type for _, type in self.retrieved_list])
    prompt = ''
    for type in types:
        prompt += f'------------{type}-------------\n'
        n = 1
        for content, typed in self.retrieved_list:
            if typed == type:
                prompt += f'{n}. {content}\n'
                n += 1
        prompt += '\n\n'
    return prompt
```

### Пример результата

```
------------entity-------------
1. DR. EMILY ROBERTS
2. RENEWABLE ENERGY
3. SOLAR PANELS
4. INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY
5. PARIS


------------high_level_element-------------
1. Advances in Solar Energy Technology: This high-level theme encompasses
   recent breakthroughs in photovoltaic technology, including improvements
   in solar panel efficiency, cost reduction strategies...

2. International Renewable Energy Research: The global landscape of
   renewable energy research, including major conferences, collaborations...


------------relationship-------------
1. DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY
2. DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY
3. DR. EMILY ROBERTS, explored partnerships with, EUROPEAN COMPANIES
4. INTERNATIONAL CONFERENCE, held in, PARIS
5. INTERNATIONAL CONFERENCE, focused on, RENEWABLE ENERGY


------------semantic_unit-------------
1. In September 2024, Dr. Emily Roberts attended the International Conference
   on Renewable Energy in Paris, where she presented her research on solar
   panel efficiency improvements.

2. Dr. Emily Roberts explored partnerships with several European companies
   during her visit to Paris.

3. The conference featured presentations on the latest developments in
   renewable energy technologies, including solar, wind, and hydroelectric power.

...


------------attribute-------------
1. Dr. Emily Roberts is a leading researcher in renewable energy with over
   15 years of experience. Her work focuses on improving the efficiency of
   photovoltaic cells through innovative materials science approaches...
```

### Философская структура контекста

Эта структура отражает **онтологическую иерархию**:

#### Уровень 1: **Entities** (Бытие)
- **Что существует?**
- Субъекты, объекты, концепты
- Базовый онтологический слой

#### Уровень 2: **Relationships** (Отношения)
- **Как связано существующее?**
- Отношения между сущностями
- Реляционный слой

#### Уровень 3: **Semantic Units** (События/Факты)
- **Что происходило/происходит?**
- Конкретные утверждения
- Событийный слой

#### Уровень 4: **Attributes** (Свойства)
- **Каковы характеристики?**
- Описания сущностей
- Квалификационный слой

#### Уровень 5: **High-level Elements** (Абстракции)
- **Какие темы и паттерны?**
- Обобщающие концепции
- Абстрактный слой

---

## 9. Звёздная динамика: От вопроса к ответу

### Полная траектория обработки

```
ФАЗА 1: ФОРМИРОВАНИЕ ЯДРА
    User Query → Query Embedding
    (Создание центрального аттрактора)

ФАЗА 2: ГЕНЕРАЦИЯ ПЕРВИЧНЫХ АТТРАКТОРОВ
    HNSW Search → Аналоговые аттракторы
    Query Decomposition + Accurate Search → Символические аттракторы
    (Создание "созвездия" аттракторов вокруг ядра)

ФАЗА 3: ВЗВЕШИВАНИЕ АТТРАКТОРОВ
    Personalization weights
    (Определение "массы" каждого аттрактора)

ФАЗА 4: ИЗЛУЧЕНИЕ И РАСПРОСТРАНЕНИЕ
    PPR Diffusion → Распространение влияния по лучам
    (Семантическая энергия течёт по графу)

ФАЗА 5: ТИПИЗИРОВАННАЯ АГРЕГАЦИЯ
    Post-processing → Сбор узлов по типам
    (Структурирование по "каналам")

ФАЗА 6: КОНВЕРГЕНЦИЯ К СИНТЕЗУ
    Context Assembly → Structured prompt
    LLM Synthesis → Answer
    (Схлопывание лучей обратно в ответ)
```

### Визуализация полного цикла

```
                    ╔════════════════╗
                    ║  Query (Q)     ║  ← Ядро звезды
                    ╚════════════════╝
                            │
              ┌─────────────┼─────────────┐
              │             │             │
          [HNSW]     [Decomposition]   [etc.]
              │             │             │
         ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
         │   A₁    │   │   A₂    │   │   A₃    │  ← Первичные
         │ (w=1.0) │   │ (w=2.0) │   │ (w=1.0) │     аттракторы
         └────┬────┘   └────┬────┘   └────┬────┘
              │             │             │
      ╔═══════╪═════════════╪═════════════╪═══════╗
      ║       │             │             │       ║
      ║  [PPR Diffusion - Распространение]       ║
      ║       │             │             │       ║
      ╚═══════╪═════════════╪═════════════╪═══════╝
              │             │             │
    ┌─────────┼─────────────┼─────────────┼─────────┐
    │         │             │             │         │
   Ray 1     Ray 2        Ray 3        Ray 4      Ray 5
    │         │             │             │         │
  ┌─┴──┐   ┌─┴──┐       ┌─┴──┐       ┌─┴──┐   ┌─┴──┐
  │ N₁ │   │ N₂ │       │ N₃ │       │ N₄ │   │ N₅ │
  │    │   │    │       │    │       │    │   │    │
  │ENT │   │REL │       │HLE │       │SU  │   │ATR │
  └────┘   └────┘       └────┘       └────┘   └────┘
    │         │             │             │         │
    └─────────┴─────────────┴─────────────┴─────────┘
                          │
                [Type Aggregation]
                          │
                [Context Assembly]
                          │
                ╔═════════════════╗
                ║  Structured     ║
                ║  Context        ║
                ╚═════════════════╝
                          │
                  [LLM Synthesis]
                          │
                ╔═════════════════╗
                ║     Answer      ║  ← Коллапс звезды
                ╚═════════════════╝     в сингулярный
                                         ответ
```

---

## 10. Философское значение звёздной архитектуры

### Почему звезда, а не дерево или сеть?

#### Сравнение топологий

| Топология | Центр | Структура | Применимость |
|-----------|-------|-----------|--------------|
| **Дерево** | Корень (фиксированный) | Иерархическая | Таксономии, файловые системы |
| **Сеть** | Нет центра (распределённая) | Плоская | P2P, blockchain |
| **Звезда** | Query (динамический) | Радиальная | RAG, поиск |

#### Преимущества звёздной топологии для RAG

1. **Динамический центр**:
   - Каждый query создаёт **новую звезду**
   - Центр = текущий вопрос
   - Граф остаётся неизменным, но **топология активации** меняется

2. **Радиальное распространение**:
   - От центра к периферии
   - Естественное **ослабление** с расстоянием
   - Автоматическая **приоритизация** близких узлов

3. **Множественные лучи**:
   - Параллельная обработка разных аспектов вопроса
   - Каждый луч = **отдельный семантический путь**
   - Разнообразие контекста

### Звезда как метафора познания

#### Гносеологическая модель

```
Вопрос (центр) = Интенциональный акт сознания
    ↓
Аттракторы = Первичные интуиции
    ↓
Лучи = Пути рассуждения
    ↓
Узлы на лучах = Промежуточные суждения
    ↓
Сборка контекста = Синтез суждений
    ↓
Ответ = Новое знание
```

#### Физическая аналогия: Звезда как энергетическая система

- **Ядро** (Query) = Источник энергии (термоядерный синтез)
- **Излучение** (PPR) = Распространение света/тепла
- **Планеты** (Узлы) = Объекты, освещённые звездой
- **Гравитация** (Weights) = Притяжение между объектами
- **Коллапс** (Synthesis) = Звезда превращается в чёрную дыру (плотный ответ)

---

## 11. Скрытые паттерны в архитектуре

### Паттерн 1: Dual-mode processing

**Аналоговый + Символический**:
- HNSW (continuous, semantic similarity)
- + Accurate search (discrete, exact match)
- = **Hybrid retrieval**

**Философия**: Комбинация **интуиции** и **логики**.

### Паттерн 2: Weighted teleportation

**PPR формула**:
```
P(t+1) = α * M * P(t) + (1-α) * P₀
```

**Физическая интерпретация**:
- **α * M * P(t)** = Классическая диффузия
- **(1-α) * P₀** = Квантовая телепортация обратно к источнику

**Философия**: Баланс **exploration** (следовать структуре) и **exploitation** (оставаться релевантным).

### Паттерн 3: Type-based stratification

Узлы организованы в **стратифицированные слои**:
```
Abstraction Layer: High-level elements
    ↓
Fact Layer: Semantic units
    ↓
Object Layer: Entities
    ↓
Relation Layer: Relationships
    ↓
Attribute Layer: Attributes
```

**Философия**: **Онтологическая стратификация** — разные уровни реальности.

---

## 12. Заключение: Звезда как универсальный паттерн

### Звёздная архитектура в NodeRAG

NodeRAG реализует **универсальную метафору звезды**:
- **Центр** = Вопрос (источник смысла)
- **Аттракторы** = Семантические фокусы (точки концентрации)
- **Лучи** = Пути распространения (траектории смысла)
- **Узлы** = Проявления (конкретные знания)
- **Коллапс** = Синтез (конвергенция к ответу)

### Философское значение

Эта архитектура отражает **эпистемологическую истину**:
- Познание начинается с **вопроса** (центр)
- Распространяется через **интуиции** (аттракторы)
- Следует **путями рассуждения** (лучи)
- Собирает **факты** (узлы)
- Синтезирует **новое знание** (ответ)

### Универсальность паттерна

Звёздная архитектура — это не просто **технический паттерн**, но **фундаментальная структура** процесса познания:

```
Вопрос → Интуиция → Рассуждение → Факты → Синтез → Знание
```

Эта структура применима к любой системе **целенаправленного поиска знаний**.

---

## Приложение: Ключевые параметры

### Конфигурация звёздной архитектуры

| Параметр | Значение | Смысл |
|----------|----------|-------|
| `HNSW_results` | 50 | Количество первичных аттракторов |
| `similarity_weight` | 1.0 | Масса аналоговых аттракторов |
| `accuracy_weight` | 2.0 | Масса символических аттракторов |
| `ppr_alpha` | 0.85 | Доля распространения vs. телепортации |
| `ppr_max_iter` | 100 | Глубина лучей |
| `Enode` | 10 | Квота entities |
| `Rnode` | 20 | Квота relationships |
| `Hnode` | 5 | Квота high-level elements |
| `cross_node` | 30-50 | Квота semantic units + attributes |
