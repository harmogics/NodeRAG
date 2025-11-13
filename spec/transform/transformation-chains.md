# Цепочки семантических преобразований в NodeRAG

## Обзор

Этот документ описывает полные цепочки семантических преобразований в NodeRAG, показывая как данные проходят через различные стадии обработки от исходного документа до финального ответа.

---

## Полная цепочка преобразований

```
┌─────────────────────────────────────────────────────────────────┐
│                      INDEXING PIPELINE                           │
└─────────────────────────────────────────────────────────────────┘

📄 Raw Document (неструктурированный текст)
     │
     │ [T1: Semantic Chunking]
     │ ➜ Разбиение по семантическим границам
     │ ➜ Сохранение контекстной целостности
     ↓
📝 Text Units (~1048 токенов каждый)
     │
     │ [T2: Text Decomposition] ⭐ КРИТИЧЕСКАЯ
     │ ➜ Извлечение атомарных фактов
     │ ➜ Идентификация сущностей (UPPERCASE)
     │ ➜ Извлечение отношений (triplets)
     │ ➜ Темпоральная нормализация
     │ ➜ LLM Agent (Temperature 0.0)
     ↓
{Semantic Units, Entities, Relationships}
     │
     │ [T7: Graph Construction]
     │ ➜ Создание типизированных узлов
     │ ➜ Формирование рёбер
     │ ➜ Агрегация весов
     │ ➜ Hash-based дедупликация
     ↓
🕸️ Knowledge Graph (NetworkX, гетерогенный)
     │
     ├─────────────────┬─────────────────────────┐
     │                 │                         │
     │ [T4: Attribute  │ [T6: Embedding         │ [T8: Community
     │  Generation] ⭐  │  Generation]            │  Detection]
     │                 │                         │
     │ ➜ Агрегация     │ ➜ Text Units →         │ ➜ Leiden Algorithm
     │   контекста     │   Vectors               │ ➜ Кластеризация
     │ ➜ Синтез        │ ➜ Semantic Units →     │   узлов
     │   описания      │   Vectors               │ ➜ Модульность
     │ ➜ 10-15%        │ ➜ Entities →           │
     │   сущностей     │   Vectors               │
     │ ➜ LLM Agent     │ ➜ OpenAI/Gemini        │
     │   (Temp 0.0)    │   Models                │
     ↓                 ↓                         ↓
📋 Attributes     🔢 Embeddings (512-1536)   👥 Communities
     │                 │                         │
     │                 └─────────┬───────────────┘
     │                           │
     │                           │ [T5: Community
     │                           │  Summarization] ⭐
     │                           │
     │                           │ ➜ Абстракция
     │                           │   концепций
     │                           │ ➜ Тематическая
     │                           │   экстракция
     │                           │ ➜ Генерация
     │                           │   заголовков
     │                           │ ➜ LLM Agent
     │                           │   (Temp 0.0)
     │                           ↓
     │                      🎯 High-Level Elements
     │                           │
     │                           │ [T6: Embedding
     │                           │  Generation]
     │                           ↓
     └─────────────────────→ 🔢 All Embeddings
                                 │
                                 ↓
                            📊 HNSW Index
                                 │
                                 │
┌────────────────────────────────┴─────────────────────────────────┐
│                    ANSWER GENERATION PIPELINE                     │
└──────────────────────────────────────────────────────────────────┘
                                 ↓
❓ User Query (естественный язык)
     │
     │ [T1: Query Decomposition]
     │ ➜ Извлечение сущностей
     │ ➜ Идентификация концепций
     │ ➜ Расширение запроса
     │ ➜ Определение намерения
     ↓
{Entities, Concepts, Keywords, Intent}
     │
     │ Embedding Generation
     ↓
🔍 Query Embedding (512-1536 dim)
     │
     │ [T2: HNSW Retrieval] ⭐ КРИТИЧЕСКАЯ
     │ ➜ Semantic similarity search
     │ ➜ Top-K по cosine similarity
     │ ➜ Multi-type retrieval:
     │   • text_units
     │   • entities
     │   • semantic_units
     │   • attributes
     │   • high_level_elements
     ↓
📦 Seed Nodes (Top-K, 20-100 узлов)
     │
     │ [T3: Personalized PageRank] ⭐ КРИТИЧЕСКАЯ
     │ ➜ Random walk от seed nodes
     │ ➜ Структурная релевантность
     │ ➜ Multi-hop reasoning
     │ ➜ Обнаружение непрямых связей
     │ ➜ Expansion factor: 2-3x
     ↓
📦 Expanded Node Set (50-300 узлов)
     │
     │ [T4: Context Assembly] ⭐ КРИТИЧЕСКАЯ
     │ ➜ Иерархическая организация:
     │   1. THEMES (High-Level Elements)
     │   2. KEY ENTITIES (Entities + Attributes)
     │   3. RELATIONSHIPS (Connections)
     │   4. FACTS (Semantic Units)
     │ ➜ Дедупликация
     │ ➜ Приоритизация
     │ ➜ Токен-бюджет (4000-8000)
     ↓
📋 Structured Context
     │
     │ [T5: Answer Synthesis] ⭐ КРИТИЧЕСКАЯ
     │ ➜ Информационный синтез
     │ ➜ Стилевая адаптация
     │ ➜ Фактическая обоснованность
     │ ➜ Нарративная конструкция
     │ ➜ LLM Generation (Temp 0.7-1.0)
     ↓
📝 Natural Language Answer
     │
     │ [T6: Post-processing]
     │ ➜ Форматирование (Markdown)
     │ ➜ Добавление цитирований
     │ ➜ Валидация качества
     ↓
✅ Final Answer to User
```

---

## Детальные цепочки по стадиям

### Стадия 1: Подготовка документа

```
Raw Document
    │
    ├─ Размер: Любой (может быть >100,000 токенов)
    ├─ Формат: Неструктурированный текст
    └─ Проблемы: Слишком большой для LLM, нет структуры
    │
    │ T1: Semantic Chunking
    │ Agent: SemanticTextSplitter
    │ Метод: Разбиение по границам \n\n > \n > . > ! > ? > ;
    │ Цель: Создать оптимальные для LLM фрагменты
    ↓
Text Units
    │
    ├─ Размер: ~1048 токенов каждый
    ├─ Свойства: Семантически целостные
    └─ Количество: N = ceil(doc_size / 1048)
```

**Характеристики преобразования:**
- **Тип**: Сегментация
- **Сохранение**: Полное (потерь данных нет)
- **Детерминизм**: Полностью детерминированное
- **Latency**: Низкая (~секунды для больших документов)

---

### Стадия 2: Извлечение структуры (Центральная)

```
Text Unit
    │
    ├─ Вход: "В сентябре 2024 года доктор Эмили Робертс посетила
    │         Париж для участия в Международной конференции по
    │         возобновляемой энергии, где представила революционное
    │         исследование о повышении эффективности солнечных панелей."
    │
    │ T2: Text Decomposition ⭐
    │ Agent: LLM (GPT-4, Gemini-Pro)
    │ Temperature: 0.0 (детерминизм)
    │ Schema: Structured JSON output
    │ Методы:
    │   1. Сегментация → атомарные факты
    │   2. Парафраз → сжатие с деталями
    │   3. Entity extraction → UPPERCASE нормализация
    │   4. Relationship extraction → triplets
    │   5. Temporal normalization → YYYY-MM формат
    ↓
Decomposition Result
    │
    ├─ Semantic Units: [
    │   "В сентябре 2024 года доктор Эмили Робертс приняла участие
    │    в Международной конференции по возобновляемой энергии в Париже.",
    │   "Доктор Эмили Робертс представила исследование о повышении
    │    эффективности солнечных панелей."
    │  ]
    │
    ├─ Entities: [
    │   "DR. EMILY ROBERTS",
    │   "2024-09",
    │   "PARIS",
    │   "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
    │   "SOLAR PANEL EFFICIENCY"
    │  ]
    │
    └─ Relationships: [
    │   "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
    │   "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY, located in, PARIS",
    │   "DR. EMILY ROBERTS, researched, SOLAR PANEL EFFICIENCY"
    │  ]
```

**Характеристики преобразования:**
- **Тип**: Экстракция + Нормализация + Структурирование
- **Сжатие**: ~30-50% (парафраз)
- **Обогащение**: Добавление структуры (entities, relations)
- **Детерминизм**: Высокий (Temperature 0.0)
- **Latency**: Средняя (~2-5 сек на Text Unit)
- **Критичность**: МАКСИМАЛЬНАЯ (основа всей системы)

---

### Стадия 3: Построение графа знаний

```
{Semantic Units, Entities, Relationships}
    │
    │ T7: Graph Construction
    │ Engine: NetworkX
    │ Операции:
    │   1. Create nodes по типам
    │   2. Assign hash_id = hash(content)
    │   3. Create edges (semantic_unit → entity)
    │   4. Create edges (relationship → source/target entity)
    │   5. Aggregate weights (increment for duplicates)
    │   6. Store properties
    ↓
Knowledge Graph
    │
    ├─ Nodes:
    │   • semantic_unit (атомарные факты)
    │   • entity (сущности)
    │   • relationship (связи)
    │   • text_unit (исходные фрагменты)
    │
    ├─ Edges:
    │   • semantic_unit → entity (содержит упоминание)
    │   • relationship → source_entity
    │   • relationship → target_entity
    │   • semantic_unit → text_unit (принадлежит)
    │
    └─ Properties:
        • node_type: тип узла
        • content: текстовое содержание
        • hash_id: уникальный идентификатор
        • weight: количество упоминаний
        • metadata: дополнительная информация

Пример структуры:

    [Text Unit 1]
         │
         ├─→ [Semantic Unit 1] ──→ [DR. EMILY ROBERTS]
         │                           ↑
         │                           │
         └─→ [Semantic Unit 2] ──→ [Relationship: attended] ──→ [CONFERENCE]
```

**Характеристики преобразования:**
- **Тип**: Структурирование (flat → graph)
- **Обогащение**: Добавление связей и топологии
- **Дедупликация**: Через hash_id
- **Детерминизм**: Полностью детерминированное
- **Latency**: Низкая (~миллисекунды на узел)

---

### Стадия 4: Обогащение сущностей (Селективное)

```
Knowledge Graph → Entity Selection
    │
    │ Критерии отбора:
    │   1. K-core decomposition (core_number ≥ 2)
    │   2. Betweenness centrality (топ 20%)
    │   3. Weight > 1 (множественные упоминания)
    │ Результат: ~10-15% сущностей
    ↓
Selected Entities: [DR. EMILY ROBERTS, EUROPEAN RESEARCH INSTITUTE, ...]
    │
    │ Для каждой сущности:
    │
    │ Collect Context:
    │   • Все связанные semantic_units
    │   • Все связанные relationships
    │   • Community membership
    │
    │ T4: Attribute Generation ⭐
    │ Agent: LLM
    │ Temperature: 0.0
    │ Input: Entity + Context (до 4000 токенов)
    │ Методы:
    │   1. Агрегация всех упоминаний
    │   2. Синтез в связное описание
    │   3. Приоритизация важной информации
    │   4. Стилевая трансформация (character sketch)
    │   5. Дедупликация фактов
    ↓
Attribute Node
    │
    ├─ Content: "Доктор Эмили Робертс - видный исследователь в
    │            области возобновляемой энергии, специализирующийся
    │            на повышении эффективности солнечных панелей..."
    │            [100-2000 слов]
    │
    ├─ Связь: Attribute → Entity (describes)
    │
    └─ Преимущество: Богатый контекст для поиска и ответов
```

**Характеристики преобразования:**
- **Тип**: Агрегация + Синтез + Абстракция
- **Расширение**: Много фактов → одно описание
- **Селективность**: 10-15% сущностей
- **Детерминизм**: Высокий (Temperature 0.0)
- **Latency**: Высокая (~5-10 сек на сущность)
- **Критичность**: ВЫСОКАЯ (улучшает качество поиска)

---

### Стадия 5: Векторизация (Параллельная)

```
Все текстовые узлы:
    ├─ Text Units
    ├─ Semantic Units
    ├─ Entities (имена)
    └─ Attributes
    │
    │ T6: Embedding Generation
    │ Models:
    │   • OpenAI: text-embedding-3-small (1536 dim)
    │   • Gemini: text-embedding-004 (768 dim)
    │ Batch size: 100
    │ Метод: Dense vector encoding
    ↓
Embeddings для каждого узла
    │
    ├─ Dimensionality: 512-1536
    ├─ Normalization: L2 norm
    └─ Property: embedding vector
    │
    │ HNSW Indexing
    │ Parameters:
    │   • M: 16 (количество соседей)
    │   • ef_construction: 200
    │   • ef_search: 50
    │   • metric: cosine
    ↓
HNSW Index (для быстрого поиска)
```

**Характеристики преобразования:**
- **Тип**: Кодирование (text → vector)
- **Сжатие**: Семантическое сжатие в плотное пространство
- **Сохранение**: Semantic similarity
- **Детерминизм**: Полностью детерминированное
- **Latency**: Средняя (~100ms на батч из 100)
- **Критичность**: ВЫСОКАЯ (основа для поиска)

---

### Стадия 6: Тематическое обобщение

```
Knowledge Graph
    │
    │ T8: Community Detection
    │ Algorithm: Leiden
    │ Цель: Найти семантические кластеры
    │ Метод: Модульность optimization
    │ Parameters:
    │   • resolution: 1.0
    │   • randomness: 0.01
    ↓
Communities (кластеры узлов)
    │
    ├─ Community 1: [узлы о renewable energy research]
    ├─ Community 2: [узлы о deforestation research]
    └─ Community 3: [узлы о environmental conservation]
    │
    │ Для каждого сообщества:
    │
    │ Collect Texts:
    │   • Semantic units в сообществе
    │   • Attributes в сообществе
    │   • Ограничение: до 8000 токенов
    │
    │ T5: Community Summarization ⭐
    │ Agent: LLM
    │ Temperature: 0.0
    │ Методы:
    │   1. Тематическая экстракция
    │   2. Абстракция (facts → concepts)
    │   3. Генерация заголовков (2-5 слов)
    │   4. Синтез описаний (100-200 слов)
    │   5. Разнообразие (3-5 элементов на сообщество)
    ↓
High-Level Elements
    │
    ├─ Element 1:
    │   • Title: "Renewable Energy Research and Innovation"
    │   • Description: "Cutting-edge research in sustainable
    │                   energy technologies, focusing on solar
    │                   panel efficiency improvements..."
    │
    ├─ Element 2:
    │   • Title: "Biodiversity Conservation Research"
    │   • Description: "Field research documenting species..."
    │
    └─ Связь: High-Level Element → Community nodes
    │
    │ T6: Embedding Generation
    ↓
High-Level Embeddings → HNSW Index
```

**Характеристики преобразования:**
- **Тип**: Кластеризация + Абстракция + Синтез
- **Сжатие**: Много узлов → несколько концепций
- **Повышение уровня**: Факты → Темы
- **Детерминизм**: Средний (Leiden стохастический, LLM детерминированный)
- **Latency**: Средняя (~5-10 сек на сообщество)
- **Критичность**: ВЫСОКАЯ (обеспечивает концептуальный поиск)

---

## Цепочка генерации ответа (детальная)

### Этап 1: Понимание запроса

```
User Query: "Какие исследования проводила доктор Эмили Робертс?"
    │
    │ T1: Query Decomposition
    │ Методы:
    │   1. Entity recognition (NER)
    │   2. Concept extraction
    │   3. Keyword extraction
    │   4. Intent classification
    │   5. Query expansion
    ↓
Query Components:
    │
    ├─ Entities: ["доктор эмили робертс", "DR. EMILY ROBERTS"]
    ├─ Concepts: ["исследования", "research", "научная работа"]
    ├─ Keywords: ["проводила", "studies"]
    ├─ Intent: "factual_query" (поиск фактов)
    └─ Expansion: ["публикации", "проекты", "конференции"]
```

---

### Этап 2-3: Двухуровневый поиск

```
Query Components
    │
    │ Embedding Generation
    ↓
Query Embedding: [0.234, -0.567, 0.891, ...] (1536 dim)
    │
    │ T2: HNSW Retrieval ⭐
    │ Parallel search across node types:
    │
    ├─→ Search in entities (Top-20)
    │   Result: DR. EMILY ROBERTS (0.92)
    │
    ├─→ Search in attributes (Top-10)
    │   Result: Attribute of DR. EMILY ROBERTS (0.87)
    │
    ├─→ Search in semantic_units (Top-30)
    │   Results: ["Доктор Эмили Робертс опубликовала..." (0.89),
    │             "В сентябре 2024..." (0.85), ...]
    │
    ├─→ Search in high_level_elements (Top-10)
    │   Result: "Renewable Energy Research" (0.83)
    │
    └─→ Search in text_units (Top-20)
        Results: [original text fragments]
    ↓
Seed Nodes (Top-K=50 combined)
    │
    ├─ DR. EMILY ROBERTS (0.92)
    ├─ Attribute: "Доктор Эмили Робертс - видный..." (0.87)
    ├─ Semantic Unit: "Доктор Эмили Робертс опубликовала..." (0.89)
    ├─ High-Level: "Renewable Energy Research" (0.83)
    └─ [46 more nodes]
    │
    │ T3: Personalized PageRank ⭐
    │ Method: Random walk с teleportation к seed nodes
    │ Parameters:
    │   • alpha: 0.85 (damping factor)
    │   • max_iter: 100
    │   • teleport distribution: пропорционально seed scores
    │
    │ Процесс:
    │   1. Initialize: seed nodes = high probability
    │   2. Iterate: распространение probability по рёбрам
    │   3. Converge: когда изменения < tolerance
    │   4. Rank: все узлы по PPR score
    ↓
Expanded Node Set (Top-150 by PPR score)
    │
    ├─ Seed nodes (50)
    └─ Discovered nodes (100):
        ├─ EUROPEAN RESEARCH INSTITUTE (0.78) - via "works at"
        ├─ SOLAR PANEL EFFICIENCY (0.75) - via "researches"
        ├─ INTERNATIONAL CONFERENCE... (0.72) - via semantic_unit
        ├─ PARIS (0.68) - via event location
        └─ Related semantic_units (96 more)
```

---

### Этап 4: Сборка контекста (Иерархическая)

```
Expanded Node Set (150 nodes)
    │
    │ T4: Context Assembly ⭐
    │
    │ Step 1: Группировка по типу
    ├─→ High-Level Elements: 5 nodes
    ├─→ Entities: 8 nodes
    ├─→ Attributes: 3 nodes
    ├─→ Relationships: 15 nodes
    ├─→ Semantic Units: 80 nodes
    └─→ Text Units: 39 nodes
    │
    │ Step 2: Дедупликация
    │   • Remove duplicate hash_ids
    │   • Merge identical content
    │   Результат: 150 → 112 unique nodes
    │
    │ Step 3: Приоритизация
    │   Ranking factors:
    │   • PPR score (40%)
    │   • Node type priority (30%)
    │   • Semantic similarity (20%)
    │   • Diversity (10%)
    │
    │ Step 4: Токен-бюджет allocation
    │   Total budget: 4000 tokens
    │   Allocation:
    │   • High-Level: 800 tokens (20%)
    │   • Entities+Attributes: 1200 tokens (30%)
    │   • Relationships: 400 tokens (10%)
    │   • Semantic Units: 1600 tokens (40%)
    │
    │ Step 5: Форматирование
    ↓
Structured Context (3847 tokens):

═══════════════════════════════════════
=== ТЕМЫ ===

# Исследования и инновации в возобновляемой энергии
Передовые исследования в области технологий устойчивой энергетики,
фокусирующиеся на повышении эффективности солнечных панелей и
международном сотрудничестве через научные конференции...
[5 high-level elements, 780 tokens]

═══════════════════════════════════════
=== КЛЮЧЕВЫЕ СУЩНОСТИ ===

## DR. EMILY ROBERTS
Доктор Эмили Робертс - видный исследователь в области возобновляемой
энергии, специализирующийся на повышении эффективности солнечных панелей...
[8 entities with attributes, 1180 tokens]

═══════════════════════════════════════
=== СВЯЗИ ===

- DR. EMILY ROBERTS работает в EUROPEAN RESEARCH INSTITUTE
- DR. EMILY ROBERTS исследует SOLAR PANEL EFFICIENCY
- DR. EMILY ROBERTS посетила INTERNATIONAL CONFERENCE...
[15 relationships, 387 tokens]

═══════════════════════════════════════
=== ФАКТЫ ===

• В сентябре 2024 года доктор Эмили Робертс посетила Международную
  конференцию по возобновляемой энергии в Париже
• Доктор Робертс опубликовала исследование, демонстрирующее 15%-ное
  увеличение эффективности солнечных панелей
• Исследование было представлено на конференции и получило высокую
  оценку международного научного сообщества
...
[42 semantic units selected, 1500 tokens]

═══════════════════════════════════════
```

---

### Этап 5: Синтез ответа

```
Input to LLM:
    │
    ├─ System Prompt: "Ты - помощник по анализу знаний. Используй
    │                  только предоставленный контекст. Отвечай точно..."
    │
    ├─ User Query: "Какие исследования проводила доктор Эмили Робертс?"
    │
    └─ Context: [structured context above, 3847 tokens]
    │
    │ T5: Answer Synthesis ⭐
    │ Model: GPT-4 / Gemini-Pro
    │ Temperature: 0.7 (баланс точность/естественность)
    │ Max tokens: 500
    │ Top-p: 0.9
    │
    │ Процесс:
    │   1. Понимание запроса в контексте
    │   2. Извлечение релевантных фактов
    │   3. Объединение фактов (information fusion)
    │   4. Организация в нарратив
    │   5. Добавление связующих фраз
    │   6. Проверка полноты ответа
    ↓
Natural Language Answer (342 tokens):

"Доктор Эмили Робертс проводит исследования в области возобновляемой
энергии, специализируясь на технологиях солнечных панелей. Её работа
направлена на повышение эффективности фотоэлектрических систем, и она
опубликовала результаты, демонстрирующие 15%-ное увеличение эффективности
солнечных панелей.

Как ведущий учёный Европейского исследовательского института, доктор
Робертс активно участвует в международном научном сообществе. В сентябре
2024 года она представила своё революционное исследование на Международной
конференции по возобновляемой энергии в Париже, подчёркивая его значимость
для развития устойчивой энергетики.

Её исследования имеют практическое применение: доктор Робертс установила
партнёрства с европейскими компаниями для внедрения передовых технологий
в области возобновляемой энергии, способствуя переходу к более экологичным
энергетическим решениям."
    │
    │ T6: Post-processing
    │   • Markdown formatting
    │   • Citation insertion (optional)
    │   • Quality validation
    ↓
Final Answer ✅
```

---

## Сравнение цепочек: Индексация vs Генерация ответа

| Аспект | Индексация | Генерация ответа |
|--------|-----------|------------------|
| **Направление** | Bottom-up (текст → концепции) | Top-down (запрос → факты) |
| **Стратегия** | Exhaustive processing | Selective retrieval |
| **Параллелизм** | Высокий (батчи документов) | Низкий (один запрос) |
| **Latency** | Минуты-часы | Секунды |
| **Детерминизм** | Высокий (Temp 0.0) | Средний (Temp 0.7) |
| **Критические этапы** | Decomposition, Attribute, Community | Retrieval, Context, Synthesis |
| **LLM роль** | Extraction & Structuring | Navigation & Generation |
| **Данные** | Все документы | Только релевантное |
| **Цель** | Создать структуру знаний | Найти и синтезировать ответ |

---

## Ключевые паттерны преобразований

### 1. Cascade (Каскад)
Последовательные преобразования, где выход одного — вход другого.

```
A → T1 → B → T2 → C → T3 → D
```

**Пример**: Text → Chunking → Text Units → Decomposition → Graph

---

### 2. Parallel (Параллель)
Независимые преобразования от одного входа.

```
     ┌→ T1 → B
A ───┼→ T2 → C
     └→ T3 → D
```

**Пример**: Graph → {Attribute Gen, Embedding Gen, Community Detection}

---

### 3. Convergence (Схождение)
Множественные входы объединяются в один выход.

```
A → T1 ─┐
B → T2 ─┼→ D
C → T3 ─┘
```

**Пример**: {Seed Nodes, Graph} → PPR → Expanded Nodes

---

### 4. Hierarchical (Иерархический)
Преобразования создают уровни абстракции.

```
Level 0: Raw Data
    ↓ T1
Level 1: Structured Data
    ↓ T2
Level 2: Enriched Data
    ↓ T3
Level 3: Abstract Concepts
```

**Пример**: Text → Units → Facts → Graph → Attributes → Communities → Themes

---

### 5. Selective (Селективный)
Преобразование применяется только к части данных.

```
A ─┬→ filter → T → B'
   └→ (не обработано) → B
```

**Пример**: Entities → (top 15%) → Attribute Generation

---

## Метрики эффективности цепочек

### Индексация

| Метрика | Значение | Комментарий |
|---------|----------|-------------|
| **Throughput** | ~100-500 docs/hour | Зависит от размера документов |
| **Total latency** | 5-30 min/doc | Для документа средней длины |
| **LLM calls** | N_units + N_important_entities + N_communities | Основные затраты |
| **Data expansion** | 3-5x | Граф больше исходных документов |
| **Information loss** | <5% | Минимальные потери при парафразе |

### Генерация ответа

| Метрика | Значение | Комментарий |
|---------|----------|-------------|
| **Throughput** | ~20-60 queries/min | С кэшированием |
| **Total latency** | 2-5 seconds | Для типичного запроса |
| **Retrieval recall** | 85-95% | HNSW + PPR |
| **Answer relevance** | 90-95% | При наличии данных |
| **Context utilization** | 60-80% | Полезная информация в контексте |

---

## Оптимизационные стратегии

### Для индексации

1. **Batch Processing**
   - Параллельная обработка Text Units
   - Батчевая генерация эмбеддингов (100 за раз)

2. **Incremental Updates**
   - Обработка только новых документов
   - Инкрементальное обновление графа

3. **Adaptive Chunking**
   - Динамическая настройка размера chunks
   - Учёт структуры документа

### Для генерации ответа

1. **Query Caching**
   - Кэш результатов для похожих запросов
   - TTL: 1 час

2. **Early Termination**
   - Прекращение PPR при достижении порога
   - Ограничение расширения узлов

3. **Context Pruning**
   - Агрессивная дедупликация
   - Приоритизация high-value узлов

4. **Streaming Generation**
   - Потоковый вывод ответа
   - Улучшение UX

---

## Заключение

Цепочки преобразований в NodeRAG образуют двухфазную систему:

**Фаза 1: Индексация** - Преобразует неструктурированные документы в богатую, многоуровневую структуру знаний (граф + векторы + концепты)

**Фаза 2: Генерация** - Навигирует по структуре знаний и синтезирует точные ответы из найденных фрагментов

Ключевой принцип: **Структурируй один раз, используй многократно** - затраты на индексацию окупаются качеством и скоростью генерации ответов.
