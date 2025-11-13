# Pipeline Architecture Patterns

## Обзор

NodeRAG использует несколько ключевых архитектурных паттернов на уровне pipeline, которые обеспечивают модульность, расширяемость и надежность системы обработки документов и генерации ответов.

---

## 1. State Machine Pattern (Конечный автомат)

### Описание

**Файл**: `NodeRAG/build/Node.py:23-262`

Система построена как **конечный автомат** с явно определенными состояниями и переходами. Это обеспечивает предсказуемость выполнения pipeline и возможность восстановления после ошибок.

### Реализация

```python
class State(Enum):
    INIT = "INIT"
    DOCUMENT_PIPELINE = "Document pipeline"
    TEXT_PIPELINE = "Text pipeline"
    GRAPH_PIPELINE = "Graph pipeline"
    ATTRIBUTE_PIPELINE = "Attribute pipeline"
    EMBEDDING_PIPELINE = "Embedding pipeline"
    SUMMARY_PIPELINE = "Summary pipeline"
    INSERT_TEXT = "Insert text pipeline"
    HNSW_PIPELINE = "HNSW pipeline"
    FINISHED = "FINISHED"
    ERROR = "ERROR"
```

### Последовательность состояний

```python
self.state_sequence = [
    State.INIT,
    State.DOCUMENT_PIPELINE,
    State.TEXT_PIPELINE,
    State.GRAPH_PIPELINE,
    State.ATTRIBUTE_PIPELINE,
    State.EMBEDDING_PIPELINE,
    State.SUMMARY_PIPELINE,
    State.INSERT_TEXT,
    State.HNSW_PIPELINE,
    State.FINISHED
]
```

### State Transition Logic

```python
async def state_transition(self):
    try:
        while True:
            self.update_state_tree()
            index = self.state_sequence.index(self.Current_state)

            if self.Current_state != State.FINISHED:
                # Переход к следующему состоянию
                self.Current_state = self.state_sequence[index+1]

            if self.Current_state == State.FINISHED:
                # Проверка incremental mode
                if self.Is_incremental:
                    self.Current_state = State.DOCUMENT_PIPELINE
                    self.Is_incremental = False
                else:
                    return

            # Выполнение pipeline для текущего состояния
            await self.state_pipeline_map[self.Current_state](self.config).main()

    except Exception as e:
        # Error handling
        self.store_state()
        raise Exception(f'Error in {self.Current_state}.{e}')
```

### State-to-Pipeline Mapping

```python
self.state_pipeline_map = {
    State.DOCUMENT_PIPELINE: document_pipline,
    State.TEXT_PIPELINE: text_pipline,
    State.GRAPH_PIPELINE: Graph_pipeline,
    State.ATTRIBUTE_PIPELINE: Attribution_generation_pipeline,
    State.EMBEDDING_PIPELINE: Embedding_pipeline,
    State.SUMMARY_PIPELINE: SummaryGeneration,
    State.INSERT_TEXT: Insert_text,
    State.HNSW_PIPELINE: HNSW_pipeline
}
```

### Преимущества

1. **Явная последовательность**: Четкий порядок выполнения этапов
2. **Восстановление**: Возможность сохранения и загрузки состояния
3. **Отладка**: Легко отследить, на каком этапе произошла ошибка
4. **Расширяемость**: Легко добавить новые состояния
5. **Визуализация**: State tree для мониторинга прогресса

### Пример использования

```python
node_rag = NodeRag(config)

# Загрузить состояние если было прерывание
node_rag.load_state()

# Запустить pipeline
node_rag.run()  # async

# После завершения или ошибки состояние сохраняется
node_rag.store_state()
```

---

## 2. Observer Pattern (Наблюдатель)

### Описание

**Файл**: `NodeRAG/build/Node.py:95-102`

Pipeline поддерживает **Observer Pattern** для уведомления внешних компонентов о изменениях состояния. Это позволяет декуплировать UI от бизнес-логики.

### Реализация

```python
class NodeRag:
    def __init__(self, config: NodeConfig):
        self.observers = []

    def add_observer(self, observer):
        self.observers.append(observer)

    def notify_state_change(self):
        for observer in self.observers:
            observer.update(self.Current_state.value)

    @Current_state.setter
    def Current_state(self, state: State):
        self._Current_state = state
        self.notify_state_change()  # Уведомление observers
```

### Observer Interface

```python
# Предполагаемый интерфейс observer
class StateObserver:
    def update(self, state: str):
        """Вызывается при изменении состояния"""
        print(f"State changed to: {state}")
```

### Пример использования

```python
class WebUIObserver:
    def update(self, state: str):
        # Обновить прогресс-бар в web UI
        websocket.send({"type": "state_update", "state": state})

# Добавление observer
node_rag = NodeRag(config)
node_rag.add_observer(WebUIObserver())

# При изменении состояния observer автоматически уведомляется
```

### Преимущества

1. **Decoupling**: UI отделен от business logic
2. **Multiple observers**: Можно подписать несколько observers
3. **Real-time updates**: Немедленное уведомление об изменениях
4. **Testability**: Легко тестировать без UI

---

## 3. Template Method Pattern (Шаблонный метод)

### Описание

**Файл**: `NodeRAG/build/pipeline/document_pipeline.py`

Каждый pipeline следует **единообразному интерфейсу** с методом `main()`. Это обеспечивает предсказуемость и упрощает расширение.

### Базовая структура

```python
class BasePipeline:
    def __init__(self, config: NodeConfig):
        self.config = config

    async def main(self):
        """Template method - определяет алгоритм"""
        raise NotImplementedError
```

### Конкретная реализация

```python
class document_pipline:
    def __init__(self, config: NodeConfig):
        self.config = config
        self.documents_path = self.load_document_path()
        self.indices = self.config.indices

    @info_timer(message='Document Pipeline')
    async def main(self):
        """Шаблон выполнения document pipeline"""
        self.integrity_check()      # 1. Проверка целостности
        self.increment_doc()         # 2. Incremental processing
        self.store_text_data()       # 3. Сохранение text units
        self.store_documents_data()  # 4. Сохранение documents
        self.store_readable_index()  # 5. Сохранение индексов
```

### Алгоритм Template Method

```
1. integrity_check()
   ├─ Проверить существование cache директории
   ├─ Проверить completion статус
   └─ Очистить если incomplete

2. increment_doc()
   ├─ Загрузить существующие doc IDs
   ├─ Вычислить новые documents
   └─ Фильтровать уже обработанные

3. store_text_data()
   ├─ Разбить documents на text units
   ├─ Создать Parquet records
   └─ Сохранить с append mode

4. store_documents_data()
   ├─ Создать document records
   └─ Сохранить metadata

5. store_readable_index()
   └─ Сохранить human-readable индексы
```

### Преимущества

1. **Единообразие**: Все pipelines имеют одинаковый интерфейс
2. **Расширяемость**: Легко добавить новый pipeline
3. **Поддерживаемость**: Понятная структура каждого этапа
4. **Testability**: Каждый шаг можно тестировать отдельно

---

## 4. Chain of Responsibility Pattern (Цепочка обязанностей)

### Описание

Pipeline этапы формируют **цепочку обработки**, где каждый этап отвечает за свою часть преобразования и передает результат следующему.

### Визуализация цепочки

```
Document Pipeline
      ↓ (text_units)
Text Pipeline
      ↓ (semantic_units, entities, relationships)
Graph Pipeline
      ↓ (knowledge graph)
Attribute Pipeline
      ↓ (enriched graph with attributes)
Embedding Pipeline
      ↓ (embeddings)
Summary Pipeline
      ↓ (high-level elements)
Insert Text Pipeline
      ↓ (updated data structures)
HNSW Pipeline
      ↓ (HNSW index)
```

### Data Flow между этапами

```python
# Document Pipeline → выход
text_units = []  # Сохранено в cache/text.parquet

# Text Pipeline → вход
text_units = storage.load_parquet('cache/text.parquet')

# Text Pipeline → выход
semantic_units = []  # Сохранено в cache/semantic_units.parquet
entities = []        # Сохранено в cache/entities.parquet
relationships = []   # Сохранено в cache/relationships.parquet

# Graph Pipeline → вход
semantic_units = storage.load_parquet('cache/semantic_units.parquet')
entities = storage.load_parquet('cache/entities.parquet')
relationships = storage.load_parquet('cache/relationships.parquet')

# Graph Pipeline → выход
knowledge_graph = nx.Graph()  # Сохранено в cache/base_graph.pkl
```

### Persistence между этапами

```python
class Pipeline:
    def __init__(self, config):
        self.config = config

    async def main(self):
        # 1. Загрузить результаты предыдущего этапа
        input_data = self.load_previous_stage()

        # 2. Обработать
        output_data = self.process(input_data)

        # 3. Сохранить для следующего этапа
        self.save_for_next_stage(output_data)
```

### Преимущества

1. **Fault tolerance**: Каждый этап может быть перезапущен независимо
2. **Modularity**: Изменение одного этапа не влияет на другие
3. **Debugging**: Можно инспектировать промежуточные результаты
4. **Incremental processing**: Новые документы проходят через цепочку

---

## 5. Async Pipeline Pattern (Асинхронный конвейер)

### Описание

Все pipelines используют **async/await** для эффективной обработки I/O операций (LLM API calls, disk I/O).

### Реализация

```python
class NodeRag:
    async def _run_async(self):
        """Асинхронное выполнение pipeline"""
        self.load_state()

        # Инициализация
        self.Is_incremental = await INIT_pipeline(self.config).main()

        if self.Error_type != State.NO_ERROR:
            await self.error_handler()

        if self.Error_type == State.NO_ERROR:
            await self.state_transition()

    def run(self):
        """Синхронная обертка"""
        asyncio.run(self._run_async())
```

### Async в pipelines

```python
class text_pipline:
    async def main(self):
        # Параллельная обработка text units
        tasks = []
        for text_unit in text_units:
            task = self.decompose_text_async(text_unit)
            tasks.append(task)

        # Ожидание всех результатов
        results = await asyncio.gather(*tasks)

        return results
```

### Batch Processing с async

```python
async def process_batch(items, batch_size=10):
    """Обработка items батчами для контроля concurrency"""
    results = []

    for i in range(0, len(items), batch_size):
        batch = items[i:i+batch_size]
        batch_results = await asyncio.gather(
            *[process_item(item) for item in batch]
        )
        results.extend(batch_results)

    return results
```

### Преимущества

1. **Performance**: Параллельная обработка LLM requests
2. **Efficiency**: Не блокирует на I/O операциях
3. **Scalability**: Можно обрабатывать много документов одновременно
4. **Rate limiting**: Легко контролировать concurrency level

---

## 6. Error Recovery Pattern (Восстановление после ошибок)

### Описание

**Файл**: `NodeRAG/build/Node.py:214-242`

Система поддерживает **автоматическое восстановление** после ошибок с сохранением прогресса.

### Error Types

```python
class State(Enum):
    # ... other states
    ERROR = "ERROR"           # Общая ошибка
    ERROR_LOG = "ERROR_LOG"   # Ошибка логирована
    ERROR_CACHE = "ERROR_CACHE"  # Ошибка кэша
    NO_ERROR = "NO_ERROR"
```

### Error Handler

```python
async def error_handler(self):
    self.update_state_tree()

    if self.Error_type in [State.ERROR_LOG, State.ERROR]:
        # Повторить выполнение с начала этапа
        self.console.print("[red]Error logged. Rerun from current state.[/red]")
        try:
            await self.state_pipeline_map[self.Current_state](self.config).main()
        except Exception as e:
            self.store_state()
            raise f'Error in {self.Current_state} pipeline. {e}'

    if self.Error_type == State.ERROR_CACHE:
        # Повторить с rerun (использует кэш)
        self.console.print("[red]Error cached. Rerun from current state.[/red]")
        try:
            await self.state_pipeline_map[self.Current_state](self.config).rerun()
        except Exception as e:
            self.store_state()
            raise f'Error in {self.Current_state} pipeline. {e}'

    self.Error_type = State.NO_ERROR
```

### State Persistence

```python
def store_state(self):
    """Сохранение состояния на диск"""
    state_dict = {
        'Current_state': self.Current_state.value,
        'Error_type': self.Error_type.value,
        'Is_incremental': self.Is_incremental
    }
    json.dump(state_dict, open(self.config.state_path, 'w'))

def load_state(self):
    """Загрузка состояния с диска"""
    if os.path.exists(self.config.state_path):
        state_dict = json.load(open(self.config.state_path, 'r'))
        self.Current_state = State(state_dict['Current_state'])
        self.Error_type = State(state_dict['Error_type'])
        self.Is_incremental = state_dict['Is_incremental']
```

### Recovery Workflow

```
1. Pipeline запускается
2. Ошибка на этапе GRAPH_PIPELINE
3. State сохраняется:
   {
     "Current_state": "Graph pipeline",
     "Error_type": "ERROR_CACHE",
     "Is_incremental": false
   }
4. User перезапускает pipeline
5. load_state() восстанавливает Current_state
6. error_handler() детектирует ERROR_CACHE
7. Повторяет GRAPH_PIPELINE.rerun()
8. Продолжает со следующего этапа
```

### Преимущества

1. **Resilience**: Система может восстанавливаться после сбоев
2. **No data loss**: Прогресс сохраняется
3. **Intelligent retry**: Разные стратегии для разных типов ошибок
4. **User-friendly**: Автоматическое восстановление без ручного вмешательства

---

## 7. Incremental Processing Pattern (Инкрементальная обработка)

### Описание

**Файл**: `NodeRAG/build/pipeline/document_pipeline.py:102-111`

Pipeline поддерживает **инкрементальную обработку** — добавление новых документов без переиндексации существующих.

### Реализация

```python
def increment_doc(self) -> None:
    """Фильтрация уже обработанных документов"""
    if os.path.exists(self.config.documents_path):
        # Загрузить hash IDs уже обработанных documents
        exist_doc_id = storage.load_parquet(
            self.config.documents_path
        )['doc_hash_id'].tolist()

        # Вычислить новые documents (set difference)
        increment_doc_id = list(set(self.hash_ids) - set(exist_doc_id))

        # Обработать только новые
        self._documents = [
            doc for doc in self.documents
            if doc.hash_id in increment_doc_id
        ]
    else:
        # Первый запуск - обработать все
        self._documents = self.documents

    self.documents_path = [doc.path for doc in self.documents]
    self._hash_ids = None
```

### Hash-based Detection

```python
import hashlib

class document:
    @property
    def hash_id(self):
        """Hash контента документа для детекции изменений"""
        return hashlib.sha256(
            self.raw_context.encode()
        ).hexdigest()
```

### Append Mode в storage

```python
def store_documents_data(self):
    """Сохранение с append mode"""
    doc_list = [...]

    storage(doc_list).save_parquet(
        self.config.documents_path,
        append=os.path.exists(self.config.documents_path)  # Append если файл существует
    )
```

### Workflow

```
First Run:
  - Обработать все documents в input/
  - Сохранить в cache/documents.parquet
  - hash_ids = [hash1, hash2, hash3]

Add new document:
  - Новый document в input/ → hash4
  - Load exist_doc_id = [hash1, hash2, hash3]
  - Detect increment = [hash4]
  - Обработать только hash4
  - Append к cache/documents.parquet
```

### Преимущества

1. **Efficiency**: Не переобрабатывает существующие документы
2. **Scalability**: Можно добавлять документы постепенно
3. **Cost savings**: Экономия на LLM API calls
4. **Time savings**: Быстрее чем full re-indexing

---

## 8. Cache Integrity Pattern (Целостность кэша)

### Описание

**Файл**: `NodeRAG/build/pipeline/document_pipeline.py:23-29, 92-100`

Система проверяет **целостность кэша** и автоматически очищает incomplete cache.

### Реализация

```python
def integrity_check(self):
    """Проверка целостности cache директории"""
    if not os.path.exists(self.config.cache):
        os.makedirs(self.config.cache)
    elif self.cache_completion_check():
        # Cache полный - продолжить
        pass
    else:
        # Cache incomplete - очистить
        self.delete_cache()

def cache_completion_check(self) -> bool:
    """Проверка что все необходимые файлы существуют"""
    files_name = ['documents.parquet', 'text.parquet', 'indices.json']
    files = os.listdir(self.config.cache)
    return all([file in files for file in files_name])

def delete_cache(self) -> None:
    """Очистка incomplete cache"""
    for file in os.listdir(self.config.cache):
        os.remove(os.path.join(self.config.cache, file))
    self.config.console.print('[red]Incomplete cache deleted[/red]')
```

### Expected Cache Files

```
cache/
├── documents.parquet          # Document metadata
├── text.parquet               # Text units
├── indices.json               # Human-readable indices
├── semantic_units.parquet     # Semantic units
├── entities.parquet           # Entities
├── relationships.parquet      # Relationships
├── attributes.parquet         # Attributes
├── high_level_elements.parquet  # High-level elements
├── base_graph.pkl             # Knowledge graph
├── hnsw_graph.pkl             # HNSW graph
└── hnsw_index/                # HNSW index files
```

### Atomic Operations

```python
# Вместо частичного сохранения, сохраняем атомарно:
import tempfile
import shutil

def atomic_save(data, path):
    """Атомарное сохранение файла"""
    # Сохранить во временный файл
    temp_path = path + '.tmp'
    storage(data).save_parquet(temp_path)

    # Атомарно переместить (на Unix системах)
    shutil.move(temp_path, path)
```

### Преимущества

1. **Data integrity**: Гарантия целостности кэша
2. **Failure recovery**: Автоматическая очистка после сбоя
3. **Consistency**: Всегда валидное состояние
4. **User experience**: Не нужно ручное вмешательство

---

## 9. Progress Tracking Pattern (Отслеживание прогресса)

### Описание

Pipelines предоставляют **real-time feedback** о прогрессе выполнения через console и observers.

### Реализация

```python
def update_state_tree(self):
    """Визуализация прогресса"""
    self.console.clear()
    tree = Tree("[cyan]🚀 Processing Pipeline[/cyan]")
    index = self.state_sequence.index(self.Current_state)

    # Отметить завершенные этапы
    for i in range(index + 1):
        tree.add(f"[green]{self.state_sequence[i].value} Done[/green]")

    self.console.print(tree)
```

### Progress Bar в pipelines

```python
from tqdm import tqdm

class document_pipline:
    def store_text_data(self):
        text_list = []

        # Tracker для прогресса
        self.config.tracker.set(len(self.documents), desc="Processing text")

        for doc in self.documents:
            doc.split()
            for text in doc.text_units:
                text_list.append({...})

            # Обновить прогресс
            self.config.tracker.update()

        self.config.tracker.close()
```

### Rich Console Output

```python
from rich.console import Console

console = Console()

# Цветной вывод с форматированием
console.print("[green]Pipeline finished.[/green]")
console.print(f"[blue]Processing {self.Current_state.value}...[/blue]")
console.print("[red]Error occurred![/red]")
```

### Преимущества

1. **User feedback**: Пользователь видит прогресс
2. **Debugging**: Легко понять где система находится
3. **Time estimation**: Можно оценить время завершения
4. **Professional UX**: Приятный визуальный feedback

---

## Заключение

Pipeline архитектура NodeRAG демонстрирует использование **классических Design Patterns**:

1. **State Machine** - управление жизненным циклом
2. **Observer** - decoupling UI от logic
3. **Template Method** - единообразие pipelines
4. **Chain of Responsibility** - модульная обработка
5. **Async** - эффективность I/O
6. **Error Recovery** - resilience
7. **Incremental Processing** - scalability
8. **Cache Integrity** - data integrity
9. **Progress Tracking** - user experience

Эти паттерны обеспечивают:
- ✅ Модульность и расширяемость
- ✅ Fault tolerance и recovery
- ✅ Эффективность обработки
- ✅ Хороший UX
- ✅ Поддерживаемость кода

---

## Ссылки

- **State Machine**: `NodeRAG/build/Node.py`
- **Pipelines**: `NodeRAG/build/pipeline/`
- **Storage**: `NodeRAG/storage/storage.py`
- **Config**: `NodeRAG/config/Node_config.py`
