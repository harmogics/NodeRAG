# Этапы Pipeline Обработки Документов

## Обзор

Pipeline NodeRAG состоит из 9 последовательных этапов, каждый из которых обогащает граф знаний новым слоем информации. Все этапы поддерживают инкрементальную обработку и обработку ошибок.

## 1. INIT Pipeline

**Файл**: `NodeRAG/build/pipeline/INIT_pipeline.py`

### Цель
Проверка структуры проекта и определение режима обработки (полная или инкрементальная).

### Этапы Работы

1. **Проверка структуры папок**
   - Валидация существования main_folder и input_folder
   - Выброс исключения при отсутствии необходимых директорий

2. **Загрузка файлов**
   - Сканирование input_folder
   - Поддержка форматов: `.txt`, `.md` или custom через `docu_type`
   - Формирование списка путей к документам

3. **Проверка инкрементальности**
   - Вычисление SHA256 хеша от списка путей к документам
   - Сравнение с предыдущим запуском
   - Определение необходимости обработки новых файлов

4. **Сохранение состояния**
   - Запись document_path_hash в JSON
   - Сохранение списка обрабатываемых документов

### Выходные Данные
- Boolean: True если обнаружены новые документы, False иначе
- Файл: `document_hash.json` с хешами и путями

---

## 2. Document Pipeline

**Файл**: `NodeRAG/build/pipeline/document_pipeline.py`

### Цель
Загрузка документов, их первичное разбиение на text units и сохранение в storage.

### Этапы Работы

1. **Integrity Check**
   - Проверка наличия cache директории
   - Валидация полноты кеша (documents.parquet, text.parquet, indices.json)
   - Удаление неполного кеша при необходимости

2. **Загрузка документов**
   ```python
   document(raw_context, path, semantic_text_splitter)
   ```
   - Чтение содержимого файлов
   - Создание объектов Document
   - Генерация hash_id (SHA256) и human_readable_id

3. **Инкрементальная обработка**
   - Загрузка существующих doc_hash_id из parquet
   - Фильтрация только новых документов
   - Обновление списка для обработки

4. **Семантическое разбиение на Text Units**
   ```python
   doc.split() → List[Text_unit]
   ```
   - Использование SemanticTextSplitter
   - Chunk size: 1048 tokens (по умолчанию)
   - Учет семантических границ: `\n\n`, `\n`, `.`, `。`, etc.

5. **Сохранение данных**
   - **documents.parquet**: метаданные документов
   - **text.parquet**: все text units с контекстом
   - **indices.json**: human-readable индексы

### Выходные Данные

**documents.parquet**:
```
- doc_id: human_readable_id
- doc_hash_id: SHA256 hash
- text_id: список ID text units
- text_hash_id: список hash ID text units
- path: путь к файлу
```

**text.parquet**:
```
- text_id: human_readable_id
- hash_id: SHA256 hash
- type: 'text'
- context: текстовое содержимое
- doc_id: родительский document ID
- doc_hash_id: родительский document hash
- embedding: None (заполняется позже)
```

---

## 3. Text Pipeline

**Файл**: `NodeRAG/build/pipeline/text_pipeline.py`

### Цель
Семантическая декомпозиция text units через LLM, извлечение semantic units, entities и relationships.

### Этапы Работы

1. **Загрузка text units** из text.parquet

2. **Инкрементальная обработка**
   - Проверка text_decomposition.jsonl на уже обработанные hash_id
   - Фильтрация необработанных text units

3. **Асинхронная декомпозиция**
   ```python
   await text.text_decomposition(config)
   ```
   - Параллельная обработка всех text units
   - Использование промпта text_decomposition_prompt
   - Structured output через JSON schema

4. **LLM запрос** (для каждого text unit):
   - **Input**: текст text unit
   - **Промпт**: text_decomposition_prompt (см. раздел LLM Agents)
   - **Output**: JSON с массивом semantic units

5. **Обработка ошибок**
   - Кеширование неудачных запросов в LLM_error.jsonl
   - Возможность rerun с повторной обработкой ошибочных запросов
   - Retry logic с exponential backoff

6. **Сохранение результатов**
   - Запись в text_decomposition.jsonl
   - Формат: {text_hash_id, text_id, response}

### Структура Output от LLM

```json
{
  "Output": [
    {
      "semantic_unit": "Summarized semantic unit text",
      "entities": ["ENTITY1", "ENTITY2", "LOCATION"],
      "relationships": [
        "ENTITY1, relation description, ENTITY2",
        "ENTITY2, works at, LOCATION"
      ]
    }
  ]
}
```

### Выходные Данные

**text_decomposition.jsonl** (одна строка на text unit):
```json
{
  "text_hash_id": "abc123...",
  "text_id": "T00001",
  "response": {
    "Output": [...]
  },
  "processed": false
}
```

---

## 4. Graph Pipeline

**Файл**: `NodeRAG/build/pipeline/graph_pipeline.py`

### Цель
Построение гетерогенного графа из результатов text decomposition.

### Этапы Работы

1. **Загрузка данных**
   - Чтение text_decomposition.jsonl
   - Загрузка существующего графа (если есть)
   - Фильтрация необработанных записей (processed=False)

2. **Асинхронное построение графа**
   ```python
   await self.graph_tasks(data)
   ```
   - Параллельная обработка всех decomposition результатов

3. **Для каждого semantic unit**:

   a) **Добавление Semantic Unit Node**
   ```python
   semantic_unit = Semantic_unit(semantic_unit_text, text_hash_id)
   G.add_node(semantic_unit.hash_id, type='semantic_unit', weight=1)
   ```
   - Узел с hash_id от контента
   - Инкремент веса при повторном добавлении

   b) **Добавление Entity Nodes**
   ```python
   entity = Entity(entity_text, text_hash_id)
   G.add_node(entity.hash_id, type='entity', weight=1)
   ```
   - Узел для каждой сущности
   - Инкремент веса для повторяющихся

   c) **Добавление Relationship Nodes**
   ```python
   relationship = Relationship(relationship_tuple, text_hash_id)
   # relationship_tuple: [source, relation, target]
   G.add_node(relationship.hash_id, type='relationship', weight=1)
   G.add_node(source.hash_id, type='entity', weight=1)
   G.add_node(target.hash_id, type='entity', weight=1)
   G.add_edge(source.hash_id, relationship.hash_id, weight=1)
   G.add_edge(relationship.hash_id, target.hash_id, weight=1)
   ```
   - Relationship как отдельный узел
   - Связывает source и target entities
   - Проверка корректности формата (3 элемента)
   - Реконструкция через LLM при некорректном формате

   d) **Связывание Semantic Unit с Entities**
   ```python
   G.add_edge(semantic_unit.hash_id, entity.hash_id, weight=1)
   ```

4. **Сохранение данных**
   - **semantic_units.parquet**: все semantic units
   - **entities.parquet**: все entities
   - **relationship.parquet**: все relationships
   - **graph.pkl**: NetworkX граф
   - Обновление processed=True в text_decomposition.jsonl

### Структура Графа

```
Semantic Unit (SU)
    ├─→ Entity A (E1)
    ├─→ Entity B (E2)
    └─→ Entity C (E3)

Relationship (R1)
    ├─→ Entity A (E1) [source]
    └─→ Entity B (E2) [target]
```

### Выходные Данные

**semantic_units.parquet**:
```
- hash_id: SHA256
- human_readable_id: S00001
- type: 'semantic_unit'
- context: текст semantic unit
- text_hash_id: родительский text unit
- weight: частота появления
- embedding: None
- insert: None
```

**entities.parquet**:
```
- hash_id: SHA256
- human_readable_id: E00001
- type: 'entity'
- context: имя сущности (UPPERCASE)
- text_hash_id: родительский text unit
- weight: частота появления
```

**relationship.parquet**:
```
- hash_id: SHA256
- human_readable_id: R00001
- type: 'relationship'
- unique_relationship: frozenset(source_hash, target_hash)
- context: "source relation target"
- text_hash_id: родительский text unit
- weight: частота появления
```

---

## 5. Attribute Pipeline

**Файл**: `NodeRAG/build/pipeline/attribute_generation.py`

### Цель
Генерация детальных описаний (атрибутов) для важных узлов графа.

### Этапы Работы

1. **Определение важных узлов**
   ```python
   NodeImportance(graph, console).main()
   ```

   Используются два метода:

   a) **K-core decomposition**
   - k = round(log(|V|) × avg_degree^0.5)
   - Отбор entity nodes с weight > 1

   b) **Betweenness centrality**
   - Вычисление betweenness для k=10 узлов
   - Отбор узлов с centrality > avg × log10(|V|)
   - Только entity nodes с weight > 1

2. **Инкрементальная фильтрация**
   - Исключение узлов с уже существующими attributes

3. **Сбор материалов для каждого узла**
   ```python
   entity = mapper.get(node, 'context')
   semantic_neighbours = [context для semantic_unit соседей]
   relationship_neighbours = [context для relationship соседей]
   ```

4. **Проверка token limit**
   - Если query превышает лимит → отбор соседей по важности
   - Сортировка соседей по весу их соседей
   - Добавление пока не достигнут token limit

5. **LLM генерация атрибута**
   ```python
   await API_client({
     'query': attribute_generation_prompt.format(
       entity=entity,
       semantic_units=semantic_neighbours,
       relationships=relationship_neighbours
     )
   })
   ```

6. **Добавление в граф**
   ```python
   attribute = Attribute(response, node)
   G.add_node(attribute.hash_id, type='attribute', weight=1)
   G.add_edge(node, attribute.hash_id, weight=1)
   G.nodes[node]['attributes'] = [attribute.hash_id]
   ```

### Выходные Данные

**attributes.parquet**:
```
- node: hash_id сущности
- type: 'attribute'
- context: сгенерированное описание
- hash_id: SHA256
- human_readable_id: A00001
- weight: вес родительской сущности
- embedding: None
```

---

## 6. Embedding Pipeline

**Файл**: `NodeRAG/build/pipeline/embedding.py`

### Цель
Генерация векторных представлений для всех узлов требующих embedding.

### Этапы Работы

1. **Загрузка Mapper**
   - Объединение text.parquet, semantic_units.parquet, attributes.parquet
   - Создание мапинга hash_id → context

2. **Поиск узлов без embedding**
   ```python
   none_embedding_ids = mapper.find_none_embeddings()
   ```
   - Проверка поля embedding: None

3. **Batch обработка**
   - Разбиение на batches (по умолчанию: embedding_batch_size)
   - Асинхронная обработка batches

4. **Генерация embeddings**
   ```python
   await embedding_client(context_list)
   ```
   - Batch запрос к embedding модели
   - Поддержка OpenAI и Gemini embedding моделей

5. **Обработка пустых контекстов**
   - Фильтрация узлов с пустым context
   - Удаление из mapper для пустых узлов

6. **Сохранение**
   - Временное кеширование в embedding_cache.jsonl
   - Вставка в embedding.parquet
   - Обновление mapper с embedding='done'

7. **Error handling**
   - Кеширование ошибок в LLM_error.jsonl
   - Поддержка rerun для повторной обработки ошибок

### Выходные Данные

**embedding.parquet**:
```
- hash_id: SHA256
- embedding: List[float] (размерность зависит от модели)
```

---

## 7. Summary Pipeline

**Файл**: `NodeRAG/build/pipeline/summary_generation.py`

### Цель
Community detection и генерация высокоуровневых концептуальных элементов.

### Этапы Работы

1. **Преобразование графа**
   ```python
   G_ig = IGraph(NetworkX_graph).to_igraph()
   ```
   - Конвертация в igraph для алгоритмов community detection

2. **Community detection**
   ```python
   partition = la.find_partition(G_ig, la.ModularityVertexPartition)
   ```
   - Использование Leiden algorithm
   - Оптимизация modularity
   - Отбор узлов с embeddings в каждом community

3. **Создание Community Summary объектов**
   ```python
   Community_summary(community_nodes, mapper, graph, config)
   ```
   - Для каждого community
   - Доступ к mapper для получения контекста

4. **Генерация high-level elements**

   Для каждого community:

   a) **Сбор контекста**
   - Semantic units из community
   - Attributes узлов в community
   - Сортировка по важности при превышении token limit

   b) **LLM запрос**
   ```python
   await client({
     'query': community_summary_prompt.format(content=context),
     'response_format': high_level_element_json
   })
   ```

   c) **Output структура**:
   ```json
   {
     "high_level_elements": [
       {
         "title": "Concept Title",
         "description": "Detailed description"
       }
     ]
   }
   ```

5. **Создание High-Level Element узлов**
   ```python
   he = High_level_elements(description, title, config)
   G.add_node(he.hash_id, type='high_level_element', weight=1)
   G.add_node(he.title_hash_id, type='high_level_element_title', weight=1)
   G.add_edge(he.hash_id, he.title_hash_id, weight=1)
   ```

6. **Генерация embeddings для HE**
   - Batch обработка всех high-level elements
   - Использование embedding_client

7. **Связывание с исходными узлами**

   Два режима:

   a) **Если узлов много**: KMeans кластеризация
   - Объединение embeddings HE и semantic units
   - KMeans с k = ceil(sqrt(|nodes|))
   - Связывание HE только с узлами в том же кластере

   b) **Если узлов мало**: прямое связывание
   - Связывание HE со всеми related узлами

8. **Сохранение**
   - **high_level_elements.parquet**: HE nodes
   - **high_level_elements_titles.parquet**: title nodes
   - **embedding.parquet**: HE embeddings
   - **graph.pkl**: обновленный граф

### Выходные Данные

**high_level_elements.parquet**:
```
- type: 'high_level_element'
- title_hash_id: hash ID title node
- context: description текст
- hash_id: SHA256
- human_readable_id: H00001
- related_nodes: список связанных узлов
- embedding: 'done'
```

**high_level_elements_titles.parquet**:
```
- type: 'high_level_element_title'
- hash_id: SHA256
- context: title текст
- human_readable_id: H00001
```

---

## 8. Insert Text Pipeline

**Файл**: `NodeRAG/build/pipeline/Insert_text.py`

### Цель
Вставка text unit узлов в граф и связывание с semantic units.

### Выходные Данные
- Обновленный граф с text unit узлами

---

## 9. HNSW Pipeline

**Файл**: `NodeRAG/build/pipeline/HNSW_graph.py`

### Цель
Построение HNSW (Hierarchical Navigable Small World) индексов для быстрого векторного поиска.

### Выходные Данные
- HNSW индексы для эффективного retrieval

---

## Обработка Ошибок

Все pipeline'ы поддерживают:

1. **Error caching**
   - Сохранение неудачных LLM запросов в LLM_error.jsonl
   - Сохранение input_data и meta_data

2. **Retry механизм**
   - Exponential backoff для API ошибок
   - Max 4 попытки с таймаутами 2s, 4s, 8s, 16s

3. **Rerun функциональность**
   - Метод rerun() для повторной обработки ошибок
   - Очистка error cache после успешной обработки

4. **State management**
   - Сохранение текущего состояния pipeline
   - Возможность продолжения с точки прерывания

## Инкрементальность

Все pipeline'ы поддерживают инкрементальную обработку:

1. Проверка existing data
2. Фильтрация уже обработанных элементов
3. Обработка только новых данных
4. Append mode для всех storage операций
