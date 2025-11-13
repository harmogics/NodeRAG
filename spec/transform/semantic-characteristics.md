# Семантические характеристики и связи преобразований

## Обзор

Этот документ содержит детальный анализ семантических характеристик всех преобразований в NodeRAG, их взаимосвязи и роли в общей архитектуре системы.

---

## Таксономия семантических преобразований

### По типу операции

| Тип | Описание | Преобразования | Роль |
|-----|----------|----------------|------|
| **Extraction** | Извлечение элементов из текста | Text Decomposition, Query Decomposition | Основа структурирования |
| **Normalization** | Приведение к стандартной форме | Entity normalization (UPPERCASE), Date formatting | Унификация данных |
| **Abstraction** | Повышение уровня абстракции | Community Summarization | Концептуализация |
| **Aggregation** | Объединение множественных элементов | Attribute Generation, Context Assembly | Обогащение контекста |
| **Synthesis** | Создание нового контента из частей | Answer Synthesis, Community Summarization | Генерация знаний |
| **Encoding** | Преобразование в векторное пространство | Embedding Generation | Семантический поиск |
| **Structuring** | Создание связей и структуры | Graph Construction | Организация знаний |
| **Navigation** | Поиск по структуре | HNSW Retrieval, PPR | Доступ к знаниям |
| **Segmentation** | Разбиение на части | Semantic Chunking | Подготовка данных |
| **Clustering** | Группировка похожих элементов | Community Detection | Тематическая организация |

---

## Детальная характеристика по измерениям

### 1. Semantic Chunking

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Низкий | Работает на уровне сырого текста |
| **Семантическая глубина** | Поверхностная | Основана на синтаксических границах |
| **Контекстная зависимость** | Низкая | Не требует понимания содержания |
| **Информационное преобразование** | Сохраняющее | 100% информации сохраняется |
| **Структурная сложность** | Простая | Linear segmentation |
| **Детерминизм** | Полный | Одинаковый результат всегда |
| **Обратимость** | Полная | Можно восстановить исходный документ |
| **Гранулярность** | Средняя | ~1048 токенов |

#### Входные и выходные семантические свойства

**Вход:**
- Тип: Неструктурированный текст
- Размер: Произвольный
- Связность: Высокая (единый документ)
- Семантическая целостность: Переменная

**Выход:**
- Тип: Фрагменты текста
- Размер: ~1048 токенов каждый
- Связность: Средняя (в пределах фрагмента)
- Семантическая целостность: Высокая (сохранены границы)

#### Семантические связи с другими преобразованиями

```
Semantic Chunking
    │
    ├─→ Предоставляет оптимальный контекст для Text Decomposition
    ├─→ Минимизирует потерю семантики на границах
    └─→ Обеспечивает параллельную обработку
```

**Зависимости:**
- **Входные**: Нет (первое преобразование)
- **Выходные**: Text Decomposition требует семантически целостных chunks

**Влияние на качество:**
- Плохое chunking → потеря контекста в Decomposition → неполные facts
- Хорошее chunking → сохранение нарратива → полные semantic units

---

### 2. Text Decomposition ⭐

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Средний | Извлечение структурированных фактов |
| **Семантическая глубина** | Глубокая | Понимание значения, сущностей, отношений |
| **Контекстная зависимость** | Высокая | Требует понимания контекста |
| **Информационное преобразование** | Сжимающее | ~30-50% сжатие через парафраз |
| **Структурная сложность** | Высокая | Multiple output types (units, entities, relations) |
| **Детерминизм** | Высокий | Temperature 0.0 |
| **Обратимость** | Частичная | Возможна потеря деталей |
| **Гранулярность** | Атомарная | Один факт на semantic unit |

#### Входные и выходные семантические свойства

**Вход:**
- Тип: Text Unit (неструктурированный)
- Размер: ~1048 токенов
- Связность: Высокая
- Семантическая плотность: Средняя-высокая

**Выход:**
- **Semantic Units**:
  - Тип: Атомарные факты (парафразированные)
  - Гранулярность: Одно утверждение
  - Связность: Самодостаточные
- **Entities**:
  - Тип: Нормализованные именованные сущности
  - Формат: UPPERCASE
  - Типы: People, Places, Organizations, Concepts, Dates
- **Relationships**:
  - Тип: Семантические связи
  - Формат: {source, relation, target} triplets
  - Направленность: Directed

#### Семантические трансформации внутри процесса

1. **Segmentation**: Text → Atomic statements
   - Разбивает сложные предложения на простые утверждения
   - Сохраняет семантическую независимость каждого

2. **Paraphrasing**: Original text → Concise form
   - Удаляет избыточность
   - Сохраняет критические детали
   - Стандартизирует форму

3. **Entity Extraction**: Text → Named entities
   - Идентифицирует все упоминания
   - Резолвит кореференции (he → JOHN DOE)
   - Нормализует формат

4. **Relationship Extraction**: Text → Triplets
   - Идентифицирует субъект-предикат-объект
   - Извлекает явные и неявные связи
   - Структурирует в формат {A, relation, B}

5. **Temporal Normalization**: "September 2024" → "2024-09"
   - Стандартизирует временные выражения
   - Обеспечивает машинную обработку

#### Семантические связи с другими преобразованиями

```
Text Decomposition
    │
    ├─→ Предоставляет сырьё для Graph Construction
    ├─→ Semantic Units → используются в Context Assembly
    ├─→ Entities → основа для Attribute Generation
    ├─→ Relationships → структурируют граф
    └─→ Все элементы → Embedding Generation
```

**Зависимости:**
- **Входные**: Semantic Chunking (качественные Text Units)
- **Выходные**:
  - Graph Construction (требует entities + relationships)
  - Attribute Generation (требует entities)
  - Embedding Generation (требует semantic units)

**Влияние на качество:**
- **Критическое**: Качество всей системы зависит от точности декомпозиции
- Ошибки в извлечении entity → неправильные связи в графе
- Неточный парафраз → потеря важной информации
- Пропущенные relationships → неполный граф

#### Семантические паттерны

**Паттерн 1: Entity Co-reference Resolution**
```
Input: "Dr. Emily Roberts visited Paris. She presented research."
Output:
  - Entities: ["DR. EMILY ROBERTS", "PARIS"]
  - Semantic Units:
    1. "Dr. Emily Roberts visited Paris."
    2. "Dr. Emily Roberts presented research."  # "She" → "Dr. Emily Roberts"
```

**Паттерн 2: Implicit Relationship Extraction**
```
Input: "John works at Microsoft in Seattle."
Output:
  - Relationships:
    1. "JOHN, works at, MICROSOFT"
    2. "MICROSOFT, located in, SEATTLE"  # Implicit
```

**Паттерн 3: Event Decomposition**
```
Input: "The conference in Paris lasted three days and featured 50 speakers."
Output:
  - Semantic Units:
    1. "The conference took place in Paris."
    2. "The conference lasted three days."
    3. "The conference featured 50 speakers."
```

---

### 3. Graph Construction

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Средний | Создание структурной абстракции |
| **Семантическая глубина** | Средняя | Кодирует связи, но не их значение |
| **Контекстная зависимость** | Низкая | Механическое создание структуры |
| **Информационное преобразование** | Структурирующее | Добавляет топологию |
| **Структурная сложность** | Очень высокая | Гетерогенный мультиграф |
| **Детерминизм** | Полный | Детерминированная процедура |
| **Обратимость** | Полная | Можно извлечь исходные элементы |
| **Гранулярность** | Узловая | Каждый элемент = узел |

#### Входные и выходные семантические свойства

**Вход:**
- Тип: Structured data (semantic units, entities, relationships)
- Формат: JSON/Dictionary
- Связность: Неявная (через упоминания)

**Выход:**
- Тип: Heterogeneous graph
- Узлы: Типизированные (6 типов)
- Рёбра: Семантические связи
- Свойства: Rich attributes per node
- Топология: Power-law distribution (hub entities)

#### Типы узлов и их семантика

| Тип узла | Семантическое значение | Свойства | Роль в графе |
|----------|----------------------|----------|--------------|
| `text_unit` | Исходный фрагмент документа | content, hash_id, doc_id | Корневой узел, связь с документом |
| `semantic_unit` | Атомарный факт | content, hash_id, entities | Базовая единица знаний |
| `entity` | Именованная сущность | name, type, weight | Hub nodes, центры связей |
| `relationship` | Семантическая связь | source, relation, target | Структурные рёбра |
| `attribute` | Описание сущности | content, entity_id | Контекстное обогащение |
| `high_level_element` | Тематическая концепция | title, description, community_id | Концептуальный слой |

#### Типы рёбер и их семантика

| Ребро | Семантика | Направление | Вес |
|-------|----------|-------------|-----|
| `text_unit → semantic_unit` | "содержит" | Directed | 1 |
| `semantic_unit → entity` | "упоминает" | Directed | 1 |
| `relationship → source_entity` | "from" | Directed | relationship weight |
| `relationship → target_entity` | "to" | Directed | relationship weight |
| `attribute → entity` | "describes" | Directed | 1 |
| `high_level_element → community` | "summarizes" | Undirected | 1 |

#### Семантические связи с другими преобразованиями

```
Graph Construction
    │
    ├─→ Обеспечивает структуру для PPR navigation
    ├─→ Топология определяет важность узлов (для Attribute Gen)
    ├─→ Community structure → основа для Community Detection
    └─→ Связи между узлами → контекст для Attribute Generation
```

**Зависимости:**
- **Входные**: Text Decomposition (entities, semantic units, relationships)
- **Выходные**:
  - Attribute Generation (нужна топология графа)
  - Community Detection (анализирует структуру)
  - PPR (навигация по рёбрам)

**Влияние на качество:**
- Плотность графа → эффективность PPR
- Hub entities → качество Attribute Generation
- Community structure → релевантность Community Summarization

#### Топологические семантические свойства

**Централизация (Centrality)**:
- **Degree centrality**: Количество связей → важность сущности
- **Betweenness centrality**: Мостовые узлы → ключевые концепты
- **PageRank**: Global importance

**Сообщества (Communities)**:
- **Modularity**: Плотность внутригрупповых связей
- **Семантическая когерентность**: Узлы в сообществе связаны тематически

**Дистанция (Distance)**:
- **Shortest paths**: Семантическая близость концепций
- **Multi-hop reasoning**: Непрямые связи

---

### 4. Attribute Generation ⭐

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Средне-высокий | Синтез из множественных фактов |
| **Семантическая глубина** | Глубокая | Понимание роли и значимости сущности |
| **Контекстная зависимость** | Очень высокая | Требует всего контекста сущности |
| **Информационное преобразование** | Агрегирующее + Синтезирующее | Много фактов → одно описание |
| **Структурная сложность** | Средняя | Narrative text |
| **Детерминизм** | Высокий | Temperature 0.0 |
| **Обратимость** | Низкая | Потеря детализации |
| **Гранулярность** | Макро | 100-2000 слов |

#### Входные и выходные семантические свойства

**Вход:**
- Тип: Entity + Graph context
- Контекст:
  - Все semantic_units с упоминанием entity
  - Все relationships с entity
  - Community membership
- Размер: До 4000 токенов контекста

**Выход:**
- Тип: Narrative description
- Стиль: Character sketch / Product description
- Размер: 100-2000 слов
- Семантика: Comprehensive profile

#### Семантические операции

1. **Context Aggregation**: Сбор всех упоминаний
   - Traversal по рёбрам графа
   - Сбор semantic_units
   - Сбор relationships

2. **Synthesis**: Объединение в нарратив
   - Merge duplicate information
   - Organize chronologically
   - Create flowing narrative

3. **Prioritization**: Выбор важного
   - Rank facts by relevance
   - Truncate if > 4000 tokens context
   - Keep most informative

4. **Style Transformation**: Facts → Description
   - Convert triplets to sentences
   - Add connecting phrases
   - Professional tone

#### Семантические паттерны

**Паттерн: Multi-source fusion**
```
Input:
  Entity: DR. EMILY ROBERTS
  Semantic Units:
    - "Dr. Emily Roberts attended conference in Paris"
    - "Dr. Emily Roberts published research on solar panels"
  Relationships:
    - "DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE"
    - "DR. EMILY ROBERTS, researches, SOLAR PANEL EFFICIENCY"

Process:
  1. Identify roles: researcher, conference attendee
  2. Identify affiliations: European Research Institute
  3. Identify expertise: solar panel efficiency
  4. Synthesize timeline: publication → conference presentation

Output:
  "Dr. Emily Roberts is a prominent researcher in renewable energy,
   specializing in solar panel efficiency. Affiliated with the European
   Research Institute, she has published groundbreaking research and
   presented findings at international conferences including the
   International Conference on Renewable Energy in Paris..."
```

#### Семантические связи с другими преобразованиями

```
Attribute Generation
    │
    ├─→ Enriches entities for better retrieval (HNSW)
    ├─→ Provides context for Answer Synthesis
    └─→ Enables comprehensive entity understanding
```

**Зависимости:**
- **Входные**:
  - Graph Construction (topology + context)
  - Text Decomposition (semantic units)
- **Выходные**:
  - Embedding Generation (rich text for embeddings)
  - Context Assembly (detailed entity descriptions)

**Влияние на качество:**
- Богатые attributes → лучший recall в HNSW
- Comprehensive descriptions → более полные ответы
- Selective generation → оптимизация стоимости

---

### 5. Community Detection

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Высокий | Идентификация тематических кластеров |
| **Семантическая глубина** | Средняя | Основана на структуре, не на содержании |
| **Контекстная зависимость** | Средняя | Использует топологию графа |
| **Информационное преобразование** | Классифицирующее | Назначение кластеров |
| **Структурная сложность** | Высокая | Graph partitioning |
| **Детерминизм** | Низкий | Стохастический алгоритм |
| **Обратимость** | Нет | Потеря информации о границах |
| **Гранулярность** | Мезоскопическая | Группы узлов |

#### Входные и выходные семантические свойства

**Вход:**
- Тип: Knowledge Graph
- Структура: Heterogeneous, weighted
- Размер: Тысячи-миллионы узлов

**Выход:**
- Тип: Node partitions (communities)
- Свойства:
  - High modularity (плотные внутренние связи)
  - Semantic coherence (тематическая связность)
- Размер: 10-100+ communities

#### Семантическая интерпретация communities

**Community = Тематический кластер**
- Узлы в community семантически связаны
- Представляют общий "топик" или "концепцию"
- Примеры:
  - Community 1: "Renewable Energy Research"
  - Community 2: "Biodiversity Conservation"
  - Community 3: "Climate Change Impacts"

#### Семантические связи с другими преобразованиями

```
Community Detection
    │
    ├─→ Группирует контекст для Community Summarization
    ├─→ Обеспечивает тематическую организацию
    └─→ Identifies implicit topic structure
```

**Зависимости:**
- **Входные**: Graph Construction (topology)
- **Выходные**: Community Summarization (требует кластеры)

**Влияние на качество:**
- Качество кластеризации → релевантность тем
- Размер communities → уровень детализации тем
- Modularity → семантическая когерентность

---

### 6. Community Summarization ⭐

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Очень высокий | Концептуализация |
| **Семантическая глубина** | Очень глубокая | Понимание общих тем и паттернов |
| **Контекстная зависимость** | Очень высокая | Требует всего контекста community |
| **Информационное преобразование** | Абстрагирующее + Синтезирующее | Факты → Концепты |
| **Структурная сложность** | Средняя | Title + Description pairs |
| **Детерминизм** | Высокий | Temperature 0.0 |
| **Обратимость** | Очень низкая | Значительная потеря деталей |
| **Гранулярность** | Макро-концептуальная | 3-5 themes per community |

#### Входные и выходные семантические свойства

**Вход:**
- Тип: Community texts (aggregated)
- Источники:
  - Semantic units in community
  - Attributes in community
- Размер: До 8000 токенов
- Семантика: Множественные связанные факты

**Выход:**
- Тип: High-Level Elements
- Формат: {title, description} pairs
- Количество: 3-5 per community
- Семантика: Abstract themes/concepts
- Характер:
  - Title: Concise (2-5 words)
  - Description: Comprehensive (100-200 words)

#### Семантические операции

1. **Thematic Extraction**: Facts → Themes
   - Identify common concepts
   - Extract patterns
   - Find overarching themes

2. **Abstraction**: Concrete → Abstract
   - "Dr. Roberts researched solar panels" → "Renewable Energy Innovation"
   - Multiple facts → single concept

3. **Diversity Enforcement**: Non-redundant themes
   - Select diverse concepts
   - Avoid overlap
   - Cover different aspects

4. **Title Generation**: Concept → Concise label
   - 2-5 words
   - Descriptive
   - Professional

5. **Description Synthesis**: Theme → Narrative
   - 100-200 words
   - Comprehensive
   - Contextual

#### Семантические паттерны

**Паттерн: Bottom-up concept formation**
```
Input (Community texts):
  - "Dr. Emily Roberts attended renewable energy conference in Paris"
  - "Research on solar panel efficiency improvements"
  - "Published findings on 15% efficiency increase"
  - "Collaboration with European companies"

Process:
  1. Identify entities: Dr. Roberts, solar panels, efficiency, collaboration
  2. Identify actions: research, publish, collaborate, present
  3. Identify domain: renewable energy
  4. Abstract to concept: "Renewable Energy Research and Collaboration"
  5. Synthesize description

Output:
  {
    "title": "Renewable Energy Research and Innovation",
    "description": "Cutting-edge research in sustainable energy technologies,
                    focusing on solar panel efficiency improvements and
                    international collaboration through scientific conferences
                    and industry partnerships."
  }
```

**Паттерн: Multi-level abstraction**
```
Level 0 (Facts): "Solar panel efficiency increased by 15%"
Level 1 (Event): "Research breakthrough in solar technology"
Level 2 (Theme): "Renewable Energy Innovation"
Level 3 (Concept): "Sustainable Technology Development"  ← High-Level Element
```

#### Семантические связи с другими преобразованиями

```
Community Summarization
    │
    ├─→ Creates conceptual layer above facts
    ├─→ Enables thematic search (via HNSW)
    ├─→ Provides "big picture" context for Answer Synthesis
    └─→ Links concrete facts to abstract concepts
```

**Зависимости:**
- **Входные**:
  - Community Detection (clusters)
  - Text Decomposition (semantic units)
  - Attribute Generation (attributes)
- **Выходные**:
  - Embedding Generation (theme embeddings)
  - Context Assembly (THEMES section)

**Влияние на качество:**
- Качество абстракции → релевантность тематического поиска
- Diversity → coverage of query types
- Descriptions → context richness in answers

---

### 7. Embedding Generation

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Средний | Vector representation |
| **Семантическая глубина** | Глубокая | Кодирует семантическое значение |
| **Контекстная зависимость** | Средняя | Модель учит контекст |
| **Информационное преобразование** | Кодирующее | Text → Dense vector |
| **Структурная сложность** | Высокая | 512-1536 dimensions |
| **Детерминизм** | Полный | Детерминированная модель |
| **Обратимость** | Нет | Потеря поверхностной структуры |
| **Гранулярность** | Векторная | Continuous space |

#### Входные и выходные семантические свойства

**Вход:**
- Тип: Text (любой узел с content)
- Размер: Variable
- Типы:
  - Text units (~1048 tokens)
  - Semantic units (~50-200 tokens)
  - Entity names (~5-20 tokens)
  - Attributes (100-2000 tokens)
  - High-level elements (150-250 tokens)

**Выход:**
- Тип: Dense vector
- Размер: 512-1536 dimensions
- Свойства:
  - Normalized (L2 norm = 1)
  - Semantic similarity preserved
  - Cosine distance ≈ semantic distance

#### Семантические свойства векторного пространства

**Property 1: Semantic Similarity**
```
Similar texts → Similar vectors

Example:
  "renewable energy research" ≈ "sustainable energy studies"
  cosine_similarity ≈ 0.85
```

**Property 2: Compositionality** (частично)
```
"solar panel" + "efficiency" ≈ "solar panel efficiency"
```

**Property 3: Clustering**
```
Texts in same community → Vectors in same region
```

**Property 4: Dimensionality**
```
High-dimensional space → Rich semantic representation
1536 dimensions >> thousands of concepts
```

#### Семантические связи с другими преобразованиями

```
Embedding Generation
    │
    ├─→ Enables HNSW Retrieval (semantic search)
    ├─→ Vectorizes all node types
    └─→ Foundation for similarity-based operations
```

**Зависимости:**
- **Входные**: Все текстовые узлы (из всех предыдущих преобразований)
- **Выходные**:
  - HNSW Retrieval (queries vectors)
  - Clustering, similarity analysis

**Влияние на качество:**
- Качество embeddings → точность retrieval
- Dimensionality → expressiveness
- Model choice → domain adaptation

---

### 8. Query Decomposition

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Средний | Structural analysis |
| **Семантическая глубина** | Средняя | Understanding intent |
| **Контекстная зависимость** | Низкая-средняя | Query-only analysis |
| **Информационное преобразование** | Аналитическое | Query → Components |
| **Структурная сложность** | Средняя | Multiple components |
| **Детерминизм** | Средний | Зависит от метода |
| **Обратимость** | Высокая | Можно реконструировать query |
| **Гранулярность** | Компонентная | Entities, concepts, keywords |

#### Семантические операции

1. **Entity Recognition**: Identify query entities
2. **Concept Identification**: Extract abstract concepts
3. **Intent Classification**: Determine query type
4. **Query Expansion**: Generate related terms
5. **Keyword Extraction**: Core search terms

#### Семантические связи с другими преобразованиями

```
Query Decomposition
    │
    ├─→ Guides HNSW Retrieval (what to search)
    ├─→ Informs Context Assembly (what context is relevant)
    └─→ Shapes Answer Synthesis (what to emphasize)
```

---

### 9. HNSW Retrieval ⭐

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Средний | Vector space navigation |
| **Семантическая глубина** | Средняя | Similarity-based |
| **Контекстная зависимость** | Средняя | Query context |
| **Информационное преобразование** | Фильтрующее | Many nodes → Top-K |
| **Структурная сложность** | Высокая | Hierarchical graph |
| **Детерминизм** | Полный | Детерминированный поиск |
| **Обратимость** | Нет | Потеря информации о non-selected |
| **Гранулярность** | Top-K | Usually 20-100 nodes |

#### Семантические свойства поиска

**Semantic Similarity Search**
- Не keyword matching
- Понимает синонимы, парафразы
- Инвариантен к формулировке

**Multi-type Retrieval**
- Поиск по разным типам узлов
- Разная стратегия для каждого типа:
  - Entities: High precision
  - Semantic units: High recall
  - High-level elements: Thematic

**Top-K Selection**
- Ранжирование по релевантности
- Cutoff по score threshold
- Diversity consideration

#### Семантические связи с другими преобразованиями

```
HNSW Retrieval
    │
    ├─→ Provides seed nodes для PPR
    ├─→ Initial recall for Context Assembly
    └─→ Leverages Embedding Generation
```

---

### 10. Personalized PageRank ⭐

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Высокий | Structural reasoning |
| **Семантическая глубина** | Высокая | Понимание связей |
| **Контекстная зависимость** | Очень высокая | Использует всю топологию |
| **Информационное преобразование** | Расширяющее | Seed → Expanded set |
| **Структурная сложность** | Очень высокая | Graph traversal |
| **Детерминизм** | Полный | Детерминированный алгоритм |
| **Обратимость** | Частичная | Можно проследить пути |
| **Гранулярность** | Расширенная | 2-3x from seeds |

#### Семантические операции

1. **Random Walk**: Навигация по связям
   - Follows edges probabilistically
   - Returns to seeds with probability α

2. **Proximity Scoring**: Оценка близости
   - Closer nodes → higher scores
   - Multiple paths → higher scores

3. **Multi-hop Reasoning**: Непрямые связи
   - Discovers indirect relationships
   - A → B → C (A related to C through B)

4. **Context Expansion**: Обогащение контекста
   - Finds related entities
   - Finds connected facts

#### Семантические паттерны обнаружения

**Паттерн 1: Transitive relationships**
```
Query: "Dr. Emily Roberts"
Direct: DR. EMILY ROBERTS
1-hop: EUROPEAN RESEARCH INSTITUTE (via "works at")
2-hop: RENEWABLE ENERGY PROJECTS (via institute's projects)
```

**Паттерн 2: Shared context**
```
Seed: Entity A, Entity B
PPR discovers: Semantic units mentioning both
→ Context linking A and B
```

**Паттерн 3: Community co-membership**
```
Seed: DR. EMILY ROBERTS
PPR discovers: Other researchers in same community
→ Related work and collaborations
```

#### Семантические связи с другими преобразованиями

```
Personalized PageRank
    │
    ├─→ Leverages Graph Construction (topology)
    ├─→ Expands HNSW results (context enrichment)
    └─→ Provides comprehensive node set для Context Assembly
```

---

### 11. Context Assembly ⭐

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Высокий | Meta-organization |
| **Семантическая глубина** | Высокая | Understanding relevance hierarchy |
| **Контекстная зависимость** | Очень высокая | Query + Retrieved nodes |
| **Информационное преобразование** | Организующее | Nodes → Structured context |
| **Структурная сложность** | Высокая | Hierarchical sections |
| **Детерминизм** | Средний-высокий | Heuristic-based |
| **Обратимость** | Высокая | Можно извлечь узлы |
| **Гранулярность** | Секционная | THEMES → ENTITIES → RELATIONS → FACTS |

#### Семантические операции

1. **Hierarchical Grouping**: По типу и релевантности
2. **Deduplication**: Удаление избыточности
3. **Prioritization**: Ранжирование по важности
4. **Token Budget Management**: Оптимизация размера
5. **Formatting**: Структурирование для LLM

#### Семантическая иерархия контекста

```
Level 1: THEMES (High-Level Elements)
  ↓ Provides: Overall context, big picture

Level 2: KEY ENTITIES (Entities + Attributes)
  ↓ Provides: Who/what the query is about

Level 3: RELATIONSHIPS (Connections)
  ↓ Provides: How entities are connected

Level 4: FACTS (Semantic Units)
  ↓ Provides: Detailed supporting information
```

**Семантическая логика:**
- General → Specific (абстракция уменьшается)
- Context → Details (детализация увеличивается)
- Themes frame entities, entities frame facts

#### Семантические связи с другими преобразованиями

```
Context Assembly
    │
    ├─→ Organizes output from HNSW + PPR
    ├─→ Leverages type hierarchy from Graph
    ├─→ Prepares optimal input для Answer Synthesis
    └─→ Bridges retrieval and generation
```

---

### 12. Answer Synthesis ⭐

#### Семантические измерения

| Измерение | Характеристика | Значение/Описание |
|-----------|---------------|-------------------|
| **Уровень абстракции** | Высокий | Natural language generation |
| **Семантическая глубина** | Очень глубокая | Deep understanding + creativity |
| **Контекстная зависимость** | Максимальная | Query + Full context |
| **Информационное преобразование** | Генеративное | Context → Natural answer |
| **Структурная сложность** | Высокая | Coherent narrative |
| **Детерминизм** | Средний | Temperature 0.7-1.0 |
| **Обратимость** | Нет | Невозможно извлечь исходный контекст |
| **Гранулярность** | Предложенческая | Flowing text |

#### Семантические операции

1. **Information Fusion**: Merge facts
   - Combine related information
   - Remove redundancy
   - Create coherent narrative

2. **Style Adaptation**: Context → Natural language
   - Conversational tone
   - Professional style
   - Readable flow

3. **Completeness Checking**: Address all aspects
   - Ensure query fully answered
   - Cover all relevant context
   - Balance detail and conciseness

4. **Factual Grounding**: Stay within context
   - Don't hallucinate
   - Cite appropriately
   - Maintain accuracy

5. **Narrative Construction**: Facts → Story
   - Temporal ordering
   - Causal linking
   - Smooth transitions

#### Семантические паттерны генерации

**Паттерн 1: Fact fusion**
```
Input facts:
  - "Dr. Roberts researches solar panels"
  - "Dr. Roberts works at European Research Institute"

Output:
  "Dr. Roberts, a researcher at the European Research Institute,
   specializes in solar panel technology."
```

**Паттерн 2: Temporal organization**
```
Input facts (unordered):
  - "Published research in 2024"
  - "Presented at conference in Paris"
  - "Started research in 2022"

Output (ordered):
  "Dr. Roberts began her research in 2022, published findings in 2024,
   and subsequently presented them at a conference in Paris."
```

**Паттерн 3: Causal linking**
```
Input facts:
  - "Efficiency increased by 15%"
  - "New coating material used"

Output:
  "By utilizing a new coating material, Dr. Roberts achieved a 15%
   increase in solar panel efficiency."
```

#### Семантические связи с другими преобразованиями

```
Answer Synthesis
    │
    ├─→ Consumes Context Assembly output
    ├─→ Final transformation in chain
    └─→ Delivers value to user
```

---

## Общая таблица семантических характеристик

| Преобразование | Абстракция | Семант. глубина | Контекст. завис. | Детерминизм | Обратимость | Критичность |
|----------------|-----------|-----------------|------------------|-------------|-------------|-------------|
| Semantic Chunking | Низкая | Поверх. | Низкая | Полный | Полная | Средняя |
| Text Decomposition | Средняя | Глубокая | Высокая | Высокий | Частичная | **КРИТИЧ** |
| Graph Construction | Средняя | Средняя | Низкая | Полный | Полная | Высокая |
| Attribute Generation | Средне-выс. | Глубокая | Очень выс. | Высокий | Низкая | **КРИТИЧ** |
| Community Detection | Высокая | Средняя | Средняя | Низкий | Нет | Средняя |
| Community Summarization | Очень выс. | Очень глуб. | Очень выс. | Высокий | Очень низ. | **КРИТИЧ** |
| Embedding Generation | Средняя | Глубокая | Средняя | Полный | Нет | Высокая |
| Query Decomposition | Средняя | Средняя | Низкая-сред. | Средний | Высокая | Высокая |
| HNSW Retrieval | Средняя | Средняя | Средняя | Полный | Нет | **КРИТИЧ** |
| Personalized PageRank | Высокая | Высокая | Очень выс. | Полный | Частичная | **КРИТИЧ** |
| Context Assembly | Высокая | Высокая | Очень выс. | Средне-выс. | Высокая | **КРИТИЧ** |
| Answer Synthesis | Высокая | Очень глуб. | Максимал. | Средний | Нет | **КРИТИЧ** |

---

## Матрица зависимостей преобразований

```
               Chunk Decomp Graph Attr Comm.D Comm.S Embed Query HNSW PPR  Ctx  Answ
Chunking         -     ✓     -    -     -      -      -     -    -    -    -    -
Decomposition    -     -     ✓    ✓     -      -      ✓     -    -    -    -    -
Graph            -     -     -    ✓     ✓      -      -     -    -    ✓    -    -
Attribute        -     -     -    -     -      -      ✓     -    -    -    ✓    -
Community D      -     -     -    -     -      ✓      -     -    -    -    -    -
Community S      -     -     -    -     -      -      ✓     -    -    -    ✓    -
Embedding        -     -     -    -     -      -      -     ✓    ✓    -    -    -
Query            -     -     -    -     -      -      -     -    ✓    -    -    -
HNSW             -     -     -    -     -      -      -     -    -    ✓    -    -
PPR              -     -     -    -     -      -      -     -    -    -    ✓    -
Context          -     -     -    -     -      -      -     -    -    -    -    ✓
Answer           -     -     -    -     -      -      -     -    -    -    -    -

✓ = зависимость (строка зависит от столбца)
```

---

## Семантическая эволюция данных

### Путь документа через систему

```
Stage 0: Raw Document
  Семантика: Неструктурированный нарратив
  Форма: Prose text

Stage 1: Text Units
  Семантика: Фрагменты нарратива
  Форма: Chunked text
  Эволюция: Segmentation

Stage 2: Semantic Units + Entities + Relationships
  Семантика: Атомарные факты + Нормализованные сущности + Связи
  Форма: Structured JSON
  Эволюция: Extraction + Normalization

Stage 3: Knowledge Graph
  Семантика: Топологическая структура знаний
  Форма: Heterogeneous graph
  Эволюция: Structuring

Stage 4: Enriched Graph (+ Attributes)
  Семантика: Контекстно обогащённые сущности
  Форма: Graph + narrative nodes
  Эволюция: Aggregation + Synthesis

Stage 5: Vectorized Graph
  Семантика: Гибридное представление (structure + vectors)
  Форма: Graph + embeddings
  Эволюция: Encoding

Stage 6: Conceptual Graph (+ Communities + High-Level Elements)
  Семантика: Многоуровневая абстракция (факты + темы)
  Форма: Hierarchical knowledge structure
  Эволюция: Abstraction + Clustering

Final: Indexed Knowledge
  Семантика: Queryable, navigable knowledge structure
  Форма: Graph + HNSW indices
  Готовность: Ready for retrieval
```

### Путь запроса через систему

```
Stage 0: User Query
  Семантика: Естественный язык
  Форма: Text string

Stage 1: Query Components
  Семантика: Структурированный запрос
  Форма: {entities, concepts, keywords, intent}
  Эволюция: Decomposition

Stage 2: Query Embedding
  Семантика: Векторное представление
  Форма: Dense vector
  Эволюция: Encoding

Stage 3: Seed Nodes
  Семантика: Релевантные узлы знаний
  Форма: Top-K nodes
  Эволюция: Similarity search

Stage 4: Expanded Node Set
  Семантика: Обогащённый контекст
  Форма: Extended node set
  Эволюция: Graph navigation

Stage 5: Structured Context
  Семантика: Иерархически организованная информация
  Форма: Sectioned text
  Эволюция: Hierarchical assembly

Stage 6: Natural Language Answer
  Семантика: Человекочитаемый ответ
  Форма: Coherent narrative
  Эволюция: Generative synthesis

Final: Formatted Answer
  Семантика: Готовый ответ
  Форма: Markdown formatted text
  Готовность: Delivered to user
```

---

## Заключение

Семантические преобразования в NodeRAG образуют **сложную экосистему**, где:

1. **Каждое преобразование** имеет уникальные семантические характеристики
2. **Преобразования связаны** через входы/выходы и зависимости
3. **Критические преобразования** (⭐) определяют качество системы
4. **Семантическая эволюция** проходит от конкретного к абстрактному (индексация) и обратно (генерация)
5. **Баланс детерминизма**: Высокий при индексации (точность), средний при генерации (естественность)

**Ключевой принцип**: Преобразования не просто обрабатывают данные — они **создают семантические слои**, каждый из которых служит определённой цели в понимании и генерации знаний.
