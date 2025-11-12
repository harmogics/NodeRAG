# LLM Агенты и Их Роли в NodeRAG

## Обзор

NodeRAG использует языковые модели в качестве специализированных агентов для различных задач обработки знаний. Каждый агент выполняет четко определенную функцию с использованием специфичных промптов и structured output.

---

## Архитектура LLM Системы

### Клиенты

**Файл**: `NodeRAG/LLM/LLM.py`

Поддерживаемые провайдеры:
- **OpenAI** (GPT-4, GPT-4o-mini, GPT-3.5)
- **Azure OpenAI**
- **Gemini** (Gemini Pro, Gemini Flash)

### Базовая Структура

```python
class LLM(LLMBase):
    - model_name: str
    - api_keys: str
    - config: ModelConfig (temperature, max_tokens)

    Methods:
    - predict(input: LLM_message) → LLMOutput
    - predict_async(input: LLM_message) → LLMOutput
```

### Конфигурация

```python
ModelConfig = {
    "max_tokens": 10000,
    "temperature": 0.0  # Детерминированность для extraction задач
}
```

### Error Handling

```python
@backoff.on_exception(
    backoff.expo,
    [RateLimitError, Timeout, APIConnectionError, JSONDecodeError],
    max_time=30,
    max_tries=4
)
```

**Стратегия**:
- Exponential backoff: 1s, 2s, 4s, 8s
- Максимум 4 попытки
- Таймаут 30 секунд
- Кеширование неудачных запросов

---

## Агент 1: Text Decomposition Agent

### Роль
Разбиение text units на семантические единицы с извлечением сущностей и связей.

### Файл Промпта
`NodeRAG/utils/prompt/text_decomposition.py`

### Задачи

1. **Semantic Segmentation**
   - Разделить текст на semantic units
   - Каждый unit = одно событие/факт/концепт

2. **Entity Extraction**
   - Извлечь все сущности из ОРИГИНАЛЬНОГО текста
   - Формат: UPPERCASE
   - Типы: люди, места, организации, времена, концепты

3. **Relationship Extraction**
   - Выявить связи между сущностями
   - Формат: "ENTITY_A, RELATION_TYPE, ENTITY_B"
   - Relation type = описательное предложение

### Промпт (English)

```python
text_decomposition_prompt = """
Goal: Given a text, segment it into multiple semantic units, each
containing detailed descriptions of specific events or activities.
Perform the following tasks:

1. Provide a summary for each semantic unit while retaining all crucial
   details relevant to the original context.

2. Extract all entities directly from the original text of each semantic
   unit, not from the paraphrased summary. Format each entity name in
   UPPERCASE. You should extract all entities including times, locations,
   people, organizations and all kinds of entities.

3. From the entities extracted in Step 2, list all relationships within
   the semantic unit and the corresponding original context in the form
   of string seperated by comma: "ENTITY_A, RELATION_TYPE, ENTITY_B".
   The RELATION_TYPE could be a descriptive sentence, while the entities
   involved in the relationship must come from the entity names extracted
   in Step 2. Please make sure the string contains three elements
   representing two entities and the relationship type.

Requirements:
1. Temporal Entities: Represent time entities based on the available
   details without filling in missing parts. Use specific formats based
   on what parts of the date or time are mentioned in the text.

Each semantic unit should be represented as a dictionary containing three
keys: semantic_unit (a paraphrased summary of each semantic unit),
entities (a list of entities extracted directly from the original text of
each semantic unit, formatted in UPPERCASE), and relationships (a list of
extracted relationship strings that contain three elements, where the
relationship type is a descriptive sentence). All these dictionaries
should be stored in a list to facilitate management and access.

Text:{text}
"""
```

### Пример Input/Output

**Input**:
```
In September 2024, Dr. Emily Roberts traveled to Paris to attend the
International Conference on Renewable Energy. During her visit, she
explored partnerships with several European companies and presented her
latest research on solar panel efficiency improvements.
```

**Output**:
```json
{
  "Output": [
    {
      "semantic_unit": "In September 2024, Dr. Emily Roberts attended
                        the International Conference on Renewable Energy
                        in Paris, where she presented her research on
                        solar panel efficiency improvements and explored
                        partnerships with European companies.",
      "entities": [
        "DR. EMILY ROBERTS",
        "2024-09",
        "PARIS",
        "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
        "EUROPEAN COMPANIES",
        "SOLAR PANEL EFFICIENCY"
      ],
      "relationships": [
        "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
        "DR. EMILY ROBERTS, explored partnerships with, EUROPEAN COMPANIES",
        "DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY"
      ]
    }
  ]
}
```

### Structured Output Schema

```json
{
  "Output": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "semantic_unit": {"type": "string"},
        "entities": {
          "type": "array",
          "items": {"type": "string"}
        },
        "relationships": {
          "type": "array",
          "items": {"type": "string"}
        }
      },
      "required": ["semantic_unit", "entities", "relationships"]
    }
  }
}
```

### Вызов

```python
# В Text_unit.text_decomposition()
prompt = config.prompt_manager.text_decomposition.format(text=self.raw_context)
json_format = config.prompt_manager.text_decomposition_json

input_data = {
    'query': prompt,
    'response_format': json_format
}

response = await config.API_client(input_data,
                                   cache_path=config.LLM_error_cache,
                                   meta_data={'text_hash_id': self.hash_id})
```

### Ключевые Характеристики

- **Temperature**: 0.0 (детерминированность)
- **Формат**: Structured JSON output
- **Параллелизм**: Все text units обрабатываются асинхронно
- **Ошибки**: Кешируются для повторной обработки

---

## Агент 2: Relationship Reconstruction Agent

### Роль
Исправление некорректно извлеченных relationships (не 3 элемента).

### Файл Промпта
`NodeRAG/utils/prompt/relationship_reconstraction.py`

### Задача

Преобразовать некорректный relationship в формат "source, relation, target".

### Промпт

```python
relationship_reconstraction_prompt = """
Given a relationship description that may not follow the standard format,
reconstruct it into three clear components:
1. Source entity
2. Relationship type
3. Target entity

Original relationship: {relationship}

Return in format:
{{
  "source": "SOURCE_ENTITY",
  "relationship": "relationship description",
  "target": "TARGET_ENTITY"
}}
"""
```

### Пример

**Input**:
```
["DR. EMILY ROBERTS", "attended conference in Paris"]
```

**Output**:
```json
{
  "source": "DR. EMILY ROBERTS",
  "relationship": "attended conference in",
  "target": "PARIS"
}
```

### Вызов

```python
# В Graph_pipeline.reconstruct_relationship()
if len(relationship) != 3:
    query = self.prompt_manager.relationship_reconstraction.format(
        relationship=relationship
    )
    json_format = self.prompt_manager.relationship_reconstraction_json
    input_data = {'query': query, 'response_format': json_format}
    response = await self.API_request(input_data)
    return [response['source'], response['relationship'], response['target']]
```

### Характеристики

- **Вызывается**: Только при некорректном формате
- **Retry**: До 3 раз с backoff
- **Fallback**: При неудаче relationship отбрасывается

---

## Агент 3: Attribute Generation Agent

### Роль
Создание детальных описаний для важных сущностей.

### Файл Промпта
`NodeRAG/utils/prompt/attribute_generation_prompt.py`

### Задача

Сгенерировать "character sketch" или "product description" для сущности на основе:
- Соседних Semantic Units
- Соседних Relationships

### Промпт (English)

```python
attribute_generation_prompt = """
Generate a concise summary of the given entity, capturing its essential
attributes and important relevant relationships. The summary should read
like a character sketch in a novel or a product description, providing
an engaging yet precise overview. Ensure the output only includes the
summary of the entity without any additional explanations or metadata.
The length must not exceed 2000 words but can be shorter if the input
material is limited. Focus on distilling the most important insights
with a smooth narrative flow, highlighting the entity's core traits and
meaningful connections.

Entity: {entity}
Related Semantic Units: {semantic_units}
Related Relationships: {relationships}
"""
```

### Пример Input/Output

**Input**:
```
Entity: DR. EMILY ROBERTS

Related Semantic Units:
- In September 2024, Dr. Emily Roberts attended the International Conference...
- Dr. Emily Roberts published research on solar panel efficiency...
- Dr. Emily Roberts leads a team at the European Research Institute...

Related Relationships:
- DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY
- DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE
- DR. EMILY ROBERTS, collaborates with, EUROPEAN COMPANIES
```

**Output**:
```
Dr. Emily Roberts is a prominent researcher in the field of renewable
energy, with a particular focus on enhancing solar panel efficiency. In
September 2024, she played a key role at the International Conference on
Renewable Energy in Paris, where she presented groundbreaking research on
photovoltaic system improvements. As a leading scientist at the European
Research Institute, Dr. Roberts has established numerous partnerships with
European companies to advance sustainable energy technologies. Her work is
characterized by a commitment to bridging academic research with practical
industrial applications, making her a central figure in the renewable
energy community.
```

### Вызов

```python
# В Attribution_generation_pipeline.generate_attribution()
entity = self.mapper.get(node, 'context')

semantic_neighbours = '\n'.join([
    self.mapper.get(neighbour, 'context')
    for neighbour in self.G.neighbors(node)
    if self.G.nodes[neighbour]['type'] == 'semantic_unit'
])

relationship_neighbours = '\n'.join([
    self.mapper.get(neighbour, 'context')
    for neighbour in self.G.neighbors(node)
    if self.G.nodes[neighbour]['type'] == 'relationship'
])

query = self.prompt_manager.attribute_generation.format(
    entity=entity,
    semantic_units=semantic_neighbours,
    relationships=relationship_neighbours
)

response = await self.API_client({'query': query})
```

### Token Management

**Проблема**: Слишком много соседей → превышение token limit

**Решение**: Приоритизация по важности

```python
def get_important_neibours_material(node):
    sorted_neighbours = SortedDict()

    # Сортировка по весу соседей соседей
    for neighbour in G.neighbors(node):
        value = sum(G.nodes[n]['weight'] for n in G.neighbors(neighbour))
        sorted_neighbours[neighbour] = value

    # Добавление в порядке убывания важности
    query = ''
    for neighbour in reversed(sorted_neighbours):
        temp_query = prompt.format(...)
        if not token_counter.token_limit(temp_query):
            query = temp_query
        else:
            break

    return query
```

### Характеристики

- **Style**: Narrative, engaging
- **Length**: До 2000 слов
- **Focus**: Core traits + meaningful connections
- **Context**: Агрегация из графа
- **Frequency**: Только для важных узлов

---

## Агент 4: Community Summary Agent

### Роль
Извлечение high-level concepts из community clusters.

### Файл Промпта
`NodeRAG/utils/prompt/community_summary.py`

### Задача

Из кластера текстов извлечь:
- Концепты
- Темы
- Теории
- Потенциальные воздействия
- Ключевые инсайты

### Промпт (English)

```python
community_summary = """
You will receive a set of text data from the same cluster. Your task is
to extract distinct categories of high-level information, such as
concepts, themes, relevant theories, potential impacts, and key insights.
Each piece of information should include a concise title and a
corresponding description, reflecting the unique perspectives within the
text cluster.

Please do not attempt to include all possible information; instead,
select the elements that have the most significance and diversity in this
cluster. Avoid redundant information—if there are highly similar elements,
combine them into a single, comprehensive entry. Ensure that the
high-level information reflects the varied dimensions within the text,
providing a well-rounded overview.

Clustered text data:
{content}
"""
```

### Structured Output Schema

```json
{
  "high_level_elements": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "title": {"type": "string"},
        "description": {"type": "string"}
      },
      "required": ["title", "description"]
    }
  }
}
```

### Пример Input/Output

**Input** (clustered texts):
```
Clustered text data:
- Dr. Emily Roberts attended renewable energy conference in Paris
- John Miller conducted research in Amazon rainforest on deforestation
- Both contribute to environmental conservation efforts
- Research on solar panel efficiency improvements
- Documentation of new species and wildlife impacts
```

**Output**:
```json
{
  "high_level_elements": [
    {
      "title": "Renewable Energy Research and Innovation",
      "description": "Cutting-edge research in sustainable energy
                      technologies, focusing on solar panel efficiency
                      improvements and international collaboration through
                      scientific conferences."
    },
    {
      "title": "Biodiversity Conservation and Ecosystem Research",
      "description": "Field research documenting species diversity and
                      analyzing environmental impacts of deforestation in
                      critical ecosystems like the Amazon Rainforest."
    },
    {
      "title": "Environmental Conservation Through Science",
      "description": "The intersection of various scientific disciplines
                      working towards environmental protection, combining
                      technological innovation with ecological preservation."
    }
  ]
}
```

### Вызов

```python
# В Community_summary.generate_community_summary()
content = '\n'.join([
    self.mapper.get(node, 'context')
    for node in self.used_unit
])

query = self.prompt.community_summary.format(content=content)

input_data = {
    'query': query,
    'response_format': self.prompt.high_level_element_json
}

self.response = await self.client(input_data)
```

### Community Selection

**used_unit** включает:
- Semantic units в community
- Attributes узлов в community
- Attribute neighbors сущностей в community

### Token Management

Аналогично Attribute Generation:
- Сортировка по важности
- Добавление до достижения token limit

### Характеристики

- **Избирательность**: Не все инфо, только значимое
- **Diversity**: Разнообразие аспектов
- **No redundancy**: Объединение похожих элементов
- **Multi-dimensional**: Разные измерения темы

---

## Embedding Agent

### Роль
Генерация векторных представлений для узлов.

### Модели

**OpenAI**:
```python
class OpenAI_Embedding(LLM):
    - model_name: "text-embedding-3-small" или "text-embedding-ada-002"
    - dimension: 1536 (ada-002) или 512/1536/3072 (3-small)
```

**Gemini**:
```python
class Gemini_Embedding(LLM):
    - model_name: "text-embedding-004"
    - dimension: 768
```

### Вызов

```python
# В Embedding_pipeline.get_embeddings()
context_dict = {
    hash_id: mapper.get(hash_id, 'context')
    for hash_id in node_ids
}

embedding_input = list(context_dict.values())
ids = list(context_dict.keys())

embedding_output = await self.embedding_client(
    embedding_input,
    cache_path=self.config.LLM_error_cache,
    meta_data={'ids': ids}
)
```

### Batch Processing

```python
batch_size = config.embedding_batch_size  # Default: 100

for i in range(0, len(ids), batch_size):
    batch = ids[i:i+batch_size]
    await process_batch(batch)
```

### Характеристики

- **Batch mode**: Эффективная обработка
- **Async**: Параллельные запросы
- **Filtering**: Пропуск пустых context
- **Error handling**: Кеширование и retry

---

## Роли Агентов: Сводная Таблица

| Агент                    | Роль                        | Input                | Output                      | Temperature |
|--------------------------|-----------------------------|--------------------- |-----------------------------|-------------|
| Text Decomposition       | Semantic extraction         | Text Unit            | SU + Entities + Relations   | 0.0         |
| Relationship Recon.      | Format correction           | Malformed relation   | Corrected relation triplet  | 0.0         |
| Attribute Generation     | Entity description          | Entity + Neighbours  | Narrative description       | 0.0         |
| Community Summary        | Concept extraction          | Cluster texts        | High-level concepts         | 0.0         |
| Embedding                | Vector representation       | Any text             | Embedding vector            | N/A         |

---

## Промпт Менеджер

**Файл**: `NodeRAG/utils/prompt/prompt_manager.py`

### Структура

```python
class PromptManager:
    # Промпты
    - text_decomposition: str
    - relationship_reconstraction: str
    - attribute_generation: str
    - community_summary: str

    # JSON схемы
    - text_decomposition_json: Schema
    - relationship_reconstraction_json: Schema
    - high_level_element_json: Schema

    # Языковые версии
    - text_decomposition_Chinese: str
    - attribute_generation_Chinese: str
    - community_summary_Chinese: str
```

### Использование

```python
config.prompt_manager.text_decomposition.format(text="...")
config.prompt_manager.text_decomposition_json  # Schema для OpenAI
```

---

## Structured Output

### OpenAI Implementation

```python
response = client.beta.chat.completions.parse(
    model=model_name,
    messages=messages,
    response_format=json_schema
)
json_response = response.choices[0].message.parsed.model_dump_json()
```

### Gemini Implementation

```python
config = genai.types.GenerateContentConfig(
    response_mime_type="application/json",
    response_schema=json_schema
)
response = client.models.generate_content(contents=messages, config=config)
json_response = json.loads(response.text)
```

### Преимущества

- **Гарантированный формат**: Всегда валидный JSON
- **Валидация**: Автоматическая проверка schema
- **Парсинг**: Не нужен ручной парсинг
- **Ошибки**: Меньше ошибок форматирования

---

## Error Handling и Retry Logic

### Стратегия

```python
@backoff.on_exception(
    backoff.expo,
    [RateLimitError, Timeout, APIConnectionError, JSONDecodeError],
    max_time=30,
    max_tries=4
)
async def _create_completion_async(...):
    # LLM call
```

### Error Caching

**Файл**: `LLM_error.jsonl`

**Формат**:
```json
{
  "input_data": {
    "query": "...",
    "response_format": {...}
  },
  "meta_data": {
    "text_hash_id": "...",
    "text_id": "..."
  },
  "error": "Error message",
  "timestamp": "2024-11-12T15:30:00"
}
```

### Rerun Process

```python
# В pipeline.rerun()
with open(config.LLM_error_cache, 'r') as f:
    error_cache = [json.loads(line) for line in f]

tasks = []
for cached_item in error_cache:
    input_data = cached_item['input_data']
    meta_data = cached_item['meta_data']
    tasks.append(retry_request(input_data, meta_data))

await asyncio.gather(*tasks)
```

---

## Оптимизация и Best Practices

### 1. Параллелизм

```python
# Обработка всех text units параллельно
async_tasks = [
    text.text_decomposition(config)
    for text in text_units
]
await asyncio.gather(*async_tasks)
```

### 2. Batching для Embeddings

```python
# Batch = 100 узлов
for i in range(0, len(nodes), 100):
    batch = nodes[i:i+100]
    await process_batch(batch)
```

### 3. Token Management

- Проверка token count перед LLM вызовом
- Приоритизация важного контекста
- Truncation при превышении

### 4. Caching

- Кеширование успешных результатов
- Кеширование ошибок для retry
- Промежуточные JSONL файлы

### 5. Детерминированность

- Temperature = 0.0 для extraction
- Structured output для consistency
- Retry с теми же параметрами

---

## Многоязычность

Система поддерживает:
- English prompts (по умолчанию)
- Chinese prompts (альтернатива)

**Переключение**:
```python
if config.language == 'Chinese':
    prompt = prompt_manager.text_decomposition_Chinese
else:
    prompt = prompt_manager.text_decomposition
```

**Примечание**: Примеры в промптах адаптированы под язык.
