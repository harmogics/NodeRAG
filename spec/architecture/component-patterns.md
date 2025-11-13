# Component Architecture Patterns

## Обзор

NodeRAG использует компонентную архитектуру для представления различных семантических единиц. Ключевые паттерны обеспечивают type safety, extensibility и semantic consistency.

---

## 1. Abstract Base Class Pattern (ABC)

### Описание

**Файл**: `NodeRAG/build/component/unit.py`

Все компоненты наследуются от **Unit_base**, определяющего общий интерфейс для всех семантических единиц.

### Base Interface

```python
from abc import ABC, abstractmethod

class Unit_base(ABC):

    @property
    @abstractmethod
    def hash_id(self):
        """Уникальный hash-based идентификатор"""
        ...

    @property
    @abstractmethod
    def human_readable_id(self):
        """Человекочитаемый идентификатор"""
        ...

    def call_action(self, action: str, *args, **kwargs) -> None:
        """Динамический вызов методов компонента"""
        method = getattr(self, action, None)

        if callable(method):
            method(*args, **kwargs)
        else:
            raise ValueError(f"Action {action} not found")
```

### Component Hierarchy

```
Unit_base (ABC)
    ├── document
    ├── text_unit
    ├── semantic_unit
    ├── entity
    ├── relationship
    ├── attribute
    └── community
```

### Преимущества

1. **Interface enforcement**: Гарантия наличия hash_id и human_readable_id
2. **Polymorphism**: Единообразная работа с разными типами компонентов
3. **Type safety**: ABC предотвращает создание incomplete implementations
4. **Extensibility**: Легко добавить новые типы компонентов

---

## 2. Hash-based Identity Pattern

### Описание

Каждый компонент имеет **content-based hash ID** для уникальной идентификации и deduplication.

### Реализация

```python
import hashlib

class semantic_unit(Unit_base):
    def __init__(self, content: str):
        self.content = content

    @property
    def hash_id(self):
        """Content-based hash"""
        return hashlib.sha256(
            self.content.encode()
        ).hexdigest()
```

### Deduplication

```python
# Автоматическая дедупликация по hash_id
seen_hashes = set()
unique_units = []

for unit in semantic_units:
    if unit.hash_id not in seen_hashes:
        seen_hashes.add(unit.hash_id)
        unique_units.append(unit)
```

### Incremental Updates

```python
# Детектировать новые/измененные документы
existing_doc_hashes = load_existing_hashes()
new_documents = [
    doc for doc in documents
    if doc.hash_id not in existing_doc_hashes
]
```

### Преимущества

1. **Deduplication**: Автоматическое удаление дубликатов
2. **Change detection**: Детектирование модифицированного контента
3. **Content addressing**: Идентификация по содержанию, не по имени файла
4. **Distributed consistency**: Одинаковый контент → одинаковый hash везде

---

## 3. Lazy Loading Pattern

### Описание

**Файл**: `NodeRAG/build/pipeline/document_pipeline.py:36-55`

Компоненты используют **lazy loading** для оптимизации памяти и производительности.

### Реализация

```python
class document_pipline:
    def __init__(self, config: NodeConfig):
        self.config = config
        self.documents_path = self.load_document_path()
        self._documents = None  # Cached value
        self._hash_ids = None
        self._human_readable_id = None

    @property
    def documents(self):
        """Lazy loading документов"""
        if self._documents is None:
            self._documents = []
            for path in self.documents_path:
                with open(path, 'r', encoding='utf-8') as f:
                    raw_context = f.read()
                self._documents.append(
                    document(raw_context, path, self.config.semantic_text_splitter)
                )
        return self._documents

    @property
    def hash_ids(self):
        """Lazy computation hash IDs"""
        if not self._hash_ids:
            self._hash_ids = [doc.hash_id for doc in self.documents]
        return self._hash_ids
```

### Кэширование результатов

```python
class Component:
    def __init__(self):
        self._cached_value = None

    @property
    def expensive_operation(self):
        """Кэшированная дорогая операция"""
        if self._cached_value is None:
            # Вычислить только один раз
            self._cached_value = self._compute()
        return self._cached_value
```

### Преимущества

1. **Memory efficiency**: Данные загружаются по требованию
2. **Performance**: Вычисление только при необходимости
3. **Caching**: Результаты кэшируются
4. **Scalability**: Можно работать с большими датасетами

---

## 4. Builder Pattern (Композиция компонентов)

### Описание

Компоненты строятся **композиционно** из более простых единиц.

### Document → Text Units

```python
class document(Unit_base):
    def __init__(self, raw_context: str, path: str, text_splitter):
        self.raw_context = raw_context
        self.path = path
        self.text_splitter = text_splitter
        self._text_units = None

    def split(self):
        """Разбиение документа на text units"""
        if self._text_units is None:
            # Использовать SemanticTextSplitter
            chunks = self.text_splitter.split(self.raw_context)

            # Создать text_unit компоненты
            self._text_units = [
                text_unit(chunk, self.hash_id, i)
                for i, chunk in enumerate(chunks)
            ]

    @property
    def text_units(self):
        if self._text_units is None:
            self.split()
        return self._text_units
```

### Text Unit → Semantic Units

```python
class text_unit(Unit_base):
    def __init__(self, content: str, doc_hash_id: str, index: int):
        self.content = content
        self.doc_hash_id = doc_hash_id
        self.index = index
        self._semantic_units = None

    async def decompose(self, llm_client):
        """Декомпозиция на semantic units"""
        if self._semantic_units is None:
            # LLM decomposition
            result = await llm_client.decompose(self.content)

            # Создать semantic_unit компоненты
            self._semantic_units = [
                semantic_unit(
                    unit['semantic_unit'],
                    unit['entities'],
                    unit['relationships']
                )
                for unit in result['Output']
            ]

        return self._semantic_units
```

### Composition Hierarchy

```
document
    └── text_units[]
            └── semantic_units[]
                    ├── entities[]
                    └── relationships[]
```

### Преимущества

1. **Hierarchical structure**: Естественная иерархия компонентов
2. **Encapsulation**: Каждый компонент знает свои дочерние элементы
3. **Lazy decomposition**: Разбиение по требованию
4. **Traceability**: Легко проследить от document к semantic unit

---

## 5. Data Transfer Object (DTO) Pattern

### Описание

Компоненты служат как **DTOs** для передачи данных между слоями системы.

### Storage Format

```python
class document(Unit_base):
    def to_dict(self) -> dict:
        """Сериализация для storage"""
        return {
            'doc_id': self.human_readable_id,
            'doc_hash_id': self.hash_id,
            'text_id': self.text_human_readable_id,
            'text_hash_id': self.text_hash_id,
            'path': self.path
        }

    @classmethod
    def from_dict(cls, data: dict):
        """Десериализация из storage"""
        return cls(
            raw_context=data['content'],
            path=data['path']
        )
```

### Parquet Serialization

```python
# Конвертация компонентов в Parquet records
def store_documents_data(self):
    doc_list = []
    for doc in self.documents:
        doc_list.append({
            'doc_id': doc.human_readable_id,
            'doc_hash_id': doc.hash_id,
            'text_id': doc.text_human_readable_id,
            'text_hash_id': doc.text_hash_id,
            'path': doc.path
        })

    # Сохранить как Parquet
    storage(doc_list).save_parquet(
        self.config.documents_path,
        append=os.path.exists(self.config.documents_path)
    )
```

### Graph Serialization

```python
# Конвертация компонентов в graph nodes
def to_graph_node(component):
    return {
        'id': component.hash_id,
        'type': component.__class__.__name__,
        'content': component.content,
        'properties': component.properties
    }
```

### Преимущества

1. **Layer decoupling**: Разделение business logic от persistence
2. **Format flexibility**: Легко изменить формат хранения
3. **Serialization**: Простая сериализация/десериализация
4. **Testing**: Легко создавать test fixtures

---

## 6. Flyweight Pattern (Разделяемые данные)

### Описание

Компоненты разделяют **immutable данные** для экономии памяти.

### String Interning

```python
import sys

class entity(Unit_base):
    def __init__(self, name: str):
        # Intern строки для экономии памяти
        self.name = sys.intern(name.upper())
```

### Shared References

```python
# Entities являются shared между semantic units
entity_pool = {}

def get_entity(name: str):
    """Flyweight factory для entities"""
    name_normalized = name.upper()

    if name_normalized not in entity_pool:
        entity_pool[name_normalized] = entity(name_normalized)

    return entity_pool[name_normalized]

# Использование
semantic_unit1.entities = [get_entity("DR. EMILY ROBERTS")]
semantic_unit2.entities = [get_entity("DR. EMILY ROBERTS")]

# Обе ссылаются на один объект
assert semantic_unit1.entities[0] is semantic_unit2.entities[0]
```

### Graph Weight Aggregation

```python
# В графе дублирующиеся узлы увеличивают weight
for node, data in new_graph.nodes(data=True):
    if node in self.graph:
        # Увеличить weight вместо создания дубликата
        self.graph.nodes[node]['weight'] = (
            self.graph.nodes[node].get('weight', 0) + data.get('weight', 0)
        )
    else:
        self.graph.add_node(node, **data)
```

### Преимущества

1. **Memory efficiency**: Экономия памяти для дубликатов
2. **Reference equality**: Быстрое сравнение по identity
3. **Deduplication**: Автоматическая дедупликация
4. **Performance**: Меньше allocations

---

## 7. Strategy Pattern (Pluggable Processing)

### Описание

Компоненты поддерживают **pluggable strategies** для обработки.

### Text Splitter Strategy

```python
# Разные стратегии разбиения текста
class TextSplitterStrategy(ABC):
    @abstractmethod
    def split(self, text: str) -> List[str]:
        pass

class SemanticTextSplitter(TextSplitterStrategy):
    def split(self, text: str) -> List[str]:
        # Semantic boundary splitting
        ...

class FixedSizeTextSplitter(TextSplitterStrategy):
    def split(self, text: str) -> List[str]:
        # Fixed-size chunking
        ...

# Использование
class document:
    def __init__(self, content: str, splitter: TextSplitterStrategy):
        self.content = content
        self.splitter = splitter

    def split(self):
        return self.splitter.split(self.content)
```

### LLM Strategy

```python
# Разные LLM providers
class document:
    def __init__(self, content: str, llm_client):
        self.content = content
        self.llm_client = llm_client  # Strategy

    async def process(self):
        # Использует pluggable LLM client
        result = await self.llm_client.process(self.content)
        return result
```

### Преимущества

1. **Flexibility**: Легко менять алгоритмы
2. **Testability**: Можно использовать mock strategies
3. **Extensibility**: Добавление новых strategies без изменения кода
4. **Configuration**: Runtime выбор strategy

---

## 8. Factory Pattern (Component Creation)

### Описание

Создание компонентов через **factory functions** для обеспечения consistency.

### Entity Factory

```python
def create_entity(name: str, metadata: dict = None):
    """Factory для создания entities"""
    # Нормализация
    normalized_name = name.upper().strip()

    # Валидация
    if not normalized_name:
        raise ValueError("Entity name cannot be empty")

    # Создание
    return entity(
        name=normalized_name,
        metadata=metadata or {}
    )
```

### Relationship Factory

```python
def create_relationship(source: str, relation: str, target: str):
    """Factory для создания relationships"""
    # Валидация формата
    if not all([source, relation, target]):
        raise ValueError("Relationship requires source, relation, target")

    # Нормализация entities
    source = source.upper()
    target = target.upper()

    return relationship(source=source, relation=relation, target=target)
```

### Graph Node Factory

```python
def create_graph_node(component: Unit_base, node_type: str):
    """Factory для создания graph nodes из компонентов"""
    return {
        'id': component.hash_id,
        'type': node_type,
        'content': getattr(component, 'content', ''),
        'weight': 1,
        'hash_id': component.hash_id
    }
```

### Преимущества

1. **Consistency**: Единообразное создание компонентов
2. **Validation**: Централизованная валидация
3. **Normalization**: Автоматическая нормализация
4. **Encapsulation**: Скрытие деталей создания

---

## 9. Command Pattern (call_action)

### Описание

**Файл**: `NodeRAG/build/component/unit.py:14-20`

Компоненты поддерживают **динамический вызов методов** через command pattern.

### Реализация

```python
class Unit_base(ABC):
    def call_action(self, action: str, *args, **kwargs) -> None:
        """Execute action на компоненте"""
        method = getattr(self, action, None)

        if callable(method):
            method(*args, **kwargs)
        else:
            raise ValueError(f"Action {action} not found")
```

### Использование

```python
# Динамический вызов методов
component = document(content, path)

# Вместо прямого вызова
component.split()

# Можно использовать command pattern
component.call_action('split')

# С параметрами
component.call_action('process', mode='async', timeout=30)
```

### Batch Processing

```python
# Применить одно action к множеству компонентов
components = [doc1, doc2, doc3]

for comp in components:
    comp.call_action('validate')
    comp.call_action('process')
    comp.call_action('save')
```

### Преимущества

1. **Dynamic dispatch**: Вызов методов по имени
2. **Batch operations**: Применение action к коллекции
3. **Extensibility**: Новые actions без изменения клиентского кода
4. **Logging/Monitoring**: Можно обернуть call_action для мониторинга

---

## 10. Type Hierarchy Metadata Pattern

### Описание

Компоненты хранят **type metadata** для runtime type checking и routing.

### Type Tagging

```python
class Component(Unit_base):
    @property
    def component_type(self) -> str:
        """Runtime type identifier"""
        return self.__class__.__name__

# Использование
node_type = component.component_type
# 'document', 'text_unit', 'semantic_unit', etc.
```

### Type-based Routing

```python
def process_component(component: Unit_base):
    """Type-based processing"""
    match component.component_type:
        case 'document':
            return process_document(component)
        case 'text_unit':
            return process_text_unit(component)
        case 'semantic_unit':
            return process_semantic_unit(component)
        case _:
            raise ValueError(f"Unknown type: {component.component_type}")
```

### Graph Type System

```python
# Граф хранит type для каждого node
graph.nodes[node_id]['type'] = component.component_type

# Type-based filtering
entities = [
    node for node, data in graph.nodes(data=True)
    if data['type'] == 'entity'
]
```

### Преимущества

1. **Runtime type checking**: Проверка типов в runtime
2. **Type-based routing**: Маршрутизация по типу
3. **Filtering**: Фильтрация компонентов по типу
4. **Introspection**: Рефлексия для debugging

---

## Заключение

Component архитектура NodeRAG использует **классические OOP паттерны**:

1. **Abstract Base Class** - общий интерфейс
2. **Hash-based Identity** - content addressing
3. **Lazy Loading** - оптимизация памяти
4. **Builder** - композиция компонентов
5. **DTO** - data transfer
6. **Flyweight** - разделяемые данные
7. **Strategy** - pluggable processing
8. **Factory** - централизованное создание
9. **Command** - динамический dispatch
10. **Type Metadata** - runtime type system

Эти паттерны обеспечивают:
- ✅ Type safety через ABC
- ✅ Deduplication через hash-based identity
- ✅ Memory efficiency через lazy loading и flyweight
- ✅ Extensibility через strategy и factory
- ✅ Flexibility через command pattern

---

## Ссылки

- **Base Unit**: `NodeRAG/build/component/unit.py`
- **Components**: `NodeRAG/build/component/`
- **Pipelines**: `NodeRAG/build/pipeline/`
