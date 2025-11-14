# Transformer-based Text Embeddings

## Описание алгоритма

**Transformer embeddings** — это глубокие нейросетевые модели, использующие архитектуру Transformer для преобразования текста в плотные векторные представления фиксированной размерности. В NodeRAG используется модель **OpenAI text-embedding-3-small**, которая генерирует 1536-мерные векторы.

### Архитектура Transformer

**Основные компоненты**:

1. **Self-Attention Mechanism** — позволяет модели фокусироваться на релевантных частях текста
2. **Multi-Head Attention** — параллельное внимание на разных аспектах семантики
3. **Feed-Forward Networks** — нелинейные трансформации
4. **Positional Encoding** — кодирование позиции токенов в последовательности

**Формула Self-Attention**:

```
Attention(Q, K, V) = softmax(Q·K^T / √d_k) · V

где:
- Q = queries (запросы)
- K = keys (ключи)
- V = values (значения)
- d_k = размерность ключей
```

### Процесс генерации embedding

```
Input Text
    ↓
[Tokenization] (BPE/WordPiece)
    ↓
Token IDs: [101, 2054, 2003, ..., 102]
    ↓
[Token Embeddings] + [Positional Encodings]
    ↓
Transformer Encoder Layers (12-24 слоя)
    ↓ [Self-Attention + FFN] × N
    ↓
Pooling (CLS token или Mean pooling)
    ↓
Output Embedding: ℝ^1536
```

---

## Цель применения в NodeRAG

### 1. Семантическое представление узлов

**Задача**: Преобразовать текстовое содержание узлов графа в векторы для similarity search.

**Применяется к**:
- **Semantic Units** — атомарные факты из текста
- **Entities** — именованные сущности
- **Attributes** — детальные описания важных entities
- **High-level Elements** — абстрактные темы из сообществ

```python
text = "Solar panels convert sunlight into electricity efficiently."
embedding = openai.embeddings.create(
    model="text-embedding-3-small",
    input=text
)
# embedding.data[0].embedding = [0.021, -0.045, 0.012, ..., 0.033]  # 1536 dims
```

### 2. Query Embedding для поиска

**Задача**: Преобразовать пользовательский запрос в вектор для сравнения с узлами.

```python
query = "How do solar panels work?"
query_emb = embedding_model(query)  # ℝ^1536

# Cosine similarity с узлами графа
similarities = cosine_similarity(query_emb, node_embeddings)
# → Top-k наиболее релевантных узлов
```

### 3. Кластеризация и группировка

**Задача**: Группировать семантически близкие узлы для создания связей в графе.

**Пример**: Community summarization используется k-means на embeddings для определения, какие high-level elements связаны с какими узлами сообщества.

---

## Реализация

### Тип реализации

**Сторонняя (External API)**: OpenAI Embeddings API

### Библиотека/Пакет

**Пакет**: `openai==1.66.3`

**Класс**: `OpenAI_Embedding` в `NodeRAG/LLM/LLM.py:200-245`

```python
from openai import OpenAI, AsyncOpenAI

class OpenAI_Embedding(LLM):
    def __init__(self, model_name: str, api_keys: str | None, Config: ModelConfig|None):
        if api_keys is None:
            api_keys = os.getenv("OPENAI_API_KEY")

        self.client = OpenAI(api_key=api_keys)
        self.client_async = AsyncOpenAI(api_key=api_keys)

    def _create_embedding(self, input: Embedding_message) -> Embedding_output:
        response = self.client.embeddings.create(
            model=self.model_name,  # "text-embedding-3-small"
            input=input  # Список строк (batch)
        )
        return [res.embedding for res in response.data]
```

**Документация API**: https://platform.openai.com/docs/guides/embeddings

### Альтернативные реализации

NodeRAG также поддерживает:

**Google Gemini Embeddings**:
```python
class Gemini_Embedding(LLM):
    def _create_embedding(self, input: Embedding_message) -> Embedding_output:
        response = self.client.models.embed_content(
            model=self.model_name,  # "models/text-embedding-004"
            contents=input
        )
        return [res.values for res in response.embeddings]
```

**Пакет**: `google-genai`

---

## Параметры модели

### OpenAI text-embedding-3-small

| Параметр | Значение | Описание |
|----------|----------|----------|
| **Model Name** | text-embedding-3-small | Идентификатор модели |
| **Dimensions** | 1536 | Размерность выходного вектора |
| **Max Input Tokens** | 8,191 | Максимальная длина входа |
| **Context Window** | 8,191 tokens | Окно контекста |
| **Encoding** | cl100k_base | Токенизатор (BPE) |
| **Output Type** | float32[] | Тип данных вектора |

### Ценообразование

| Метрика | Стоимость |
|---------|-----------|
| Per 1M input tokens | $0.02 |
| Per 1K input tokens | $0.00002 |
| Per document (500 tokens avg) | $0.00001 |

**Пример расчета** (200K узлов, 100 tokens/узел):
```
200,000 nodes × 100 tokens = 20M tokens
20M × $0.02/1M = $0.40
```

### Конфигурация в NodeRAG

**Файл**: `config.yaml`

```yaml
embedding:
  model: "text-embedding-3-small"
  provider: "openai"  # или "gemini"
  dimensions: 1536
  batch_size: 100  # Batch processing
```

---

## Производительность

### Скорость

| Операция | Латентность | Throughput |
|----------|-------------|------------|
| Single request (1 text) | ~100-200ms | ~5-10 QPS |
| Batch request (100 texts) | ~500-1000ms | ~100-200 texts/s |
| Async parallel (10 tasks) | ~500ms | ~1,000-2,000 texts/s |

**NodeRAG Benchmark** (200K узлов):
- Последовательная обработка: ~11 часов
- Batch processing (batch_size=100): ~17 минут
- Async + Batch (10 concurrent): ~2-3 минуты

### Память

**Single embedding**:
```
1536 dimensions × 4 bytes (float32) = 6,144 bytes ≈ 6 KB
```

**200K embeddings**:
```
200,000 × 6 KB = 1.2 GB (raw в памяти)
```

**С Parquet сжатием** (~50%):
```
1.2 GB → ~600 MB (на диске)
```

### Качество

**OpenAI text-embedding-3-small benchmarks**:

| Dataset | Accuracy | Recall@10 |
|---------|----------|-----------|
| MTEB (English) | 62.3% | 0.891 |
| MTEB (Multilingual) | 58.7% | 0.854 |
| Semantic Similarity | 0.82 (Spearman ρ) | - |

**Сравнение моделей**:

| Model | Dimensions | Quality Score | Cost (per 1M) |
|-------|------------|---------------|---------------|
| text-embedding-3-small | 1536 | 62.3 | $0.02 |
| text-embedding-3-large | 3072 | 64.6 | $0.13 |
| text-embedding-ada-002 | 1536 | 61.0 | $0.10 |
| Cohere embed-v3 | 1024 | 62.8 | $0.10 |

---

## Математическая модель

### Transformer Encoder

**Layer-by-layer трансформация**:

```
h^(0) = Embed(x) + PosEncode(x)

for l = 1 to L:
    # Multi-Head Self-Attention
    attn^(l) = MultiHead(h^(l-1))
    h̃^(l) = LayerNorm(h^(l-1) + attn^(l))

    # Feed-Forward Network
    ffn^(l) = FFN(h̃^(l))
    h^(l) = LayerNorm(h̃^(l) + ffn^(l))

# Pooling (CLS token или mean)
embedding = Pool(h^(L))
```

### Multi-Head Attention

```
MultiHead(X) = Concat(head₁, head₂, ..., head_h) · W^O

где каждая голова:
head_i = Attention(X·W_i^Q, X·W_i^K, X·W_i^V)
```

### Positional Encoding

**Sinusoidal encoding** (original Transformer):

```
PE(pos, 2i) = sin(pos / 10000^(2i/d))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d))
```

Или **Learned positional embeddings** (BERT-style).

### Normalization

**Layer Normalization**:

```
LayerNorm(x) = γ · (x - μ) / √(σ² + ε) + β

где:
- μ = mean(x)
- σ² = variance(x)
- γ, β = learnable parameters
```

---

## Связь с другими ML алгоритмами

### 1. Embeddings → HNSW (Approximate k-NN)

**Цепочка**:
```
Text → Transformer → Embedding → HNSW Index → Fast k-NN Search
```

См. [Approximate k-NN](./approximate-knn.md)

### 2. Embeddings → K-means Clustering

**Для Community Summarization**:
```
Community Nodes → Embeddings → K-means → Cluster Assignment → Graph Edges
```

См. [Clustering Algorithms](./clustering.md)

### 3. Embeddings → Cosine Similarity

**Для HNSW Search**:
```
Query Embedding ∙ Node Embeddings → Cosine Distances → Top-k Neighbors
```

**Формула**:
```
cosine_similarity(u, v) = (u · v) / (||u|| · ||v||)
```

### 4. Embeddings + Graph → Hybrid Retrieval

**Dual-mode approach**:
```
HNSW (embedding-based) ∪ Accurate Search (graph-based) → PPR Personalization
```

См. [Graph ML Algorithms](./graph-ml.md)

---

## Преимущества и ограничения

### Преимущества

✅ **Семантическое понимание**: Улавливает смысл, а не только keywords
✅ **Multilingual**: Поддерживает 100+ языков
✅ **Контекстуальность**: Учитывает контекст (в отличие от Word2Vec)
✅ **Pre-trained**: Не требует дообучения для большинства задач
✅ **Stable API**: OpenAI поддерживает стабильность версий

### Ограничения

❌ **Token Limit**: Максимум 8,191 токенов (NodeRAG решает через chunking)
❌ **Cost**: Платный API ($0.02/1M tokens)
❌ **Latency**: ~100-200ms на запрос (решается через batching)
❌ **Model Updates**: При обновлении модели нужно пересоздать все embeddings
❌ **Black Box**: Нет контроля над внутренними параметрами

### Миtigations в NodeRAG

1. **Token Limit** → Semantic chunking (~200-300 tokens per unit)
2. **Cost** → Batch processing + caching embeddings
3. **Latency** → Async + parallel requests
4. **Updates** → Версионирование embeddings cache

---

## Альтернативы

### Локальные модели (Self-hosted)

**Sentence-BERT**:
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')
embedding = model.encode(text)  # 384D
```

**Преимущества**: Бесплатно, приватность
**Недостатки**: Ниже качество (~85% от OpenAI), требует GPU

**BGE (BAAI General Embedding)**:
```python
model = SentenceTransformer('BAAI/bge-large-en-v1.5')
embedding = model.encode(text)  # 1024D
```

**Преимущества**: Open-source, хорошее качество
**Недостатки**: Требует GPU, медленнее OpenAI API

### Другие коммерческие API

| Provider | Model | Dimensions | Cost (per 1M) |
|----------|-------|------------|---------------|
| **Cohere** | embed-english-v3.0 | 1024 | $0.10 |
| **Google** | text-embedding-004 | 768 | Free (limited) |
| **Voyage AI** | voyage-large-2 | 1536 | $0.12 |
| **Together AI** | m2-bert-80M-8k-retrieval | 768 | $0.02 |

---

## Код примеры

### Генерация single embedding

```python
from NodeRAG.LLM import OpenAI_Embedding

client = OpenAI_Embedding(
    model_name="text-embedding-3-small",
    api_keys=None,  # Uses OPENAI_API_KEY env var
    Config=None
)

text = "Renewable energy is crucial for sustainability."
embedding = client.API_client([text])

print(f"Dimensions: {len(embedding[0])}")  # 1536
print(f"First 5 values: {embedding[0][:5]}")
```

### Batch generation

```python
texts = [
    "Solar panels harness solar energy.",
    "Wind turbines generate electricity from wind.",
    "Hydroelectric dams use water flow for power."
]

embeddings = client.API_client(texts)

for i, emb in enumerate(embeddings):
    print(f"Text {i+1}: {len(emb)} dimensions")
```

### Async batch processing

```python
import asyncio

async def generate_embeddings(texts):
    embeddings = await client.API_client_async(texts)
    return embeddings

texts = ["text1", "text2", ..., "text100"]
embeddings = asyncio.run(generate_embeddings(texts))
```

### Embedding pipeline (NodeRAG)

```python
from NodeRAG.build.pipeline.embedding import Embedding_pipeline

pipeline = Embedding_pipeline(config)

# Генерация для всех узлов без embeddings
await pipeline.main()

# Результат: embeddings сохранены в Parquet
```

---

## Рекомендации

### Для разработчиков

1. **Используйте batch API**: До 100 текстов за запрос
   ```python
   # Плохо: 100 запросов
   for text in texts:
       emb = client.request(text)

   # Хорошо: 1 batch запрос
   embs = client.request(texts)
   ```

2. **Кэшируйте embeddings**: Parquet формат для эффективного хранения
3. **Мониторьте token usage**: Используйте tiktoken для подсчета
4. **Retry logic**: Обрабатывайте rate limits через backoff

### Для исследователей

1. **Evaluate quality**: Сравните с ground truth через cosine similarity
2. **Visualize**: t-SNE или UMAP для визуализации embedding space
3. **Analyze distribution**: Изучите norm и variance embeddings
4. **Compare models**: Benchmark разных embedding моделей

---

## Дополнительные ресурсы

### Документация

- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
- [Vector Search Dependencies](../dependencies/vector-search.md)
- [Embedding Generation](../vectors/embedding-generation.md)

### Научные статьи

1. **Vaswani, A., et al. (2017)**. "Attention Is All You Need." NeurIPS 2017.
2. **Devlin, J., et al. (2018)**. "BERT: Pre-training of Deep Bidirectional Transformers."
3. **Neelakantan, A., et al. (2022)**. "Text and Code Embeddings by Contrastive Pre-Training."

### Связанные алгоритмы

- [Approximate k-NN (HNSW)](./approximate-knn.md)
- [K-means Clustering](./clustering.md)
- [Graph ML Algorithms](./graph-ml.md)

---

**Последнее обновление**: 2025-11-13

**См. также**: [ML Algorithms Overview](./README.md)
