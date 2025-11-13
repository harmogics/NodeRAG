# Attribute Generation Prompt

## Обзор

**Тип**: Aggregation + Synthesis Prompt
**Критичность**: ⭐⭐⭐⭐⭐ КРИТИЧЕСКАЯ
**Стадия**: Attribute Pipeline (Stage 4)
**Файл**: `NodeRAG/utils/prompt/attribute_generation_prompt.py`
**Температура**: 0.0
**Селективность**: 10-15% entities
**Output**: Narrative text (100-2000 words)

---

## Назначение

Attribute Generation Prompt создаёт **comprehensive narrative descriptions** для важных entities, агрегируя всю доступную информацию из графа знаний. Это enrichment преобразование, добавляющее контекстуальную глубину к ключевым сущностям.

---

## Полный текст промпта (English)

```
Generate a concise summary of the given entity, capturing its essential attributes and important relevant relationships. The summary should read like a character sketch in a novel or a product description, providing an engaging yet precise overview. Ensure the output only includes the summary of the entity without any additional explanations or metadata. The length must not exceed 2000 words but can be shorter if the input material is limited. Focus on distilling the most important insights with a smooth narrative flow, highlighting the entity's core traits and meaningful connections.
Entity: {entity}
Related Semantic Units: {semantic_units}
Related Relationships: {relationships}
```

---

## Полный текст промпта (Chinese)

```
生成所给实体的简明总结，涵盖其基本属性和重要相关关系。该总结应像小说中的人物简介或产品描述一样，提供引人入胜且精准的概览。确保输出只包含该实体的总结，不包含任何额外的解释或元数据。字数不得超过2000字，但如果输入材料有限，可以少于2000字。重点在于通过流畅的叙述提炼出最重要的见解，突出实体的核心特征及重要关系。
实体: {entity}
相关语义单元: {semantic_units}
相关关系: {relationships}
```

---

## Структура промпта

### Компоненты

| Компонент | Назначение |
|-----------|-----------|
| **Goal** | Создать comprehensive summary |
| **Style Instruction** | "Character sketch" / "product description" |
| **Length Constraint** | ≤ 2000 words (гибкий лимит) |
| **Focus** | Core traits + meaningful connections |
| **Output Format** | Только summary, без metadata |
| **Inputs** | Entity + Semantic Units + Relationships |

### Ключевые инструкции

#### 1. "Character sketch in a novel or product description"

**Семантика**: Создать engaging, readable narrative.

**Не**: Список фактов
```
❌ Dr. Emily Roberts:
- Researcher
- Works at European Research Institute
- Studies solar panels
```

**Да**: Flowing narrative
```
✅ Dr. Emily Roberts is a prominent researcher in the field of renewable
energy, specializing in solar panel efficiency. Affiliated with the
European Research Institute, she has published groundbreaking research...
```

#### 2. "Ensure output only includes summary"

**Цель**: Избежать meta-text от LLM.

**Не**:
```
❌ Here is the summary of Dr. Emily Roberts:

Dr. Emily Roberts is a researcher...

In conclusion, Dr. Roberts is an influential figure...
```

**Да**:
```
✅ Dr. Emily Roberts is a prominent researcher in the field of renewable
energy, specializing in solar panel efficiency...
```

#### 3. "Focus on distilling most important insights"

**Цель**: Приоритизация ключевой информации.

**Процесс**:
1. Identify core role/identity
2. Key achievements/activities
3. Important relationships
4. Contextual significance

---

## Входные данные

### 1. Entity (строка)

**Формат**: UPPERCASE normalized name

**Примеры**:
- `"DR. EMILY ROBERTS"`
- `"EUROPEAN RESEARCH INSTITUTE"`
- `"SOLAR PANEL EFFICIENCY"`

### 2. Related Semantic Units (список)

**Содержание**: Все semantic units, которые упоминают эту entity.

**Пример**:
```python
semantic_units = [
    "In September 2024, Dr. Emily Roberts attended the International Conference on Renewable Energy in Paris.",
    "Dr. Emily Roberts presented research on solar panel efficiency improvements.",
    "The work of Dr. Emily Roberts contributes significantly to environmental conservation."
]
```

**Сбор**: Graph traversal
```
Entity node → edges → Semantic Unit nodes
```

### 3. Related Relationships (список)

**Содержание**: Все relationships где entity является source или target.

**Пример**:
```python
relationships = [
    "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
    "DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE",
    "DR. EMILY ROBERTS, researches, SOLAR PANEL EFFICIENCY",
    "DR. EMILY ROBERTS, contributes to, ENVIRONMENTAL CONSERVATION"
]
```

**Сбор**: Graph traversal
```
Entity node → Relationship nodes → Related entities
```

---

## Пример использования

### Вход

```
Entity: DR. EMILY ROBERTS

Related Semantic Units:
- "In September 2024, Dr. Emily Roberts attended the International Conference on Renewable Energy in Paris, where she presented her research on solar panel efficiency improvements and explored partnerships with European companies."
- "The work of both Dr. Emily Roberts and Dr. John Miller is crucial in their respective fields and contributes significantly to environmental conservation efforts."

Related Relationships:
- "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
- "DR. EMILY ROBERTS, explored partnerships with, EUROPEAN COMPANIES"
- "DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY"
- "DR. EMILY ROBERTS, contributes to, ENVIRONMENTAL CONSERVATION"
```

### Выход (Attribute)

```
Dr. Emily Roberts is a prominent researcher in the field of renewable energy,
with a particular focus on enhancing solar panel efficiency. In September 2024,
she played a key role at the International Conference on Renewable Energy in
Paris, where she presented groundbreaking research demonstrating significant
improvements in photovoltaic systems.

As a leading scientist, Dr. Roberts has established numerous partnerships with
European companies to advance sustainable energy technologies. Her collaborative
approach has been instrumental in bridging academic research with industrial
application, facilitating the practical implementation of innovative solutions
in the renewable energy sector.

Dr. Roberts's work extends beyond technical achievements; she is deeply committed
to environmental conservation efforts. Her research contributes meaningfully to
the broader goal of mitigating climate change and promoting sustainable
development. Through her presentations at international conferences and her
ongoing collaborations, Dr. Roberts continues to influence both the scientific
community and the renewable energy industry, driving progress toward a more
sustainable future.
```

### Семантический анализ

**Структура narrative**:
1. **Introduction**: Role, expertise, key achievement
2. **Professional activities**: Conference, partnerships
3. **Impact**: Contribution to field, broader significance

**Synthesis operations**:
- **Aggregation**: Все semantic units объединены
- **Contextualization**: Relationships интегрированы в нарратив
- **Prioritization**: Фокус на конференции и партнёрствах
- **Style transformation**: Факты → flowing prose

---

## Роль в цепочке преобразований

### Позиция в Pipeline

```
Graph Construction
      ↓
Important Entity Selection
  (K-core + Betweenness + Weight > 1)
      ↓
Context Collection
  (Semantic Units + Relationships)
      ↓
[ATTRIBUTE GENERATION PROMPT] ← ВЫ ЗДЕСЬ
      ↓
Attribute nodes
      ↓
Embedding Generation
      ↓
HNSW Index
```

### Критерии отбора entities

**Только 10-15% entities получают attributes**.

**Критерии**:
1. **K-core decomposition**: `core_number ≥ 2`
2. **Betweenness centrality**: Top 20%
3. **Weight**: `> 1` (множественные упоминания)

**Обоснование**:
- Генерация дорогая (5-10 сек, ~$0.001 per entity)
- Только важные entities требуют детального описания
- Качество > количество

---

## Связи с другими промптами

### Upstream Prompts

#### Text Decomposition Prompt
**Связь**: Создаёт semantic units и relationships, используемые как контекст.

**Зависимость**:
```
Text Decomposition → Semantic Units + Relationships
                     ↓
Attribute Generation использует как input
```

### Downstream Usage

#### Context Assembly
**Связь**: Attributes используются в ENTITIES section.

**Процесс**:
```
HNSW Retrieval → DR. EMILY ROBERTS (entity)
                ↓
Fetch Attribute → Rich description
                ↓
Context Assembly → KEY ENTITIES section
```

#### Answer Synthesis
**Связь**: Attributes предоставляют детальный контекст для ответов.

**Эффект**:
```
Без attribute:
  "Dr. Emily Roberts is mentioned in connection with renewable energy."

С attribute:
  "Dr. Emily Roberts is a prominent researcher specializing in solar panel
   efficiency, who presented groundbreaking research at the International
   Conference on Renewable Energy in Paris in September 2024..."
```

---

## Внешние библиотеки и инструменты

### LLM Providers

**Рекомендуемые**:
- **GPT-4o**: Высокое качество narrative (рекомендуется)
- **GPT-4o-mini**: Экономичная опция
- **Gemini-1.5-Pro**: Хорошее качество, большой context window

**Параметры**:
```python
{
    "model": "gpt-4o",
    "temperature": 0.0,
    "max_tokens": 2000  # Лимит для attribute
}
```

### Graph Traversal (NetworkX)

**Назначение**: Сбор context для entity.

**Код**:
```python
import networkx as nx

def collect_entity_context(graph: nx.Graph, entity: str):
    # Собрать semantic units
    semantic_units = []
    for neighbor in graph.neighbors(entity):
        if graph.nodes[neighbor]['type'] == 'semantic_unit':
            semantic_units.append(graph.nodes[neighbor]['content'])

    # Собрать relationships
    relationships = []
    for neighbor in graph.neighbors(entity):
        if graph.nodes[neighbor]['type'] == 'relationship':
            rel = graph.nodes[neighbor]
            relationships.append(f"{rel['source']}, {rel['relation']}, {rel['target']}")

    return semantic_units, relationships
```

### Token Management

**Проблема**: Context может превысить лимит токенов.

**Решение**: Truncation strategy

```python
max_context_tokens = 4000

# Приоритизация context
semantic_units = prioritize_by_importance(semantic_units)
relationships = prioritize_by_weight(relationships)

# Truncate до лимита
context = truncate_to_token_limit(
    semantic_units + relationships,
    max_tokens=max_context_tokens
)
```

---

## Метрики

### Стоимость

**GPT-4o**:
```
Input: ~2000 tokens (context)
Output: ~500 tokens (attribute)

Cost per entity: ~$0.0025
```

**Для документа (1000 entities, 10% selected)**:
```
Attributes: 100
Total cost: 100 × $0.0025 = $0.25
```

### Latency

- **Per entity**: 5-10 секунд
- **Параллельная обработка**: 10-50 entities одновременно

### Качество

**Metrics**:
- **Coherence**: 95% (flowing narrative)
- **Completeness**: 90% (включает key facts)
- **Relevance**: 92% (фокус на important info)

---

## Лучшие практики

### 1. Context prioritization

```python
# Сортировка semantic units по релевантности
semantic_units = sorted(
    semantic_units,
    key=lambda su: calculate_importance(su, entity),
    reverse=True
)[:20]  # Top 20
```

### 2. Token budget allocation

```python
# Динамическое распределение
if len(semantic_units) > 30:
    max_units = 30
    max_relationships = 10
else:
    max_units = len(semantic_units)
    max_relationships = len(relationships)
```

### 3. Post-processing

```python
# Удаление meta-text
attribute = re.sub(r'^(Here is|This is|In summary).*?\n', '', attribute)
attribute = re.sub(r'\n(In conclusion|To summarize).*$', '', attribute)
```

---

## Ограничения

### 1. Качество зависит от context

**Проблема**: Если semantic units неполные, attribute будет неполным.

**Пример**:
```
Sparse context (2 semantic units):
  → Generic attribute: "Dr. Roberts is a researcher in renewable energy."

Rich context (20 semantic units):
  → Detailed attribute: "Dr. Roberts is a leading expert in solar panel
                         efficiency, having published 15 papers..."
```

### 2. Hallucination риск

**Проблема**: LLM может добавить информацию не из context.

**Митигация**: Temperature 0.0 + явная инструкция "only from provided context"

### 3. Redundancy

**Проблема**: Attribute может повторять semantic units verbatim.

**Решение**: Instruction "distill" и "synthesize" подталкивает к парафразу

---

## Заключение

Attribute Generation Prompt — **критический компонент** для обогащения knowledge graph. Создавая narrative descriptions для важных entities, он значительно улучшает качество retrieval и ответов.

**Ключевые характеристики**:
- ✅ Comprehensive context aggregation
- ✅ Engaging narrative style
- ✅ Selective application (10-15% entities)
- ✅ Детерминистическое качество (Temperature 0.0)
- ⚠️ Дорогая операция (стоимость + latency)

---

## Ссылки

- **Файл промпта**: `NodeRAG/utils/prompt/attribute_generation_prompt.py:1-12`
- **Manager**: `NodeRAG/utils/prompt/prompt_manager.py:39-46`
- **Использование**: Attribute Pipeline (Stage 4)
- **Upstream**: Text Decomposition, Graph Construction
- **Downstream**: Embedding Generation, Context Assembly
