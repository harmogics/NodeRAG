# LLM-based Algorithms в NodeRAG

## Введение

NodeRAG использует **Large Language Models (LLM)** для пяти ключевых задач: структурированного извлечения информации, генерации описаний и синтеза ответов. Все LLM-based алгоритмы реализованы через **сторонний API OpenAI** с моделью **GPT-4o**.

**Ключевая особенность**: LLM обеспечивают **semantic consciousness** — глубокое понимание смысла, невозможное для классических алгоритмов.

---

## Классификация LLM-алгоритмов

| Алгоритм | Pipeline | Цель | Модель | Пакет |
|----------|----------|------|--------|-------|
| **Text Decomposition** | Indexing | Structured Extraction | GPT-4o | `openai==1.66.3` |
| **Query Decomposition** | Query | Entity Extraction | GPT-4o | `openai==1.66.3` |
| **Attribute Generation** | Indexing | Entity Description | GPT-4o | `openai==1.66.3` |
| **Community Summarization** | Indexing | Theme Extraction | GPT-4o | `openai==1.66.3` |
| **Answer Generation** | Query | Question Answering | GPT-4o | `openai==1.66.3` |

---

## 1. Text Decomposition

### Описание алгоритма

**Text Decomposition** — это structured information extraction алгоритм, который разбивает сырой текст на **semantic units**, извлекает **entities** и определяет **relationships** между ними.

**Тип задачи**: Structured Output Generation (JSON)

### Цель применения в NodeRAG

Text Decomposition — это **первый этап indexing pipeline**, создающий структурированный граф знаний:

1. **Semantic Chunking** — Разбиение текста на смысловые единицы
2. **Entity Extraction** — Извлечение всех сущностей (люди, места, организации, времена)
3. **Relationship Detection** — Определение связей между entities

**Результат**: Structured JSON для построения knowledge graph

### Prompt Template

```python
text_decomposition_prompt = """
Goal: Given a text, segment it into multiple semantic units, each containing detailed descriptions of specific events or activities.
Perform the following tasks:
1. Provide a summary for each semantic unit while retaining all crucial details relevant to the original context.
2. Extract all entities directly from the original text of each semantic unit, not from the paraphrased summary. Format each entity name in UPPERCASE. You should extract all entities including times, locations, people, organizations and all kinds of entities.
3. From the entities extracted in Step 2, list all relationships within the semantic unit and the corresponding original context in the form of string seperated by comma : "ENTITY_A, RELATION_TYPE, ENTITY_B". The RELATION_TYPE could be a descriptive sentence, while the entities involved in the relationship must come from the entity names extracted in Step 2. Please make sure the string contains three elements representing two entities and the relationship type.

requirements:
1. Temporal Entities: Represent time entities based on the available details without filling in missing parts. Use specific formats based on what parts of the date or time are mentioned in the text.

Each semantic unit should be represented as a dictionary containing three keys: semantic_unit (a paraphrased summary of each semantic unit), entities (a list of entities extracted directly from the original text of each semantic unit, formatted in UPPERCASE), and relationships (a list of extracted relationship strings that contain three elements, where the relationship type is a descriptive sentence). All these dictionaries should be stored in a list to facilitate management and access.

Text: {text}
"""
```

### Пример Output

**Input text**:
```
In September 2024, Dr. Emily Roberts traveled to Paris to attend the International Conference on Renewable Energy. During her visit, she explored partnerships with several European companies and presented her latest research on solar panel efficiency improvements.
```

**Output (JSON)**:
```json
[
  {
    "semantic_unit": "In September 2024, Dr. Emily Roberts attended the International Conference on Renewable Energy in Paris, where she presented her research on solar panel efficiency improvements and explored partnerships with European companies.",
    "entities": [
      "DR. EMILY ROBERTS",
      "2024-09",
      "PARIS",
      "INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
      "EUROPEAN COMPANIES",
      "SOLAR PANEL EFFICIENCY"
    ],
    "relationships": [
      "DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY",
      "DR. EMILY ROBERTS, explored partnerships with, EUROPEAN COMPANIES",
      "DR. EMILY ROBERTS, presented research on, SOLAR PANEL EFFICIENCY"
    ]
  }
]
```

### Реализация

**Тип**: Сторонний API (OpenAI)
**Модель**: `gpt-4o`
**Пакет**: `openai==1.66.3`
**Файл**: `NodeRAG/utils/prompt/text_decomposition.py`

```python
from openai import OpenAI

class OpenAI_LLM:
    def __init__(self, api_key: str, model_name: str = "gpt-4o"):
        self.client = OpenAI(api_key=api_key)
        self.model_name = model_name

    def text_decomposition(self, text: str) -> dict:
        """
        Structured extraction: text → semantic units + entities + relationships
        """
        prompt = text_decomposition_prompt.format(text=text)

        response = self.client.chat.completions.create(
            model=self.model_name,
            messages=[
                {"role": "system", "content": "You are an expert at structured information extraction."},
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"},  # Structured Output
            temperature=0.0  # Deterministic extraction
        )

        return json.loads(response.choices[0].message.content)
```

### Производительность

**Вычислительная сложность**:
- **Time**: O(L) где L — длина текста (API latency ~1-3s per chunk)
- **Cost**: Зависит от количества tokens

**Практические метрики**:
- **Input tokens**: ~500-2000 per chunk
- **Output tokens**: ~300-1000 per chunk
- **Latency**: ~1-3 seconds per chunk
- **Cost**: ~$0.005-0.015 per chunk (GPT-4o: $2.50/1M input, $10/1M output)

**Для корпуса 10K документов**:
- **Total chunks**: ~50K-100K
- **Execution time**: ~30-60 min (async batching)
- **Total cost**: ~$5-10

### Гиперпараметры

| Параметр | Значение | Описание |
|----------|----------|----------|
| `temperature` | 0.0 | Детерминированное извлечение |
| `response_format` | `json_object` | Structured Output (JSON) |
| `model` | `gpt-4o` | Баланс качества и стоимости |

---

## 2. Query Decomposition

### Описание алгоритма

**Query Decomposition** — это entity extraction алгоритм, который извлекает **ключевые entities** из пользовательского query для создания **symbolic attractors** в PPR.

**Тип задачи**: Entity Extraction (JSON list)

### Цель применения в NodeRAG

Query Decomposition дополняет **HNSW analog attractors** символическими аттракторами:

1. **Entity Extraction** — Извлечение ключевых сущностей из query
2. **Symbolic Attractors** — Создание точных привязок в графе (entity nodes)
3. **Dual-Mode Search** — Комбинация analog (HNSW) + symbolic (entities) для PPR

**Результат**: List of entities для accurate search в графе

### Prompt Template

```python
decompose_query = '''
Please break down the following query into a single list. Each item in the list should either be a main entity (such as a key noun or object). If you have high confidence about the user's intent or domain knowledge, you may also include closely related terms. If uncertain, please only extract entities and semantic chunks directly from the query. Please try to reduce the number of common nouns in the list. Ensure all elements are organized within one unified list.
Query: {query}
'''
```

### Пример Output

**Input query**:
```
What were the main contributions of Dr. Emily Roberts to renewable energy research?
```

**Output (JSON)**:
```json
{
  "elements": [
    "DR. EMILY ROBERTS",
    "RENEWABLE ENERGY",
    "RESEARCH",
    "CONTRIBUTIONS"
  ]
}
```

### Реализация

**Тип**: Сторонний API (OpenAI)
**Модель**: `gpt-4o`
**Пакет**: `openai==1.66.3`
**Файл**: `NodeRAG/search/search.py`

```python
class NodeSearch:
    def decompose_query(self, query: str) -> List[str]:
        """
        Entity extraction из query для symbolic attractors
        """
        prompt = self.config.prompt_manager.decompose_query.format(query=query)

        response = self.config.API_client.request({
            'query': prompt,
            'response_format': self.config.prompt_manager.decomposed_text_json
        })

        return response['elements']

    def accurate_search(self, entities: List[str]) -> List[str]:
        """
        Regex-based matching entities → graph node IDs
        """
        accurate_results = []

        for entity in entities:
            words = entity.lower().split()
            pattern = re.compile(r'\b' + r'\s+'.join(map(re.escape, words)) + r'\b')
            result = [
                id for id, text in self.accurate_id_to_text.items()
                if pattern.search(text.lower())
            ]
            if result:
                accurate_results.extend(result)

        return accurate_results
```

### Производительность

**Вычислительная сложность**:
- **Time**: O(1) per query (API latency ~0.5-1s)
- **Cost**: Minimal (short queries)

**Практические метрики**:
- **Input tokens**: ~20-100 per query
- **Output tokens**: ~10-50 per query
- **Latency**: ~0.5-1 second
- **Cost**: ~$0.0001-0.0005 per query

### Гиперпараметры

| Параметр | Значение | Описание |
|----------|----------|----------|
| `temperature` | 0.0 | Детерминированное извлечение |
| `response_format` | `json_object` | Structured Output (list) |
| `model` | `gpt-4o` | Баланс качества и скорости |

### Связь с другими компонентами

**Dual-Mode Attractors для PPR**:

```
Query Text
    ↓
┌───┴────┐
│        │
HNSW    Query
Search  Decomposition
│        │
Analog  Symbolic
Attractors  Attractors
│        │
└───┬────┘
    ↓
Personalized PageRank
    ↓
Weighted Nodes
```

---

## 3. Attribute Generation

### Описание алгоритма

**Attribute Generation** — это text summarization алгоритм, который создаёт **детальные описания** для important entities на основе их окружения в графе.

**Тип задачи**: Narrative Summarization (character sketch style)

### Цель применения в NodeRAG

Attribute Generation обогащает граф **нарративным контекстом**:

1. **Entity Enrichment** — Создание детальных описаний для ~10-15% entities
2. **Context Consolidation** — Агрегация информации из semantic units и relationships
3. **Semantic Enhancement** — Улучшение качества semantic search (embeddings)

**Результат**: Rich textual attributes для entity nodes

### Prompt Template

```python
attribute_generation_prompt = """
Generate a concise summary of the given entity, capturing its essential attributes and important relevant relationships. The summary should read like a character sketch in a novel or a product description, providing an engaging yet precise overview. Ensure the output only includes the summary of the entity without any additional explanations or metadata. The length must not exceed 2000 words but can be shorter if the input material is limited. Focus on distilling the most important insights with a smooth narrative flow, highlighting the entity's core traits and meaningful connections.

Entity: {entity}
Related Semantic Units: {semantic_units}
Related Relationships: {relationships}
"""
```

### Пример Output

**Input**:
- **Entity**: "DR. EMILY ROBERTS"
- **Semantic Units**: ["In September 2024, Dr. Emily Roberts attended...", "Dr. Roberts published 3 papers on solar efficiency..."]
- **Relationships**: ["DR. EMILY ROBERTS, attended, CONFERENCE", "DR. EMILY ROBERTS, published, RESEARCH"]

**Output**:
```
Dr. Emily Roberts is a leading researcher in renewable energy, specializing in solar panel efficiency improvements. In September 2024, she attended the International Conference on Renewable Energy in Paris, where she presented her groundbreaking research and explored partnerships with European companies. Her work has resulted in three major publications on solar efficiency, contributing significantly to the field. Dr. Roberts is recognized for her innovative approaches to sustainable energy solutions and her collaborative efforts with international organizations.
```

### Реализация

**Тип**: Сторонний API (OpenAI)
**Модель**: `gpt-4o`
**Пакет**: `openai==1.66.3`
**Файл**: `NodeRAG/build/pipeline/attribute_generation.py`

```python
class Attribution_generation_pipeline:
    def __init__(self, config: NodeConfig):
        self.config = config
        self.API_client = config.API_client
        self.G = storage.load(self.config.graph_path)
        self.important_nodes = []

    def get_important_nodes(self):
        """
        K-Core + Betweenness для отбора ~10-15% entities
        """
        node_importance = NodeImportance(self.G, self.config.console)
        important_nodes = node_importance.main()
        self.important_nodes = important_nodes

    def get_neighbours_material(self, node: str):
        """
        Агрегация context из соседей entity в графе
        """
        entity = self.mapper.get(node, 'context')
        semantic_neighbours = '' + '\n'
        relationship_neighbours = '' + '\n'

        for neighbour in self.G.neighbors(node):
            if self.G.nodes[neighbour]['type'] == 'semantic_unit':
                semantic_neighbours += f'{self.mapper.get(neighbour, "context")}\n'
            elif self.G.nodes[neighbour]['type'] == 'relationship':
                relationship_neighbours += f'{self.mapper.get(neighbour, "context")}\n'

        query = self.prompt_manager.attribute_generation.format(
            entity=entity,
            semantic_units=semantic_neighbours,
            relationships=relationship_neighbours
        )
        return query

    async def generate_attribution(self, node: str):
        """
        LLM summarization для entity
        """
        query = self.get_neighbours_material(node)

        response = await self.API_client({'query': query})

        if response is not None:
            attribute = Attribute(response, node)
            self.G.nodes[node]['attributes'] = [attribute.hash_id]
            self.G.add_node(attribute.hash_id, type='attribute', weight=1)
            self.G.add_edge(node, attribute.hash_id, weight=1)
```

### Производительность

**Вычислительная сложность**:
- **Time**: O(N_important) где N_important ~10-15% entities (API latency ~1-3s)
- **Cost**: Зависит от количества context

**Практические метрики**:
- **Input tokens**: ~500-1500 per entity (context from neighbours)
- **Output tokens**: ~200-800 per entity (narrative summary)
- **Latency**: ~1-3 seconds per entity
- **Cost**: ~$0.005-0.015 per entity

**Для корпуса 10K документов**:
- **Important entities**: ~5K-10K
- **Execution time**: ~15-30 min (async batching)
- **Total cost**: ~$10-15

### Гиперпараметры

| Параметр | Значение | Описание |
|----------|----------|----------|
| `temperature` | 0.5-0.7 | Умеренная креативность |
| `max_tokens` | 1000 | Ограничение длины (~2000 words) |
| `model` | `gpt-4o` | Баланс качества и стоимости |

---

## 4. Community Summarization

### Описание алгоритма

**Community Summarization** — это abstract summarization алгоритм, который создаёт **high-level themes** из сообществ, обнаруженных Leiden algorithm.

**Тип задачи**: Multi-Document Summarization (abstract themes)

### Цель применения в NodeRAG

Community Summarization создаёт **иерархию абстракций**:

1. **Theme Extraction** — Извлечение общих тем из кластера текстов
2. **High-Level Elements** — Создание узлов-абстракций (HLE)
3. **Diversity Preservation** — Сохранение разнообразия перспектив

**Результат**: High-level elements (HLE) для иерархического поиска

### Prompt Template

```python
community_summary = """
You will receive a set of text data from the same cluster. Your task is to extract distinct categories of high-level information, such as concepts, themes, relevant theories, potential impacts, and key insights. Each piece of information should include a concise title and a corresponding description, reflecting the unique perspectives within the text cluster.

Please do not attempt to include all possible information; instead, select the elements that have the most significance and diversity in this cluster. Avoid redundant information—if there are highly similar elements, combine them into a single, comprehensive entry. Ensure that the high-level information reflects the varied dimensions within the text, providing a well-rounded overview.

clustered text data:
{content}
"""
```

### Пример Output

**Input** (semantic units from Leiden community):
```
[
  "Dr. Emily Roberts attended renewable energy conference...",
  "Solar panel efficiency research shows 20% improvement...",
  "European partnerships for sustainable energy projects...",
  ...
]
```

**Output (JSON)**:
```json
{
  "high_level_elements": [
    {
      "title": "Renewable Energy Research Advances",
      "description": "Recent breakthroughs in solar panel efficiency, including 20% improvements through novel materials and design optimizations, are driving significant progress in renewable energy technology."
    },
    {
      "title": "International Collaboration in Sustainability",
      "description": "Cross-border partnerships between researchers and European companies are fostering innovation in sustainable energy solutions, with a focus on practical implementation and knowledge sharing."
    }
  ]
}
```

### Реализация

**Тип**: Сторонний API (OpenAI)
**Модель**: `gpt-4o`
**Пакет**: `openai==1.66.3`
**Файл**: `NodeRAG/build/pipeline/summary_generation.py`

```python
class SummaryGeneration:
    def __init__(self, config: NodeConfig):
        self.config = config
        self.G = storage.load(config.graph_path)
        self.G_ig = IGraph(self.G).to_igraph()
        self.communities = []

    def partition(self):
        """
        Leiden community detection
        """
        partition = la.find_partition(self.G_ig, la.ModularityVertexPartition)

        for i, community in enumerate(partition):
            community_names = [
                self.G_ig.vs[node]['name']
                for node in community
                if self.G_ig.vs[node]['name'] in self.mapper.embeddings
            ]
            self.communities.append(
                Community_summary(community_names, self.mapper, self.G, self.config)
            )

    async def generate_community_summary(self, community: Community_summary):
        """
        LLM summarization для community → high-level elements
        """
        await community.generate_community_summary()

        if isinstance(community.response, str):
            return

        community_dict = {
            'community': community.community_node,
            'response': community.response,
            'hash_id': community.hash_id,
            'human_readable_id': community.human_readable_id
        }

        with open(self.config.summary_path, 'a', encoding='utf-8') as f:
            f.write(json.dumps(community_dict, ensure_ascii=False) + '\n')
```

### Производительность

**Вычислительная сложность**:
- **Time**: O(N_communities) где N_communities ~ √N (API latency ~2-5s)
- **Cost**: Зависит от размера community

**Практические метрики**:
- **Input tokens**: ~1000-5000 per community (aggregated text)
- **Output tokens**: ~200-1000 per community (HLE titles + descriptions)
- **Latency**: ~2-5 seconds per community
- **Cost**: ~$0.01-0.05 per community

**Для корпуса 10K документов**:
- **Communities**: ~200-500
- **Execution time**: ~10-20 min (async batching)
- **Total cost**: ~$5-10

### Гиперпараметры

| Параметр | Значение | Описание |
|----------|----------|----------|
| `temperature` | 0.3-0.5 | Умеренная креативность |
| `max_tokens` | 1500 | Множественные HLE |
| `model` | `gpt-4o` | Баланс качества и стоимости |

---

## 5. Answer Generation

### Описание алгоритма

**Answer Generation** — это question answering алгоритм, который синтезирует финальный ответ из **retrieved context** (PPR results).

**Тип задачи**: Abstractive Question Answering (RAG)

### Цель применения в NodeRAG

Answer Generation — это **финальный этап query pipeline**:

1. **Context Synthesis** — Интеграция multiple sources из retrieval
2. **Fact Verification** — Проверка evidence-based ответа
3. **Creative Generation** — Связный, читаемый текст

**Результат**: Multi-paragraph answer на основе retrieved context

### Prompt Template

```python
answer_prompt = """
---Role---

You are a thorough assistant responding to questions based on retrieved information.

---Goal---

Provide a clear and accurate response. Carefully review and verify the retrieved data, and integrate any relevant necessary knowledge to comprehensively address the user's question.
If you are unsure of the answer, just say so. Do not fabricate information.
Do not include details not supported by the provided evidence.

---Target response length and format---

Multiple Paragraphs

---Retrived Context---

{info}

---Query---

{query}
"""
```

### Пример Output

**Input**:
- **Query**: "What were the main contributions of Dr. Emily Roberts?"
- **Retrieved Context**: [Semantic units, entities, attributes, relationships from PPR]

**Output**:
```
Dr. Emily Roberts has made significant contributions to renewable energy research, particularly in the field of solar panel efficiency. In September 2024, she presented her latest research at the International Conference on Renewable Energy in Paris, where she showcased improvements in solar panel efficiency that achieved up to 20% better performance through novel materials and design optimizations.

Her work extends beyond technical innovation to include building strategic partnerships with European companies focused on sustainable energy solutions. Through these collaborations, Dr. Roberts has facilitated the practical implementation of research findings and promoted knowledge sharing across international borders. Her published research papers have become foundational references in the field, contributing significantly to the advancement of renewable energy technology.
```

### Реализация

**Тип**: Сторонний API (OpenAI)
**Модель**: `gpt-4o`
**Пакет**: `openai==1.66.3`
**Файл**: `NodeRAG/search/search.py`

```python
class NodeSearch:
    def answer(self, query: str, id_type: bool = True):
        """
        RAG: Retrieval + Answer Generation
        """
        # 1. Retrieval (HNSW + Query Decomposition + PPR)
        retrieval = self.search(query)

        ans = Answer(query, retrieval)

        # 2. Context Assembly
        if id_type:
            retrieved_info = ans.structured_prompt
        else:
            retrieved_info = ans.unstructured_prompt

        # 3. Answer Generation (LLM)
        prompt = self.config.prompt_manager.answer.format(
            info=retrieved_info,
            query=query
        )

        ans.response = self.config.API_client.request({'query': prompt})

        return ans

    async def answer_async(self, query: str, id_type: bool = True):
        """
        Async version для batch processing
        """
        retrieval = self.search(query)
        ans = Answer(query, retrieval)

        if id_type:
            retrieved_info = ans.structured_prompt
        else:
            retrieved_info = ans.unstructured_prompt

        prompt = self.config.prompt_manager.answer.format(
            info=retrieved_info,
            query=query
        )

        ans.response = await self.config.API_client({'query': prompt})

        return ans
```

### Производительность

**Вычислительная сложность**:
- **Time**: O(1) per query (API latency ~1-3s)
- **Cost**: Зависит от количества retrieved context

**Практические метрики**:
- **Input tokens**: ~500-2000 per query (retrieved context)
- **Output tokens**: ~200-800 per query (answer)
- **Latency**: ~1-3 seconds
- **Cost**: ~$0.005-0.020 per query

### Гиперпараметры

| Параметр | Значение | Описание |
|----------|----------|----------|
| `temperature` | 0.7-1.0 | Креативный синтез |
| `max_tokens` | 1000 | Multiple paragraphs |
| `model` | `gpt-4o` | Баланс качества и стоимости |

---

## Сравнительная таблица LLM-алгоритмов

| Алгоритм | Pipeline | Input Tokens | Output Tokens | Temperature | Cost per Call |
|----------|----------|--------------|---------------|-------------|---------------|
| **Text Decomposition** | Indexing | 500-2000 | 300-1000 | 0.0 | $0.005-0.015 |
| **Query Decomposition** | Query | 20-100 | 10-50 | 0.0 | $0.0001-0.0005 |
| **Attribute Generation** | Indexing | 500-1500 | 200-800 | 0.5-0.7 | $0.005-0.015 |
| **Community Summary** | Indexing | 1000-5000 | 200-1000 | 0.3-0.5 | $0.01-0.05 |
| **Answer Generation** | Query | 500-2000 | 200-800 | 0.7-1.0 | $0.005-0.020 |

---

## Архитектура взаимодействия

### Indexing Pipeline

```
Raw Text
    ↓
Text Decomposition (LLM)
    ↓
Semantic Units + Entities + Relationships
    ↓
Graph Construction
    ↓
K-Core + Betweenness
    ↓
Attribute Generation (LLM)
    ↓
Leiden Clustering
    ↓
Community Summarization (LLM)
    ↓
High-Level Elements
    ↓
Knowledge Graph
```

### Query Pipeline

```
User Query
    ↓
┌───────────┴───────────┐
│                       │
Query Embedding      Query Decomposition (LLM)
│                       │
HNSW Search          Accurate Search
│                       │
Analog Attractors    Symbolic Attractors
│                       │
└───────────┬───────────┘
            ↓
   Personalized PageRank
            ↓
      Weighted Nodes
            ↓
    Context Assembly
            ↓
  Answer Generation (LLM)
            ↓
      Final Answer
```

---

## Концептуальные связи

### Semantic Consciousness

**LLM как "semantic consciousness" системы**:

```
Traditional Algorithms              LLM Algorithms
(Syntactic Processing)         (Semantic Understanding)
         ↓                              ↓
    Regex, Rules                 Deep Semantics
    Pattern Matching             Context Understanding
    Statistical ML               Reasoning & Synthesis
```

**Комплементарность**:
- **HNSW**: Быстрый векторный поиск, но не понимает смысл
- **LLM**: Глубокое понимание, но медленное и дорогое

### Question as Key Paradigm

**LLM реализуют "semantic key matching"**:

```
Query (Question)
       ↓
LLM Decomposition → Semantic Keys (entities)
       ↓
Graph Traversal → Lock Opening (PPR)
       ↓
Retrieved Context
       ↓
LLM Synthesis → Answer (treasure)
```

### Иерархия абстракций

**LLM создают multiple levels of abstraction**:

```
КОНКРЕТИКА ↓

Raw Text (documents)
    ↓ Text Decomposition (LLM)
Semantic Units (events, facts)
    ↓ Text Decomposition (LLM)
Entities + Relationships (structured)
    ↓ Attribute Generation (LLM)
Attributes (narrative descriptions)
    ↓ Community Summarization (LLM)
High-Level Elements (themes, concepts)

АБСТРАКЦИЯ ↑
```

---

## Реализация: OpenAI API

### Core Package

| Библиотека | Версия | Модель | Лицензия |
|------------|--------|--------|----------|
| `openai` | 1.66.3 | GPT-4o | Proprietary API |

### API Configuration

```python
from openai import OpenAI

class OpenAI_LLM(LLMBase):
    def __init__(self, api_key: str, model_name: str = "gpt-4o"):
        self.client = OpenAI(api_key=api_key)
        self.model_name = model_name
        self.config = {
            'temperature': 0.7,
            'max_tokens': 1000
        }

    def predict(self, input: LLM_message) -> LLMOutput:
        """
        Synchronous LLM call
        """
        messages = []

        if 'system_prompt' in input:
            messages.append({
                "role": "system",
                "content": input['system_prompt']
            })

        messages.append({
            "role": "user",
            "content": input['query']
        })

        response = self.client.chat.completions.create(
            model=self.model_name,
            messages=messages,
            **self.config
        )

        return response.choices[0].message.content

    async def predict_async(self, input: LLM_message) -> LLMOutput:
        """
        Asynchronous LLM call для batch processing
        """
        # Similar to predict but with async OpenAI client
        pass
```

### Cost Estimation

**GPT-4o Pricing** (2025):
- **Input**: $2.50 per 1M tokens
- **Output**: $10.00 per 1M tokens

**Indexing Pipeline (10K docs)**:
- Text Decomposition: ~$5-10
- Attribute Generation: ~$10-15
- Community Summarization: ~$5-10
- **Total**: ~$20-35

**Query Pipeline (per query)**:
- Query Decomposition: ~$0.0001-0.0005
- Answer Generation: ~$0.005-0.020
- **Total**: ~$0.01-0.02

---

## Альтернативные реализации

### 1. Google Gemini

**Модель**: `gemini-1.5-pro`, `gemini-1.5-flash`
**Пакет**: `google-genai`

```python
import google.generativeai as genai

class Gemini_LLM(LLMBase):
    def __init__(self, api_key: str, model_name: str = "gemini-1.5-pro"):
        genai.configure(api_key=api_key)
        self.model = genai.GenerativeModel(model_name)

    def predict(self, input: LLM_message) -> LLMOutput:
        response = self.model.generate_content(input['query'])
        return response.text
```

**Pros**: Дешевле, длинный context (1M tokens)
**Cons**: Меньше контроля над structured output

### 2. Local LLMs (Llama, Mistral)

**Модели**: `llama-3-70b`, `mistral-large`
**Пакет**: `transformers`, `vllm`

**Pros**: Бесплатно, privacy, offline
**Cons**: Требует GPU, lower quality

### 3. Anthropic Claude

**Модель**: `claude-3-opus`, `claude-3-sonnet`
**Пакет**: `anthropic`

**Pros**: Высокое качество reasoning, длинный context
**Cons**: Дороже GPT-4o

**NodeRAG выбор**: GPT-4o (баланс качества, стоимости, reliability)

---

## Trade-offs и оптимизации

### 1. Model Selection

**Temperature Trade-off**:
- **Low (0.0-0.3)**: Детерминированное извлечение (Text/Query Decomposition)
- **Medium (0.5-0.7)**: Баланс (Attribute Generation, Community Summary)
- **High (0.7-1.0)**: Креативный синтез (Answer Generation)

### 2. Cost Optimization

**Strategies**:
1. **Batch processing**: Async API calls для параллелизма
2. **Caching**: Кэшировать decomposition, attributes
3. **Prompt engineering**: Сокращение prompt length
4. **Model selection**: GPT-4o-mini для простых задач

### 3. Latency Reduction

**Strategies**:
1. **Async calls**: `asyncio.gather()` для batch
2. **Streaming**: `stream_chat()` для real-time answers
3. **Early stopping**: max_tokens limits

---

## Рекомендации

### Для разработчиков

1. **Error handling**: Retry logic для API failures
2. **Rate limiting**: Respect OpenAI rate limits (10K RPM)
3. **Monitoring**: Логировать token usage, costs
4. **Prompt versioning**: Track prompt changes для reproducibility

### Для исследователей

1. **Ablation studies**: Попробовать без LLM (baselines)
2. **Prompt engineering**: A/B testing разных prompts
3. **Alternative models**: Gemini, Claude, local LLMs
4. **Evaluation metrics**: ROUGE, BLEU для summarization; F1 для extraction

---

## Дополнительные ресурсы

### Papers

- **GPT-4**: OpenAI (2023). "GPT-4 Technical Report"
- **RAG**: Lewis et al. (2020). "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"
- **Prompt Engineering**: Wei et al. (2022). "Chain-of-Thought Prompting Elicits Reasoning in LLMs"

### Связанные разделы

- [Transformer Embeddings](./transformer-embeddings.md)
- [Graph ML Algorithms](./graph-ml.md)
- [LLM and AI Dependencies](../dependencies/llm-and-ai.md)
- [Question as Key Paradigm](../research/question-as-key-paradigm.md)
- [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)

---

**Последнее обновление**: 2025-11-14
**Версия документации**: 1.0
**Для вопросов**: См. README в корне проекта
