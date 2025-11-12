# Узлы Графа и Стратегия Chunking

## Обзор

NodeRAG использует гетерогенный граф с 8 типами узлов, каждый из которых играет специфическую роль в представлении знаний. Процесс chunking является многоуровневым и семантически-ориентированным.

---

## Иерархия Узлов

```
Document
    └── Text Units (semantic chunks)
            ├── Semantic Units
            │       └── связаны с Entities
            ├── Entities
            │       ├── связаны с Semantic Units
            │       ├── участвуют в Relationships
            │       └── могут иметь Attributes
            └── Relationships
                    └── связывают Entities

Communities (выявляются алгоритмически)
    └── High-Level Elements
            └── связаны с Semantic Units и Entities
```

---

## Типы Узлов: Детальное Описание

### 1. Document Nodes

**Класс**: `NodeRAG/build/component/document.py`

```python
class document(Unit_base):
    - raw_context: str          # Полный текст документа
    - path: str                 # Путь к файлу
    - hash_id: str              # SHA256(raw_context)
    - human_readable_id: str    # D00001, D00002, ...
    - text_units: List[Text_unit]
```

**Роль**:
- Представляют исходный документ
- Корневой узел для всех производных данных
- Связь с файловой системой

**Связи**:
- Родитель для Text Units

**Индексация**:
- Не имеют embeddings
- Хранятся в documents.parquet

---

### 2. Text Unit Nodes

**Класс**: `NodeRAG/build/component/text_unit.py`

```python
class Text_unit(Unit_base):
    - raw_context: str          # Текст chunk'а
    - hash_id: str              # SHA256(raw_context)
    - human_readable_id: str    # T00001, T00002, ...
```

**Роль**:
- Базовая единица для семантической обработки
- Результат семантического chunking документа
- Входные данные для LLM decomposition

**Связи**:
- Дети Document узлов
- Родители для Semantic Units (косвенно)

**Chunking стратегия**: см. раздел "Семантический Chunking"

**Индексация**:
- Имеют embeddings
- Хранятся в text.parquet

---

### 3. Semantic Unit Nodes

**Класс**: `NodeRAG/build/component/semantic_unit.py`

```python
class Semantic_unit(Unit_base):
    - raw_context: str          # Паraphrase semantic unit
    - text_hash_id: str         # Родительский Text Unit
    - hash_id: str              # SHA256(raw_context)
    - human_readable_id: str    # S00001, S00002, ...
```

**Роль**:
- Семантически связная единица информации
- Описывает одно событие, факт или концепт
- Результат LLM декомпозиции Text Unit
- Суммаризированная форма части текста

**Связи**:
- Связаны с Text Unit (через text_hash_id)
- Связаны с множеством Entity узлов
- Могут быть связаны с High-Level Elements

**Характеристики**:
- **Weight**: частота появления семантически идентичного unit
- Дедупликация через hash_id

**Индексация**:
- Имеют embeddings
- Хранятся в semantic_units.parquet

**Пример**:
```
Original Text Unit:
"In September 2024, Dr. Emily Roberts traveled to Paris to attend
the International Conference on Renewable Energy."

Semantic Unit:
"In September 2024, Dr. Emily Roberts attended the International
Conference on Renewable Energy in Paris."
```

---

### 4. Entity Nodes

**Класс**: `NodeRAG/build/component/entity.py`

```python
class Entity(Unit_base):
    - raw_context: str          # Имя сущности (UPPERCASE)
    - text_hash_id: str         # Родительский Text Unit
    - hash_id: str              # SHA256(raw_context)
    - human_readable_id: str    # E00001, E00002, ...
```

**Роль**:
- Представляют именованные сущности
- Ключевые концепты, актеры, объекты
- Связующие элементы графа

**Типы сущностей**:
- Люди (DR. EMILY ROBERTS)
- Организации (INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY)
- Места (PARIS)
- Временные метки (2024-09)
- Концепты (SOLAR PANEL EFFICIENCY)
- Любые другие именованные объекты

**Нормализация**:
- Всегда UPPERCASE
- Автоматическая дедупликация через hash_id

**Связи**:
- Связаны с Semantic Units (упоминания)
- Участвуют в Relationships (как source или target)
- Могут иметь Attributes (для важных)

**Характеристики**:
- **Weight**: количество упоминаний в разных контекстах
- Узлы с высоким weight считаются важными

**Индексация**:
- Не имеют собственных embeddings
- Хранятся в entities.parquet

**Пример**:
```json
{
  "hash_id": "abc123...",
  "human_readable_id": "E00042",
  "type": "entity",
  "context": "DR. EMILY ROBERTS",
  "text_hash_id": "def456...",
  "weight": 15
}
```

---

### 5. Relationship Nodes

**Класс**: `NodeRAG/build/component/relationship.py`

```python
class Relationship(Unit_base):
    - relationship_tuple: [source, relation, target]
    - source: Entity              # Source entity
    - target: Entity              # Target entity
    - unique_relationship: frozenset(source_hash, target_hash)
    - raw_context: str            # "source relation target"
    - hash_id: str                # SHA256(frozenset)
    - human_readable_id: str      # R00001, R00002, ...
```

**Роль**:
- Представляют связи между сущностями
- Описательная форма отношений (не просто тип)
- Агрегация контекстов для одной пары сущностей

**Структура**:
```
ENTITY_A, descriptive relation sentence, ENTITY_B
```

**Особенности**:
- Relation type может быть полным описательным предложением
- Направленность: source → target
- Дедупликация: одна пара сущностей = один Relationship узел
- Множественные контексты объединяются

**Связи в графе**:
```
Source Entity → Relationship Node → Target Entity
```

**Характеристики**:
- **Weight**: частота упоминания связи
- **unique_relationship**: frozenset для дедупликации
- Накопление контекстов при повторах

**Индексация**:
- Не имеют embeddings
- Хранятся в relationship.parquet

**Примеры**:
```
"DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
"DR. EMILY ROBERTS, explored partnerships with, EUROPEAN COMPANIES"
"DR. JOHN MILLER, conducted fieldwork in, AMAZON RAINFOREST"
```

**Реконструкция некорректных relationships**:
Если relationship не содержит ровно 3 элемента, используется LLM для реконструкции:
```python
query = relationship_reconstraction_prompt.format(relationship=relationship)
response = await API_client(query)
# Output: {source, relationship, target}
```

---

### 6. Attribute Nodes

**Класс**: `NodeRAG/build/component/attribute.py`

```python
class Attribute(Unit_base):
    - raw_context: str          # Детальное описание сущности
    - node: str                 # hash_id родительской Entity
    - hash_id: str              # SHA256(raw_context)
    - human_readable_id: str    # A00001, A00002, ...
```

**Роль**:
- Богатые описания важных сущностей
- Агрегация информации из соседних узлов
- Повышение качества retrieval для ключевых концептов

**Критерии важности** (выбор сущностей для attributes):

1. **K-core decomposition**:
   - k = round(log(|V|) × sqrt(avg_degree))
   - Entities в k-core с weight > 1

2. **Betweenness centrality**:
   - betweenness > avg × log10(|V|)
   - Entities с weight > 1

**Процесс генерации**:
1. Сбор всех соседних Semantic Units
2. Сбор всех соседних Relationships
3. Формирование промпта с entity и контекстами
4. LLM генерация description

**Связи**:
- Связаны с одной Entity узлом
- Entity.attributes = [attribute.hash_id]

**Характеристики**:
- Похожи на "character sketch" или "product description"
- До 2000 слов
- Narrative style

**Индексация**:
- Имеют embeddings
- Хранятся в attributes.parquet

**Пример структуры**:
```
Entity: "DR. EMILY ROBERTS"
Attribute: "Dr. Emily Roberts is a leading researcher in renewable
energy, specializing in solar panel efficiency improvements. In
September 2024, she attended the International Conference on Renewable
Energy in Paris, where she presented her latest research findings and
explored potential partnerships with several European companies. Her
work focuses on enhancing the efficiency of photovoltaic systems..."
```

---

### 7. High-Level Element Nodes

**Класс**: `NodeRAG/build/component/community.py` → `High_level_elements`

```python
class High_level_elements(Unit_base):
    - context: str              # Description концепта
    - title: str                # Название концепта
    - hash_id: str              # SHA256(context)
    - title_hash_id: str        # SHA256(title)
    - human_readable_id: str    # H00001, H00002, ...
    - embedding: List[float]
    - related_node: List[str]   # Связанные узлы из community
```

**Роль**:
- Концептуальный слой системы
- Темы, идеи, теории, паттерны
- Результат community detection + LLM summarization

**Процесс создания**:

1. **Community Detection**:
   - Leiden algorithm на графе
   - Modularity optimization
   - Выделение плотно связанных кластеров

2. **LLM Summarization**:
   - Сбор Semantic Units и Attributes из community
   - Промпт: извлечь high-level concepts
   - Output: список {title, description}

3. **Node Creation**:
   - Два узла: content node + title node
   - Content хранит полное описание
   - Title используется для быстрого поиска

**Связи**:
- Связаны с Semantic Units из community
- Связаны с Entities через community
- Связь title ↔ content узлов

**Связывание стратегии**:

Если узлов много (threshold превышен):
- KMeans кластеризация embeddings
- Связывание только внутри кластеров

Если узлов мало:
- Прямое связывание со всеми related узлами

**Характеристики**:
- **Weight**: всегда 1 (при создании)
- Могут покрывать multiple communities

**Индексация**:
- Имеют embeddings (и content и title)
- Хранятся в:
  - high_level_elements.parquet (content)
  - high_level_elements_titles.parquet (titles)

**Примеры**:
```json
{
  "title": "Environmental Conservation Efforts",
  "description": "This concept encompasses the collaborative work of
  researchers and organizations aimed at protecting natural ecosystems.
  It includes fieldwork in endangered areas like the Amazon Rainforest,
  academic research on renewable energy solutions, and international
  partnerships for sustainable development."
}
```

---

### 8. Community Summary Nodes

**Класс**: `NodeRAG/build/component/community.py` → `Community_summary`

**Роль**:
- Промежуточный объект для генерации High-Level Elements
- Не сохраняется как узел графа
- Используется только в процессе генерации

---

## Семантический Chunking

### Стратегия Разбиения

**Класс**: `NodeRAG/utils/text_spliter.py` → `SemanticTextSplitter`

**Параметры**:
```python
chunk_size: int = 1048          # Максимум токенов
model_name: str = "gpt-4o-mini" # Для подсчета токенов
```

### Алгоритм

1. **Начальная оценка**:
   ```python
   end = start + chunk_size * 4  # Предполагаем 4 символа = 1 токен
   current_chunk = text[start:end]
   ```

2. **Проверка токенов**:
   ```python
   while token_counter(current_chunk) > chunk_size:
       # Нужно уменьшить chunk
   ```

3. **Поиск семантических границ** (в порядке приоритета):
   ```python
   boundaries = ['\n\n', '\n', '。', '.', '！', '!', '？', '?', '；', ';']
   ```
   - `\n\n`: параграфы (highest priority)
   - `\n`: новые строки
   - `.`, `。`: предложения
   - `!`, `！`, `?`, `？`: вопросы и восклицания
   - `;`, `；`: сложные предложения

4. **Применение границы**:
   ```python
   boundary_pos = current_chunk.rfind(boundary)
   if boundary_pos != -1:
       end = start + boundary_pos + len(boundary)
   ```

5. **Fallback при отсутствии границ**:
   ```python
   end = start + int(len(current_chunk) // 1.2)
   ```
   - Принудительное уменьшение на ~17%
   - Продолжение итерации

6. **Finalization**:
   ```python
   chunk = current_chunk.strip()
   if chunk:
       chunks.append(chunk)
   start = end
   ```

### Особенности

**Преимущества**:
- Сохраняет семантическую целостность
- Не разрывает предложения (по возможности)
- Приоритет параграфам
- Поддержка Unicode (китайская пунктуация)

**Trade-offs**:
- Chunks могут быть разного размера
- Гарантирован максимум в chunk_size токенов
- Минимальный размер не гарантирован

### Пример Chunking

**Исходный текст** (1500 tokens):
```
Параграф 1 (600 tokens)...

Параграф 2 (500 tokens)...

Параграф 3 (400 tokens)...
```

**Результат** (chunk_size=1048):
```
Chunk 1: Параграф 1 + Параграф 2 (1100 → урезается до 1048)
  → Разбивается на границе \n\n между параграфами

Chunk 1: Параграф 1 (600 tokens)
Chunk 2: Параграф 2 + Параграф 3 (900 tokens)
```

---

## Взаимодействие Узлов

### Пример Полного Flow

**Входной документ**:
```
"In September 2024, Dr. Emily Roberts traveled to Paris to attend the
International Conference on Renewable Energy. She presented research on
solar panel efficiency."
```

**1. Document Node**:
```
D00001: [весь документ]
```

**2. Text Unit** (после semantic chunking):
```
T00001: [весь текст, так как < 1048 tokens]
```

**3. LLM Decomposition** → **Semantic Units**:
```
S00001: "In September 2024, Dr. Emily Roberts attended the
International Conference on Renewable Energy in Paris, where she
presented research on solar panel efficiency."
```

**4. Entities** (из S00001):
```
E00001: "DR. EMILY ROBERTS"
E00002: "2024-09"
E00003: "PARIS"
E00004: "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY"
E00005: "SOLAR PANEL EFFICIENCY"
```

**5. Relationships** (из S00001):
```
R00001: Source=E00001, Relation="attended", Target=E00004
R00002: Source=E00001, Relation="presented research on", Target=E00005
```

**6. Graph Structure**:
```
S00001 ----> E00001 (DR. EMILY ROBERTS)
       ----> E00002 (2024-09)
       ----> E00003 (PARIS)
       ----> E00004 (INTL CONFERENCE)
       ----> E00005 (SOLAR PANEL EFFICIENCY)

R00001 ----> E00001 (source)
       ----> E00004 (target)

R00002 ----> E00001 (source)
       ----> E00005 (target)
```

**7. Attribute** (если E00001 важна):
```
A00001 → "Dr. Emily Roberts is a researcher in renewable energy
specializing in solar technologies. In 2024, she participated in
international conferences in Paris..."
```

**8. Community Detection** + **High-Level Element**:
```
Community: {S00001, E00001, E00004, E00005, ...}
↓
H00001 (title): "Renewable Energy Research and Innovation"
H00001 (content): "This high-level concept covers cutting-edge research
in renewable energy, including solar panel technology improvements and
international scientific collaboration through conferences and
partnerships."
```

---

## Свойства Узлов

### Общие для всех

- **hash_id**: SHA256 от контента (для дедупликации)
- **human_readable_id**: Последовательный ID (D00001, T00001, ...)
- **type**: Тип узла

### Специфичные

| Узел               | Weight | Embedding | Parent Link     |
|--------------------|--------|-----------|-----------------|
| Document           | -      | -         | -               |
| Text Unit          | -      | ✓         | doc_hash_id     |
| Semantic Unit      | ✓      | ✓         | text_hash_id    |
| Entity             | ✓      | -         | text_hash_id    |
| Relationship       | ✓      | -         | text_hash_id    |
| Attribute          | (from entity) | ✓  | node (entity)   |
| High-Level Element | ✓      | ✓         | related_nodes   |

### Weight Семантика

- **Semantic Unit**: сколько раз идентичный unit появился
- **Entity**: количество различных контекстов упоминания
- **Relationship**: частота упоминания связи
- **High-Level Element**: начинается с 1, может инкрементироваться

---

## Storage Format

Все узлы хранятся в **Parquet** файлах:

```
cache/
├── documents.parquet
├── text.parquet
├── semantic_units.parquet
├── entities.parquet
├── relationship.parquet
├── attributes.parquet
├── high_level_elements.parquet
├── high_level_elements_titles.parquet
├── embedding.parquet
└── graph.pkl (NetworkX граф)
```

**Преимущества Parquet**:
- Колоночное хранение
- Компрессия
- Эффективные фильтры
- Поддержка сложных типов (List, Struct)

---

## Индексация

### Узлы с Embeddings

- Text Units
- Semantic Units
- Attributes
- High-Level Elements (content и title)

### HNSW Индексы

Для быстрого векторного поиска строятся HNSW (Hierarchical Navigable Small World) индексы.

**Параметры**:
- M: количество связей на слой
- ef_construction: размер динамического списка кандидатов

**Использование**:
- Приближенный nearest neighbor search
- Логарифмическая сложность поиска
- Trade-off между точностью и скоростью
