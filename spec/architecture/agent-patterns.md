# Agent Patterns

## Обзор

Этот документ описывает агентские и LLM-ориентированные архитектурные паттерны, используемые в NodeRAG для абстрагирования взаимодействий с языковыми моделями и embedding-сервисами.

---

## 1. Abstract LLM Interface Pattern

### Назначение
Создание универсального интерфейса для различных LLM providers (OpenAI, Gemini, Azure и др.), позволяющего легко переключаться между провайдерами без изменения бизнес-логики.

### Реализация

**Файл**: `NodeRAG/LLM/LLM_base.py:1-72`

```python
from typing import TypeVar, Generic, Dict, Any

# Generic type parameters
I = TypeVar('I')  # Input type
O = TypeVar('O')  # Output type

class LLMBase(ABC, Generic[I, O]):
    """Abstract base class for all LLM providers"""

    def __init__(self,
                 model_name: str,
                 api_keys: str | None,
                 config: ModelConfig | None = None):
        self.model_name = model_name
        self.api_keys = api_keys
        self.config = config

    @abstractmethod
    def predict(self, input: I) -> O:
        """Synchronous prediction"""
        pass

    @abstractmethod
    async def predict_async(self, input: I) -> O:
        """Asynchronous prediction"""
        pass
```

### Преимущества

1. **Provider Agnostic**: Бизнес-логика не зависит от конкретного провайдера
2. **Type Safety**: Использование Generic types для type hints
3. **Async Support**: Встроенная поддержка асинхронных операций
4. **Unified Interface**: Одинаковый интерфейс для всех провайдеров

### Использование

```python
# Любой LLM provider реализует этот интерфейс
llm: LLMBase = get_llm_provider()

# Синхронный вызов
result = llm.predict(input_data)

# Асинхронный вызов
result = await llm.predict_async(input_data)
```

---

## 2. Adapter Pattern для LLM Providers

### Назначение
Адаптация различных API (OpenAI, Gemini) к единому интерфейсу через implementation-specific adapters.

### Реализация

**Файл**: `NodeRAG/LLM/LLM.py:83-198`

#### OpenAI Adapter

```python
class OPENAI(LLM):
    """Adapter for OpenAI API"""

    def __init__(self, model_name: str, api_keys: str | None, Config: ModelConfig):
        super().__init__(model_name, api_keys, Config)

        if self.api_keys is None:
            self.api_keys = os.getenv("OPENAI_API_KEY")

        self.client = OpenAI(api_key=self.api_keys)
        self.client_async = AsyncOpenAI(api_key=self.api_keys)
        self.config = self.extract_config(Config)

    def extract_config(self, config: ModelConfig) -> ModelConfig:
        """Transform generic config to OpenAI-specific format"""
        options = {
            "max_tokens": config.get("max_tokens", 10000),
            "temperature": config.get("temperature", 0.0),
        }
        return options

    @backoff.on_exception(backoff.expo,
                          [RateLimitError, Timeout, APIConnectionError, JSONDecodeError],
                          max_time=30,
                          max_tries=4)
    async def _create_completion_async(self, messages, response_format=None):
        params = {
            "model": self.model_name,
            "messages": messages,
            **self.config
        }
        if response_format:
            params["response_format"] = response_format
            response = await self.client_async.beta.chat.completions.parse(**params)
            json_response = response.choices[0].message.parsed.model_dump_json()
            return json.loads(json_response)
        else:
            response = await self.client_async.chat.completions.create(**params)
            return response.choices[0].message.content.strip()

    def messages(self, input: LLM_message) -> OpenAI_message:
        """Transform generic message format to OpenAI format"""
        messages = []
        if input.get("system_prompt"):
            messages.append({
                "role": "system",
                "content": input["system_prompt"]
            })
        content = [{"type": "text", "text": input["query"]}]
        messages.append({"role": "user", "content": content})
        return messages
```

#### Gemini Adapter

```python
class Gemini(LLM):
    """Adapter for Google Gemini API"""

    def __init__(self, model_name: str, api_keys: str | None, Config: ModelConfig):
        super().__init__(model_name, api_keys)
        if api_keys is None:
            api_keys = os.getenv('GOOGLE_API_KEY')

        self.client = genai.Client(api_key=api_keys)
        self.config = self.extract_config(Config)

    def messages(self, input: LLM_message) -> Gemini_content:
        """Transform generic message format to Gemini format"""
        query = ''
        if input.get("system_prompt"):
            query = 'system_prompt:\n' + input["system_prompt"]
        query = query + '\nquery:\n' + input["query"]
        content = [query]
        return content

    @backoff.on_exception(backoff.expo,
                          [ResourceExhausted, TooManyRequests, InternalServerError, JSONDecodeError],
                          max_time=30,
                          max_tries=4)
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

### Ключевые особенности

1. **Message Format Adaptation**: Каждый adapter преобразует generic message format в provider-specific format
2. **Config Extraction**: Адаптация generic config к provider-specific parameters
3. **Error Handling**: Provider-specific exception types обрабатываются индивидуально
4. **Backoff Strategy**: Экспоненциальный backoff для retry logic

---

## 3. Factory Pattern для LLM Routing

### Назначение
Динамическое создание правильного LLM adapter на основе конфигурации.

### Реализация

**Файл**: `NodeRAG/LLM/LLM_route.py:12-34`

```python
def LLM_route(config: ModelConfig) -> LLM:
    """Route the request to the appropriate LLM service provider"""

    service_provider = config.get("service_provider")
    model_name = config.get("model_name")
    embedding_model_name = config.get("embedding_model_name", None)
    api_keys = config.get("api_keys", None)

    match service_provider:
        case "openai":
            return OPENAI(model_name, api_keys, config)
        case "openai_embedding":
            return OpenAI_Embedding(embedding_model_name, api_keys, config)
        case "gemini":
            return Gemini(model_name, api_keys, config)
        case "gemini_embedding":
            return Gemini_Embedding(embedding_model_name, api_keys, config)
        case _:
            raise ValueError("Service provider not supported")
```

### Использование

```python
config = {
    "service_provider": "openai",
    "model_name": "gpt-4o",
    "max_tokens": 10000,
    "temperature": 0.0
}

llm = LLM_route(config)
# llm теперь является экземпляром OPENAI класса
```

### Преимущества

1. **Centralized Creation Logic**: Вся логика создания в одном месте
2. **Easy Extension**: Добавление нового provider требует только добавления case в match
3. **Configuration-Driven**: Provider выбирается через конфигурацию, не hardcode

---

## 4. Rate Limiting Pattern с Semaphore

### Назначение
Контроль количества одновременных запросов к LLM API для соблюдения rate limits и предотвращения перегрузки.

### Реализация

**Файл**: `NodeRAG/LLM/LLM_route.py:37-71`

```python
class API_client():
    """Wrapper around LLM with rate limiting"""

    def __init__(self, config: ModelConfig) -> None:
        self.llm = LLM_route(config)
        self.rate_limit = config.get("rate_limit", 50)
        self.semaphore = asyncio.Semaphore(self.rate_limit)

    @cache_error_async
    async def __call__(self, input: I, *, cache_path: str|None=None, meta_data: Dict|None=None) -> O:
        """Execute LLM request with rate limiting"""
        async with self.semaphore:
            response = await self.llm.predict_async(input)
        return response

    @cache_error
    def request(self, input: I, *, cache_path: str|None=None, meta_data: Dict|None=None) -> O:
        """Synchronous request"""
        response = self.llm.predict(input)
        return response

    def stream_chat(self, input: I):
        """Streaming chat response"""
        yield from self.llm.stream_chat(input)
```

### Использование

```python
# В pipeline
async_task = []
for text in texts:
    # Semaphore автоматически ограничивает количество одновременных запросов
    async_task.append(client(input_data))

# Максимум rate_limit запросов выполняются одновременно
await asyncio.gather(*async_task)
```

**Файл**: `NodeRAG/build/pipeline/text_pipeline.py:27-35`

```python
async def text_decomposition_pipline(self) -> None:
    async_task = []
    self.config.tracker.set(len(self.texts), 'Text Decomposition')

    for index, row in self.texts.iterrows():
        text = Text_unit(row['context'], row['hash_id'], row['text_id'])
        # Каждый запрос проходит через rate limiter
        async_task.append(text.text_decomposition(self.config))

    # Semaphore контролирует параллелизм
    await asyncio.gather(*async_task)
```

### Преимущества

1. **Automatic Rate Limiting**: Semaphore автоматически управляет очередью
2. **Configurable**: rate_limit настраивается через конфигурацию
3. **Non-blocking**: Использует async/await для эффективного использования ресурсов
4. **Provider Compliance**: Предотвращает нарушение rate limits провайдера

---

## 5. Singleton Pattern для LLM State

### Назначение
Обеспечение единого экземпляра LLM client во всей системе для эффективного использования ресурсов.

### Реализация

**Файл**: `NodeRAG/LLM/LLM_state.py:1-25`

```python
# Global state
api_client = None
embedding_client = None

def set_api_client(client: LLMBase | None):
    """Set the global API client"""
    if client is None:
        raise ValueError("Please provide a valid API client information")
    global api_client
    api_client = client
    return api_client

def get_api_client():
    """Get the global API client"""
    return api_client

def set_embedding_client(client: LLMBase | None):
    """Set the global embedding client"""
    if client is None:
        raise ValueError("Please provide a valid API client information")
    global embedding_client
    embedding_client = client
    return embedding_client

def get_embedding_client():
    """Get the global embedding client"""
    return embedding_client
```

### Использование

**Файл**: `NodeRAG/utils/prompt/prompt_manager.py:9-12`

```python
from ...LLM.LLM_state import get_api_client

# Получение глобального клиента
API_request = get_api_client()

class prompt_manager():
    def translate(self, prompt: str):
        prompt = translate_prompt.format(language=self.language, prompt=prompt)
        input_dict = {'prompt': prompt}
        # Использование глобального клиента
        response = API_request.request(input_dict)
        return response
```

### Преимущества

1. **Resource Efficiency**: Один клиент для всех операций
2. **Connection Pooling**: Переиспользование HTTP connections
3. **Consistent Configuration**: Единая конфигурация для всей системы
4. **Easy Access**: Простой доступ из любого модуля

---

## 6. Backoff Decorator Pattern

### Назначение
Автоматическая retry logic с экспоненциальным backoff для обработки временных сбоев API.

### Реализация

**Файл**: `NodeRAG/LLM/LLM.py:108-153`

```python
from backoff import on_exception, expo
from openai import RateLimitError, Timeout, APIConnectionError
from json.decoder import JSONDecodeError

class OPENAI(LLM):
    @backoff.on_exception(backoff.expo,
                          [RateLimitError, Timeout, APIConnectionError, JSONDecodeError],
                          max_time=30,  # Maximum 30 seconds total retry time
                          max_tries=4)  # Maximum 4 attempts
    def _create_completion(self, messages, response_format=None):
        """Create completion with automatic retry on transient errors"""
        params = {
            "model": self.model_name,
            "messages": messages,
            **self.config
        }

        if response_format:
            params["response_format"] = response_format
            response = self.client.beta.chat.completions.parse(**params)
            json_response = response.choices[0].message.parsed.model_dump_json()
            json_response = json.loads(json_response)
            return json_response
        else:
            response = self.client.chat.completions.create(**params)
            return response.choices[0].message.content.strip()
```

### Backoff Strategy

**Экспоненциальный backoff**:
```
Попытка 1: Немедленно
Попытка 2: Ждем ~1 секунда
Попытка 3: Ждем ~2 секунды
Попытка 4: Ждем ~4 секунды
```

**Обработка ошибок**:
- `RateLimitError`: Rate limit превышен → retry
- `Timeout`: Таймаут запроса → retry
- `APIConnectionError`: Проблема соединения → retry
- `JSONDecodeError`: Некорректный JSON → retry

### Использование

```python
# Decorator применяется автоматически
response = await llm._create_completion_async(messages, response_format)
# Если произойдет RateLimitError, автоматически retry с backoff
```

### Преимущества

1. **Automatic Recovery**: Автоматическое восстановление от временных сбоев
2. **Exponential Backoff**: Избегает thundering herd problem
3. **Configurable**: Настраиваемые max_time и max_tries
4. **Provider-Specific**: Разные exception types для разных providers

---

## 7. Error Caching Pattern

### Назначение
Сохранение failed requests для последующего retry, обеспечивая надежность pipeline.

### Реализация

**Файл**: `NodeRAG/logging/error.py` (referenced)

```python
@cache_error_async
async def __call__(self, input: I, *, cache_path: str|None=None, meta_data: Dict|None=None) -> O:
    """
    Если запрос fails, сохраняется в cache_path для последующего retry
    """
    async with self.semaphore:
        response = await self.llm.predict_async(input)
    return response
```

**Файл**: `NodeRAG/build/pipeline/text_pipeline.py:48-74`

```python
async def rerun(self) -> None:
    """Rerun failed requests from error cache"""
    self.texts = self.load_texts()

    # Загрузка failed requests из cache
    with open(self.config.LLM_error_cache, 'r', encoding='utf-8') as f:
        LLM_store = []
        for line in f:
            line = json.loads(line)
            LLM_store.append(line)

    # Очистка cache после загрузки
    clear_cache(self.config.LLM_error_cache)

    # Повторная попытка
    await self.rerun_request(LLM_store)
    self.config.tracker.close()
    await self.text_decomposition_pipline()

async def rerun_request(self, LLM_store: List[Dict]) -> None:
    tasks = []
    self.config.tracker.set(len(LLM_store), 'Rerun LLM on error cache of text decomposition pipeline')

    for store in LLM_store:
        input_data = store['input_data']
        store.pop('input_data')
        input_data.update({'response_format': self.config.prompt_manager.text_decomposition})
        tasks.append(self.request_save(input_data, store, self.config))

    await asyncio.gather(*tasks)
```

### Error Detection

**Файл**: `NodeRAG/build/pipeline/text_pipeline.py:87-101`

```python
def check_error_cache(self) -> None:
    """Check if there are errors in the cache and raise exception if found"""
    if os.path.exists(self.config.LLM_error_cache):
        num = 0

        with open(self.config.LLM_error_cache, 'r', encoding='utf-8') as f:
            for line in f:
                num += 1

        if num > 0:
            self.config.console.print(f"[red]LLM Error Detected, There are {num} errors")
            self.config.console.print("[red]Please check the error log")
            self.config.console.print("[red]The error cache is named LLM_error.jsonl, stored in the cache folder")
            self.config.console.print("[red]Please fix the error and run the pipeline again")
            raise Exception("Error happened in text decomposition pipeline, Error cached.")
```

### Workflow

```
Text Pipeline Start
      ↓
Process all texts
      ↓
Some requests fail → Saved to LLM_error.jsonl
      ↓
check_error_cache() detects errors
      ↓
User fixes issues (e.g., API key, rate limits)
      ↓
rerun() reprocesses failed requests
      ↓
Success → Continue pipeline
```

### Преимущества

1. **Fault Tolerance**: Pipeline не останавливается при единичных сбоях
2. **Debugging**: Все failed requests сохранены для анализа
3. **Efficient Retry**: Retry только failed requests, не весь dataset
4. **Data Integrity**: Не теряем обработанные данные

---

## 8. Lazy Import Pattern

### Назначение
Отложенная загрузка тяжелых библиотек (OpenAI, Google AI) для ускорения startup time.

### Реализация

**Файл**: `NodeRAG/utils/lazy_import.py` (referenced)

**Файл**: `NodeRAG/LLM/LLM.py:45-51`

```python
from ..utils.lazy_import import LazyImport

# Библиотеки не загружаются до первого использования
OpenAI = LazyImport('openai', 'OpenAI')
AzureOpenAI = LazyImport('openai', 'AzureOpenAI')
AsyncOpenAI = LazyImport('openai', 'AsyncOpenAI')
AsyncAzureOpenAI = LazyImport('openai', 'AsyncAzureOpenAI')
genai = LazyImport("google.genai")
```

### Преимущества

1. **Faster Startup**: Модули загружаются только при необходимости
2. **Reduced Memory**: Неиспользуемые библиотеки не загружаются в память
3. **Conditional Dependencies**: Можно использовать систему без установки всех providers

---

## 9. Prompt Manager Pattern

### Назначение
Централизованное управление всеми промптами с поддержкой multilingual templates.

### Реализация

**Файл**: `NodeRAG/utils/prompt/prompt_manager.py:14-98`

```python
class prompt_manager():
    def __init__(self, language: str):
        self.language = language

    @property
    def text_decomposition(self):
        """Get text decomposition prompt in the configured language"""
        match self.language:
            case 'English':
                return text_decomposition_prompt
            case "Chinese":
                return text_decomposition_prompt_Chinese
            case _:
                return self.translate(text_decomposition_prompt)

    @property
    def relationship_reconstraction(self):
        match self.language:
            case 'English':
                return relationship_reconstraction_prompt
            case "Chinese":
                return relationship_reconstraction_prompt_Chinese
            case _:
                return self.translate(relationship_reconstraction_prompt)

    @property
    def attribute_generation(self):
        match self.language:
            case 'English':
                return attribute_generation_prompt
            case "Chinese":
                return attribute_generation_prompt_Chinese
            case _:
                return self.translate(attribute_generation_prompt)

    @property
    def community_summary(self):
        match self.language:
            case 'English':
                return community_summary
            case "Chinese":
                return community_summary_Chinese
            case _:
                return self.translate(community_summary)

    @property
    def decompose_query(self):
        match self.language:
            case 'English':
                return decompos_query
            case "Chinese":
                return decompos_query_Chinese
            case _:
                return self.translate(decompos_query)

    @property
    def answer(self):
        match self.language:
            case 'English':
                return answer_prompt
            case "Chinese":
                return answer_prompt_Chinese
            case _:
                return self.translate(answer_prompt)

    def translate(self, prompt: str):
        """Translate prompt to the configured language using LLM"""
        prompt = translate_prompt.format(language=self.language, prompt=prompt)
        input_dict = {'prompt': prompt}
        response = API_request.request(input_dict)
        return response

    @property
    def text_decomposition_json(self):
        """Get Pydantic schema for text decomposition"""
        return text_decomposition

    @property
    def relationship_reconstraction_json(self):
        return relationship_reconstraction

    @property
    def high_level_element_json(self):
        return High_level_element

    @property
    def decomposed_text_json(self):
        return decomposed_text
```

### Использование

```python
# В graph pipeline
prompt = self.config.prompt_manager.relationship_reconstraction.format(
    relationship=relationship
)
json_format = self.config.prompt_manager.relationship_reconstraction_json

input_data = {'query': prompt, 'response_format': json_format}
response = await self.API_request(input_data)
```

**Файл**: `NodeRAG/build/pipeline/graph_pipeline.py:183-189`

### Преимущества

1. **Centralized Management**: Все промпты в одном месте
2. **Multilingual Support**: Автоматическое переключение языка
3. **Lazy Translation**: Перевод on-demand для нестандартных языков
4. **Schema Coupling**: Промпты и их JSON schemas управляются вместе
5. **Type Safety**: Properties обеспечивают type hints

---

## 10. Structured Output Pattern (Pydantic)

### Назначение
Обеспечение строгой типизации и валидации LLM output через Pydantic schemas.

### Реализация

**Файл**: `NodeRAG/utils/prompt/json_format.py:1-14`

```python
from pydantic import BaseModel

class text_decomposition(BaseModel):
    """Schema for text decomposition output"""
    Output: list[semantic_group]

class semantic_group(BaseModel):
    semantic_unit: str
    entities: list[str]
    relationships: list[str]

class relationship_reconstraction(BaseModel):
    """Schema for relationship reconstruction output"""
    source: str
    relationship: str
    target: str
```

### Интеграция с LLM

**Файл**: `NodeRAG/LLM/LLM.py:119-126`

```python
if response_format:
    # Pydantic schema передается в API
    params["response_format"] = response_format
    response = self.client.beta.chat.completions.parse(**params)

    # Автоматическая валидация и десериализация
    json_response = response.choices[0].message.parsed.model_dump_json()
    json_response = json.loads(json_response)
    return json_response
```

### Преимущества

1. **Type Safety**: Compile-time type checking
2. **Automatic Validation**: Pydantic автоматически валидирует структуру
3. **Documentation**: Schema служит документацией для output format
4. **Error Detection**: Ранее обнаружение некорректного формата output

---

## 11. Command Pattern для Agent Actions

### Назначение
Инкапсуляция agent operations как commands для динамического вызова.

### Реализация

**Файл**: `NodeRAG/build/component/unit.py:15-22`

```python
class Unit_base(ABC):
    def call_action(self, action: str, *args, **kwargs) -> None:
        """
        Dynamically invoke action method on the component.
        Supports calling any method by name.
        """
        method = getattr(self, action, None)
        if callable(method):
            method(*args, **kwargs)
        else:
            raise AttributeError(f"Action '{action}' not found or not callable")
```

### Использование

**Примеры agent actions**:

```python
# Text decomposition action
class Text_unit(Unit_base):
    async def text_decomposition(self, config):
        """Action: decompose text into semantic units"""
        prompt = config.prompt_manager.text_decomposition.format(text=self.content)
        response = await config.client(prompt)
        # Process response...

# Community summary action
class Community_summary(Unit_base):
    async def generate_community_summary(self):
        """Action: generate summary for community"""
        query = self.get_query()
        input = {'query': query, 'response_format': self.prompt.high_level_element_json}
        self.response = await self.client(input)
```

**Dynamic invocation**:

```python
unit.call_action('text_decomposition', config)
community.call_action('generate_community_summary')
```

### Преимущества

1. **Decoupling**: Вызывающий код не зависит от конкретных методов
2. **Dynamic Dispatch**: Действия могут выбираться динамически
3. **Extensibility**: Легко добавлять новые actions

---

## 12. Context Truncation Pattern

### Назначение
Интеллектуальное усечение контекста при превышении token limits с сохранением наиболее важной информации.

### Реализация

**Файл**: `NodeRAG/build/component/community.py:56-82`

```python
class Community_summary(Unit_base):
    def get_normal_query(self):
        """Get query with all semantic units"""
        content = ''
        for node in self.used_unit:
            content += self.mapper.get(node, 'context') + '\n'
        query = self.prompt.community_summary.format(content=content)
        return query

    def get_important_node_query(self):
        """Get query with only most important nodes (when token limit exceeded)"""
        weights_dict = SortedDict()

        # Calculate importance by weight of neighbors
        for name in self.used_unit:
            weight = 0
            for neighbour in self.graph.neighbors(name):
                weight += self.graph[neighbour]['weight']
            weights_dict[name] = weight

        weights_dict = reversed(weights_dict)
        query_old = ''

        # Incrementally add nodes until token limit
        for i in range(len(weights_dict) + 1):
            query = self.get_query(weights_dict.keys()[:i])
            if self.token_counter.token_limit(query):
                return query_old  # Return previous valid query
            query_old = query

    def get_query(self):
        """Main method: try full query, fallback to important nodes"""
        query = self.get_normal_query()
        if self.token_counter.token_limit(query):
            return self.get_important_node_query()
        return query
```

### Strategy

1. **Try Full Context**: Сначала пытается включить весь контекст
2. **Check Token Limit**: Проверяет, превышает ли token limit
3. **Fallback to Important Nodes**: Если превышает, использует только важные узлы
4. **Incremental Addition**: Добавляет узлы по убыванию важности до достижения лимита

### Преимущества

1. **Graceful Degradation**: Не падает при больших контекстах
2. **Importance Preservation**: Сохраняет наиболее важную информацию
3. **Configurable**: Token limit настраивается
4. **Deterministic**: Воспроизводимый выбор контекста

---

## Заключение

Agent patterns в NodeRAG обеспечивают:

- ✅ **Provider Abstraction**: Единый интерфейс для всех LLM providers
- ✅ **Type Safety**: Pydantic schemas для structured output
- ✅ **Fault Tolerance**: Backoff, error caching, retry logic
- ✅ **Rate Limiting**: Automatic throttling с semaphores
- ✅ **Multilingual Support**: Centralized prompt management
- ✅ **Resource Efficiency**: Singleton state, lazy imports
- ✅ **Context Management**: Intelligent truncation для token limits

Эти паттерны делают систему надежной, расширяемой и эффективной при работе с различными LLM providers.
