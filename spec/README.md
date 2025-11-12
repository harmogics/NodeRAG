# NodeRAG: Документация Pipeline Обработки и Индексации

Эта директория содержит полную документацию системы обработки и индексации документов в NodeRAG, с детальным описанием всех этапов, работы с узлами графа, формирования концептуального слоя, и ролей языковых агентов.

## 📚 Содержание Документации

### [00-overview.md](./00-overview.md)
**Обзор системы NodeRAG**

Введение в архитектуру и основные концепции:
- Общая архитектура системы
- Типы узлов гетерогенного графа
- Многоуровневая структура представления знаний
- Роль языковых моделей
- Структура pipeline
- Форматы хранения данных

**Рекомендуется читать первым** для понимания общей картины.

---

### [01-pipeline-stages.md](./01-pipeline-stages.md)
**Детальное описание всех этапов pipeline**

Полная документация 9 этапов обработки:

1. **INIT Pipeline** - Проверка структуры и инкрементальности
2. **Document Pipeline** - Загрузка и семантическое chunking
3. **Text Pipeline** - Декомпозиция через LLM
4. **Graph Pipeline** - Построение гетерогенного графа
5. **Attribute Pipeline** - Генерация описаний важных узлов
6. **Embedding Pipeline** - Создание векторных представлений
7. **Summary Pipeline** - Community detection и концептуальный слой
8. **Insert Text Pipeline** - Вставка text units в граф
9. **HNSW Pipeline** - Построение индексов для поиска

Для каждого этапа описаны:
- Цель и задачи
- Входные и выходные данные
- Алгоритмы и процессы
- Форматы данных
- Обработка ошибок
- Инкрементальность

---

### [02-graph-nodes-and-chunking.md](./02-graph-nodes-and-chunking.md)
**Узлы графа и стратегия chunking**

Детальное описание компонентов системы:

#### Типы узлов:
1. **Document Nodes** - Исходные документы
2. **Text Unit Nodes** - Семантические chunks
3. **Semantic Unit Nodes** - Извлеченные факты
4. **Entity Nodes** - Именованные сущности
5. **Relationship Nodes** - Связи между сущностями
6. **Attribute Nodes** - Детальные описания
7. **High-Level Element Nodes** - Концептуальный слой
8. **Community Summary** - Промежуточные объекты

#### Семантический Chunking:
- Алгоритм `SemanticTextSplitter`
- Приоритет семантических границ
- Управление размером токенов
- Примеры разбиения

#### Взаимодействие узлов:
- Структура графа
- Свойства узлов
- Storage format
- Индексация

---

### [03-llm-agents-and-roles.md](./03-llm-agents-and-roles.md)
**Языковые агенты и их функции**

Полное описание 5 типов LLM агентов:

#### 1. Text Decomposition Agent
- Семантическая декомпозиция текста
- Извлечение entities и relationships
- Промпты и примеры
- Structured output schema

#### 2. Relationship Reconstruction Agent
- Исправление некорректных relationships
- Преобразование в формат triplet

#### 3. Attribute Generation Agent
- Создание narrative descriptions
- Сбор контекста из графа
- Token management
- Генерация "character sketch"

#### 4. Community Summary Agent
- Извлечение high-level concepts
- Обработка community clusters
- Diversity и non-redundancy

#### 5. Embedding Agent
- Генерация векторных представлений
- Batch processing
- Поддержка OpenAI и Gemini

#### Дополнительно:
- Архитектура LLM системы
- Error handling и retry logic
- Structured output
- Промпт менеджер
- Многоязычность

---

### [04-conceptual-layer-and-semantics.md](./04-conceptual-layer-and-semantics.md)
**Концептуальный слой и семантическая обработка**

Глубокое погружение в семантику:

#### Семантическая иерархия:
- От raw documents до концептов
- 5 уровней абстракции
- Трансформация знаний

#### Формирование концептуального слоя:
1. **Community Detection**
   - Leiden algorithm
   - Modularity optimization
   - Preprocessing и filtering

2. **Community Characterization**
   - Сбор контекста
   - Token management
   - LLM extraction

3. **Embedding Generation**
   - Векторизация концептов
   - Batch processing

4. **Linking Strategy**
   - KMeans для больших communities
   - Прямое связывание для малых

#### Семантическая декомпозиция:
- Event-based segmentation
- Paraphrasing принципы
- Entity extraction правила
- UPPERCASE normalization
- Temporal entity format
- Descriptive relationships

#### Формирование концептов:
- Abstraction механизмы
- Multi-faceted nature
- Significance-based отбор
- Merge стратегии

#### Retrieval Strategy:
- Многоуровневый поиск
- Graph expansion
- Precision ranking

#### Полный пример:
- От документа до концептов
- Все промежуточные этапы
- Retrieval flow

---

### [05-complete-processing-flow.md](./05-complete-processing-flow.md)
**Полный end-to-end flow обработки**

Визуальное и текстовое представление всего процесса:

#### ASCII диаграмма pipeline:
- Все 9 этапов
- Детальные блоки операций
- LLM агенты в контексте
- Data flow между этапами

#### Роли агентов - сводка:
- Таблица всех агентов
- Criticality ratings
- Frequency of use

#### Ключевые метрики:
- Размер данных для 100 документов
- Количество LLM вызовов
- Время обработки
- Параллелизация vs Sequential

#### Инкрементальная обработка:
- Сценарий добавления новых документов
- Оптимизация времени
- Дедупликация

#### Error handling flow:
- Диаграмма обработки ошибок
- Retry logic
- Error caching
- Rerun process

#### State management:
- State file структура
- Восстановление после прерывания

#### Storage layout:
- Файловая структура проекта
- Назначение каждого файла

---

## 🎯 Как Использовать Документацию

### Для новых пользователей:
1. Начните с [00-overview.md](./00-overview.md) для общего понимания
2. Прочитайте [05-complete-processing-flow.md](./05-complete-processing-flow.md) для визуализации
3. Углубитесь в конкретные темы по необходимости

### Для разработчиков:
1. [01-pipeline-stages.md](./01-pipeline-stages.md) - для понимания кода pipeline
2. [02-graph-nodes-and-chunking.md](./02-graph-nodes-and-chunking.md) - для работы с графом
3. [03-llm-agents-and-roles.md](./03-llm-agents-and-roles.md) - для модификации промптов

### Для исследователей:
1. [04-conceptual-layer-and-semantics.md](./04-conceptual-layer-and-semantics.md) - для понимания семантики
2. [03-llm-agents-and-roles.md](./03-llm-agents-and-roles.md) - для анализа использования LLM
3. [02-graph-nodes-and-chunking.md](./02-graph-nodes-and-chunking.md) - для graph algorithms

---

## 🔑 Ключевые Концепции

### Гетерогенный Граф
NodeRAG использует граф с 8 типами узлов, каждый со специфической ролью в представлении знаний.

### Семантическое Chunking
Разбиение текста не по фиксированному размеру, а по семантическим границам (параграфы, предложения).

### LLM как Агенты
Языковые модели выполняют специализированные функции: decomposition, extraction, summarization, embedding.

### Концептуальный Слой
Высокоуровневые темы и концепты, автоматически извлеченные через community detection + LLM.

### Многоуровневый Retrieval
Поиск сначала на концептуальном уровне, затем expansion через граф, финальный ranking на детальном уровне.

### Инкрементальность
Поддержка добавления новых документов без полной переиндексации, с дедупликацией через hash_id.

---

## 📊 Архитектурная Диаграмма (Упрощенная)

```
                    ┌─────────────────┐
                    │   Documents     │
                    └────────┬────────┘
                             ↓
                    ┌─────────────────┐
                    │  Text Units     │
                    │  (Chunking)     │
                    └────────┬────────┘
                             ↓
         ┌───────────────────┴───────────────────┐
         ↓                                       ↓
┌─────────────────┐                   ┌─────────────────┐
│ Semantic Units  │                   │    Entities     │
│   + Relations   │←─────────────────→│  + Relations    │
└────────┬────────┘                   └────────┬────────┘
         │                                     │
         ↓                                     ↓
┌─────────────────┐                   ┌─────────────────┐
│   Embeddings    │                   │   Attributes    │
│  (Vectorize)    │                   │  (Important)    │
└────────┬────────┘                   └────────┬────────┘
         │                                     │
         └──────────────┬──────────────────────┘
                        ↓
              ┌─────────────────┐
              │   Communities   │
              │  (Leiden Algo)  │
              └────────┬────────┘
                       ↓
              ┌─────────────────┐
              │  High-Level     │
              │   Elements      │
              │  (Concepts)     │
              └────────┬────────┘
                       ↓
              ┌─────────────────┐
              │  Knowledge Graph│
              │   + HNSW Index  │
              └─────────────────┘
```

---

## 🛠 Технические Детали

### Языки и Библиотеки
- **Python 3.10+**
- **NetworkX** - Граф манипуляции
- **igraph** + **leidenalg** - Community detection
- **Faiss** - KMeans clustering и HNSW
- **Pandas** - Data manipulation
- **PyArrow** - Parquet I/O
- **OpenAI / Gemini** - LLM APIs
- **Asyncio** - Асинхронная обработка

### Форматы Данных
- **Parquet** - Структурированные данные узлов
- **Pickle** - NetworkX граф
- **JSONL** - LLM responses и промежуточные данные
- **JSON** - Конфигурация и метаданные

### Алгоритмы
- **SemanticTextSplitter** - Chunking с semantic boundaries
- **Leiden Algorithm** - Community detection
- **K-core decomposition** - Important nodes
- **Betweenness centrality** - Important nodes
- **KMeans** - Clustering для linking
- **HNSW** - Approximate nearest neighbor search

---

## 📈 Производительность

### Для 100 документов (~10MB):
- **Узлов в графе**: ~14,500
- **Ребер**: ~25,000
- **LLM вызовов**: ~2,320
- **Embeddings**: ~10,300
- **Время обработки**: ~40 минут (параллельно)
- **Без параллелизации**: ~8 часов

### Инкрементальное добавление 10 документов:
- **Время**: ~8 минут
- **Speedup**: 5x по сравнению с full reindexing

---

## 🔄 Workflow для Разработки

### Модификация Pipeline:
1. Изучите [01-pipeline-stages.md](./01-pipeline-stages.md)
2. Найдите соответствующий файл в `NodeRAG/build/pipeline/`
3. Модифицируйте код
4. Обновите документацию
5. Протестируйте с small dataset

### Добавление Нового Типа Узла:
1. Создайте класс в `NodeRAG/build/component/`
2. Наследуйте от `Unit_base`
3. Добавьте в graph_pipeline.py
4. Обновите storage logic
5. Документируйте в [02-graph-nodes-and-chunking.md](./02-graph-nodes-and-chunking.md)

### Модификация Промптов:
1. Изучите [03-llm-agents-and-roles.md](./03-llm-agents-and-roles.md)
2. Найдите промпт в `NodeRAG/utils/prompt/`
3. Модифицируйте текст и/или schema
4. Тестируйте output quality
5. Документируйте изменения

---

## 🔗 Связанные Ресурсы

### Официальная документация:
- [NodeRAG Website](https://terry-xu-666.github.io/NodeRAG_web/)
- [PyPI Package](https://pypi.org/project/NodeRAG/)
- [GitHub Repository](https://github.com/Terry-Xu-666/NodeRAG)

### Научная статья:
- [arXiv Paper](https://arxiv.org/abs/arkiv) (when available)

### Примеры использования:
- [Indexing Tutorial](https://terry-xu-666.github.io/NodeRAG_web/docs/indexing/)
- [Answering Tutorial](https://terry-xu-666.github.io/NodeRAG_web/docs/answer/)

---

## 📝 Обновления Документации

**Версия**: 1.0.0
**Дата**: 2025-11-12
**Основано на**: NodeRAG v0.1.0

### История изменений:
- **2025-11-12**: Первая версия полной документации pipeline

---

## 👥 Контрибьюторы

Документация создана на основе анализа кодовой базы NodeRAG v0.1.0.

### Для вопросов и предложений:
- Откройте issue на [GitHub](https://github.com/Terry-Xu-666/NodeRAG/issues)
- Посетите [официальный сайт](https://terry-xu-666.github.io/NodeRAG_web/)

---

## 📄 Лицензия

Документация следует лицензии проекта NodeRAG: MIT License

---

**Приятного изучения NodeRAG!** 🚀
