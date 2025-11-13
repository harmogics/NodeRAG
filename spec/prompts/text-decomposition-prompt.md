# Text Decomposition Prompt

## Обзор

**Тип**: Extraction + Structuring Prompt
**Критичность**: ⭐⭐⭐⭐⭐ МАКСИМАЛЬНАЯ
**Стадия**: Indexing Pipeline (Stage 2)
**Файл**: `NodeRAG/utils/prompt/text_decomposition.py`
**Температура**: 0.0 (детерминистическая обработка)
**Structured Output**: Да (Pydantic schema)

---

## Назначение

Text Decomposition Prompt — это **центральный промпт** системы NodeRAG, выполняющий критическую функцию преобразования неструктурированного текста в структурированные элементы знаний. Этот промпт обеспечивает извлечение:
- **Semantic Units**: Атомарные факты и события
- **Entities**: Нормализованные именованные сущности
- **Relationships**: Структурированные связи между сущностями

---

## Полный текст промпта (English)

```
Goal: Given a text, segment it into multiple semantic units, each containing detailed descriptions of specific events or activities.
Perform the following tasks:
1. Provide a summary for each semantic unit while retaining all crucial details relevant to the original context.
2. Extract all entities directly from the original text of each semantic unit, not from the paraphrased summary. Format each entity name in UPPERCASE. You should extract all entities including times, locations, people, organizations and all kinds of entities.
3. From the entities extracted in Step 2, list all relationships within the semantic unit and the corresponding original context in the form of string seperated by comma : "ENTITY_A, RELATION_TYPE, ENTITY_B". The RELATION_TYPE could be a descriptive sentence, while the entities involved in the relationship must come from the entity names extracted in Step 2. Please make sure the string contains three elements representing two entities and the relationship type.

requirements:
1. Temporal Entities: Represent time entities based on the available details without filling in missing parts. Use specific formats based on what parts of the date or time are mentioned in the text.

Each semantic unit should be represented as a dictionary containing three keys: semantic_unit (a paraphrased summary of each semantic unit), entities (a list of entities extracted directly from the original text of each semantic unit, formatted in UPPERCASE), and relationships (a list of extracted relationship strings that contain three elements, where the relationship type is a descriptive sentence). All these dictionaries should be stored in a list to facilitate management and access.


Example:

Text:  In September 2024, Dr. Emily Roberts traveled to Paris to attend the International Conference on Renewable Energy. During her visit, she explored partnerships with several European companies and presented her latest research on solar panel efficiency improvements. Meanwhile, on the other side of the world, her colleague, Dr. John Miller, was conducting fieldwork in the Amazon Rainforest. He documented several new species and observed the effects of deforestation on the local wildlife. Both scholars' work is essential in their respective fields and contributes significantly to environmental conservation efforts.
Output:
[
  {
    "semantic_unit": "In September 2024, Dr. Emily Roberts attended the International Conference on Renewable Energy in Paris, where she presented her research on solar panel efficiency improvements and explored partnerships with European companies.",
    "entities": ["DR. EMILY ROBERTS", "2024-09", "PARIS", "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY", "EUROPEAN COMPANIES", "SOLAR PANEL EFFICIENCY"],
    "relationships": [
      "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
      "DR. EMILY ROBERTS, explored partnerships with, EUROPEAN COMPANIES",
      "DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY"
    ]
  },
  {
    "semantic_unit": "Dr. John Miller conducted fieldwork in the Amazon Rainforest, documenting several new species and observing the effects of deforestation on local wildlife.",
    "entities": ["DR. JOHN MILLER", "AMAZON RAINFOREST", "NEW SPECIES", "DEFORESTATION", "LOCAL WILDLIFE"],
    "relationships": [
      "DR. JOHN MILLER, conducted fieldwork in, AMAZON RAINFOREST",
      "DR. JOHN MILLER, documented, NEW SPECIES",
      "DR. JOHN MILLER, observed the effects of, DEFORESTATION on LOCAL WILDLIFE"
    ]
  },
  {
    "semantic_unit": "The work of both Dr. Emily Roberts and Dr. John Miller is crucial in their respective fields and contributes significantly to environmental conservation efforts.",
    "entities": ["DR. EMILY ROBERTS", "DR. JOHN MILLER", "ENVIRONMENTAL CONSERVATION"],
    "relationships": [
      "DR. EMILY ROBERTS, contributes to, ENVIRONMENTAL CONSERVATION",
      "DR. JOHN MILLER, contributes to, ENVIRONMENTAL CONSERVATION"
    ]
  }
]


#########
Real_Data:
#########
Text:{text}
```

---

## Полный текст промпта (Chinese)

```
目标：给定一个文本，将该文本被划分为多个语义单元，每个单元包含对特定事件或活动的详细描述。
执行以下任务：
1.为每个语义单元提供总结，同时保留与原始上下文相关的所有关键细节。
2.直接从每个语义单元的原始文本中提取所有实体，而不是从改写的总结中提取。
3.从第2步中提取的实体中列出语义单元内的所有关系,其中关系类型可以是描述性句子。使用格式"ENTITY_A,RELATION_TYPE,ENTITY_B"，请确保字符串中包含三个元素，分别表示两个实体和关系类型。

要求：

时间实体：根据文本中提到的日期或时间的具体部分来表示时间实体，不填补缺失部分。

每个语义单元应以一个字典表示,包含三个键:semantic_unit(每个语义单元的概括性总结)、entities(直接从每个语义单元的原始文本中提取的实体列表,实体名格式为大写)、relationships(描述性句子形式的提取关系字符串三元组列表）。所有这些字典应存储在一个列表中，以便管理和访问。

示例:

文本:2024年9月,艾米莉·罗伯茨博士前往巴黎参加国际可再生能源会议。在她的访问期间,她与几家欧洲公司探讨了合作并介绍了她在提高太阳能板效率方面的最新研究。与此同时,在世界的另一边,她的同事约翰·米勒博士在亚马逊雨林进行实地工作。他记录了几种新物种,并观察了森林砍伐对当地野生动物的影响。两位学者的工作在各自的领域内至关重要,对环境保护工作做出了重大贡献。
输出：
[
  {
    "semantic_unit": "2024年9月,艾米莉·罗伯茨博士参加了在巴黎举行的国际可再生能源会议，她在会上介绍了她关于太阳能板效率提高的研究并探讨了与欧洲公司的合作。",
    "entities": ["艾米莉·罗伯茨博士", "2024-09", "巴黎", "国际可再生能源会议", "欧洲公司", "太阳能板效率"],
    "relationships": [
      "艾米莉·罗伯茨博士, 参加了, 国际可再生能源会议",
      "艾米莉·罗伯茨博士, 探讨了合作, 欧洲公司",
      "艾米莉·罗伯茨博士, 介绍了研究, 太阳能板效率"
    ]
  },
  {
    "semantic_unit": "约翰·米勒博士在亚马逊雨林进行实地工作，记录了几种新物种并观察了森林砍伐对当地野生动物的影响。",
    "entities": ["约翰·米勒博士", "亚马逊雨林", "新物种", "森林砍伐", "当地野生动物"],
    "relationships": [
      "约翰·米勒博士, 在, 亚马逊雨林进行实地工作",
      "约翰·米勒博士, 记录了, 新物种",
      "约翰·米勒博士, 观察了, 森林砍伐对当地野生动物的影响"
    ]
  },
  {
    "semantic_unit": "艾米莉·罗伯茨博士和约翰·米勒博士的工作在各自的领域内至关重要，对环境保护工作做出了重大贡献。",
    "entities": ["艾米莉·罗伯茨博士", "约翰·米勒博士", "环境保护"],
    "relationships": [
      "艾米莉·罗伯茨博士, 贡献于, 环境保护",
      "约翰·米勒博士, 贡献于, 环境保护"
    ]
  }
]

##########
实际数据：
##########
文本:{text}
```

---

## Структура промпта

### Компоненты

| Компонент | Назначение | Детали |
|-----------|-----------|--------|
| **Goal Statement** | Определение цели | Разбиение на semantic units |
| **Task Instructions** | Конкретные задачи | 3 задачи: summary, entities, relationships |
| **Requirements** | Дополнительные требования | Временные entity, формат |
| **Output Format** | Структура вывода | JSON с 3 ключами |
| **Example** | Демонстрация | Полный пример входа и выхода |
| **Real Data Marker** | Разделитель | Отделяет инструкцию от данных |
| **Input Placeholder** | Вход | `{text}` — Text Unit для обработки |

### Инструкции (3 задачи)

#### Задача 1: Semantic Unit Summary
**Цель**: Создать краткое резюме, сохраняя критические детали.

**Инструкция**:
> "Provide a summary for each semantic unit while retaining all crucial details relevant to the original context."

**Семантические операции**:
- **Segmentation**: Разбиение текста на атомарные смысловые единицы
- **Paraphrasing**: Перефразирование с сохранением сути
- **Detail Preservation**: Сохранение ключевых деталей

**Пример**:
```
Input (часть Text Unit):
"In September 2024, Dr. Emily Roberts traveled to Paris to attend
the International Conference on Renewable Energy."

Output (semantic_unit):
"In September 2024, Dr. Emily Roberts attended the International
Conference on Renewable Energy in Paris."

Изменения:
- "traveled to Paris to attend" → "attended...in Paris" (упрощение)
- Сохранены: время, персона, событие, локация
```

#### Задача 2: Entity Extraction
**Цель**: Извлечь все именованные сущности из оригинального текста.

**Инструкция**:
> "Extract all entities directly from the original text of each semantic unit, not from the paraphrased summary. Format each entity name in UPPERCASE. You should extract all entities including times, locations, people, organizations and all kinds of entities."

**Семантические операции**:
- **Named Entity Recognition (NER)**: Идентификация сущностей
- **Normalization**: UPPERCASE форматирование
- **Temporal Normalization**: Специальный формат для дат

**Типы сущностей**:
1. **People**: `DR. EMILY ROBERTS`, `DR. JOHN MILLER`
2. **Locations**: `PARIS`, `AMAZON RAINFOREST`
3. **Organizations**: `EUROPEAN RESEARCH INSTITUTE`, `EUROPEAN COMPANIES`
4. **Events**: `INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY`
5. **Concepts**: `SOLAR PANEL EFFICIENCY`, `ENVIRONMENTAL CONSERVATION`
6. **Temporal**: `2024-09` (September 2024)

**Важно**:
- Извлекать из **оригинального текста**, не из summary
- **UPPERCASE** для всех entities
- **Temporal entities**: Format `YYYY-MM`, `YYYY-MM-DD` в зависимости от точности

**Пример**:
```
Original text: "In September 2024, Dr. Emily Roberts..."
Entities: ["DR. EMILY ROBERTS", "2024-09", ...]

НЕ из summary: "In September 2024, Dr. Emily Roberts attended..."
```

#### Задача 3: Relationship Extraction
**Цель**: Извлечь все отношения между entities.

**Инструкция**:
> "From the entities extracted in Step 2, list all relationships within the semantic unit and the corresponding original context in the form of string seperated by comma : 'ENTITY_A, RELATION_TYPE, ENTITY_B'. The RELATION_TYPE could be a descriptive sentence, while the entities involved in the relationship must come from the entity names extracted in Step 2."

**Формат**: `"ENTITY_A, RELATION_TYPE, ENTITY_B"`

**Семантические операции**:
- **Relation Detection**: Идентификация связей между сущностями
- **Triple Formation**: Создание triplets {subject, predicate, object}
- **Descriptive Relations**: Отношения как описательные фразы

**Типы отношений**:
1. **Action**: `attended`, `conducted fieldwork in`, `documented`
2. **Affiliation**: `works at`, `member of`
3. **Attribute**: `researches`, `specializes in`
4. **Location**: `located in`, `traveled to`
5. **Temporal**: `occurred in`, `happened during`
6. **Contribution**: `contributes to`, `impacts`

**Пример**:
```
Entities: ["DR. EMILY ROBERTS", "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"]

Relationship:
"DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"

Структура:
  source: DR. EMILY ROBERTS
  relation: attended
  target: INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY
```

**Ключевое требование**:
> "Please make sure the string contains three elements representing two entities and the relationship type."

---

## Structured Output Schema (Pydantic)

```python
from pydantic import BaseModel

class semantic_group(BaseModel):
    semantic_unit: str
    entities: list[str]
    relationships: list[str]

class text_decomposition(BaseModel):
    Output: list[semantic_group]
```

**Файл**: `NodeRAG/utils/prompt/json_format.py:3-9`

**Назначение**:
- Обеспечивает **structured output** от LLM
- Гарантирует корректный формат JSON
- Валидирует типы данных

**Использование**:
```python
# LLM вызов с structured output
response = llm.generate(
    prompt=text_decomposition_prompt.format(text=text_unit),
    response_format=text_decomposition  # Pydantic schema
)

# response.Output = list[semantic_group]
```

---

## Пример использования

### Вход (Text Unit)

```
In September 2024, Dr. Emily Roberts traveled to Paris to attend the
International Conference on Renewable Energy. During her visit, she explored
partnerships with several European companies and presented her latest research
on solar panel efficiency improvements. Meanwhile, on the other side of the
world, her colleague, Dr. John Miller, was conducting fieldwork in the Amazon
Rainforest. He documented several new species and observed the effects of
deforestation on the local wildlife. Both scholars' work is essential in their
respective fields and contributes significantly to environmental conservation
efforts.
```

### Выход (Decomposition Result)

```json
[
  {
    "semantic_unit": "In September 2024, Dr. Emily Roberts attended the International Conference on Renewable Energy in Paris, where she presented her research on solar panel efficiency improvements and explored partnerships with European companies.",
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
  },
  {
    "semantic_unit": "Dr. John Miller conducted fieldwork in the Amazon Rainforest, documenting several new species and observing the effects of deforestation on local wildlife.",
    "entities": [
      "DR. JOHN MILLER",
      "AMAZON RAINFOREST",
      "NEW SPECIES",
      "DEFORESTATION",
      "LOCAL WILDLIFE"
    ],
    "relationships": [
      "DR. JOHN MILLER, conducted fieldwork in, AMAZON RAINFOREST",
      "DR. JOHN MILLER, documented, NEW SPECIES",
      "DR. JOHN MILLER, observed the effects of, DEFORESTATION on LOCAL WILDLIFE"
    ]
  },
  {
    "semantic_unit": "The work of both Dr. Emily Roberts and Dr. John Miller is crucial in their respective fields and contributes significantly to environmental conservation efforts.",
    "entities": [
      "DR. EMILY ROBERTS",
      "DR. JOHN MILLER",
      "ENVIRONMENTAL CONSERVATION"
    ],
    "relationships": [
      "DR. EMILY ROBERTS, contributes to, ENVIRONMENTAL CONSERVATION",
      "DR. JOHN MILLER, contributes to, ENVIRONMENTAL CONSERVATION"
    ]
  }
]
```

### Семантический анализ выхода

**Semantic Unit 1**: Событие конференции
- **Entities**: 6 (персона, время, место, событие, организация, концепция)
- **Relationships**: 3 (все связаны с DR. EMILY ROBERTS)
- **Characteristics**: Один связный нарратив о конференции

**Semantic Unit 2**: Полевая работа
- **Entities**: 5 (персона, место, объекты исследования)
- **Relationships**: 3 (все связаны с DR. JOHN MILLER)
- **Characteristics**: Параллельное событие в другой локации

**Semantic Unit 3**: Общий вклад
- **Entities**: 3 (2 персоны, концепция)
- **Relationships**: 2 (связи обеих персон с conservation)
- **Characteristics**: Мета-уровень, связывающий предыдущие units

---

## Роль в цепочке преобразований

### Позиция в Pipeline

```
Text Units (Stage 1)
      ↓
[TEXT DECOMPOSITION PROMPT] ← ВЫ ЗДЕСЬ
      ↓
{semantic_units, entities, relationships}
      ↓
Graph Construction (Stage 3)
```

### Предшествующее преобразование

**Semantic Chunking** → Text Units

**Входные требования**:
- Размер: ~1048 токенов
- Качество: Семантически целостные фрагменты
- Формат: Неструктурированный текст

**Влияние качества входа**:
```
Хороший chunk (на границе параграфа):
  → Полный контекст
  → Четкие entity boundaries
  → Точная extraction

Плохой chunk (разрыв посередине предложения):
  → Неполный контекст
  → Фрагментированные entities
  → Ошибки в extraction
```

### Последующие преобразования

#### 1. Graph Construction
**Зависимость**: Требует entities и relationships для создания узлов и рёбер.

```
semantic_units → semantic_unit nodes
entities → entity nodes
relationships → relationship nodes + edges
```

#### 2. Attribute Generation
**Зависимость**: Использует semantic_units как контекст для attributes.

```
Entity: DR. EMILY ROBERTS
Context: All semantic_units mentioning this entity
  → Attribute: Comprehensive description
```

#### 3. Embedding Generation
**Зависимость**: Векторизует semantic_units для HNSW индекса.

```
semantic_unit text → embedding vector
  → HNSW index for retrieval
```

#### 4. Community Detection
**Зависимость**: Entities и relationships формируют топологию графа.

```
Graph topology → Community clusters
  → Semantic communities
```

---

## Связи с другими промптами

### Downstream Prompts

#### Relationship Reconstruction Prompt
**Связь**: Исправляет некорректные relationships из этого промпта.

**Сценарий**:
```
Text Decomposition выдает:
  relationships: ["DR. EMILY ROBERTS, attended conference in Paris"]

Ошибка: Только 2 элемента вместо 3!

Relationship Reconstruction исправляет:
  {
    "source": "DR. EMILY ROBERTS",
    "relation": "attended conference in",
    "target": "PARIS"
  }
```

**Частота**: ~5% relationships требуют реконструкции.

#### Attribute Generation Prompt
**Связь**: Использует semantic_units как входной контекст.

**Процесс**:
```
Text Decomposition создаёт:
  semantic_unit: "Dr. Emily Roberts attended conference..."

Attribute Generation собирает все semantic_units с DR. EMILY ROBERTS:
  → Синтезирует comprehensive attribute
```

#### Community Summary Prompt
**Связь**: Использует semantic_units для тематического обобщения.

**Процесс**:
```
Text Decomposition создаёт:
  Multiple semantic_units в community

Community Summary анализирует:
  All semantic_units в cluster
  → Извлекает high-level themes
```

### Параллельные Prompts

#### Translation Prompt (вспомогательный)
**Связь**: Переводит этот промпт на другие языки.

**Использование**:
```python
if language not in ['English', 'Chinese']:
    prompt = translate_prompt.format(
        language=language,
        prompt=text_decomposition_prompt
    )
```

---

## Внешние библиотеки и инструменты

### LLM Providers

#### 1. OpenAI GPT Models
**Файл**: `NodeRAG/LLM/LLM_state.py`

**Модели**:
- **GPT-4o**: Рекомендуемая (высокая точность)
- **GPT-4o-mini**: Экономичная опция
- **GPT-4-turbo**: Баланс скорости и качества

**Параметры**:
```python
{
    "model": "gpt-4o-mini",
    "temperature": 0.0,  # Детерминизм
    "response_format": text_decomposition  # Structured output
}
```

**API**:
```python
from openai import OpenAI

client = OpenAI(api_key=api_key)
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.0,
    response_format={"type": "json_schema", "json_schema": schema}
)
```

#### 2. Google Gemini Models
**Модели**:
- **Gemini-1.5-Pro**: Высокая точность, большой context window
- **Gemini-1.5-Flash**: Быстрая обработка

**Параметры**:
```python
{
    "model": "gemini-1.5-pro",
    "temperature": 0.0,
    "response_mime_type": "application/json",
    "response_schema": text_decomposition_schema
}
```

### Structured Output Libraries

#### Pydantic
**Версия**: 2.x
**Назначение**: Валидация и парсинг JSON

**Использование**:
```python
from pydantic import BaseModel

class semantic_group(BaseModel):
    semantic_unit: str
    entities: list[str]
    relationships: list[str]

# Автоматическая валидация
result = semantic_group(**json_data)
```

**Преимущества**:
- Автоматическая валидация типов
- Детальные сообщения об ошибках
- Интеграция с LLM structured output

### Prompt Management

#### prompt_manager
**Файл**: `NodeRAG/utils/prompt/prompt_manager.py:14-102`

**Функции**:
- Управление мультиязычными промптами
- Автоматический перевод через LLM
- Доступ к structured output schemas

**Использование**:
```python
manager = prompt_manager(language="English")
prompt = manager.text_decomposition  # Получить промпт
schema = manager.text_decomposition_json  # Получить schema
```

### Token Counting

#### tiktoken
**Назначение**: Подсчёт токенов для OpenAI моделей

**Файл**: `NodeRAG/utils/token_utils.py`

**Использование**:
```python
import tiktoken

tokenizer = tiktoken.encoding_for_model("gpt-4o-mini")
token_count = len(tokenizer.encode(text))
```

**Роль**:
- Проверка, что Text Unit ≤ chunk_size
- Оценка стоимости API вызова
- Валидация промпта + текста < context window

---

## Параметры и настройка

### Temperature

**Значение**: 0.0

**Обоснование**:
- **Детерминизм**: Одинаковые входы → одинаковые выходы
- **Точность**: Минимизация галлюцинаций
- **Воспроизводимость**: Критично для production

**Эффект**:
```
Temperature 0.0:
  → Модель выбирает наиболее вероятные токены
  → Стабильные результаты
  → Минимум случайности

Temperature > 0.0:
  → Больше вариативности
  → Риск несогласованности
  → Не подходит для extraction
```

### Max Tokens

**Оценка**:
```
Input: ~1048 tokens (Text Unit)
Prompt: ~500 tokens (инструкции + пример)
Output: ~1000-1500 tokens (decomposition result)

Total: ~2500-3000 tokens
```

**Требования**:
- Model context window ≥ 4000 tokens (безопасно)
- GPT-4o: 128K context (избыточно)
- Gemini-1.5-Pro: 1M context (избыточно)

### Retry Strategy

**Ошибки валидации**:
```python
try:
    result = text_decomposition(**response_json)
except ValidationError as e:
    # Retry с более явными инструкциями
    prompt = prompt + f"\n\nNote: {e.errors()}"
    retry_response = llm.generate(prompt)
```

**Частые ошибки**:
1. **Incorrect relationship format**: "entity1, entity2" вместо "entity1, relation, entity2"
   - Решение: Relationship Reconstruction Prompt
2. **Missing entities**: Не все entities извлечены
   - Решение: Повторная обработка или ручная проверка
3. **Lowercase entities**: Entities не в UPPERCASE
   - Решение: Post-processing нормализация

---

## Метрики качества

### Точность Extraction

| Метрика | Целевое значение | Типичное значение |
|---------|------------------|-------------------|
| **Entity Recall** | >95% | 90-95% |
| **Entity Precision** | >90% | 85-92% |
| **Relationship Recall** | >85% | 80-88% |
| **Relationship Precision** | >85% | 80-87% |
| **Format Compliance** | 100% | 95-98% (с валидацией) |

### Стоимость

**OpenAI GPT-4o-mini**:
```
Input: 1048 tokens × $0.15 / 1M = $0.00016
Output: 1200 tokens × $0.60 / 1M = $0.00072

Total per Text Unit: ~$0.00088
```

**Для документа (10K токенов)**:
```
Text Units: ~10 (1048 токенов каждый)
Total cost: 10 × $0.00088 = $0.0088 (~$0.01)
```

### Latency

**Типичные значения**:
- GPT-4o-mini: 2-5 секунд per Text Unit
- Gemini-1.5-Flash: 1-3 секунды
- GPT-4o: 3-7 секунд

**Батчевая обработка**:
```
10 Text Units × 3 сек = 30 секунд последовательно
10 Text Units параллельно = ~5 секунд (с ограничением rate limit)
```

---

## Лучшие практики

### 1. Предобработка текста

```python
# Очистка перед decomposition
text = text.strip()
text = re.sub(r'\n{3,}', '\n\n', text)  # Нормализация переносов
text = re.sub(r'\s{2,}', ' ', text)  # Удаление лишних пробелов
```

### 2. Валидация выхода

```python
# Проверка формата relationships
for rel in relationships:
    parts = rel.split(',')
    if len(parts) != 3:
        # Отправить на Relationship Reconstruction
        fixed_rel = reconstruct_relationship(rel)
```

### 3. Entity нормализация

```python
# Post-processing нормализация
entities = [e.upper().strip() for e in entities]
entities = list(set(entities))  # Дедупликация
```

### 4. Мониторинг качества

```python
# Логирование метрик
metrics = {
    'semantic_units_count': len(result.Output),
    'entities_count': sum(len(su.entities) for su in result.Output),
    'relationships_count': sum(len(su.relationships) for su in result.Output),
    'avg_entities_per_unit': avg_entities,
    'malformed_relationships': malformed_count
}
log_metrics(metrics)
```

---

## Ограничения и проблемы

### 1. Coreference Resolution

**Проблема**: Местоимения не всегда резолвятся.

```
Текст: "Dr. Roberts attended. She presented research."
Output entities: ["DR. ROBERTS", "SHE"]  ❌

Ожидалось: ["DR. ROBERTS"]
```

**Решение**: LLM обычно хорошо справляется, но не всегда.

### 2. Сложные relationships

**Проблема**: Многосторонние связи упрощаются.

```
Текст: "Dr. Roberts and Dr. Miller collaboratively researched with Institute."

Output (упрощение):
  "DR. ROBERTS, researched with, INSTITUTE"
  "DR. MILLER, researched with, INSTITUTE"

Потеря: Информация о коллаборации между Roberts и Miller
```

### 3. Implicit relationships

**Проблема**: Неявные связи могут быть пропущены.

```
Текст: "The conference was in Paris. Dr. Roberts attended the conference."

Output:
  "DR. ROBERTS, attended, CONFERENCE"

Пропущено: "CONFERENCE, located in, PARIS" (implicit)
```

### 4. Ambiguous entities

**Проблема**: Омонимы могут быть спутаны.

```
Текст: "Apple announced new research. Apple trees grow in the region."

Output:
  ["APPLE", "APPLE TREES"]

Неясно: APPLE (компания) vs APPLE (фрукт)
```

---

## Будущие улучшения

### 1. Few-shot примеры

Добавить domain-specific примеры для улучшения точности:

```python
examples = get_domain_examples(domain="medical")
prompt = prompt + f"\n\nAdditional examples:\n{examples}"
```

### 2. Confidence scores

Добавить уверенность для каждого извлечения:

```python
class semantic_group(BaseModel):
    semantic_unit: str
    entities: list[dict[str, float]]  # {"entity": "DR. ROBERTS", "confidence": 0.95}
    relationships: list[dict[str, float]]
```

### 3. Hierarchical entities

Поддержка иерархических relationships:

```python
class relationship(BaseModel):
    source: str
    relation: str
    target: str
    context: str  # Дополнительный контекст
    type: str  # Тип отношения (action, affiliation, etc.)
```

---

## Заключение

Text Decomposition Prompt — это **критически важный** компонент NodeRAG, обеспечивающий:
- ✅ Точную extraction entities и relationships
- ✅ Структурированные semantic units
- ✅ Нормализованный формат (UPPERCASE, triplets)
- ✅ Детерминистическую обработку (Temperature 0.0)
- ✅ Structured output через Pydantic

**Качество этого промпта напрямую определяет качество всего pipeline** — от графа знаний до финальных ответов.

---

## Ссылки

- **Файл промпта**: `NodeRAG/utils/prompt/text_decomposition.py:1-107`
- **Schema**: `NodeRAG/utils/prompt/json_format.py:3-9`
- **Manager**: `NodeRAG/utils/prompt/prompt_manager.py:20-27, 84-85`
- **Использование**: Text Pipeline (Stage 2)
- **Связанные документы**:
  - `spec/transform/indexing-transformations.md` — Детали преобразования
  - `spec/transform/transformation-chains.md` — Полная цепочка
  - `spec/document-processing-pipeline.md` — Обзор pipeline
