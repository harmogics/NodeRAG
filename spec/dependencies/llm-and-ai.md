# LLM и AI зависимости

## Обзор

Эта категория включает библиотеки для взаимодействия с языковыми моделями, обработки текста и структурированной валидации данных.

---

## 1. OpenAI SDK

**Пакет**: `openai==1.66.3`

**Официальный сайт**: https://github.com/openai/openai-python

### Назначение в NodeRAG

OpenAI SDK обеспечивает **основное взаимодействие** с LLM для всех текстовых трансформаций.

### Применение

**Файл**: `NodeRAG/LLM/LLM.py:83-198`

```python
from openai import OpenAI, AsyncOpenAI, RateLimitError, Timeout, APIConnectionError

class OPENAI(LLM):
    def __init__(self, model_name: str, api_keys: str | None, Config: ModelConfig):
        self.client = OpenAI(api_key=self.api_keys)
        self.client_async = AsyncOpenAI(api_key=self.api_keys)

    async def _create_completion_async(self, messages, response_format=None):
        params = {
            "model": self.model_name,
            "messages": messages,
            **self.config
        }
        if response_format:
            # Structured output с Pydantic
            response = await self.client_async.beta.chat.completions.parse(**params)
            return response.choices[0].message.parsed.model_dump_json()
        else:
            # Обычный text output
            response = await self.client_async.chat.completions.create(**params)
            return response.choices[0].message.content.strip()
```

### Используемые функции

| Функция | Назначение | Использование |
|---------|-----------|---------------|
| `chat.completions.create()` | LLM inference | Text generation |
| `chat.completions.parse()` | Structured output | Pydantic validation |
| `embeddings.create()` | Embedding generation | Vectorization |

### Интеграция

**Трансформации**:
- Text Decomposition → `gpt-4o` / `gpt-4o-mini`
- Attribute Generation → `gpt-4o`
- Community Summarization → `gpt-4o`
- Query Decomposition → `gpt-4o-mini`
- Answer Generation → `gpt-4o`

**Embeddings**:
- `text-embedding-3-small` (1536 dim) — основная модель для векторизации

### Зависимости OpenAI SDK

| Зависимость | Назначение |
|-------------|-----------|
| `httpx` | HTTP клиент (sync + async) |
| `pydantic` | Validation для structured outputs |
| `anyio` | Async I/O framework |
| `distro` | Platform detection |
| `sniffio` | Async library detection |
| `tqdm` | Progress bars |
| `jiter` | Fast JSON parsing |

---

## 2. Pydantic

**Пакет**: `pydantic==2.10.6`, `pydantic-core==2.27.2`

**Официальный сайт**: https://docs.pydantic.dev/

### Назначение в NodeRAG

Pydantic обеспечивает **строгую типизацию** и **валидацию** всех LLM outputs.

### Применение

**Файл**: `NodeRAG/utils/prompt/json_format.py`

```python
from pydantic import BaseModel

class semantic_group(BaseModel):
    semantic_unit: str
    entities: list[str]
    relationships: list[str]

class text_decomposition(BaseModel):
    Output: list[semantic_group]

class relationship_reconstraction(BaseModel):
    source: str
    relationship: str
    target: str

class decomposed_text(BaseModel):
    elements: list[str]

class elements(BaseModel):
    title: str
    description: str

class High_level_element(BaseModel):
    high_level_elements: list[elements]
```

### Роль в системе

#### 1. Schema Definition

Каждый промпт имеет **соответствующую Pydantic модель**:

| Промпт | Pydantic Schema | Output |
|--------|----------------|---------|
| Text Decomposition | `text_decomposition` | Structured semantic units |
| Relationship Reconstruction | `relationship_reconstraction` | Triplet (source, rel, target) |
| Attribute Generation | — | Free text (no schema) |
| Community Summarization | `High_level_element` | Theme list |
| Query Decomposition | `decomposed_text` | Entity list |
| Answer Generation | — | Free text (no schema) |

#### 2. Automatic Validation

```python
# OpenAI parse API автоматически использует Pydantic schema
response = await client.beta.chat.completions.parse(
    response_format=text_decomposition  # Pydantic model
)
# Response автоматически валидируется
```

**Преимущества**:
- ✅ Type safety
- ✅ Automatic validation
- ✅ Clear error messages
- ✅ JSON serialization/deserialization

#### 3. Runtime Type Checking

Pydantic проверяет:
- Типы полей (str, list, etc.)
- Required vs. optional fields
- Constraints (min length, pattern, etc.)

### Зависимости Pydantic

| Зависимость | Назначение |
|-------------|-----------|
| `pydantic-core` | Rust-based validation engine |
| `typing-extensions` | Advanced type hints |
| `annotated-types` | Type annotations |

---

## 3. Tiktoken

**Пакет**: `tiktoken==0.9.0`

**Официальный сайт**: https://github.com/openai/tiktoken

### Назначение в NodeRAG

Tiktoken выполняет **токенизацию** для подсчёта токенов и определения границ chunks.

### Применение

**Файл**: `NodeRAG/utils/token_utils.py` (referenced)

```python
import tiktoken

def get_encoding(encoding_name: str = "cl100k_base"):
    """Get BPE encoding for GPT-4/GPT-3.5-turbo"""
    return tiktoken.get_encoding(encoding_name)

def token_counter(text: str, encoding=None) -> int:
    """Count tokens in text"""
    if encoding is None:
        encoding = get_encoding()
    tokens = encoding.encode(text)
    return len(tokens)

def token_limit(text: str, max_tokens: int = 8000) -> bool:
    """Check if text exceeds token limit"""
    return token_counter(text) > max_tokens
```

### Использование

#### 1. Semantic Chunking

**Файл**: `spec/semantic-chunking-strategy.md`

```python
chunk_size = 1048  # tokens

while start < text_len:
    end = start + chunk_size * 4  # Approximate char count
    current_chunk = text[start:end]

    # Check token count
    while token_counter(current_chunk) > chunk_size:
        # Reduce chunk size
        ...
```

**Зачем**: Обеспечить, что каждый chunk **не превышает** LLM context window.

#### 2. Context Truncation

**Файл**: `NodeRAG/build/component/community.py:74-82`

```python
def get_important_node_query(self):
    """Incrementally add nodes until token limit"""
    query_old = ''
    for i in range(len(weights_dict) + 1):
        query = self.get_query(weights_dict.keys()[:i])
        if self.token_counter.token_limit(query):
            return query_old  # Return previous valid query
        query_old = query
```

**Зачем**: Предотвратить **overflow** context window при community summarization.

#### 3. Cost Estimation

```python
tokens = token_counter(prompt + response)
cost = tokens * price_per_1k_tokens / 1000
```

### Encoding Models

| Model | Encoding | Vocabulary Size | Used for |
|-------|----------|-----------------|----------|
| GPT-4, GPT-3.5-turbo | `cl100k_base` | ~100k tokens | All prompts |
| GPT-4o | `o200k_base` | ~200k tokens | Newer models |

### Зависимости Tiktoken

| Зависимость | Назначение |
|-------------|-----------|
| `regex` | Advanced pattern matching for BPE |
| `requests` | Download encoding data |

---

## 4. Google Generative AI SDK

**Пакет**: `google-api-core>=2.24.2` (через requirements.in)

**Официальный сайт**: https://ai.google.dev/

### Назначение в NodeRAG

Поддержка **Google Gemini** как альтернативного LLM provider.

### Применение

**Файл**: `NodeRAG/LLM/LLM.py:253-386`

```python
from ..utils.lazy_import import LazyImport
genai = LazyImport("google.genai")

class Gemini(LLM):
    def __init__(self, model_name: str, api_keys: str | None, Config: ModelConfig):
        self.client = genai.Client(api_key=api_keys)

    async def _create_completion_async(self, messages, response_format=None):
        params = {
            "model": self.model_name,
            "contents": messages,
        }
        if response_format:
            config = genai.types.GenerateContentConfig(
                temperature=self.config.get("temperature", 0.0),
                max_output_tokens=self.config.get("max_tokens", 10000),
                response_mime_type="application/json",
                response_schema=response_format,
            )
            response = await self.client.aio.models.generate_content(**params, config=config)
            return json.loads(response.text)
        else:
            config = genai.types.GenerateContentConfig(
                temperature=self.config.get("temperature", 0.0),
                max_output_tokens=self.config.get("max_tokens", 10000),
            )
            response = await self.client.aio.models.generate_content(**params, config=config)
            return response.text
```

### Модели Gemini

| Model | Use Case |
|-------|----------|
| `gemini-1.5-flash` | Fast, cheap inference |
| `gemini-1.5-pro` | High-quality reasoning |
| `text-embedding-004` | Embeddings (768 dim) |

### Adapter Pattern

NodeRAG использует **Adapter Pattern**, чтобы OpenAI и Gemini имели единый интерфейс:

```python
# Factory pattern
def LLM_route(config: ModelConfig) -> LLM:
    service_provider = config.get("service_provider")
    match service_provider:
        case "openai":
            return OPENAI(model_name, api_keys, config)
        case "gemini":
            return Gemini(model_name, api_keys, config)
```

**Преимущество**: Переключение provider без изменения кода.

---

## 5. Backoff

**Пакет**: `backoff==2.2.1`

**Официальный сайт**: https://github.com/litl/backoff

### Назначение в NodeRAG

Автоматический **exponential backoff** для retry logic при ошибках API.

### Применение

**Файл**: `NodeRAG/LLM/LLM.py:108-153`

```python
import backoff
from openai import RateLimitError, Timeout, APIConnectionError
from json.decoder import JSONDecodeError

class OPENAI(LLM):
    @backoff.on_exception(
        backoff.expo,  # Exponential backoff
        [RateLimitError, Timeout, APIConnectionError, JSONDecodeError],
        max_time=30,   # Max 30 seconds total retry time
        max_tries=4    # Max 4 attempts
    )
    async def _create_completion_async(self, messages, response_format=None):
        # API call
        response = await self.client_async.beta.chat.completions.parse(**params)
        return response
```

### Стратегия Backoff

**Попытки**:
```
Attempt 1: Immediate
Attempt 2: Wait ~1s
Attempt 3: Wait ~2s
Attempt 4: Wait ~4s
```

**Обрабатываемые ошибки**:
- `RateLimitError`: Rate limit exceeded
- `Timeout`: Request timeout
- `APIConnectionError`: Network issues
- `JSONDecodeError`: Malformed JSON response

### Почему важно

**Без backoff**: Одна временная ошибка → весь pipeline падает

**С backoff**: Временные ошибки → автоматический retry → success

**Философия**: **Fault tolerance** — система устойчива к временным сбоям.

---

## Зависимости зависимостей (LLM stack)

### HTTP Слой

| Пакет | Роль |
|-------|------|
| `httpx==0.28.1` | Modern HTTP client (sync + async) |
| `httpcore==1.0.7` | Low-level HTTP protocol |
| `h11==0.14.0` | HTTP/1.1 protocol implementation |
| `certifi==2025.1.31` | SSL certificate bundle |
| `idna==3.10` | Internationalized domain names |

### Async Слой

| Пакет | Роль |
|-------|------|
| `anyio==4.8.0` | Async framework abstraction (asyncio/trio) |
| `sniffio==1.3.1` | Detect async library |
| `exceptiongroup==1.2.2` | Exception groups for async |

### Type Hints

| Пакет | Роль |
|-------|------|
| `typing-extensions==4.12.2` | Backport of latest typing features |
| `annotated-types==0.7.0` | Runtime annotation support |

---

## Сравнение LLM Providers

| Аспект | OpenAI | Gemini |
|--------|--------|--------|
| **Models** | GPT-4o, GPT-4o-mini, GPT-3.5-turbo | Gemini 1.5 Pro/Flash |
| **Structured Output** | Beta API with Pydantic | JSON schema via config |
| **Embeddings** | text-embedding-3-small (1536 dim) | text-embedding-004 (768 dim) |
| **Pricing** | $2.50-$10 per 1M input tokens | $1.25-$3.50 per 1M input tokens |
| **Rate Limits** | 10,000 RPM (tier 5) | 2,000 RPM (free tier) |
| **Adapter** | `OPENAI` class | `Gemini` class |

---

## Итоговая роль в архитектуре

```
User Query
      ↓
[Query Decomposition]
      ↓
HNSW Search
      ↓
      ├─→ [Embeddings] ← tiktoken (tokenization)
      │        ↓
      │   OpenAI SDK / Gemini SDK
      │        ↓
      └─→ [LLM Transformations] ← Pydantic (validation)
                ↓
         [Answer Generation]
                ↓
        Natural Language Response

All API calls:
    ├─ Backoff (retry logic)
    ├─ HTTPX (async HTTP)
    └─ Pydantic (structured outputs)
```

### Ключевые принципы

1. **Provider Abstraction**: Unified interface для OpenAI и Gemini
2. **Structured Outputs**: Pydantic schemas для всех transformations
3. **Token Management**: Tiktoken для chunking и cost control
4. **Fault Tolerance**: Backoff для automatic recovery
5. **Async-First**: AsyncOpenAI и httpx для high throughput
