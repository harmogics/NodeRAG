# NodeRAG Dependencies Overview

This directory contains comprehensive documentation of all third-party packages, libraries, and dependencies used in the NodeRAG system, organized by their roles in information processing.

## 📚 Documentation Structure

The dependency documentation is organized into four main categories based on information processing aspects:

### 1. [LLM and AI Dependencies](./llm-and-ai.md)
Core dependencies for language model integration, structured output, and tokenization:
- **OpenAI SDK** - Primary LLM integration (GPT-4o, text-embedding-3-small)
- **Pydantic** - Structured output validation and schema enforcement
- **Tiktoken** - Tokenization for chunking and cost estimation
- **Google Generative AI** - Gemini model support
- **Backoff** - Exponential backoff retry logic for API calls

### 2. [Graph Operations Dependencies](./graph-operations.md)
Dependencies for graph construction, analysis, and community detection:
- **NetworkX** - Primary graph library (heterogeneous knowledge graph)
- **igraph + leidenalg** - High-performance community detection
- **SciPy** - Sparse matrix operations for Personalized PageRank
- **SortedContainers** - Efficient sorted data structures

### 3. [Vector Search Dependencies](./vector-search.md)
Dependencies for embedding storage, similarity search, and clustering:
- **hnswlib-noderag** - Custom HNSW fork with graph export feature
- **FAISS** - K-means clustering for graph pruning
- **NumPy** - Underlying vector operations and array manipulation

### 4. [Data Processing and Utilities](./data-processing-and-utilities.md)
Dependencies for data I/O, async operations, CLI, and visualization:
- **Pandas & PyArrow** - Tabular data handling and Parquet I/O
- **aiohttp & anyio** - Async HTTP and async abstraction
- **Streamlit & PyVis** - Web UI and graph visualization
- **Rich & tqdm** - Terminal output and progress bars
- **Click & Jinja2** - CLI interface and templating

## 📊 Complete Dependency Map

### Core Dependencies by Processing Layer

```
┌─────────────────────────────────────────────────────────────────┐
│                    Application Layer                             │
│  • Streamlit (2.0.5) - Web UI                                   │
│  • Flask (3.1.0) - Web server                                   │
│  • Click (8.1.8) - CLI interface                                │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                     Semantic Layer                               │
│  • OpenAI (1.66.3) - LLM integration                            │
│  • Pydantic (2.10.6) - Structured output                        │
│  • Tiktoken (0.9.0) - Tokenization                              │
│  • Backoff (2.2.1) - Retry logic                                │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                     Graph Layer                                  │
│  • NetworkX (3.4.2) - Graph construction                        │
│  • igraph (0.11.8) + leidenalg (0.10.2) - Communities           │
│  • SciPy (1.12.0) - Sparse PPR                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                     Vector Layer                                 │
│  • hnswlib-noderag (0.8.2) - HNSW search                        │
│  • FAISS (1.10.0) - K-means clustering                          │
│  • NumPy (1.26.4) - Vector operations                           │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                     Storage Layer                                │
│  • Pandas (2.2.3) - Tabular data                                │
│  • PyArrow (19.0.1) - Parquet I/O                               │
│  • Pickle (built-in) - Graph serialization                      │
│  • YAML (6.0.2) - Configuration                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 🎯 Summary Table

### Core Dependencies (Direct Usage)

| Package | Version | Category | Primary Role | Performance Impact |
|---------|---------|----------|--------------|-------------------|
| openai | 1.66.3 | LLM/AI | LLM API integration | High (API latency) |
| pydantic | 2.10.6 | LLM/AI | Structured output validation | Medium (parsing) |
| tiktoken | 0.9.0 | LLM/AI | Tokenization & cost estimation | Low (fast C++) |
| networkx | 3.4.2 | Graph | Knowledge graph construction | Medium (Python) |
| igraph | 0.11.8 | Graph | Fast community detection | High (C core) |
| leidenalg | 0.10.2 | Graph | Leiden algorithm | High (C++) |
| scipy | 1.12.0 | Graph | Sparse matrix PPR | High (optimized) |
| hnswlib-noderag | 0.8.2 | Vector | HNSW similarity search | Very High (C++) |
| faiss-cpu | 1.10.0 | Vector | K-means clustering | Very High (C++/SIMD) |
| numpy | 1.26.4 | Vector | Array operations | High (BLAS) |
| pandas | 2.2.3 | Data | Tabular data processing | Medium |
| pyarrow | 19.0.1 | Data | Parquet I/O | High (C++) |
| aiohttp | 3.11.11 | Async | Async HTTP client | High (concurrent) |
| streamlit | 2.0.5 | UI | Web interface | Low (dev only) |

### Utility Dependencies

| Package | Version | Category | Primary Role |
|---------|---------|----------|--------------|
| backoff | 2.2.1 | Resilience | Exponential backoff retry |
| tenacity | 9.0.0 | Resilience | Advanced retry logic |
| tqdm | 4.67.1 | UI | Progress bars |
| rich | 13.9.4 | UI | Terminal formatting |
| requests | 2.32.3 | HTTP | Synchronous HTTP |
| click | 8.1.8 | CLI | Command-line interface |
| jinja2 | 3.1.5 | Templates | Prompt templating |
| pyvis | 0.3.2 | Visualization | Interactive graph visualization |
| sortedcontainers | 2.4.0 | Data Structures | Sorted collections |
| anyio | 4.8.0 | Async | Async abstraction layer |
| typing-extensions | 4.12.2 | Development | Enhanced type hints |

### Transitive Dependencies (Auto-installed)

| Package | Version | Required By | Role |
|---------|---------|-------------|------|
| httpx | 0.28.1 | openai | HTTP/2 client for OpenAI |
| distro | 1.9.0 | openai | OS detection |
| sniffio | 1.3.1 | anyio | Async library detection |
| idna | 3.10 | requests | International domain names |
| certifi | 2024.12.14 | requests | CA certificates |
| charset-normalizer | 3.4.0 | requests | Encoding detection |
| urllib3 | 2.3.0 | requests | Low-level HTTP |
| MarkupSafe | 3.0.2 | jinja2 | Safe string escaping |
| aiohappyeyeballs | 2.4.4 | aiohttp | Fast DNS resolution |
| aiosignal | 1.3.2 | aiohttp | Signal management |
| attrs | 24.3.0 | aiohttp | Class decorators |
| frozenlist | 1.5.0 | aiohttp | Frozen lists |
| multidict | 6.1.0 | aiohttp | Multi-value dicts |
| propcache | 0.2.1 | aiohttp | Property caching |
| yarl | 1.18.3 | aiohttp | URL parsing |

## 🔄 Dependency Flow in NodeRAG Pipeline

### Indexing Pipeline
```
Document Input
    │
    ├─→ [OpenAI + Tiktoken] → Text Decomposition → Semantic Units
    │                           │
    │                           ├─→ [NetworkX] → Graph Construction
    │                           │
    │                           └─→ [OpenAI] → Entity Embeddings
    │                                   │
    │                                   └─→ [hnswlib-noderag] → HNSW Index
    │
    ├─→ [NetworkX + igraph + leidenalg] → Community Detection
    │       │
    │       └─→ [OpenAI + Pydantic] → Community Summaries
    │               │
    │               └─→ [hnswlib-noderag] → HNSW Graph Update
    │
    └─→ [Pandas + PyArrow] → Parquet Export
```

### Query Pipeline
```
User Query
    │
    ├─→ [OpenAI + Tiktoken] → Query Embedding
    │       │
    │       └─→ [hnswlib-noderag] → HNSW Retrieval (enter points)
    │
    ├─→ [OpenAI + Pydantic] → Query Decomposition
    │       │
    │       └─→ [re + NetworkX] → Accurate Search (regex matching)
    │
    ├─→ [NetworkX + SciPy] → Sparse PPR (graph expansion)
    │       │
    │       └─→ Top-K Selection → Context Assembly
    │
    └─→ [OpenAI + Jinja2] → Answer Generation → Final Response
```

## 📦 Installation Profiles

### Minimal Installation (Search Only)
```bash
pip install openai==1.66.3 \
            pydantic==2.10.6 \
            tiktoken==0.9.0 \
            networkx==3.4.2 \
            scipy==1.12.0 \
            hnswlib-noderag==0.8.2 \
            numpy==1.26.4 \
            pandas==2.2.3 \
            pyarrow==19.0.1 \
            pyyaml==6.0.2
```

### Full Installation (with Indexing)
```bash
pip install -r requirements.txt
```

Additional requirements for indexing:
- `igraph==0.11.8` + `leidenalg==0.10.2` (community detection)
- `faiss-cpu==1.10.0` (k-means clustering)
- `backoff==2.2.1` (API retry logic)

### Development Installation (with UI)
```bash
pip install -r requirements.txt
pip install streamlit==2.0.5 pyvis==0.3.2 flask==3.1.0
```

## 🔗 Critical Dependencies Explained

### Why These Specific Versions?

1. **openai==1.66.3**: Latest stable with `beta.chat.completions.parse()` for structured outputs
2. **pydantic==2.10.6**: v2.x required for response_format integration with OpenAI SDK
3. **tiktoken==0.9.0**: Latest with `cl100k_base` encoding for GPT-4o
4. **networkx==3.4.2**: Latest with improved node attribute handling
5. **hnswlib-noderag==0.8.2**: Custom fork with `export_graph()` method for HNSW graph extraction
6. **numpy==1.26.4**: Last version before v2.0 breaking changes
7. **scipy==1.12.0**: Compatible with NumPy 1.26.x for sparse operations

### Custom Forks

**hnswlib-noderag** is a custom fork of the original `hnswlib` library:
- **Original**: https://github.com/nmslib/hnswlib
- **Fork**: https://github.com/Wannabeasmartguy/hnswlib
- **Key Addition**: `export_graph()` method to extract HNSW graph structure as NetworkX graph
- **Why Fork**: NodeRAG needs to concatenate HNSW graph with knowledge graph for unified PPR
- **Compatibility**: Drop-in replacement for original hnswlib

## 🎨 Dependency Design Patterns

### 1. Adapter Pattern (LLM Integration)
```python
# Abstract LLM interface
class LLM(ABC):
    @abstractmethod
    async def _create_completion_async(self, messages, response_format=None):
        pass

# Concrete adapters
class OPENAI(LLM):  # Uses openai SDK
class Gemini(LLM):   # Uses google-generativeai SDK
```

### 2. Strategy Pattern (Graph Operations)
```python
# Different community detection strategies
if use_igraph:
    communities = leidenalg.find_partition(...)  # Fast C++ implementation
else:
    communities = nx.community.louvain_communities(...)  # Pure Python
```

### 3. Facade Pattern (Storage)
```python
# storage.py provides unified interface
storage.save(graph)  # Uses pickle, pandas, or pyarrow depending on type
storage.load(path)   # Auto-detects format
```

### 4. Decorator Pattern (Resilience)
```python
@backoff.on_exception(backoff.expo, Exception, max_tries=5)
async def request_with_retry(prompt):
    return await openai_client.request(prompt)
```

## 🔧 Configuration and Environment

### Required Environment Variables
```bash
OPENAI_API_KEY=sk-...           # For OpenAI models
GOOGLE_API_KEY=...              # For Gemini models (optional)
```

### Configuration Files
- `config.yaml` - Uses **PyYAML** for loading
- Prompt templates - Uses **Jinja2** for rendering
- Model configs - Uses **Pydantic** for validation

## 📈 Performance Characteristics

### Speed Rankings (Operations per Second)

| Operation | Library | Speed | Benchmark |
|-----------|---------|-------|-----------|
| Vector similarity | hnswlib | ~10,000 QPS | k=10, 1M vectors |
| Sparse matrix multiply | SciPy | ~5,000 OPS | 100K nodes, 500K edges |
| Community detection | leidenalg | ~1,000 graphs/s | 10K nodes, 50K edges |
| Token counting | tiktoken | ~100,000 docs/s | Average document |
| LLM API call | OpenAI | ~10-50 req/s | Rate limited |
| Graph operations | NetworkX | ~1,000 OPS | Python overhead |
| Parquet I/O | PyArrow | ~500 MB/s | Compression dependent |

### Memory Footprint (Typical Usage)

| Component | Library | Memory | Notes |
|-----------|---------|--------|-------|
| HNSW index | hnswlib | ~4 GB | 1M vectors @ 1536 dims |
| Knowledge graph | NetworkX | ~2 GB | 500K nodes, 2M edges |
| Embeddings cache | NumPy | ~6 GB | 1M vectors @ float32 |
| Community graph | igraph | ~500 MB | Temporary during Leiden |
| PPR computation | SciPy | ~1 GB | Sparse CSR matrix |
| Total runtime | - | ~10-15 GB | For large corpus |

## 🚀 Optimization Tips

### 1. Use Precomputed Embeddings
Cache OpenAI embeddings to avoid redundant API calls:
```python
# Uses Pandas + PyArrow for efficient storage
embedding_cache = pd.read_parquet('embeddings.parquet')
```

### 2. Batch API Requests
Use `asyncio` with `aiohttp` for concurrent LLM calls:
```python
async with aiohttp.ClientSession() as session:
    tasks = [request_async(prompt) for prompt in batch]
    results = await asyncio.gather(*tasks)
```

### 3. Sparse Matrix Operations
SciPy CSR format for efficient PPR:
```python
trans_matrix = sparse.csr_matrix((data, (row, col)))
probs = trans_matrix.dot(probs)  # Fast matrix-vector multiply
```

### 4. Parallel Community Detection
Use igraph's C core for 10-100x speedup over pure Python:
```python
import igraph as ig
G_igraph = ig.Graph.from_networkx(G)
communities = leidenalg.find_partition(G_igraph, leidenalg.ModularityVertexPartition)
```

## 🔍 Debugging and Development

### Useful Development Dependencies

| Package | Purpose | Usage |
|---------|---------|-------|
| IPython | Enhanced REPL | `ipython -i script.py` |
| tqdm | Progress tracking | `for item in tqdm(items):` |
| rich | Pretty printing | `rich.print(data)` |
| streamlit | Quick UI prototyping | `streamlit run app.py` |
| pyvis | Graph visualization | `net.show('graph.html')` |

### Dependency Conflict Resolution

**Common Issues**:

1. **NumPy version conflicts**: Many libraries require numpy < 2.0
   ```bash
   pip install "numpy<2.0"
   ```

2. **Pydantic v1 vs v2**: Ensure all packages use Pydantic v2
   ```bash
   pip install "pydantic>=2.0"
   ```

3. **FAISS installation**: Use `faiss-cpu` for CPU-only environments
   ```bash
   pip install faiss-cpu  # NOT faiss-gpu
   ```

4. **igraph + leidenalg**: Must be installed together
   ```bash
   pip install igraph leidenalg
   ```

## 📚 External Resources

### Official Documentation Links

**LLM & AI**:
- [OpenAI Python SDK](https://github.com/openai/openai-python)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Tiktoken GitHub](https://github.com/openai/tiktoken)
- [Google Generative AI](https://ai.google.dev/api/python/google/generativeai)

**Graph Operations**:
- [NetworkX Documentation](https://networkx.org/documentation/stable/)
- [igraph Python Documentation](https://igraph.org/python/)
- [Leidenalg Documentation](https://leidenalg.readthedocs.io/)
- [SciPy Sparse Matrices](https://docs.scipy.org/doc/scipy/reference/sparse.html)

**Vector Search**:
- [hnswlib GitHub](https://github.com/nmslib/hnswlib)
- [hnswlib-noderag Fork](https://github.com/Wannabeasmartguy/hnswlib)
- [FAISS Documentation](https://faiss.ai/)
- [NumPy Documentation](https://numpy.org/doc/)

**Data & Utilities**:
- [Pandas Documentation](https://pandas.pydata.org/docs/)
- [PyArrow Documentation](https://arrow.apache.org/docs/python/)
- [aiohttp Documentation](https://docs.aiohttp.org/)
- [Streamlit Documentation](https://docs.streamlit.io/)

## 🎓 Learning Path

For developers new to NodeRAG dependencies, recommended learning order:

1. **Start with basics**: NumPy, Pandas (data handling)
2. **Learn graph fundamentals**: NetworkX (graph construction and operations)
3. **Understand LLM integration**: OpenAI SDK, Pydantic (structured outputs)
4. **Explore vector search**: hnswlib, FAISS (similarity search and clustering)
5. **Advanced graph algorithms**: igraph, leidenalg, SciPy (communities and PPR)
6. **Async patterns**: aiohttp, anyio (concurrent operations)
7. **UI and visualization**: Streamlit, PyVis (prototyping and debugging)

## 📝 Version Compatibility Matrix

| NodeRAG | Python | OpenAI | Pydantic | NetworkX | NumPy | Notes |
|---------|--------|--------|----------|----------|-------|-------|
| 0.1.x | 3.10+ | 1.66+ | 2.10+ | 3.4+ | 1.26.x | Current |
| Future | 3.11+ | 2.0+ | 2.x | 3.x | 2.x | Planned |

## 🆘 Support and Troubleshooting

### Common Error Messages

1. **"No module named 'hnswlib_noderag'"**
   ```bash
   pip install hnswlib-noderag  # Note the dash, not underscore
   ```

2. **"OpenAI API key not found"**
   ```bash
   export OPENAI_API_KEY='sk-...'
   ```

3. **"scipy.sparse.csr_matrix has no attribute 'dot'"**
   - Ensure NumPy < 2.0 for compatibility

4. **"igraph.InternalError: Error at leiden.c"**
   - Reinstall both igraph and leidenalg together

### Performance Issues

1. **Slow HNSW search**: Increase `ef` parameter (default: 50)
2. **High memory usage**: Reduce `HNSW_results` config (default: 50)
3. **API rate limits**: Implement backoff or reduce concurrency
4. **Slow PPR**: Use sparse matrices (SciPy CSR format)

---

## 📋 Quick Reference Checklist

### Before Starting Development
- [ ] Python 3.10+ installed
- [ ] All dependencies from `requirements.txt` installed
- [ ] OpenAI API key configured
- [ ] Sufficient RAM (minimum 16 GB recommended)
- [ ] Disk space for vector indices and graphs

### For Indexing Pipeline
- [ ] OpenAI SDK (LLM calls)
- [ ] Pydantic (structured outputs)
- [ ] NetworkX (graph construction)
- [ ] igraph + leidenalg (community detection)
- [ ] hnswlib-noderag (vector indexing)
- [ ] FAISS (clustering)
- [ ] Pandas + PyArrow (storage)

### For Search Pipeline
- [ ] OpenAI SDK (embeddings + answer generation)
- [ ] NetworkX (graph traversal)
- [ ] SciPy (sparse PPR)
- [ ] hnswlib-noderag (similarity search)
- [ ] Pandas (loading cached data)

### For UI/Visualization
- [ ] Streamlit (web interface)
- [ ] PyVis (interactive graphs)
- [ ] Rich (terminal output)
- [ ] tqdm (progress bars)

---

**Last Updated**: 2025-11-13
**NodeRAG Version**: 0.1.x
**Total Dependencies**: ~60 packages (core + transitive)
