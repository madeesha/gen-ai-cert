[← Back to Domain 1](README.md)

# 🔎 Task 1.5 · Design retrieval mechanisms for FM augmentation

> **Exam domain:** D1 · Foundation Model Integration, Data Management, and Compliance (31%)
>
> **Skills covered**
> - **1.5.1** Document segmentation (chunking) for retrieval performance
> - **1.5.2** Embedding model selection and configuration
> - **1.5.3** Deploying and configuring vector search
> - **1.5.4** Advanced search architectures: semantic, hybrid, reranking
> - **1.5.5** Query handling: expansion, decomposition, transformation
> - **1.5.6** Consistent access mechanisms: function calling, MCP, standard retrieval APIs
>
> Where the vector store lives, metadata design, sharding, connectors and keeping data fresh are covered in [Task 1.4 · Vector store solutions](04-task-1.4-vector-store-solutions.md). This file is about **how content gets split, embedded, searched, ranked and served to the FM**.

## Key points
- **Retrieval quality limits answer quality.** If the right chunk never reaches the context window, no prompt or model will fix it. Fix problems in this order: **parsing → chunking → embedding → search type → reranking → query handling**.
- **Bedrock KB chunking options:** **Default** (~300 tokens, keeps sentence boundaries) · **Fixed-size** (max tokens + overlap %, default 20%) · **Hierarchical** (search on small child chunks, return the large parent) · **Semantic** (splits by embedding similarity; costs more because the FM/embeddings run at ingestion) · **None** (files already split) · **Custom Lambda** (`customTransformationConfiguration`, `POST_CHUNKING`). ⚠️ The chunking strategy and parser **can't be changed** after the data source is created.
- **Embedding model choice** depends on modality (text vs multimodal), language coverage, max input tokens, **dimensions** (storage/latency vs accuracy), domain fit and cost. **Titan Text Embeddings V2**: 256/512/1024 dims, 8,192 tokens, 100+ languages, `normalize` flag. Going 1024→512 keeps ~99% of accuracy and 1024→256 keeps ~97%.
- **Use the same embedding model and dimensions for documents and queries.** The vector index dimension must equal the model's output dimension, or ingestion fails. Changing models means **re-embedding the whole corpus** into a new index.
- **Bulk embedding:** use **Bedrock batch inference** (`CreateModelInvocationJob`, JSONL in S3 → S3). It's asynchronous and cheaper than on-demand. Lambda/Step Functions fan-out suits incremental or near-real-time work. Cohere Embed accepts **up to 96 texts per call**, while Titan V2 takes **one `inputText` per call**.
- **Vector search choices:** **Bedrock KB** (managed: quick-create OpenSearch Serverless or S3 Vectors, or bring your own Aurora pgvector, Pinecone, Redis, MongoDB Atlas or Neptune Analytics) · **OpenSearch Service/Serverless** (k-NN with HNSW/IVF, FAISS/Lucene, hybrid, neural plugin, ms latency at scale) · **Aurora PostgreSQL pgvector** (SQL plus vectors next to relational data, HNSW/IVFFlat, `<=>` cosine).
- 🎯 **PQ11** 🎯 **PQ7** **Hybrid search = BM25 keyword + vector**, normalised and combined. It catches exact terms (SKUs, error codes, names, acronyms) that dense vectors miss. In a KB, use `overrideSearchType: HYBRID`; on OpenSearch Serverless this needs a filterable text field. **S3 Vectors is semantic-only.**
- 🎯 **PQ8** **Reranking** is a second stage. A cross-encoder rescores the top-N candidates (for example retrieve 50 → rerank → keep 5). Models are **Amazon Rerank 1.0** and **Cohere Rerank 3.5**, used through the `Rerank` API or `rerankingConfiguration` in Retrieve/RetrieveAndGenerate. It improves precision and **sends fewer tokens to the FM**. Text only.
- **Query handling:** multi-part question → **query decomposition** (`QUERY_DECOMPOSITION`) · vague or short query → **expansion/rewrite** (FM) or **HyDE** · follow-up turn ("what about its price?") → **conversational rewrite** using session history · filter values hidden in the query → **implicit metadata filtering** · multi-step pipeline with retries → **Step Functions**.
- **Consistent access:** wrap retrieval as **one tool**. That can be a Converse `toolSpec` (function calling), a Bedrock Agent action group or KB association, an **MCP server** (for example the AWS Bedrock KB Retrieval MCP server, or **AgentCore Gateway** turning Lambdas/APIs into MCP tools), or a standard internal **retrieval API** behind API Gateway. Any agent or model can then call it the same way.
- 🎯 **PQ15** **API choice:** **RetrieveAndGenerate** = managed RAG with citations and session memory · **Retrieve** = chunks + scores only, so you own reranking, prompts and generation · `numberOfResults` defaults to **5** with a max of **100**.

> [!IMPORTANT]
> **🎯 Practice question 15 — auditors must see which sources produced each answer: real time, no redeploys, LEAST overhead**
>
> - **Bedrock Knowledge Bases + `RetrieveAndGenerate`** returns **`citations`** with every answer: the source document location (S3 URI) and **metadata**, generated **in real time** at no extra cost.
> - Use **`sessionId`** for conversation context, and store the citations from each response (for example in S3 or CloudWatch Logs) as the **audit log**.
> - **Document updates:** re-sync the data source, and the managed index updates without redeploying the app. Citations stay accurate because they point to the indexed source version.
> - ❌ **Guardrails + parsing attribution from logs:** guardrails filter content, they don't track sources, and the parsing is custom work.
> - ❌ **Custom RDS/DynamoDB tracking middleware, or a SageMaker model that *predicts* sources:** a lot of work, adds latency, and isn't real tracing.

## Notes

### Skill 1.5.1 — Document segmentation (chunking) approaches

> *Develop effective document segmentation approaches to optimize retrieval performance for FM context augmentation (for example, by using Amazon Bedrock chunking capabilities, Lambda functions to implement fixed-size chunking, custom processing for hierarchical chunking based on content structure).*

#### Why chunking matters
- Embedding models compress a chunk into **one vector**. If a chunk mixes several topics, its vector is an **average of those meanings** ("semantic dilution") and matches none of them well.
- **Chunks that are too small** match precisely but **lack context**: the FM gets "...it must be renewed every 12 months" without knowing what "it" is. You also store more vectors and pay for more lookups.
- **Chunks that are too large** carry more context but also more noise. Similarity scores drop, you use more prompt tokens (cost and latency), and you risk the **"lost in the middle"** effect, where the model overlooks facts buried in the middle of a long context.
- Chunk size is limited by the **embedding model's max input tokens**, and by the **FM's context window × number of retrieved chunks** you plan to inject.
- Rule of thumb: chunk to the **unit of meaning users ask about**. FAQ = one Q&A pair. Policy = one clause or section. API docs = one endpoint. Support tickets = one ticket.

#### Bedrock Knowledge Bases chunking strategies

| Strategy | How it works | Key settings | Best for | Trade-offs |
| --- | --- | --- | --- | --- |
| **Default** | Splits into ~300-token chunks, keeping sentence boundaries | none | Quick start, general prose | Not structure-aware |
| **Fixed-size** | Fixed token windows with overlap | `maxTokens`, `overlapPercentage` (default 300 tokens / 20%) | Homogeneous text, predictable cost | Can cut through ideas; tune overlap (10–20% typical) |
| 🎯 **PQ11** **Hierarchical** | Two levels: **parent** chunks split into **child** chunks. Search runs on children; the matching **parent is returned** to the FM | `levelConfigurations` (parent max tokens, child max tokens), `overlapTokens` | Long structured docs: manuals, legal contracts, research papers | Precise matching *and* broad context; more tokens per result; fewer distinct results returned |
| 🎯 **PQ54** 🎯 **PQ2** **Semantic** | Embeds sentences and splits where similarity between neighbouring sentences drops past a threshold | `maxTokens`, `bufferSize` (neighbouring sentences grouped), `breakpointPercentileThreshold` | Text with no clear structure where topics shift mid-section (transcripts, narratives) | **Extra embedding cost at ingestion**; less predictable chunk sizes |
| **No chunking** | Each file becomes one chunk | none | Files you already split upstream, short FAQ entries | Large files exceed model limits |
| **Custom (Lambda)** | Your Lambda receives the parsed/chunked content from S3 and writes back chunks plus metadata | `customTransformationConfiguration` + intermediate S3 bucket | Domain rules: split on clause numbers, code functions, table rows; add metadata per chunk | You own and maintain the code |

> [!IMPORTANT]
> **🎯 Practice question 54 — fixed 500-token chunks split legal arguments; p95 >2 s; storage 90 → 360 GB**
>
> - **Switch to chunking on semantic boundaries** (whole arguments, clauses, sections) and **regenerate the embeddings** for the new chunks. Coherent chunks mean **better relevance** and **fewer, more useful chunks per query**, which also helps latency.
> - ❌ **Static summaries on CloudFront:** no live retrieval, and legal content changes.
> - ❌ **Raise dimensions from 768 to 4,096:** more memory and compute, and it **doesn't fix fragmented chunks**.
> - ❌ **DynamoDB keyword indexes:** no vector similarity, so no semantic search.
> - ⚠️ **Remember:** changing chunking means **re-ingesting and re-embedding** (in a Bedrock KB, chunking is fixed per data source).

> [!IMPORTANT]
> **🎯 Practice question 11 — 50 textbooks × 500 pages: irrelevant info + missing context, low latency, no hallucinations**
>
> - **Hierarchical chunking** (small **~200-token child** chunks for precise matching, larger **~1,000-token parent** chunks returned for context) suits long textbooks where a concept spans several sections.
> - **Hybrid search** (vector + keyword) → better precision, and catches exact domain terms.
> - **Relevance score threshold** → drop low-confidence chunks so the FM doesn't answer from weak context, which reduces hallucinations.
> - ❌ **Semantic chunking + query expansion:** splits by topic but has **no multi-level context** and no hybrid search.
> - ❌ **Fixed 256-token chunks + metadata filter:** breaks textbook concepts apart.
> - ❌ **Semantic cache + query rewriting + feedback loops:** doesn't fix the real problem, which is chunking and search.
> - 🔁 **Compare with PQ2:** there the parent chunk **spanned unrelated sections**, so semantic chunking won. Here the textbook needs **broad context around precise matches**, so hierarchical wins. Look for the cue: *"miss critical context" in long structured docs → hierarchical*; *"cites adjacent irrelevant sections" → semantic*.

> [!IMPORTANT]
> **🎯 Practice question 2 — keep meaning across related paragraphs (25M papers)**
>
> - Queries span methodology, results and discussion, and answers cite irrelevant sections → **semantic chunking**. It splits only where the topic really shifts.
> - **`bufferSize: 1`** → each sentence is embedded **with its immediate neighbours**, so local flow is kept.
> - **`breakpointPercentileThreshold: 85`** → split only at the **least-similar 15%** of adjacent sentence pairs, giving fewer, more coherent chunks.
> - ❌ **Hierarchical (1,000/200):** the returned **parent** can span neighbouring, unrelated sections, which causes off-topic citations.
> - ❌ **Fixed-size 300 / 10%:** cuts through semantic boundaries.
> - ❌ **No chunking + manual split + rerank:** doesn't scale to 25M documents, and reranking can't fix bad chunks.

- Defaults worth knowing (⚠️ verify in docs): hierarchical in the console starts at a **parent of 1,500 and child of 300 tokens, overlap 60**. Semantic starts at **max 300 tokens, buffer 0, breakpoint 95th percentile**.
- 🎯 **PQ70** **Parsing comes before chunking.** Use the **default parser** for plain text (no extra cost). Use **FM parsing** (a foundation model as parser) or **Bedrock Data Automation** for PDFs with tables, scans, charts and images. Tables that are badly parsed and then chunked are a classic root cause of "the answer is in the document but RAG can't find it".
- ⚠️ **Parser, chunking strategy and embedding model are fixed** once the data source/KB is created. To change them, create a new data source or KB and re-ingest.

> [!IMPORTANT]
> **🎯 Practice question 70 — KB over PDFs with charts and tables + relational DBs, citations, LEAST overhead**
>
> - **Bedrock Data Automation as the KB parser** → multimodal extraction from **charts, tables and text** in the PDFs.
> - **KB structured data retrieval** (a structured data store: natural language → SQL over Redshift / Glue Data Catalog) → queries the **relational data** directly. Answers still come with **source attribution**.
> - ❌ **Claude as the parser + "federated querying":** per the answer key, weaker at charts and tables than BDA. (FM parsing *can* handle tables, but **BDA** is the expected answer when "multimodal" plus "least overhead" appear.)
> - ❌ **GraphRAG on Neptune + Lambda to turn the DB into a graph:** custom work, and doesn't query relational data natively.
> - ❌ **OpenSearch hierarchical chunking + Lambda SQL generator:** no multimodal parsing, and custom SQL code.

```python
# bedrock-agent: create a data source with hierarchical chunking + FM parsing
bedrock_agent.create_data_source(
    knowledgeBaseId=kb_id,
    name="manuals",
    dataSourceConfiguration={"type": "S3", "s3Configuration": {"bucketArn": bucket_arn}},
    vectorIngestionConfiguration={
        "chunkingConfiguration": {
            "chunkingStrategy": "HIERARCHICAL",
            "hierarchicalChunkingConfiguration": {
                "levelConfigurations": [{"maxTokens": 1500}, {"maxTokens": 300}],
                "overlapTokens": 60,
            },
        },
        "parsingConfiguration": {
            "parsingStrategy": "BEDROCK_FOUNDATION_MODEL",
            "bedrockFoundationModelConfiguration": {"modelArn": parser_model_arn},
        },
    },
)
```

#### Custom chunking with Lambda (Bedrock KB)
- Configure `vectorIngestionConfiguration.customTransformationConfiguration` with:
  - `intermediateStorage.s3Location`, the S3 bucket where Bedrock writes input files for your Lambda and reads its output.
  - `transformations: [{ stepToApply: "POST_CHUNKING", transformationFunction: { transformationLambdaConfiguration: { lambdaArn } } }]`.
- Set the KB's chunking strategy to **NONE** if the Lambda does all the splitting. Otherwise the Lambda can post-process Bedrock's chunks, for example to add metadata or merge small chunks.
- The Lambda reads a batch file from S3 (content plus file metadata), writes back JSON with `fileContents[]` (each item has `contentBody`, `contentType` and `contentMetadata`), and returns the output file locations.
- The KB service role needs `lambda:InvokeFunction` on that Lambda and access to the intermediate bucket.
- Common uses: splitting legal text on `§`/clause numbers, code by function/class, Markdown by headers, CSV by rows with the header repeated in each chunk, and **adding per-chunk metadata** (section title, page, product) for later filtering.

#### DIY chunking outside a KB (Lambda / SageMaker Processing / Glue)
- **Fixed-size with overlap**: split by tokens (use a tokenizer that matches the embedding model). Overlap prevents losing a fact that spans a boundary.
- **Recursive character splitting** (LangChain `RecursiveCharacterTextSplitter`): tries `\n\n` → `\n` → `.` → space until each piece fits. It's cheap and respects paragraph boundaries better than a raw fixed split.
- **Structure-aware (hierarchical) chunking**: parse headings (H1/H2/H3, numbered sections, HTML DOM), build a tree, and produce chunks per leaf section. Prepend the **breadcrumb** ("Manual > Safety > Battery") to each chunk. This is often the best value for structured docs.
- **Contextual chunk enrichment**: before embedding, prepend a short FM-generated summary of the document or section to each chunk. This disambiguates pronouns and abbreviations and costs one FM call per chunk at ingestion.
- **Parent-document / small-to-big retrieval**: embed small chunks but store a `parent_id`. At query time, fetch the parent text. This is the DIY equivalent of KB hierarchical chunking.
- Scale: one Lambda per S3 object event for steady trickles. Use **Step Functions Distributed Map** or **SageMaker Processing / AWS Glue** for large backfills (Lambda has a 15-minute limit).

```python
# Minimal fixed-size token chunker with overlap (Lambda)
def chunk(tokens, size=500, overlap=50):
    step = size - overlap
    return [tokens[i:i + size] for i in range(0, max(len(tokens) - overlap, 1), step)]
```

#### Choosing a strategy (decision guide)

| Content | Recommended |
| --- | --- |
| Short, self-contained records (FAQs, product cards, tickets) | **No chunking** (one record per file) or small fixed-size |
| Long manuals or contracts with headings | **Hierarchical** (KB) or structure-aware custom Lambda |
| Transcripts, emails, prose with no structure | **Semantic** |
| PDFs with tables, scans, charts | **FM parsing / Bedrock Data Automation** + hierarchical |
| Code repositories | Custom Lambda (split by function/class) |
| Unknown or mixed, need a baseline fast | **Default/fixed 300 tokens, 20% overlap**, then evaluate |

- **Evaluate, don't guess.** Build a golden Q&A set, measure **context recall/precision** (Bedrock KB evaluation / RAG evaluation jobs, RAGAS), and compare strategies by running parallel KBs or data sources.

---

### Skill 1.5.2 — Selecting and configuring embedding solutions

> *Select and configure optimal embedding solutions to create efficient vector representations for semantic search (for example, by using Amazon Titan embeddings based on dimensionality and domain fit, by evaluating performance characteristics of Amazon Bedrock embedding models, by using Lambda functions to batch generate embeddings).*

#### Selection criteria
| Criterion | What to check |
| --- | --- |
| **Modality** | Text-only, or images/video/audio as well? Cross-modal queries (text → image) need a **multimodal model with a shared vector space** |
| **Language** | Multilingual corpus or queries? Cross-lingual retrieval (query in French, doc in English)? |
| **Max input tokens** | Must be ≥ your chunk size. Inputs over the limit are truncated or rejected |
| **Dimensions** | Higher = more expressive but more storage, RAM and latency. Lower = cheaper and faster with a small accuracy loss |
| **Output type** | float32 vs int8/binary. Binary cuts storage sharply but reduces precision, and not every store supports it |
| **Domain fit** | Benchmark on *your* data (recall@k, MRR, nDCG). General benchmarks such as MTEB are only a starting point |
| **Cost and throughput** | Price per 1K input tokens, batch inference support, quotas (TPM/RPM) |
| **Integration** | Supported by Bedrock KB / OpenSearch neural plugin / your vector store? |

#### Bedrock embedding models (exam-relevant)
| Model | Modality | Dimensions | Notes |
| --- | --- | --- | --- |
| 🎯 **PQ41** **Amazon Titan Text Embeddings V2** | Text | **1024 (default) / 512 / 256** | Up to **8,192 tokens**; **100+ languages**; `normalize: true` gives unit vectors (use cosine/dot product); 1024→512 ≈ 99% and →256 ≈ 97% retrieval accuracy; supports float and binary output |
| Amazon Titan Embeddings G1 – Text | Text | 1536 | Older; prefer V2 for new builds |
| **Amazon Titan Multimodal Embeddings G1** | Image + text | 1024 (default) / 384 / 256 | Image search, product catalogues, "find images like this text" |
| **Amazon Nova Multimodal Embeddings** | Text, images, documents, video, audio | Several sizes (⚠️ verify in docs) | One shared space for all modalities → **cross-modal retrieval**, agentic RAG over mixed media |
| **Cohere Embed English / Multilingual v3** | Text (v3 also images) | 1024 | Max **512 tokens** per text; `input_type` = `search_document` vs `search_query`; up to **96 texts per request**; int8/binary output types |
| **Cohere Embed v4** | Text + images (interleaved) | 256–1536 | Business docs with tables, charts, screenshots; long context; 100+ languages |

> [!IMPORTANT]
> **🎯 Practice question 41 — 50k medical documents in EN/FR/DE, precise terminology, minimal code**
>
> - **Titan Text Embeddings V2** handles **many languages natively, including English, French and German**, so **embed each document in its original language with no translation**. Medical terminology stays intact.
> - **Semantic chunking at the medical-section level** keeps the terminology in context, at a **moderate dimension** (for example 1024 or 512).
> - ❌ **Amazon Translate per query:** adds latency plus custom code for language detection and routing.
> - ❌ **Translate everything to English first:** loses nuance, and **cross-language queries give sub-optimal results** (per the AWS docs).
> - ❌ **Titan *Multimodal* Embeddings on page images:** made for image–text pairs, not text-heavy documents.

- ⚠️ **Asymmetric models** (Cohere) need `input_type="search_document"` at ingestion and `"search_query"` at query time. Using the wrong one quietly hurts recall.
- **Normalisation and distance metric must match.** Normalised vectors work with cosine or inner product. Configure the index `space_type` (OpenSearch) or `distanceMetric` (S3 Vectors) to match what the model was trained for.

```python
# Titan Text Embeddings V2 via InvokeModel
body = {"inputText": chunk_text, "dimensions": 512, "normalize": True, "embeddingTypes": ["float"]}
resp = bedrock_runtime.invoke_model(modelId="amazon.titan-embed-text-v2:0", body=json.dumps(body))
vector = json.loads(resp["body"].read())["embedding"]
```

#### Dimensionality trade-offs
- Storage ≈ `num_vectors × dims × bytes_per_dim`. For example, 10M chunks × 1024 dims × 4 bytes ≈ **41 GB** of raw vectors, before HNSW graph overhead (which can roughly double RAM). At 256 dims that drops to about 10 GB.
- Lower dims → smaller HNSW graphs, faster k-NN and cheaper OpenSearch OCUs/instances. Combine with **byte/binary quantization** (OpenSearch, Titan binary) for further savings. See [Task 1.4](04-task-1.4-vector-store-solutions.md) for store-side tuning.
- Exam pattern: *"Reduce vector storage cost with minimal accuracy loss"* → **Titan V2 at 512 or 256 dims** and/or **binary embeddings / quantization**.

#### Domain fit and when general embeddings aren't enough
- Very specialised vocabulary (medical codes, legal citations, part numbers) can embed poorly. Options, from cheapest to most expensive:
  1. **Hybrid search**, so BM25 catches exact tokens (Skill 1.5.4).
  2. **Contextual enrichment / metadata** in the chunk text.
  3. **Query expansion with domain synonyms** (Skill 1.5.5).
  4. **Fine-tuned or custom embedding model** hosted on **SageMaker AI** (for example a sentence-transformers model from JumpStart, fine-tuned on in-domain pairs), called from a Lambda or OpenSearch ML connector. Bedrock KB requires a supported Bedrock embedding model, so a custom embedding model implies a **DIY pipeline** (OpenSearch/pgvector) rather than a managed KB.

#### Batch-generating embeddings
| Pattern | When | Notes |
| --- | --- | --- |
| **Bedrock KB sync (ingestion job)** | Using a managed KB | Bedrock chunks and embeds for you; incremental on re-sync |
| **Bedrock batch inference** (`CreateModelInvocationJob`) | Large backfills (millions of chunks), not latency-sensitive | Input JSONL in S3 → output JSONL in S3; asynchronous; **lower price than on-demand** for supported models; has a minimum record count and job quotas (⚠️ verify in docs) |
| **Lambda + SQS fan-out** | Steady incremental stream (new docs every few minutes) | S3 event → SQS → Lambda (batch of chunks) → embed → upsert. Use **reserved concurrency** and backoff to stay within Bedrock TPM/RPM quotas |
| **Step Functions Distributed Map** | Large backfill with orchestration, retries, progress tracking | Map over S3 objects; each child invokes Lambda; built-in retry on `ThrottlingException` |
| **SageMaker Processing / Batch Transform** | Custom embedding model on SageMaker | Batch Transform for offline inference at scale |
| **OpenSearch neural ingest pipeline** | Embeddings generated inside OpenSearch | `text_embedding` processor calls Bedrock via an ML connector |

- Throttling: catch `ThrottlingException`, retry with **exponential backoff and jitter**, request quota increases, or move bulk work to batch inference.
- Batch multiple texts per call where the model allows it (Cohere: up to 96). Titan V2 is **one text per request**, so parallelise.
- **Idempotency:** generate a deterministic ID per chunk (for example a hash of `doc_id + chunk_index + content`). Re-runs then upsert instead of creating duplicates.

```python
# Lambda: embed a batch of chunks pulled from SQS, upsert into OpenSearch
def handler(event, _):
    docs = []
    for rec in event["Records"]:
        c = json.loads(rec["body"])
        emb = embed_titan_v2(c["text"])          # retry/backoff inside
        docs += [{"index": {"_index": "kb-chunks", "_id": c["chunk_id"]}},
                 {"text": c["text"], "embedding": emb, **c["metadata"]}]
    opensearch.bulk(body=docs)
```

---

### Skill 1.5.3 — Deploying and configuring vector search 🎯 PQ36

> *Deploy and configure vector search solutions to enable semantic search capabilities for FM augmentation (for example, by using OpenSearch Service with vector search capabilities, Amazon Aurora with the pgvector extension, Amazon Bedrock Knowledge Bases with managed vector store functionality).*

> [!IMPORTANT]
> **🎯 Practice question 36 — 20M restaurants + 200M reviews in PostgreSQL, natural-language search, P95 <500 ms, hourly updates, LEAST dev effort**
>
> - **Move the search data to Amazon OpenSearch Service.** Embed descriptions, reviews and menus with a **Bedrock embedding model**, embed each query with the **same model**, and run a **k-NN search**. It's managed and scales horizontally for peaks.
> - ❌ **Keyword-only OpenSearch with custom analysers:** can't handle complex natural-language queries.
> - ❌ **Bedrock KB + custom ingestion pipeline:** built for **RAG (generation)** and adds overhead when only semantic search is needed.
> - ❌ **RDS PostgreSQL + pgvector + Lambda:** works, but per the answer key it means **more integration work** at this scale.
> - 🔁 **Compare with other questions:** pgvector wins when the cue is *"keep the vectors next to existing relational data / the team knows PostgreSQL"*. OpenSearch wins on *"massive scale, high QPS, low development effort for search"*.

#### Option comparison
| | **Bedrock Knowledge Bases** | **OpenSearch Service / Serverless** | **Aurora PostgreSQL + pgvector** |
| --- | --- | --- | --- |
| Effort | Lowest: managed ingestion, chunking, embedding, retrieval, citations | Medium: you design the index and pipelines (unless used behind a KB) | Medium: SQL schema + indexes |
| Search types | Semantic, hybrid (store-dependent), metadata filters, reranking, query decomposition | k-NN (exact/ANN), BM25, **hybrid**, neural sparse, aggregations, filtering | Vector + full SQL (joins, WHERE, transactions); full-text via `tsvector` |
| Scale/latency | Depends on backing store | Billions of vectors, **ms latency**; Serverless = no cluster management | Millions of vectors comfortably; scale with instance size and read replicas |
| Pick when | "Least operational overhead", "managed RAG", "citations" | "Hybrid search", "high QPS", "fine control over k-NN", "existing OpenSearch" | "Vectors next to relational/transactional data", "team knows PostgreSQL", "SQL filtering/joins" |

#### Bedrock Knowledge Bases — managed vector search
- Create KB → choose the **embedding model** (and dimensions) → choose the **vector store**:
  - **Quick create**: Bedrock provisions an **OpenSearch Serverless** collection/index or an **S3 Vectors** bucket/index.
  - **Bring your own**: Aurora PostgreSQL (pgvector), OpenSearch Serverless/managed cluster, Pinecone, Redis Enterprise Cloud, MongoDB Atlas, Neptune Analytics (GraphRAG). Provide the field mappings (vector, text, metadata) and credentials via **Secrets Manager** where needed.
- Query via `bedrock-agent-runtime`:
  - `Retrieve` → chunks + `score` + `location` + `metadata`.
  - `RetrieveAndGenerate` → FM answer + `citations`; `sessionId` gives multi-turn context.
- ⚠️ Vector store type, embedding model and chunking are **fixed after creation**.

```python
# Retrieve with hybrid search, metadata filter and reranking
resp = agent_runtime.retrieve(
    knowledgeBaseId=kb_id,
    retrievalQuery={"text": "How do I reset the X200 controller?"},
    retrievalConfiguration={
        "vectorSearchConfiguration": {
            "numberOfResults": 20,                     # default 5, max 100
            "overrideSearchType": "HYBRID",            # or SEMANTIC
            "filter": {"andAll": [
                {"equals": {"key": "product", "value": "X200"}},
                {"greaterThanOrEquals": {"key": "year", "value": 2023}},
            ]},
            "rerankingConfiguration": {
                "type": "BEDROCK_RERANKING_MODEL",
                "bedrockRerankingConfiguration": {
                    "modelConfiguration": {"modelArn": rerank_model_arn},
                    "numberOfRerankedResults": 5,
                },
            },
        }
    },
)
```

- Filter operators: `equals`, `notEquals`, `greaterThan(OrEquals)`, `lessThan(OrEquals)`, `in`, `notIn`, `startsWith`, `stringContains`, `listContains`, combined with `andAll` / `orAll`. Not every operator works on every store (⚠️ verify in docs).

#### OpenSearch Service / Serverless vector search
- **Index setup**: `"index.knn": true`; a `knn_vector` field with `dimension` = model dims; `method` = **HNSW** (no training, best latency/recall) or **IVF** (needs training, smaller memory with PQ); `engine` = **FAISS** (required for KB metadata filtering on OpenSearch Serverless) or Lucene; `space_type` = `cosinesimil` / `l2` / `innerproduct`.
- **HNSW tuning**: `m` (graph links per node; higher = better recall, more memory), `ef_construction` (build quality vs build time), `ef_search` (query recall vs latency).
- **Filtering**: efficient k-NN filtering (pre-filter during the ANN search) vs post-filter (risk of returning fewer than k results). Prefer **efficient filtering** with FAISS/Lucene.
- **Serverless**: create a **vector search collection** (type `VECTORSEARCH`); capacity in **OCUs**; needs encryption, network and **data access policies** (grant the KB role or app role).
- **Neural plugin**: create an ML connector to Bedrock (for example Titan Embeddings) → register the model → an ingest pipeline with a `text_embedding` processor embeds at index time → the `neural` query embeds the query text automatically. Your app never handles vectors.

```json
PUT kb-chunks
{
  "settings": { "index.knn": true },
  "mappings": { "properties": {
    "embedding": { "type": "knn_vector", "dimension": 1024,
      "method": { "name": "hnsw", "engine": "faiss", "space_type": "innerproduct",
                  "parameters": { "m": 16, "ef_construction": 128 } } },
    "text":     { "type": "text" },
    "product":  { "type": "keyword" },
    "year":     { "type": "integer" }
  } }
}
```

#### Aurora PostgreSQL with pgvector
- Enable the extension: `CREATE EXTENSION IF NOT EXISTS vector;`
- Column type `vector(n)`. Distance operators: `<->` **L2**, `<=>` **cosine distance**, `<#>` **negative inner product**.
- Index types: **HNSW** (better recall/latency, slower build, more memory; `m`, `ef_construction`, query-time `hnsw.ef_search`) or **IVFFlat** (faster build, needs data present first, `lists`, query-time `ivfflat.probes`). Use the opclass that matches your distance: `vector_cosine_ops`, `vector_l2_ops`, `vector_ip_ops`.
- **Aurora as a Bedrock KB store** requires: an Aurora PostgreSQL cluster with **RDS Data API enabled**, credentials in **Secrets Manager**, and a table with columns for the ID (uuid primary key), the embedding (`vector(dims)`), chunk text, and metadata (json). Map the column names in the KB config. Create an HNSW index on the embedding column (and a text index for hybrid search — ⚠️ verify in docs).
- Strength: **join** vector hits with relational data (pricing, inventory, entitlements) and apply **row-level security** in one query.

```sql
CREATE TABLE kb_chunks (
  id uuid PRIMARY KEY,
  embedding vector(1024),
  chunks text,
  metadata json,
  tenant_id text
);
CREATE INDEX ON kb_chunks USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);

-- Top 5 semantically similar chunks for one tenant
SET hnsw.ef_search = 100;
SELECT id, chunks, 1 - (embedding <=> $1::vector) AS similarity
FROM kb_chunks
WHERE tenant_id = $2
ORDER BY embedding <=> $1::vector
LIMIT 5;
```

#### Other AWS vector-capable services (recognise them in answers)
- **S3 Vectors**: cheapest, subsecond, semantic only, for large archival corpora. **MemoryDB / ElastiCache (Valkey/Redis)**: in-memory vector search with single-digit-ms latency, used for **semantic caching**. **DocumentDB**: vector search next to JSON documents. **Neptune Analytics**: GraphRAG. **Kendra**: managed enterprise search with connectors and its own `Retrieve` API for passages (no vector DB to manage), which can act as a KB-style retriever.

---

### Skill 1.5.4 — Advanced search architectures (semantic, hybrid, reranking)

> *Create advanced search architectures to improve the relevance and accuracy of retrieved information for FM context (for example, by using OpenSearch for semantic search, hybrid search that combines keywords and vectors, Amazon Bedrock reranker models).*

#### Retrieval types
| Type | Strength | Weakness |
| --- | --- | --- |
| **Lexical / BM25** | Exact terms: IDs, error codes, names, rare jargon, acronyms | No synonyms or paraphrases ("car" ≠ "automobile") |
| **Dense semantic (k-NN)** | Meaning, paraphrase, cross-lingual | Misses exact rare tokens; embeds "ERR-4012" poorly |
| **Neural sparse** (OpenSearch) | Learned term expansion with inverted-index speed | Needs sparse encoding models; less common |
| **Hybrid (BM25 + dense)** | Best of both; usually the highest recall in enterprise corpora | Needs score normalisation and weight tuning |
| **Graph (GraphRAG)** | Multi-hop relations between entities | Graph build and maintenance cost |

#### Hybrid search in OpenSearch 🎯 PQ7 🎯 PQ59
- Create a **search pipeline** with a `normalization-processor` (`min_max` or `l2` normalisation; `arithmetic_mean` combination with **weights**). Then send a `hybrid` query with a BM25 `match` subquery and a `neural`/`knn` subquery.
- Newer OpenSearch versions also support **Reciprocal Rank Fusion (RRF)**, which combines ranks instead of raw scores (⚠️ verify version support).

> [!IMPORTANT]
> **🎯 Practice question 59 — legal assistant: semantic relationships + exact terms and citations, fast and precise (same pattern as PQ7)**
>
> - **OpenSearch hybrid search** (vector for **concepts** + keyword/BM25 for **exact terms and citations**) **+ a Bedrock reranker** → fast, precise top results.
> - ❌ **KB default retrieval + post-processing in Bedrock:** extra latency and no keyword precision.
> - ❌ **Vector search + keyword filters in RDS merged by Lambda:** two systems and slow.
> - ❌ **KB default vector search + query expansion:** **no reranker** and no keyword matching.

> [!IMPORTANT]
> **🎯 Practice question 7 — exact regulatory terms (99.9% recall) + conceptual search (95% precision), <500 ms**
>
> - **OpenSearch hybrid search = BM25 keyword + vector similarity in one query.** BM25 gives **exact-term recall** and vectors give **conceptual matches**.
> - Combine them with **weighted result fusion** (the search pipeline **normalisation processor** with weights, or RRF).
> - Then **rerank with a Bedrock reranker (Cohere Rerank)** so the most relevant documents fill the FM's context window.
> - ❌ **Vector-only + let the FM reorder:** misses exact terms and has no real reranking.
> - ❌ **BM25 in OpenSearch + vectors in RDS, merged in the app:** extra network hops, slow, and no reranker.
> - ❌ **KB semantic search + Comprehend post-processing:** no keyword + vector fusion, no reranking, and extra latency.

```json
PUT /_search/pipeline/hybrid-pipeline
{ "phase_results_processors": [ { "normalization-processor": {
    "normalization": { "technique": "min_max" },
    "combination":   { "technique": "arithmetic_mean", "parameters": { "weights": [0.3, 0.7] } } } } ] }

GET kb-chunks/_search?search_pipeline=hybrid-pipeline
{ "query": { "hybrid": { "queries": [
    { "match":  { "text": "ERR-4012 pump overheating" } },
    { "neural": { "embedding": { "query_text": "ERR-4012 pump overheating", "model_id": "<bedrock-model-id>", "k": 50 } } }
] } } }
```

- **Bedrock KB hybrid**: `overrideSearchType: "HYBRID"`. Supported on OpenSearch Serverless (needs a filterable text field, otherwise it falls back to semantic) and on some other stores. Not supported on S3 Vectors. If unset, Bedrock chooses.

#### Reranking (two-stage retrieval) 🎯 PQ8
- Stage 1 (**retriever**, bi-encoder): fast and cheap; returns a wide candidate set (top 25–100) for recall.
- Stage 2 (**reranker**, cross-encoder): reads query and passage together and scores relevance precisely. It's slower per item, so run it only on the candidates. Keep the top 3–5 for precision.
- **Bedrock reranker models**: **Amazon Rerank 1.0** and **Cohere Rerank 3.5**.
  - Standalone: `bedrock-agent-runtime` **`Rerank`** API (query + list of text sources) → returns indices + relevance scores. Use it with *any* retriever (OpenSearch, pgvector, Kendra).
  - Integrated: `rerankingConfiguration` inside `Retrieve` / `RetrieveAndGenerate`, with `numberOfRerankedResults`. You can also pass `metadataConfiguration` to rerank on selected metadata fields.
- Benefits: higher precision → **fewer chunks in the prompt** → **lower FM token cost and latency**, and fewer hallucinations from irrelevant context.
- Costs: extra latency (a reranker call) and per-query rerank pricing. Rerankers work on **text only**.
- Alternative: **LLM-as-reranker** (prompt the FM to score passages). It's flexible but slower and more expensive than a dedicated reranker.

> [!IMPORTANT]
> **🎯 Practice question 8 — relevant precedents sit at positions 15–40, but the FM can only use 10, <3 s**
>
> - **Turn on KB reranking:** retrieve **50**, have a **cross-encoder reranker** (Amazon Rerank or Cohere Rerank) score each **query + document pair together**, and pass only the **top 10** (`numberOfRerankedResults`) to the FM.
> - The reranker adds a **new, stronger relevance signal**, so documents buried at positions 15–40 move up into the top 10, all within the latency budget.
> - ❌ **Recomputing cosine similarity in Lambda:** it's the same signal as the first search, so the order barely changes.
> - ❌ **3 KBs with different chunk sizes + TF-IDF merge:** lexical scoring, more complexity and more latency.
> - ❌ **Raising temperature:** that changes how random the generated text is, not how documents are ranked.

```python
resp = agent_runtime.rerank(
    queries=[{"type": "TEXT", "textQuery": {"text": question}}],
    sources=[{"type": "INLINE", "inlineDocumentSource": {"type": "TEXT", "textDocument": {"text": c}}}
             for c in candidates],
    rerankingConfiguration={"type": "BEDROCK_RERANKING_MODEL",
        "bedrockRerankingConfiguration": {"numberOfResults": 5,
            "modelConfiguration": {"modelArn": "arn:aws:bedrock:us-west-2::foundation-model/amazon.rerank-v1:0"}}},
)
```

#### Other relevance techniques
- **Metadata filtering**: restrict the search space (product, date, region, tenant, access level) before similarity. It gives a big precision gain and also enforces access control. Design is in [Task 1.4](04-task-1.4-vector-store-solutions.md).
- 🎯 **PQ11** **Score thresholds**: drop chunks below a minimum similarity, so the FM can say "I don't know" instead of answering from weak context.
- **Maximal Marginal Relevance (MMR)/diversity**: avoid five near-duplicate chunks from the same page.
- **Context assembly**: order chunks by relevance, put the most relevant at the **start or end** (to avoid "lost in the middle"), deduplicate, include the source title.
- **Semantic cache** (MemoryDB/ElastiCache vector search or OpenSearch): reuse answers for semantically similar repeated questions to cut latency and cost.
- **GraphRAG** (Bedrock KB + Neptune Analytics): multi-hop questions ("which suppliers of parts in recalled model X are also in region Y?").

#### Reference architecture (DIY advanced retrieval)
```
User → API Gateway → Lambda (orchestrator)
   ├─ (1) query rewrite/expand  → Bedrock FM (small, fast model)
   ├─ (2) hybrid search          → OpenSearch (BM25 + k-NN, metadata filter, top 50)
   ├─ (3) rerank                 → Bedrock Rerank API (keep top 5)
   ├─ (4) assemble prompt        → Prompt Management template
   └─ (5) generate               → Bedrock Converse (+ Guardrails, contextual grounding check)
```

---

### Skill 1.5.5 — Query handling systems (expansion, decomposition, transformation)

> *Develop sophisticated query handling systems to improve the retrieval effectiveness and result quality for FM augmentation (for example, by using Amazon Bedrock for query expansion, Lambda functions for query decomposition, Step Functions for query transformation).*

#### Why transform queries
- User queries are often **short, ambiguous, conversational, multi-part, or worded differently from the documents**. If you embed them as they are, you get a vector that sits far from the relevant chunks.

#### Techniques
| Technique | What it does | Use when | AWS implementation |
| --- | --- | --- | --- |
| **Query rewriting / normalisation** | Fix typos, expand acronyms, remove chit-chat, make the query self-contained | Noisy user input | Lambda (rules, dictionaries) + small Bedrock model |
| **Conversational (contextual) rewrite** | Turn a follow-up ("and for the premium tier?") into a standalone query using chat history | Multi-turn chat | `RetrieveAndGenerate` with `sessionId` (managed); or DIY with history from **DynamoDB** + FM rewrite |
| **Query expansion** | Add synonyms, related terms or multiple paraphrases (multi-query), retrieve for each, merge (RRF/dedupe) | Vocabulary mismatch, low recall | Bedrock FM generates N variants → parallel retrieval |
| **HyDE** (hypothetical document embeddings) | FM writes a hypothetical answer; embed *that* and search | Short questions vs long technical docs | Bedrock FM → embed → k-NN |
| **Query decomposition** | Split a multi-part or comparative question into sub-queries, retrieve separately, combine | "Compare X and Y", "A and also B" | **Bedrock KB `QUERY_DECOMPOSITION`**; or Lambda + FM; or Step Functions Map/Parallel |
| **Step-back prompting** | Generate a broader, more general question to fetch background context | Questions that need principles first | FM call before retrieval |
| **Metadata/filter extraction** (self-query) | Pull structured filters ("2024", "EU", "product X") out of the natural-language query | Corpus has rich metadata | **Bedrock KB implicit filtering** (`implicitFilterConfiguration`); or FM + JSON schema → filter |
| **Routing** | Classify the query and send it to the right index/KB/tool (HR KB vs IT KB vs SQL) | Multiple domains or sources | Small FM or **Amazon Comprehend custom classifier**; Lambda router; agents |
| **Text-to-SQL / structured** | Turn the question into SQL for structured data | Numbers or aggregates in tables | Bedrock KB **structured data store** (Redshift) or FM + Athena/Redshift |

#### Bedrock-managed options
- **Query decomposition** in `RetrieveAndGenerate`:
```python
agent_runtime.retrieve_and_generate(
    input={"text": "Compare the 2023 and 2024 parental leave policies for UK staff"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": kb_id,
            "modelArn": gen_model_arn,
            "orchestrationConfiguration": {
                "queryTransformationConfiguration": {"type": "QUERY_DECOMPOSITION"}
            },
        },
    },
)
```
- **Implicit metadata filtering**: in `vectorSearchConfiguration.implicitFilterConfiguration`, provide the `metadataAttributes` (key, type, description) and a `modelArn`. The FM derives filters from the query automatically. Use it when users don't pick filters in a UI.
- **Session context**: passing `sessionId` back into `RetrieveAndGenerate` lets Bedrock use earlier turns, so follow-up questions are reformulated with context.

#### Orchestrating with Lambda and Step Functions
- **Lambda** for simple, synchronous, low-latency transforms (one FM call to rewrite or decompose and return JSON).
- **Step Functions** when the flow has multiple steps, branches, parallelism, retries or human review:
  - 🎯 **PQ29** **Express workflows** for synchronous, high-volume, short (<5 min) request-path pipelines. **Standard** for long-running or auditable ones.
  - **Parallel/Map** states fan out sub-queries. **Choice** routes by query type. Built-in **Retry/Catch** handles Bedrock throttling.
  - Direct service integrations: Step Functions can call **`bedrock:InvokeModel`** (optimised integration) and other AWS SDK actions (for example `bedrockagentruntime:Retrieve`) without a Lambda.

> [!WARNING]
> **🎯 Practice question 29 — batch of S3 reports: Bedrock sentiment + BDA segment extraction, MOST operationally efficient**
>
> - **Step Functions *Standard* workflow** with a **Map state** that iterates over the reports (concurrency is configurable; use a **Distributed Map** for very large sets).
> - **Optimised Bedrock `InvokeModel` integration** for sentiment, with no Lambda needed.
> - **Lambda 1** starts BDA (`InvokeDataAutomationAsync` with the BDA project). **Lambda 2** checks the job status, inside a **Wait → check → Choice back-off loop** in the state machine, so no Lambda function runs for a long time.
>
> Traps:
> - ❌ **Express workflow:** **5-minute maximum duration** and no `.sync` / `.waitForTaskToken` patterns, so it's wrong for long asynchronous jobs.
> - ❌ **Parallel state:** runs a **fixed set of branches**. It doesn't iterate over items, which is what **Map** does.
> - ❌ **One Lambda that starts BDA *and* waits for it:** a long-running function.
> - ❌ **Bedrock Flows "retrieval node" to read a set of S3 files:** Flows retrieval nodes query **knowledge bases**, and the S3 retrieval node reads a single object by key (verify in docs). Flows also has no polling or back-off for asynchronous BDA.
> - ❌ **Amazon Lex node for sentiment:** Lex is for chatbots, not orchestration.

```
Step Functions (Express) query-transformation workflow
 Start → Classify (Bedrock InvokeModel: {type, filters, subqueries[]})
       → Choice: type == "comparative" ?
            ├─ yes → Map(subqueries): Retrieve (KB) → Rerank → collect
            └─ no  → Rewrite/HyDE → Retrieve (KB)
       → Merge + dedupe (Lambda) → Generate (Bedrock Converse) → End
 Retry: ThrottlingException, backoff 2x, max 5 · Catch → fallback "no-RAG" answer
```

- **Design guidance**:
  - Use a **small, fast, cheap model** (for example Amazon Nova Micro/Lite, Claude Haiku) for transformation steps. Save the large model for final generation.
  - Return **structured JSON** from transformation steps (use tool use / JSON schema) so downstream steps can parse it reliably.
  - Each transformation step **adds latency**. Cache rewrites for repeated queries, and skip decomposition when the classifier says the query is simple.
  - Log original and transformed queries (CloudWatch) to debug retrieval misses and evaluate the effect.

---

### Skill 1.5.6 — Consistent access mechanisms (function calling, MCP, standard APIs)

> *Create consistent access mechanisms to enable seamless integration with FMs (for example, by using function calling interfaces for vector search, Model Context Protocol [MCP] clients for vector queries, standardized API patterns for retrieval augmentation).*

#### Why standardise retrieval access
- Several apps, agents and models need the same knowledge. If each team writes its own embedding + query code, behaviour drifts (different filters, k, rerankers), security varies, and changes are hard to roll out.
- Expose retrieval as **one contract** (input: query + filters + k; output: chunks + scores + citations). Then the vector store, embedding model or reranker can change **behind** the interface without touching callers.

#### Pattern 1 — Function calling (tool use) with the Converse API
- Define a `search_knowledge_base` tool with a JSON schema. The FM decides when to call it. Your code runs the retrieval and returns results as `toolResult`, and the FM answers grounded on them.
- This is **agentic RAG**: the model can choose to search, refine its query and search again, or skip retrieval for chit-chat.

```python
tool_config = {"tools": [{"toolSpec": {
    "name": "search_knowledge_base",
    "description": "Search internal product docs. Use for any question about products, setup or troubleshooting.",
    "inputSchema": {"json": {"type": "object", "properties": {
        "query":   {"type": "string", "description": "Standalone search query"},
        "product": {"type": "string", "enum": ["X100", "X200"]},
        "top_k":   {"type": "integer", "default": 5}},
        "required": ["query"]}}}}]}

resp = bedrock_runtime.converse(modelId=model_id, messages=messages, toolConfig=tool_config)
# if resp["stopReason"] == "tool_use": run retrieve(), append toolResult message, call converse again
```

#### Pattern 2 — Bedrock Agents / AgentCore
- **Bedrock Agents**: associate a **Knowledge Base** directly (the agent decides when to query it based on the KB description) or add an **action group** (Lambda + OpenAPI/function schema) for custom retrieval.
- **Amazon Bedrock AgentCore**: agents built with any framework (Strands, LangGraph, CrewAI) reach tools through **AgentCore Gateway**. Gateway turns **Lambda functions, OpenAPI/Smithy APIs** and existing MCP servers into **MCP-compatible tools**, with inbound auth (OAuth/Cognito) and outbound credential handling. A retrieval Lambda exposed this way is available to every agent in the organisation.

#### Pattern 3 — Model Context Protocol (MCP)
- **MCP** is an open standard (client–server, JSON-RPC) for exposing **tools, resources and prompts** to AI applications. An **MCP client** (inside the agent/host app) discovers tools from **MCP servers** (`tools/list`) and calls them (`tools/call`).
- Retrieval as an MCP server: tools such as `query_knowledge_base(query, kb_id, filters, top_k)` and `list_knowledge_bases()`. AWS publishes an open-source **Bedrock Knowledge Base Retrieval MCP server** (awslabs MCP collection) that wraps `Retrieve`, with optional reranking.
- Transport: **stdio** (local) or **streamable HTTP** (remote; host it on AgentCore Runtime, Lambda or ECS behind auth).
- Benefits: write once, use from any MCP-capable client (Claude Desktop/Code, Amazon Q Developer, Strands agents, LangGraph). The tool schema is discovered at runtime, so no SDK coupling is needed.

```json
{
  "name": "query_knowledge_base",
  "description": "Semantic + hybrid search over company knowledge bases. Returns chunks with source URIs.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query":   { "type": "string" },
      "kb_id":   { "type": "string" },
      "filters": { "type": "object", "description": "Metadata filter, e.g. {\"department\": \"HR\"}" },
      "top_k":   { "type": "integer", "default": 5, "maximum": 20 }
    },
    "required": ["query", "kb_id"]
  }
}
```

#### Pattern 4 — Standardised retrieval API (internal "retrieval service")
- **API Gateway (REST/HTTP) → Lambda → retriever** (KB Retrieve / OpenSearch / pgvector → Rerank). It returns a fixed response schema.
- Cross-cutting concerns live in one place:
  - **AuthN/Z**: Cognito/IAM authorizers. Map caller identity → **metadata filter** (tenant, department, clearance) so users only retrieve what they may see.
  - **Throttling and usage plans** per consuming team, plus caching (API Gateway cache or semantic cache).
  - **Configuration** (k, search type, reranker on/off, KB ID per tenant) in **AWS AppConfig**. Change behaviour without redeploying, in the same way as dynamic model selection in [Task 1.2](02-task-1.2-select-configure-fms.md).
  - **Observability**: CloudWatch metrics (latency, hit count, empty results), X-Ray traces, logged queries for evaluation.
  - **Versioning**: `/v1/retrieve`, so the contract is stable while the internals change.

```json
// POST /v1/retrieve  → response
{
  "query": "reset X200 controller",
  "results": [
    { "text": "To reset the X200 ...", "score": 0.83,
      "source": "s3://docs/x200/manual.pdf", "page": 42,
      "metadata": { "product": "X200", "year": 2024 } }
  ],
  "retrieval": { "searchType": "HYBRID", "reranked": true, "kb": "kb-prod-01" }
}
```

#### Choosing the pattern
| Need | Choose |
| --- | --- |
| Single app, FM should decide when to search | **Converse tool use** (function calling) |
| Managed agent with a KB, no code for retrieval logic | **Bedrock Agent + KB association** |
| Many agents/frameworks/IDEs share the same retrieval tools | **MCP server** (optionally via **AgentCore Gateway**) |
| Non-agent apps, strict governance, per-team quotas | **API Gateway retrieval service** |
| Just need managed RAG with citations | **RetrieveAndGenerate** directly |

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Long manuals with headings; need precise match but full-section context | **Hierarchical chunking** (search child, return parent) |
| Transcripts/prose with no structure; topics shift mid-document | **Semantic chunking** |
| Split contracts on clause numbers and add a clause ID to each chunk | **Custom chunking Lambda** (`customTransformationConfiguration`, `POST_CHUNKING`) |
| Files are already pre-split one-record-per-file (FAQs) | **No chunking** |
| PDFs with tables and scans retrieve poorly | **FM parsing** or **Bedrock Data Automation** parser |
| Want to change chunking strategy on an existing KB data source | Not possible → **create a new data source/KB** and re-ingest |
| Cut vector storage cost with minimal accuracy loss | **Titan Text Embeddings V2 at 512/256 dims**; binary embeddings / quantization |
| Text query must find images/video | **Multimodal embeddings** (Nova Multimodal / Titan Multimodal) — shared vector space |
| Corpus and queries in many languages | **Titan V2** (100+ languages) or **Cohere Embed Multilingual** |
| Embed millions of chunks for a backfill, cheapest, not urgent | **Bedrock batch inference** (JSONL in S3) |
| Embedding Lambda hits `ThrottlingException` | **Exponential backoff + jitter**, SQS buffering, reserved concurrency, quota increase, or batch inference |
| Ingestion fails after switching embedding model | **Index dimension mismatch** → new index matching the model's dims, re-embed |
| 🎯 **PQ15** Least operational overhead managed RAG with citations | **Bedrock Knowledge Bases + RetrieveAndGenerate** |
| Vectors must be joined with relational/transactional data; team knows SQL | **Aurora PostgreSQL + pgvector** |
| Billions of vectors, ms latency, hybrid search | **OpenSearch (Serverless)** |
| Queries with exact part numbers/error codes miss results | **Hybrid search** (BM25 + vectors) |
| KB hybrid search silently behaves like semantic | OpenSearch Serverless index lacks a **filterable text field** (or store is S3 Vectors) |
| Retrieved chunks relevant-ish but wrong ones rank first; too many tokens sent | **Reranker** (Amazon Rerank / Cohere Rerank 3.5) — retrieve wide, rerank, keep top few |
| Need reranking over OpenSearch/pgvector results (not a KB) | Standalone **`Rerank` API** |
| "Compare A vs B" questions return only A's docs | **Query decomposition** (`QUERY_DECOMPOSITION`) |
| Follow-up questions ("what about the price?") retrieve garbage | **Conversational query rewrite** / `sessionId` with RetrieveAndGenerate |
| Users type short vague queries; vocabulary differs from docs | **Query expansion / multi-query / HyDE** with a small Bedrock model |
| Extract year/region filters from natural language automatically | **Implicit metadata filtering** (`implicitFilterConfiguration`) |
| Multi-step query pipeline with branching, parallel sub-queries and retries | **Step Functions** (Express for synchronous request path) |
| Let the FM decide when to search the docs | **Converse API tool use** with a `search_knowledge_base` tool |
| Many agents across frameworks must share one retrieval tool | **MCP server** (e.g. Bedrock KB Retrieval MCP server) / **AgentCore Gateway** |
| Expose existing retrieval Lambda as tools for agents with OAuth | **AgentCore Gateway** (Lambda → MCP tool) |
| Central retrieval for many teams, per-team quotas, tenant isolation | **API Gateway + Lambda retrieval service**, identity → metadata filter, AppConfig for settings |
| Need raw chunks + scores to apply own reranking/prompt | **Retrieve** API (not RetrieveAndGenerate) |

## Flashcards

<details>
<summary><b>Q:</b> How does hierarchical chunking in Bedrock Knowledge Bases improve retrieval?</summary>

It searches on small child chunks for precise semantic matches, then returns the larger parent chunk to the FM so the answer has the full surrounding context.

*D1 · Task 1.5 · Hierarchical chunking*
</details>

<details>
<summary><b>Q:</b> What are the three settings for semantic chunking in a Bedrock KB, and what does it cost extra?</summary>

Max tokens, buffer size (how many neighbouring sentences are grouped) and breakpoint percentile threshold. Embeddings are computed at ingestion to find the breakpoints, so ingestion costs more.

*D1 · Task 1.5 · Semantic chunking*
</details>

<details>
<summary><b>Q:</b> How do you plug your own chunking logic into a Bedrock Knowledge Base?</summary>

Configure `customTransformationConfiguration` with an intermediate S3 bucket and a Lambda at the `POST_CHUNKING` step. Set chunking to NONE if the Lambda does all the splitting. The KB role needs `lambda:InvokeFunction`.

*D1 · Task 1.5 · Custom chunking*
</details>

<details>
<summary><b>Q:</b> A team wants to switch an existing KB data source from fixed-size to hierarchical chunking. What must they do?</summary>

Create a new data source (or KB) with the new strategy and re-ingest. The chunking strategy and parser can't be changed after creation.

*D1 · Task 1.5 · Immutable settings*
</details>

<details>
<summary><b>Q:</b> What chunk-size problems arise at each extreme?</summary>

Too small: precise matches but missing context and more vectors to store. Too large: semantic dilution, lower similarity scores, more prompt tokens and "lost in the middle".

*D1 · Task 1.5 · Chunk sizing*
</details>

<details>
<summary><b>Q:</b> Which dimensions does Titan Text Embeddings V2 support, and what is the accuracy trade-off?</summary>

1024 (default), 512 and 256. 512 keeps about 99% of the 1024-dim retrieval accuracy and 256 about 97%, with much lower storage and faster search.

*D1 · Task 1.5 · Titan V2 dimensions*
</details>

<details>
<summary><b>Q:</b> Why must Cohere Embed calls use different `input_type` values at ingestion and query time?</summary>

It's an asymmetric model. Use `search_document` for chunks and `search_query` for queries. Mixing them up quietly lowers recall.

*D1 · Task 1.5 · Asymmetric embeddings*
</details>

<details>
<summary><b>Q:</b> What's the most cost-effective way to embed 50 million chunks for an initial load?</summary>

Bedrock batch inference (`CreateModelInvocationJob`), with JSONL input and output in S3. It's asynchronous and priced below on-demand. Use Lambda/SQS for ongoing incremental updates.

*D1 · Task 1.5 · Batch embeddings*
</details>

<details>
<summary><b>Q:</b> A text query must retrieve matching product photos and video clips. What embedding approach is needed?</summary>

A multimodal embedding model with a shared vector space, such as Amazon Nova Multimodal Embeddings (or Titan Multimodal Embeddings G1 for image + text), for cross-modal retrieval.

*D1 · Task 1.5 · Multimodal embeddings*
</details>

<details>
<summary><b>Q:</b> What do the pgvector operators `<->`, `<=>` and `<#>` compute?</summary>

L2 (Euclidean) distance, cosine distance and negative inner product. Index with the matching opclass, for example `vector_cosine_ops` for `<=>`.

*D1 · Task 1.5 · pgvector*
</details>

<details>
<summary><b>Q:</b> What does an Aurora PostgreSQL cluster need before it can back a Bedrock Knowledge Base?</summary>

pgvector enabled, the RDS Data API enabled, credentials in Secrets Manager, and a table with ID, vector(dims), text and metadata columns mapped in the KB config (plus an HNSW index).

*D1 · Task 1.5 · Aurora as KB store*
</details>

<details>
<summary><b>Q:</b> HNSW vs IVFFlat in pgvector — which do you choose for best recall and latency?</summary>

HNSW gives better recall and latency with no training step, at the cost of slower builds and more memory. IVFFlat builds faster but needs data present first, and recall depends on `lists`/`probes`.

*D1 · Task 1.5 · Vector index type*
</details>

<details>
<summary><b>Q:</b> Users search for "ERR-4012" and semantic search returns unrelated pages. What fixes it?</summary>

Hybrid search, combining BM25 keyword matching for exact tokens with vector similarity. In a KB that's `overrideSearchType: HYBRID`. In OpenSearch it's a hybrid query with a normalization search pipeline.

*D1 · Task 1.5 · Hybrid search*
</details>

<details>
<summary><b>Q:</b> How does OpenSearch combine BM25 and k-NN scores in a hybrid query?</summary>

A search pipeline's `normalization-processor` normalises each subquery's scores (min_max/l2) and combines them (for example a weighted arithmetic mean). Newer versions also support rank-based fusion (RRF).

*D1 · Task 1.5 · OpenSearch hybrid*
</details>

<details>
<summary><b>Q:</b> Why put a reranker after the vector search instead of only increasing `numberOfResults`?</summary>

A cross-encoder reranker scores query–passage relevance more accurately than vector similarity. You retrieve wide (say 50), rerank and keep the top 5. That gives higher precision with fewer prompt tokens, so lower cost and latency.

*D1 · Task 1.5 · Reranking*
</details>

<details>
<summary><b>Q:</b> Which reranker models does Bedrock offer, and how are they called?</summary>

Amazon Rerank 1.0 and Cohere Rerank 3.5. Call them through the standalone `Rerank` API (works with any retriever) or through `rerankingConfiguration` inside Retrieve/RetrieveAndGenerate. They handle text only.

*D1 · Task 1.5 · Bedrock rerank*
</details>

<details>
<summary><b>Q:</b> "Compare the 2023 and 2024 leave policies" retrieves only 2024 chunks. Which KB feature helps?</summary>

Query decomposition (`orchestrationConfiguration.queryTransformationConfiguration.type = QUERY_DECOMPOSITION`). It splits the question into sub-queries that are retrieved separately and pooled.

*D1 · Task 1.5 · Query decomposition*
</details>

<details>
<summary><b>Q:</b> What is HyDE and when does it help?</summary>

Hypothetical Document Embeddings: an FM drafts a hypothetical answer, and that draft is embedded and searched instead of the short question. It helps when short queries sit far from long technical documents in vector space.

*D1 · Task 1.5 · HyDE*
</details>

<details>
<summary><b>Q:</b> How can a KB derive metadata filters such as year or region from a natural-language question?</summary>

Implicit metadata filtering: `implicitFilterConfiguration` with metadata attribute descriptions and a model ARN. The FM builds the filter from the query.

*D1 · Task 1.5 · Implicit filtering*
</details>

<details>
<summary><b>Q:</b> When should query transformation run in Step Functions instead of a single Lambda?</summary>

When there are several steps with branching (Choice), parallel sub-query fan-out (Map/Parallel), retries/catches for throttling, or a need for auditability. Use Express workflows for synchronous, high-volume request paths.

*D1 · Task 1.5 · Step Functions*
</details>

<details>
<summary><b>Q:</b> How do you let an FM decide by itself when to search the knowledge base?</summary>

Define a retrieval tool (`toolSpec` with a JSON input schema) in the Converse API. When `stopReason` is `tool_use`, run the search, return a `toolResult` and call Converse again (agentic RAG).

*D1 · Task 1.5 · Function calling*
</details>

<details>
<summary><b>Q:</b> Several agent frameworks and IDE assistants need the same governed retrieval tool. Which approach fits?</summary>

Expose retrieval through an MCP server, for example the Bedrock KB Retrieval MCP server, or a Lambda behind AgentCore Gateway, which presents it as an MCP tool with OAuth. Any MCP client can then discover and call it.

*D1 · Task 1.5 · MCP*
</details>

<details>
<summary><b>Q:</b> In a central retrieval API, how do you stop users retrieving documents they're not entitled to?</summary>

Authenticate the caller (Cognito/IAM authorizer) and map their identity attributes (tenant, department, clearance) to a mandatory metadata filter on every retrieval call, enforced server-side.

*D1 · Task 1.5 · Retrieval access control*
</details>

<details>
<summary><b>Q:</b> Retrieve vs RetrieveAndGenerate — which one for custom reranking and your own prompt?</summary>

Retrieve. It returns chunks, scores, locations and metadata only, and you control reranking, prompt assembly and generation. RetrieveAndGenerate is the managed end-to-end option with citations and session context.

*D1 · Task 1.5 · Retrieval APIs*
</details>

---
[← Back to Domain 1](README.md)
