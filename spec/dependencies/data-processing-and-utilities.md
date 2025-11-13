# Обработка данных и утилиты

## Обзор

Эта категория включает библиотеки для обработки структурированных данных, асинхронных операций, конфигурации и различных утилит.

---

## Часть 1: Обработка данных

### 1. Pandas

**Пакет**: `pandas==2.2.3`

**Официальный сайт**: https://pandas.pydata.org/

#### Назначение в NodeRAG

Pandas используется для **табличного представления** узлов графа и metadata.

#### Применение

**Файл**: `NodeRAG/build/pipeline/graph_pipeline.py:194-254`

```python
import pandas as pd

class Graph_pipeline:
    def save_semantic_units(self):
        """Save semantic units to Parquet"""
        semantic_units = []
        for semantic_unit in self.semantic_units:
            semantic_units.append({
                'hash_id': semantic_unit.hash_id,
                'human_readable_id': semantic_unit.human_readable_id,
                'type': 'semantic_unit',
                'context': semantic_unit.raw_context,
                'text_hash_id': semantic_unit.text_hash_id,
                'weight': self.G.nodes[semantic_unit.hash_id]['weight'],
                'embedding': None,
                'insert': None
            })

        # Assertions
        G_semantic_units = [node for node in self.G.nodes
                            if self.G.nodes[node]['type'] == 'semantic_unit']
        assert len(semantic_units) == len(G_semantic_units)

        # Convert to DataFrame and save
        df = pd.DataFrame(semantic_units)
        storage(semantic_units).save_parquet(self.config.semantic_units_path, append=True)

    def save_entities(self):
        """Save entities to Parquet"""
        entities = []
        for entity in self.entities:
            entities.append({
                'hash_id': entity.hash_id,
                'human_readable_id': entity.human_readable_id,
                'type': 'entity',
                'context': entity.raw_context,
                'text_hash_id': entity.text_hash_id,
                'weight': self.G.nodes[entity.hash_id]['weight']
            })

        df = pd.DataFrame(entities)
        storage(entities).save_parquet(self.config.entities_path, append=True)
```

#### Операции Pandas в NodeRAG

| Операция | Использование | Файл |
|----------|---------------|------|
| `pd.DataFrame()` | Create table from list of dicts | All save operations |
| `pd.read_parquet()` | Load table from Parquet | Mapper initialization |
| `df.iterrows()` | Iterate over rows | Text pipeline |
| `df['column']` | Column access | Mapper queries |
| `df[condition]` | Boolean indexing | Filtering |
| `pd.concat()` | Concatenate DataFrames | Append mode |

#### Структура данных

**Semantic Units**:
```python
{
    'hash_id': str,           # SHA256 hash
    'human_readable_id': str, # SU-00001
    'type': str,              # 'semantic_unit'
    'context': str,           # Actual text
    'text_hash_id': str,      # Parent text unit
    'weight': int,            # Frequency
    'embedding': str/None,    # 'HNSW' or None
    'insert': bool/None       # HNSW insertion status
}
```

**Entities**:
```python
{
    'hash_id': str,
    'human_readable_id': str,  # ENT-00001
    'type': str,               # 'entity'
    'context': str,            # "DR. EMILY ROBERTS"
    'text_hash_id': str,
    'weight': int
}
```

---

### 2. PyArrow

**Пакет**: `pyarrow==19.0.1`

**Официальный сайт**: https://arrow.apache.org/docs/python/

#### Назначение в NodeRAG

PyArrow обеспечивает **эффективное чтение/запись** Parquet files.

#### Применение

**Файл**: `NodeRAG/storage/storage.py` (referenced)

```python
import pandas as pd

class storage:
    def save_parquet(self, path: str, append: bool = False):
        """Save data as Parquet file"""
        df = pd.DataFrame(self.data)

        if append and os.path.exists(path):
            # Load existing
            existing = pd.read_parquet(path)  # Uses PyArrow
            df = pd.concat([existing, df], ignore_index=True)

        # Save with PyArrow engine
        df.to_parquet(path, index=False, engine='pyarrow')

    @staticmethod
    def load_parquet(path: str) -> pd.DataFrame:
        """Load Parquet file"""
        return pd.read_parquet(path, engine='pyarrow')
```

#### Parquet Format Advantages

| Advantage | Значение для NodeRAG |
|-----------|----------------------|
| **Columnar storage** | Fast column-wise queries |
| **Compression** | ~5-10x smaller than CSV |
| **Schema preservation** | Types auto-detected |
| **Fast I/O** | 10-100x faster than CSV |
| **Append support** | Incremental updates |

**Пример размеров**:
```
100k semantic units:
- CSV: ~50 MB
- JSON: ~40 MB
- Parquet: ~5-8 MB (with compression)
```

#### Используемые features

| Feature | Usage |
|---------|-------|
| `engine='pyarrow'` | Backend for pandas |
| Snappy compression | Default compression |
| Schema inference | Auto type detection |
| Column pruning | Read only needed columns |

---

### 3. Pickle (через storage)

**Пакет**: Built-in Python

**Официальный сайт**: https://docs.python.org/3/library/pickle.html

#### Назначение в NodeRAG

Pickle используется для **сериализации** сложных объектов (NetworkX graphs).

#### Применение

**Файл**: `NodeRAG/storage/storage.py` (referenced)

```python
import pickle

class storage:
    def save_pickle(self, path: str):
        """Save arbitrary Python object"""
        with open(path, 'wb') as f:
            pickle.dump(self.data, f, protocol=pickle.HIGHEST_PROTOCOL)

    @staticmethod
    def load_pickle(path: str):
        """Load arbitrary Python object"""
        with open(path, 'rb') as f:
            return pickle.load(f)
```

#### Используется для

| Object Type | File | Size (100k nodes) |
|-------------|------|-------------------|
| NetworkX Graph | `graph.pkl` | ~50-100 MB |
| HNSW Index (converted to NX) | `hnsw_graph.pkl` | ~30-50 MB |

#### Pickle vs. Parquet

| Аспект | Pickle | Parquet |
|--------|--------|---------|
| **Type support** | Any Python object | Tabular data only |
| **Cross-language** | ❌ Python-only | ✅ Multiple languages |
| **Speed** | Fast | Very fast |
| **Compression** | Moderate | Excellent |
| **Usage** | Graphs, indices | Node/edge tables |

---

### 4. YAML Configuration

**Пакеты**: `pyyaml==6.0.2`, `ruamel-yaml>=0.18.10`

**Официальный сайт**: https://pyyaml.org/

#### Назначение в NodeRAG

YAML используется для **конфигурационных файлов**.

#### Применение

**Файл**: `NodeRAG/config/*.yaml` (referenced)

```yaml
# config.yaml
llm:
  service_provider: "openai"
  model_name: "gpt-4o"
  embedding_model_name: "text-embedding-3-small"
  temperature: 0.0
  max_tokens: 10000
  rate_limit: 50

graph:
  space: "cosine"
  dim: 1536
  _ef: 200
  _m: 64

search:
  HNSW_results: 50
  ppr_alpha: 0.85
  ppr_max_iter: 100
  Enode: 10      # Entity quota
  Rnode: 20      # Relationship quota
  Hnode: 5       # High-level element quota
  cross_node: 50 # Other nodes quota
```

**Загрузка**:

```python
import yaml

class NodeConfig:
    def __init__(self, config_path: str):
        with open(config_path, 'r') as f:
            self.config = yaml.safe_load(f)

    def get(self, key, default=None):
        return self.config.get(key, default)
```

#### PyYAML vs. ruamel.yaml

| Аспект | PyYAML | ruamel.yaml |
|--------|--------|-------------|
| **Round-trip** | ❌ Loses formatting | ✅ Preserves formatting |
| **Comments** | ❌ Lost | ✅ Preserved |
| **Speed** | Faster | Slightly slower |
| **Usage** | Reading config | Writing/updating config |

---

## Часть 2: Асинхронные операции

### 1. aiohttp

**Пакет**: `aiohttp==3.11.13`

**Официальный сайт**: https://docs.aiohttp.org/

#### Назначение в NodeRAG

aiohttp обеспечивает **асинхронные HTTP запросы**.

#### Зависимости

| Пакет | Назначение |
|-------|-----------|
| `aiohappyeyeballs==2.6.1` | Fast DNS resolution |
| `aiosignal==1.3.2` | Signal support |
| `async-timeout==5.0.1` | Timeout handling |
| `frozenlist==1.5.0` | Immutable lists |
| `multidict==6.1.0` | Multi-value dicts |
| `yarl==1.18.3` | URL parsing |

#### Применение

**Примечание**: aiohttp **не используется напрямую** в NodeRAG core.

**Зависимость через**: Streamlit, возможно future async web API.

---

### 2. anyio

**Пакет**: `anyio==4.8.0`

**Официальный сайт**: https://anyio.readthedocs.io/

#### Назначение

Anyio — это **абстракция** над asyncio/trio, используемая OpenAI SDK.

#### Применение

**Внутри OpenAI SDK**:

```python
# OpenAI SDK использует anyio для async operations
import anyio

async def _create_async():
    async with anyio.create_task_group() as tg:
        tg.start_soon(task1)
        tg.start_soon(task2)
```

**NodeRAG использует**: Опосредованно через OpenAI SDK.

---

## Часть 3: Утилиты

### 1. tqdm

**Пакет**: `tqdm==4.67.1`

**Официальный сайт**: https://github.com/tqdm/tqdm

#### Назначение в NodeRAG

tqdm обеспечивает **progress bars** для длительных операций.

#### Применение

**Файл**: `NodeRAG/logging/tracker.py` (referenced)

```python
from tqdm import tqdm

class ProgressTracker:
    def __init__(self):
        self.pbar = None

    def set(self, total: int, desc: str):
        """Initialize progress bar"""
        self.pbar = tqdm(total=total, desc=desc)

    def update(self, n: int = 1):
        """Update progress"""
        if self.pbar:
            self.pbar.update(n)

    def close(self):
        """Close progress bar"""
        if self.pbar:
            self.pbar.close()
            self.pbar = None
```

**Использование**:

```python
# Text decomposition
self.config.tracker.set(len(self.texts), 'Text Decomposition')
for text in self.texts:
    await process(text)
    self.config.tracker.update()
self.config.tracker.close()
```

**Output**:
```
Text Decomposition: 100%|████████████| 1000/1000 [01:23<00:00, 12.05it/s]
```

---

### 2. Rich

**Пакет**: `rich==13.9.4`

**Официальный сайт**: https://github.com/Textualize/rich

#### Назначение в NodeRAG

Rich обеспечивает **красивый terminal output** с форматированием.

#### Применение

**Файл**: `NodeRAG/config/config.py` (referenced)

```python
from rich.console import Console

class NodeConfig:
    def __init__(self):
        self.console = Console()

    def print_success(self, message: str):
        self.console.print(f'[green]{message}[/green]')

    def print_error(self, message: str):
        self.console.print(f'[red]{message}[/red]')

    def print_warning(self, message: str):
        self.console.print(f'[yellow]{message}[/yellow]')
```

**Примеры**:

```python
# Success message
console.print('[green]Graph stored[/green]')

# Error message
console.print('[red]LLM Error Detected, There are 5 errors[/red]')

# Warning
console.print(f'[yellow]Generating HNSW graph for {len(unHNSW)} nodes[/yellow]')

# Bold
console.print('[bold green]HNSW graph saved[/bold green]')
```

#### Зависимости

| Пакет | Назначение |
|-------|-----------|
| `markdown-it-py==3.0.0` | Markdown rendering |
| `pygments==2.19.1` | Syntax highlighting |
| `mdurl==0.1.2` | URL parsing for markdown |

---

### 3. Requests

**Пакет**: `requests==2.32.3`

**Официальный сайт**: https://requests.readthedocs.io/

#### Назначение в NodeRAG

Requests используется для **синхронных HTTP запросов**.

#### Применение

**Используется в**:
- tiktoken (download encoding files)
- Streamlit (resource fetching)

**Не используется напрямую** в NodeRAG core.

#### Зависимости

| Пакет | Назначение |
|-------|-----------|
| `urllib3==2.3.0` | HTTP library |
| `certifi==2025.1.31` | SSL certificates |
| `charset-normalizer==3.4.1` | Character encoding detection |
| `idna==3.10` | Internationalized domain names |

---

### 4. Click

**Пакет**: `click==8.1.8`

**Официальный сайт**: https://click.palletsprojects.com/

#### Назначение

Click — это библиотека для **CLI interfaces**.

#### Применение

**Используется в**: Flask (dependency)

**Потенциальное использование**:

```python
import click

@click.command()
@click.option('--config', default='config.yaml', help='Config file path')
@click.option('--mode', type=click.Choice(['index', 'search']), help='Operation mode')
def main(config, mode):
    """NodeRAG CLI"""
    if mode == 'index':
        run_indexing(config)
    elif mode == 'search':
        run_search(config)

if __name__ == '__main__':
    main()
```

---

### 5. Jinja2

**Пакет**: `jinja2==3.1.6`

**Официальный сайт**: https://jinja.palletsprojects.com/

#### Назначение

Jinja2 — это **template engine**.

#### Применение в NodeRAG

**Промпт форматирование**:

**Файл**: `NodeRAG/utils/prompt/text_decomposition.py` (referenced)

```python
text_decomposition_prompt = """
Goal: Given a text, segment it into multiple semantic units.

...

Text:{text}
"""

# Usage
prompt = text_decomposition_prompt.format(text=chunk_text)
```

**Альтернативный подход с Jinja2**:

```python
from jinja2 import Template

template = Template("""
Goal: Given a text, segment it into multiple semantic units.

Text:{{ text }}
""")

prompt = template.render(text=chunk_text)
```

**В NodeRAG**: Используется **Python `.format()`**, но Jinja2 доступен как dependency (через Flask, PyVis).

---

### 6. Tenacity

**Пакет**: `tenacity==9.0.0`

**Официальный сайт**: https://github.com/jd/tenacity

#### Назначение

Tenacity — это **advanced retry library**.

#### Сравнение с Backoff

| Аспект | backoff | tenacity |
|--------|---------|----------|
| **API style** | Decorator | Decorator |
| **Features** | Simple | Advanced (conditions, callbacks) |
| **Usage in NodeRAG** | LLM retries | Streamlit (dependency) |

**NodeRAG использует**: backoff (проще и достаточно).

---

## Часть 4: Визуализация и UI

### 1. Streamlit

**Пакет**: `streamlit==1.43.2`

**Официальный сайт**: https://streamlit.io/

#### Назначение в NodeRAG

Streamlit обеспечивает **web UI** для демонстрации и взаимодействия.

#### Зависимости (большое дерево)

| Category | Packages |
|----------|----------|
| **Data** | pandas, pyarrow, numpy |
| **Visualization** | altair, pydeck |
| **Web** | tornado, watchdog |
| **Git** | gitpython |
| **Utils** | cachetools, click, protobuf, toml |

#### Потенциальное применение

**Файл**: `app.py` (example, not in repo)

```python
import streamlit as st
from NodeRAG import NodeSearch

st.title("NodeRAG Q&A System")

# Configuration
config = NodeConfig('config.yaml')
searcher = NodeSearch(config)

# User input
query = st.text_input("Enter your question:")

if st.button("Search"):
    with st.spinner("Searching..."):
        # Search
        answer = searcher.answer(query)

        # Display
        st.subheader("Answer:")
        st.write(answer.response)

        st.subheader("Retrieved Context:")
        for content, type in answer.retrieval.retrieved_list:
            st.markdown(f"**[{type}]** {content}")
```

---

### 2. PyVis

**Пакет**: `pyvis==0.3.2`

**Официальный сайт**: https://pyvis.readthedocs.io/

#### Назначение в NodeRAG

PyVis — это **interactive graph visualization**.

#### Потенциальное применение

```python
from pyvis.network import Network
import networkx as nx

def visualize_graph(G: nx.Graph, output_file: str = 'graph.html'):
    """Visualize NetworkX graph interactively"""
    net = Network(height='750px', width='100%', notebook=False)

    # Add nodes
    for node in G.nodes():
        node_type = G.nodes[node].get('type', 'unknown')
        color = {
            'entity': '#ff9999',
            'semantic_unit': '#99ccff',
            'relationship': '#99ff99',
            'attribute': '#ffcc99',
            'high_level_element': '#cc99ff'
        }.get(node_type, '#cccccc')

        net.add_node(
            node,
            label=G.nodes[node].get('context', node)[:50],
            title=G.nodes[node].get('context', node),
            color=color,
            size=G.nodes[node].get('weight', 1) * 10
        )

    # Add edges
    for u, v in G.edges():
        weight = G[u][v].get('weight', 1)
        net.add_edge(u, v, value=weight)

    # Generate
    net.show(output_file)
```

---

## Часть 5: Типизация и разработка

### 1. IPython

**Пакет**: `ipython==8.34.0`

**Официальный сайт**: https://ipython.org/

#### Назначение

IPython обеспечивает **enhanced REPL** для development.

#### Применение

**Jupyter notebooks**, **interactive debugging**, **development workflow**.

#### Зависимости

| Пакет | Назначение |
|-------|-----------|
| `jedi==0.19.2` | Code completion |
| `decorator==5.2.1` | Decorators |
| `prompt-toolkit==3.0.50` | Terminal interface |
| `pygments==2.19.1` | Syntax highlighting |
| `matplotlib-inline==0.1.7` | Inline plots |
| `traitlets==5.14.3` | Configuration system |

---

### 2. Typing Extensions

**Пакет**: `typing-extensions==4.12.2`

**Официальный сайт**: https://github.com/python/typing_extensions

#### Назначение

Backport новых typing features для Python 3.10+.

#### Примеры

```python
from typing_extensions import TypeAlias, Self

# Type aliases
NodeID: TypeAlias = str
Embedding: TypeAlias = list[float]

# Self type (Python 3.11+)
class Component:
    def copy(self) -> Self:
        return copy.deepcopy(self)
```

---

## Итоговая архитектура зависимостей

```
┌─────────────────────────────────────────────────────────┐
│                   NodeRAG System                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │   LLM    │  │  Graph   │  │  Vector  │            │
│  │ OpenAI   │  │ NetworkX │  │  HNSW    │            │
│  │ Gemini   │  │ igraph   │  │  FAISS   │            │
│  │ Pydantic │  │ Leiden   │  │  NumPy   │            │
│  │ tiktoken │  │ SciPy    │  │          │            │
│  └──────────┘  └──────────┘  └──────────┘            │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │   Data   │  │  Async   │  │  Utils   │            │
│  │  Pandas  │  │ aiohttp  │  │  tqdm    │            │
│  │ PyArrow  │  │  anyio   │  │  Rich    │            │
│  │   YAML   │  │          │  │ backoff  │            │
│  │  Pickle  │  │          │  │ requests │            │
│  └──────────┘  └──────────┘  └──────────┘            │
│                                                         │
│  ┌──────────┐  ┌──────────┐                           │
│  │    UI    │  │   Dev    │                           │
│  │Streamlit │  │ IPython  │                           │
│  │  PyVis   │  │ typing-  │                           │
│  │  Flask   │  │extensions│                           │
│  └──────────┘  └──────────┘                           │
└─────────────────────────────────────────────────────────┘
```

### Dependency Counts

| Category | Packages | Core/Transitive |
|----------|----------|-----------------|
| **LLM & AI** | 8 | 6 core + 2 transitive |
| **Graph** | 4 | 4 core |
| **Vector Search** | 3 | 3 core |
| **Data Processing** | 4 | 4 core |
| **Async** | 7 | 1 core + 6 transitive |
| **Utilities** | 10 | 5 core + 5 transitive |
| **UI** | 15 | 3 core + 12 transitive |
| **Dev Tools** | 10 | 0 core + 10 transitive |
| **Total** | ~60 | ~25 core + ~35 transitive |

### Core Dependencies (Direct)

Эти пакеты **непосредственно используются** в NodeRAG code:

1. `openai` — LLM API
2. `pydantic` — Structured outputs
3. `tiktoken` — Tokenization
4. `networkx` — Graph operations
5. `igraph` + `leidenalg` — Community detection
6. `scipy` — Sparse matrices (PPR)
7. `hnswlib-noderag` — Vector search
8. `faiss-cpu` — K-means clustering
9. `numpy` — Numerical operations
10. `pandas` — Data tables
11. `pyarrow` — Parquet I/O
12. `pyyaml` — Configuration
13. `backoff` — Retry logic
14. `tqdm` — Progress bars
15. `rich` — Terminal output
16. `requests` — HTTP (transitive)
17. `sortedcontainers` — Sorted dicts
18. `torch` — (если используется)
19. `streamlit` — Web UI (optional)
20. `pyvis` — Graph visualization (optional)
21. `flask` — Web server (optional)

### Установка

**Full installation**:
```bash
pip install -r requirements.txt
```

**Minimal installation** (без UI):
```bash
pip install openai pydantic tiktoken networkx igraph leidenalg scipy \
    hnswlib-noderag faiss-cpu numpy pandas pyarrow pyyaml \
    backoff tqdm rich sortedcontainers
```

**Development installation**:
```bash
pip install -r requirements.txt
pip install ipython jupyter pytest
```
