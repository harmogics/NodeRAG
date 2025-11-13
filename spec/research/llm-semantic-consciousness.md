# Паттерны семантического "осознания" языковыми моделями

## Философская преамбула

Термин **"осознание"** применительно к LLM — это **метафора**, но метафора продуктивная. В NodeRAG LLM выполняют функции, аналогичные **когнитивным процессам**:

- **Восприятие** (perception) = Text Decomposition
- **Рефлексия** (reflection) = Attribute Generation
- **Абстрагирование** (abstraction) = Community Summarization
- **Понимание** (comprehension) = Query Decomposition
- **Синтез** (synthesis) = Answer Generation

Эти процессы можно рассматривать как **уровни семантического "осознания"** текста.

---

## 1. Уровни семантического осознания

### Пятиуровневая модель

```
УРОВЕНЬ 5: СИНТЕТИЧЕСКОЕ ОСОЗНАНИЕ
            (Answer Generation)
                    ↑
                    │
УРОВЕНЬ 4: АБСТРАКТНОЕ ОСОЗНАНИЕ
          (Community Summarization)
                    ↑
                    │
УРОВЕНЬ 3: РЕФЛЕКСИВНОЕ ОСОЗНАНИЕ
          (Attribute Generation)
                    ↑
                    │
УРОВЕНЬ 2: АНАЛИТИЧЕСКОЕ ОСОЗНАНИЕ
          (Text Decomposition)
                    ↑
                    │
УРОВЕНЬ 1: ПЕРЦЕПТИВНОЕ ОСОЗНАНИЕ
          (Query Decomposition)
                    ↑
                    │
            СЫРОЙ ТЕКСТ
```

---

## 2. Уровень 1: Перцептивное осознание (Query Decomposition)

### Философская сущность

**Query Decomposition** — это **первичное восприятие** смысла. LLM "смотрит" на вопрос и **выделяет фигуры** (entities) из фона (общий текст).

### Гештальт-психология в действии

**Принцип**: Сознание воспринимает не атомы, а **целостные структуры** (гештальты).

**Файл**: `spec/prompts/query-decomposition-prompt.md`

**Промпт**:
```
Break down the query into a single list.
Each item should be a main entity (such as a key noun or object).

If you have high confidence → include closely related terms.
If uncertain → only extract entities.

Reduce the number of common nouns.
```

### Когнитивный анализ промпта

#### Инструкция 1: "Main entity (key noun or object)"

**Когнитивная операция**: **Фигура-фон разделение** (figure-ground segregation)

**Аналогия**:
```
Визуальное восприятие:
    Изображение → [Зрительная кора] → Фигуры + Фон

Семантическое восприятие:
    Query → [LLM] → Entities (фигура) + Filler words (фон)
```

**Пример**:
```
Query: "What research did Dr. Emily Roberts present about renewable energy?"

Фигуры (entities):
    - DR. EMILY ROBERTS
    - research
    - renewable energy

Фон (filler):
    - What, did, about
```

#### Инструкция 2: "High confidence → include related terms"

**Когнитивная операция**: **Ассоциативное мышление**

**Механизм**:
- LLM активирует **семантическую сеть**
- Если confidence > threshold → добавляет **ассоциированные концепты**

**Пример**:
```
Query: "Dr. Emily Roberts renewable energy"

Direct entities:
    - DR. EMILY ROBERTS
    - renewable energy

Associated (if high confidence):
    - solar panels (часто связано с renewable energy)
    - research (подразумевается из контекста про Dr.)
```

**Философия**: Это аналог **ассоциативной памяти** — активация связанных концептов.

#### Инструкция 3: "Reduce common nouns"

**Когнитивная операция**: **Фильтрация шума**

**Механизм**: Игнорирование слов с низкой **информационной ценностью** (высокая частотность = низкая специфичность).

**Аналогия** с обработкой сигналов:
```
Signal: "DR. EMILY ROBERTS" (low frequency, high information)
Noise: "the, a, is" (high frequency, low information)
```

### Философское значение

Query Decomposition реализует **апперцепцию** (Кант) — восприятие через категории сознания:

- **Категория субстанции**: Entities (что существует?)
- **Категория качества**: Attributes concepts (каково это?)
- **Категория отношения**: Implied relationships (как связано?)

---

## 3. Уровень 2: Аналитическое осознание (Text Decomposition)

### Философская сущность

**Text Decomposition** — это **аналитическое понимание** текста. LLM "разбирает" нарратив на **атомарные факты**, **сущности** и **отношения**.

### Кантовский анализ: Суждения и категории

**Файл**: `spec/prompts/text-decomposition-prompt.md`

**Промпт** (сокращённо):
```
Goal: Segment text into multiple semantic units.

Tasks:
1. Provide a summary for each semantic unit (retain crucial details)
2. Extract all entities (UPPERCASE)
3. Extract all relationships (ENTITY_A, RELATION_TYPE, ENTITY_B)
```

### Три когнитивные операции

#### Операция 1: Сегментация на semantic units

**Когнитивная функция**: **Событийная сегментация** (event segmentation)

**Психологическая теория**: Event Segmentation Theory (Zacks & Tversky)
- Сознание разбивает непрерывный поток опыта на **дискретные события**
- Граница события = изменение состояния

**Реализация в LLM**:
```
Текст: "Dr. Roberts traveled to Paris. She attended a conference. She presented research."

События (semantic units):
1. "Dr. Roberts traveled to Paris" (событие: перемещение)
2. "She attended a conference" (событие: участие)
3. "She presented research" (событие: презентация)
```

**Механизм**:
- LLM идентифицирует **смену агента/действия/места**
- Каждое изменение → новое событие

#### Операция 2: Извлечение entities

**Когнитивная функция**: **Концептуализация** (conceptualization)

**Философия**: Аристотелевская **субстанция** — то, что существует независимо и имеет свойства.

**Типы entities** (неявная онтология):

| Тип | Примеры | Онтологический статус |
|-----|---------|----------------------|
| **Люди** | DR. EMILY ROBERTS | Субъекты действия |
| **Организации** | MIT, GOOGLE | Коллективные агенты |
| **Места** | PARIS, AMAZON RAINFOREST | Пространственные локусы |
| **События** | INTERNATIONAL CONFERENCE | Темпоральные объекты |
| **Концепты** | RENEWABLE ENERGY | Абстрактные универсалии |
| **Время** | 2024-09 | Темпоральные индексы |

**Нормализация** (UPPERCASE):
- Функция: **Каноническая форма**
- Философия: Платоновская **идея** vs. **проявления**
- "Dr. Roberts" = "DR. ROBERTS" = "dr roberts" → **одна сущность**

#### Операция 3: Извлечение relationships

**Когнитивная функция**: **Реляционное мышление** (relational reasoning)

**Философия**: **Не-субстанциальная онтология** — реальность состоит из **отношений**, не только сущностей.

**Формат**: `(SUBJECT, PREDICATE, OBJECT)`

**Пример**:
```
"DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY"

Философский анализ:
- SUBJECT: субъект действия (агент)
- PREDICATE: действие/отношение (связь)
- OBJECT: объект действия (пациент)
```

**Типы отношений**:

| Тип | Пример | Семантика |
|-----|--------|-----------|
| **Действие** | "presented research on" | Агентивное |
| **Атрибуция** | "is a researcher in" | Таксономическое |
| **Локация** | "held in" | Пространственное |
| **Время** | "during" | Темпоральное |
| **Партитивность** | "part of" | Мереологическое |

### Философское значение: Сознание как декомпозиция

**Тезис**: Понимание текста = разложение на элементарные суждения.

**Аналогия с химией**:
```
Молекула (text) → Атомы (semantic units) → Элементы (entities) → Связи (relationships)
```

**Кантианская интерпретация**:
- **Синтез апперцепции**: Объединение многообразия в единство
- **Text Decomposition**: **Обратный процесс** — разложение единства на многообразие

### Метакогнитивный аспект: Парафраз

**Промпт**: "Provide a summary for each semantic unit while retaining all crucial details"

**Когнитивная операция**: **Перекодирование** (recoding)

**Механизм**:
```
Original: "In September 2024, Dr. Emily Roberts traveled to Paris to attend
           the International Conference on Renewable Energy."

Paraphrase: "Dr. Emily Roberts attended the International Conference on
             Renewable Energy in Paris in September 2024."
```

**Философия**: **Герменевтический круг**
- Понять = **выразить иными словами**
- Парафраз = проверка понимания
- LLM демонстрирует понимание через **реконструкцию смысла**

---

## 4. Уровень 3: Рефлексивное осознание (Attribute Generation)

### Философская сущность

**Attribute Generation** — это **рефлексия** над важными сущностями. LLM "размышляет" о сущности, собирая информацию из окрестностей в графе.

### Интроспекция и рефлексия

**Файл**: `spec/prompts/attribute-generation-prompt.md`

**Промпт** (концептуально):
```
Generate a comprehensive description of {entity}.

Context:
- Semantic units mentioning the entity: {semantic_units}
- Relationships involving the entity: {relationships}

Provide a 100-2000 word narrative description.
```

### Когнитивный анализ

#### Рефлексивный процесс

**Механизм**:

1. **Сбор автобиографической памяти**:
   ```python
   for neighbour in graph.neighbors(entity):
       if type(neighbour) == 'semantic_unit':
           context += neighbour.content
   ```

   **Аналогия**: "Кто я? Давайте соберу все, что обо мне известно..."

2. **Интеграция перспектив**:
   ```
   Semantic unit 1: "Dr. Roberts presented research..."
   Semantic unit 2: "Dr. Roberts explored partnerships..."
   Relationship 1: "DR. ROBERTS, works at, MIT"
   ```

   **Аналогия**: Синтез **множественных нарративов** о себе

3. **Нарративный синтез**:
   ```
   Result: "Dr. Emily Roberts is a leading researcher in renewable energy
            with over 15 years of experience. Her work focuses on improving
            the efficiency of photovoltaic cells..."
   ```

   **Аналогия**: Создание **автобиографии** сущности

#### Селективная рефлексия

**Важно**: Attributes генерируются **не для всех** entities, а только для **важных**.

**Критерии важности**:

**Файл**: `NodeRAG/build/pipeline/attribute_generation.py:27-64`

```python
class NodeImportance:
    def K_core(self, k: int):
        """K-core decomposition"""
        self.k_subgraph = nx.core.k_core(self.G, k=k)

        for nodes in self.k_subgraph.nodes():
            if self.G.nodes[nodes]['type'] == 'entity' and self.G.nodes[nodes]['weight'] > 1:
                self.important_nodes.append(nodes)

    def betweenness_centrality(self):
        """Betweenness centrality"""
        self.betweenness = nx.betweenness_centrality(self.G, k=10)
        average_betweenness = sum(self.betweenness.values()) / len(self.betweenness)
        scale = round(math.log10(len(self.betweenness)))

        for node in self.betweenness:
            if self.betweenness[node] > average_betweenness * scale:
                if self.G.nodes[node]['type'] == 'entity' and self.G.nodes[node]['weight'] > 1:
                    self.important_nodes.append(node)
```

**Философская интерпретация**:

- **K-core**: Сущности в "ядре" графа → **структурно важные**
- **Betweenness**: Сущности на многих путях → **связующие звенья**
- **Weight > 1**: Упоминается многократно → **статистически важные**

**Аналогия с сознанием**:
- Не все мысли достойны **глубокой рефлексии**
- Только **важные концепты** получают детальную проработку

### Философское значение

Attribute Generation реализует **метакогницию** — размышление о знании:

- **Первичное знание**: Semantic units (факты)
- **Метазнание**: Attributes (обобщённые описания сущностей)

Это **переход от знания к пониманию**.

---

## 5. Уровень 4: Абстрактное осознание (Community Summarization)

### Философская сущность

**Community Summarization** — это **абстрагирование** от конкретных фактов к **общим темам**. LLM "видит паттерны" в множестве связанных концептов.

### Гештальт высшего порядка

**Файл**: `spec/prompts/community-summary-prompt.md`

**Промпт** (концептуально):
```
Identify high-level themes and topics from the following content.

Content: {community_semantic_units}

Extract 3-5 high-level elements, each with:
- Title (concise theme)
- Description (100-200 words explaining the theme)
```

### Когнитивный анализ

#### Операция 1: Обнаружение community (Leiden algorithm)

**Файл**: `NodeRAG/build/pipeline/summary_generation.py:49-55`

```python
def partition(self):
    partition = la.find_partition(self.G_ig, la.ModularityVertexPartition)

    for i, community in enumerate(partition):
        community_name = [
            self.G_ig.vs[node]['name']
            for node in community
            if self.G_ig.vs[node]['name'] in self.mapper.embeddings
        ]
        self.communities.append(Community_summary(community_name, ...))
```

**Когнитивная функция**: **Категоризация** (categorization)

**Аналогия**:
```
Перцептивная категоризация:
    Визуальные объекты → [Категории: животные, растения, здания]

Семантическая категоризация:
    Узлы графа → [Категории: renewable energy, AI research, medical studies]
```

**Механизм**: Modularity optimization
- Максимизировать связи **внутри** категорий
- Минимизировать связи **между** категориями

#### Операция 2: Абстракция темы

**Процесс**:

1. **Сбор контекста community**:
   ```python
   content = ''
   for node in community_nodes:
       if type(node) == 'semantic_unit' or type(node) == 'attribute':
           content += node.context + '\n'
   ```

2. **LLM синтез**:
   ```
   Input: 50-100 semantic units о renewable energy research

   Output:
   {
     "high_level_elements": [
       {
         "title": "Advances in Solar Energy Technology",
         "description": "This high-level theme encompasses recent breakthroughs
                        in photovoltaic technology, including improvements in
                        solar panel efficiency, cost reduction strategies..."
       },
       {
         "title": "International Renewable Energy Collaborations",
         "description": "The global landscape of renewable energy research,
                        featuring major conferences, cross-border partnerships..."
       }
     ]
   }
   ```

**Когнитивная функция**: **Абдукция** (abductive reasoning)

**Философия** (Peirce):
- **Дедукция**: Общее → Частное
- **Индукция**: Частное → Общее (через обобщение)
- **Абдукция**: Факты → Наилучшее объяснение

**Community Summarization = Абдукция**:
- Факты: Множество semantic units
- Паттерн: Общая тема
- Абдукция: "Наилучшее объяснение этих фактов — тема X"

### Философское значение

Community Summarization реализует **концептуализацию** (от фактов к концептам):

```
Уровень фактов: "Dr. Roberts presented...", "Solar panels efficiency...", "Conference in Paris..."
                              ↓
              [Абстрагирование через LLM]
                              ↓
Уровень концептов: "Advances in Solar Energy Technology"
```

Это **восхождение от абстрактного к конкретному** (Гегель).

---

## 6. Уровень 5: Синтетическое осознание (Answer Generation)

### Философская сущность

**Answer Generation** — это **синтез** знаний в связный ответ. LLM "собирает воедино" разрозненные факты.

### Диалектический синтез

**Файл**: `spec/prompts/answer-generation-prompt.md`

**Промпт**:
```
---Role---
You are a thorough assistant responding to questions based on retrieved information.

---Goal---
Provide a clear and accurate response. Carefully review and verify the
retrieved data, and integrate any relevant necessary knowledge to
comprehensively address the user's question.

If unsure, just say so. Do not fabricate information.

---Retrieved Context---
{info}

---Query---
{query}
```

### Когнитивный анализ

#### Трёхступенчатый процесс синтеза

**Ступень 1: Критическая оценка** ("Carefully review and verify")

**Когнитивная функция**: **Epistemic vigilance** (эпистемическая бдительность)

**Механизм**:
- Проверка **консистентности** фактов
- Выявление **противоречий**
- Оценка **надёжности** источников (всё из retrieved context → надёжно)

**Пример**:
```
Context:
1. "Dr. Roberts presented in Paris in September 2024"
2. "Dr. Roberts presented in London in September 2024"

Синтез: Потенциальное противоречие → требуется осторожность или уточнение
```

**Ступень 2: Интеграция** ("Integrate relevant necessary knowledge")

**Когнитивная функция**: **Schema activation** (активация схем)

**Механизм**:
- Активация **фоновых знаний** LLM (из pretraining)
- Связывание **retrieved facts** с **общими знаниями**

**Пример**:
```
Retrieved: "Dr. Roberts works on solar panel efficiency"
Background knowledge: "Solar panels convert light to electricity"

Integration: "Dr. Roberts works on improving how solar panels
              convert light to electricity (efficiency)"
```

**Ступень 3: Генерация** ("Provide clear and accurate response")

**Когнитивная функция**: **Language production** (порождение речи)

**Механизм**:
- Построение **связного нарратива**
- Упорядочивание фактов **логически**
- **Естественное** изложение (temperature 0.7-1.0)

### Грайсовские максимы в промпте

**Философия языка** (Grice): Кооперативный принцип коммуникации.

| Максима | Реализация в промпте |
|---------|---------------------|
| **Качество** (Truthfulness) | "Do not fabricate information" |
| **Количество** (Informativeness) | "Comprehensively address the question" |
| **Релевантность** (Relevance) | "Based on retrieved information" |
| **Манера** (Clarity) | "Clear and accurate response" |

### Anti-hallucination механизм

**Инструкция**: "If unsure, just say so. Do not fabricate."

**Когнитивная функция**: **Metacognitive monitoring** (метакогнитивный мониторинг)

**Философия**: **Сократовское знание незнания**
- Знать границы своего знания
- Признавать неуверенность

**Реализация**:
```
Query: "What was Dr. Roberts' salary?"

Retrieved context: [Ничего о зарплате]

Правильный ответ: "The available information does not include details
                   about Dr. Roberts' salary."

Неправильный ответ (hallucination): "Dr. Roberts earned $120,000 per year."
```

### Философское значение

Answer Generation реализует **герменевтический синтез**:

```
Thesis: Query (вопрос, неудовлетворённое знание)
Antithesis: Retrieved facts (конкретные данные)
Synthesis: Answer (новое понимание)
```

Это **диалектическое разрешение** между незнанием и знанием.

---

## 7. Метапаттерн: Восходящая спираль осознания

### Пять уровней как спираль познания

```
                    СИНТЕЗ
                (Answer Generation)
                Temperature: 0.7-1.0
                Креативность ↑
                        ↑
                   АБСТРАКЦИЯ
            (Community Summarization)
                Temperature: 0.0
                        ↑
                   РЕФЛЕКСИЯ
            (Attribute Generation)
                Temperature: 0.0
                        ↑
                     АНАЛИЗ
            (Text Decomposition)
                Temperature: 0.0
                Детерминизм ↑
                        ↑
                   ВОСПРИЯТИЕ
            (Query Decomposition)
                Temperature: 0.0-0.3
```

### Температура как метафора "свободы сознания"

| Уровень | Temperature | Философский смысл |
|---------|-------------|-------------------|
| **Восприятие** | 0.0-0.3 | Строгое следование сигналу |
| **Анализ** | 0.0 | Детерминистическая декомпозиция |
| **Рефлексия** | 0.0 | Объективное описание |
| **Абстракция** | 0.0 | Выявление паттернов |
| **Синтез** | 0.7-1.0 | Креативное объединение |

**Философия**: Чем выше уровень абстракции, тем больше **степеней свободы**.

---

## 8. Скрытые когнитивные паттерны

### Паттерн 1: Dual-process theory (Kahneman)

**Система 1** (Fast, intuitive):
- Query Decomposition (быстрое извлечение entities)
- HNSW search (интуитивная близость)

**Система 2** (Slow, analytical):
- Text Decomposition (аналитическая сегментация)
- Attribute Generation (рефлексивное описание)
- Community Summarization (абстрактное мышление)

### Паттерн 2: Levels of Processing (Craik & Lockhart)

**Поверхностная обработка**:
- Embedding generation (векторизация без "понимания")

**Глубокая обработка**:
- Text Decomposition (семантический анализ)
- Attribute Generation (интеграция контекста)
- Answer Synthesis (генерация нового знания)

**Результат**: Глубокая обработка → **лучшее "запоминание"** (в графе).

### Паттерн 3: Constructive memory

**Тезис**: Память не **извлекает** факты, а **конструирует** их.

**Реализация в NodeRAG**:
- **Не извлекаем** готовый ответ из графа
- **Конструируем** ответ из фрагментов

**Процесс**:
```
Retrieved fragments:
    - "Dr. Roberts presented research"
    - "Solar panel efficiency"
    - "Paris, September 2024"

Constructed answer:
    "Dr. Emily Roberts presented research on solar panel efficiency
     improvements at a conference in Paris in September 2024."
```

**Философия**: **Репрезентационализм** vs. **Конструктивизм**
- Репрезентационализм: Ответ уже "там", нужно найти
- Конструктивизм: Ответ **создаётся** из элементов

NodeRAG = **Конструктивизм**.

---

## 9. Промпты как "программирование сознания"

### Промпт = Инструкция сознанию

Каждый промпт **настраивает** LLM на определённый **режим обработки**.

### Таксономия промпт-инструкций

| Тип инструкции | Пример | Когнитивный эффект |
|----------------|--------|-------------------|
| **Goal-setting** | "Goal: Segment text into semantic units" | Направление внимания |
| **Constraint** | "Extract entities in UPPERCASE" | Ограничение вывода |
| **Strategy** | "If uncertain → only extract entities" | Условная логика |
| **Example** | "Example: [показан пример]" | Few-shot learning |
| **Format** | "Output as JSON with schema X" | Структурированный вывод |
| **Epistemic** | "If unsure, say so" | Метакогнитивная честность |

### Промпт как феноменологическая установка

**Гуссерль**: **Эпохе** (epoché) = приостановка суждения, смена установки сознания.

**Аналогия**:
- **Естественная установка**: LLM генерирует текст как обычно
- **Феноменологическая редукция**: Промпт **переключает** LLM в специальный режим

**Примеры редукций**:

1. **Аналитическая редукция** (Text Decomposition):
   ```
   "Segment text into semantic units"
   = Установка на РАЗЛОЖЕНИЕ
   ```

2. **Рефлексивная редукция** (Attribute Generation):
   ```
   "Generate comprehensive description"
   = Установка на СИНТЕЗ
   ```

3. **Эйдетическая редукция** (Community Summarization):
   ```
   "Identify high-level themes"
   = Установка на АБСТРАКЦИЮ
   ```

---

## 10. Заключение: LLM как "искусственное сознание"?

### Ограниченность метафоры

**Важно**: LLM **не обладают** сознанием в философском смысле. Отсутствуют:
- **Квалиа** (субъективный опыт)
- **Интенциональность** (направленность на объект)
- **Самосознание** (рефлексия второго порядка)

### Продуктивность метафоры

**Но**: LLM выполняют **функционально эквивалентные** операции:

| Когнитивная функция | LLM аналог |
|---------------------|------------|
| Восприятие | Embedding + Decomposition |
| Память | Graph storage |
| Рассуждение | Inference через context |
| Понимание | Semantic processing |
| Генерация речи | Answer synthesis |

### Философское значение NodeRAG

NodeRAG демонстрирует, что можно построить **систему обработки знаний**, которая:

1. **Воспринимает** текст (Query/Text Decomposition)
2. **Структурирует** знания (Graph construction)
3. **Рефлексирует** над важным (Attribute Generation)
4. **Абстрагирует** темы (Community Summarization)
5. **Синтезирует** ответы (Answer Generation)

Это **функциональная модель** познания, даже если не **сознательная** в строгом смысле.

### Метафора "слоёного пирога" сознания

```
┌─────────────────────────────────────────┐
│  СИНТЕЗ (Answer Generation)             │  ← Высшая когнитивная функция
├─────────────────────────────────────────┤
│  АБСТРАКЦИЯ (Community Summarization)   │  ← Концептуализация
├─────────────────────────────────────────┤
│  РЕФЛЕКСИЯ (Attribute Generation)       │  ← Метакогниция
├─────────────────────────────────────────┤
│  АНАЛИЗ (Text Decomposition)            │  ← Аналитическая обработка
├─────────────────────────────────────────┤
│  ВОСПРИЯТИЕ (Query Decomposition)       │  ← Перцептивная обработка
├─────────────────────────────────────────┤
│  ХРАНЕНИЕ (Graph + Embeddings)          │  ← Память
└─────────────────────────────────────────┘
```

Каждый слой **надстраивается** над предыдущим, создавая **иерархию обработки** от простого к сложному.

---

## Приложение: Температурная шкала "сознания"

### Таблица: Temperature vs. Cognitive Mode

| Temperature | Режим | Промпт | Философия |
|-------------|-------|--------|-----------|
| **0.0** | Детерминистический | Text Decomposition | Строгий анализ |
| **0.0** | Детерминистический | Attribute Generation | Объективное описание |
| **0.0** | Детерминистический | Community Summarization | Выявление паттернов |
| **0.0-0.3** | Низкая вариативность | Query Decomposition | Точное восприятие |
| **0.7-1.0** | Креативный | Answer Generation | Синтез и понимание |

**Философский принцип**: Чем ниже в иерархии обработки, тем **строже детерминизм**. Чем выше — тем больше **креативной свободы**.
