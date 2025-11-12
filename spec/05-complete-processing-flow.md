# Полный Flow Обработки Документов в NodeRAG

## Обзор

Этот документ представляет end-to-end процесс обработки документов в NodeRAG, от загрузки сырого текста до построения индексируемого графа знаний с концептуальным слоем.

---

## Визуализация Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                     STAGE 0: INITIALIZATION                      │
│                        (INIT Pipeline)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Check folder structure                                      │
│      ├─ Validate main_folder exists                             │
│      └─ Validate input_folder exists                            │
│                                                                   │
│  [2] Load document files                                         │
│      ├─ Scan input_folder/*.txt, *.md                           │
│      └─ Create document_paths list                              │
│                                                                   │
│  [3] Check incrementality                                        │
│      ├─ Compute SHA256(document_paths)                          │
│      ├─ Compare with previous hash                              │
│      └─ Return: New documents detected?                         │
│                                                                   │
│  Output: document_hash.json                                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   STAGE 1: DOCUMENT PIPELINE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Load documents                                              │
│      document_objects = [                                        │
│          Document(raw_context, path, semantic_splitter)          │
│          for path in document_paths                              │
│      ]                                                            │
│                                                                   │
│  [2] Increment check                                             │
│      ├─ Load existing doc_hash_ids from documents.parquet       │
│      ├─ Filter: only new documents                              │
│      └─ Update documents list                                    │
│                                                                   │
│  [3] Semantic chunking                                           │
│      for doc in documents:                                       │
│          doc.split() → text_units                               │
│                                                                   │
│      SemanticTextSplitter:                                       │
│      ├─ chunk_size: 1048 tokens                                 │
│      ├─ Boundaries: \n\n, \n, ., 。, !, ?, ;                    │
│      └─ Output: List[Text_unit]                                 │
│                                                                   │
│  [4] Storage                                                     │
│      ├─ documents.parquet (metadata)                            │
│      ├─ text.parquet (text units)                               │
│      └─ indices.json (human_readable_ids)                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    STAGE 2: TEXT PIPELINE                        │
│                  (Semantic Decomposition via LLM)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Load text units from text.parquet                          │
│                                                                   │
│  [2] Increment check                                             │
│      ├─ Load processed hash_ids from text_decomposition.jsonl  │
│      └─ Filter: only unprocessed text units                     │
│                                                                   │
│  [3] LLM Decomposition (async parallel)                         │
│                                                                   │
│      for text_unit in text_units (parallel):                    │
│                                                                   │
│          ┌─────────────────────────────────────┐                │
│          │  LLM AGENT: Text Decomposition      │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │  Input: text_unit.raw_context       │                │
│          │                                      │                │
│          │  Prompt: text_decomposition_prompt  │                │
│          │  ├─ Segment into semantic units     │                │
│          │  ├─ Extract entities (UPPERCASE)    │                │
│          │  └─ Extract relationships (triplets)│                │
│          │                                      │                │
│          │  Output: {                           │                │
│          │    "Output": [                       │                │
│          │      {                               │                │
│          │        "semantic_unit": "...",       │                │
│          │        "entities": ["E1", "E2"],     │                │
│          │        "relationships": [            │                │
│          │          "E1, rel, E2"               │                │
│          │        ]                             │                │
│          │      }                               │                │
│          │    ]                                 │                │
│          │  }                                   │                │
│          │                                      │                │
│          │  Temperature: 0.0                    │                │
│          │  Max tokens: 10000                   │                │
│          │  Retry: 4 attempts with backoff     │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                                                                   │
│  [4] Error handling                                              │
│      ├─ Cache errors in LLM_error.jsonl                         │
│      └─ Support rerun for failed requests                       │
│                                                                   │
│  [5] Storage                                                     │
│      └─ text_decomposition.jsonl:                               │
│         {text_hash_id, text_id, response, processed: false}     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                   STAGE 3: GRAPH PIPELINE                        │
│                   (Graph Construction)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Load data                                                   │
│      ├─ text_decomposition.jsonl                                │
│      ├─ Existing graph (if incremental)                         │
│      └─ Filter: processed == false                              │
│                                                                   │
│  [2] Build graph (async parallel)                               │
│                                                                   │
│      for decomposition_result in results (parallel):            │
│                                                                   │
│          for semantic_unit in Output:                           │
│                                                                   │
│              ┌──────────────────────────────────┐               │
│              │ Add Semantic Unit Node           │               │
│              ├──────────────────────────────────┤               │
│              │ su = Semantic_unit(text, text_id)│               │
│              │ G.add_node(su.hash_id,           │               │
│              │            type='semantic_unit', │               │
│              │            weight=1)             │               │
│              └──────────────────────────────────┘               │
│                                                                   │
│              for entity in entities:                            │
│                                                                   │
│                  ┌──────────────────────────────┐               │
│                  │ Add Entity Node              │               │
│                  ├──────────────────────────────┤               │
│                  │ e = Entity(entity, text_id)  │               │
│                  │ G.add_node(e.hash_id,        │               │
│                  │            type='entity',    │               │
│                  │            weight=1)         │               │
│                  │                              │               │
│                  │ # Link SU → Entity           │               │
│                  │ G.add_edge(su.hash_id,       │               │
│                  │            e.hash_id,        │               │
│                  │            weight=1)         │               │
│                  └──────────────────────────────┘               │
│                                                                   │
│              for relationship in relationships:                 │
│                                                                   │
│                  ┌──────────────────────────────┐               │
│                  │ Parse & Add Relationship     │               │
│                  ├──────────────────────────────┤               │
│                  │ Parse: "E1, rel, E2"         │               │
│                  │                              │               │
│                  │ If len != 3:                 │               │
│                  │   ┌──────────────────┐      │               │
│                  │   │ LLM AGENT:       │      │               │
│                  │   │ Relationship     │      │               │
│                  │   │ Reconstruction   │      │               │
│                  │   └──────────────────┘      │               │
│                  │                              │               │
│                  │ r = Relationship(tuple, tid) │               │
│                  │ G.add_node(r.hash_id,        │               │
│                  │            type='relationship│               │
│                  │            weight=1)         │               │
│                  │ G.add_node(source.hash_id)   │               │
│                  │ G.add_node(target.hash_id)   │               │
│                  │                              │               │
│                  │ # Graph: S → R → T           │               │
│                  │ G.add_edge(source, r)        │               │
│                  │ G.add_edge(r, target)        │               │
│                  └──────────────────────────────┘               │
│                                                                   │
│  [3] Mark processed                                              │
│      decomposition_result['processed'] = True                   │
│                                                                   │
│  [4] Storage                                                     │
│      ├─ semantic_units.parquet                                  │
│      ├─ entities.parquet                                        │
│      ├─ relationship.parquet                                    │
│      ├─ graph.pkl (NetworkX)                                    │
│      └─ text_decomposition.jsonl (updated)                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                 STAGE 4: ATTRIBUTE PIPELINE                      │
│              (Attribute Generation for Important Nodes)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Find important nodes                                        │
│                                                                   │
│      ┌──────────────────────────────────────┐                   │
│      │ NodeImportance Algorithm             │                   │
│      ├──────────────────────────────────────┤                   │
│      │                                       │                   │
│      │ Method 1: K-core decomposition       │                   │
│      │   k = round(log(|V|) × sqrt(avg_deg))│                   │
│      │   k_core_subgraph = k_core(G, k)     │                   │
│      │   important += [e for e in k_core    │                   │
│      │                 if type=='entity'    │                   │
│      │                 and weight > 1]      │                   │
│      │                                       │                   │
│      │ Method 2: Betweenness centrality     │                   │
│      │   betweenness = nx.betweenness(G,k=10│                   │
│      │   threshold = avg × log10(|V|)       │                   │
│      │   important += [e for e in G         │                   │
│      │                 if betweenness > thr │                   │
│      │                 and type=='entity'   │                   │
│      │                 and weight > 1]      │                   │
│      │                                       │                   │
│      │ Return: unique(important)             │                   │
│      │                                       │                   │
│      └──────────────────────────────────────┘                   │
│                                                                   │
│  [2] Increment check                                             │
│      ├─ Load existing attributes from attributes.parquet        │
│      └─ Filter: nodes without attributes                        │
│                                                                   │
│  [3] Generate attributes (async parallel)                       │
│                                                                   │
│      for entity_node in important_nodes (parallel):             │
│                                                                   │
│          ┌─────────────────────────────────────┐                │
│          │ Collect neighborhood context        │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ entity = mapper.get(node, 'context')│                │
│          │                                      │                │
│          │ semantic_units = [                  │                │
│          │   mapper.get(n, 'context')          │                │
│          │   for n in G.neighbors(node)        │                │
│          │   if type == 'semantic_unit'        │                │
│          │ ]                                    │                │
│          │                                      │                │
│          │ relationships = [                   │                │
│          │   mapper.get(n, 'context')          │                │
│          │   for n in G.neighbors(node)        │                │
│          │   if type == 'relationship'         │                │
│          │ ]                                    │                │
│          │                                      │                │
│          │ If exceeds token limit:             │                │
│          │   Sort neighbors by importance      │                │
│          │   (weight of their neighbors)       │                │
│          │   Take top until token limit        │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                          ↓                                       │
│          ┌─────────────────────────────────────┐                │
│          │ LLM AGENT: Attribute Generation     │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ Prompt: attribute_generation_prompt │                │
│          │   .format(                           │                │
│          │     entity=entity,                   │                │
│          │     semantic_units="\n".join(...),   │                │
│          │     relationships="\n".join(...)     │                │
│          │   )                                  │                │
│          │                                      │                │
│          │ Output:                              │                │
│          │   "Narrative description of entity, │                │
│          │    like character sketch, up to     │                │
│          │    2000 words, capturing essential  │                │
│          │    attributes and relationships"    │                │
│          │                                      │                │
│          │ Temperature: 0.0                     │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                          ↓                                       │
│          ┌─────────────────────────────────────┐                │
│          │ Add to graph                        │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ attr = Attribute(response, node)    │                │
│          │ G.add_node(attr.hash_id,            │                │
│          │            type='attribute',        │                │
│          │            weight=1)                │                │
│          │ G.add_edge(node, attr.hash_id,      │                │
│          │            weight=1)                │                │
│          │ G.nodes[node]['attributes'] =       │                │
│          │   [attr.hash_id]                    │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                                                                   │
│  [4] Storage                                                     │
│      ├─ attributes.parquet                                      │
│      └─ graph.pkl (updated)                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                 STAGE 5: EMBEDDING PIPELINE                      │
│                (Vector Representation Generation)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Create mapper                                               │
│      Mapper([text.parquet,                                       │
│              semantic_units.parquet,                             │
│              attributes.parquet])                                │
│      → hash_id: {context, type, embedding}                      │
│                                                                   │
│  [2] Find nodes without embeddings                              │
│      none_embedding_ids = [                                      │
│        id for id, data in mapper                                │
│        if data['embedding'] is None                             │
│      ]                                                            │
│                                                                   │
│  [3] Generate embeddings (batch async)                          │
│                                                                   │
│      batch_size = config.embedding_batch_size (default: 100)    │
│                                                                   │
│      for batch in batches(none_embedding_ids, batch_size):      │
│                                                                   │
│          ┌─────────────────────────────────────┐                │
│          │ Prepare batch                       │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ context_dict = {                    │                │
│          │   id: mapper.get(id, 'context')     │                │
│          │   for id in batch                   │                │
│          │ }                                    │                │
│          │                                      │                │
│          │ # Filter empty contexts             │                │
│          │ context_dict = {                    │                │
│          │   k: v for k, v in context_dict     │                │
│          │   if v != ""                        │                │
│          │ }                                    │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                          ↓                                       │
│          ┌─────────────────────────────────────┐                │
│          │ LLM AGENT: Embedding Model          │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ Model: OpenAI or Gemini embedding   │                │
│          │                                      │                │
│          │ Input: List[context_strings]        │                │
│          │                                      │                │
│          │ Output: List[embedding_vectors]     │                │
│          │   - OpenAI: dimension 1536 or 512   │                │
│          │   - Gemini: dimension 768            │                │
│          │                                      │                │
│          │ Retry: 4 attempts with backoff      │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                          ↓                                       │
│          ┌─────────────────────────────────────┐                │
│          │ Cache and update                    │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ For each (id, embedding):           │                │
│          │   embedding_cache.jsonl.append({    │                │
│          │     'hash_id': id,                  │                │
│          │     'embedding': embedding          │                │
│          │   })                                │                │
│          │   mapper.add_attribute(id,          │                │
│          │     'embedding', 'done')            │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                                                                   │
│  [4] Insert embeddings to storage                               │
│      ├─ Read embedding_cache.jsonl                              │
│      ├─ Append to embedding.parquet                             │
│      ├─ Update source parquet files                             │
│      └─ Delete cache                                            │
│                                                                   │
│  [5] Storage                                                     │
│      ├─ embedding.parquet: {hash_id, embedding}                 │
│      ├─ text.parquet (updated with embedding='done')            │
│      ├─ semantic_units.parquet (updated)                        │
│      └─ attributes.parquet (updated)                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  STAGE 6: SUMMARY PIPELINE                       │
│         (Community Detection + Conceptual Layer Creation)        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Prepare mapper with embeddings                             │
│      mapper = Mapper([semantic_units.parquet,                   │
│                       attributes.parquet])                       │
│      mapper.add_embedding(embedding.parquet)                    │
│                                                                   │
│  [2] Convert graph                                               │
│      G_ig = IGraph(NetworkX_graph).to_igraph()                  │
│                                                                   │
│  [3] Community detection                                         │
│                                                                   │
│      ┌─────────────────────────────────────┐                    │
│      │ Leiden Algorithm                    │                    │
│      ├─────────────────────────────────────┤                    │
│      │                                      │                    │
│      │ partition = la.find_partition(      │                    │
│      │   G_ig,                              │                    │
│      │   la.ModularityVertexPartition      │                    │
│      │ )                                    │                    │
│      │                                      │                    │
│      │ Objective: Maximize modularity      │                    │
│      │                                      │                    │
│      │ Output: List[community_nodes]       │                    │
│      │   [                                  │                    │
│      │     [node1, node2, node5],          │                    │
│      │     [node3, node7, node8],          │                    │
│      │     ...                              │                    │
│      │   ]                                  │                    │
│      │                                      │                    │
│      └─────────────────────────────────────┘                    │
│                                                                   │
│  [4] Generate community summaries (async parallel)              │
│                                                                   │
│      for community in partition (parallel):                     │
│                                                                   │
│          ┌─────────────────────────────────────┐                │
│          │ Collect community context           │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ used_units = []                     │                │
│          │                                      │                │
│          │ for node in community:              │                │
│          │   if type == 'semantic_unit':       │                │
│          │     used_units.append(node)         │                │
│          │   elif type == 'attribute':         │                │
│          │     used_units.append(node)         │                │
│          │   elif has_attribute:               │                │
│          │     for neighbor in G.neighbors(node│                │
│          │       if type == 'attribute':       │                │
│          │         used_units.append(neighbor) │                │
│          │                                      │                │
│          │ If exceeds token limit:             │                │
│          │   Sort by neighbor weights          │                │
│          │   Take top units                    │                │
│          │                                      │                │
│          │ content = "\n".join([               │                │
│          │   mapper.get(u, 'context')          │                │
│          │   for u in used_units               │                │
│          │ ])                                   │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                          ↓                                       │
│          ┌─────────────────────────────────────┐                │
│          │ LLM AGENT: Community Summary        │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ Prompt: community_summary_prompt    │                │
│          │   .format(content=content)          │                │
│          │                                      │                │
│          │ Task:                                │                │
│          │ - Extract high-level concepts       │                │
│          │ - Themes, theories, impacts         │                │
│          │ - Avoid redundancy                  │                │
│          │ - Ensure diversity                  │                │
│          │                                      │                │
│          │ Output: {                            │                │
│          │   "high_level_elements": [          │                │
│          │     {                                │                │
│          │       "title": "Concept Title",     │                │
│          │       "description": "Detailed desc"│                │
│          │     }                                │                │
│          │   ]                                  │                │
│          │ }                                    │                │
│          │                                      │                │
│          │ Temperature: 0.0                     │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                          ↓                                       │
│          ┌─────────────────────────────────────┐                │
│          │ Create High-Level Element nodes     │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ for he_data in high_level_elements: │                │
│          │                                      │                │
│          │   he = High_level_elements(         │                │
│          │     description=he_data['description│                │
│          │     title=he_data['title'],         │                │
│          │     config=config                   │                │
│          │   )                                  │                │
│          │   he.related_node(community_nodes)  │                │
│          │                                      │                │
│          │   # Add content node                │                │
│          │   G.add_node(he.hash_id,            │                │
│          │              type='high_level_element│                │
│          │              weight=1)              │                │
│          │                                      │                │
│          │   # Add title node                  │                │
│          │   G.add_node(he.title_hash_id,      │                │
│          │              type='high_level_element│                │
│          │              weight=1,              │                │
│          │              related_node=he.hash_id│                │
│          │                                      │                │
│          │   # Link title ↔ content            │                │
│          │   G.add_edge(he.hash_id,            │                │
│          │              he.title_hash_id,      │                │
│          │              weight=1)              │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                                                                   │
│  [5] Generate HE embeddings (batch async)                       │
│      Similar to Stage 5, but for HE descriptions                │
│                                                                   │
│  [6] Link HE to community nodes                                 │
│                                                                   │
│      all_nodes = union(all community_nodes)                     │
│      threshold = (|all_nodes| + |HE|) / centroids               │
│                                                                   │
│      If threshold > config.Hcluster_size:                       │
│                                                                   │
│          ┌─────────────────────────────────────┐                │
│          │ KMeans Clustering Strategy          │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ # Combine embeddings                │                │
│          │ node_emb = [mapper.embeddings[n]    │                │
│          │             for n in all_nodes]     │                │
│          │ he_emb = [he.embedding              │                │
│          │           for he in HEs]            │                │
│          │ all_emb = vstack([he_emb, node_emb])│                │
│          │                                      │                │
│          │ # KMeans clustering                 │                │
│          │ centroids = ceil(sqrt(total_nodes)) │                │
│          │ kmeans = faiss.Kmeans(d, k=centroids│                │
│          │ kmeans.train(all_emb)               │                │
│          │ _, labels = kmeans.assign(all_emb)  │                │
│          │                                      │                │
│          │ # Link only same-cluster nodes      │                │
│          │ for i, he in enumerate(HEs):        │                │
│          │   for j, node in enumerate(all_nodes│                │
│          │     if labels[i] == labels[j]:      │                │
│          │       if node in he.related_nodes:  │                │
│          │         G.add_edge(node, he.hash_id,│                │
│          │                    weight=1)        │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                                                                   │
│      Else:                                                       │
│                                                                   │
│          ┌─────────────────────────────────────┐                │
│          │ Direct Linking Strategy             │                │
│          ├─────────────────────────────────────┤                │
│          │                                      │                │
│          │ for he in high_level_elements:      │                │
│          │   for node in he.related_nodes:     │                │
│          │     G.add_edge(node, he.hash_id,    │                │
│          │                weight=1)            │                │
│          │                                      │                │
│          └─────────────────────────────────────┘                │
│                                                                   │
│  [7] Storage                                                     │
│      ├─ high_level_elements.parquet                             │
│      ├─ high_level_elements_titles.parquet                      │
│      ├─ embedding.parquet (appended)                            │
│      └─ graph.pkl (updated)                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│               STAGE 7: INSERT TEXT PIPELINE                      │
│            (Insert Text Units into Graph)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Add text unit nodes to graph                               │
│      for text_unit in text_units:                               │
│        G.add_node(text_unit.hash_id,                            │
│                   type='text_unit',                             │
│                   weight=1)                                      │
│                                                                   │
│  [2] Link text units to semantic units                          │
│      Based on text_hash_id relationships                        │
│                                                                   │
│  [3] Storage                                                     │
│      └─ graph.pkl (updated)                                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  STAGE 8: HNSW PIPELINE                          │
│         (Build HNSW Indexes for Fast Vector Search)             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [1] Collect all embeddings                                     │
│      embeddings_dict = load_parquet(embedding.parquet)          │
│      → {hash_id: embedding_vector}                              │
│                                                                   │
│  [2] Build HNSW index                                            │
│                                                                   │
│      ┌─────────────────────────────────────┐                    │
│      │ HNSW (Hierarchical Navigable        │                    │
│      │       Small World) Graph            │                    │
│      ├─────────────────────────────────────┤                    │
│      │                                      │                    │
│      │ Parameters:                          │                    │
│      │ - M: links per node per layer       │                    │
│      │ - ef_construction: build param      │                    │
│      │ - ef_search: query param            │                    │
│      │                                      │                    │
│      │ Structure:                           │                    │
│      │   Layer N (sparse)                  │                    │
│      │     └─ few nodes, long jumps        │                    │
│      │   Layer 2                            │                    │
│      │     └─ more nodes                   │                    │
│      │   Layer 1                            │                    │
│      │     └─ even more nodes              │                    │
│      │   Layer 0 (dense)                   │                    │
│      │     └─ all nodes, local connections │                    │
│      │                                      │                    │
│      │ Query:                               │                    │
│      │   1. Enter at top layer             │                    │
│      │   2. Greedy search to local min     │                    │
│      │   3. Descend to next layer          │                    │
│      │   4. Repeat until layer 0           │                    │
│      │   5. Return k nearest neighbors     │                    │
│      │                                      │                    │
│      │ Complexity: O(log N) average        │                    │
│      │                                      │                    │
│      └─────────────────────────────────────┘                    │
│                                                                   │
│  [3] Create separate indexes (optional)                         │
│      - Text units index                                          │
│      - Semantic units index                                      │
│      - Attributes index                                          │
│      - High-level elements index                                 │
│                                                                   │
│  [4] Storage                                                     │
│      └─ HNSW index files                                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                         COMPLETION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Final Output:                                                   │
│                                                                   │
│  ├─ Heterogeneous Knowledge Graph                               │
│  │  ├─ Document nodes                                           │
│  │  ├─ Text unit nodes                                          │
│  │  ├─ Semantic unit nodes                                      │
│  │  ├─ Entity nodes                                             │
│  │  ├─ Relationship nodes                                       │
│  │  ├─ Attribute nodes                                          │
│  │  └─ High-level element nodes                                 │
│  │                                                                │
│  ├─ Vector Embeddings                                            │
│  │  ├─ Text units                                               │
│  │  ├─ Semantic units                                           │
│  │  ├─ Attributes                                               │
│  │  └─ High-level elements                                      │
│  │                                                                │
│  ├─ HNSW Indexes for Fast Search                                │
│  │                                                                │
│  └─ Conceptual Layer                                            │
│     └─ High-level themes and concepts                           │
│                                                                   │
│  Ready for:                                                      │
│  - Semantic search                                               │
│  - Graph traversal                                               │
│  - Multi-hop reasoning                                           │
│  - Question answering                                            │
│  - Knowledge extraction                                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Роли LLM Агентов: Сводка

### 1. Text Decomposition Agent
**Pipeline**: Text Pipeline (Stage 2)
**Input**: Text unit (string)
**Output**: Semantic units + Entities + Relationships (JSON)
**Frequency**: Для каждого text unit
**Criticality**: ⭐⭐⭐⭐⭐ (Fundamental)

### 2. Relationship Reconstruction Agent
**Pipeline**: Graph Pipeline (Stage 3)
**Input**: Malformed relationship (list)
**Output**: Corrected triplet (JSON)
**Frequency**: Only when format incorrect
**Criticality**: ⭐⭐⭐ (Error handling)

### 3. Attribute Generation Agent
**Pipeline**: Attribute Pipeline (Stage 4)
**Input**: Entity + Neighborhood context (string)
**Output**: Narrative description (string)
**Frequency**: Only for important entities
**Criticality**: ⭐⭐⭐⭐ (Enrichment)

### 4. Community Summary Agent
**Pipeline**: Summary Pipeline (Stage 6)
**Input**: Community texts (string)
**Output**: High-level concepts (JSON)
**Frequency**: For each community
**Criticality**: ⭐⭐⭐⭐⭐ (Conceptual layer)

### 5. Embedding Agent
**Pipeline**: Embedding Pipeline (Stage 5), Summary Pipeline (Stage 6)
**Input**: Text (string or list)
**Output**: Vector (float array)
**Frequency**: For all indexable nodes
**Criticality**: ⭐⭐⭐⭐⭐ (Search capability)

---

## Ключевые Метрики

### Размер Данных (примерный)

**Для 100 документов (~10MB text)**:
- Documents: 100 nodes
- Text Units: ~2000 nodes
- Semantic Units: ~5000 nodes
- Entities: ~3000 nodes
- Relationships: ~4000 nodes
- Attributes: ~200 nodes (для important entities)
- High-Level Elements: ~100 nodes

**Total Graph**: ~14,500 nodes, ~25,000 edges

### LLM Вызовы

**Для 100 документов**:
- Text Decomposition: ~2000 calls
- Relationship Reconstruction: ~100 calls (5% failure rate)
- Attribute Generation: ~200 calls
- Community Summary: ~20 calls (20 communities)
- Embeddings: ~10,300 embeddings (batch processed)

**Total API calls**: ~2,320 LLM calls + 104 embedding batches

### Время Обработки

**С параллелизацией** (приблизительно):
- INIT: <1 sec
- Document Pipeline: ~30 sec
- Text Pipeline: ~15 min (async, 2000 parallel calls)
- Graph Pipeline: ~2 min
- Attribute Pipeline: ~5 min (200 parallel calls)
- Embedding Pipeline: ~3 min (batched)
- Summary Pipeline: ~10 min
- Insert Text: ~1 min
- HNSW: ~2 min

**Total**: ~40 minutes для 100 документов

**Без параллелизации**: ~8 hours

---

## Инкрементальная Обработка

### Сценарий: Добавление 10 новых документов

**Этапы**:

1. **INIT**: Обнаружение новых файлов (hash check)
2. **Document**: Обработка только 10 новых
3. **Text**: Декомпозиция только новых text units (~200)
4. **Graph**: Добавление новых узлов и ребер
   - Дедупликация entities через hash_id
   - Инкремент weight для существующих
5. **Attribute**: Только новые important entities (~20)
6. **Embedding**: Только новые узлы (~500)
7. **Summary**: Re-run community detection
   - Может изменить communities
   - Создание новых HE
8. **HNSW**: Rebuild индексов (incremental HNSW возможен)

**Время**: ~8 минут (vs 40 минут для full reindexing)

---

## Error Handling Flow

```
┌─────────────────────────────────┐
│ LLM Call                        │
└─────────────────────────────────┘
              ↓
         [Success?]
         ↙        ↘
      Yes          No
       ↓            ↓
    Return    ┌─────────────────┐
    Result    │ Retry (backoff) │
              │ Max 4 attempts  │
              └─────────────────┘
                      ↓
                 [Success?]
                 ↙        ↘
              Yes          No
               ↓            ↓
            Return    ┌──────────────────┐
            Result    │ Cache to         │
                      │ LLM_error.jsonl  │
                      └──────────────────┘
                             ↓
                      [Continue pipeline]
                             ↓
                      [End of pipeline]
                             ↓
                      ┌──────────────────┐
                      │ Check error cache│
                      │ If errors:       │
                      │   Raise exception│
                      └──────────────────┘
                             ↓
                      [User fixes issues]
                             ↓
                      ┌──────────────────┐
                      │ Rerun pipeline   │
                      │ - Load error cache│
                      │ - Retry all      │
                      │ - Clear cache    │
                      └──────────────────┘
```

---

## State Management

### State File: `state.json`

```json
{
  "Current_state": "EMBEDDING_PIPELINE",
  "Error_type": "NO_ERROR",
  "Is_incremental": false
}
```

### Восстановление после прерывания

```python
noderag = NodeRag(config)
noderag.load_state()  # Восстановление из state.json

if noderag.Error_type != State.NO_ERROR:
    await noderag.error_handler()  # Обработка ошибок
else:
    await noderag.state_transition()  # Продолжение pipeline
```

---

## Storage Layout

```
project/
├─ input/                      # Исходные документы
│  ├─ document1.txt
│  └─ document2.md
│
├─ cache/                      # Обработанные данные
│  ├─ documents.parquet
│  ├─ text.parquet
│  ├─ semantic_units.parquet
│  ├─ entities.parquet
│  ├─ relationship.parquet
│  ├─ attributes.parquet
│  ├─ high_level_elements.parquet
│  ├─ high_level_elements_titles.parquet
│  ├─ embedding.parquet
│  ├─ graph.pkl
│  ├─ indices.json
│  ├─ document_hash.json
│  ├─ text_decomposition.jsonl
│  ├─ summary_cache.jsonl (temp)
│  ├─ embedding_cache.jsonl (temp)
│  └─ LLM_error.jsonl (errors)
│
└─ state.json                  # Pipeline state
```

---

## Заключение

NodeRAG pipeline представляет собой сложную многоэтапную систему, которая:

1. **Загружает** документы и разбивает на семантические chunks
2. **Извлекает** структурированные знания через LLM
3. **Строит** гетерогенный граф знаний
4. **Обогащает** важные узлы детальными описаниями
5. **Индексирует** через векторные представления
6. **Создает** концептуальный слой через community detection
7. **Оптимизирует** для быстрого поиска через HNSW

Результатом является rich, multi-layered knowledge graph, готовый для семантического поиска, reasoning и question answering.
