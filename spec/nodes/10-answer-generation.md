# Answer Generation 💬

## Обзор

Этот документ описывает, как результаты retrieval преобразуются в финальные ответы, роль каждого типа узла в контексте, стратегии сборки context, и примеры answer construction.

---

## Answer Generation Pipeline

### High-Level Flow

```
Retrieval Results
    ↓
Context Assembly
    ↓
Prompt Construction
    ↓
LLM Generation
    ↓
Final Answer
```

### Detailed Process

```python
def answer(query: str) -> Answer:
    # 1. Search and retrieval
    retrieval = search(query)
    # retrieval.search_list = [node_hash_1, node_hash_2, ...]
    # retrieval.entities = [...]
    # retrieval.relationships = [...]
    # retrieval.he_titles = [...]

    # 2. Context assembly
    context = assemble_context(retrieval)

    # 3. Prompt construction
    prompt = answer_prompt.format(
        info=context,
        query=query
    )

    # 4. LLM call
    response = LLM_client({'query': prompt})

    # 5. Return answer
    return Answer(query=query, retrieval=retrieval, response=response)
```

---

## Context Assembly Strategies

### Strategy 1: Structured Context

**Format**: Organize by node types

```python
def structured_context(retrieval):
    context_parts = []

    # 1. High-Level Elements (themes)
    if retrieval.he_titles:
        context_parts.append("=== THEMES ===")
        for he_title_hash in retrieval.he_titles:
            title = mapper.get(he_title_hash, 'context')
            he_hash = G.nodes[he_title_hash]['related_node']
            description = mapper.get(he_hash, 'context')
            context_parts.append(f"\nTheme: {title}\n{description}")

    # 2. Entities (key actors/concepts)
    if retrieval.entities:
        context_parts.append("\n=== KEY ENTITIES ===")
        for entity_hash in retrieval.entities:
            entity_name = mapper.get(entity_hash, 'context')
            context_parts.append(f"\n- {entity_name}")

            # Include attribute if exists
            if 'attributes' in G.nodes[entity_hash]:
                for attr_hash in G.nodes[entity_hash]['attributes']:
                    attr_text = mapper.get(attr_hash, 'context')
                    context_parts.append(f"  Description: {attr_text}")

    # 3. Relationships (structured connections)
    if retrieval.relationships:
        context_parts.append("\n=== RELATIONSHIPS ===")
        for rel_hash in retrieval.relationships:
            rel_context = mapper.get(rel_hash, 'context')
            # Parse relationship (may have multiple formulations)
            formulations = rel_context.split('\t')
            context_parts.append(f"\n- {formulations[0]}")

    # 4. Semantic Units (detailed facts)
    context_parts.append("\n=== FACTS ===")
    for node_hash in retrieval.search_list:
        node_type = G.nodes[node_hash]['type']

        if node_type == 'semantic_unit':
            su_context = mapper.get(node_hash, 'context')
            context_parts.append(f"\n- {su_context}")

        elif node_type == 'attribute':
            attr_context = mapper.get(node_hash, 'context')
            context_parts.append(f"\n- {attr_context}")

        elif node_type == 'text_unit':
            tu_context = mapper.get(node_hash, 'context')
            context_parts.append(f"\n[Text Unit] {tu_context}")

        elif node_type == 'high_level_element':
            # Already included in themes section
            pass

    return '\n'.join(context_parts)
```

**Example Output**:
```
=== THEMES ===

Theme: Renewable Energy Research
This theme covers research and developments in renewable energy,
particularly focusing on solar panel technology and efficiency
improvements...

=== KEY ENTITIES ===

- DR. EMILY ROBERTS
  Description: Dr. Emily Roberts is a leading researcher at the
  European Research Institute, specializing in renewable energy...

- EUROPEAN RESEARCH INSTITUTE
- SOLAR PANEL EFFICIENCY

=== RELATIONSHIPS ===

- DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE
- DR. EMILY ROBERTS researches SOLAR PANEL EFFICIENCY
- EUROPEAN RESEARCH INSTITUTE located in EUROPE

=== FACTS ===

- In September 2024, Dr. Emily Roberts attended the International
  Conference on Renewable Energy in Paris.
- Her research demonstrated a 15% improvement in solar panel efficiency.
- The conference focused on advances in photovoltaic systems.
...
```

### Strategy 2: Unstructured Context

**Format**: Flat list of all contexts

```python
def unstructured_context(retrieval):
    contexts = []

    for node_hash in retrieval.search_list:
        context = mapper.get(node_hash, 'context')
        contexts.append(context)

    # Add entities (names only)
    for entity_hash in retrieval.entities:
        entity_name = mapper.get(entity_hash, 'context')
        contexts.append(f"Entity: {entity_name}")

    # Add relationships
    for rel_hash in retrieval.relationships:
        rel_context = mapper.get(rel_hash, 'context')
        contexts.append(rel_context)

    return '\n\n'.join(contexts)
```

**Example Output**:
```
In September 2024, Dr. Emily Roberts attended the International
Conference on Renewable Energy in Paris.

Dr. Emily Roberts is a leading researcher at the European Research
Institute, specializing in renewable energy technologies.

Her research demonstrated a 15% improvement in solar panel efficiency.

Entity: DR. EMILY ROBERTS
Entity: EUROPEAN RESEARCH INSTITUTE
Entity: SOLAR PANEL EFFICIENCY

DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE
DR. EMILY ROBERTS researches SOLAR PANEL EFFICIENCY
...
```

### Comparison

| Aspect          | Structured                  | Unstructured          |
|-----------------|-----------------------------|-----------------------|
| Organization    | Clear sections              | Flat list             |
| LLM Processing  | Easier to navigate          | Must infer structure  |
| Token Efficiency| Section headers add tokens  | More compact          |
| Use Case        | Complex queries             | Simple queries        |

**Recommended**: Structured для better answer quality

---

## Role of Each Node Type

### Semantic Units

**Role**: Detailed facts и evidence

**Contribution**:
- Specific claims and statements
- Event descriptions
- Factual details

**In Context**:
```
FACTS:
- Dr. Emily Roberts attended conference in Paris, September 2024
- Research demonstrated 15% efficiency improvement
- Conference focused on renewable energy advances
```

**Answer Usage**:
```
Dr. Emily Roberts attended the International Conference on Renewable
Energy in Paris in September 2024, where she presented research
demonstrating a 15% improvement in solar panel efficiency.
```

**Advantages**:
✅ Precise, atomic facts
✅ Direct citations possible
✅ High信度 (from specific text units)

### Entities

**Role**: Key actors и concepts

**Contribution**:
- Identify главные действующие лица
- Establish context
- Enable entity-centric answers

**In Context**:
```
KEY ENTITIES:
- DR. EMILY ROBERTS
- EUROPEAN RESEARCH INSTITUTE
- PARIS
- 2024-09
```

**Answer Usage**:
```
Dr. Emily Roberts, a researcher at the European Research Institute,
traveled to Paris in September 2024...
```

**Advantages**:
✅ Structure answer around entities
✅ Ensure key entities mentioned
✅ Connect facts через entities

### Relationships

**Role**: Structured connections

**Contribution**:
- Explicit (subject, predicate, object) triples
- Clear connections between entities
- Multi-hop reasoning support

**In Context**:
```
RELATIONSHIPS:
- DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE
- DR. EMILY ROBERTS researches SOLAR PANEL EFFICIENCY
- EUROPEAN RESEARCH INSTITUTE located in EUROPE
```

**Answer Usage**:
```
Dr. Emily Roberts works at the European Research Institute, where
she conducts research on solar panel efficiency. The institute is
based in Europe.
```

**Advantages**:
✅ Clear, structured knowledge
✅ Reduces ambiguity
✅ Supports reasoning chains

### Attributes

**Role**: Comprehensive entity descriptions

**Contribution**:
- Rich background information
- Aggregated knowledge о entity
- Context для understanding entity role

**In Context**:
```
- DR. EMILY ROBERTS
  Description: Dr. Emily Roberts is a leading researcher at the
  European Research Institute, specializing in renewable energy
  technologies. Her work focuses on solar panel efficiency
  improvements, and she has published multiple papers demonstrating
  significant breakthroughs. She is an active participant in
  international conferences.
```

**Answer Usage**:
```
Dr. Emily Roberts is a prominent renewable energy researcher at the
European Research Institute. She specializes in solar panel efficiency
and has achieved significant breakthroughs in the field, regularly
presenting her work at international conferences.
```

**Advantages**:
✅ Comprehensive overview
✅ Reduces need for multiple semantic units
✅ Coherent, LLM-generated summaries

### High-Level Elements

**Role**: Thematic framing

**Contribution**:
- Abstract context
- Thematic organization
- Broader perspective

**In Context**:
```
THEMES:

Theme: Renewable Energy Research
This theme covers research and developments in renewable energy,
particularly focusing on solar panel technology and efficiency
improvements. It includes various studies on enhancing energy
conversion rates and innovations in solar cell materials.
```

**Answer Usage**:
```
In the field of renewable energy research, significant advances are
being made in solar panel technology. Dr. Emily Roberts' work
exemplifies these efforts, with her focus on improving photovoltaic
efficiency.
```

**Advantages**:
✅ Sets broader context
✅ Helps LLM understand domain
✅ Improves answer coherence

### Text Units

**Role**: Broader context fallback

**Contribution**:
- Original text chunks
- Paragraph-level context
- Narrative flow

**In Context**:
```
[Text Unit]
Renewable energy research is crucial for addressing climate change.
In September 2024, Dr. Emily Roberts from the European Research
Institute attended the International Conference on Renewable Energy
in Paris, where she presented research on solar panel efficiency
improvements. Her work demonstrated a 15% increase in photovoltaic
performance, representing a significant breakthrough in the field.
```

**Answer Usage**:
- LLM может directly quote или paraphrase
- Preserves original formulations
- Provides context around specific facts

**When Used**:
- Semantic units too fragmented
- Need broader narrative
- Query requires paragraph-level context

---

## Prompt Construction

### Answer Prompt Template

```python
answer_prompt = """
You are a helpful assistant answering questions based on provided
information from a knowledge graph.

Information Retrieved:
{info}

User Question: {query}

Instructions:
1. Answer based ONLY on the information provided above
2. If information is insufficient, say "I don't have enough information"
3. Cite specific facts when possible
4. Be concise but comprehensive
5. Maintain factual accuracy

Answer:
"""
```

### Variations

**1. Citation-Focused**:
```python
"""
...
When making claims, reference the source information by saying
"According to the retrieved information, ..." or "Based on the facts, ..."
...
"""
```

**2. Structured Answer**:
```python
"""
...
Provide your answer in the following structure:
1. Direct answer (1-2 sentences)
2. Supporting details (2-3 sentences)
3. Additional context if relevant (1-2 sentences)
...
"""
```

**3. Conversational**:
```python
"""
...
Provide a natural, conversational answer as if explaining to a colleague.
Be friendly but professional.
...
"""
```

---

## Answer Assembly Examples

### Example 1: Entity-Centric Query

**Query**: "Who is Dr. Emily Roberts?"

**Retrieved Nodes**:
- Entity: DR. EMILY ROBERTS
- Attribute: (comprehensive description)
- Semantic Units: 5 facts about Dr. Roberts
- Relationships: 3 connections

**Context**:
```
KEY ENTITIES:
- DR. EMILY ROBERTS
  Description: Dr. Emily Roberts is a leading researcher at the
  European Research Institute, specializing in renewable energy
  technologies. Her primary focus is on improving solar panel
  efficiency. She has achieved significant breakthroughs and is
  an active participant in international conferences.

RELATIONSHIPS:
- DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE
- DR. EMILY ROBERTS researches SOLAR PANEL EFFICIENCY
- DR. EMILY ROBERTS presented at INTERNATIONAL CONFERENCE

FACTS:
- In September 2024, Dr. Emily Roberts attended conference in Paris
- Her research demonstrated a 15% improvement in solar panel efficiency
- She is recognized for her work in sustainable energy research
```

**Generated Answer**:
```
Dr. Emily Roberts is a leading researcher at the European Research
Institute, where she specializes in renewable energy technologies,
with a particular focus on improving solar panel efficiency. Her work
has achieved significant breakthroughs, including a demonstrated 15%
improvement in photovoltaic performance. Dr. Roberts is an active
participant in the international research community, having presented
her findings at conferences such as the International Conference on
Renewable Energy held in Paris in September 2024. She is widely
recognized for her contributions to sustainable energy research.
```

**Node Contributions**:
- **Attribute**: Основа answer (comprehensive overview)
- **Semantic Units**: Specific details (15%, Paris, September 2024)
- **Relationships**: Structure (works at, researches, presented at)
- **Entities**: Context (institute, locations, topics)

### Example 2: Relationship Query

**Query**: "Where does Dr. Emily Roberts work and what does she research?"

**Retrieved Nodes**:
- Entities: DR. EMILY ROBERTS, EUROPEAN RESEARCH INSTITUTE, SOLAR PANEL EFFICIENCY
- Relationships: works at, researches
- Semantic Units: 3 facts
- Attribute: description of institute

**Context**:
```
KEY ENTITIES:
- DR. EMILY ROBERTS
- EUROPEAN RESEARCH INSTITUTE
  Description: The European Research Institute is a prominent research
  organization based in Europe, focusing on renewable energy and
  sustainable technologies.
- SOLAR PANEL EFFICIENCY

RELATIONSHIPS:
- DR. EMILY ROBERTS works at EUROPEAN RESEARCH INSTITUTE
- DR. EMILY ROBERTS researches SOLAR PANEL EFFICIENCY
- EUROPEAN RESEARCH INSTITUTE researches RENEWABLE ENERGY

FACTS:
- Dr. Emily Roberts focuses on improving photovoltaic systems
- The institute conducts cutting-edge research in solar technology
- Research has demonstrated significant efficiency improvements
```

**Generated Answer**:
```
Dr. Emily Roberts works at the European Research Institute, a prominent
research organization based in Europe that focuses on renewable energy
and sustainable technologies. Her research specifically centers on solar
panel efficiency, where she investigates improvements to photovoltaic
systems. This work aligns with the institute's broader focus on renewable
energy research and has demonstrated significant efficiency improvements
in solar technology.
```

**Node Contributions**:
- **Relationships**: Core structure (works at, researches)
- **Attribute**: Institute context
- **Entities**: Key actors and concepts
- **Semantic Units**: Supporting details

### Example 3: Concept Query

**Query**: "What is renewable energy research about?"

**Retrieved Nodes**:
- High-Level Element: "Renewable Energy Research" theme
- Semantic Units: 10 facts about research
- Entities: SOLAR PANEL EFFICIENCY, PHOTOVOLTAIC SYSTEMS, WIND ENERGY
- Attributes: 2 descriptions

**Context**:
```
THEMES:

Theme: Renewable Energy Research
This theme covers research and developments in renewable energy,
particularly focusing on solar panel technology, photovoltaic systems,
and efficiency improvements. It includes studies on enhancing energy
conversion rates, innovations in solar cell materials, and practical
applications in sustainable energy systems. The theme also encompasses
international collaborations and conferences dedicated to advancing
renewable energy solutions.

KEY ENTITIES:
- SOLAR PANEL EFFICIENCY
- PHOTOVOLTAIC SYSTEMS
- WIND ENERGY
- SUSTAINABLE ENERGY SYSTEMS

FACTS:
- Research focuses on improving energy conversion efficiency
- Innovations include new solar cell materials
- International conferences facilitate collaboration
- 15% efficiency improvements have been demonstrated
- Practical applications in sustainable systems are being developed
...
```

**Generated Answer**:
```
Renewable energy research encompasses a broad range of studies and
developments focused on sustainable energy solutions. Key areas include
solar panel technology and photovoltaic systems, where researchers are
working to improve energy conversion efficiency through innovations in
solar cell materials. Recent work has demonstrated significant progress,
with efficiency improvements of up to 15% being achieved. The field also
includes wind energy and the development of practical applications for
sustainable energy systems. International collaboration plays a vital
role, with conferences bringing together researchers to share advances
and coordinate efforts in developing next-generation renewable energy
technologies.
```

**Node Contributions**:
- **High-Level Element**: Thematic framing, overview
- **Semantic Units**: Specific facts and examples
- **Entities**: Key concepts and topics
- **Attributes**: Detailed descriptions

---

## Context Optimization

### Token Budget Management

**Challenge**: LLM context window limits (4K-128K tokens)

**Strategies**:

**1. Prioritization**:
```python
def prioritize_nodes(retrieval_nodes, query_embedding, budget=2000):
    # Score nodes by relevance
    scored_nodes = []
    for node_hash in retrieval_nodes:
        node_embedding = mapper.get_embedding(node_hash)
        similarity = cosine_similarity(query_embedding, node_embedding)
        token_cost = estimate_tokens(mapper.get(node_hash, 'context'))

        scored_nodes.append({
            'hash': node_hash,
            'similarity': similarity,
            'tokens': token_cost,
            'value': similarity / token_cost  # Value per token
        })

    # Sort by value
    scored_nodes.sort(key=lambda x: x['value'], reverse=True)

    # Select до budget
    selected = []
    total_tokens = 0
    for node in scored_nodes:
        if total_tokens + node['tokens'] <= budget:
            selected.append(node['hash'])
            total_tokens += node['tokens']

    return selected
```

**2. Summarization**:
```python
def summarize_if_needed(context, max_tokens=2000):
    if estimate_tokens(context) > max_tokens:
        # LLM summarization
        summary = LLM_client({
            'query': f"Summarize the following in ~{max_tokens} tokens:\n\n{context}"
        })
        return summary
    return context
```

**3. Hierarchical Selection**:
```python
priority_order = [
    'attribute',           # Comprehensive descriptions (high value)
    'high_level_element',  # Thematic context
    'semantic_unit',       # Detailed facts
    'relationship',        # Structured connections
    'text_unit'            # Broader context (last resort)
]

# Select по priority
for node_type in priority_order:
    type_nodes = [n for n in retrieval if G.nodes[n]['type'] == node_type]
    # Add до budget
    ...
```

### Deduplication

**Challenge**: Overlapping information в разных nodes

**Strategy**:
```python
def deduplicate_content(contexts):
    unique_contexts = []
    seen_hashes = set()

    for context in contexts:
        # Content-based deduplication
        content_hash = hashlib.md5(context.encode()).hexdigest()

        if content_hash not in seen_hashes:
            unique_contexts.append(context)
            seen_hashes.add(content_hash)

    return unique_contexts
```

**Fuzzy deduplication**:
```python
from difflib import SequenceMatcher

def fuzzy_deduplicate(contexts, threshold=0.85):
    unique_contexts = []

    for context in contexts:
        # Check similarity with existing
        is_duplicate = False
        for existing in unique_contexts:
            similarity = SequenceMatcher(None, context, existing).ratio()
            if similarity > threshold:
                is_duplicate = True
                break

        if not is_duplicate:
            unique_contexts.append(context)

    return unique_contexts
```

---

## Answer Quality Optimization

### 1. Clear Instructions

**In prompt**:
```
- Answer based ONLY on provided information
- If insufficient, explicitly state limitations
- Cite facts when possible
- Be concise but comprehensive
- Maintain factual accuracy
- Do not speculate or add information not in context
```

### 2. Format Guidance

**Structured output**:
```
Provide answer in this format:

Summary: [1-2 sentence direct answer]

Details: [2-4 sentences with supporting information]

Context: [Optional 1-2 sentences for broader context]
```

### 3. Example-Based Learning

**Few-shot prompting**:
```python
prompt = f"""
Example:

Question: Who is Marie Curie?
Information: Marie Curie was a physicist... [facts]
Answer: Marie Curie was a pioneering physicist and chemist...

---

Question: {query}
Information: {context}
Answer:
"""
```

### 4. Iterative Refinement

**Multi-step generation**:
```python
# Step 1: Generate draft
draft = LLM_client({'query': base_prompt})

# Step 2: Verify against context
verification_prompt = f"""
Draft answer: {draft}
Original information: {context}

Are there any factual errors or unsupported claims in the draft?
If yes, provide corrections.
"""
corrections = LLM_client({'query': verification_prompt})

# Step 3: Final answer
if corrections:
    final = apply_corrections(draft, corrections)
else:
    final = draft
```

---

## Answer Types and Strategies

### Factoid Questions

**Example**: "When did X happen?"

**Strategy**:
- Prioritize semantic units (specific facts)
- Include entities (dates, places)
- Minimal broader context

**Context Focus**:
```
FACTS:
- Event happened on DATE at LOCATION
- Participants included PERSON
```

### Descriptive Questions

**Example**: "What is X?"

**Strategy**:
- Prioritize attributes (comprehensive descriptions)
- Include high-level elements (thematic context)
- Add semantic units (supporting details)

**Context Focus**:
```
THEMES: [high-level element]

DESCRIPTION: [attribute]

FACTS: [semantic units for details]
```

### Analytical Questions

**Example**: "Why did X happen?" or "How does X work?"

**Strategy**:
- Include relationships (causal connections)
- Multiple semantic units (evidence chain)
- Attributes for background

**Context Focus**:
```
RELATIONSHIPS: [causal connections]

FACTS: [supporting evidence]

BACKGROUND: [attributes]
```

### Comparative Questions

**Example**: "What is the difference between X and Y?"

**Strategy**:
- Multiple entities (X, Y)
- Attributes для both
- Relationships showing connections/differences

**Context Focus**:
```
ENTITY X:
- Attribute: [description of X]
- Facts: [...]

ENTITY Y:
- Attribute: [description of Y]
- Facts: [...]

RELATIONSHIPS:
- X related to Y...
```

---

## Best Practices

### 1. Context Quality over Quantity

**Prefer**:
- 5 highly relevant semantic units
- 1-2 comprehensive attributes
- Key relationships

**Over**:
- 20 marginally relevant semantic units
- No structured information

### 2. Structured Context

**Always use sections**:
- Themes
- Entities
- Relationships
- Facts

**Benefits**:
- Easier LLM navigation
- Better answer organization
- Clear source attribution

### 3. Include Metadata

**In context**:
```
FACT: [semantic unit]
Source: Document "renewable_energy_2024.txt"
Confidence: High (mentioned 3 times)
```

**Benefits**:
- Transparency
- Source tracking
- Confidence assessment

### 4. Prompt Engineering

**Iterate and test**:
- Try different prompt formulations
- A/B test with sample queries
- Collect feedback on answer quality

---

## Диагностика Answer Quality

### Metrics

**1. Factual Accuracy**:
```python
# Manual evaluation
correct_facts = count_correct_facts(answer, ground_truth)
total_facts = count_total_facts(answer)
accuracy = correct_facts / total_facts
```

**2. Completeness**:
```python
# Coverage of key information
key_points_covered = check_key_points(answer, expected_points)
completeness = len(key_points_covered) / len(expected_points)
```

**3. Relevance**:
```python
# Relevance to query
relevance_score = cosine_similarity(
    embedding(answer),
    embedding(query)
)
```

**4. Coherence**:
```python
# Manual evaluation (1-5 scale)
# - Logical flow
# - Clarity
# - Organization
```

### Common Issues

**Issue 1: Hallucination**

**Symptoms**: Answer contains facts not in context

**Causes**:
- Weak prompt instructions
- LLM overconfidence
- Insufficient context

**Fixes**:
```python
# Strengthen prompt
"""
IMPORTANT: Only use information explicitly provided in the context above.
Do not add any information from your training data.
If information is missing, say "I don't have information about that."
"""
```

**Issue 2: Incomplete Answers**

**Symptoms**: Key information missing

**Causes**:
- Insufficient retrieval
- Low retrieval quotas
- Poor ranking

**Fixes**:
- Increase `cross_node`, `Enode` quotas
- Improve retrieval (HNSW k, PPR alpha)
- Better query decomposition

**Issue 3: Incoherent Answers**

**Symptoms**: Disjointed, unclear response

**Causes**:
- Unstructured context
- Too much fragmented information
- No thematic framing

**Fixes**:
- Use structured context assembly
- Include high-level elements
- Add attributes для coherence

---

## FAQ

**Q: Как предотвратить hallucinations?**
A: Сильные prompt instructions ("only use provided information"), verification steps, temperature=0 для deterministic output.

**Q: Structured vs unstructured context - что лучше?**
A: Structured почти всегда лучше. Дает LLM clear organization, улучшает answer quality.

**Q: Как обработать queries с недостаточной информацией?**
A: LLM должен явно сказать "I don't have enough information about X". Include это в prompt instructions.

**Q: Нужно ли always включать все типы узлов?**
A: Нет. Include только релевантные для query. Factoid questions могут не нуждаться в high-level elements, etc.

**Q: Как измерить answer quality в production?**
A: Комбинация: user feedback (thumbs up/down), automated metrics (relevance, factuality checks), periodic manual review.

---

## Связанные Документы

**Node Types** (all contribute to answers):
- [Semantic Unit Nodes](./03-semantic-unit-nodes.md) - detailed facts
- [Entity Nodes](./04-entity-nodes.md) - key actors/concepts
- [Relationship Nodes](./05-relationship-nodes.md) - structured connections
- [Attribute Nodes](./06-attribute-nodes.md) - comprehensive descriptions
- [High-Level Element Nodes](./07-high-level-element-nodes.md) - thematic framing

**Process**:
- [Search and Retrieval](./09-search-and-retrieval.md) - provides nodes для context

---

[← Назад: Search and Retrieval](./09-search-and-retrieval.md) | [К обзору ↑](./README.md)
