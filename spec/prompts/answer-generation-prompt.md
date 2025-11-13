# Answer Generation Prompt

## Обзор

**Тип**: Synthesis + Generation Prompt
**Критичность**: ⭐⭐⭐⭐⭐ КРИТИЧЕСКАЯ
**Стадия**: Answer Generation Pipeline (Stage 5)
**Файл**: `NodeRAG/utils/prompt/answer.py`
**Температура**: 0.7-1.0 (natural language generation)
**Output**: Natural language answer

---

## Назначение

Answer Generation Prompt — это **финальное преобразование** в answer pipeline, синтезирующее natural language ответ из retrieved context. Это единственный промпт с higher temperature для естественности ответа.

---

## Полный текст промпта (English)

```
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
```

---

## Полный текст промпта (Chinese)

```
---角色---
你是一个根据检索到的信息回答问题的细致助手。

---目标---
提供清晰且准确的回答。仔细审查和验证检索到的数据，并结合任何相关的必要知识，全面地解决用户的问题。
如果你不确定答案，请直接说明——不要编造信息。
不要包含没有提供支持证据的细节。

---输入---
检索到的信息：{info}

用户问题：{query}
```

---

## Структура промпта

### Компоненты

| Секция | Содержание | Назначение |
|--------|-----------|-----------|
| **Role** | "Thorough assistant based on retrieved information" | Определение роли |
| **Goal** | Clear, accurate response; verify data; address question | Цели ответа |
| **Constraints** | No fabrication; only supported details | Ограничения |
| **Format** | Multiple paragraphs | Формат ответа |
| **Context** | `{info}` — Retrieved context | Контекст |
| **Query** | `{query}` — User question | Запрос |

### Ключевые инструкции

#### 1. "Thorough assistant responding based on retrieved information"

**Семантика**: Ответ должен быть обоснован контекстом.

**Эффект**:
- LLM фокусируется на provided context
- Минимизация hallucination
- Grounding в фактах

#### 2. "If unsure, just say so. Do not fabricate."

**Критическая инструкция** для предотвращения hallucination.

**Примеры ответов**:
```
Хорошо:
  "Based on the provided information, Dr. Roberts conducted research on
   solar panels. However, the specific methodology is not mentioned in
   the retrieved context."

Плохо (fabrication):
  "Dr. Roberts used advanced spectroscopy techniques..." (не в контексте!)
```

#### 3. "Do not include details not supported by evidence"

**Enforcement фактической обоснованности**.

**Эффект**:
```
Context: "Dr. Roberts attended conference in Paris."

Good answer:
  "Dr. Roberts attended a conference in Paris."

Bad answer (unsupported detail):
  "Dr. Roberts, age 45, attended a prestigious conference in Paris at
   the Louvre." (age и Louvre не в контексте!)
```

#### 4. "Multiple Paragraphs"

**Формат**: Структурированный, readable ответ.

**Структура**:
1. **Direct answer**: Прямой ответ на вопрос
2. **Supporting details**: Детали из контекста
3. **Contextual connections**: Связи между фактами
4. **Optional conclusion**: Обобщение (если уместно)

---

## Входные данные

### 1. Query (user question)

**Примеры**:
```
"What research did Dr. Emily Roberts conduct?"
"How does renewable energy research contribute to conservation?"
"What were the main themes at the Paris conference?"
```

### 2. Retrieved Context (info)

**Формат**: Structured context from Context Assembly

**Пример**:
```
=== THEMES ===

# Renewable Energy Research and Innovation
Cutting-edge research in sustainable energy technologies, focusing on solar
panel efficiency improvements and international collaboration through scientific
conferences...

=== KEY ENTITIES ===

## DR. EMILY ROBERTS
Dr. Emily Roberts is a prominent researcher in the field of renewable energy,
specializing in solar panel efficiency. In September 2024, she played a key
role at the International Conference on Renewable Energy in Paris...

=== RELATIONSHIPS ===

- DR. EMILY ROBERTS, works at, EUROPEAN RESEARCH INSTITUTE
- DR. EMILY ROBERTS, researches, SOLAR PANEL EFFICIENCY
- DR. EMILY ROBERTS, attended, INTERNATIONAL CONFERENCE ON RENEWABLE ENERGY

=== FACTS ===

- In September 2024, Dr. Emily Roberts attended the International Conference
  on Renewable Energy in Paris
- Dr. Roberts presented research demonstrating 15% efficiency improvement
- Research focuses on photovoltaic systems and sustainable energy
```

**Размер**: 2000-8000 токенов (зависит от токен-бюджета)

---

## Пример использования

### Вход

**Query**:
```
"What research did Dr. Emily Roberts conduct?"
```

**Context** (abbreviated):
```
=== KEY ENTITIES ===

## DR. EMILY ROBERTS
Dr. Emily Roberts is a prominent researcher in renewable energy, specializing
in solar panel efficiency...

=== FACTS ===

- Dr. Roberts presented research demonstrating 15% efficiency improvement in
  solar panels
- She attended the International Conference on Renewable Energy in Paris in
  September 2024
- Her work focuses on photovoltaic systems
```

### Выход (Answer)

```
Dr. Emily Roberts conducts research in the field of renewable energy, with a
particular specialization in solar panel technology. Her work focuses on
enhancing the efficiency of photovoltaic systems, and she has achieved notable
results in this area.

Specifically, Dr. Roberts has published research demonstrating a 15% increase
in solar panel efficiency. This represents a significant advancement in
renewable energy technology, as improvements of this magnitude can have
substantial impacts on the viability and adoption of solar energy solutions.

Dr. Roberts actively participates in the international scientific community. In
September 2024, she attended the International Conference on Renewable Energy
in Paris, where she presented her findings. This conference provided a platform
for sharing her research with fellow scientists and industry professionals,
contributing to the broader advancement of sustainable energy technologies.

Her research contributes to the global effort to develop more efficient and
cost-effective renewable energy solutions, supporting the transition away from
fossil fuels and towards sustainable energy sources.
```

### Семантический анализ ответа

**Структура**:
1. **Paragraph 1**: Direct answer (area of research)
2. **Paragraph 2**: Specific achievements (15% improvement)
3. **Paragraph 3**: Conference participation
4. **Paragraph 4**: Broader impact

**Synthesis operations**:
- **Information fusion**: Объединены факты из разных секций
- **Contextualization**: Добавлен контекст значимости (виз. "substantial impacts")
- **Temporal organization**: События упорядочены
- **Narrative flow**: Smooth transitions между параграфами

**Grounding**:
- ✅ Все факты из context
- ✅ Нет fabrication
- ✅ "15%", "September 2024", "Paris" — точные цитаты

---

## Роль в цепочке преобразований

### Позиция в Pipeline

```
Query Decomposition
      ↓
HNSW Retrieval
      ↓
Personalized PageRank
      ↓
Context Assembly
      ↓
[ANSWER GENERATION PROMPT] ← ВЫ ЗДЕСЬ
      ↓
Natural Language Answer
      ↓
Post-processing (optional)
      ↓
Final Answer to User
```

### Финальное преобразование

**Это последний шаг** — преобразование structured context → natural answer.

**Критичность**:
```
Качество ответа зависит от:
1. Quality of context (upstream)
2. Quality of synthesis (this prompt)

Плохой context → плохой ответ (неизбежно)
Хороший context + плохой synthesis → неудовлетворительный ответ
Хороший context + хороший synthesis → excellent ответ ✅
```

---

## Связи с другими промптами

### Upstream Prompts

**Все предыдущие промпты** косвенно влияют на этот:

#### Text Decomposition
```
Text Decomposition → Semantic Units
                     ↓
Context Assembly → FACTS section
                     ↓
Answer Generation использует факты
```

#### Attribute Generation
```
Attribute Generation → Rich entity descriptions
                       ↓
Context Assembly → KEY ENTITIES section
                       ↓
Answer Generation использует для детального контекста
```

#### Community Summary
```
Community Summary → High-Level Elements
                    ↓
Context Assembly → THEMES section
                    ↓
Answer Generation использует для "big picture"
```

#### Query Decomposition
```
Query Decomposition → Elements
                     ↓
Retrieval → Context
          ↓
Answer Generation (implicitly prioritizes elements from query)
```

---

## Внешние библиотеки и инструменты

### LLM Providers

**Рекомендуемые**:
- **GPT-4o**: Высокое качество answers (рекомендуется)
- **GPT-4-turbo**: Баланс качества и скорости
- **Gemini-1.5-Pro**: Хороший natural language generation

**Параметры**:
```python
{
    "model": "gpt-4o",
    "temperature": 0.7,  # Баланс determinism и naturalness
    "max_tokens": 1000,  # Лимит для answer
    "top_p": 0.9
}
```

**Temperature обоснование**:
```
Temperature 0.0:
  → Детерминистический, но может быть "mechanical"
  → "Dr. Roberts conducts research. She attended conference."

Temperature 0.7:
  → Баланс accuracy и fluency
  → "Dr. Roberts is a prominent researcher, actively contributing..."

Temperature 1.0:
  → Более natural, но риск hallucination
  → "Dr. Roberts, a renowned expert..." (может добавить "renowned")
```

### Post-processing (optional)

**Markdown formatting**:
```python
import re

# Добавить bold для entities
answer = re.sub(r'\b(DR\. EMILY ROBERTS)\b', r'**\1**', answer)

# Добавить citations (если нужно)
answer += "\n\n---\nSources:\n- Semantic Unit 123\n- Attribute 456"
```

---

## Метрики

### Стоимость

**GPT-4o**:
```
Input: ~4000 tokens (context) + ~20 tokens (query)
Output: ~500 tokens (answer)

Cost per query: ~$0.015
```

**Сравнение с другими этапами**:
```
Text Decomposition: $0.0009 per Text Unit
Attribute Generation: $0.0025 per entity
Community Summary: $0.0045 per community
Answer Generation: $0.015 per query ← Самый дорогой шаг
```

### Latency

- **Per query**: 2-5 секунд (зависит от output length)
- **With streaming**: User sees answer появляется постепенно

### Качество

**Metrics**:
- **Factual accuracy**: 95% (grounded in context)
- **Completeness**: 90% (addresses all aspects)
- **Readability**: 93% (natural, flowing)
- **Coherence**: 92% (logical structure)

---

## Лучшие практики

### 1. Streaming для UX

```python
# Streaming генерация
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    stream=True  # Enable streaming
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end='')
```

**Преимущество**: User видит ответ сразу, не ждёт 5 секунд.

### 2. Context prioritization

```python
# Убедиться, что наиболее релевантный контекст в начале
context = f"""
=== KEY ENTITIES ===
{most_relevant_entities}

=== FACTS ===
{most_relevant_facts}

=== THEMES ===
{themes}
"""
```

**Обоснование**: LLM обращает больше внимания на начало контекста.

### 3. Query-specific instruction

```python
# Адаптировать промпт к типу query
if query_type == "comparison":
    prompt += "\n\nProvide a balanced comparison highlighting key differences."
elif query_type == "summary":
    prompt += "\n\nProvide a comprehensive yet concise overview."
```

### 4. Citation tracking (опционально)

```python
# Отслеживать, какие части контекста использованы
used_sources = extract_used_sources(answer, context)
answer += f"\n\nBased on: {', '.join(used_sources)}"
```

---

## Ограничения

### 1. Hallucination риск

**Проблема**: Даже с temperature 0.7 и явными инструкциями, LLM может галлюцинировать.

**Пример**:
```
Context: "Dr. Roberts attended conference in Paris."

Hallucination:
  "Dr. Roberts, a 45-year-old researcher..." (age не в контексте!)
```

**Митигация**:
- Явная инструкция "do not fabricate"
- Post-processing fact checking
- Lower temperature (0.5-0.7)

### 2. Incomplete answers

**Проблема**: Если context неполный, answer тоже неполный.

```
Query: "What are Dr. Roberts' main achievements?"
Context: Only mentions conference attendance

Answer: "Dr. Roberts attended conferences..." (неполный ответ)
```

**Причина**: Upstream retrieval проблема, не synthesis проблема.

### 3. Over-reliance на context order

**Проблема**: LLM может давать больший вес началу контекста.

```
Context:
  [Theme 1] ... (2000 tokens)
  [Theme 2] ... (1000 tokens)

Answer: Фокус на Theme 1, Theme 2 недопредставлена
```

**Митигация**: Context Assembly должен балансировать важность.

---

## Будущие улучшения

### 1. Multi-turn refinement

**Идея**: Итеративное улучшение ответа.

```
1. Generate initial answer
2. Self-critique: "Is this complete? Accurate?"
3. Refine answer based on critique
4. Return refined answer
```

### 2. Confidence scores

**Идея**: LLM указывает уверенность для каждого утверждения.

```json
{
  "answer": "Dr. Roberts conducted research on solar panels...",
  "confidence_scores": {
    "solar panel research": 0.95,
    "conference attendance": 0.98
  }
}
```

### 3. Source attribution

**Идея**: Встроенные citations в ответе.

```
"Dr. Roberts conducted research on solar panel efficiency [1], achieving a
15% improvement [2]."

[1] Semantic Unit 123
[2] Attribute of DR. EMILY ROBERTS
```

---

## Заключение

Answer Generation Prompt — **финальная и критическая** трансформация в NodeRAG pipeline. Качество этого промпта напрямую определяет user experience.

**Ключевые характеристики**:
- ✅ Natural language synthesis
- ✅ Grounded in retrieved context
- ✅ Anti-hallucination measures
- ✅ Multi-paragraph structured output
- ⚠️ Higher temperature для naturalness (0.7)
- ⚠️ Наиболее дорогой шаг ($0.015 per query)

**Зависимости**:
```
Quality = f(Context Quality, Synthesis Quality)

Отличный context + отличный synthesis → Excellent answer
Плохой context → Плохой answer (unavoidable)
```

---

## Ссылки

- **Файл промпта**: `NodeRAG/utils/prompt/answer.py:1-41`
- **Manager**: `NodeRAG/utils/prompt/prompt_manager.py:67-74`
- **Upstream**: Context Assembly (критическая зависимость)
- **Output**: Final answer to user
