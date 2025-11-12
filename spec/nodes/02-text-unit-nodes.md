# Text Unit Nodes 📝

## Обзор

**Text Unit Nodes** представляют семантические chunks текста, полученные в результате intelligent chunking исходных документов. Это базовые единицы для LLM обработки и первый уровень semantic decomposition.

---

## Характеристики

### Класс
```python
NodeRAG/build/component/text_unit.py

class Text_unit(Unit_base):
    - raw_context: str       # Текст chunk'а
    - hash_id: str           # SHA256(raw_context)
    - human_readable_id: str # T00001, T00002, ...
```

### Свойства

| Свойство             | Тип       | Описание                              |
|----------------------|-----------|---------------------------------------|
| `raw_context`        | str       | Содержимое text unit (~1048 tokens)   |
| `hash_id`            | str       | SHA256 хеш от raw_context             |
| `human_readable_id`  | str       | Последовательный ID (T00001, ...)     |

### Metadata

| Параметр             | Значение                       |
|----------------------|--------------------------------|
| Type                 | `text_unit`                    |
| Has Embedding        | ✅ Да                          |
| Has Weight           | ❌ Нет                         |
| Searchable           | ✅ Да (через embedding)        |
| Storage              | `text.parquet`                 |
| In Graph             | ✅ Да (добавляется позже)      |

---

## Создание

### Pipeline Stage
**Document Pipeline** (Stage 1) - создание
**Insert Text Pipeline** (Stage 7) - добавление в граф

### Процесс

```python
# 1. Semantic chunking документа
texts = semantic_text_splitter.split(document.raw_context)

# 2. Создание Text Unit объектов
text_units = [Text_unit(text) for text in texts]

# 3. Автоматическая генерация идентификаторов
for text_unit in text_units:
    text_unit.hash_id  # SHA256(raw_context)
    text_unit.human_readable_id  # T00001, T00002, ...
```

### Semantic Chunking

**Алгоритм**: `SemanticTextSplitter`

**Параметры**:
- `chunk_size`: 1048 tokens (default)
- `model_name`: "gpt-4o-mini" (для подсчета токенов)

**Приоритет границ**:
1. `\n\n` - параграфы (highest priority)
2. `\n` - новые строки
3. `.`, `。` - предложения
4. `!`, `！`, `?`, `？` - вопросы и восклицания
5. `;`, `；` - сложные предложения

**Пример chunking**:
```python
# Исходный документ (2000 tokens)
doc = """Параграф 1 (600 tokens)...

Параграф 2 (700 tokens)...

Параграф 3 (700 tokens)..."""

# Результат chunking (chunk_size=1048)
chunks = [
    "Параграф 1...\n\nПараграф 2...",  # 1300 → урезается до 1048
    "Параграф 3..."                     # 700 tokens
]
# Фактически:
chunks = [
    "Параграф 1...",      # 600 tokens (граница после П1)
    "Параграф 2...\n\nПараграф 3..."  # 1400 → урезается до 1048
]
```

---

## Роль в Графе

### Позиция
```
[Document]
    ↓
[Text Unit] ← входная точка для LLM
    ↓
[Semantic Units] (множество)
```

### Связи

**Входящие связи**:
- **От Document**: Родительская связь (через `doc_hash_id`)

**Исходящие связи**:
- **К Semantic Units**: Косвенная через `text_hash_id` в Semantic Unit
- После Stage 7: Прямые ребра в графе

### Graph Properties

```python
# После Insert Text Pipeline
G.add_node(
    text_unit.hash_id,
    type='text_unit',
    weight=1
)

# Связи с semantic units (если есть)
for semantic_unit in related_semantic_units:
    G.add_edge(text_unit.hash_id, semantic_unit.hash_id, weight=1)
```

---

## Storage Format

### text.parquet

```python
{
    'text_id': 'T00001',              # human_readable_id
    'hash_id': 'abc123...',           # hash_id
    'type': 'text',                   # тип узла
    'context': 'text content...',     # raw_context
    'doc_id': 'D00001',               # родительский document ID
    'doc_hash_id': 'def456...',       # родительский document hash
    'embedding': [0.1, 0.2, ...],     # vector (после Embedding Pipeline)
}
```

### Пример записи
```json
{
  "text_id": "T00001",
  "hash_id": "b4g9c3d5e6f7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7",
  "type": "text",
  "context": "Renewable energy research is crucial for addressing climate change. In September 2024, Dr. Emily Roberts from the European Research Institute attended the International Conference on Renewable Energy in Paris...",
  "doc_id": "D00001",
  "doc_hash_id": "a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6",
  "embedding": null
}
```

После Embedding Pipeline:
```json
{
  ...
  "embedding": [0.123, -0.456, 0.789, ...],  // 1536-dim vector
}
```

---

## LLM Processing

### Text Decomposition

**Pipeline**: Text Pipeline (Stage 2)

**Процесс**:
```python
# Для каждого text unit
async def text_decomposition(self, config):
    # 1. Формирование промпта
    prompt = config.prompt_manager.text_decomposition.format(
        text=self.raw_context
    )

    # 2. LLM вызов
    input_data = {
        'query': prompt,
        'response_format': config.prompt_manager.text_decomposition_json
    }

    response = await config.API_client(
        input_data,
        cache_path=config.LLM_error_cache,
        meta_data={
            'text_hash_id': self.hash_id,
            'text_id': self.human_readable_id
        }
    )

    # 3. Сохранение результата
    with open(config.text_decomposition_path, 'a') as f:
        data = {
            'text_hash_id': self.hash_id,
            'text_id': self.human_readable_id,
            'response': response
        }
        f.write(json.dumps(data) + '\n')
```

**Output**:
```json
{
  "Output": [
    {
      "semantic_unit": "Dr. Emily Roberts attended conference in Paris",
      "entities": ["DR. EMILY ROBERTS", "PARIS", "2024-09"],
      "relationships": [
        "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE"
      ]
    },
    {
      "semantic_unit": "Conference focused on renewable energy",
      "entities": ["RENEWABLE ENERGY", "INTERNATIONAL CONFERENCE"],
      "relationships": [
        "INTERNATIONAL CONFERENCE, focused on, RENEWABLE ENERGY"
      ]
    }
  ]
}
```

### Параллелизация

```python
# Все text units обрабатываются параллельно
async_tasks = []
for text_unit in text_units:
    task = text_unit.text_decomposition(config)
    async_tasks.append(task)

await asyncio.gather(*async_tasks)
```

---

## Embeddings

### Генерация

**Pipeline**: Embedding Pipeline (Stage 5)

```python
# Batch processing
batch_size = config.embedding_batch_size  # default: 100

for i in range(0, len(text_units), batch_size):
    batch = text_units[i:i+batch_size]
    contexts = [tu.raw_context for tu in batch]

    # Embedding model call
    embeddings = await embedding_client(contexts)

    # Сохранение
    for tu, emb in zip(batch, embeddings):
        save_embedding(tu.hash_id, emb)
```

### Размерность

| Model                      | Dimension |
|----------------------------|-----------|
| OpenAI text-embedding-3-small | 512, 1536, 3072 |
| OpenAI text-embedding-ada-002 | 1536      |
| Gemini text-embedding-004     | 768       |

---

## Использование в Поиске

### Direct Search

**Тип**: Векторный поиск по полному контексту chunk'а

**Use case**: Когда нужен broad context, не точечный факт

```python
# Query
query = "What did Dr. Emily Roberts research?"

# Embedding
query_embedding = embedding_model(query)

# Search in text units
results = HNSW_index.search(
    query_embedding,
    k=10,
    filter=lambda x: x['type'] == 'text_unit'
)

# Results
for result in results:
    text_unit = load_text_unit(result['hash_id'])
    print(f"Similarity: {result['score']}")
    print(f"Context: {text_unit.raw_context}")
```

### Expand to Details

После нахождения релевантного text unit, expand к semantic units:

```python
# Нашли релевантный text unit
text_unit_hash_id = "abc123..."

# Получить все semantic units из этого text unit
semantic_units = load_semantic_units_by_text(text_unit_hash_id)

# Semantic units содержат более точные факты
for su in semantic_units:
    print(su.raw_context)
```

### Advantages

✅ **Broader context**: Целый chunk, не отдельный факт
✅ **Less fragmented**: Меньше риска потерять связь между фактами
✅ **Original text**: Точный текст из документа, не paraphrased

### Disadvantages

❌ **Less precise**: Может содержать нерелевантные части
❌ **Larger context**: Занимает больше tokens в LLM context window
❌ **Duplicates**: Разные semantic units из одного text unit

---

## Использование в Answer Generation

### Как Source Context

**Роль**: Предоставление полного контекста для LLM

```python
# После retrieval
relevant_text_units = search_results  # Top-k text units

# Формирование context для answer generation
context = "\n\n---\n\n".join([
    f"Source {i+1}:\n{tu.raw_context}"
    for i, tu in enumerate(relevant_text_units)
])

# Answer generation
answer = await answer_llm({
    'query': user_query,
    'context': context
})
```

### Как Fallback

Если semantic units недостаточно:

```python
# 1. Попытка с semantic units
semantic_results = search_semantic_units(query, k=5)

if len(semantic_results) < threshold:
    # 2. Fallback к text units для broader context
    text_results = search_text_units(query, k=3)
    combined_context = semantic_results + text_results
else:
    combined_context = semantic_results

answer = generate_answer(query, combined_context)
```

### Преимущества для Answer

✅ **Complete sentences**: Полные предложения и параграфы
✅ **Original wording**: Оригинальные формулировки автора
✅ **Context preservation**: Сохранен контекст вокруг фактов

---

## Связь с Semantic Units

### Один-ко-многим

```
[Text Unit T00001]
    ↓ decomposition
    ├── [Semantic Unit S00001]
    ├── [Semantic Unit S00002]
    ├── [Semantic Unit S00003]
    └── [Semantic Unit S00004]
```

### Обратная навигация

```python
# От Semantic Unit к Text Unit
semantic_unit_data = load_semantic_unit('S00001')
text_hash_id = semantic_unit_data['text_hash_id']

text_unit_data = load_text_unit_by_hash(text_hash_id)
text_unit_context = text_unit_data['context']
```

### Агрегация

```python
# Все semantic units из text unit
text_units_hash = "abc123..."
semantic_units = decomposition_data[
    decomposition_data['text_hash_id'] == text_unit_hash
]['response']['Output']

print(f"Text unit decomposed into {len(semantic_units)} semantic units")
```

---

## Примеры

### Пример 1: Короткий текст

**Input** (200 tokens):
```
Dr. Emily Roberts is a leading researcher. She works at European
Research Institute. Her research focuses on solar panel efficiency.
```

**Text Units**: 1 unit (весь текст < 1048 tokens)
```python
Text_unit {
    raw_context: "Dr. Emily Roberts is a leading researcher...",
    hash_id: "abc...",
    human_readable_id: "T00001"
}
```

**Decomposition** → 3 Semantic Units:
1. "Dr. Emily Roberts is a leading researcher"
2. "Dr. Emily Roberts works at European Research Institute"
3. "Her research focuses on solar panel efficiency"

### Пример 2: Длинный текст

**Input** (3000 tokens): Научная статья

**Text Units**: 3 units
```python
T00001 (1000 tokens): Introduction and background
T00002 (1048 tokens): Methodology section (cut at sentence boundary)
T00003 (952 tokens): Results and conclusion
```

**Decomposition**:
- T00001 → 8 semantic units
- T00002 → 12 semantic units
- T00003 → 10 semantic units

Total: 30 semantic units

---

## Метрики

### Размер

**Оптимальный размер**: 500-1048 tokens

**Статистика** (для 100 документов ~10MB):
- Average size: 800 tokens
- Min size: 200 tokens
- Max size: 1048 tokens
- Total text units: ~2000

### Decomposition Rate

**Average**: 1 Text Unit → 5 Semantic Units

**Зависит от**:
- Плотность информации
- Структурированность текста
- Количество фактов

---

## Best Practices

### 1. Chunking Strategy

✅ **Хорошо**:
- Сохранять семантические границы
- Не разрывать предложения
- Приоритет параграфам

❌ **Плохо**:
- Фиксированный размер без учета границ
- Разрыв в середине предложения

### 2. Size Trade-offs

**Слишком маленькие chunks** (< 300 tokens):
- Потеря контекста
- Фрагментация информации
- Больше chunks → больше LLM calls

**Слишком большие chunks** (> 1500 tokens):
- Превышение context window некоторых моделей
- Менее точный retrieval
- Сложнее для LLM decomposition

**Оптимум**: 500-1048 tokens

### 3. Processing

- **Параллелизация**: Обрабатывайте все text units асинхронно
- **Error handling**: Кешируйте ошибки для retry
- **Incremental**: Обрабатывайте только новые text units

---

## Диагностика

### Проверка chunking

```python
# Загрузить text units
df = storage.load_parquet(config.text_path)

# Статистика размеров
contexts = df['context'].tolist()
token_counts = [count_tokens(ctx) for ctx in contexts]

print(f"Average size: {np.mean(token_counts)} tokens")
print(f"Min size: {np.min(token_counts)} tokens")
print(f"Max size: {np.max(token_counts)} tokens")

# Проверка на очень маленькие chunks
small_chunks = [t for t in token_counts if t < 200]
print(f"Small chunks (<200 tokens): {len(small_chunks)}")
```

### Проверка decomposition

```python
# Загрузить decomposition results
with open(config.text_decomposition_path) as f:
    results = [json.loads(line) for line in f]

# Статистика decomposition rate
rates = [
    len(r['response']['Output'])
    for r in results
    if 'response' in r
]

print(f"Average SU per TU: {np.mean(rates)}")
print(f"Max SU per TU: {np.max(rates)}")

# Найти text units без decomposition
decomposed_hashes = {r['text_hash_id'] for r in results}
all_hashes = set(df['hash_id'])
missing = all_hashes - decomposed_hashes

if missing:
    print(f"Warning: {len(missing)} text units not decomposed")
```

---

## FAQ

**Q: Почему text units добавляются в граф только на Stage 7?**
A: Они создаются рано, но граф строится из semantic units. Text units добавляются позже для полноты и навигации.

**Q: Нужно ли искать по text units или semantic units?**
A: Зависит от use case. Semantic units - для точных фактов, text units - для broader context.

**Q: Как выбрать optimal chunk_size?**
A: 1048 tokens - хороший баланс. Больше = больше context, но сложнее decomposition. Меньше = точнее, но фрагментировано.

**Q: Что делать с очень короткими документами (< 200 tokens)?**
A: Они создадут 1 text unit. LLM decomposition все равно извлечет semantic units, даже если их мало.

---

## Связанные Узлы

**Родитель**:
- [Document Nodes](./01-document-nodes.md) - исходные документы

**Дети**:
- [Semantic Unit Nodes](./03-semantic-unit-nodes.md) - извлеченные факты

**Используется в**:
- [Search and Retrieval](./09-search-and-retrieval.md) - для broad context search
- [Answer Generation](./10-answer-generation.md) - как source context

---

[← Назад: Document Nodes](./01-document-nodes.md) | [Следующий: Semantic Unit Nodes →](./03-semantic-unit-nodes.md)
