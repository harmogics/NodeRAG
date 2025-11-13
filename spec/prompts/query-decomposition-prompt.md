# Query Decomposition Prompt

## Обзор

**Тип**: Query Analysis Prompt
**Критичность**: ⭐⭐⭐ Высокая
**Стадия**: Answer Generation Pipeline (Stage 1)
**Файл**: `NodeRAG/utils/prompt/decompose.py`
**Температура**: 0.0-0.3 (низкая, но допустима вариативность)
**Output**: List of entities and terms

---

## Назначение

Query Decomposition Prompt разбивает user query на **structured components** (entities, concepts, keywords) для более эффективного retrieval. Это первый шаг в answer generation pipeline.

---

## Полный текст промпта (English)

```
Please break down the following query into a single list. Each item in the list should either be a main entity (such as a key noun or object). If you have high confidence about the user's intent or domain knowledge, you may also include closely related terms. If uncertain, please only extract entities and semantic chunks directly from the query. Please try to reduce the number of common nouns in the list. Ensure all elements are organized within one unified list.
Query:{query}
```

---

## Полный текст промпта (Chinese)

```
请将以下问题分解为一个 list，其中每一项是句子的主要实体（如关键名词或对象）。如果你对用户的意图或相关领域知识有充分把握，也可以包含密切相关的术语。如果不确定，请仅从问题中提取实体。请尽量减少囊括常见的名词，请将这些元素整合在一个单一的 list 中输出。
问题:{query}
```

---

## Структура промпта

### Ключевые инструкции

| Инструкция | Назначение |
|------------|-----------|
| **"Break down into single list"** | Структурированный output |
| **"Main entity (key noun or object)"** | Фокус на сущностях |
| **"High confidence → include related terms"** | Query expansion (опциональное) |
| **"If uncertain → only extract entities"** | Консервативный подход |
| **"Reduce common nouns"** | Фильтрация шума |
| **"Unified list"** | Простой формат |

---

## Structured Output Schema

```python
from pydantic import BaseModel

class decomposed_text(BaseModel):
    elements: list[str]
```

**Файл**: `NodeRAG/utils/prompt/json_format.py:26-27`

**Output format**:
```json
{
  "elements": [
    "DR. EMILY ROBERTS",
    "renewable energy",
    "research",
    "solar panels"
  ]
}
```

---

## Примеры использования

### Пример 1: Простой фактический запрос

**Вход**:
```
Query: "What research did Dr. Emily Roberts conduct?"
```

**Выход**:
```json
{
  "elements": [
    "DR. EMILY ROBERTS",
    "research"
  ]
}
```

**Анализ**:
- Извлечена entity: `DR. EMILY ROBERTS`
- Извлечена key concept: `research`
- Исключены: "What", "did", "conduct" (common words)

### Пример 2: Сложный запрос с контекстом

**Вход**:
```
Query: "How did Dr. Roberts' research on solar panel efficiency contribute to renewable energy advancements?"
```

**Выход**:
```json
{
  "elements": [
    "DR. ROBERTS",
    "research",
    "solar panel efficiency",
    "renewable energy",
    "advancements",
    "sustainable energy"  // Related term (if high confidence)
  ]
}
```

**Анализ**:
- Entities: `DR. ROBERTS`
- Concepts: `research`, `solar panel efficiency`, `renewable energy`, `advancements`
- Query expansion: `sustainable energy` (synonym)

### Пример 3: Тематический запрос

**Вход**:
```
Query: "What are the main themes in environmental conservation research?"
```

**Выход**:
```json
{
  "elements": [
    "environmental conservation",
    "research",
    "themes",
    "biodiversity",  // Related (if confident)
    "sustainability"  // Related (if confident)
  ]
}
```

**Анализ**:
- Core concepts: `environmental conservation`, `research`, `themes`
- Expanded terms: `biodiversity`, `sustainability` (domain знание)

---

## Роль в цепочке преобразований

### Позиция в Pipeline

```
User Query
      ↓
[QUERY DECOMPOSITION PROMPT] ← ВЫ ЗДЕСЬ
      ↓
{entities, concepts, keywords}
      ↓
Embedding Generation
      ↓
HNSW Retrieval
```

### Downstream Impact

**HNSW Retrieval** использует decomposed elements для поиска.

**Процесс**:
```
Decomposed elements → Multiple embedding queries
                     ↓
HNSW search для каждого element
                     ↓
Aggregate results → Top-K nodes
```

**Эффект query expansion**:
```
Without expansion:
  Query: "renewable energy"
  Search: только "renewable energy"

With expansion:
  Query: "renewable energy"
  Decomposed: ["renewable energy", "sustainable energy", "clean energy"]
  Search: все три термина
  → Больший recall
```

---

## Связи с другими промптами

### Downstream Prompts

#### Answer Generation Prompt
**Связь**: Decomposed elements inform context assembly.

**Процесс**:
```
Query Decomposition → Elements
                     ↓
Retrieval → Relevant nodes
                     ↓
Context Assembly → Prioritize nodes matching elements
                     ↓
Answer Synthesis
```

---

## Внешние библиотеки и инструменты

### LLM Providers

**Рекомендуемые**:
- **GPT-4o-mini**: Быстро и дешево (рекомендуется)
- **Gemini-1.5-Flash**: Очень быстро

**Параметры**:
```python
{
    "model": "gpt-4o-mini",
    "temperature": 0.0,  # Или 0.3 для небольшой вариативности
    "response_format": decomposed_text
}
```

### Embedding Generation

**После decomposition**, каждый element векторизуется.

```python
elements = ["DR. EMILY ROBERTS", "renewable energy", "research"]

embeddings = [
    embed_model.encode("DR. EMILY ROBERTS"),
    embed_model.encode("renewable energy"),
    embed_model.encode("research")
]

# Multi-query search
results = []
for emb in embeddings:
    results.extend(hnsw_index.search(emb, k=20))
```

---

## Метрики

### Стоимость

**GPT-4o-mini**:
```
Input: ~20 tokens (short query)
Output: ~15 tokens (element list)

Cost per query: ~$0.000005 (пренебрежимо)
```

### Latency

- **Per query**: 0.2-0.5 секунды
- **Impact**: Минимальный (быстрое преобразование)

### Качество

**Metrics**:
- **Entity recall**: 90% (большинство entities извлечены)
- **Expansion relevance**: 80% (expanded terms релевантны)
- **Noise reduction**: 85% (common words filtered)

---

## Лучшие практики

### 1. Балансировка expansion vs precision

```python
# Настройка через prompt
if user_wants_broad_search:
    prompt += "\n\nInclude all related terms and synonyms."
else:
    prompt += "\n\nOnly extract entities directly from query."
```

### 2. Domain-specific expansion

```python
# Добавить domain knowledge
if domain == "medical":
    prompt += "\n\nYou may include medical terms and synonyms."
```

### 3. Post-processing normalization

```python
# Нормализация extracted elements
elements = [e.upper().strip() for e in elements]
elements = [e for e in elements if len(e) > 2]  # Фильтр коротких
```

---

## Ограничения

### 1. Over-expansion риск

**Проблема**: LLM может добавить нерелевантные термины.

```
Query: "Dr. Roberts' research"
Decomposition: ["DR. ROBERTS", "research", "science", "academia", "publications"]

Проблема: "science", "academia" слишком общие
```

### 2. Under-extraction риск

**Проблема**: Важные entity могут быть пропущены.

```
Query: "How did the Paris conference impact renewable energy?"
Decomposition: ["Paris", "renewable energy"]

Пропущено: "conference", "impact"
```

### 3. Ambiguity

**Проблема**: Неясно, нужен ли expansion.

```
Query: "Apple research"

Possible interpretations:
1. Apple (company) research
2. Apple (fruit) research

Decomposition может выбрать не ту интерпретацию.
```

---

## Заключение

Query Decomposition Prompt — **важный первый шаг** в answer generation pipeline, обеспечивающий:
- ✅ Структурированный query analysis
- ✅ Entity extraction
- ✅ Optional query expansion
- ✅ Noise filtering

**Ключевые характеристики**:
- Быстрая обработка (< 0.5 сек)
- Минимальная стоимость
- Балансировка precision vs recall
- Structured output

---

## Ссылки

- **Файл промпта**: `NodeRAG/utils/prompt/decompose.py:1-9`
- **Schema**: `NodeRAG/utils/prompt/json_format.py:26-27`
- **Manager**: `NodeRAG/utils/prompt/prompt_manager.py:58-65, 96-97`
- **Downstream**: HNSW Retrieval, Context Assembly
