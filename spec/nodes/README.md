# NodeRAG: Структурная Карта Узлов Графа

## Обзор

Эта директория содержит детальное описание всех типов узлов в гетерогенном графе NodeRAG, их взаимосвязей и влияния на процессы поиска и генерации ответов.

## 🗺️ Структурная Карта

### Типы Узлов (Node Types)

1. **[Document Nodes](./01-document-nodes.md)** 📄
   - Корневые узлы системы
   - Представляют исходные документы
   - Метаданные и связь с файловой системой

2. **[Text Unit Nodes](./02-text-unit-nodes.md)** 📝
   - Семантические chunks текста
   - Результат semantic chunking
   - Базовые единицы для LLM обработки

3. **[Semantic Unit Nodes](./03-semantic-unit-nodes.md)** 💡
   - Атомарные факты и события
   - Извлеченные через LLM decomposition
   - Суммаризированная семантика

4. **[Entity Nodes](./04-entity-nodes.md)** 🏷️
   - Именованные сущности
   - Ключевые концепты и объекты
   - Узлы-коннекторы графа

5. **[Relationship Nodes](./05-relationship-nodes.md)** 🔗
   - Связи между сущностями
   - Описательные отношения
   - Триплеты знаний

6. **[Attribute Nodes](./06-attribute-nodes.md)** 📋
   - Детальные описания важных сущностей
   - Агрегированный контекст
   - Narrative descriptions

7. **[High-Level Element Nodes](./07-high-level-element-nodes.md)** 🎯
   - Концептуальный слой
   - Темы и идеи
   - Результат community detection

### Взаимосвязи и Процессы

8. **[Node Relationships](./08-node-relationships.md)** 🕸️
   - Типы ребер графа
   - Паттерны связей
   - Граф топология

9. **[Search and Retrieval](./09-search-and-retrieval.md)** 🔍
   - Стратегии поиска
   - Использование разных типов узлов
   - Многоуровневый retrieval

10. **[Answer Generation](./10-answer-generation.md)** 💬
    - Формулировка ответов
    - Роль каждого типа узлов
    - Context assembly

---

## 🎯 Архитектура: Общая Схема

```
┌────────────────────────────────────────────────────────────┐
│                    KNOWLEDGE GRAPH                          │
└────────────────────────────────────────────────────────────┘

                        [Document]
                             │
                    ┌────────┴────────┐
                    │                 │
              [Text Unit]      [Text Unit]
                    │                 │
        ┌───────────┼─────────┐      │
        │           │         │      │
   [Semantic]  [Semantic] [Semantic] ...
        │           │         │
        ├───────────┼─────────┤
        │           │         │
    [Entity]   [Entity]  [Entity]
        │           │         │
        │      [Relationship] │
        │           │         │
        └───────────┼─────────┘
                    │
              [Attribute]
                    │
        ┌───────────┴───────────┐
        │                       │
   [Community]            [Community]
        │                       │
[High-Level Element]  [High-Level Element]
```

---

## 📊 Иерархия Узлов

### По Уровню Абстракции

```
Level 0: Raw Data
└─ Document Nodes

Level 1: Chunks
└─ Text Unit Nodes

Level 2: Extracted Facts
├─ Semantic Unit Nodes
├─ Entity Nodes
└─ Relationship Nodes

Level 3: Enrichment
└─ Attribute Nodes

Level 4: Concepts
└─ High-Level Element Nodes
```

### По Роли в Системе

**Structural Nodes** (структурные):
- Document Nodes
- Text Unit Nodes

**Content Nodes** (контентные):
- Semantic Unit Nodes
- Entity Nodes
- Relationship Nodes

**Enrichment Nodes** (обогащающие):
- Attribute Nodes

**Conceptual Nodes** (концептуальные):
- High-Level Element Nodes

---

## 🔑 Ключевые Характеристики

### Embeddings

| Node Type              | Has Embedding | Searchable |
|------------------------|---------------|------------|
| Document               | ❌            | ❌         |
| Text Unit              | ✅            | ✅         |
| Semantic Unit          | ✅            | ✅         |
| Entity                 | ❌            | ❌         |
| Relationship           | ❌            | ❌         |
| Attribute              | ✅            | ✅         |
| High-Level Element     | ✅            | ✅         |

### Weight (Частота)

| Node Type              | Has Weight | Meaning                    |
|------------------------|------------|----------------------------|
| Document               | ❌         | -                          |
| Text Unit              | ❌         | -                          |
| Semantic Unit          | ✅         | Частота появления          |
| Entity                 | ✅         | Количество упоминаний      |
| Relationship           | ✅         | Частота связи              |
| Attribute              | (inherited)| От родительской entity     |
| High-Level Element     | ✅         | Всегда 1 при создании      |

### Storage

| Node Type              | Parquet File                         |
|------------------------|--------------------------------------|
| Document               | documents.parquet                    |
| Text Unit              | text.parquet                         |
| Semantic Unit          | semantic_units.parquet               |
| Entity                 | entities.parquet                     |
| Relationship           | relationship.parquet                 |
| Attribute              | attributes.parquet                   |
| High-Level Element     | high_level_elements.parquet          |
| HE Title               | high_level_elements_titles.parquet   |

---

## 🔍 Использование в Retrieval

### Точечный Поиск (Precise)
**Используются**: Semantic Units, Text Units
- Конкретные факты
- Точные цитаты
- Детальная информация

### Контекстный Поиск (Contextual)
**Используются**: Entities, Relationships, Attributes
- Связанная информация
- Контекстные детали
- Enriched descriptions

### Концептуальный Поиск (Conceptual)
**Используются**: High-Level Elements
- Темы и идеи
- Абстрактные концепты
- Широкий контекст

---

## 💡 Использование в Answer Generation

### Базовые Факты
**Источник**: Semantic Units
```
"Dr. Emily Roberts attended a conference in Paris."
```

### Детальный Контекст
**Источник**: Attributes
```
"Dr. Emily Roberts is a leading researcher in renewable energy,
specializing in solar panel efficiency..."
```

### Концептуальный Фон
**Источник**: High-Level Elements
```
"This relates to the broader theme of Environmental Conservation
Through Scientific Research..."
```

### Связи и Отношения
**Источник**: Relationships
```
"Dr. Emily Roberts works at European Research Institute and
collaborates with European companies."
```

---

## 🎨 Визуализация: Node Properties

### Document Node
```
┌─────────────────────────┐
│  Document               │
├─────────────────────────┤
│ ✓ raw_context          │
│ ✓ path                 │
│ ✓ hash_id              │
│ ✓ human_readable_id    │
│ ✗ embedding            │
│ ✗ weight               │
└─────────────────────────┘
```

### Text Unit Node
```
┌─────────────────────────┐
│  Text Unit              │
├─────────────────────────┤
│ ✓ raw_context          │
│ ✓ hash_id              │
│ ✓ human_readable_id    │
│ ✓ doc_hash_id          │
│ ✓ embedding            │
│ ✗ weight               │
└─────────────────────────┘
```

### Semantic Unit Node
```
┌─────────────────────────┐
│  Semantic Unit          │
├─────────────────────────┤
│ ✓ raw_context          │
│ ✓ hash_id              │
│ ✓ human_readable_id    │
│ ✓ text_hash_id         │
│ ✓ embedding            │
│ ✓ weight               │
└─────────────────────────┘
```

### Entity Node
```
┌─────────────────────────┐
│  Entity                 │
├─────────────────────────┤
│ ✓ raw_context (UPPER)  │
│ ✓ hash_id              │
│ ✓ human_readable_id    │
│ ✓ text_hash_id         │
│ ✗ embedding            │
│ ✓ weight               │
│ ? attributes (optional)│
└─────────────────────────┘
```

### Relationship Node
```
┌─────────────────────────┐
│  Relationship           │
├─────────────────────────┤
│ ✓ raw_context          │
│ ✓ unique_relationship  │
│ ✓ hash_id              │
│ ✓ human_readable_id    │
│ ✓ text_hash_id         │
│ ✗ embedding            │
│ ✓ weight               │
│ ✓ source (Entity)      │
│ ✓ target (Entity)      │
└─────────────────────────┘
```

### Attribute Node
```
┌─────────────────────────┐
│  Attribute              │
├─────────────────────────┤
│ ✓ raw_context          │
│ ✓ hash_id              │
│ ✓ human_readable_id    │
│ ✓ node (parent Entity) │
│ ✓ embedding            │
│ ✓ weight (inherited)   │
└─────────────────────────┘
```

### High-Level Element Node
```
┌─────────────────────────┐
│  High-Level Element     │
├─────────────────────────┤
│ ✓ context (description)│
│ ✓ title                │
│ ✓ hash_id              │
│ ✓ title_hash_id        │
│ ✓ human_readable_id    │
│ ✓ embedding            │
│ ✓ weight               │
│ ✓ related_nodes        │
└─────────────────────────┘
```

---

## 🚀 Quick Reference

### Создание Узлов

**Document**: При загрузке файлов (Document Pipeline)
**Text Unit**: При semantic chunking (Document Pipeline)
**Semantic Unit**: Через LLM decomposition (Text Pipeline)
**Entity**: Через LLM extraction (Text Pipeline)
**Relationship**: Через LLM extraction (Text Pipeline)
**Attribute**: Для важных entities (Attribute Pipeline)
**High-Level Element**: Через community detection + LLM (Summary Pipeline)

### Поиск по Типам

**Для фактов**: Semantic Units → Text Units
**Для сущностей**: Entities → Attributes → Semantic Units
**Для концептов**: High-Level Elements → related nodes
**Для связей**: Relationships → connected Entities

### Граф Навигация

```python
# От документа к фактам
Document → Text Units → Semantic Units

# От сущности к фактам
Entity → Semantic Units (упоминания)

# От сущности к описанию
Entity → Attribute (если есть)

# От сущности к связям
Entity → Relationship Nodes → другие Entities

# От концепта к фактам
High-Level Element → Semantic Units в community
```

---

## 📖 Рекомендуемый Порядок Чтения

### Для понимания структуры:
1. Начните с Document и Text Unit nodes
2. Изучите Semantic Units как базовые факты
3. Понять Entities и Relationships
4. Углубиться в Attributes для enrichment
5. Освоить High-Level Elements для концептов

### Для работы с поиском:
1. [Search and Retrieval](./09-search-and-retrieval.md)
2. [Node Relationships](./08-node-relationships.md)
3. Specific node types по необходимости

### Для генерации ответов:
1. [Answer Generation](./10-answer-generation.md)
2. [High-Level Element Nodes](./07-high-level-element-nodes.md)
3. [Attribute Nodes](./06-attribute-nodes.md)

---

## 🔗 Связанные Ресурсы

- [Основная документация](../README.md)
- [Pipeline Stages](../01-pipeline-stages.md)
- [LLM Agents](../03-llm-agents-and-roles.md)
- [Conceptual Layer](../04-conceptual-layer-and-semantics.md)

---

**Версия**: 1.0.0
**Дата**: 2025-11-12
