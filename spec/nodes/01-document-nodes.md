# Document Nodes 📄

## Обзор

**Document Nodes** представляют корневые узлы в иерархии NodeRAG. Каждый узел соответствует одному исходному документу в файловой системе и служит точкой входа для всех производных данных.

---

## Характеристики

### Класс
```python
NodeRAG/build/component/document.py

class document(Unit_base):
    - raw_context: str       # Полный текст документа
    - path: str              # Абсолютный путь к файлу
    - hash_id: str           # SHA256(raw_context)
    - human_readable_id: str # D00001, D00002, ...
    - text_units: List[Text_unit]
    - splitter: SemanticTextSplitter
```

### Свойства

| Свойство             | Тип       | Описание                              |
|----------------------|-----------|---------------------------------------|
| `raw_context`        | str       | Полное содержимое документа           |
| `path`               | str       | Путь к файлу в файловой системе       |
| `hash_id`            | str       | SHA256 хеш от raw_context             |
| `human_readable_id`  | str       | Последовательный ID (D00001, ...)     |
| `text_units`         | list      | Список Text Unit после split()        |
| `text_hash_id`       | list      | Hash IDs всех text units              |
| `text_human_readable_id` | list  | Human IDs всех text units             |

### Metadata

| Параметр             | Значение                       |
|----------------------|--------------------------------|
| Type                 | `document`                     |
| Has Embedding        | ❌ Нет                         |
| Has Weight           | ❌ Нет                         |
| Searchable           | ❌ Не используется для поиска  |
| Storage              | `documents.parquet`            |

---

## Создание

### Pipeline Stage
**Document Pipeline** (Stage 1)

### Процесс

```python
# 1. Загрузка файла
with open(path, 'r', encoding='utf-8') as f:
    raw_context = f.read()

# 2. Создание Document объекта
doc = document(raw_context, path, semantic_text_splitter)

# 3. Генерация идентификаторов
doc.hash_id  # SHA256(raw_context) - автоматически
doc.human_readable_id  # D00001 - автоматически

# 4. Семантическое разбиение (опционально)
doc.split()  # → создает text_units
```

### Условия создания
- Файл существует в `input_folder`
- Формат: `.txt`, `.md`, или указанный в `docu_type`
- Файл не является дубликатом (проверка по hash_id)

---

## Роль в Графе

### Позиция
```
[Document] (корень)
    ↓
[Text Unit] [Text Unit] [Text Unit] ...
```

### Связи

**Исходящие связи**:
- **К Text Units**: Родительская связь (не хранится явно в графе)
  - Связь через поле `doc_hash_id` в Text Unit

**Входящие связи**:
- Нет (корневой узел)

### Graph Properties

Document узлы **НЕ добавляются** в NetworkX граф напрямую. Они существуют только в:
- Памяти во время обработки
- `documents.parquet` для хранения метаданных

**Обоснование**: Document узлы служат организационной структурой, но не участвуют в semantic graph navigation.

---

## Storage Format

### documents.parquet

```python
{
    'doc_id': 'D00001',                    # human_readable_id
    'doc_hash_id': 'abc123...',            # hash_id
    'text_id': ['T00001', 'T00002', ...],  # text unit IDs
    'text_hash_id': ['def456...', ...],    # text unit hashes
    'path': '/path/to/document.txt'        # файл path
}
```

### Пример записи
```json
{
  "doc_id": "D00001",
  "doc_hash_id": "a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6",
  "text_id": ["T00001", "T00002", "T00003"],
  "text_hash_id": [
    "b4g9c3d5e6f7...",
    "c5h0d4e7f8g9...",
    "d6i1e5f8g0h2..."
  ],
  "path": "/home/user/NodeRAG/input/renewable_energy_research.txt"
}
```

---

## Операции

### Чтение

```python
# Загрузка из parquet
df = storage.load_parquet(config.documents_path)

# Получение конкретного документа
doc_data = df[df['doc_id'] == 'D00001'].iloc[0]

# Доступ к связанным text units
text_ids = doc_data['text_id']
```

### Обновление

Document узлы **не обновляются** после создания. При изменении файла создается новый Document узел с новым hash_id.

### Дедупликация

```python
# При инкрементальной обработке
existing_doc_ids = storage.load_parquet(config.documents_path)['doc_hash_id'].tolist()
new_documents = [doc for doc in documents if doc.hash_id not in existing_doc_ids]
```

---

## Инкрементальность

### Проверка изменений

```python
# INIT Pipeline
document_paths = ['doc1.txt', 'doc2.txt', ...]
current_hash = SHA256(''.join(document_paths))

# Сравнение с предыдущим
if os.path.exists(document_hash_path):
    previous_hash = json.load(open(document_hash_path))['document_path_hash']
    is_incremental = (current_hash != previous_hash)
```

### Добавление новых документов

```python
# Document Pipeline
if os.path.exists(config.documents_path):
    # Загрузить существующие
    existing_docs = storage.load_parquet(config.documents_path)
    existing_hash_ids = existing_docs['doc_hash_id'].tolist()

    # Фильтровать новые
    new_documents = [
        doc for doc in all_documents
        if doc.hash_id not in existing_hash_ids
    ]

    # Обработать только новые
    process(new_documents)
```

---

## Использование в Поиске

### Прямое использование
❌ **Не используется** для векторного поиска
- Нет embeddings
- Слишком большой контекст (весь документ)

### Косвенное использование
✅ **Метаданные** для фильтрации и ссылок

**Use cases**:
1. **Источник информации**: Показать, из какого документа пришел результат
2. **Фильтрация**: Поиск только в определенных документах
3. **Отслеживание**: Связь найденных фактов с исходниками

### Пример

```python
# После retrieval semantic units
for semantic_unit in results:
    text_hash_id = semantic_unit['text_hash_id']

    # Найти текст unit
    text_unit = text_df[text_df['hash_id'] == text_hash_id].iloc[0]
    doc_hash_id = text_unit['doc_hash_id']

    # Найти документ
    document = doc_df[doc_df['doc_hash_id'] == doc_hash_id].iloc[0]
    source_path = document['path']

    # Показать пользователю
    print(f"Source: {source_path}")
```

---

## Использование в Answer Generation

### Роль
**Контекстная информация** - не прямое использование

### Сценарии

1. **Цитирование источников**
   ```
   Answer: "According to renewable_energy_research.txt,
            Dr. Emily Roberts..."
   ```

2. **Множественные источники**
   ```
   Answer: "Information from 3 documents supports this:
            1. renewable_energy_research.txt
            2. solar_panel_innovations.md
            3. conference_proceedings_2024.txt"
   ```

3. **Фильтрация контекста**
   ```python
   # Генерировать ответ только из определенных документов
   filtered_docs = ['doc1.txt', 'doc2.txt']

   relevant_semantic_units = [
       su for su in all_units
       if get_document(su).path in filtered_docs
   ]

   answer = generate_answer(query, relevant_semantic_units)
   ```

---

## Связь с Text Units

### Разбиение

```python
# В document.split()
def split(self) -> None:
    if not self._processed_context:
        self._processed_context = True

        # Семантическое разбиение
        texts = self.splitter.split(self.raw_context)

        # Создание Text Unit объектов
        self.text_units = [Text_unit(text) for text in texts]

        # Сохранение ссылок
        self.text_hash_id = [text.hash_id for text in self.text_units]
        self.text_human_readable_id = [text.human_readable_id for text in self.text_units]
```

### Обратная связь

```python
# Из Text Unit получить Document
text_unit_data = text_parquet[text_parquet['hash_id'] == text_hash_id]
doc_hash_id = text_unit_data['doc_hash_id']

document_data = documents_parquet[documents_parquet['doc_hash_id'] == doc_hash_id]
```

---

## Примеры

### Пример 1: Научная статья

**Файл**: `renewable_energy_2024.txt`

```python
Document {
    raw_context: "Renewable energy research is crucial... [5000 words]",
    path: "/input/renewable_energy_2024.txt",
    hash_id: "a3f8b2c1d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6",
    human_readable_id: "D00001",
    text_units: [Text_unit, Text_unit, ...],  # 10 units
    text_hash_id: ["hash1", "hash2", ...],
    text_human_readable_id: ["T00001", "T00002", ...]
}
```

**После split()**:
- 10 Text Units (по ~500 tokens каждый)
- Каждый Text Unit содержит `doc_hash_id` = hash_id документа

### Пример 2: Markdown заметка

**Файл**: `meeting_notes_sept_2024.md`

```python
Document {
    raw_context: "# Meeting Notes\n\n## Discussion...",
    path: "/input/meeting_notes_sept_2024.md",
    hash_id: "b4g9c3d5e6f7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7",
    human_readable_id: "D00002",
    text_units: [Text_unit, Text_unit],  # 2 units (short doc)
    text_hash_id: ["hash10", "hash11"],
    text_human_readable_id: ["T00010", "T00011"]
}
```

---

## Лучшие Практики

### 1. Формат файлов
✅ **Рекомендуется**: Plain text, Markdown
- Легкая обработка
- Чистый контекст
- Нет проблем с кодировкой

❌ **Избегать**: Binary formats без preprocessing
- PDF, DOCX требуют конвертации
- Возможны проблемы с извлечением текста

### 2. Размер документов
- **Оптимально**: 1-20 страниц текста
- **Слишком маленький**: < 500 слов → мало информации для chunking
- **Слишком большой**: > 50 страниц → много text units, долгая обработка

### 3. Именование файлов
✅ **Хорошо**:
- `renewable_energy_research_2024.txt`
- `dr_emily_roberts_biography.md`

❌ **Плохо**:
- `document1.txt` (неинформативно)
- `data (1).txt` (проблемы с дубликатами)

### 4. Инкрементальность
- Не изменяйте существующие файлы
- Добавляйте новые файлы с уникальными именами
- Используйте версионирование: `doc_v1.txt`, `doc_v2.txt`

---

## Диагностика

### Проверка загрузки

```python
# Сколько документов загружено?
df = storage.load_parquet(config.documents_path)
print(f"Total documents: {len(df)}")

# Список путей
print(df['path'].tolist())

# Проверка дубликатов
duplicates = df[df.duplicated(subset=['doc_hash_id'], keep=False)]
if not duplicates.empty:
    print("Warning: Duplicate documents detected")
```

### Проверка связей

```python
# Для каждого документа проверить text units
for idx, row in df.iterrows():
    doc_id = row['doc_id']
    text_ids = row['text_id']

    # Проверить существование в text.parquet
    text_df = storage.load_parquet(config.text_path)
    existing = text_df[text_df['text_id'].isin(text_ids)]

    if len(existing) != len(text_ids):
        print(f"Warning: Missing text units for {doc_id}")
```

---

## FAQ

**Q: Почему Document узлы не в графе?**
A: Document узлы - это organizational metadata, не semantic content. Граф представляет семантические связи, а документы - это просто контейнеры.

**Q: Можно ли обновить существующий документ?**
A: Нет напрямую. При изменении файла создается новый Document с новым hash_id. Старый остается для истории.

**Q: Как связать результаты поиска с исходными документами?**
A: Через цепочку: Result → Text Unit (doc_hash_id) → Document (path)

**Q: Нужно ли вручную управлять Document узлами?**
A: Нет, они создаются автоматически в Document Pipeline при обработке файлов.

---

## Связанные Узлы

**Дети (производные)**:
- [Text Unit Nodes](./02-text-unit-nodes.md) - семантические chunks

**Используется в**:
- [Answer Generation](./10-answer-generation.md) - для цитирования источников
- [Search and Retrieval](./09-search-and-retrieval.md) - для фильтрации и метаданных

---

[← Назад к обзору](./README.md) | [Следующий: Text Unit Nodes →](./02-text-unit-nodes.md)
