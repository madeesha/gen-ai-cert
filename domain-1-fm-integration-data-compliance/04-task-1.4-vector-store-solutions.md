[← Back to Domain 1](README.md)

# 🗄️ Task 1.4 · Design and implement vector store solutions

> **Domain 1 · Foundation Model Integration, Data Management, and Compliance (31%)**
>
> - **Skill 1.4.1** · Advanced vector database architectures for FM augmentation
> - **Skill 1.4.2** · Metadata frameworks for search precision and context awareness
> - **Skill 1.4.3** · High-performance vector database architectures at scale
> - **Skill 1.4.4** · Integration components for document systems, knowledge bases and wikis
> - **Skill 1.4.5** · Data maintenance systems that keep vector stores current and accurate
>
> Chunking, embedding model choice, k-NN/ANN query configuration, hybrid search, reranking and query handling are covered in [Task 1.5](05-task-1.5-retrieval-mechanisms.md). This file is about the **store**: its architecture, metadata, scale, connectors and data freshness.

## Key points
- **Default managed answer = Bedrock Knowledge Bases.** It covers ingestion, chunking, embedding, storage, retrieval and citations. The vector store underneath is your architecture choice: **OpenSearch Serverless** (hybrid search, ms latency, high QPS) · **S3 Vectors** (cheapest, subsecond, semantic-only) · **Aurora PostgreSQL pgvector** (you already run Postgres, want SQL joins) · **Neptune Analytics** (GraphRAG) · OpenSearch managed clusters · MongoDB Atlas · Pinecone · Redis Enterprise Cloud.
- ⚠️ The **vector store, embedding model, chunking strategy and parser are fixed when a KB is created**. To change any of them, create a new KB or data source and re-ingest. Index dimension must equal embedding dimension, otherwise **ingestion fails**.
- **Separate the concerns.** Use **S3** as the system of record for raw documents. Use **RDS/Aurora or DynamoDB** for the document catalog, metadata, ACLs and ingestion state. Use a **vector store** for embeddings plus chunk text. Link them with a shared **document ID**.
- **DynamoDB has no native vector search.** Pair it with a vector store, or use the **DynamoDB zero-ETL integration with OpenSearch Service** so DynamoDB data becomes vector-searchable.
- **Metadata makes retrieval precise.** In KBs, add a `<file>.metadata.json` sidecar next to each S3 object (`metadataAttributes`) and **re-sync** after adding or changing it. Filter at query time with `equals`, `in`, `greaterThan`, `andAll`/`orAll` and similar operators. Use metadata for **time** (recency), **authorship**, **domain/topic**, **tenant** and **ACL**.
- KBs read the **sidecar `.metadata.json`**, not S3 user-defined metadata or object tags. Use S3 object metadata and tags for lifecycle, governance and catalog jobs, then project them into the sidecar if retrieval needs them.
- **Topic-based segmentation:** use OpenSearch with the **Neural plugin** (ML Commons connector to Bedrock, plus an ingest pipeline with a `text_embedding` processor). OpenSearch embeds documents at ingest and queries at search time. Run **one index per domain** or a **topic field plus filter**.
- **Scale on OpenSearch managed clusters:** primary shard count is **fixed at index creation**; replicas can change later. Target roughly **10–50 GiB per shard**. HNSW graphs live in **off-heap native memory**, so size the cluster for vector RAM or use **quantization / disk-based mode**. OpenSearch **Serverless** scales automatically in **OCUs**, and you don't manage shards.
- **Multi-index / hierarchical indexing:** keep separate indexes or KBs per domain, tenant or language. Use **routing** (agent, classifier or metadata) to choose the right index. Use a coarse summary index first, then a fine chunk index (two-stage retrieval).
- **Connectors:** KB has native connectors for **S3, Web Crawler, Confluence, SharePoint, Salesforce**, plus **custom data source** (direct ingestion API). Credentials go in **Secrets Manager**. Other sources: **Amazon Q Business / Kendra connectors**, **AppFlow** (SaaS → S3), **DataSync/Transfer Family** (file shares → S3), **Lambda** (APIs/wikis → S3), **OpenSearch Ingestion** (S3/DynamoDB/Kafka → OpenSearch).
- **Freshness:** an **ingestion job (StartIngestionJob)** is an **incremental** sync of adds, changes and deletes. Event-driven pattern: S3 event → **EventBridge → SQS (buffer/debounce) → Lambda → StartIngestionJob**. Scheduled pattern: **EventBridge Scheduler**. Near-real-time pattern: **IngestKnowledgeBaseDocuments / DeleteKnowledgeBaseDocuments** (direct ingestion, no full sync).
- **Zero-downtime full rebuild:** build a new index (blue/green), then **swap an OpenSearch alias** or point the app at the new KB ID. Use deterministic chunk IDs for idempotent **upserts**, and delete stale chunks when a document changes or is removed.

## Notes

### Skill 1.4.1 — Advanced vector database architectures for FM augmentation

> *Create advanced vector database architectures specifically for FM augmentation to enable efficient semantic retrieval beyond traditional search capabilities (for example, by using Amazon Bedrock Knowledge Bases for hierarchical organization, Amazon OpenSearch Service with the Neural plugin for Amazon Bedrock integration for topic-based segmentation, Amazon RDS with Amazon S3 document repositories, Amazon DynamoDB with vector databases for metadata and embeddings).*

#### Why a vector DB is "beyond traditional search"
- Traditional search (BM25, SQL `LIKE`, inverted indexes) matches **terms**. A vector store matches **meaning**: query and documents are embedded into the same vector space, and the nearest neighbours (k-NN/ANN) are returned.
- FM augmentation needs more from a store than "find similar text":
  - **Chunk text stored with the vector**, so it can go straight into the prompt
  - **Source pointers** (S3 URI, page, URL) for citations and traceability
  - **Metadata** for filtering by tenant, ACL, date or topic
  - **Low latency** at the retrieval step, because it sits on the user's critical path
  - **Freshness**, so the FM doesn't answer from stale facts
- Design question: **"Which store, how is it organised, and what's around it?"** (catalog, metadata, sync, access control).

#### Vector store options on AWS — decision table

| Store | Strengths | Limits / gotchas | Choose when |
| --- | --- | --- | --- |
| **Bedrock KB + OpenSearch Serverless** (vector search collection) | Fully managed, **hybrid search** (semantic + keyword), ms latency, scales in OCUs, KB quick-create | Minimum OCU baseline cost even when idle (verify current minimums); no shard control; metadata filtering on a KB index needs **FAISS** engine | Default production RAG with hybrid search and high QPS |
| **Bedrock KB + S3 Vectors** | **Lowest cost** (up to ~90% cheaper), pay-per-use, huge scale, KB quick-create | **Semantic only** (no hybrid), subsecond rather than ms latency; filterable metadata **2 KB**, total **40 KB** per vector | Large or archival corpora, cost-sensitive, latency-tolerant |
| **OpenSearch Service (managed cluster)** | Full control: shards, instance types, engines (FAISS/Lucene), **Neural plugin**, search pipelines, quantization, disk-based vectors | You run capacity, upgrades and shard design | Very large scale, custom tuning, existing OpenSearch estate |
| **Aurora PostgreSQL / RDS for PostgreSQL + pgvector** | SQL + vectors in one DB; **joins** with relational metadata/ACL tables; HNSW and IVFFlat indexes; KB-supported (Aurora) | Vertical scaling limits; you tune indexes; KB needs a specific table schema | Team already on Postgres; transactional metadata; strong relational needs |
| **Neptune Analytics** | Graph + vector; **GraphRAG** in KB; multi-hop over entities and relationships | Graph modelling effort | Questions that need relationship traversal (root cause, compliance lineage) |
| **Amazon DocumentDB** (vector search) | JSON documents + vectors in a MongoDB-compatible store | Not a KB-native store (verify) | Existing DocumentDB apps |
| 🎯 **PQ23** **MemoryDB / ElastiCache (Valkey/Redis OSS) vector search** | In-memory, **single-digit-ms**, high throughput | RAM cost; dataset must fit in memory | Ultra-low latency, **semantic cache**, real-time recommendations |
| **DynamoDB** | Serverless key-value store for **metadata, document state, chat history** | **No native vector search** | Catalog and state alongside a separate vector store, or zero-ETL to OpenSearch |
| **Kendra GenAI index** | Managed enterprise search with many connectors; can back a KB | Kendra pricing model | Enterprise search with connectors, minimal ML work |
| Third-party: **Pinecone, MongoDB Atlas, Redis Enterprise Cloud** | KB-supported | External account, credentials in Secrets Manager | Org standard is already one of these |

#### Pattern A — Bedrock Knowledge Bases for hierarchical organisation
- "Hierarchical organisation" in KBs works at three levels:
  1. **Within documents:** **hierarchical chunking** creates **parent and child chunks**. The query matches small child chunks (precise), and the larger **parent chunk is returned** to the FM (context). Details are in [Task 1.5](05-task-1.5-retrieval-mechanisms.md).
  2. **Across documents:** a **folder/prefix structure** in S3 (`/hr/policies/…`, `/legal/contracts/…`) maps to **metadata attributes** (`department`, `doc_type`), which you filter on at retrieval.
  3. **Across the corpus:** **multiple data sources per KB** (up to 5 per KB, per existing notes; verify quotas), or **multiple KBs per domain**. A **Bedrock Agent** or router chooses which KB to query. Each KB can have its own chunking and parsing tuned to its content type.
- KB architecture (managed):

```
S3 / Confluence / SharePoint / Salesforce / Web / Custom
        │  (sync = ingestion job)
        ▼
 Parse (default | FM parser | Bedrock Data Automation)
        ▼
 Chunk (default | fixed | hierarchical | semantic | none | Lambda custom)
        ▼
 Embed (Titan Text Embeddings V2, Cohere Embed, …)
        ▼
 Vector store (OSS | S3 Vectors | Aurora | Neptune Analytics | …)
        ▲
 Retrieve / RetrieveAndGenerate  ←  app, Agent, Flow
```

- KB also supports **structured data stores** (for example **Amazon Redshift**, with natural language → SQL) and **GraphRAG with Neptune Analytics**. Use these when the "knowledge" is tables or relationships rather than prose.
- Key KB properties for exam scenarios: **citations by default**, **session context** for multi-turn, **Guardrails**, **reranking** and **query decomposition** at query time.

#### Pattern B — OpenSearch Service + Neural plugin + Bedrock for topic-based segmentation
- The **Neural plugin** (neural search) lets OpenSearch **call an embedding model itself**, at ingest and at query time. The app sends raw text, not vectors.
- Components:
  1. **ML Commons connector** to Bedrock (for example Titan Text Embeddings V2). The connector uses an IAM role that OpenSearch assumes (`bedrock:InvokeModel`).
  2. **Register and deploy** the remote model, which returns a `model_id`.
  3. **Ingest pipeline** with a `text_embedding` processor that maps `text` → `text_embedding`.
  4. **k-NN index** using that pipeline as `default_pipeline`.
  5. A **`neural` query** at search time. OpenSearch embeds the query text with the same model.
- **Topic-based segmentation** options:
  - **Index per topic/domain** (`kb-legal`, `kb-hr`, `kb-engineering`). Each has its own mappings and tuning, and you can drop or rebuild one topic independently. Route queries by intent.
  - **Single index + `topic` keyword field**, with a filter inside the k-NN query. Simpler to run, but noisy topics share the same graph.
  - Topic labels can come from **document source**, **Amazon Comprehend** (custom classification or topic modelling) or an **FM classifier** at ingest time.

```json
PUT /_ingest/pipeline/bedrock-embed
{
  "processors": [
    { "text_embedding": {
        "model_id": "<bedrock-titan-model-id>",
        "field_map": { "text": "text_embedding" } } }
  ]
}

PUT /kb-legal
{
  "settings": { "index.knn": true, "default_pipeline": "bedrock-embed",
                "number_of_shards": 3, "number_of_replicas": 1 },
  "mappings": { "properties": {
    "text":           { "type": "text" },
    "topic":          { "type": "keyword" },
    "doc_id":         { "type": "keyword" },
    "published_at":   { "type": "date" },
    "text_embedding": { "type": "knn_vector", "dimension": 1024,
      "method": { "name": "hnsw", "engine": "faiss", "space_type": "l2",
                  "parameters": { "m": 16, "ef_construction": 128 } } }
  } }
}

GET /kb-legal/_search
{ "query": { "neural": { "text_embedding": {
    "query_text": "termination clause notice period",
    "model_id": "<bedrock-titan-model-id>", "k": 10,
    "filter": { "term": { "topic": "employment" } } } } } }
```

- ⚠️ `dimension` must match the embedding model output (Titan V2: 1024 / 512 / 256, as configured).

#### Pattern C — Amazon RDS/Aurora with S3 document repositories
- **S3** holds originals (PDFs, DOCX, images), versioned, with lifecycle rules and encryption. It is the **source of truth**.
- **RDS/Aurora** holds the **document catalog**: `doc_id`, title, author, owner, department, classification, version, S3 URI, checksum, ACL groups, ingestion status. Relational joins and constraints suit governance data.
- **Vectors** live either:
  - in the **same Aurora PostgreSQL** database via **pgvector**, so one SQL query does `vector distance + JOIN on ACL/metadata`, or
  - in a separate vector store keyed by `doc_id`/`chunk_id`.
- Flow: upload to S3 → event → Lambda writes the catalog row → chunk and embed → write vectors with `doc_id` → at query time, filter by user groups (SQL) and rank by similarity.
- Aurora as a **KB vector store** requires a pre-created table with (verify exact names in the docs):
  - an `id` (uuid primary key), an `embedding vector(<dims>)` column, a `chunks text` column, a `metadata json` column, and optionally custom metadata columns for filtering
  - an **HNSW index** on the embedding column (for example `vector_cosine_ops`)
  - the **RDS Data API** enabled, with credentials in **Secrets Manager**

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE bedrock_kb.chunks (
  id uuid PRIMARY KEY,
  embedding vector(1024),
  chunks text,
  metadata json,
  department text          -- custom metadata column for filtering
);
CREATE INDEX ON bedrock_kb.chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON bedrock_kb.chunks USING gin (to_tsvector('simple', chunks)); -- text search
```

- pgvector index choice: **HNSW** gives better recall and latency but builds slower and uses more memory. **IVFFlat** builds faster with less memory, needs data present before building (lists), and has lower recall.

#### Pattern D — DynamoDB with vector databases for metadata and embeddings
- **DynamoDB** is the **fast, serverless metadata/state layer**: document registry, chunk → source mapping, ingestion checkpoints, content hashes for change detection, per-tenant config, **conversation history**.
- The **vector DB** (OpenSearch, S3 Vectors, Aurora) stores embeddings plus minimal filter metadata.
- Typical flow:
  1. Vector search returns `chunk_id`s.
  2. **BatchGetItem** on DynamoDB fetches rich metadata (title, URL, owner, freshness, permissions).
  3. The app filters, enriches and builds the prompt.
- Why split: vector stores have **metadata size limits** (for example S3 Vectors 2 KB filterable), and DynamoDB gives ms key lookups, **TTL**, **Streams** (which trigger re-embedding) and cheap updates without re-indexing vectors.
- **DynamoDB zero-ETL integration with OpenSearch Service** (via OpenSearch Ingestion) replicates table changes into OpenSearch, where an ingest pipeline can embed them. This gives vector search over DynamoDB data without custom ETL.

#### Architecture anti-patterns (exam distractors)
- Storing embeddings in DynamoDB and doing **brute-force similarity in Lambda**: doesn't scale and isn't ANN.
- Using **Amazon Kendra** (classic) or **CloudSearch** when the scenario says "semantic similarity over embeddings with Bedrock"; prefer KB or OpenSearch vector search.
- Using **RDS MySQL** for vectors when the scenario says pgvector; the extension is PostgreSQL.
- **Fine-tuning** an FM to "add knowledge that changes weekly"; that's a RAG plus freshness problem.

---

### Skill 1.4.2 — Metadata frameworks for search precision and context awareness 🎯 PQ40

> *Develop comprehensive metadata frameworks to improve search precision and context awareness for FM interactions (for example, by using S3 object metadata for document timestamps, custom attributes for authorship information, tagging systems for domain classification).*

> [!IMPORTANT]
> **🎯 Practice question 40 — KB searches too many content types and time ranges: slow and irrelevant, minimal change**
>
> - **Turn on metadata filtering in the existing Bedrock KB.** Attach metadata (content type, date, category) to each S3 document, and pass **filters at query time** (`equals`, `in`, `greaterThan` on a date…). The candidate set shrinks **before** vector scoring, so results are faster and more relevant.
> - ❌ **Moving to OpenSearch Service or Serverless:** re-platforming and re-indexing, when the KB already filters natively.
> - ❌ **A domain-adapted embedding model:** the question rules out fine-tuning, and the problem is **scope**, not embedding quality.
> - ⚠️ **Detail:** the answer key says "index S3 object metadata". In practice KBs read filterable metadata from the **`<file>.metadata.json` sidecar** (see the key points above), not from S3 user-defined metadata or tags. Same idea, but use the sidecar in the real setup, and re-sync after adding it.

#### What metadata does in RAG
- **Pre-filtering narrows the search space**, giving fewer but more relevant candidates and higher precision. ("2024 policy only", "only docs I'm allowed to see".)
- **Context for the FM:** inject `title`, `author`, `date` and `source` into the prompt so the model can say "according to the 2024 HR handbook…" and resolve conflicts (newer wins).
- **Ranking signals:** boost recency or authority (official > wiki draft).
- **Governance:** classification (public/internal/confidential), tenant ID, retention, PII flags.
- **Traceability:** citations back to source URI and version.

#### A metadata schema (design once, apply everywhere)

| Category | Example attributes | Source | Used for |
| --- | --- | --- | --- |
| **Temporal** | `created_at`, `last_modified`, `effective_date`, `expires_at`, `year` | S3 `Last-Modified`, `x-amz-meta-*`, source system | Recency filters, "as of" questions, expiry |
| **Authorship / provenance** | `author`, `owner_team`, `source_system`, `source_url`, `version`, `approved_by` | Custom attributes, CMS fields | Authority boost, citations, audit |
| **Domain classification** | `department`, `domain`, `product`, `doc_type` (policy/FAQ/runbook), `topic` | S3 tags, prefix, Comprehend/FM classifier | Topic filtering, routing to an index |
| **Access control** | `tenant_id`, `allowed_groups`, `classification` | IAM/IdP groups, source ACLs | Security filtering (never trust the FM to hide data) |
| **Language / locale** | `language`, `region` | Comprehend `DetectDominantLanguage` | Language-specific retrieval |
| **Structural** | `section`, `page`, `heading_path`, `chunk_index` | Parser | Context assembly, neighbour expansion |
| **Quality** | `review_status`, `confidence`, `content_hash` | Pipeline | Exclude drafts, detect changes |

- Use **controlled vocabularies** (enums) for filter fields (`department ∈ {hr, legal, eng}`); free text breaks `equals` filters.
- Normalise types: store dates as **numbers (epoch) or ISO dates**, depending on what the store's range filter supports. KB numeric filters (`greaterThan`) need **number** type, so `year: 2024` should be a number, not `"2024"`.

#### S3 object metadata vs object tags vs KB sidecar files

| Mechanism | Limits / behaviour | Role in RAG |
| --- | --- | --- |
| **System metadata** (`Last-Modified`, `Content-Type`, ETag) | Set by S3 | Timestamps for freshness, change detection (ETag/hash) |
| **User-defined metadata** `x-amz-meta-*` | Set at upload; **~2 KB** total; changing it needs a **copy** of the object | Authorship and timestamps carried with the object; **read by your pipeline** |
| **Object tags** | Up to **10 tags** per object; mutable; usable in IAM conditions, lifecycle and replication rules | Domain classification, data classification, lifecycle governance |
| **S3 Metadata (journal/inventory tables)** | Queryable metadata tables (Apache Iceberg/S3 Tables, via Athena) | Find which objects changed or have which tags at scale; drive sync jobs |
| **KB sidecar `<file>.metadata.json`** | Same prefix as the source file; **this is what Bedrock KB ingests** as filterable metadata (sidecar size limit; verify, about 10 KB) | Filterable attributes at retrieval |

- ⚠️ **Bedrock KB does not automatically read S3 user metadata or tags.** A pipeline step (Lambda on `ObjectCreated`) can read `x-amz-meta-*` and tags, then write the `.metadata.json` sidecar.
- ⚠️ After adding or changing sidecars, **re-sync** the data source.

```json
// s3://corp-kb/hr/leave-policy-2024.pdf.metadata.json
{
  "metadataAttributes": {
    "department": "hr",
    "doc_type": "policy",
    "author": "Jane Doe",
    "year": 2024,
    "effective_date": 20240401,
    "allowed_groups": ["all-staff", "hr"]
  }
}
```

- The newer sidecar format supports **typed values** and an **`includeForEmbedding`** flag, which embeds the attribute with the chunk text so it influences semantic matching (verify current syntax):

```json
{ "metadataAttributes": {
    "author": { "value": { "type": "STRING", "stringValue": "Jane Doe" },
                "includeForEmbedding": true } } }
```

- **Built-in KB metadata:** every chunk also carries system fields such as `x-amz-bedrock-kb-source-uri`, `x-amz-bedrock-kb-chunk-id` and `x-amz-bedrock-kb-data-source-id`. You can filter on the source URI, for example to limit retrieval to one document.
- **Connector metadata:** Confluence, SharePoint, Salesforce and Web Crawler sources automatically attach source fields (title, URL, author, dates; exact fields vary by connector).

#### Query-time filtering in Bedrock KB
- Use `retrievalConfiguration.vectorSearchConfiguration.filter` with `Retrieve` and `RetrieveAndGenerate`.
- Operators: `equals`, `notEquals`, `greaterThan`, `greaterThanOrEquals`, `lessThan`, `lessThanOrEquals`, `in`, `notIn`, `startsWith`, `listContains`, `stringContains`, combined with `andAll` / `orAll`. Some operators are **only supported on certain vector stores** (verify in docs).

```python
resp = bedrock_agent_runtime.retrieve(
    knowledgeBaseId="KB123",
    retrievalQuery={"text": "How many days of parental leave?"},
    retrievalConfiguration={"vectorSearchConfiguration": {
        "numberOfResults": 8,
        "filter": {"andAll": [
            {"equals": {"key": "department", "value": "hr"}},
            {"greaterThanOrEquals": {"key": "year", "value": 2024}},
            {"listContains": {"key": "allowed_groups", "value": "all-staff"}}
        ]}}})
```

- **Implicit metadata filtering:** the KB can use an FM to **derive filters from the natural-language query** ("2023 legal contracts" → `year=2023 AND department=legal`), given a schema describing the metadata keys. Use it when users won't select filters themselves.
- **Security filtering:** build the filter **server-side from the authenticated identity** (Cognito/IdP group claims), never from user-supplied input. Metadata ACL filtering is a **retrieval-time control**. Index or KB separation per tenant is a **stronger isolation** boundary.

#### Multi-tenant isolation choices

| Approach | Isolation | Cost / ops | When |
| --- | --- | --- | --- |
| **Metadata filter** (`tenant_id`) in a shared index | Logical; a bug in the filter leaks data | Cheapest | Many small tenants, same schema |
| **Index (or KB) per tenant** | Stronger; separate index and IAM resource | More objects to manage | Regulated tenants, distinct data shapes |
| **Account / collection per tenant** | Strongest | Highest | Strict compliance, noisy-neighbour concerns |

#### Metadata enrichment pipeline
- Lambda on `s3:ObjectCreated` does the following:
  - Read the object's `x-amz-meta-author` and `Last-Modified`, plus its tags (`GetObjectTagging`).
  - Run **Amazon Comprehend**: `DetectDominantLanguage`, entities, a custom classifier for `domain`, and `DetectPiiEntities` to flag PII.
  - Optionally use an **FM** (Bedrock) for a summary, keywords or topic labels.
  - Write the `.metadata.json` sidecar and update the catalog (DynamoDB/RDS).
- Use **AWS Glue Data Catalog** or the catalog table for schema governance of metadata keys. Data validation itself is covered in [Task 1.3](03-task-1.3-data-validation-processing.md).

---

### Skill 1.4.3 — High-performance vector database architectures at scale

> *Implement high-performance vector database architectures to optimize semantic search performance at scale for FM retrieval (for example, by using OpenSearch sharding strategies, multi-index approaches for specialized domains, hierarchical indexing techniques).*

#### Performance levers at a glance

| Lever | Improves | Cost |
| --- | --- | --- |
| Right **shard count/size** | Parallelism, balanced nodes | Too many shards → overhead |
| **Replicas** | Read throughput (QPS), availability | 2× or more storage and memory |
| **Metadata pre-filtering** | Smaller candidate set, precision | Needs good schema |
| **Multi-index per domain** | Smaller graphs, tailored settings | Routing logic |
| **Hierarchical (two-stage) indexing** | Latency on huge corpora | Extra index and pipeline |
| **Quantization** (byte/FP16, PQ, binary) | Memory ↓, cost ↓ | Some recall loss |
| **Lower dimensions** (Titan V2 1024→512/256) | Memory, latency | Small accuracy loss ([Task 1.5](05-task-1.5-retrieval-mechanisms.md)) |
| **HNSW params** (`m`, `ef_construction`, `ef_search`) | Recall vs latency trade | Memory/build time |
| **Disk-based vector search** | Very large corpora at low cost | Higher latency than in-memory |
| **Caching** (semantic cache in MemoryDB/ElastiCache) | Repeated queries | Staleness management |

#### OpenSearch sharding strategies (managed clusters)
- An index is split into **primary shards**, each with **replicas**. A query fans out to one copy of every shard, then results merge (top-k per shard → global top-k).
- ⚠️ The **number of primary shards is set at index creation**. Changing it needs **reindex**, **_split/_shrink**, or a new index plus an alias swap. **Replicas can change at any time.**
- Sizing heuristics (AWS guidance):
  - **Shard size about 10–30 GiB** for search-latency-sensitive workloads (up to ~50 GiB for write-heavy ones)
  - Make **shard count a multiple of the data-node count**, so load is even across nodes and AZs
  - Avoid oversharding: each shard costs heap and coordination overhead
- **Vector memory sizing (HNSW, FAISS)** follows the OpenSearch rule of thumb: `memory ≈ 1.1 × (4 × d + 8 × m) × num_vectors` bytes, multiplied by (1 + replicas).
  - Example: 10 M vectors, d = 1024, m = 16 → 1.1 × (4096 + 128) × 10 M ≈ **46 GB**. With 1 replica, ≈ 93 GB of **native (off-heap) memory** across the cluster.
  - The k-NN graphs load into **off-heap memory**, governed by a circuit-breaker limit (verify the default %). Breaching it means graphs get evicted and latency spikes.
  - Use **memory-optimised instances (r-family)** for vector-heavy data nodes; consider **OR1/OR2/OM2** instances (verify current names).
- **Reduce memory:**
  - **FAISS byte/FP16 scalar quantization**: 4 bytes → 1 or 2 bytes per dimension
  - **Product quantization (IVF-PQ)**: needs a training step
  - **Binary quantization**, and **disk-based vector search** (`mode: on_disk` with a compression level), which keeps compressed vectors in memory and **rescores** with full-precision vectors on disk
- **Write performance during bulk load:** set `refresh_interval: -1` and `number_of_replicas: 0`, bulk index, then restore both and **force-merge** (fewer segments means faster k-NN search).
- **Search tuning:** `ef_search` higher → better recall, slower. `k` and `size` larger → more memory per query. Filtered k-NN: FAISS supports **efficient filtering** (it picks pre-filter, exact search or ANN based on how restrictive the filter is).
- **Dedicated master nodes** (3, across AZs) and **multi-AZ with standby** for production resilience.
- **OpenSearch Serverless:** **no shard management**. Capacity is in **OCUs** (separate indexing and search OCUs), scales automatically, and you set **max OCU** limits for cost control. Use a **vector search collection** type. Trade-off: less tuning control than managed clusters.

#### Multi-index approaches for specialised domains
- **Why:**
  - Each domain gets its own **embedding model** (for example code vs legal text vs multilingual), **dimension**, **chunking**, **mapping** and **HNSW settings**.
  - A smaller graph per index gives faster search and higher recall.
  - You can **rebuild one domain** without touching others.
  - **Access control per index** (fine-grained access control / IAM data-access policies).
- **Routing options:**
  1. **Static:** UI or app context picks the index (the "HR assistant" app → `kb-hr`).
  2. **Classifier:** Comprehend custom classification, or a cheap FM, maps the query to a domain or index.
  3. **Agentic:** a Bedrock Agent or Strands agent with **one KB/tool per domain**; the model picks one, or several.
  4. **Fan-out + merge:** query several indexes in parallel (Lambda / Step Functions Parallel), normalise scores, then **rerank** ([Task 1.5](05-task-1.5-retrieval-mechanisms.md)).
- **Index aliases:** the app queries `kb-hr` (an alias) → `kb-hr-v7`. This enables blue/green rebuilds (Skill 1.4.5).
- KB equivalent: **one KB per domain** (each with its own vector index), or one KB with **multiple data sources** plus a `domain` metadata filter.

#### Hierarchical indexing techniques
- **Parent–child (small-to-big):** index small chunks for matching and return the parent section. In KB this is **hierarchical chunking**; in OpenSearch you can use `nested` fields or a `parent_id` lookup.
- **Two-stage / coarse-to-fine:**
  - **Level 1:** a **document-summary index** (one vector per document, built from an FM-generated summary).
  - **Level 2:** a **chunk index** filtered to the top-N document IDs from level 1.
  - This cuts the search space on huge corpora and reduces "many near-duplicate chunks from irrelevant documents".
- **Taxonomy hierarchy:** filter on `domain` → `sub_domain` → `doc_type` metadata before vector search.
- **Graph hierarchy:** **GraphRAG (Neptune Analytics)** for entity → relationship → document hierarchies and multi-hop questions.
- The **IVF** ANN method is itself a hierarchical idea (cluster centroids → buckets), and **HNSW** is a layered proximity graph. Engine detail is in the existing note [06 · Vector search & OpenSearch](<../../study-notes/Certifications/AWS/AIP-C01 - Gen AI Pro/06-vector-search-opensearch.md>).

#### Monitoring vector store performance
- **OpenSearch CloudWatch metrics:** `SearchLatency`, `SearchRate`, `IndexingLatency`, `JVMMemoryPressure`, `CPUUtilization`, `FreeStorageSpace`, **k-NN metrics** (`KNNGraphMemoryUsage`, `KNNCacheCapacityReached`, `KNNHitCount`/`MissCount`; verify names), `ThreadpoolSearchRejected`.
- **Serverless:** `SearchOCU`, `IndexingOCU`, latency metrics.
- **Aurora pgvector:** Performance Insights, `EXPLAIN ANALYZE` to confirm the HNSW index is used, `hnsw.ef_search` tuning.
- Track **retrieval quality** too (recall@k, MRR with a labelled set) after every scaling change. Quantization or lower dimensions can silently cut recall (evaluation is in Domain 5).

---

### Skill 1.4.4 — Integration components for document systems, knowledge bases and wikis

> *Use AWS services to create integration components to connect with resources (for example, document management systems, knowledge bases, internal wikis for comprehensive data integration in GenAI applications).*

#### Bedrock Knowledge Bases data source connectors

| Connector | Typical source | Auth / config | Notes |
| --- | --- | --- | --- |
| **Amazon S3** | Exported docs, data lake, DMS exports | IAM (KB service role); cross-account via bucket policy; KMS key grant if CMK-encrypted | Most common; inclusion prefixes; `.metadata.json` sidecars |
| **Web Crawler** | Public or internal websites, docs portals | Seed URLs, **scope** (same host / subdomains), include/exclude **regex**, crawl rate | Respects robots.txt; crawls only permitted URLs |
| **Confluence** (Cloud) | Internal wiki spaces and pages | Basic auth (API token) or **OAuth 2.0**; credentials in **Secrets Manager** | Include/exclude spaces, page types, attachments |
| **SharePoint** (Online) | Document libraries, sites | **Entra ID (Azure AD) app** credentials in Secrets Manager (OAuth) | Site URLs, include/exclude patterns |
| **Salesforce** | Knowledge articles, cases, objects | OAuth (connected app) in Secrets Manager | Choose standard and custom objects |
| **Custom data source** | Anything your code can read | **Direct ingestion APIs**: `IngestKnowledgeBaseDocuments`, `DeleteKnowledgeBaseDocuments` | Push text or S3 references in real time, without a sync job |

- Several connectors were in **preview** at launch; check current GA status and supported auth types.
- ⚠️ Connector credentials go **in AWS Secrets Manager**, never in code or environment variables. The KB service role needs `secretsmanager:GetSecretValue` (and `kms:Decrypt` if the secret uses a CMK).
- **Private networking:** KB connectors reach SaaS endpoints over the internet. For on-premises or VPC-only systems, **export to S3** first (DataSync, Lambda in VPC, AWS Transfer Family). Use **VPC endpoints (PrivateLink)** for Bedrock, S3 and OpenSearch to keep traffic off the public internet.

#### Other integration building blocks

| Need | Service | Pattern |
| --- | --- | --- |
| SaaS apps (Salesforce, ServiceNow, Zendesk, Slack…) → S3 | **Amazon AppFlow** | Scheduled or event flows into S3, then KB sync |
| On-prem file shares / NAS (SMB/NFS) → S3 | **AWS DataSync** | Scheduled incremental transfers |
| SFTP drops from partners | **AWS Transfer Family** | Land files in S3 → event → ingest |
| 40+ enterprise connectors (ServiceNow, Jira, Box, Google Drive, Slack…) with ACL awareness | **Amazon Q Business** connectors / **Amazon Kendra** connectors (Kendra GenAI index can back a KB) | When the main need is enterprise search with **document-level ACLs** from the source |
| REST APIs / internal wikis without a connector | **Lambda** (+ EventBridge Scheduler) | Pull via API → normalise to text/markdown + metadata → S3 or direct ingestion |
| Relational DB content | **AWS Glue** jobs, **DMS** → S3; or **KB structured data store (Redshift)** for NL → SQL | Prose columns → RAG; tabular facts → SQL retrieval |
| Streams (Kafka/MSK, Kinesis) | **OpenSearch Ingestion** pipelines, Lambda | Near-real-time embedding into OpenSearch |
| DynamoDB tables | **DynamoDB zero-ETL → OpenSearch Service** | CDC replication + ingest pipeline embedding |
| Complex documents (scans, tables, forms) | **Bedrock Data Automation**, **Textract**, KB FM parser | Parse before chunking (details in [Task 1.3](03-task-1.3-data-validation-processing.md)) |
| Exposing retrieval to agents and tools | **MCP server** wrapping KB Retrieve, AgentCore Gateway, action groups | Standard access layer ([Task 1.5](05-task-1.5-retrieval-mechanisms.md), Skill 1.5.6) |

#### Reference integration architecture (custom, multi-source)

```
Confluence ─┐                         ┌─► DynamoDB (catalog, hashes, status)
SharePoint ─┤  native KB connectors   │
Salesforce ─┘ ───────────────────────►│
                                       │
Jira/ServiceNow ─► AppFlow ─► S3 raw ─►│ EventBridge ─► SQS ─► Lambda
Internal wiki API ─► Lambda (scheduled)│   (normalise → markdown + metadata.json)
File shares ─► DataSync ─► S3 raw ─────┘            │
                                                    ▼
                                         S3 curated ─► KB data source ─► sync
                                                    │
                                    Bedrock KB (embed) ─► OpenSearch Serverless
```

- **Normalise early:** convert HTML, wiki markup and DOCX to **clean text/markdown**; strip navigation chrome; keep headings (for structural chunking); attach metadata (source URL, author, last-modified, space/site).
- **Preserve source ACLs:** map source permissions (Confluence space groups, SharePoint groups) into `allowed_groups` metadata, then apply identity-based filters at query time (Skill 1.4.2). Amazon Q Business handles source ACL sync natively; with a KB you design it yourself.
- **Idempotency:** key every document by `source_system + source_id`, and store a `content_hash`. Unchanged hash means skip.
- **Error handling:** SQS DLQ for failed docs, retries with backoff for source API throttling, CloudWatch alarms on DLQ depth and ingestion failures.

#### Security for integration components
- 🎯 **PQ14** **Least-privilege KB service role:** `s3:GetObject`/`ListBucket` on specific prefixes, `bedrock:InvokeModel` on the embedding model, vector store data-plane permissions (`aoss:APIAccessAll` plus a **data access policy** for OpenSearch Serverless; `s3vectors:*Vectors` on the index ARN for S3 Vectors; `rds-data:*` plus `secretsmanager:GetSecretValue` for Aurora).
- **Encryption:** KMS CMKs for the S3 source, vector store, KB and ingestion transient data.
- **Network:** OpenSearch Serverless **network policy** (VPC endpoint only vs public), and VPC endpoints for Bedrock runtime.
- **Audit:** CloudTrail logs KB management calls, `StartIngestionJob` and `Retrieve`.

> [!WARNING]
> **🎯 Practice question 14 — Lambda gets 400 from BedrockAgentRuntime + 403 from OpenSearch Serverless (choose 2)**
>
> Both errors come from **missing permissions on the Lambda execution role** and a **missing OpenSearch Serverless data access policy**:
> - ✅ **IAM on the Lambda role:** add the Bedrock runtime action it calls (**`bedrock:InvokeAgent`**; for KB calls, `bedrock:Retrieve` / `bedrock:RetrieveAndGenerate`). Add **`aoss:APIAccessAll`**, the IAM permission that lets a principal use the OpenSearch Serverless **data plane**.
> - ✅ **OpenSearch Serverless data access policy with pattern-based resource rules**, for example `"Resource": ["collection/invest-*"]` and `["index/invest-*/*"]`. Permissions then **apply to new collections and indexes automatically** that match the pattern, so this scales.
> - AOSS needs **both**: the IAM permission (`aoss:APIAccessAll`) **and** a data access policy that names the principal. With only one of them you get **403**.
>
> Traps:
> - ❌ **`es:ESHttp*`** applies only to **managed OpenSearch Service domains**. OpenSearch Serverless uses the **`aoss:*`** namespace.
> - ❌ **Secrets Manager credentials:** OpenSearch Serverless doesn't use usernames or passwords. It uses **IAM + data access policies** only.
> - ❌ **VPC endpoint policy:** controls network access, not authorisation, so neither the 400 nor the 403 goes away.

---

### Skill 1.4.5 — Data maintenance systems to keep vector stores current and accurate

> *Design and deploy data maintenance systems to ensure that vector stores contain current and accurate information for FM augmentation (for example, by using incremental update mechanisms, real-time change detection systems, automated synchronization workflows, scheduled refresh pipelines).*

#### Why it matters
- Stale vectors lead the FM to answer confidently from **outdated or deleted** content. This looks like hallucination but is a data problem.
- Deleted source documents that remain as vectors are a **compliance risk** (right to erasure, revoked access).
- Changed embedding models or chunking leave **mixed vector spaces**. Don't mix: re-embed everything.

#### Bedrock KB sync mechanics
- **Ingestion job** (`StartIngestionJob`, console **Sync**):
  - **Incremental**: it processes only **added, modified or deleted** files since the last sync, and removes vectors for deleted files.
  - Steps: list source → detect changes → parse → chunk → embed → write/delete in the vector store.
  - Status via `GetIngestionJob` / `ListIngestionJobs`, with statistics (scanned, indexed, modified, deleted, failed).
  - ⚠️ **One running ingestion job per data source at a time.** Starting another returns a conflict error, so **serialise or debounce** triggers.
  - Logs: enable KB **log delivery** (CloudWatch Logs / S3 / Firehose) to see per-document failures (unsupported file, too large, parsing errors).
- **Direct ingestion** (`IngestKnowledgeBaseDocuments`, `DeleteKnowledgeBaseDocuments`, `GetKnowledgeBaseDocuments`, `ListKnowledgeBaseDocuments`):
  - Add, update or delete **specific documents** immediately, without scanning the whole source.
  - Works for **custom** data sources (inline text or S3 location) and S3 data sources. For S3 sources, the S3 content must also reflect the change, or the next full sync will revert it (verify behaviour).
  - Best for **near-real-time** freshness (seconds to minutes).
- **Data deletion policy** on a data source: **Delete** (default; vectors are removed when the data source is deleted) vs **Retain**.

#### Pattern 1 — Event-driven (real-time change detection)

```
S3 PutObject/DeleteObject ─► EventBridge rule ─► SQS (batch window, e.g. 5 min)
                                                    │
                                                    ▼
                                    Lambda: if no job RUNNING → StartIngestionJob
                                            else → leave msgs / retry later
```

- **S3 Event Notifications → EventBridge** for `Object Created` / `Object Deleted`.
- **SQS buffer** absorbs bursts (1,000 uploads should give one sync, not 1,000). Use the Lambda event source mapping **batch window** or an EventBridge Scheduler debounce.
- Alternative for per-document latency: Lambda calls **`IngestKnowledgeBaseDocuments`** or **`DeleteKnowledgeBaseDocuments`** for that object.
- Change detection on other sources:

| Source | Change signal |
| --- | --- |
| S3 | Event Notifications / EventBridge; **S3 Metadata journal tables** |
| DynamoDB | **DynamoDB Streams** → Lambda (re-embed changed item) or zero-ETL → OpenSearch |
| Aurora/RDS | **DMS CDC** → Kinesis/S3, or app-level outbox table |
| Confluence / SharePoint / Salesforce | KB connector incremental sync (scheduled); source webhooks → API Gateway → Lambda for faster updates |
| 🎯 **PQ64** Websites | Scheduled web crawler re-sync; sitemap `lastmod` |
| Kafka / Kinesis | OpenSearch Ingestion pipeline (streaming upserts) |

> [!IMPORTANT]
> **🎯 Practice question 64 — replace the FAQ with an AgentCore agent over 700k forum posts (+1,500 a day), refresh daily, never surface PII**
>
> - **Bedrock KB on OpenSearch Serverless** with a **web crawler data source** (scheduled daily sync) → the agent uses **RAG**.
> - **Guardrails sensitive information filters** on the agent → **mask or block PII** in responses.
> - ❌ **Custom Step Functions/Lambda/SQS/Comprehend crawler + putting the data in the prompt:** heavy custom work, and 700k posts can't fit in a context window.
> - ❌ **S3 Vectors + Macie + Lambda redaction:** per the answer key, Macie **can't scan S3 vector buckets** and S3 Vectors doesn't support the web crawler (verify in docs).
> - ❌ **Aurora + "prompt the model to block PII":** prompts can't reliably enforce PII rules. The answer key also says Aurora doesn't support the web crawler (verify in docs).
> - ⚠️ In the KB docs, data sources usually aren't tied to the vector store, so check current compatibility. For the exam: **web crawler + OpenSearch Serverless + Guardrails PII**.

#### Pattern 2 — Scheduled refresh pipelines
- **EventBridge Scheduler** (cron) → Lambda or **Step Functions** → `StartIngestionJob` for each data source → poll `GetIngestionJob` (Wait/Choice loop) → on failure, SNS alert or retry.
- Use for connector-based sources (Confluence, SharePoint, Web Crawler) and for nightly reconciliation.
- Step Functions workflow example:
  1. **Extract:** AppFlow/Lambda pulls changed records since the `last_watermark` stored in DynamoDB.
  2. **Validate:** Glue Data Quality or Lambda checks (non-empty, size, language, PII) ([Task 1.3](03-task-1.3-data-validation-processing.md)).
  3. **Write** to S3 curated, plus sidecars.
  4. **Sync** with `StartIngestionJob` and wait for completion.
  5. **Verify:** run a smoke-test query set (golden questions) through `Retrieve`, and check expected doc IDs come back.
  6. **Publish metrics** (CloudWatch: docs indexed, failures, freshness lag) and update the watermark.

#### Pattern 3 — Incremental upserts in self-managed stores (OpenSearch / pgvector)
- **Deterministic IDs:** `chunk_id = hash(doc_id + chunk_index + version)`, or `doc_id#chunk_n`. Re-ingestion **overwrites** in place (idempotent).
- **On document update:** chunk counts can change, so **delete all chunks for `doc_id`** (`_delete_by_query` / `DELETE WHERE doc_id=`) and then insert the new ones. Or keep a version field and filter `is_current = true`.
- **Content hashing:** skip unchanged docs (compare `sha256(text)` with the catalog). This saves embedding cost.
- **Soft delete / TTL:** add `expires_at` metadata and filter it out; a scheduled purge job removes the vectors.
- **Batching embeddings:** use Bedrock **batch inference** or batched Lambda calls for large backfills. Respect `InvokeModel` throughput quotas and use backoff on `ThrottlingException`.

#### Pattern 4 — Full rebuild (re-index) with zero downtime
- Triggers: **new embedding model** or dimension, a chunking change, a mapping/shard change, corruption.
- **Blue/green:**
  - **OpenSearch:** build `kb-hr-v8` in the background → validate (recall on a golden set) → **atomic alias swap** `kb-hr` → v8 → delete v7 later.
  - **Bedrock KB:** create a **new KB** (because embedding model, chunking and vector store are immutable) → ingest → evaluate → switch the KB ID in config (**AppConfig** / Parameter Store) → retire the old KB, its vector index and its role.
- ⚠️ **Never mix embeddings from two models in one index.** Query vectors from model B are meaningless against document vectors from model A.

#### Data accuracy and quality controls
- **Freshness SLA metric:** `now − last_successful_sync`, and per-document `source_last_modified − indexed_at`. Alarm when it exceeds the SLA.
- **Reconciliation job:** compare the source inventory (S3 Inventory/S3 Metadata, connector listings) with the vector store's document IDs to find **orphans** (deleted at source, still indexed) and **misses**.
- **Conflict handling:** keep `effective_date` / `version` metadata, filter to the latest version, or tell the FM in the prompt that **newest wins**.
- **Ingestion failure triage:** unsupported formats, files over the size limit (KB S3 files **under 50 MB**), encrypted files without a KMS grant, **dimension mismatch**, throttling on the embedding model. Find them in KB logs, `GetIngestionJob` failure reasons and the DLQ.
- **Right to erasure / revoked access:** delete at the source, then sync or call `DeleteKnowledgeBaseDocuments`, and verify with a `Retrieve` that filters on the source URI. Also purge caches (semantic cache, conversation memory).

#### Freshness strategy decision table

| Requirement | Strategy |
| --- | --- |
| "Changes visible within seconds/minutes" | **Direct ingestion API** per document, or OpenSearch Ingestion / streaming upserts |
| "Daily/nightly is fine" | **EventBridge Scheduler → StartIngestionJob** |
| "Bursty uploads, avoid duplicate syncs" | **S3 → EventBridge → SQS batch → Lambda** (single job, check for a running job first) |
| "SaaS wiki changes" | Native connector + scheduled incremental sync (+ webhooks for urgent updates) |
| "Changed embedding model" | **Full re-index into a new KB/index + blue/green switch** |
| "Deleted docs still appearing in answers" | Sync after delete / `DeleteKnowledgeBaseDocuments`; check data deletion policy; reconcile orphans |
| "Avoid re-embedding unchanged content" | **Content hash** in a DynamoDB catalog; skip unchanged |

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Fully managed RAG store with citations, minimal ops | **Bedrock Knowledge Bases** (quick-create vector store) |
| Need hybrid (keyword + semantic) search with ms latency at high QPS | KB on **OpenSearch Serverless** |
| Billions of vectors, archive, lowest cost, subsecond OK | KB on **Amazon S3 Vectors** |
| Team already runs PostgreSQL, wants SQL joins with ACL tables | **Aurora PostgreSQL + pgvector** |
| Multi-hop questions over entities and relationships | **GraphRAG with Neptune Analytics** |
| OpenSearch should call Bedrock to embed at ingest and query time | **Neural plugin**: ML Commons connector + ingest pipeline (`text_embedding`) + `neural` query |
| Separate legal/HR/engineering content with different tuning | **Index (or KB) per domain** + query routing |
| Store doc metadata, ingestion status and content hashes cheaply with ms lookups | **DynamoDB** catalog keyed by `doc_id` alongside the vector store |
| Vector search over existing DynamoDB data without custom ETL | **DynamoDB zero-ETL integration → OpenSearch Service** |
| Filter KB results by year/department/author | **`.metadata.json` sidecar** + `retrievalConfiguration` filter (re-sync after adding) |
| Users won't pick filters; infer them from the question | **Implicit metadata filtering** in KB |
| Uploaded S3 objects have `x-amz-meta-author`, but the KB filter doesn't work | KB doesn't read S3 user metadata → **generate the sidecar** via Lambda |
| Prevent users seeing docs from other teams in RAG answers | **Identity-derived metadata filter** (server-side) or **per-tenant index/KB** |
| OpenSearch latency spikes, k-NN graphs evicted | Out of **off-heap memory**: scale memory-optimised nodes, **quantize**, reduce dims, disk-based mode |
| Need more primary shards on an existing index | **Reindex / `_split`** into a new index + **alias swap** (primaries fixed at creation) |
| Read QPS too low on OpenSearch cluster | Add **replicas** (and nodes) |
| Huge corpus, many near-duplicate chunks from irrelevant docs | **Two-stage hierarchical index** (doc-summary index → chunk index) or hierarchical chunking |
| Index a Confluence wiki / SharePoint library into RAG | **KB native connector**, credentials in **Secrets Manager** |
| Ingest Jira/ServiceNow/Zendesk data | **AppFlow → S3** → KB, or **Q Business/Kendra connectors** |
| On-prem NFS/SMB documents into RAG | **DataSync → S3** → KB sync |
| Must preserve source document ACLs automatically across 40+ SaaS sources | **Amazon Q Business** (or Kendra) connectors with ACL sync |
| New S3 docs should be searchable within minutes, uploads are bursty | **S3 → EventBridge → SQS (batch) → Lambda → StartIngestionJob** |
| Second sync fails with a conflict while one is running | **One ingestion job per data source at a time** → serialise/debounce |
| Push individual docs into a KB in real time from an app | **`IngestKnowledgeBaseDocuments`** (custom data source / direct ingestion) |
| Nightly refresh of a wiki/web crawl | **EventBridge Scheduler → StartIngestionJob** (Step Functions to wait + verify) |
| Switching to a new embedding model | **New KB / new index**, re-embed everything, **blue/green switch** (never mix models) |
| Deleted documents still show up in answers | **Sync** (incremental deletes) or `DeleteKnowledgeBaseDocuments`; reconcile orphans; clear caches |
| Avoid paying to re-embed unchanged documents | **Content hash** check (DynamoDB catalog) before embedding |
| KB ingestion fails for every document after setup | **Index dimension ≠ embedding dimension**, or missing KMS/S3 permissions |

## Flashcards

<details>
<summary><b>Q:</b> Which four KB settings can't be changed after creation, and what do you do if one must change?</summary>

The parser, chunking strategy, embedding model and vector store. Create a new KB (or new data source/index), re-ingest, then switch traffic blue/green.

*D1 · Task 1.4 · KB immutability*
</details>

<details>
<summary><b>Q:</b> When do you choose S3 Vectors over OpenSearch Serverless as the KB vector store?</summary>

When cost matters most and the corpus is large or archival, subsecond latency is acceptable, and semantic-only search is enough. Choose OpenSearch Serverless for hybrid search, ms latency or high QPS.

*D1 · Task 1.4 · Store selection*
</details>

<details>
<summary><b>Q:</b> Does DynamoDB support native vector similarity search? How is it used in a RAG architecture?</summary>

No. It serves as the metadata/state layer (doc catalog, chunk mapping, hashes, chat history) next to a vector store, or its data is replicated to OpenSearch via zero-ETL for vector search.

*D1 · Task 1.4 · DynamoDB role*
</details>

<details>
<summary><b>Q:</b> What four components let OpenSearch embed text with Bedrock automatically?</summary>

An ML Commons connector to Bedrock (with an IAM role), a registered and deployed remote model (`model_id`), an ingest pipeline with a `text_embedding` processor set as the index default pipeline, and a `neural` query at search time.

*D1 · Task 1.4 · Neural plugin*
</details>

<details>
<summary><b>Q:</b> Name two ways to do topic-based segmentation in OpenSearch, with a trade-off for each.</summary>

Index per topic: isolated tuning, rebuilds and access, but needs routing. Single index with a `topic` keyword filter: simpler to run, but topics share one graph and settings.

*D1 · Task 1.4 · Topic segmentation*
</details>

<details>
<summary><b>Q:</b> In an RDS/Aurora + S3 document repository design, what lives where?</summary>

S3 holds the original documents (system of record). RDS/Aurora holds the catalog, metadata and ACLs (and vectors via pgvector, or a separate store keyed by `doc_id`).

*D1 · Task 1.4 · RDS + S3 pattern*
</details>

<details>
<summary><b>Q:</b> How do you attach filterable metadata to an S3 document in a Bedrock KB?</summary>

Put a `<filename>.metadata.json` sidecar with `metadataAttributes` next to the file, then re-sync the data source.

*D1 · Task 1.4 · Metadata sidecar*
</details>

<details>
<summary><b>Q:</b> Documents carry `x-amz-meta-author`, but KB filters on `author` return nothing. Why?</summary>

KBs don't ingest S3 user-defined metadata or tags. Generate a `.metadata.json` sidecar from them (for example with Lambda on ObjectCreated) and re-sync.

*D1 · Task 1.4 · S3 metadata gotcha*
</details>

<details>
<summary><b>Q:</b> A filter `year greaterThan 2022` doesn't work. What's a likely cause?</summary>

The attribute was stored as a string ("2023"). Range operators need numeric types.

*D1 · Task 1.4 · Metadata typing*
</details>

<details>
<summary><b>Q:</b> What does implicit metadata filtering do in Bedrock KB?</summary>

An FM derives metadata filters from the natural-language query, using a described metadata schema. For example, "2023 legal contracts" becomes year = 2023 AND department = legal.

*D1 · Task 1.4 · Implicit filtering*
</details>

<details>
<summary><b>Q:</b> How should ACL-based metadata filters be built to be secure?</summary>

Server-side, from the authenticated identity's group claims, never from user input. Use per-tenant indexes or KBs for stronger isolation.

*D1 · Task 1.4 · Secure filtering*
</details>

<details>
<summary><b>Q:</b> Can you change the number of primary shards on an existing OpenSearch index?</summary>

No. It is fixed at creation. Reindex or `_split` into a new index and swap an alias. Replica count can change at any time.

*D1 · Task 1.4 · Sharding*
</details>

<details>
<summary><b>Q:</b> Where do OpenSearch HNSW graphs live, and what happens when that memory runs out?</summary>

In off-heap native memory. Beyond the circuit-breaker limit, graphs are evicted or loads fail, and latency spikes. Fixes: memory-optimised nodes, quantization, fewer dimensions, disk-based mode.

*D1 · Task 1.4 · Vector memory*
</details>

<details>
<summary><b>Q:</b> Approximately how much memory do 10 M HNSW vectors of 1024 dims with m=16 need (no replicas)?</summary>

About 1.1 × (4×1024 + 8×16) × 10 M ≈ 46 GB of native memory, doubled with one replica.

*D1 · Task 1.4 · Capacity sizing*
</details>

<details>
<summary><b>Q:</b> What is two-stage (coarse-to-fine) hierarchical indexing?</summary>

Search a document-summary index first to pick the top-N documents, then run vector search over the chunk index filtered to those document IDs. This shrinks the search space and noise.

*D1 · Task 1.4 · Hierarchical indexing*
</details>

<details>
<summary><b>Q:</b> Which KB data source connectors are native, and where do their credentials go?</summary>

S3, Web Crawler, Confluence, SharePoint, Salesforce, plus custom (direct ingestion). Credentials go in AWS Secrets Manager.

*D1 · Task 1.4 · Connectors*
</details>

<details>
<summary><b>Q:</b> How do you bring on-prem file shares and Zendesk tickets into a KB?</summary>

DataSync (file shares → S3) and AppFlow (Zendesk → S3), then KB S3 data source sync.

*D1 · Task 1.4 · Integration*
</details>

<details>
<summary><b>Q:</b> Is a KB ingestion job a full re-index?</summary>

No. It's incremental: only added, modified or deleted files are processed, and deleted files' vectors are removed.

*D1 · Task 1.4 · Incremental sync*
</details>

<details>
<summary><b>Q:</b> Thousands of S3 uploads trigger a Lambda that calls StartIngestionJob, and many calls fail. Why, and what's the fix?</summary>

Only one ingestion job can run per data source at a time. Buffer events in SQS with a batch window and start a job only if none is running.

*D1 · Task 1.4 · Debounced sync*
</details>

<details>
<summary><b>Q:</b> How can an app make a single document searchable in a KB immediately, without a full sync?</summary>

Call `IngestKnowledgeBaseDocuments` (and `DeleteKnowledgeBaseDocuments` for removals). This is direct ingestion.

*D1 · Task 1.4 · Direct ingestion*
</details>

<details>
<summary><b>Q:</b> How do you update a changed document in a self-managed OpenSearch vector index without leaving stale chunks?</summary>

Delete all chunks for the `doc_id` (or mark the old version not current), then insert new chunks with deterministic IDs. Skip unchanged docs using a content hash.

*D1 · Task 1.4 · Upserts*
</details>

<details>
<summary><b>Q:</b> How do you switch to a new embedding model with zero downtime?</summary>

Build a new index or KB with the new model, re-embed everything, validate on a golden set, then swap the alias or KB ID (AppConfig). Never mix models in one index.

*D1 · Task 1.4 · Blue/green re-index*
</details>

<details>
<summary><b>Q:</b> Users still get answers from a policy that was deleted last week. What do you check?</summary>

Whether a sync ran after the delete (or `DeleteKnowledgeBaseDocuments` was called), the data deletion policy, orphan reconciliation against the source inventory, and stale semantic caches.

*D1 · Task 1.4 · Stale data*
</details>

<details>
<summary><b>Q:</b> Which metric best signals vector store freshness, and how do you alert on it?</summary>

Freshness lag: now − last successful sync, or source last-modified − indexed-at. Publish it as a CloudWatch custom metric with an alarm at the SLA threshold.

*D1 · Task 1.4 · Freshness monitoring*
</details>

---
[← Back to Domain 1](README.md)
