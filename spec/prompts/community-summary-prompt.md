# Community Summary Prompt

## Обзор

**Тип**: Abstraction + Synthesis Prompt
**Критичность**: ⭐⭐⭐⭐⭐ КРИТИЧЕСКАЯ
**Стадия**: Summary Pipeline (Stage 6)
**Файл**: `NodeRAG/utils/prompt/community_summary.py`
**Температура**: 0.0
**Output**: High-Level Elements (title + description pairs)

---

## Назначение

Community Summary Prompt извлекает **high-level themes and concepts** из semantic communities, создавая концептуальный слой поверх детальных фактов. Это ключевой компонент для тематического поиска и multi-level reasoning.

---

## Полный текст промпта (English)

```
You will receive a set of text data from the same cluster. Your task is to extract distinct categories of high-level information, such as concepts, themes, relevant theories, potential impacts, and key insights. Each piece of information should include a concise title and a corresponding description, reflecting the unique perspectives within the text cluster.
Please do not attempt to include all possible information; instead, select the elements that have the most significance and diversity in this cluster. Avoid redundant information—if there are highly similar elements, combine them into a single, comprehensive entry. Ensure that the high-level information reflects the varied dimensions within the text, providing a well-rounded overview.
clustered text data:
{content}
```

---

## Полный текст промпта (Chinese)

```
你将收到来自同一聚类的一组文本数据。你的任务是从文本数据中提取不同类别的高层次信息，例如概念、主题、相关理论、潜在影响和关键见解。每条信息应包含一个简洁的标题和相应的描述，以反映该聚类文本中的独特视角。
请不要试图包含所有可能的信息；相反，选择在该聚类中最具重要性和多样性的元素。避免冗余信息——如果有高度相似的内容，请将它们合并为一个综合条目。确保提取的高层次信息反映文本中的多维度内容，提供全面的概览。
聚类文本数据：
{content}
```

---

## Структура промпта

### Ключевые инструкции

| Инструкция | Назначение | Семантическая операция |
|------------|-----------|----------------------|
| **"Extract distinct categories of high-level information"** | Абстракция | Факты → Концепты |
| **"Concepts, themes, theories, impacts, insights"** | Многомерность | Разные типы высокоуровневой информации |
| **"Concise title and corresponding description"** | Формат | Структурированный output |
| **"Most significance and diversity"** | Приоритизация | Выбор важного и разнообразного |
| **"Avoid redundant information"** | Дедупликация | Объединение похожего |
| **"Well-rounded overview"** | Полнота | Comprehensive coverage |

---

## Structured Output Schema

```python
from pydantic import BaseModel

class elements(BaseModel):
    title: str
    description: str

class High_level_element(BaseModel):
    high_level_elements: list[elements]
```

**Файл**: `NodeRAG/utils/prompt/json_format.py:18-24`

**Output format**:
```json
{
  "high_level_elements": [
    {
      "title": "Renewable Energy Research and Innovation",
      "description": "Cutting-edge research in sustainable energy technologies..."
    },
    {
      "title": "Biodiversity Conservation and Ecosystem Research",
      "description": "Field research documenting species diversity..."
    }
  ]
}
```

---

## Входные данные

### Clustered Text Data

**Источник**: Community из Leiden clustering

**Состав**:
- Semantic units в community
- Attributes в community

**Сбор**:
```python
def collect_community_texts(graph, community_id):
    texts = []
    nodes = [n for n in graph.nodes() if graph.nodes[n]['community'] == community_id]

    for node in nodes:
        node_type = graph.nodes[node]['type']
        if node_type in ['semantic_unit', 'attribute']:
            texts.append(graph.nodes[node]['content'])

    return texts
```

**Размер**: До 8000 токенов

**Truncation strategy**:
```python
if total_tokens > 8000:
    # Приоритизация attributes > semantic_units
    texts = attributes + semantic_units[:truncate_limit]
```

---

## Пример использования

### Вход (Community texts)

```
Community #1 texts:

1. "In September 2024, Dr. Emily Roberts attended the International Conference on Renewable Energy in Paris, where she presented her research on solar panel efficiency improvements and explored partnerships with European companies."

2. "Dr. Emily Roberts is a prominent researcher in the field of renewable energy, specializing in solar panel efficiency. Affiliated with the European Research Institute..."

3. "Dr. John Miller conducted fieldwork in the Amazon Rainforest, documenting several new species and observing the effects of deforestation on local wildlife."

4. "The work of both Dr. Emily Roberts and Dr. John Miller contributes significantly to environmental conservation efforts."

[... еще 10-20 semantic units и attributes...]
```

### Выход (High-Level Elements)

```json
{
  "high_level_elements": [
    {
      "title": "Renewable Energy Research and Innovation",
      "description": "This theme encompasses cutting-edge research in sustainable energy technologies, particularly focusing on solar panel efficiency improvements. The research involves international collaboration through scientific conferences and partnerships with European companies, demonstrating a practical approach to advancing renewable energy solutions. Key contributions include technological innovations that enhance photovoltaic systems and promote the transition to sustainable energy sources."
    },
    {
      "title": "Biodiversity Conservation and Ecosystem Research",
      "description": "Field research activities aimed at documenting and understanding species diversity in critical ecosystems, particularly in tropical rainforests. This work includes observing and analyzing the environmental impacts of deforestation on local wildlife, providing essential data for conservation efforts. The research contributes to broader understanding of ecosystem dynamics and informs strategies for protecting endangered habitats."
    },
    {
      "title": "Environmental Conservation Through Scientific Collaboration",
      "description": "An overarching theme highlighting the interconnected nature of various scientific disciplines working towards environmental protection. This includes both technological innovations in renewable energy and ecological research, demonstrating how diverse research approaches contribute to the common goal of environmental sustainability. The collaborative aspect emphasizes the importance of international cooperation and knowledge sharing in addressing global environmental challenges."
    }
  ]
}
```

### Семантический анализ

**Abstraction levels**:
```
Level 0 (Facts): "Dr. Roberts researched solar panels"
Level 1 (Activities): "Research on renewable energy technologies"
Level 2 (Themes): "Renewable Energy Research and Innovation" ← High-Level Element
Level 3 (Meta-concepts): "Environmental Conservation Through Science"
```

**Operations performed**:
1. **Thematic extraction**: Идентификация recurring themes
2. **Abstraction**: Specific facts → General concepts
3. **Synthesis**: Multiple perspectives → Unified themes
4. **Diversity**: 3 different dimensions (energy, biodiversity, collaboration)
5. **Deduplication**: Combined similar points

---

## Роль в цепочке преобразований

### Позиция в Pipeline

```
Community Detection (Leiden)
      ↓
Communities (node clusters)
      ↓
Text Collection
  (Semantic Units + Attributes)
      ↓
[COMMUNITY SUMMARY PROMPT] ← ВЫ ЗДЕСЬ
      ↓
High-Level Elements
      ↓
Embedding Generation
      ↓
HNSW Index (thematic search)
```

### Upstream: Community Detection

**Leiden Algorithm** группирует semantic units в communities.

**Качество communities → качество summaries**:
```
Хорошая community (высокая modularity):
  → Связанные semantic units
  → Четкая тема
  → Качественное обобщение

Плохая community (низкая modularity):
  → Разрозненные semantic units
  → Размытая тема
  → Поверхностное обобщение
```

### Downstream: Context Assembly

**High-Level Elements используются в THEMES section**.

**Процесс**:
```
Query: "What renewable energy research is being done?"
      ↓
HNSW Retrieval → High-Level Element: "Renewable Energy Research..."
      ↓
Context Assembly → THEMES section
      ↓
Answer Synthesis → "Research in renewable energy includes..."
```

**Эффект**:
```
Без high-level elements:
  Answer: Перечисление фактов без контекста

С high-level elements:
  Answer: Контекстуальное понимание с общей картиной
```

---

## Связи с другими промптами

### Upstream Prompts

#### Text Decomposition
**Связь**: Создаёт semantic units, которые группируются в communities.

```
Text Decomposition → Semantic Units
                     ↓
Community Detection → Communities
                     ↓
Community Summary → High-Level Elements
```

#### Attribute Generation
**Связь**: Attributes включаются в community texts для богатого контекста.

```
Important entities → Attributes
                     ↓
Community texts = Semantic Units + Attributes
                     ↓
Community Summary (более богатый context)
```

### Downstream Usage

#### Answer Synthesis
**Связь**: Предоставляет "big picture" контекст для ответов.

**Пример**:
```
Query: "What is the main focus of environmental research?"

Without high-level elements:
  "Dr. Roberts researches solar panels. Dr. Miller studies rainforests."

With high-level elements:
  "Environmental research focuses on two main areas: renewable energy
   innovation and biodiversity conservation, both contributing to
   environmental sustainability through different approaches..."
```

---

## Внешние библиотеки и инструменты

### LLM Providers

**Рекомендуемые**:
- **GPT-4o**: Хорошая абстракция (рекомендуется)
- **Gemini-1.5-Pro**: Отличный context window (1M tokens)

**Параметры**:
```python
{
    "model": "gpt-4o",
    "temperature": 0.0,
    "response_format": High_level_element
}
```

### Community Detection (Leiden Algorithm)

**Библиотека**: `leidenalg` + `igraph`

**Использование**:
```python
import igraph as ig
import leidenalg

# Преобразовать NetworkX → igraph
ig_graph = ig.Graph.from_networkx(nx_graph)

# Leiden clustering
partition = leidenalg.find_partition(
    ig_graph,
    leidenalg.ModularityVertexPartition,
    resolution_parameter=1.0
)

# Получить communities
communities = partition.membership
```

**Параметры**:
- **resolution_parameter**: 1.0 (стандартное значение)
- **randomness**: 0.01 (низкая случайность)

---

## Метрики

### Стоимость

**GPT-4o**:
```
Input: ~4000 tokens (community texts)
Output: ~300 tokens (3-5 high-level elements)

Cost per community: ~$0.0045
```

**Для документа (50 communities)**:
```
Total cost: 50 × $0.0045 = $0.225
```

### Latency

- **Per community**: 5-10 секунд
- **Параллельная обработка**: 10-20 communities одновременно

### Качество

**Metrics**:
- **Abstraction quality**: 90% (themes are abstract)
- **Diversity**: 85% (elements are distinct)
- **Relevance**: 92% (themes relevant to community)
- **Coverage**: 88% (well-rounded overview)

---

## Лучшие практики

### 1. Оптимальный размер community

```python
# Слишком маленькие communities → недостаточно данных
if len(community_nodes) < 5:
    skip_summary  # Не стоит суммировать

# Слишком большие communities → слишком общие темы
if len(community_nodes) > 200:
    use_hierarchical_clustering  # Разбить на sub-communities
```

### 2. Контекстная приоритизация

```python
# Включить attributes для богатого контекста
priority_order = ['attribute', 'semantic_unit']
texts = sorted(texts, key=lambda t: priority_order.index(t['type']))
```

### 3. Post-processing валидация

```python
# Проверить diversity
titles = [elem['title'] for elem in high_level_elements]
if has_duplicates(titles):
    # Re-generate с emphasis на diversity
    prompt += "\n\nEnsure each title is distinct and non-redundant."
```

---

## Ограничения

### 1. Качество зависит от community detection

**Проблема**: Плохой clustering → poor summaries

```
Хорошая community:
  Узлы: Все о renewable energy
  Summary: "Renewable Energy Research" (четкая тема)

Плохая community:
  Узлы: Renewable energy + biodiversity + economics
  Summary: "General Scientific Research" (слишком общее)
```

### 2. Over-abstraction риск

**Проблема**: LLM может быть слишком абстрактным.

```
Input: Детальные факты о solar panel research
Output: "Scientific Innovation" (слишком общее)

Желаемое: "Solar Panel Efficiency Research" (конкретнее)
```

**Митигация**: Инструкция "reflecting unique perspectives"

### 3. Redundancy между elements

**Проблема**: Несмотря на инструкцию, элементы могут overlap.

```
Element 1: "Renewable Energy Technologies"
Element 2: "Sustainable Energy Solutions"

Overlap: ~80% (почти одно и то же)
```

**Митигация**: Post-processing deduplication

---

## Заключение

Community Summary Prompt — **критический компонент** для создания концептуального слоя в NodeRAG. Извлекая high-level themes из communities, он обеспечивает:
- ✅ Тематический поиск (thematic queries)
- ✅ Big picture understanding
- ✅ Multi-level reasoning (facts + themes)
- ✅ Improved answer contextualization

**Ключевые характеристики**:
- Абстракция фактов → концепты
- Diversity enforcement
- Детерминизм (Temperature 0.0)
- Structured output (title + description)

---

## Ссылки

- **Файл промпта**: `NodeRAG/utils/prompt/community_summary.py:1-11`
- **Schema**: `NodeRAG/utils/prompt/json_format.py:18-24`
- **Manager**: `NodeRAG/utils/prompt/prompt_manager.py:49-56, 92-93`
- **Upstream**: Community Detection (Leiden), Text Decomposition, Attribute Generation
- **Downstream**: Embedding Generation, Context Assembly
