# Embedding Generation: Векторное представление семантики

## Концептуальный обзор

**Embedding Generation** — это процесс преобразования текста в **плотные векторные представления** (embeddings) в многомерном пространстве. В NodeRAG используется модель **OpenAI text-embedding-3-small** для создания 1536-мерных векторов, которые кодируют семантическое содержание текста.

### Философская сущность

В контексте [парадигмы вопроса как ключа](../research/question-as-key-paradigm.md), embeddings выполняют функцию **геометрической проекции концепций**:

```
Текст (символическое) → Embedding (геометрическое) → Семантическое пространство
     ↓                       ↓                              ↓
Дискретное            Непрерывное                    Измеримое
```

Это переход от **символьного** представления (слова, предложения) к **числовому** представлению (векторы), где **семантическая близость** измеряется **геометрическим расстоянием**.

---

## Роль в системе

### Зачем embeddings?

**Проблема 1: Символьный поиск недостаточен**

```python
query = "renewable energy"
text = "solar power and wind turbines"

# Точный поиск: NO MATCH (разные слова)
# Embedding поиск: HIGH SIMILARITY (та же тема)
```

**Проблема 2: Граф не измеряет семантическую близость**

```python
# Граф показывает структурные связи:
[SOLAR ENERGY] ---is_type_of---> [RENEWABLE ENERGY]

# Но как близки "solar panels" и "photovoltaic cells"?
# Embeddings: cosine_similarity = 0.85 (очень близко!)
```

**Решение NodeRAG**: **Hybrid approach** — комбинация embeddings (аналоговая близость) + graph (структурные связи).

---

## Embedding Space: 1536-мерное семантическое пространство

### Математическая модель

Каждый текст t преобразуется в вектор в ℝ¹⁵³⁶:

```
embedding: T → ℝ¹⁵³⁶
t (text) ↦ v (vector)
```

где `v = [v₁, v₂, ..., v₁₅₃₆]` — вещественнозначный вектор.

### Свойства embedding space

**1. Семантическая близость → Геометрическая близость**

```
cosine_similarity(emb("dog"), emb("puppy")) ≈ 0.85
cosine_similarity(emb("dog"), emb("car")) ≈ 0.15
```

**2. Векторная арифметика**

```
emb("king") - emb("man") + emb("woman") ≈ emb("queen")
```

(Не гарантируется для всех embeddings, но общее свойство)

**3. Кластеризация по темам**

```
Тема "Renewable Energy":
  emb("solar power")
  emb("wind energy")
  emb("hydroelectric")
     ↓ образуют кластер в пространстве
```

### Dimensionality: Почему 1536?

**OpenAI text-embedding-3-small**: 1536 измерений

**Trade-offs**:

| Размерность | Качество | Память | Скорость |
|-------------|----------|--------|----------|
| 384 (small) | Средне | Низкая | Быстро |
| 768 (medium) | Хорошо | Средняя | Средне |
| 1536 (large) | Отлично | Высокая | Медленно |
| 3072 (xl) | Наилучше | Очень высокая | Очень медленно |

**NodeRAG выбор**: 1536 — баланс качества и эффективности.

---

## Реализация в NodeRAG

### Класс `OpenAI_Embedding`

**Файл**: `NodeRAG/LLM/LLM.py:200-245`

```python
class OpenAI_Embedding(LLM):

    def __init__(self,
                 model_name: str,
                 api_keys: str | None,
                 Config: ModelConfig|None) -> None:

        super().__init__(model_name, api_keys, Config)

        if api_keys is None:
            api_keys = os.getenv("OPENAI_API_KEY")

        self.client = OpenAI(api_key=api_keys)
        self.client_async = AsyncOpenAI(api_key=api_keys)
```

**Модель**: `text-embedding-3-small` (default в NodeRAG)

### Синхронная генерация

```python
@backoff.on_exception(backoff.expo,
                      [RateLimitError, Timeout, APIConnectionError],
                      max_time=30,
                      max_tries=4)
def _create_embedding(self, input: Embedding_message) -> Embedding_output:
    response = self.client.embeddings.create(
        model=self.model_name,
        input=input
    )
    return [res.embedding for res in response.data]

@error_handler
def API_client(self, input: Embedding_message) -> Embedding_output:
    response = self._create_embedding(input)
    return response
```

**Resilience**:
- **Exponential backoff**: Для RateLimitError, Timeout, APIConnectionError
- **max_tries=4**: До 4 повторных попыток
- **max_time=30**: Максимум 30 секунд на все попытки
- **error_handler**: Логирование и кэширование ошибок

### Асинхронная генерация

```python
@backoff.on_exception(backoff.expo,
                      [RateLimitError, Timeout, APIConnectionError],
                      max_time=30,
                      max_tries=4)
async def _create_embedding_async(self, input: Embedding_message) -> Embedding_output:
    response = await self.client_async.embeddings.create(
        model=self.model_name,
        input=input
    )
    return [res.embedding for res in response.data]

@error_handler_async
async def API_client_async(self, input: Embedding_message) -> Embedding_output:
    response = await self._create_embedding_async(input)
    return response
```

**Зачем async**: Для параллельной обработки множества текстов (см. [Batch Processing](./batch-processing.md)).

---

## Embedding Pipeline

### Класс `Embedding_pipeline`

**Файл**: `NodeRAG/build/pipeline/embedding.py`

#### Инициализация

```python
def __init__(self, config: NodeConfig):
    self.config = config
    self.embedding_client = self.config.embedding_client
    self.mapper = self.load_mapper()
```

**Mapper**: Хранит маппинг node_id → context для всех узлов.

#### Поиск узлов без embeddings

```python
def load_mapper(self) -> Mapper:
    mapping_list = [
        self.config.text_path,
        self.config.semantic_units_path,
        self.config.attributes_path
    ]
    mapping_list = [path for path in mapping_list if os.path.exists(path)]
    return Mapper(mapping_list)
```

**Какие узлы получают embeddings**:
- **Semantic Units**: ✓
- **Entities**: ✓
- **Attributes**: ✓
- **High-level Elements**: ✓
- **Relationships**: ✗ (не нужны для HNSW)
- **Texts**: ✗ (слишком длинные)

#### Batch Generation

```python
async def generate_embeddings(self):
    tasks = []
    none_embedding_ids = self.mapper.find_none_embeddings()

    self.config.tracker.set(
        math.ceil(len(none_embedding_ids) / self.config.embedding_batch_size),
        desc='Generating embeddings'
    )

    # Разбиваем на батчи
    for i in range(0, len(none_embedding_ids), self.config.embedding_batch_size):
        context_dict = {}

        for id in none_embedding_ids[i:i+self.config.embedding_batch_size]:
            context_dict[id] = self.mapper.get(id, 'context')

        tasks.append(self.get_embeddings(context_dict))

    # Параллельное выполнение
    await asyncio.gather(*tasks)

    self.config.tracker.close()
```

**Batch size**: `config.embedding_batch_size` (обычно 100)

**Зачем batching**: OpenAI API поддерживает batch requests — дешевле и быстрее.

#### Получение embeddings для батча

```python
async def get_embeddings(self, context_dict: Dict[str, Embedding_message]):
    # Фильтруем пустые контексты
    empty_ids = [key for key, value in context_dict.items() if value == ""]

    if len(empty_ids) > 0:
        context_dict = {key: value for key, value in context_dict.items() if value != ""}

        for empty_id in empty_ids:
            self.mapper.delete(empty_id)

    # Извлекаем тексты и IDs
    embedding_input = list(context_dict.values())
    ids = list(context_dict.keys())

    # API вызов
    embedding_output = await self.embedding_client(
        embedding_input,
        cache_path=self.config.LLM_error_cache,
        meta_data={'ids': ids}
    )

    if embedding_output == 'Error cached':
        return

    # Сохраняем в кэш
    with open(self.config.embedding_cache, 'a', encoding='utf-8') as f:
        for i in range(len(ids)):
            line = {'hash_id': ids[i], 'embedding': embedding_output[i]}
            f.write(json.dumps(line) + '\n')

    self.config.tracker.update()
```

**Кэширование**:
- **embedding_cache**: Временный файл (JSON lines)
- **LLM_error_cache**: Кэш ошибок для retry

#### Вставка embeddings в хранилище

```python
def insert_embeddings(self):
    if not os.path.exists(self.config.embedding_cache):
        return None

    with open(self.config.embedding_cache, 'r', encoding='utf-8') as f:
        lines = []

        for line in f:
            line = json.loads(line.strip())

            if isinstance(line['embedding'], str):
                continue  # Ошибка генерации

            self.mapper.add_attribute(line['hash_id'], 'embedding', 'done')
            lines.append(line)

    # Сохраняем в Parquet
    storage(lines).save_parquet(
        self.config.embedding,
        append=os.path.exists(self.config.embedding)
    )

    self.mapper.update_save()
```

**Формат хранения**: Parquet (эффективное сжатие для векторов)

---

## Использование в системе

### 1. Индексирование

**После Text Decomposition и Graph Construction**:

```python
# Узлы созданы, но без embeddings
semantic_units = [SU-1, SU-2, ..., SU-N]
entities = [ENT-1, ENT-2, ..., ENT-M]
attributes = [ATTR-1, ATTR-2, ..., ATTR-K]

# Генерация embeddings
embedding_pipeline = Embedding_pipeline(config)
await embedding_pipeline.main()

# Теперь узлы имеют embeddings
for node in nodes:
    embedding = mapper.get_embedding(node.hash_id)  # [v₁, v₂, ..., v₁₅₃₆]
```

### 2. HNSW Indexing

**После Embedding Generation**:

```python
# Загружаем embeddings
embeddings = mapper.load_embeddings()

# Добавляем в HNSW
nodes = []
for hash_id, embedding in embeddings:
    nodes.append((hash_id, embedding))

hnsw.add_nodes(nodes)
```

См. [HNSW Algorithm](../graph/hnsw-algorithm.md)

### 3. Query Embedding

**Во время поиска**:

```python
def search(self, query: str):
    # Генерация query embedding
    query_embedding = np.array(
        self.config.embedding_client.request(query),
        dtype=np.float32
    )

    # HNSW search
    HNSW_results = self.hnsw.search(
        query_embedding,
        HNSW_results=self.config.HNSW_results
    )

    return HNSW_results
```

**Файл**: `NodeRAG/search/search.py:84-85`

---

## Производительность

### Стоимость (OpenAI Pricing)

**Model**: text-embedding-3-small

| Метрика | Значение |
|---------|----------|
| Cost per 1M tokens | $0.02 |
| Cost per 1K tokens | $0.00002 |
| Average document (500 tokens) | $0.00001 |
| 100K documents | $1 |

**Типичный корпус** (10K documents, ~200K nodes):

```
200K nodes × 100 tokens/node = 20M tokens
20M tokens × $0.02/1M = $0.40
```

**Очень дешево**!

### Скорость

**OpenAI API**:

| Метрика | Значение |
|---------|----------|
| Throughput | ~3,000 texts/minute |
| Batch size | 100 (optimal) |
| Latency (single) | ~100-200ms |
| Latency (batch of 100) | ~500-1000ms |

**NodeRAG Benchmark** (200K nodes):

```
200K nodes / 100 (batch size) = 2,000 batches
2,000 batches × 0.5s/batch = 1,000s ≈ 17 minutes
```

**С параллелизмом** (10 concurrent tasks):

```
1,000s / 10 = 100s ≈ 1-2 minutes
```

### Память

**Single Embedding**:

```
1536 dimensions × 4 bytes/float32 = 6,144 bytes ≈ 6 KB
```

**200K Embeddings**:

```
200K × 6 KB = 1.2 GB (raw)
```

**С Parquet сжатием** (~50% compression):

```
1.2 GB → ~600 MB (disk)
```

**HNSW Index** (в RAM):

```
1.2 GB (embeddings) + 2-3 GB (HNSW structure) ≈ 4-5 GB
```

---

## Библиотеки

### OpenAI SDK

**Библиотека**: `openai==1.66.3`

**Используемые методы**:

```python
from openai import OpenAI, AsyncOpenAI

# Синхронный клиент
client = OpenAI(api_key=api_key)
response = client.embeddings.create(
    model="text-embedding-3-small",
    input=["text1", "text2", ...]
)

# Асинхронный клиент
client_async = AsyncOpenAI(api_key=api_key)
response = await client_async.embeddings.create(
    model="text-embedding-3-small",
    input=["text1", "text2", ...]
)
```

**Документация**: [LLM and AI Dependencies](../dependencies/llm-and-ai.md)

### NumPy

**Библиотека**: `numpy==1.26.4`

**Используется для**:
- Конвертация embeddings в np.array
- Векторные операции
- Подготовка для HNSW

```python
import numpy as np

# Конвертация embedding в NumPy array
embedding = np.array(embedding_list, dtype=np.float32)

# Нормализация (для cosine similarity)
embedding = embedding / np.linalg.norm(embedding)
```

**Документация**: [Vector Search Dependencies](../dependencies/vector-search.md)

### PyArrow (Parquet)

**Библиотека**: `pyarrow==19.0.1`

**Используется для**:
- Эффективное хранение embeddings
- Сжатие векторов
- Быстрая загрузка

```python
import pyarrow.parquet as pq

# Сохранение
table = pa.table({
    'hash_id': ids,
    'embedding': embeddings  # List[List[float]]
})
pq.write_table(table, 'embeddings.parquet')

# Загрузка
table = pq.read_table('embeddings.parquet')
```

**Документация**: [Data Processing Dependencies](../dependencies/data-processing-and-utilities.md)

---

## Связь с графовыми алгоритмами

### 1. Embedding → HNSW

Embeddings **индексируются** в HNSW для быстрого поиска:

```
Embeddings → HNSW Index → k-NN Query
```

См. [HNSW Algorithm](../graph/hnsw-algorithm.md)

### 2. HNSW → PPR

HNSW results становятся **персонализацией** для PPR:

```
Query → Query Embedding → HNSW Search → Attractors → PPR
```

См. [Personalized PageRank](../graph/personalized-pagerank.md)

### 3. Embeddings → K-means

High-level element embeddings **кластеризуются** для связи с community nodes:

```
Community → HLE Embeddings → K-means → Cluster Assignment → Graph Edges
```

См. [K-means Clustering](./kmeans-clustering.md) и [Leiden Algorithm](../graph/leiden-community-detection.md)

### 4. Graph + Embeddings = Hybrid

```
Knowledge Graph (структура) + Embeddings (семантика) = Unified Retrieval
                              ↓
                    Graph Concatenation
                              ↓
                       Unified Graph
                              ↓
                     PPR with Hybrid Info
```

См. [Graph Construction](../graph/graph-construction.md)

---

## Концептуальные связи

### Embedding Space как онтологическое пространство

В [парадигме вопроса как ключа](../research/question-as-key-paradigm.md), embedding space представляет **онтологический континуум**:

```
Концепция (абстракция)
       ↓ [Projection]
Embedding (геометрическая точка)
       ↓ [Distance]
Проявления (близкие концепции)
```

**Философская интерпретация**:
- **Близость в пространстве** ≈ **концептуальная родственность**
- **Расстояние** ≈ **семантическая дистанция**
- **Кластеры** ≈ **категории** или **темы**

### Дуальность: Аналоговое vs Символическое

Embeddings представляют **аналоговую** сторону NodeRAG (см. [Star Pattern Architecture](../research/star-pattern-architecture.md)):

| Embeddings (Аналоговое) | Graph (Символическое) |
|-------------------------|----------------------|
| Непрерывное пространство | Дискретные узлы и рёбра |
| Косинусное расстояние | Пути в графе |
| Имплицитные связи | Явные отношения |
| System 1 (быстрое) | System 2 (медленное) |
| Интуиция | Логика |

**Синтез**: Hybrid intelligence = embeddings + graph

### Интенция как проекция

Вопрос пользователя **проецируется** в embedding space (см. [Question as Key Paradigm](../research/question-as-key-paradigm.md)):

```
Вопрос (интенция, незнание)
       ↓ [Embedding Model]
Query Vector (геометрическая интенция)
       ↓ [HNSW Search]
Аттракторы (проявления интенции)
```

**Философия**: Интенция становится **измеримой** через проекцию в геометрическое пространство.

---

## Альтернативные модели

### Сравнение embedding models

| Model | Dimensions | Cost (per 1M tokens) | Quality |
|-------|------------|---------------------|---------|
| **OpenAI text-embedding-3-small** | 1536 | $0.02 | Отлично |
| OpenAI text-embedding-3-large | 3072 | $0.13 | Наилучше |
| OpenAI text-embedding-ada-002 | 1536 | $0.10 | Хорошо |
| Cohere embed-english-v3.0 | 1024 | $0.10 | Отлично |
| Google Gecko | 768 | Free (limited) | Хорошо |
| Sentence-BERT (local) | 384-768 | Free | Средне |

**Почему NodeRAG использует text-embedding-3-small**:
1. **Баланс**: Хорошее качество при низкой стоимости
2. **Размерность**: 1536D достаточно для высокой точности
3. **Скорость**: Быстрая генерация
4. **Стабильность**: Mature API от OpenAI

### Локальные альтернативы

Для **self-hosted** решений:

```python
from sentence_transformers import SentenceTransformer

# Загрузка модели
model = SentenceTransformer('all-MiniLM-L6-v2')

# Генерация embedding
embedding = model.encode(text)  # 384D
```

**Преимущества**:
- Бесплатно (после загрузки модели)
- Приватность (данные не покидают сервер)
- Нет rate limits

**Недостатки**:
- Ниже качество (~80-85% от OpenAI)
- Меньше размерность (384-768D)
- Требует GPU для быстрой генерации

---

## Ограничения и edge cases

### 1. Token Limit

**OpenAI limit**: 8,191 tokens per input

**NodeRAG решение**: Semantic Units уже chunked (~200-300 tokens), поэтому никогда не превышаем лимит.

### 2. Multilingual Support

**OpenAI text-embedding-3-small**: Поддерживает 100+ языков, но качество варьируется:

```
English: 0.95 quality
Russian: 0.90 quality
Chinese: 0.88 quality
Arabic: 0.85 quality
```

**NodeRAG**: Работает для многоязычных корпусов, но с небольшой потерей качества для non-English.

### 3. Out-of-Distribution Texts

Если текст сильно отличается от обучающего набора OpenAI (например, специализированный жаргон):

```python
text = "The hypergolic propellant UDMH exhibits toxicity..."
# Embedding может быть менее точным
```

**Решение**: Fine-tuning модели (не поддерживается text-embedding-3-small) или domain adaptation.

### 4. Embedding Drift

При обновлении модели OpenAI embeddings могут **слегка измениться**:

```
text-embedding-3-small (v1) → emb₁
text-embedding-3-small (v2) → emb₂

cosine(emb₁, emb₂) ≈ 0.98 (очень близко, но не идентично)
```

**NodeRAG**: При обновлении модели нужно **пересоздать все embeddings** и **пересобрать HNSW index**.

---

## Код examples

### Генерация single embedding

```python
from NodeRAG.LLM import OpenAI_Embedding

# Инициализация
embedding_client = OpenAI_Embedding(
    model_name="text-embedding-3-small",
    api_keys=None,  # Использует OPENAI_API_KEY из env
    Config=None
)

# Генерация
text = "Renewable energy technologies are advancing rapidly."
embedding = embedding_client.API_client([text])

print(f"Embedding shape: {len(embedding[0])}")  # 1536
print(f"First 5 dimensions: {embedding[0][:5]}")
```

### Batch generation

```python
texts = [
    "Solar panels convert sunlight to electricity.",
    "Wind turbines generate power from wind.",
    "Hydroelectric dams use water flow."
]

embeddings = embedding_client.API_client(texts)

print(f"Generated {len(embeddings)} embeddings")
for i, emb in enumerate(embeddings):
    print(f"Text {i+1}: {len(emb)} dimensions")
```

### Async batch generation

```python
import asyncio

texts = ["text1", "text2", ..., "text100"]

async def generate():
    embeddings = await embedding_client.API_client_async(texts)
    return embeddings

embeddings = asyncio.run(generate())
```

### Полный embedding pipeline

```python
from NodeRAG.build.pipeline.embedding import Embedding_pipeline
from NodeRAG.config import NodeConfig

config = NodeConfig(...)

# Инициализация
pipeline = Embedding_pipeline(config)

# Генерация для всех узлов без embeddings
await pipeline.main()

# Результат: все узлы имеют embeddings в Parquet файле
```

---

## Практические рекомендации

### Для разработчиков

1. **Batch requests**: Всегда используйте batch API (до 100 текстов за раз)
   ```python
   # Плохо: 100 отдельных запросов
   for text in texts:
       emb = client.request(text)

   # Хорошо: 1 batch запрос
   embeddings = client.request(texts)
   ```

2. **Cache embeddings**: Сохраняйте в Parquet для быстрой загрузки
   ```python
   # Сохранение
   storage(embeddings).save_parquet('embeddings.parquet')

   # Загрузка
   embeddings = storage.load('embeddings.parquet')
   ```

3. **Monitor costs**: Логируйте число токенов
   ```python
   import tiktoken

   enc = tiktoken.get_encoding("cl100k_base")
   tokens = len(enc.encode(text))
   cost = tokens * 0.00000002  # $0.02 per 1M tokens
   ```

4. **Handle errors**: Используйте retry logic
   ```python
   @backoff.on_exception(backoff.expo, Exception, max_tries=4)
   def generate_embedding(text):
       return client.request(text)
   ```

### Для исследователей

1. **Evaluate quality**: Сравните с ground truth
   ```python
   from sklearn.metrics.pairwise import cosine_similarity

   query_emb = embed("renewable energy")
   doc1_emb = embed("solar power")
   doc2_emb = embed("fossil fuels")

   sim1 = cosine_similarity([query_emb], [doc1_emb])[0][0]  # Ожидаем высокий
   sim2 = cosine_similarity([query_emb], [doc2_emb])[0][0]  # Ожидаем низкий
   ```

2. **Visualize embedding space**: t-SNE или UMAP
   ```python
   from sklearn.manifold import TSNE
   import matplotlib.pyplot as plt

   # Уменьшаем 1536D → 2D
   tsne = TSNE(n_components=2)
   embeddings_2d = tsne.fit_transform(embeddings)

   plt.scatter(embeddings_2d[:, 0], embeddings_2d[:, 1])
   plt.show()
   ```

3. **Analyze distribution**: Гистограмма norm и variance
   ```python
   import numpy as np

   norms = [np.linalg.norm(emb) for emb in embeddings]
   variances = [np.var(emb) for emb in embeddings]

   plt.hist(norms, bins=50)
   plt.title('Embedding Norms Distribution')
   plt.show()
   ```

---

## Дополнительные ресурсы

### Документация

- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- [LLM and AI Dependencies](../dependencies/llm-and-ai.md)
- [Vector Search Dependencies](../dependencies/vector-search.md)

### Научные статьи

1. **Mikolov, T., et al. (2013)**. "Efficient Estimation of Word Representations in Vector Space." arXiv:1301.3781.

2. **Devlin, J., et al. (2018)**. "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding." arXiv:1810.04805.

3. **Neelakantan, A., et al. (2022)**. "Text and Code Embeddings by Contrastive Pre-Training." arXiv:2201.10005.

### Связанные алгоритмы

- [HNSW Algorithm](../graph/hnsw-algorithm.md) — использует embeddings
- [Cosine Similarity](./cosine-similarity.md) — мера близости embeddings
- [K-means Clustering](./kmeans-clustering.md) — кластеризация embeddings
- [Batch Processing](./batch-processing.md) — эффективная генерация

---

**Последнее обновление**: 2025-11-13

**См. также**:
- [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)
- [Звёздная архитектура](../research/star-pattern-architecture.md)
- [Indexing Transformations](../transform/indexing-transformations.md)
