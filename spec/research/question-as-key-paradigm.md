# Парадигма вопроса как ключа в семантическом пространстве

## Философская основа

В NodeRAG реализована глубокая **эпистемологическая парадигма**, в которой вопрос функционирует не просто как запрос к базе данных, но как **ключ к семантическому пространству**, связывающий абстрактные концепции с их конкретными проявлениями в знаниях.

### Концептуальная модель

```
ВОПРОС (абстракция)
      ↓
  [Embedding]
      ↓
КЛЮЧ в векторном пространстве
      ↓
  [HNSW Search]
      ↓
ТОЧКИ ВХОДА в семантическую сеть
      ↓
  [PPR Expansion]
      ↓
ПРОЯВЛЕНИЯ концепции в знаниях
```

---

## 1. Вопрос как семантический аттрактор

### Фундаментальная идея

Вопрос в NodeRAG — это не запрос к индексу, а **семантический аттрактор** в пространстве знаний. Он создаёт **гравитационное поле** в embedding space, притягивающее релевантные проявления концепции.

### Реализация в коде

**Файл**: `NodeRAG/search/search.py:78-104`

```python
def search(self, query: str):
    retrieval = Retrieval(...)

    # 1. ВОПРОС → КЛЮЧ (embedding)
    query_embedding = np.array(
        self.config.embedding_client.request(query),
        dtype=np.float32
    )

    # 2. КЛЮЧ → ТОЧКИ ВХОДА (HNSW search)
    HNSW_results = self.hnsw.search(
        query_embedding,
        HNSW_results=self.config.HNSW_results
    )

    # 3. Decompose query → СТРУКТУРНЫЕ КОМПОНЕНТЫ
    decomposed_entities = self.decompose_query(query)
    accurate_results = self.accurate_search(decomposed_entities)

    # 4. ПЕРСОНАЛИЗАЦИЯ семантического поля
    personalization = {
        ids: self.config.similarity_weight
        for ids in retrieval.HNSW_results
    }
    personalization.update({
        id: self.config.accuracy_weight
        for id in retrieval.accurate_results
    })

    # 5. ТОЧКИ ВХОДА → ПРОЯВЛЕНИЯ (PPR expansion)
    weighted_nodes = self.graph_search(personalization)

    return retrieval
```

### Философский анализ трансформации

#### Стадия 1: **Абстракция → Вектор**

```python
query_embedding = embedding_client.request(query)
```

**Эпистемологическая сущность**:
- Вопрос (естественный язык) преобразуется в **геометрическую точку** в 1536-мерном пространстве
- Каждое измерение представляет **скрытую семантическую характеристику**
- Близость в этом пространстве = **концептуальная родственность**

**Философская параллель**: Платоновская идея vs. её проекция в чувственный мир. Embedding — это **проекция абстрактного вопроса** в измеримое пространство.

#### Стадия 2: **Вектор → Точки входа**

```python
HNSW_results = self.hnsw.search(query_embedding, HNSW_results=50)
```

**Эпистемологическая сущность**:
- HNSW поиск находит **ближайшие проекции** концепции
- Не точное совпадение, а **семантическая близость**
- Approximate Nearest Neighbor = **аппроксимация идеи через проявления**

**Философская параллель**: Герменевтический круг — понимание целого через части. HNSW results — это **первичные точки понимания**, через которые раскрывается смысл вопроса.

#### Стадия 3: **Декомпозиция вопроса**

```python
decomposed_entities = self.decompose_query(query)
```

**Файл**: `NodeRAG/utils/prompt/decompose.py`

**Промпт**:
```
Break down the query into a single list. Each item should be a main entity.
If you have high confidence → include closely related terms.
If uncertain → only extract entities.
```

**Эпистемологическая сущность**:
- Вопрос деконструируется на **атомарные концептуальные единицы**
- LLM выполняет **семантическую декомпозицию**, не синтаксическую
- Результат: список **концептуальных ключей**

**Философская параллель**: Аналитическая философия — разложение сложной проблемы на простые составляющие. Каждая entity — это **атом смысла**.

**Пример**:

```
Query: "What research did Dr. Emily Roberts present about renewable energy?"

Decomposed:
{
  "elements": [
    "DR. EMILY ROBERTS",
    "research",
    "renewable energy",
    "presentation"
  ]
}
```

#### Стадия 4: **Точное совпадение** (Accurate Search)

```python
accurate_results = self.accurate_search(decomposed_entities)
```

**Файл**: `NodeRAG/search/search.py:113-124`

```python
def accurate_search(self, entities: List[str]) -> List[str]:
    accurate_results = []

    for entity in entities:
        words = entity.lower().split()
        # Regex pattern для ТОЧНОГО совпадения фразы
        pattern = re.compile(r'\b' + r'\s+'.join(map(re.escape, words)) + r'\b')
        result = [id for id, text in self.accurate_id_to_text.items()
                  if pattern.search(text.lower())]
        if result:
            accurate_results.extend(result)

    return accurate_results
```

**Эпистемологическая сущность**:
- Дополнение **аппроксимативного поиска** (HNSW) **точным совпадением**
- Особенно важно для **собственных имен** и **специфических терминов**
- "DR. EMILY ROBERTS" найдется точно, даже если embedding не совсем близок

**Философская параллель**: Комбинация **интуиции** (HNSW, семантическая близость) и **логики** (accurate search, точное соответствие).

---

## 2. Персонализация семантического поля

### Концептуальная идея: Weighted Personalization

После того как найдены **точки входа**, система создаёт **персонализированное семантическое поле** вокруг этих точек.

**Файл**: `NodeRAG/search/search.py:96-99`

```python
personalization = {
    ids: self.config.similarity_weight
    for ids in retrieval.HNSW_results
}
personalization.update({
    id: self.config.accuracy_weight
    for id in retrieval.accurate_results
})
```

### Философский анализ

**Два типа знания**:

1. **Аналоговое знание** (`similarity_weight`):
   - Найдено через **семантическую близость**
   - Может быть **метафорически связано**
   - Более широкий контекст

2. **Символическое знание** (`accuracy_weight`):
   - Найдено через **точное совпадение**
   - **Буквально упоминает** сущности из вопроса
   - Более узкий, но точный контекст

**Веса как "гравитационная масса"**:
- `similarity_weight` (например, 1.0) = стандартная масса
- `accuracy_weight` (например, 2.0) = двойная масса

**Эффект**: Узлы с точным совпадением имеют **большее влияние** на последующую expansion.

---

## 3. Personalized PageRank: Распространение семантики

### Концептуальная модель

После создания персонализированного поля, система использует **Personalized PageRank** для **распространения семантического влияния** по графу.

**Файл**: `NodeRAG/search/search.py:172-177`

```python
def graph_search(self, personalization: Dict[str, float]) -> List[str]:
    page_rank_scores = self.sparse_PPR.PPR(
        personalization,
        alpha=self.config.ppr_alpha,
        max_iter=self.config.ppr_max_iter
    )
    return [id for id, score in page_rank_scores]
```

### Математическая формула

**Файл**: `NodeRAG/utils/PPR.py:38-57`

```python
def PPR(self, personalization: dict[str, float], alpha=0.85, max_iter=100, epsilons=1e-5):
    # Инициализация вероятностного распределения
    probs = np.zeros(len(self.nodes))
    for node, prob in personalization.items():
        probs[self.nodes.index(node)] = prob
    probs = probs / np.sum(probs)  # Нормализация

    # Power iteration
    for i in range(max_iter):
        probs_old = probs.copy()

        # PPR формула:
        # probs = alpha * M * probs + (1-alpha) * personalization
        probs = alpha * self.trans_matrix.dot(probs) + (1 - alpha) * probs

        # Проверка сходимости
        if np.linalg.norm(probs - probs_old) < epsilons:
            break

    return sorted(zip(self.nodes, probs), key=itemgetter(1), reverse=True)
```

### Философская интерпретация формулы

#### Формула PPR

```
P(t+1) = α · M · P(t) + (1-α) · P₀

где:
- P(t) = вероятностное распределение на шаге t
- M = матрица переходов (граф)
- P₀ = персонализированное распределение (query)
- α = damping factor (обычно 0.85)
```

#### Философская сущность компонентов

1. **`α · M · P(t)`** — **Распространение по структуре знаний**
   - **`M`** = структура связей между концепциями
   - **`M · P(t)`** = "блуждание" по графу знаний
   - **Философия**: Знания **связаны** через отношения. Релевантность **распространяется** по этим связям.

2. **`(1-α) · P₀`** — **Возврат к вопросу**
   - **`P₀`** = исходный вопрос (точки входа)
   - **`(1-α)`** = вероятность "телепортации" обратно к вопросу
   - **Философия**: Expansion не должна **отклоняться** слишком далеко от исходного вопроса. Это **якорь семантики**.

3. **`α = 0.85`** — **Баланс exploration/exploitation**
   - 85% времени: **следуем связям** (exploration)
   - 15% времени: **возвращаемся к вопросу** (exploitation)
   - **Философия**: Баланс между **расширением контекста** и **сохранением релевантности**.

### Визуализация процесса

```
Итерация 0:
Вопрос → [Точки входа имеют вероятность 1.0, остальные 0]

Итерация 1:
[Точки входа] → распространяют вероятность на соседей
              ↓
    [Соседние узлы получают часть вероятности]

Итерация 2:
[Соседние узлы] → распространяют дальше
                ↓
      [Соседи соседей получают вероятность]

При этом на каждой итерации:
- 85% вероятности распространяется дальше
- 15% возвращается к точкам входа

Результат через ~10-20 итераций:
Узлы ранжированы по "семантической близости" к вопросу через структуру графа
```

### Эпистемологическая интерпретация

**PPR как "семантическая диффузия"**:

Представьте, что вопрос — это **источник семантической энергии**. Эта энергия:
1. **Инжектируется** в точки входа (HNSW + accurate results)
2. **Диффундирует** по графу через рёбра (отношения)
3. **Ослабевает** с расстоянием от источника
4. **Частично возвращается** к источнику (damping)

**Результат**: Узлы с **высоким PPR score** — это те, которые:
- **Структурно близки** к точкам входа
- **Часто встречаются** на путях от точек входа
- **Семантически связаны** с вопросом через граф

---

## 4. Концепция и проявление: Онтологическая структура

### Фундаментальная дихотомия

В NodeRAG реализована философская дихотомия **концепция vs. проявление**:

| Концепция | Проявление |
|-----------|-----------|
| Вопрос (абстракция) | Семантические узлы |
| Идея | Факты |
| Универсалия | Партикулярии |
| Смысл | Референт |

### Как вопрос связывает концепцию и проявление

#### Процесс "заземления" абстракции

```
КОНЦЕПЦИЯ (вопрос)
        ↓
    [Embedding]
        ↓
ГЕОМЕТРИЧЕСКАЯ ПРОЕКЦИЯ
        ↓
    [HNSW Search]
        ↓
БЛИЖАЙШИЕ ПРОЯВЛЕНИЯ в знаниях
        ↓
    [PPR Expansion]
        ↓
ВСЕ РЕЛЕВАНТНЫЕ ПРОЯВЛЕНИЯ через структуру
        ↓
    [Context Assembly]
        ↓
УПОРЯДОЧЕННЫЕ ПРОЯВЛЕНИЯ концепции
        ↓
    [LLM Synthesis]
        ↓
ОТВЕТ (синтез проявлений → новая абстракция)
```

### Философская параллель: Феноменология Гуссерля

**Интенциональность сознания**:
- Вопрос = **интенциональный акт**, направленный на объект
- Embedding = **ноэма** (смысл акта)
- HNSW results = **первичные явления** (Erscheinungen)
- PPR expansion = **горизонт явления** (все возможные аспекты)
- Context assembly = **конституирование объекта** из явлений

---

## 5. Трансформация: вопрос → ключ → проявления → ответ

### Полная онтологическая цепочка

#### 1. Вопрос как интенциональный акт

```python
query = "What research did Dr. Emily Roberts present?"
```

**Онтологический статус**: Абстрактная интенция, не имеющая проявления в мире знаний.

#### 2. Ключ как геометрическая проекция интенции

```python
query_embedding = [0.23, -0.45, 0.78, ..., 0.12]  # 1536 dimensions
```

**Онтологический статус**: Математическая проекция смысла. Не концепция и не проявление, а **медиатор** между ними.

#### 3. Точки входа как первичные проявления

```python
HNSW_results = [
    "SU-12345",  # "Dr. Emily Roberts presented research on solar panels"
    "SU-67890",  # "Renewable energy conference in Paris, September 2024"
    "ATTR-456",  # "Dr. Emily Roberts is a leading researcher in renewable energy"
]
```

**Онтологический статус**: Конкретные факты, **частичные проявления** концепции.

#### 4. Расширенный контекст как полный горизонт проявлений

```python
PPR_results = [
    "SU-12345",  # (точка входа)
    "ENTITY-789",  # "DR. EMILY ROBERTS"
    "ATTR-456",  # Attribute об исследователе
    "REL-123",  # "DR. EMILY ROBERTS, presented at, CONFERENCE"
    "HLE-999",  # "Advances in Solar Energy Technology" (high-level theme)
    "SU-11111",  # Semantic unit о другом исследовании в той же области
]
```

**Онтологический статус**: **Полный горизонт** проявлений концепции через структуру знаний.

#### 5. Структурированный контекст как упорядоченные проявления

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

**Результат**:

```
------------entity-------------
1. DR. EMILY ROBERTS
2. RENEWABLE ENERGY
3. SOLAR PANELS

------------high_level_element-------------
1. Advances in Solar Energy Technology
2. International Renewable Energy Research

------------relationship-------------
1. DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY
2. DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE

------------semantic_unit-------------
1. Dr. Emily Roberts presented her research on solar panel efficiency
2. The conference focused on renewable energy innovations
3. Several European companies showed interest in collaboration
```

**Онтологический статус**: Проявления **организованы по типам**, создавая **структурированную феноменологию**.

#### 6. Ответ как синтез проявлений → новая абстракция

```python
answer = """
Dr. Emily Roberts presented research on solar panel efficiency improvements
at the International Conference on Renewable Energy in Paris in September 2024.
Her work focused on innovative approaches to increase solar energy conversion
rates and attracted significant interest from European companies for potential
partnerships.
"""
```

**Онтологический статус**: **Синтез** конкретных проявлений обратно в **абстрактное знание**. Ответ — это **новая концепция**, синтезированная из проявлений.

---

## 6. Эпистемологические уровни трансформации

### Таблица уровней

| Уровень | Онтологический статус | Эпистемический статус | Реализация |
|---------|----------------------|----------------------|------------|
| **0. Вопрос** | Абстрактная интенция | Неудовлетворённое знание | `query` string |
| **1. Ключ** | Геометрическая проекция | Потенциальное знание | `query_embedding` vector |
| **2. Точки входа** | Первичные проявления | Актуальное но частичное знание | HNSW + accurate results |
| **3. Семантическое поле** | Взвешенные проявления | Приоритизированное знание | `personalization` dict |
| **4. Горизонт проявлений** | Полные проявления через структуру | Структурное знание | PPR results |
| **5. Упорядоченный контекст** | Категоризованные проявления | Организованное знание | `structured_prompt` |
| **6. Ответ** | Синтезированная абстракция | Удовлетворённое знание | LLM response |

### Философское значение каждого уровня

#### Уровень 0 → 1: Абстракция → Математика

**Философская проблема**: Как измерить смысл?

**Решение NodeRAG**: Embedding function как **гомоморфизм** из пространства смыслов в векторное пространство.

**Ключевое свойство**: Сохранение семантической близости.
```
distance(embedding(A), embedding(B)) ∝ semantic_similarity(A, B)
```

#### Уровень 1 → 2: Математика → Конкретные проявления

**Философская проблема**: Как найти проявления абстрактной идеи?

**Решение NodeRAG**: Approximate Nearest Neighbor в embedding space.

**Ключевое свойство**: Аппроксимация идеального поиска с субквадратичной сложностью.

#### Уровень 2 → 4: Частичные проявления → Полный горизонт

**Философская проблема**: Как найти **все релевантные** проявления, не только прямо упомянутые?

**Решение NodeRAG**: Personalized PageRank — распространение релевантности по структуре знаний.

**Ключевое свойство**: Использование **отношений** (relationships) как путей семантической связности.

#### Уровень 4 → 6: Проявления → Синтез

**Философская проблема**: Как из множества фактов создать связный ответ?

**Решение NodeRAG**: LLM synthesis с structured context.

**Ключевое свойство**: Диалектический синтез — **thesis** (вопрос) + **antithesis** (факты) → **synthesis** (ответ).

---

## 7. Метафизика вопроса-ключа

### Вопрос как "отмычка" к знаниям

В классической метафоре "ключ-замок":
- **Ключ** = query embedding
- **Замок** = knowledge graph
- **Механизм** = HNSW + PPR

Но в NodeRAG это **не жёсткое соответствие**:

#### Традиционный поиск (SQL):
```sql
SELECT * FROM knowledge WHERE entity = 'DR. EMILY ROBERTS'
```
**Метафора**: Ключ подходит **только к одному замку**.

#### NodeRAG поиск:
```python
embedding = embed(query)
HNSW_results = approximate_nearest(embedding, top_k=50)
PPR_results = personalized_pagerank(HNSW_results, graph)
```
**Метафора**: Ключ **резонирует** с множеством замков, открывая **целую область** знаний.

### Резонанс вместо совпадения

**Философская идея**: Вопрос не **совпадает** с ответом, а **резонирует** с проявлениями.

**Аналогия**:
- Камертон (440 Hz) → вызывает резонанс в струнах с близкой частотой
- Вопрос (embedding) → вызывает "резонанс" в семантически близких узлах

**Реализация**:
- Cosine similarity = мера "резонанса"
- HNSW = быстрое нахождение "резонирующих" узлов
- PPR = распространение "резонанса" по структуре

---

## 8. Заключение: вопрос как онтологический мост

### Фундаментальная роль вопроса

В NodeRAG вопрос выполняет **онтологически-эпистемическую** функцию:

1. **Онтологически**: Вопрос **актуализирует** потенциальные связи в графе знаний
   - До вопроса: граф — это потенциальная структура
   - После вопроса: активируется **подграф релевантных проявлений**

2. **Эпистемически**: Вопрос **направляет** процесс познания
   - Задаёт **интенцию** (что ищем?)
   - Определяет **критерий релевантности** (через embedding)
   - Ограничивает **горизонт поиска** (через personalization)

### Концепция → Проявление: полный цикл

```
КОНЦЕПЦИЯ (вопрос)
        ↓
    [математическая проекция]
        ↓
ГЕОМЕТРИЧЕСКИЙ КЛЮЧ (embedding)
        ↓
    [приближённый поиск]
        ↓
ПЕРВИЧНЫЕ ПРОЯВЛЕНИЯ (HNSW results)
        ↓
    [структурное распространение]
        ↓
ПОЛНЫЕ ПРОЯВЛЕНИЯ (PPR results)
        ↓
    [категоризация]
        ↓
УПОРЯДОЧЕННЫЕ ПРОЯВЛЕНИЯ (structured context)
        ↓
    [LLM синтез]
        ↓
НОВАЯ КОНЦЕПЦИЯ (answer)
```

### Философское значение

NodeRAG реализует **герменевтический круг**:
- **Вопрос** (предпонимание) → определяет что искать
- **Проявления** (тексты) → дают частичное понимание
- **Синтез** (ответ) → новое понимание, которое может породить **новые вопросы**

Это **диалектический процесс познания**:
- Thesis: Вопрос (незнание)
- Antithesis: Факты (знание)
- Synthesis: Ответ (новое знание + новое незнание)

---

## Приложение: Ключевые файлы

### Реализация парадигмы

| Компонент | Файл | Строки |
|-----------|------|--------|
| **Question → Key** | `NodeRAG/search/search.py` | 84-85 |
| **Key → Entry points** | `NodeRAG/search/search.py` | 85-86 |
| **Query decomposition** | `NodeRAG/search/search.py` | 91-94 |
| **Personalization** | `NodeRAG/search/search.py` | 96-99 |
| **PPR expansion** | `NodeRAG/utils/PPR.py` | 38-57 |
| **Context structure** | `NodeRAG/search/Answer_base.py` | 65-76 |
| **LLM synthesis** | `NodeRAG/search/search.py` | 139-140 |

### Промпты

| Трансформация | Файл |
|---------------|------|
| Query decomposition | `spec/prompts/query-decomposition-prompt.md` |
| Answer generation | `spec/prompts/answer-generation-prompt.md` |
