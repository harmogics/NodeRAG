# NodeRAG Language Model Prompts - Overview

## Обзор

Этот документ представляет comprehensive overview всех language model prompts, используемых в системе NodeRAG. Промпты являются **критическими компонентами** системы, определяющими качество семантических преобразований на каждом этапе pipeline.

---

## Таксономия промптов

### По назначению

| Категория | Промпты | Назначение |
|-----------|---------|-----------|
| **Extraction** | Text Decomposition | Извлечение структурированных элементов из текста |
| **Error Correction** | Relationship Reconstruction | Исправление некорректных форматов |
| **Aggregation** | Attribute Generation | Синтез информации о сущностях |
| **Abstraction** | Community Summary | Извлечение high-level концепций |
| **Query Analysis** | Query Decomposition | Разбор пользовательского запроса |
| **Synthesis** | Answer Generation | Генерация финального ответа |
| **Utility** | Translation | Перевод промптов на другие языки |

---

## Полный список промптов

### Индексационные промпты (Indexing)

| № | Промпт | Файл | Критичность | Temperature | Output Type |
|---|--------|------|-------------|-------------|-------------|
| 1 | **Text Decomposition** | `text_decomposition.py` | ⭐⭐⭐⭐⭐ | 0.0 | Structured JSON |
| 2 | **Relationship Reconstruction** | `relationship_reconstraction.py` | ⭐⭐ | 0.0 | Structured JSON |
| 3 | **Attribute Generation** | `attribute_generation_prompt.py` | ⭐⭐⭐⭐⭐ | 0.0 | Narrative text |
| 4 | **Community Summary** | `community_summary.py` | ⭐⭐⭐⭐⭐ | 0.0 | Structured JSON |

### Поисково-генеративные промпты (Answer Generation)

| № | Промпт | Файл | Критичность | Temperature | Output Type |
|---|--------|------|-------------|-------------|-------------|
| 5 | **Query Decomposition** | `decompose.py` | ⭐⭐⭐ | 0.0-0.3 | Structured JSON |
| 6 | **Answer Generation** | `answer.py` | ⭐⭐⭐⭐⭐ | 0.7-1.0 | Natural language |

### Вспомогательные промпты

| № | Промпт | Файл | Назначение |
|---|--------|------|-----------|
| 7 | **Translation** | `translation.py` | Перевод промптов на любой язык |

---

## Детальное описание промптов

### 1. Text Decomposition Prompt ⭐⭐⭐⭐⭐

**Файл документации**: [`text-decomposition-prompt.md`](./text-decomposition-prompt.md)

**Назначение**: Центральный промпт системы, выполняющий декомпозицию текста на структурированные элементы.

**Входы**:
- Text Unit (~1048 токенов)

**Выходы**:
```json
{
  "Output": [
    {
      "semantic_unit": "Паратраз атомарного факта",
      "entities": ["ENTITY1", "ENTITY2", ...],
      "relationships": ["ENT1, relation, ENT2", ...]
    }
  ]
}
```

**Ключевые операции**:
1. Segmentation (text → atomic units)
2. Paraphrasing (сохранение деталей)
3. Entity extraction (NER + normalization)
4. Relationship extraction (triplets)
5. Temporal normalization

**Стадия**: Indexing Pipeline (Stage 2)
**Селективность**: 100% Text Units
**Стоимость**: ~$0.0009 per Text Unit
**Latency**: 2-5 секунд

---

### 2. Relationship Reconstruction Prompt ⭐⭐

**Файл документации**: [`relationship-reconstruction-prompt.md`](./relationship-reconstruction-prompt.md)

**Назначение**: Вспомогательный промпт для исправления некорректно форматированных relationships.

**Входы**:
- Malformed relationship string

**Выходы**:
```json
{
  "source": "ENTITY_A",
  "relationship": "relation_type",
  "target": "ENTITY_B"
}
```

**Триггер**: Когда relationship не содержит ровно 3 элемента
**Стадия**: Graph Construction (Stage 3, on-demand)
**Частота**: ~5% relationships
**Стоимость**: ~$0.00001 per relationship
**Latency**: 0.5-1 секунда

---

### 3. Attribute Generation Prompt ⭐⭐⭐⭐⭐

**Файл документации**: [`attribute-generation-prompt.md`](./attribute-generation-prompt.md)

**Назначение**: Создание comprehensive narrative descriptions для важных сущностей.

**Входы**:
- Entity name
- Related Semantic Units (контекст)
- Related Relationships (связи)

**Выходы**:
- Narrative description (100-2000 слов)
- Стиль: "character sketch" или "product description"

**Ключевые операции**:
1. Context aggregation (сбор всех упоминаний)
2. Synthesis (объединение в narrative)
3. Prioritization (выбор важного)
4. Style transformation (facts → prose)

**Стадия**: Attribute Pipeline (Stage 4)
**Селективность**: 10-15% entities (K-core + Betweenness + Weight > 1)
**Стоимость**: ~$0.0025 per entity
**Latency**: 5-10 секунд

---

### 4. Community Summary Prompt ⭐⭐⭐⭐⭐

**Файл документации**: [`community-summary-prompt.md`](./community-summary-prompt.md)

**Назначение**: Извлечение high-level themes и concepts из semantic communities.

**Входы**:
- Clustered text data (semantic units + attributes в community)

**Выходы**:
```json
{
  "high_level_elements": [
    {
      "title": "Concise theme title (2-5 words)",
      "description": "Comprehensive description (100-200 words)"
    }
  ]
}
```

**Ключевые операции**:
1. Thematic extraction (recurring patterns)
2. Abstraction (facts → concepts)
3. Diversity enforcement (non-redundant themes)
4. Synthesis (comprehensive overview)

**Стадия**: Summary Pipeline (Stage 6)
**Селективность**: Для каждой community
**Стоимость**: ~$0.0045 per community
**Latency**: 5-10 секунд

---

### 5. Query Decomposition Prompt ⭐⭐⭐

**Файл документации**: [`query-decomposition-prompt.md`](./query-decomposition-prompt.md)

**Назначение**: Разбор user query на structured components для эффективного retrieval.

**Входы**:
- User query (естественный язык)

**Выходы**:
```json
{
  "elements": [
    "main entity 1",
    "concept 1",
    "related term 1"
  ]
}
```

**Ключевые операции**:
1. Entity extraction
2. Concept identification
3. Query expansion (опционально)
4. Noise filtering

**Стадия**: Answer Pipeline (Stage 1)
**Селективность**: Каждый query
**Стоимость**: ~$0.000005 per query (пренебрежимо)
**Latency**: 0.2-0.5 секунды

---

### 6. Answer Generation Prompt ⭐⭐⭐⭐⭐

**Файл документации**: [`answer-generation-prompt.md`](./answer-generation-prompt.md)

**Назначение**: Финальный synthesis — генерация natural language ответа из retrieved context.

**Входы**:
- User query
- Structured context (THEMES + ENTITIES + RELATIONSHIPS + FACTS)

**Выходы**:
- Natural language answer (multiple paragraphs)

**Ключевые операции**:
1. Information fusion (объединение фактов)
2. Contextualization (добавление context)
3. Narrative construction (flowing prose)
4. Factual grounding (no hallucination)

**Стадия**: Answer Pipeline (Stage 5)
**Селективность**: Каждый query
**Temperature**: 0.7-1.0 (для naturalness)
**Стоимость**: ~$0.015 per query (самый дорогой)
**Latency**: 2-5 секунд

---

## Цепочки преобразований

### Indexing Pipeline

```
📄 Document
     ↓
  Semantic Chunking
     ↓
📝 Text Units
     ↓
[1. TEXT DECOMPOSITION PROMPT] ⭐⭐⭐⭐⭐
     ↓
{semantic_units, entities, relationships}
     ↓
  Graph Construction
     ↓ (5% relationships)
[2. RELATIONSHIP RECONSTRUCTION PROMPT] ⭐⭐
     ↓
🕸️ Knowledge Graph
     ├─→ Important Entity Selection (10-15%)
     │        ↓
     │   [3. ATTRIBUTE GENERATION PROMPT] ⭐⭐⭐⭐⭐
     │        ↓
     │   📋 Attributes
     │
     └─→ Community Detection
              ↓
         [4. COMMUNITY SUMMARY PROMPT] ⭐⭐⭐⭐⭐
              ↓
         🎯 High-Level Elements
              ↓
         Embedding Generation
              ↓
         📊 HNSW Index
```

### Answer Generation Pipeline

```
❓ User Query
     ↓
[5. QUERY DECOMPOSITION PROMPT] ⭐⭐⭐
     ↓
{entities, concepts, keywords}
     ↓
  Embedding + HNSW Retrieval
     ↓
  Personalized PageRank
     ↓
  Context Assembly
     ↓
📋 Structured Context
     ↓
[6. ANSWER GENERATION PROMPT] ⭐⭐⭐⭐⭐
     ↓
✅ Natural Language Answer
```

---

## Сравнительная таблица промптов

### По ключевым характеристикам

| Промпт | Критичность | Temp | Детерминизм | Output | Стоимость | Latency |
|--------|-------------|------|-------------|--------|-----------|---------|
| Text Decomposition | ⭐⭐⭐⭐⭐ | 0.0 | Полный | JSON | $0.0009 | 2-5s |
| Relationship Reconstruction | ⭐⭐ | 0.0 | Полный | JSON | $0.00001 | 0.5-1s |
| Attribute Generation | ⭐⭐⭐⭐⭐ | 0.0 | Полный | Text | $0.0025 | 5-10s |
| Community Summary | ⭐⭐⭐⭐⭐ | 0.0 | Полный | JSON | $0.0045 | 5-10s |
| Query Decomposition | ⭐⭐⭐ | 0.0-0.3 | Высокий | JSON | $0.000005 | 0.2-0.5s |
| Answer Generation | ⭐⭐⭐⭐⭐ | 0.7-1.0 | Средний | Text | $0.015 | 2-5s |

### По семантическим операциям

| Промпт | Extraction | Normalization | Abstraction | Aggregation | Synthesis | Generation |
|--------|-----------|---------------|-------------|-------------|-----------|------------|
| Text Decomposition | ✅✅ | ✅✅ | ⚪ | ⚪ | ⚪ | ⚪ |
| Relationship Reconstruction | ✅ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ |
| Attribute Generation | ⚪ | ⚪ | ⚪ | ✅✅ | ✅✅ | ⚪ |
| Community Summary | ✅ | ⚪ | ✅✅ | ⚪ | ✅✅ | ⚪ |
| Query Decomposition | ✅ | ✅ | ⚪ | ⚪ | ⚪ | ⚪ |
| Answer Generation | ⚪ | ⚪ | ⚪ | ⚪ | ✅✅ | ✅✅ |

✅✅ = Основная операция
✅ = Вспомогательная операция
⚪ = Не применяется

---

## Внешние библиотеки и инструменты

### LLM Providers

#### OpenAI
**Модели**:
- **GPT-4o**: Рекомендуется для всех критических промптов
- **GPT-4o-mini**: Экономичная опция для менее критичных
- **GPT-4-turbo**: Баланс скорости и качества

**Использование**:
```python
from openai import OpenAI

client = OpenAI(api_key=api_key)
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.0,
    response_format={"type": "json_schema", "json_schema": schema}
)
```

**Structured Output**: ✅ (через `response_format`)

#### Google Gemini
**Модели**:
- **Gemini-1.5-Pro**: Высокое качество, огромный context window (1M tokens)
- **Gemini-1.5-Flash**: Быстрая обработка

**Использование**:
```python
import google.generativeai as genai

model = genai.GenerativeModel('gemini-1.5-pro')
response = model.generate_content(
    prompt,
    generation_config={
        'temperature': 0.0,
        'response_mime_type': 'application/json',
        'response_schema': schema
    }
)
```

**Structured Output**: ✅ (через `response_schema`)

### Structured Output Library

#### Pydantic
**Версия**: 2.x
**Файл**: `NodeRAG/utils/prompt/json_format.py`

**Schemas**:
```python
from pydantic import BaseModel

# Text Decomposition
class semantic_group(BaseModel):
    semantic_unit: str
    entities: list[str]
    relationships: list[str]

# Relationship Reconstruction
class relationship_reconstraction(BaseModel):
    source: str
    relationship: str
    target: str

# Community Summary
class elements(BaseModel):
    title: str
    description: str

class High_level_element(BaseModel):
    high_level_elements: list[elements]

# Query Decomposition
class decomposed_text(BaseModel):
    elements: list[str]
```

**Преимущества**:
- Автоматическая валидация JSON
- Type safety
- Детальные error messages
- Интеграция с LLM APIs

### Prompt Management

#### prompt_manager
**Файл**: `NodeRAG/utils/prompt/prompt_manager.py`

**Функциональность**:
- Мультиязычность (English, Chinese, + автоперевод)
- Централизованное управление
- Lazy loading
- Access к schemas

**Использование**:
```python
from NodeRAG.utils.prompt.prompt_manager import prompt_manager

manager = prompt_manager(language="English")

# Получить промпты
text_decomp = manager.text_decomposition
attr_gen = manager.attribute_generation
answer = manager.answer

# Получить schemas
text_decomp_schema = manager.text_decomposition_json
high_level_schema = manager.high_level_element_json
```

**Автоматический перевод**:
```python
# Для неподдерживаемых языков автоматически переводит через LLM
manager = prompt_manager(language="Spanish")
prompt = manager.text_decomposition  # Автоперевод с English
```

### Token Counting

#### tiktoken
**Файл**: `NodeRAG/utils/token_utils.py`

**Использование**:
```python
import tiktoken

tokenizer = tiktoken.encoding_for_model("gpt-4o")
token_count = len(tokenizer.encode(text))
```

**Роль**:
- Проверка размера inputs
- Оценка стоимости API calls
- Управление token budgets

---

## Стоимость и производительность

### Стоимость по этапам

**Для документа (10,000 токенов = ~10 Text Units)**:

| Этап | Промпт | Количество вызовов | Стоимость per call | Итого |
|------|--------|-------------------|-------------------|-------|
| Stage 2 | Text Decomposition | 10 | $0.0009 | $0.009 |
| Stage 3 | Relationship Reconstruction | ~0.5 (5%) | $0.00001 | $0.000005 |
| Stage 4 | Attribute Generation | ~15 (15% entities) | $0.0025 | $0.0375 |
| Stage 6 | Community Summary | ~10 communities | $0.0045 | $0.045 |
| **Indexing Total** | | | | **$0.09** |

**Для query**:

| Этап | Промпт | Стоимость |
|------|--------|-----------|
| Stage 1 | Query Decomposition | $0.000005 |
| Stage 5 | Answer Generation | $0.015 |
| **Answer Total** | | **$0.015** |

**Общая стоимость**:
- **Indexing**: ~$0.09 per document (10K tokens)
- **Answer**: ~$0.015 per query

### Latency по этапам

**Indexing (последовательно)**:
```
Text Decomposition: 10 × 3s = 30s
Attribute Generation: 15 × 7s = 105s
Community Summary: 10 × 7s = 70s

Total: ~205s (~3.5 minutes) per document
```

**Indexing (параллельно)**:
```
With batching (10 parallel):
  Text Decomposition: ~5s
  Attribute Generation: ~15s
  Community Summary: ~10s

Total: ~30s per document
```

**Answer Generation**:
```
Query Decomposition: 0.3s
Retrieval (HNSW + PPR): 0.5s
Context Assembly: 0.2s
Answer Generation: 3s

Total: ~4s per query
```

---

## Лучшие практики

### 1. Temperature настройка

```python
# Indexing промпты: Temperature 0.0 (детерминизм)
temperatures = {
    'text_decomposition': 0.0,
    'relationship_reconstruction': 0.0,
    'attribute_generation': 0.0,
    'community_summary': 0.0,
}

# Answer generation: Temperature 0.7 (naturalness)
answer_temperature = 0.7
```

### 2. Batch processing

```python
# Батчевая обработка Text Units
async def process_batch(text_units):
    tasks = [decompose_text(unit) for unit in text_units]
    results = await asyncio.gather(*tasks)
    return results
```

### 3. Error handling

```python
from pydantic import ValidationError

try:
    result = text_decomposition(**response_json)
except ValidationError as e:
    # Retry с более явными инструкциями
    logger.warning(f"Validation error: {e}")
    result = retry_with_clarification(prompt, error=e)
```

### 4. Мониторинг качества

```python
# Логирование метрик для каждого промпта
metrics = {
    'prompt_name': 'text_decomposition',
    'success': True,
    'latency': 2.5,
    'tokens_used': {'input': 1048, 'output': 1200},
    'cost': 0.0009,
    'validation_errors': 0
}
log_prompt_metrics(metrics)
```

---

## Мультиязычность

### Поддерживаемые языки

**Нативно**:
- English
- Chinese (中文)

**Через автоперевод**:
- Любой язык через Translation Prompt

### Translation Prompt

**Файл**: `NodeRAG/utils/prompt/translation.py`

**Текст**:
```
Goal: Translate the given prompt into {language}.
You are provided with a prompt in English. Your task is to translate the
prompt from English to {language}. Please ensure the translation is accurate
and maintains the original meaning, context and format.
original prompt text:{prompt}
Output:
```

**Использование**:
```python
manager = prompt_manager(language="Spanish")
# Автоматически переведёт все промпты на Spanish
```

---

## Troubleshooting

### Проблема 1: Hallucination в ответах

**Симптомы**: Answer содержит информацию не из context.

**Решения**:
1. Понизить temperature (0.5 вместо 0.7)
2. Усилить инструкцию "do not fabricate"
3. Post-processing fact checking

### Проблема 2: Некорректный JSON format

**Симптомы**: ValidationError от Pydantic.

**Решения**:
1. Использовать structured output API (OpenAI, Gemini)
2. Retry с примером правильного формата
3. Post-processing JSON fixing

### Проблема 3: Неполные entity extractions

**Симптомы**: Text Decomposition пропускает entities.

**Решения**:
1. Добавить few-shot примеры в промпт
2. Использовать более мощную модель (GPT-4o вместо GPT-4o-mini)
3. Post-processing с NER библиотекой (spaCy)

### Проблема 4: Высокая стоимость

**Симптомы**: Indexing слишком дорогой.

**Решения**:
1. Использовать GPT-4o-mini вместо GPT-4o
2. Уменьшить селективность Attribute Generation (5% вместо 15%)
3. Оптимизировать chunk_size для меньшего количества Text Units

---

## Заключение

Система промптов NodeRAG представляет собой **тщательно спроектированную архитектуру** для преобразования неструктурированных документов в структурированные знания и генерации точных ответов.

**Ключевые принципы**:
1. **Детерминизм** для indexing (Temperature 0.0)
2. **Naturalness** для answer generation (Temperature 0.7)
3. **Structured output** через Pydantic
4. **Мультиязычность** через prompt manager
5. **Селективность** для дорогих операций (Attribute Gen, Community Sum)

**Критические промпты** (⭐⭐⭐⭐⭐):
- Text Decomposition (основа графа)
- Attribute Generation (enrichment)
- Community Summary (abstraction)
- Answer Generation (user experience)

**Качество системы = f(Prompt Quality, LLM Quality, Context Quality)**

---

## Дополнительные ресурсы

### Документация промптов

- [Text Decomposition Prompt](./text-decomposition-prompt.md)
- [Relationship Reconstruction Prompt](./relationship-reconstruction-prompt.md)
- [Attribute Generation Prompt](./attribute-generation-prompt.md)
- [Community Summary Prompt](./community-summary-prompt.md)
- [Query Decomposition Prompt](./query-decomposition-prompt.md)
- [Answer Generation Prompt](./answer-generation-prompt.md)

### Общая документация

- [Document Processing Pipeline](../document-processing-pipeline.md)
- [Indexing Transformations](../transform/indexing-transformations.md)
- [Answer Generation Transformations](../transform/answer-generation-transformations.md)
- [Transformation Chains](../transform/transformation-chains.md)
- [Semantic Characteristics](../transform/semantic-characteristics.md)

### Исходный код

- **Промпты**: `NodeRAG/utils/prompt/`
- **Schemas**: `NodeRAG/utils/prompt/json_format.py`
- **Manager**: `NodeRAG/utils/prompt/prompt_manager.py`
- **LLM API**: `NodeRAG/LLM/LLM_state.py`
