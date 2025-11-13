# Графовые алгоритмы в NodeRAG: Полный обзор

## Введение

Этот раздел документации описывает **все графовые алгоритмы**, используемые в NodeRAG для построения, анализа и навигации по knowledge graph. Каждый алгоритм играет специфическую роль в архитектуре системы и связан с концептуальными подходами, описанными в [`spec/research/`](../research/).

---

## Обзор алгоритмов

### 1. [HNSW (Hierarchical Navigable Small World)](./hnsw-algorithm.md)

**Назначение**: Аппроксимативный поиск ближайших соседей в embedding space

**Роль**: **Первичные аттракторы** в [звёздной архитектуре](../research/star-pattern-architecture.md)

```
Query → Embedding → HNSW Search → Top-k Nodes (аналоговые аттракторы)
```

**Ключевые характеристики**:
- **Сложность**: O(log N) для k-NN query
- **Библиотека**: hnswlib-noderag==0.8.2 (custom fork)
- **Параметры**: M=64, ef_construction=200, ef=50
- **Recall**: ~0.95-0.98 vs brute force

**Философия**: **Перцептивное осознание** — быстрое, дорефлексивное схватывание релевантных элементов (см. [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), уровень 1).

---

### 2. [Personalized PageRank (PPR)](./personalized-pagerank.md)

**Назначение**: Распространение семантического влияния от аттракторов по графу

**Роль**: **Лучи семантической обработки** в [звёздной архитектуре](../research/star-pattern-architecture.md)

```
Аттракторы → Персонализация → PPR → Weighted Nodes → Context Assembly
```

**Ключевые характеристики**:
- **Сложность**: O(E · T), где T ~10-20 итераций
- **Библиотека**: scipy==1.12.0 (sparse matrices)
- **Параметры**: alpha=0.85, max_iter=100
- **Веса персонализации**: HNSW=1.0, Decomposed=2.0

**Философия**: **Герменевтический круг** — движение от известного (аттракторы) к неизвестному (связанные узлы) и обратно (см. [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)).

---

### 3. [Leiden Community Detection](./leiden-community-detection.md)

**Назначение**: Детекция тематических кластеров в knowledge graph

**Роль**: Идентификация **семантических сообществ** для генерации high-level summaries

```
Knowledge Graph → Leiden Algorithm → Communities → LLM Summarization → High-level Elements
```

**Ключевые характеристики**:
- **Сложность**: O(N log N)
- **Библиотека**: igraph==0.11.8 + leidenalg==0.10.2
- **Метрика**: Modularity Q ~0.4-0.6
- **Результат**: ~500-1000 сообществ для типичного корпуса

**Философия**: **Абстрактное осознание** — категоризация и гештальт-синтез (см. [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), уровень 4).

---

### 4. [K-Core Decomposition & Betweenness Centrality](./k-core-centrality.md)

**Назначение**: Идентификация важных entities для детального анализа

**Роль**: **Селективное внимание** — выбор ~10-15% entities для attribute generation

```
Knowledge Graph → K-Core ∩ Betweenness → Important Entities → LLM Attribute Generation
```

**Ключевые характеристики**:
- **K-Core**: O(E) — плотные подграфы
- **Betweenness**: O(k · E) с sampling (k=10)
- **Библиотека**: networkx==3.4.2
- **Результат**: ~10K-15K important entities из ~100K

**Философия**: **Рефлексивное осознание** — метакогниция о том, что заслуживает глубокого анализа (см. [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md), уровень 3).

---

### 5. [Graph Construction & Concatenation](./graph-construction.md)

**Назначение**: Построение гетерогенного knowledge graph и объединение с HNSW graph

**Роль**: Создание **унифицированного графа** для PPR навигации

```
Text → LLM Decomposition → Nodes/Edges → Knowledge Graph
                                              │
                              ┌───────────────┴───────────────┐
                              │                               │
                        Knowledge Graph                  HNSW Graph (Layer 0)
                              │                               │
                              └───────────────┬───────────────┘
                                              │
                                      [Graph Concatenation]
                                              │
                                      Unified Graph for PPR
```

**Ключевые характеристики**:
- **Типы узлов**: 6 типов (semantic_unit, entity, relationship, attribute, high_level_element, text)
- **Операции**: O(V + E) для построения
- **Библиотека**: networkx==3.4.2
- **Размер**: ~500K узлов, ~2M рёбер (типичный корпус)

**Философия**: **От текста к структуре** — переход от линейной к сетевой репрезентации знаний (см. [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)).

---

## Архитектура взаимодействия алгоритмов

### Индексирование (Indexing Pipeline)

```
1. Document Input
        ↓
2. Text Decomposition (LLM)
        ↓
   Semantic Units + Entities + Relationships
        ↓
3. [GRAPH CONSTRUCTION]
        ↓
   Knowledge Graph (base)
        ↓
4. [K-CORE + BETWEENNESS]
        ↓
   Important Entities (~10-15%)
        ↓
5. LLM Attribute Generation
        ↓
   Knowledge Graph + Attributes
        ↓
6. [LEIDEN ALGORITHM]
        ↓
   Communities
        ↓
7. LLM Community Summarization
        ↓
   High-level Elements
        ↓
8. Embedding Generation
        ↓
   [HNSW INDEXING]
        ↓
   HNSW Index + HNSW Graph (Layer 0)
        ↓
9. [GRAPH CONCATENATION]
        ↓
   Unified Graph (Knowledge + HNSW)
```

**Время**: ~60-120 минут для 10K documents

### Поиск (Query Pipeline)

```
1. User Query
        ↓
2. Query Embedding
        ↓
3. [HNSW SEARCH]
        ↓
   Top-50 Nodes (аналоговые аттракторы, weight=1.0)
        ↓
4. Query Decomposition (LLM)
        ↓
   Entities
        ↓
5. Accurate Search (regex)
        ↓
   Matched Entities (символические аттракторы, weight=2.0)
        ↓
6. [PERSONALIZED PAGERANK]
        ↓
   All Nodes with PPR scores
        ↓
7. Post-processing (типизированная агрегация)
        ↓
   Context Assembly (~65-70 узлов)
        ↓
8. LLM Answer Generation
        ↓
   Final Answer
```

**Время**: ~200-500 ms (без LLM), ~1-2s (с LLM)

---

## Сравнительная таблица алгоритмов

| Алгоритм | Сложность | Библиотека | Роль | Фаза |
|----------|-----------|------------|------|------|
| **HNSW** | O(log N) | hnswlib-noderag | Аналоговые аттракторы | Query |
| **PPR** | O(E · T) | scipy | Лучи семантического влияния | Query |
| **Leiden** | O(N log N) | igraph + leidenalg | Детекция сообществ | Indexing |
| **K-Core** | O(E) | networkx | Плотные подграфы | Indexing |
| **Betweenness** | O(k · E) | networkx | Мосты графа | Indexing |
| **Graph Construction** | O(V + E) | networkx | Построение структуры | Indexing |
| **Graph Concatenation** | O(V + E) | networkx | Объединение графов | Indexing |

---

## Концептуальные связи

### Звёздная архитектура

Графовые алгоритмы реализуют [звёздную топологию](../research/star-pattern-architecture.md):

```
                         Query (центр)
                              │
                      [Query Embedding]
                              │
                      ╔═══════╩═══════╗
                      ║               ║
                [HNSW Search]   [Decomposition]
                      ║               ║
                ╔═════╩═══════════════╩═════╗
                ║         ║         ║        ║
           Attractor₁ Attractor₂ ... Attractor₅₀
            (w=1.0)    (w=2.0)         (w=1.0)
                ║         ║         ║        ║
                ╚═════════╩═════════╩════════╝
                              │
                      [Персонализация]
                              │
                     [PPR Diffusion]
                              │
                  ═══════════════════════════
                  ││││││││││││││││││││││││││  Лучи
                  ═══════════════════════════
                              │
                      [Post-processing]
                              │
                      Context Assembly
```

### Парадигма вопроса как ключа

Графовые алгоритмы реализуют [парадигму вопроса как ключа](../research/question-as-key-paradigm.md):

```
Вопрос (концепция, незнание)
       ↓ [Embedding]
Query Vector (проекция в ℝ¹⁵³⁶)
       ↓ [HNSW]
Аттракторы (проявления концепции)
       ↓ [PPR]
Семантическое поле (расширенный контекст)
       ↓ [LLM Synthesis]
Ответ (новое знание + новое незнание)
```

### Уровни осознания

Графовые алгоритмы соответствуют уровням [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md):

| Уровень | Алгоритм | Когнитивная функция |
|---------|----------|---------------------|
| **0: Предсознательное** | HNSW Search | Интуитивное схватывание |
| **2: Аналитическое** | Graph Construction | Структурирование знания |
| **3: Рефлексивное** | K-Core + Betweenness | Метакогниция (важность) |
| **4: Абстрактное** | Leiden + Summarization | Категоризация, гештальт |
| **System 2** | PPR Expansion | Медленное рассуждение |

---

## Библиотеки и зависимости

### Core Dependencies

| Библиотека | Версия | Алгоритмы | Документация |
|------------|--------|-----------|--------------|
| **hnswlib-noderag** | 0.8.2 | HNSW | [Vector Search](../dependencies/vector-search.md) |
| **networkx** | 3.4.2 | Все графовые | [Graph Operations](../dependencies/graph-operations.md) |
| **scipy** | 1.12.0 | PPR (sparse) | [Graph Operations](../dependencies/graph-operations.md) |
| **igraph** | 0.11.8 | Leiden | [Graph Operations](../dependencies/graph-operations.md) |
| **leidenalg** | 0.10.2 | Leiden | [Graph Operations](../dependencies/graph-operations.md) |
| **numpy** | 1.26.4 | Векторные операции | [Vector Search](../dependencies/vector-search.md) |
| **faiss-cpu** | 1.10.0 | K-means (community clustering) | [Vector Search](../dependencies/vector-search.md) |

### Почему именно эти библиотеки?

1. **hnswlib-noderag**: Custom fork с `export_graph()` для concatenation
2. **networkx**: Pure Python, удобный API, достаточно быстрый для <1M узлов
3. **scipy**: Оптимизированные sparse матрицы (C/Fortran)
4. **igraph + leidenalg**: C core — в 10-100x быстрее NetworkX для community detection
5. **numpy**: BLAS-optimized векторные операции

---

## Производительность

### Сравнение скорости (500K узлов, 2M рёбер)

| Операция | Время | Throughput |
|----------|-------|------------|
| HNSW Query (k=50) | ~5-10 ms | ~100-200 QPS |
| PPR Computation | ~100-200 ms | ~5-10 QPS |
| K-Core Decomposition | ~5-10 s | - |
| Betweenness (k=10) | ~30-60 s | - |
| Leiden Community Detection | ~30-60 s | - |
| Graph Concatenation | ~10-30 s | - |

### Memory Footprint

| Компонент | Память |
|-----------|--------|
| HNSW Index | ~4-6 GB |
| Knowledge Graph | ~1-2 GB |
| HNSW Graph | ~500 MB |
| Unified Graph | ~2-3 GB |
| PPR Computation | ~1 GB |
| **Total Runtime** | ~10-15 GB |

---

## Оптимизация и trade-offs

### 1. HNSW Parameters

**Trade-off**: Recall vs. Speed

| Параметр | Низкий | Средний | Высокий |
|----------|--------|---------|---------|
| M | 32 | 64 | 128 |
| ef_construction | 100 | 200 | 400 |
| ef (search) | 25 | 50 | 100 |
| **Recall** | 0.90 | 0.95 | 0.98 |
| **Speed** | Быстрее | Средне | Медленнее |

**NodeRAG**: M=64, ef_construction=200, ef=50 (баланс)

### 2. PPR Alpha

**Trade-off**: Exploration vs. Exploitation

| Alpha | Exploration | Exploitation | Diversity |
|-------|-------------|--------------|-----------|
| 0.50  | Низкий      | Высокий      | Низкая    |
| 0.85  | Средний     | Средний      | Средняя   |
| 0.95  | Высокий     | Низкий       | Высокая   |

**NodeRAG**: α=0.85 (стандартный PageRank)

### 3. K-Core k Parameter

**Trade-off**: Coverage vs. Selectivity

| k | Coverage | Selectivity | Cost |
|---|----------|-------------|------|
| 10 | Высокий (~20%) | Низкая | Высокая |
| 20 | Средний (~10%) | Средняя | Средняя |
| 30 | Низкий (~5%) | Высокая | Низкая |

**NodeRAG**: k = log(N) · √(avg_degree) (адаптивный)

---

## Расширения и будущие направления

### 1. Dynamic Graph Updates

**Проблема**: Сейчас граф пересоздаётся при каждом обновлении.

**Решение**: Инкрементальные обновления:
- HNSW: поддерживает `add_items()` без пересчёта
- PPR: можно делать локально
- Leiden: только для новых сообществ

### 2. Approximate PPR

Для очень больших графов (>10M узлов):
- Monte Carlo random walks
- Forward push algorithm
- Bidirectional search

### 3. Multi-hop PPR

Итеративный PPR для глубокого поиска:
```
PPR₁ → Top-100 → PPR₂ → Top-100 → ...
```

### 4. Hierarchical Communities

Многоуровневая иерархия сообществ:
```
Level 0: Малые сообщества (10-50 узлов)
Level 1: Средние сообщества (100-500 узлов)
Level 2: Большие сообщества (1000+ узлов)
```

### 5. Temporal Graphs

Добавление временной оси:
```
Node: (id, type, timestamp)
Edge: (u, v, weight, timestamp)
```

PPR с temporal decay:
```
PPR(v, t) с учётом возраста узлов
```

---

## Практические рекомендации

### Для разработчиков

1. **Monitoring**:
   - Логируйте время выполнения каждого алгоритма
   - Отслеживайте размер графа (nodes, edges)
   - Мониторьте HNSW recall vs ground truth

2. **Caching**:
   - Кэшируйте HNSW index (долгое построение)
   - Кэшируйте unified graph (для PPR)
   - Кэшируйте communities (для повторного суммирования)

3. **Incremental Updates**:
   - Используйте `append=True` для Parquet файлов
   - Добавляйте в HNSW без пересоздания
   - Пересчитывайте сообщества только для новых узлов

4. **Error Handling**:
   - HNSW query может вернуть < k results (малый индекс)
   - PPR может не сходиться (disconnected components)
   - Leiden может вернуть 1 сообщество (слишком плотный граф)

### Для исследователей

1. **Hyperparameter Tuning**:
   - Grid search для HNSW parameters (M, ef)
   - Экспериментируйте с PPR alpha ∈ [0.7, 0.95]
   - Настройте k-core k для разных корпусов

2. **Evaluation**:
   - HNSW: Recall@k vs brute force
   - PPR: Precision@k, nDCG@k (с ground truth)
   - Communities: Modularity, NMI (если есть labels)

3. **Ablation Studies**:
   - HNSW vs brute force
   - PPR vs random walk
   - Leiden vs Louvain vs Label Propagation
   - K-Core vs Degree vs PageRank для important nodes

4. **Visualization**:
   - HNSW Layer 0 graph structure
   - PPR score distribution
   - Community size distribution
   - Important nodes в контексте графа

---

## Дополнительные ресурсы

### Документация алгоритмов

- [HNSW Algorithm](./hnsw-algorithm.md)
- [Personalized PageRank](./personalized-pagerank.md)
- [Leiden Community Detection](./leiden-community-detection.md)
- [K-Core and Centrality](./k-core-centrality.md)
- [Graph Construction](./graph-construction.md)

### Концептуальные исследования

- [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md)
- [Звёздная архитектура](../research/star-pattern-architecture.md)
- [LLM Semantic Consciousness](../research/llm-semantic-consciousness.md)

### Зависимости и библиотеки

- [Vector Search Dependencies](../dependencies/vector-search.md)
- [Graph Operations Dependencies](../dependencies/graph-operations.md)
- [LLM and AI Dependencies](../dependencies/llm-and-ai.md)

### Другие разделы spec/

- [Architecture Patterns](../architecture/) — Программные паттерны
- [Prompts](../prompts/) — LLM промпты
- [Transformations](../transform/) — Семантические трансформации
- [Semantic Chunking](../semantic-chunking-strategy.md) — Стратегия chunking

---

## Навигация по документации

### Для новичков

**Рекомендуемый путь обучения**:

1. Начните с [Graph Construction](./graph-construction.md) — понимание структуры
2. Изучите [HNSW Algorithm](./hnsw-algorithm.md) — основа поиска
3. Продолжите с [Personalized PageRank](./personalized-pagerank.md) — расширение контекста
4. Дополните [Leiden Algorithm](./leiden-community-detection.md) — кластеризация
5. Завершите [K-Core and Centrality](./k-core-centrality.md) — важные узлы

### Для ML-инженеров

**Фокус на производительность и оптимизацию**:

1. [HNSW Algorithm](./hnsw-algorithm.md) — параметры, recall, throughput
2. [Personalized PageRank](./personalized-pagerank.md) — sparse matrices, convergence
3. [Graph Construction](./graph-construction.md) — concatenation, unbalance adjust

### Для исследователей

**Фокус на концепции и методологию**:

1. [Звёздная архитектура](../research/star-pattern-architecture.md) — общая картина
2. [Personalized PageRank](./personalized-pagerank.md) — философия диффузии
3. [Парадигма вопроса как ключа](../research/question-as-key-paradigm.md) — эпистемология
4. [Leiden Algorithm](./leiden-community-detection.md) — категоризация и гештальт

---

## Часто задаваемые вопросы (FAQ)

### Q1: Почему используется неориентированный граф?

**A**: Семантические связи обычно **симметричны**:
```
"Dr. Roberts presented at the conference"
→ [DR. ROBERTS] ↔ [CONFERENCE]
```

Если "Dr. Roberts" связан с "Conference", то и "Conference" связан с "Dr. Roberts".

### Q2: Можно ли использовать FAISS вместо HNSW?

**A**: Да, но HNSW имеет преимущества:
- Выше recall (~0.98 vs ~0.95 для FAISS IVF)
- Проще (не требует обучения)
- Custom fork с `export_graph()`

### Q3: Почему PPR, а не другие centrality меры?

**A**: PPR:
- **Персонализируемый** (не все узлы равноценны)
- **Учитывает структуру** (не только embeddings)
- **Балансирует exploration/exploitation** (через α)

### Q4: Можно ли использовать GPU для графовых операций?

**A**:
- **HNSW**: CPU-only (hnswlib не поддерживает GPU)
- **PPR**: Можно использовать cuGraph (NVIDIA RAPIDS), но для <1M узлов CPU достаточно
- **Leiden**: igraph CPU-only

### Q5: Как обрабатываются multi-lingual корпусы?

**A**: OpenAI embeddings **multilingual** (поддерживают 100+ языков). HNSW и PPR работают на embeddings, поэтому язык не имеет значения.

### Q6: Что происходит при пустых HNSW results?

**A**: Если HNSW возвращает < k results (малый корпус), PPR всё равно работает с теми, что есть. Минимум 1 аттрактор достаточен для PPR.

---

## Заключение

Графовые алгоритмы в NodeRAG образуют **целостную экосистему** для семантической навигации:

1. **HNSW**: Быстрое интуитивное схватывание (System 1)
2. **PPR**: Медленное структурное рассуждение (System 2)
3. **Leiden**: Категоризация и абстракция
4. **K-Core + Betweenness**: Метакогниция о важности
5. **Graph Construction**: От текста к структуре

Вместе они реализуют **философию NodeRAG** — комбинация аналогового (embeddings) и символического (graph) для hybrid semantic intelligence.

---

**Последнее обновление**: 2025-11-13

**Версия документации**: 1.0

**Для вопросов и предложений**: См. [GitHub Issues](https://github.com/...)
