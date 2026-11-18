[← Back to Domain 4](README.md)

# ⚡ Task 4.2 · Optimize application performance

> **Exam domain:** D4 · Operational Efficiency and Optimization for GenAI Applications (12%)
>
> - **Skill 4.2.1:** Responsive AI systems: latency vs cost trade-offs and user experience
> - **Skill 4.2.2:** Retrieval performance: relevance and speed
> - **Skill 4.2.3:** FM throughput optimisation
> - **Skill 4.2.4:** FM performance for specific use cases: parameters and A/B testing
> - **Skill 4.2.5:** Efficient resource allocation for FM workloads
> - **Skill 4.2.6:** System-level performance for GenAI workflows

## Key points
- **Latency = TTFT + output tokens ÷ output speed.** Input size drives TTFT, output length drives the rest, and streaming only fixes *perceived* latency. Find which part dominates before choosing a fix (4.2.1).
- **Latency vs cost → choose by how predictable and how urgent the request is:** predictable → pre-compute · urgent with the same model → latency-optimised inference · cost-sensitive → smaller model or cascade · independent parts → parallel calls · nobody waiting → batch (4.2.1).
- **Benchmark on percentiles (P50/P90/P99)** with realistic prompt and output lengths, never on averages (4.2.1).
- **Retrieval speed vs recall is mostly an index-tuning problem:** HNSW `ef_search` is the runtime dial, memory is the scaling limit, and quantisation buys headroom. This file is the home for OpenSearch k-NN tuning (4.2.2).
- **Retrieval relevance:** preprocess the query, use hybrid search for exact terms, then rerank wide → narrow so the FM gets fewer, better chunks (4.2.2; concepts in 1.5.4).
- **Throughput is capped in tokens (TPM) as well as requests (RPM)**, per model per Region. Protect the quota (SQS, bounded concurrency, backoff), then raise the ceiling (CRIS → quota increase → PT/Reserved), and move bulk work to batch (4.2.3).
- **Parameters by use case:** deterministic tasks → low temperature and tight `maxTokens`; creative tasks → higher temperature/topP. `top_k` goes in `additionalModelRequestFields` (4.2.4).
- **Prove every change with an A/B test** on quality, latency and cost (4.2.4).
- **Plan capacity in tokens**, and autoscale self-hosted LLMs on **concurrency**, not invocations, with scheduled scaling ahead of known peaks (4.2.5).
- **System-level wins usually come from fewer hops:** trace every hop, cut round trips, reuse connections, go async for long jobs (4.2.6).

## Notes

### Skill 4.2.1 — Responsive AI systems (latency vs cost trade-offs)

> *Create responsive AI systems to address latency-cost tradeoffs and improve the user experience with FMs (for example, by using pre-computation to perform predictable queries, latency-optimized Amazon Bedrock models for time-sensitive applications, parallel requests for complex workflows, response streaming, performance benchmarking).*

#### Anatomy of LLM latency

| Component | What drives it | Main levers |
| --- | --- | --- |
| **Network + queueing** | Distance to Region, TLS handshakes, throttling and retries | Same-Region clients, connection reuse, keep-alive, enough quota |
| **Prefill (prompt processing)** → shows up as **TTFT** | **Input token count**, model size | Shorter prompts, **prompt caching**, fewer/smaller RAG chunks, smaller model |
| **Decode (generation)** → **output tokens/sec** | Model size, hardware, **output token count** | Smaller/faster model, **latency-optimised inference**, cap `maxTokens`, ask for concise output |
| **Post-processing** | Guardrails, validation, tool calls, extra hops | Run checks in parallel, async guardrail mode for streaming, fewer agent loops |

- **End-to-end latency ≈ TTFT + (output tokens ÷ output tokens per second)**. Output tokens are generated one at a time, so **cutting output length is often the biggest win**.
- **Perceived latency ≠ actual latency.** Streaming does not make the full response arrive sooner, but users see the first words after TTFT. Chat UIs should almost always stream.
- **Agents multiply latency.** Each reasoning step is a full FM call plus a tool call. Reduce the number of steps (better tool descriptions, fewer tools, deterministic workflows for known paths) before tuning each call.

#### Latency vs cost: the decision table

| Requirement | Pattern | Cost effect |
| --- | --- | --- |
| Sub-second answer to a **known/predictable** question | **Pre-compute** + cache (DynamoDB / ElastiCache / CloudFront) | Lowest per request (one batch call, many reads) |
| Interactive chat, **time-sensitive** | **Latency-optimised inference** or a smaller/faster model + streaming | Latency-optimised has a premium price, so use it only where it matters |
| Interactive but cost-sensitive | Smaller model (e.g. Haiku/Nova Lite/Micro class) + **cascade** to a bigger model only on low confidence | Low |
| Complex multi-part task | **Parallel** sub-calls, then an aggregation call | Same tokens, much lower wall-clock time |
| Non-interactive (reports, enrichment, evals) | **Batch inference** (async, discounted, verify in docs for the current discount) | Lowest |
| Repeated long system prompt / documents | **Prompt caching** (see [Task 4.1 · Skill 4.1.4](01-task-4.1-cost-optimization-resource-efficiency.md#skill-414--intelligent-caching-systems)) | Lower cost *and* lower TTFT |

The sections below take each pattern in turn: pre-computation, latency-optimised inference, parallel requests and streaming, then how to benchmark the result.

#### Pre-computation for predictable queries
- **Find the predictable workload:** top-N FAQ intents, product descriptions, nightly summaries of tickets or news, "explain this dashboard" for fixed reports, personalised content that can be generated ahead of time.
- **Pipeline:** EventBridge Scheduler → Step Functions → **Bedrock batch inference** (`CreateModelInvocationJob`, JSONL in S3) → results to S3 → load into DynamoDB/OpenSearch → serve through API Gateway/CloudFront.
- **Freshness:** regenerate when the source data changes (S3 events / DynamoDB Streams → EventBridge) or on a TTL.
- **Combine with a semantic cache** so near-duplicate user questions hit pre-computed answers (caching mechanics are covered in [Task 4.1 · Skill 4.1.4](01-task-4.1-cost-optimization-resource-efficiency.md#skill-414--intelligent-caching-systems)).
- ⚠️ **Gotcha:** pre-computed answers can go stale or ignore user context. Only pre-compute content that is **not user-specific or is safely parameterisable**.

#### Latency-optimised inference in Amazon Bedrock
- Per-request flag. No fine-tuning or set-up needed:

```json
{
  "modelId": "us.anthropic.claude-3-5-haiku-20241022-v1:0",
  "messages": [{"role": "user", "content": [{"text": "Summarise this alert in one line."}]}],
  "inferenceConfig": {"maxTokens": 100, "temperature": 0.2},
  "performanceConfig": {"latency": "optimized"}
}
```

| Model | Model ID (base) | Where (via **cross-Region inference profile**) |
| --- | --- | --- |
| Amazon Nova Pro | `amazon.nova-pro-v1:0` | US East (N. Virginia), US East (Ohio), US West (Oregon) |
| Anthropic Claude 3.5 Haiku | `anthropic.claude-3-5-haiku-20241022-v1:0` | US East (Ohio), US West (Oregon) |
| Meta Llama 3.1 70B Instruct | `meta.llama3-1-70b-instruct-v1:0` | US East (Ohio), US West (Oregon) |
| Meta Llama 3.1 405B Instruct | `meta.llama3-1-405b-instruct-v1:0` | US East (Ohio) |

- **Status:** the AWS docs label it **preview**, and the model list changes, so verify in docs for current models.
- **Default** is `"standard"`. Requests are only optimised when you ask for it.
- **Quota fallback:** when the latency-optimised usage quota is reached, Bedrock **tries to serve at standard latency and bills at standard rates**. The latency mode actually served shows in the **API response and CloudTrail**, and its metrics appear in CloudWatch under `model-id+latency-optimized`.
- **Llama 3.1 405B** latency-optimised requests support up to **~11K total input + output tokens**. Larger requests fall back to standard.
- 🎯 **Exam cue:** "time-sensitive customer-facing app, keep the same model and accuracy, faster responses, no re-architecture" → **latency-optimised inference**. "Reduce latency by moving off a large model" → **smaller model / distillation** (different trade-off: accuracy may drop).

#### Parallel requests for complex workflows
- **When:** a task breaks into **independent** sub-tasks: summarise 10 documents, extract several field groups, call several specialist models, run guardrail checks alongside generation. Latency becomes the **slowest branch**, not the sum of all branches.
- **Step Functions:**
  - **Parallel state** → fixed set of branches (e.g. sentiment + entities + summary of the same text).
  - **Map state** → dynamic array (one branch per document/chunk). Use `MaxConcurrency` to stay under Bedrock RPM/TPM. **Distributed Map** handles very large fan-outs from S3 (thousands of items).
  - Native **`bedrock:invokeModel`** optimised integration, with `Retry` on throttling errors (error names: verify in docs).
- **Code:** `asyncio` with a semaphore (bound concurrency) or `concurrent.futures.ThreadPoolExecutor`. boto3 clients are thread-safe; create one client and reuse it.

```python
import asyncio, boto3
from functools import partial

brt = boto3.client("bedrock-runtime")          # reuse one client (connection pooling)
sem = asyncio.Semaphore(8)                     # bound concurrency to protect RPM/TPM quota

async def ask(prompt: str) -> str:
    async with sem:
        loop = asyncio.get_running_loop()
        resp = await loop.run_in_executor(None, partial(
            brt.converse,
            modelId="us.amazon.nova-lite-v1:0",
            messages=[{"role": "user", "content": [{"text": prompt}]}],
            inferenceConfig={"maxTokens": 300, "temperature": 0.2},
        ))
        return resp["output"]["message"]["content"][0]["text"]

async def fan_out(chunks):
    partials = await asyncio.gather(*(ask(f"Summarise:\n{c}") for c in chunks))
    return await ask("Combine these summaries:\n" + "\n---\n".join(partials))  # aggregation step
```

- **Map-reduce summarisation** = parallel "map" summaries + one "reduce" call. This beats one huge prompt on latency and on context-window limits.
- ⚠️ Parallelism **multiplies concurrent RPM/TPM demand**, so throttling rises. Bound concurrency and add backoff with jitter (mechanics in [Task 2.4 · Skill 2.4.3](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md#skill-243--resilient-fm-systems)).

#### Response streaming
The performance angle only. APIs (`ConverseStream`, `InvokeModelWithResponseStream`), front-end transports (WebSocket API, Lambda response streaming, API Gateway REST response streaming) and their limits → [Task 2.4 · Skill 2.4.2](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md#skill-242--real-time-ai-interaction-systems--pq24).
- Streaming does **not** reduce cost or total generation time. It improves **perceived** responsiveness and lets users cancel early, which does save output tokens if you stop the stream.
- **Guardrails + streaming latency:** **synchronous** mode buffers and scans chunks before release (safer, adds latency). **Asynchronous** mode streams immediately and scans in the background (lower latency, but some unsafe text may be shown before it is caught). Configuration → [Task 3.1 · Skill 3.1.2](../domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md#skill-312--content-safety-frameworks-for-harmful-outputs).

#### Performance benchmarking
- **Metrics to capture:** TTFT, output tokens/sec, end-to-end latency, error/throttle rate, cost per request, and quality score (from evaluation, see Domain 5).
- **Report percentiles:** **P50** (typical), **P90/P99** (tail; what users complain about). LLM latency has long tails because output lengths vary.
- **Realistic load:** replay real prompt and completion length distributions. Short synthetic prompts flatter TTFT.
- **Tools:** custom load generator (Locust/k6/`asyncio`) against a staging endpoint. **CloudWatch Bedrock metrics** (`InvocationLatency`, `InputTokenCount`, `OutputTokenCount`, `InvocationThrottles`; TTFT metric availability: verify in docs). **SageMaker Inference Recommender** for instance-type load tests. **Bedrock model evaluation** for the quality side.
- **Compare like for like:** same prompts, same `maxTokens`, same Region and concurrency. Record model ID, inference profile, latency mode and parameters for every run.

---

### Skill 4.2.2 — Retrieval performance (relevance and speed)

> *Enhance retrieval performance to improve the relevance and speed of retrieved information for FM context augmentation (for example, by using index optimization, query preprocessing, hybrid search implementation with custom scoring).*

Basics (chunking, embeddings, hybrid and rerank concepts) are covered in [Task 1.5](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md), and store architecture and sharding in [Task 1.4 · Skill 1.4.3](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md#skill-143--high-performance-vector-database-architectures-at-scale). This section is about **tuning**: first the index itself (speed and memory), then filtering, then the relevance levers that sit around the search (query preprocessing, hybrid scoring, reranking).

#### Index optimisation (OpenSearch k-NN)

| Knob | Where | Effect | Trade-off |
| --- | --- | --- | --- |
| `m` (graph links per node) | Index mapping (build) | Higher → better recall, more memory, slower indexing | Typical 16–64; OpenSearch default 16 (verify in docs for your version) |
| `ef_construction` | Index mapping (build) | Higher → better-quality graph, better recall | Slower ingestion; default 100 (verify in docs) |
| `ef_search` | Index setting / query (search) | Higher → better recall, **higher query latency** | The main **runtime** recall vs latency dial; default 100 (verify in docs) |
| Engine `faiss` vs `lucene` | Mapping | Faiss: large scale, quantisation, GPU-friendly build. Lucene: smaller indexes, good efficient filtering | nmslib is deprecated in newer versions (verify in docs) |
| `space_type` | Mapping | Must match the embedding model's training (`cosinesimil`, `l2`, `innerproduct`) | Wrong metric → silently poor relevance |
| Shards/replicas | Index settings | Replicas → more query throughput; shards → parallelism and bigger capacity | Too many small shards add overhead |
| Segment count | Force-merge after bulk load | Fewer segments → faster k-NN search (one graph per segment) | Force-merge is expensive. Run it on read-heavy, rarely-updated indexes |

- 🎯 **PQ35** **Memory is the main scaling constraint for HNSW.** OpenSearch Service gives **half the instance RAM to the JVM heap (up to 32 GiB)**, and k-NN graphs use **up to 50% of the remaining half** by default (`knn.memory.circuit_breaker.limit`). A 32 GiB node therefore holds **~8 GiB of graphs**. Watch the **`KNNGraphMemoryUsage`** CloudWatch metric.
- **Memory estimate (HNSW, float32):** ≈ `1.1 × (4 × dimension + 8 × m)` bytes per vector. For example, 10 M × 1024-dim with m=16 ≈ 1.1 × (4096 + 128) × 10 M ≈ **46 GB** → quantise or use larger/more nodes.
- **Quantisation / compression** (fit more vectors, lower latency, small recall cost):
  - **fp16** (Faiss scalar quantisation) → ~2× smaller
  - **byte vectors** (int8) → ~4× smaller
  - **binary** → ~32× smaller (needs an embedding model that supports binary, e.g. Titan V2 binary output)
  - **product quantisation (PQ)** → very high compression, lower recall
  - **disk-based vector search** (`mode: on_disk` + `compression_level`) → compressed vectors in memory, **rescoring** with full-precision vectors from disk. Good for large, cost-sensitive corpora that can accept slightly higher latency (verify in docs for supported versions and defaults).
- **Dimension reduction at source:** Titan Text Embeddings V2 at **512 or 256 dims** keeps ~99% / ~97% of the 1024-dim accuracy for far less memory and faster search.
- **Query hygiene:**
  - Always set **`size`**. Otherwise you get `k` results **per shard/segment**.
  - Max `k` is **10,000**.
  - Use **`_msearch`** to batch many vector queries in one round trip.
  - **Warm up** indexes after deploys or restarts (k-NN warmup API) so the first queries don't pay the graph load cost.

> [!IMPORTANT]
> **🎯 Practice question 35 — millions of product embeddings, hourly updates, slow and failing similarity search at peak**
>
> - **Amazon OpenSearch Service, multi-node cluster** → k-NN load is spread across data nodes (shards and replicas), so you can **scale horizontally** for peak events.
> - **Vector-specific index settings** (HNSW parameters, enough graph memory) + **k-NN memory circuit breakers** → stop failures caused by running out of memory.
> - **Separate queues for vector operations** absorb surges, for example hourly bulk updates without slowing queries.
> - ❌ **Bedrock KB with custom partitioning:** built for RAG over documents, not a high-QPS recommendation engine.
> - ❌ **Neptune:** built for graph relationships, not peak-scale similarity search.
> - ❌ **DynamoDB with "sparse vector compression":** no native vector search, so it's all custom code.

#### Filtered search
- **Post-filtering** (`post_filter`) runs k-NN first and then filters, so you can get **fewer than k** results, or none, when the filter is selective.
- **Efficient filtering** (a `filter` clause **inside** the `knn` query, Lucene/Faiss engines) filters during the search. Results stay accurate and the count is predictable, which is better for multi-tenant (`tenant_id`) and metadata filters.

```json
GET kb-index/_search
{
  "size": 5,
  "query": {
    "knn": {
      "embedding": {
        "vector": [0.12, -0.03, "..."],
        "k": 20,
        "filter": { "bool": { "must": [
          { "term":  { "tenant_id": "acme" } },
          { "range": { "published": { "gte": "2025-01-01" } } }
        ]}}
      }
    }
  }
}
```

- **Bedrock Knowledge Bases:** metadata **filters** go in `retrievalConfiguration.vectorSearchConfiguration.filter`, and **implicit filtering** lets an FM derive filters from the user query. Operators and design → [Task 1.4 · Skill 1.4.2](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md#skill-142--metadata-frameworks-for-search-precision-and-context-awareness--pq40).

#### Query preprocessing
The techniques (normalise, standalone rewrite for multi-turn chat, multi-query expansion, decomposition with KB `QUERY_DECOMPOSITION`, HyDE, routing to the right index or KB) are explained in [Task 1.5 · Skill 1.5.5](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md#skill-155--query-handling-systems-expansion-decomposition-transformation). The performance angle:
- **Rewriting a follow-up into a standalone query** is the fix for multi-turn RAG retrieving the wrong chunks.
- **Routing** to one index or KB is also a speed win, because you search less.
- Expansions can run **in parallel**, and their results are then merged.
- ⚠️ Each preprocessing FM call **adds latency and cost**. Use a **small fast model** for rewriting, cache the rewrites, and skip rewriting for queries that are already well-formed.

#### Hybrid search with custom scoring 🎯 PQ7
Why hybrid (BM25 catches exact terms such as product codes, error IDs and acronyms that embeddings blur), the KB `"overrideSearchType": "HYBRID"` setting and a full search-pipeline example are in [Task 1.5 · Skill 1.5.4](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md#skill-154--advanced-search-architectures-semantic-hybrid-reranking). Tuning it:
- **Score fusion options:** the search pipeline's **normalization processor** normalises with `min_max` or `l2` and combines with `arithmetic_mean`, `geometric_mean` or `harmonic_mean`. **Weights** are listed in the same order as the sub-queries. **Reciprocal rank fusion (RRF)** combines ranks instead of scores (OpenSearch 2.19+ via the score-ranker processor; verify in docs).
- **Tuning the weights:** lean towards **lexical** for ID- and terminology-heavy corpora (support tickets, part numbers) and towards **semantic** for natural-language questions. Tune on a **labelled query set** (recall@k, MRR, nDCG), not by eye.
- **Custom scoring signals:** boost by recency (`function_score` decay on date), source authority or popularity, and **metadata boosts** (same product line as the user).

#### Reranking for relevance *and* speed 🎯 PQ8
Reranker models and APIs (Amazon Rerank, Cohere Rerank, KB `rerankingConfiguration`, the standalone `Rerank` API) → [Task 1.5 · Skill 1.5.4](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md#skill-154--advanced-search-architectures-semantic-hybrid-reranking).
- **Retrieve wide, rerank narrow:** retrieve the top 20–50 cheaply, rerank with a cross-encoder, then pass the **top 3–5** to the FM.
- Net effect: **better relevance** and **fewer context tokens**, so lower FM latency and cost. This usually outweighs the added rerank latency.
- Don't rerank when the first-stage retrieval is already precise or latency budgets are extremely tight. Measure both ways.

---

### Skill 4.2.3 — FM throughput optimisation

> *Implement FM throughput optimization to address the specific throughput challenges of GenAI workloads (for example, by using token processing optimization, batch inference strategies, concurrent model invocation management).*

#### Why GenAI throughput is different
- Throughput is bounded by **tokens**, not just requests: **TPM (tokens per minute)** and **RPM (requests per minute)** quotas **per model, per Region, per account** (see Service Quotas).
- Requests vary wildly in cost (200 tokens vs 50,000 tokens), so RPM alone is a poor capacity signal.
- **Quota burndown:** Bedrock reserves tokens at request start (input + `maxTokens`) and adjusts afterwards. Some newer models count output tokens at a multiplier. **An oversized `maxTokens` wastes quota even if the model writes less.** (Verify in docs for the current burndown rules per model.)

#### Token processing optimisation
Every token saved for cost ([Task 4.1 · Skill 4.1.1](01-task-4.1-cost-optimization-resource-efficiency.md#skill-411--token-efficiency-systems)) also frees quota. The throughput-specific points:
- **Trim input** (fewer, reranked chunks, rolling history summaries) and **bound output** (a realistic `maxTokens`, stop sequences, structured output). Because of quota burndown, a tight `maxTokens` matters for throughput even when answers are short.
- **Cache static prefixes:** with **prompt caching**, cached tokens are processed faster and cheaper (details in [Skill 4.1.4](01-task-4.1-cost-optimization-resource-efficiency.md#skill-414--intelligent-caching-systems)).
- **Batch small items:** classify 20 short reviews in **one** request (JSON array in, JSON array out) instead of 20 requests. This cuts RPM and repeated system-prompt tokens. ⚠️ Too many items per request hurts accuracy and makes partial failures harder to handle.
- **Count before sending:** the **CountTokens** API (model support: verify in docs) or tokenizer estimates. Route oversized prompts to summarisation or a bigger-context model.

#### Batch inference strategies
- **Bedrock batch inference** (`CreateModelInvocationJob`, JSONL in and out of S3) is discounted (cost view in [Skill 4.1.3](01-task-4.1-cost-optimization-resource-efficiency.md#skill-413--high-performance-fm-systems-utilisation-and-throughput)). The throughput point is that it **doesn't consume your on-demand interactive quota in the same way** (separate batch quotas, verify in docs). There is a minimum records-per-job quota (verify in docs).
- **Use for:** nightly enrichment, back-filling embeddings/summaries, evaluation runs, pre-computation (4.2.1), large classification jobs.
- **Not for:** anything with a user waiting. Results arrive in **minutes to hours**.
- **Embeddings at scale:** batch inference for Titan/Cohere embeddings, or Lambda/Glue jobs with bounded concurrency. Embedding models also have their own TPM/RPM quotas.
- **SageMaker:** **Batch Transform** for offline jobs on your own models. **Async Inference** for large payloads or long jobs with queueing and scale-to-zero (see [Task 2.2 · Skill 2.2.1](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-221--deploy-fms-based-on-application-needs-and-performance-requirements)).

#### Concurrent model invocation management
- **Buffer with SQS:** producers enqueue requests, and a consumer fleet (Lambda with **maximum concurrency on the SQS event source**, or ECS workers) calls Bedrock at a controlled rate. Spikes are absorbed and requests are not dropped.
- **Bound concurrency** in code (semaphores), Step Functions `MaxConcurrency`, Lambda **reserved concurrency** → stay under TPM/RPM.
- **Per-tenant fairness:** separate queues or token buckets per tenant (DynamoDB counters, API Gateway **usage plans**) so one tenant can't use up the shared model quota.
- **Backoff + jitter** on `ThrottlingException` (SDK `adaptive` retry mode does client-side rate limiting). Details in [Task 2.4 · Skill 2.4.3](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md#skill-243--resilient-fm-systems).
- **Priority lanes:** interactive traffic → on-demand / Priority tier. Background traffic → batch or Flex tier (service tiers in [Task 2.2 · Skill 2.2.1](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-221--deploy-fms-based-on-application-needs-and-performance-requirements)).

#### Raising the ceiling
When protecting the quota isn't enough, raise it. Try the no-commitment options first. When a commitment pays off is covered in [Skill 4.1.3](01-task-4.1-cost-optimization-resource-efficiency.md#skill-413--high-performance-fm-systems-utilisation-and-throughput).

| Option | What it gives | When |
| --- | --- | --- |
| **Cross-Region inference (geographic/global profile)** | Bursts routed to other Regions' capacity, so higher effective throughput | Spiky on-demand traffic, fewer throttles, no commitment. Check data-residency needs (geo vs global) |
| **Service Quotas increase** | Higher account TPM/RPM | Steady growth. Request ahead of launches |
| **Provisioned Throughput** (model units) | Dedicated, predictable throughput | Steady high volume. **Required for some custom models.** Billed hourly whether used or not |
| **Reserved service tier** | Reserved TPM capacity with overflow to Standard (verify in docs for terms) | Predictable baseline with guaranteed capacity |
| **Batch inference** | Moves bulk work off the interactive quota | Non-real-time work |
| **Smaller model** | More tokens/sec per unit of capacity | When quality allows |

---

### Skill 4.2.4 — FM performance for specific use cases (parameters and A/B tests)

> *Enhance FM performance to achieve optimal results for specific GenAI use cases (for example, by using model-specific parameter configurations, A/B testing to evaluate improvements, appropriate temperature and top-k/top-p selection based on requirements).*

#### What the parameters do (from the Bedrock docs)
- **Temperature** reshapes the next-token probability distribution. **Low** → steeper, more deterministic. **High** → flatter, more random and diverse.
- **Top K** → only the **K most likely** tokens are candidates (a fixed-size pool).
- **Top P** → only the tokens inside the **top P cumulative probability** are candidates (the pool size varies by token).
  - Example: candidates horses 0.7, zebras 0.2, unicorns 0.1.
  - Top K = 2 → horses and zebras.
  - Top P = 0.7 → horses only.
  - Top P = 0.9 → horses and zebras.
- **Length:** **maxTokens** (hard cap; output is cut off at the limit with `stopReason: max_tokens`), **stop sequences** (stop when a string is generated), and **penalties** (repetition/frequency/presence on some models).
- Defaults and ranges are **model-specific**. Check each model's parameter page.

#### Converse API: where each parameter goes

```python
resp = brt.converse(
    modelId="us.anthropic.claude-3-5-haiku-20241022-v1:0",
    system=[{"text": "Extract invoice fields as JSON. No prose."}],
    messages=[{"role": "user", "content": [{"text": invoice_text}]}],
    inferenceConfig={                      # common, model-agnostic
        "maxTokens": 400,
        "temperature": 0.0,
        "topP": 0.9,
        "stopSequences": ["</json>"],
    },
    additionalModelRequestFields={"top_k": 50},   # model-specific (field name varies by provider)
)
print(resp["stopReason"])   # end_turn | max_tokens | stop_sequence | tool_use | guardrail_intervened | ...
```

- **`inferenceConfig`**: `maxTokens`, `temperature`, `topP`, `stopSequences`.
- **`additionalModelRequestFields`**: anything model-specific, such as **top_k** (Anthropic `top_k`, other providers use other names), reasoning/thinking budgets, and penalties.
- ⚠️ `stopReason == "max_tokens"` → truncated output (broken JSON, cut-off answers). Raise `maxTokens` or ask for shorter output (see Domain 5 troubleshooting).

#### Parameter selection by use case 🎯 PQ10

| Use case | Temperature | Top P / Top K | maxTokens / stop | Notes |
| --- | --- | --- | --- | --- |
| **Data extraction, JSON/structured output** | 0 – 0.2 | Leave default or topP ≈ 0.9 | Tight cap, stop sequence after the closing tag | Add a JSON schema / tool-use forcing for format reliability |
| **Classification / routing** | 0 | Default | Very small (e.g. 5–20) | Give the label set in the prompt and return only the label |
| **Code generation** | 0 – 0.3 | Default | Generous enough for the full function | Low randomness gives fewer syntax errors |
| **RAG Q&A / factual** | 0 – 0.3 | Default | Moderate | Low temperature reduces invented detail. Grounding matters more |
| **Summarisation** | 0.2 – 0.5 | Default | Cap to the summary length | Ask for a bullet count or word budget |
| **Conversational assistant** | 0.5 – 0.7 | topP 0.9 | Moderate | Balanced tone variation |
| **Creative writing / brainstorming / marketing variants** | 0.7 – 1.0 | topP 0.9–0.95, higher top K | Larger | More diversity. Generate several candidates |
| **Synthetic data generation** | 0.8 – 1.0 | High | Per record | Diversity matters. Validate the outputs |

- **Rule of thumb:** change **temperature *or* topP**, not both at once. Both control randomness, and tuning both makes results hard to attribute.
- **Temperature 0 is not a guarantee of determinism** across calls or model versions. For strict repeatability, use deterministic code, caching or templates (for example, text-to-SQL then execute, instead of letting the FM compute numbers).
- **Reasoning models / extended thinking:** a bigger thinking budget gives better quality on hard problems but **higher latency and output-token cost**. Some models restrict temperature/top_p when thinking is enabled (verify in docs per model).

> [!IMPORTANT]
> **🎯 Practice question 10 — inaccurate answers, 200 ms–3 s latency swings, identical queries → different outputs**
>
> Fix it with **a combination** of controls, each aimed at one symptom:
> - **Lower temperature** (and top-p) → the model picks higher-probability tokens, so **identical queries give consistent outputs**.
> - **Prompt templates** (Prompt Management) → **standardised inputs**.
> - **Response caching / prompt caching** → **steadier latency** on repeated queries.
> - **Bedrock Guardrails** (for example contextual grounding) → catch **inaccurate or ungrounded** output, which matters for clinical accuracy.
> - **Retries with exponential backoff (+ jitter)** and **logging/monitoring** → **reliability**.
> - ❌ **More memory or provisioned concurrency:** Bedrock is fully managed, so there's no compute to tune.
> - ❌ **Custom model + quantisation on SageMaker:** a lot of work, and it doesn't fix consistency or accuracy.
> - ❌ **EKS microservices + Kafka:** improves scale and decoupling, not output quality.

#### A/B testing improvements
- **What to test:** model, model version, prompt version, temperature/topP, `maxTokens`, retrieval settings (k, hybrid weights, reranker on/off).
- **How to split traffic:**
  - **Prompt Management** prompt versions/variants, with the app choosing a version by weight
  - **AppConfig** feature flags or percentage rollout
  - **API Gateway canary** stage settings
  - **SageMaker production variants** with weights (`UpdateEndpointWeightsAndCapacities`)
  - Bedrock **Flows** with conditional routing
- **What to measure:** quality (LLM-as-a-judge / human ratings / task success), **latency (P50/P90)**, **tokens and cost per request**, user feedback (thumbs up/down), and business KPIs (deflection, conversion).
- **Discipline:** a fixed evaluation set for offline comparison *before* the online A/B test, enough samples for significance, one change at a time, and logging the variant ID with each request (CloudWatch dimensions / invocation log metadata). Evaluation tooling is covered in Domain 5.

---

### Skill 4.2.5 — Efficient resource allocation for FM workloads

> *Create efficient resource allocation systems specifically for FM workloads (for example, by using capacity planning for token processing requirements, utilization monitoring for prompt and completion patterns, auto-scaling configurations that are optimized for GenAI traffic patterns).*

This skill is the Domain 4 home for capacity planning and for autoscaling self-hosted models on GenAI traffic. The cost side of the same decisions (purchase options, PT right-sizing) is in [Skill 4.1.3](01-task-4.1-cost-optimization-resource-efficiency.md#skill-413--high-performance-fm-systems-utilisation-and-throughput). LLM serving internals (model loading, Fast Model Loader) are in [Task 2.2 · Skill 2.2.2](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-222--address-the-llm-specific-challenges-of-deployment).

#### Capacity planning in tokens
1. **Measure the workload:** peak RPM, **average and P95 input tokens**, **average and P95 output tokens**, and the concurrency profile (hour of day, day of week, launches). Plan for **output tokens** separately, because they drive both latency and cost.
2. **Convert to tokens/min:** `TPM_peak ≈ RPM_peak × (avg_input + avg_output)` (or `peak RPS × 60 × (avg input + avg output)`). Plan on P95 lengths, with headroom of 20–30%, and compare the result with the account **TPM/RPM quotas** for that model and Region in Service Quotas.
3. **Account for quota burndown:** reserved `maxTokens` and output multipliers may make *quota* TPM higher than the TPM actually used (verify in docs).
4. **Pick the capacity model:**
   - on-demand within quota (+ CRIS)
   - quota increase
   - **Provisioned Throughput** (model units sized from the TPM target; tokens per MU vary by model, so verify in docs or ask AWS)
   - Reserved tier
   - SageMaker instances (from load tests: tokens/sec per instance at the target latency)
   - and in every case, flatten peaks with **queues (SQS) and backoff**
5. **Check the other limits too:** **max concurrency** for agents, KB queries and Lambda.
6. **Re-plan on change:** a new prompt version that adds 2,000 tokens of context can **double** TPM overnight.

#### Monitoring prompt vs completion patterns
- **Sources:** CloudWatch `AWS/Bedrock` metrics per `ModelId` (full list in [Task 4.3 · Skill 4.3.2](03-task-4.3-monitoring-observability.md#skill-432--comprehensive-genai-monitoring-and-kpis)). Build the **input:output ratio** and **tokens per request** with metric math. **Model invocation logs** → Logs Insights by application, user or prompt version. **Application inference profiles** with tags attribute usage per team or app.
- **What the patterns tell you:**
  - High input, low output → RAG/context-heavy → **prompt caching**, fewer chunks, rerank
  - Low input, high output → generation-heavy → cap output, smaller model, latency-optimised inference
  - Rising tokens per request over time → prompt bloat or conversation history growth → summarise history
- **PT utilisation:** under-used or saturated Provisioned Throughput → right-size it as in [Skill 4.1.3](01-task-4.1-cost-optimization-resource-efficiency.md#skill-413--high-performance-fm-systems-utilisation-and-throughput).

#### Autoscaling for GenAI traffic (SageMaker)
- **Why invocations-per-instance is weak for LLMs:** one request may take 0.5 s or 30 s depending on output length, so **concurrency** reflects load better.
- **Predefined target-tracking metrics:**
  - Endpoint variants: `SageMakerVariantInvocationsPerInstance` (the classic metric), **`SageMakerVariantConcurrentRequestsPerModelHighResolution`**
  - Inference components: `SageMakerInferenceComponentInvocationsPerCopy`, **`SageMakerInferenceComponentConcurrentRequestsPerCopyHighResolution`**
  - The **high-resolution concurrency** metrics enable **faster scale-out** on bursts.
- **GPU utilisation / GPU memory** → **custom CloudWatch metrics** with target tracking or step scaling (not predefined).
- **Policy types** (Application Auto Scaling):
  - **target tracking** (default choice)
  - **step scaling** (aggressive reaction to big spikes)
  - **scheduled scaling** (diurnal or business-hours traffic: pre-warm before 9 am, scale in at night)
- **Inference components:** pack multiple models/copies per instance, scale **copies** independently (instances scale under them), and **scale to zero copies** for idle models (verify in docs for current support).
- **Serverless/async options** for spiky or idle-heavy traffic: SageMaker **Serverless** (CPU only, small models) or **Async** (scale to zero, queue-backed). Bedrock on-demand is the "no capacity management" default.

```python
aas = boto3.client("application-autoscaling")
aas.register_scalable_target(ServiceNamespace="sagemaker",
    ResourceId="endpoint/llm-ep/variant/AllTraffic",
    ScalableDimension="sagemaker:variant:DesiredInstanceCount", MinCapacity=1, MaxCapacity=8)
aas.put_scaling_policy(PolicyName="llm-tt", ServiceNamespace="sagemaker",
    ResourceId="endpoint/llm-ep/variant/AllTraffic",
    ScalableDimension="sagemaker:variant:DesiredInstanceCount", PolicyType="TargetTrackingScaling",
    TargetTrackingScalingPolicyConfiguration={
        "TargetValue": 4.0,   # concurrent requests per instance
        "PredefinedMetricSpecification": {"PredefinedMetricType": "SageMakerVariantConcurrentRequestsPerModelHighResolution"},  # verify metric name in docs
        "ScaleInCooldown": 600, "ScaleOutCooldown": 60})
```

- **LLM cold starts are slow** (pulling a large container + loading weights takes minutes), so scale out **before** demand: scheduled scaling, a lower target value, higher minimum capacity, keeping one warm copy. Use faster loading features (Fast Model Loader, cached containers) to shorten scale-out.
- **Cooldowns and scale-in protection:** scale out fast and scale in slowly (as in the example: 60 s out, 600 s in; the target-tracking default is 300 s). Use `DisableScaleIn` for scale-out-only policies.

---

### Skill 4.2.6 — System-level performance for GenAI workflows

> *Optimize FM system performance for GenAI workflows (for example, by using API call profiling for prompt-completion patterns, vector database query optimization for retrieval augmentation, latency reduction techniques specific to LLM inference, efficient service communication patterns).*

#### API call profiling
- **Trace every hop** with **X-Ray / OpenTelemetry (ADOT)**: API Gateway → Lambda → retrieval (KB/OpenSearch) → Bedrock → guardrail → post-processing. Find out which segment dominates P90. It is often retrieval, an agent loop, or a guardrail, not the FM itself.
- **Profile prompt-completion patterns** per endpoint and use case:
  - input tokens
  - output tokens
  - TTFT
  - total latency
  - cache hit rate (prompt cache + semantic cache)
  - retries/throttles
  - Log the **prompt version and model ID** with each call
- **Typical findings and fixes:**
  - Same system prompt resent on every call → **prompt caching**
  - 10 chunks sent and 2 used → **rerank + lower k**
  - Long free-text answers parsed into fields → **structured output + tight maxTokens**
  - Sequential independent calls → **parallelise**
  - Repeated identical questions → **semantic or exact cache**
  - Agent takes 6 steps for a known flow → **deterministic Step Functions workflow**

#### Vector database query optimisation (for RAG)
- Right-size **k**, using **efficient filtering**, `ef_search` tuning, quantisation and warm indexes (4.2.2).
- **Co-locate** the vector store, app and Bedrock in the **same Region**, and use **VPC endpoints** to avoid NAT and internet hops.
- **Cache embeddings** of frequent queries, and **batch** embedding calls during ingestion.
- **Scale reads:** more replicas (OpenSearch managed) or OCUs (Serverless, which scale automatically within the min/max you set). **Separate indexes** for hot domains so you search only what's needed.
- **pgvector (Aurora/RDS):** **HNSW** index (`CREATE INDEX … USING hnsw (embedding vector_cosine_ops)`) vs **IVFFlat** (needs `lists`/`probes` tuning and data before building). Tune `hnsw.ef_search`, and filter on indexed columns.

#### LLM-inference latency reduction techniques

| Technique | Mechanism | Notes |
| --- | --- | --- |
| **Shorter prompts** | Less prefill → lower TTFT | Remove redundancy; compress RAG context |
| **Prompt caching** | Reuse the processed prefix → lower TTFT and cost | Static content first, dynamic last; see [Task 4.1 · Skill 4.1.4](01-task-4.1-cost-optimization-resource-efficiency.md#skill-414--intelligent-caching-systems) |
| **Shorter outputs** | Fewer decode steps | `maxTokens`, format instructions, structured output |
| **Smaller / distilled model** | Faster per-token generation | Check quality with evals; **model distillation** in Bedrock |
| **Latency-optimised inference** | Faster serving of the same model | Preview, limited models/Regions (4.2.1) |
| **Streaming** | Better perceived latency | No change in total time |
| **Speculative decoding** (self-hosted) | A small draft model proposes tokens and the big model verifies them in parallel | Supported in self-hosted inference servers (e.g. LMI/vLLM on SageMaker). Awareness-level (verify in docs) |
| **Quantised self-hosted models** | Less memory bandwidth per token | INT8/INT4/FP8 on SageMaker with a small accuracy trade-off |
| **Continuous batching / KV cache** (self-hosted) | Higher GPU utilisation | Handled by modern serving containers ([Task 2.2 · Skill 2.2.2](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-222--address-the-llm-specific-challenges-of-deployment)) |
| **Fewer round trips** | Combine steps, parallelise, avoid needless agent iterations | Often the biggest system-level win |

#### Efficient service communication patterns
- **Reuse SDK clients and connections:** create clients **outside the Lambda handler** and enable keep-alive (`botocore.config.Config(tcp_keepalive=True)`). Raise `max_pool_connections` (default 10) for high-concurrency workers. This avoids repeated TLS handshakes.
- **Long-running calls behind proxies and NAT:** idle connections can be dropped (see the NAT/VPC endpoint idle timeout in [Task 2.4 · Skill 2.4.3](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md#skill-243--resilient-fm-systems)), so prefer streaming for long generations.
- **Regional proximity:** run compute in the Bedrock Region (or call through CRIS from the home Region), keep the vector store in the same Region, and use **PrivateLink** to stay on the AWS network.
- **Async and event-driven over synchronous chains:** for anything over a few seconds, return `202 + jobId` and process through SQS/Step Functions/EventBridge. Notify by WebSocket, polling or SNS. This avoids API Gateway timeouts and wasted idle compute.
- **Avoid chatty microservices** on the hot path. Each extra synchronous hop adds P99 latency. Combine retrieval and generation in one service where it makes sense, or use **RetrieveAndGenerate** to cut round trips.
- **Payload efficiency:** send S3 URIs for large documents/images where the API supports them, compress large payloads between services, and use gRPC/HTTP2 for internal high-throughput paths when self-hosting.
- **Timeout budgets:** set client timeouts per hop that add up to less than the overall SLA, with retries only on idempotent and retriable errors.

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Time-sensitive app, keep the same Bedrock model, faster responses, minimal change | **Latency-optimised inference** (`performanceConfig.latency = optimized`) via a CRIS profile |
| Latency-optimised quota exhausted. What happens? | Request is **served at standard latency and billed at standard rates** |
| Users complain the chatbot "feels slow", but total generation time is acceptable | **Stream** the response (ConverseStream + WebSocket/response streaming) |
| TTFT is high on long prompts that share the same system prompt and documents | **Prompt caching** + shorter prompts |
| Total latency dominated by long outputs | Cap **`maxTokens`**, ask for concise/structured output, smaller or faster model |
| Same FAQs asked thousands of times a day | **Pre-compute** with batch inference and serve from a cache (+ semantic cache) |
| Summarise 50 documents quickly | **Parallel map** summaries (Step Functions Map / asyncio with bounded concurrency) → reduce step |
| Fan-out causes `ThrottlingException` | Bound concurrency (`MaxConcurrency`, semaphores), **SQS buffer**, backoff + jitter, CRIS |
| Measure latency for an SLA | **P50/P90/P99** TTFT and end-to-end with realistic token lengths, not averages |
| Vector search recall is fine but latency too high | Lower **`ef_search`**, quantise, more replicas, force-merge segments, warm up |
| Recall too low on OpenSearch HNSW | Raise `ef_search` (query) / `m` and `ef_construction` (rebuild); check `space_type` matches the embedding model |
| OpenSearch k-NN nodes under memory pressure / circuit breaker | Watch **`KNNGraphMemoryUsage`**. Quantise (fp16/byte/binary), disk mode, lower dims, bigger nodes |
| Filtered vector query returns fewer than k results | Use **efficient filtering inside the knn query**, not `post_filter` |
| Queries with product codes / error IDs miss relevant docs | **Hybrid search** (BM25 + vector), with lexical weight tuned up |
| 🎯 **PQ7** Combine BM25 and vector scores correctly | OpenSearch **search pipeline normalization processor** (`min_max`/`l2` + weighted mean) or RRF |
| Follow-up question in chat retrieves the wrong chunks | **Rewrite** the query into a standalone query using conversation history |
| Too many chunks inflate tokens and latency | **Retrieve wide → rerank → top 3–5** |
| Nightly enrichment of 2 M records | **Bedrock batch inference** (async, discounted), not on-demand loops |
| Hundreds of tiny classification calls | **Batch items into one request** (JSON array) or batch inference |
| Spiky on-demand traffic hits TPM limits with no commitment wanted | **Cross-Region inference profile** + quota increase |
| Steady high volume needing guaranteed throughput / custom model | **Provisioned Throughput** (or Reserved tier) |
| Extraction must be consistent and parseable | **Temperature 0–0.2**, schema/structured output, stop sequence, tight maxTokens |
| Marketing team wants varied creative options | **Higher temperature/topP**, generate several candidates |
| How to set top_k in the Converse API | **`additionalModelRequestFields`** (not `inferenceConfig`) |
| Output JSON is cut off mid-object | `stopReason = max_tokens` → raise **maxTokens** or shorten the output |
| Prove a new temperature or prompt is better | **A/B test** (Prompt Management versions / AppConfig / SageMaker variants) on quality, latency and cost |
| SageMaker LLM endpoint scales too slowly on bursts | Target tracking on **`…ConcurrentRequestsPerModelHighResolution`** + scheduled scaling + warm minimum |
| Scale on GPU utilisation | **Custom CloudWatch metric** + target tracking/step scaling |
| Traffic peaks every weekday 9–11 am | **Scheduled scaling** to pre-warm before the peak |
| Find which hop causes P99 latency in a RAG app | **X-Ray / ADOT tracing** across API GW → Lambda → retrieval → Bedrock |
| Lambda calling Bedrock pays repeated connection set-up | Create the client **outside the handler**, keep-alive, connection pool |
| Generation > 29 s behind API Gateway | **Async pattern** (job ID + SQS/Step Functions) or streaming |

## Flashcards

<details>
<summary><b>Q:</b> What are the three components of LLM response latency, and which one does streaming improve?</summary>

TTFT, output tokens/sec and end-to-end latency. Streaming improves *perceived* latency, because users see output after TTFT. It doesn't shorten total generation time.

*D4 · Task 4.2 · Latency anatomy*
</details>

<details>
<summary><b>Q:</b> How do you enable latency-optimised inference on a Bedrock request, and what's the default?</summary>

`"performanceConfig": {"latency": "optimized"}`. The default is `"standard"`. It's a preview feature for selected models (Nova Pro, Claude 3.5 Haiku, Llama 3.1 70B/405B) via cross-Region inference profiles.

*D4 · Task 4.2 · Latency-optimised inference*
</details>

<details>
<summary><b>Q:</b> A latency-optimised request arrives after the latency-optimised quota is used up. What happens?</summary>

Bedrock tries to serve it at standard latency and charges standard rates. The served latency mode is visible in the response and in CloudTrail.

*D4 · Task 4.2 · Latency-optimised inference*
</details>

<details>
<summary><b>Q:</b> Which single change usually reduces end-to-end latency most for a verbose assistant?</summary>

Reducing output tokens (tight `maxTokens`, concise or structured output instructions). Decode time scales with output length.

*D4 · Task 4.2 · Output length*
</details>

<details>
<summary><b>Q:</b> When is pre-computation the right latency strategy?</summary>

When queries are predictable and not user-specific (FAQs, product descriptions, nightly summaries). Generate offline with batch inference, store the results, and serve from a cache in milliseconds.

*D4 · Task 4.2 · Pre-computation*
</details>

<details>
<summary><b>Q:</b> Step Functions Parallel vs Map: which one for summarising a variable list of documents?</summary>

Map: one iteration per item, with `MaxConcurrency` to respect Bedrock quotas (Distributed Map for very large sets). Parallel is for a fixed set of different branches.

*D4 · Task 4.2 · Parallel requests*
</details>

<details>
<summary><b>Q:</b> Which HNSW parameter is the main query-time dial between recall and latency?</summary>

`ef_search`. Higher gives better recall and slower queries. `m` and `ef_construction` are build-time settings and need a re-index to change.

*D4 · Task 4.2 · HNSW tuning*
</details>

<details>
<summary><b>Q:</b> Roughly how much k-NN graph memory does a 32 GiB OpenSearch Service data node allow by default?</summary>

About 8 GiB. Half the RAM goes to the JVM heap, and k-NN can use up to 50% of the remaining half. Watch `KNNGraphMemoryUsage`.

*D4 · Task 4.2 · OpenSearch memory*
</details>

<details>
<summary><b>Q:</b> Name four ways to reduce the memory footprint of a large vector index.</summary>

fp16 scalar quantisation, byte (int8) vectors, binary vectors, product quantisation, disk-based mode with compression + rescoring, and lower embedding dimensions (e.g. Titan V2 at 512/256). Any four of these.

*D4 · Task 4.2 · Quantisation*
</details>

<details>
<summary><b>Q:</b> Why can a k-NN query with `post_filter` return fewer than k results, and what's the fix?</summary>

The filter runs after the top-k vectors are selected, so selective filters remove most of them. Use efficient filtering (a `filter` clause inside the `knn` query).

*D4 · Task 4.2 · Filtered search*
</details>

<details>
<summary><b>Q:</b> Why does OpenSearch hybrid search need a normalization processor?</summary>

BM25 and vector similarity scores are on different scales. The search pipeline normalises them (`min_max` or `l2`) and combines them (weighted arithmetic/geometric/harmonic mean). RRF is an alternative.

*D4 · Task 4.2 · Hybrid scoring*
</details>

<details>
<summary><b>Q:</b> How does reranking improve both relevance *and* speed?</summary>

You retrieve a wide set cheaply, the cross-encoder reranker picks the best few, and the FM receives fewer, better chunks. That means fewer input tokens, so lower TTFT and cost.

*D4 · Task 4.2 · Reranking*
</details>

<details>
<summary><b>Q:</b> A multi-turn RAG chatbot retrieves irrelevant chunks for follow-up questions. What should you add?</summary>

Query rewriting: use a small fast model and the conversation history to turn the follow-up into a standalone query before retrieval.

*D4 · Task 4.2 · Query preprocessing*
</details>

<details>
<summary><b>Q:</b> Why can an oversized `maxTokens` reduce throughput even when responses are short?</summary>

Bedrock reserves quota tokens at request start based on input + `maxTokens`, then adjusts. Oversized caps temporarily consume TPM and cause throttling. Verify the current burndown rules in docs.

*D4 · Task 4.2 · Quota burndown*
</details>

<details>
<summary><b>Q:</b> Interactive users and a nightly 1 M-record enrichment job share a model quota. How do you separate them?</summary>

Move the nightly job to Bedrock batch inference (async, discounted, off the interactive path). Keep interactive traffic on-demand (with CRIS / Priority tier if needed).

*D4 · Task 4.2 · Batch inference*
</details>

<details>
<summary><b>Q:</b> How do you protect a shared Bedrock quota from traffic spikes without dropping requests?</summary>

Buffer requests in SQS, consume with bounded concurrency (Lambda SQS max concurrency / workers), retry throttles with backoff + jitter, and add per-tenant limits.

*D4 · Task 4.2 · Concurrency management*
</details>

<details>
<summary><b>Q:</b> Which parameters go in `inferenceConfig` and where does `top_k` go in the Converse API?</summary>

`inferenceConfig`: `maxTokens`, `temperature`, `topP`, `stopSequences`. Model-specific settings such as `top_k` go in `additionalModelRequestFields`.

*D4 · Task 4.2 · Converse parameters*
</details>

<details>
<summary><b>Q:</b> Top P = 0.7 with candidates horses 0.7, zebras 0.2, unicorns 0.1. Which tokens are considered?</summary>

Only "horses". It alone covers the top 70% of cumulative probability. At Top P = 0.9, horses and zebras are considered.

*D4 · Task 4.2 · Top P*
</details>

<details>
<summary><b>Q:</b> Recommended temperature range for extraction/classification vs creative copy?</summary>

Extraction/classification: about 0–0.2 (deterministic, parseable). Creative: about 0.7–1.0 with a higher topP. Tune temperature or topP, not both.

*D4 · Task 4.2 · Parameter selection*
</details>

<details>
<summary><b>Q:</b> How do you validate that a new prompt version or temperature setting is actually better?</summary>

Run an offline evaluation on a fixed dataset, then an online A/B test (Prompt Management versions, AppConfig percentage rollout, or SageMaker variant weights). Compare quality, latency percentiles, tokens/cost and user feedback, changing one variable at a time.

*D4 · Task 4.2 · A/B testing*
</details>

<details>
<summary><b>Q:</b> How do you estimate the TPM needed for a new GenAI feature?</summary>

TPM ≈ peak RPM × (average input + average output tokens), sized on P95 lengths with 20–30% headroom. Then pick on-demand + CRIS, a quota increase, PT or the Reserved tier.

*D4 · Task 4.2 · Capacity planning*
</details>

<details>
<summary><b>Q:</b> Which predefined SageMaker metric suits autoscaling an LLM endpoint better than invocations per instance, and why?</summary>

`SageMakerVariantConcurrentRequestsPerModelHighResolution` (or `…InferenceComponentConcurrentRequestsPerCopyHighResolution`). LLM request duration varies hugely, so concurrency reflects load better, and high resolution gives faster scale-out.

*D4 · Task 4.2 · Autoscaling*
</details>

<details>
<summary><b>Q:</b> Traffic to a self-hosted LLM spikes every weekday at 9 am and cold starts take minutes. What scaling set-up helps?</summary>

Scheduled scaling to pre-warm capacity before 9 am, plus target tracking on concurrency and a warm minimum instance or copy count.

*D4 · Task 4.2 · Scheduled scaling*
</details>

<details>
<summary><b>Q:</b> A Lambda function calls Bedrock on every invocation and shows high connection overhead. What do you change?</summary>

Create the boto3 client outside the handler (reused across warm invocations), enable TCP keep-alive, and size the connection pool. Run in the same Region, through a VPC endpoint.

*D4 · Task 4.2 · Service communication*
</details>

<details>
<summary><b>Q:</b> How do you find which component causes P99 latency in a RAG pipeline?</summary>

Use distributed tracing (X-Ray / ADOT) across API Gateway, Lambda, retrieval, Bedrock and guardrails. Log the prompt version, model ID and token counts per call to profile prompt and completion patterns.

*D4 · Task 4.2 · API profiling*
</details>

---
[← Back to Domain 4](README.md)
