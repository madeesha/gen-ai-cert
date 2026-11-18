[← Back to Domain 4](README.md)

# 💰 Task 4.1 · Implement cost optimization and resource efficiency strategies

> **Domain 4 · Operational Efficiency and Optimization for GenAI Applications (12%)**
> - Skill 4.1.1: Token efficiency systems (token estimation and tracking, context window optimisation, response size controls, prompt compression, context pruning, response limiting)
> - Skill 4.1.2: Cost-effective model selection frameworks (cost-capability trade-offs, tiered FM usage by query complexity, cost vs quality balancing, price-to-performance measurement, efficient inference patterns)
> - Skill 4.1.3: High-performance FM systems that maximise utilisation and throughput (batching, capacity planning, utilisation monitoring, auto scaling, Provisioned Throughput optimisation)
> - Skill 4.1.4: Intelligent caching systems that avoid unnecessary FM calls (semantic caching, result fingerprinting, edge caching, deterministic request hashing, prompt caching)

## Key points
- **Mental model: cost = tokens × price × number of calls.** The four skills are the four levers: send fewer tokens (4.1.1), use a cheaper model (4.1.2), pay a cheaper way and keep capacity busy (4.1.3), and skip the call entirely with a cache (4.1.4).
- **Output tokens cost several times more than input tokens**, so capping output (`maxTokens`, stop sequences, short formats) is usually the first and biggest win. Then prune repeated input: history, RAG chunks, tool catalogues (4.1.1).
- **Estimate before, measure after, attribute per team:** `CountTokens` before the call, the `usage` block and CloudWatch after it, and **application inference profiles + cost allocation tags** for chargeback (4.1.1).
- **Pick the cheapest model that meets the quality bar**, compared on **cost per successful task**, not price per token (4.1.2).
- **Mixed-difficulty traffic → tier it:** static routing, classifier router, cascade, **Intelligent Prompt Routing** (managed, one model family) or a **distilled** small model (4.1.2).
- **Purchase option follows traffic shape:** spiky → on-demand · deferrable → Flex or batch · latency-critical → Priority · steady and high → Reserved or PT. **Idle PT is pure waste** (4.1.3).
- **Offline volume → batch inference** (about half the on-demand price). It doesn't work with provisioned models or prompt caching (4.1.3).
- **Self-hosted models:** keep GPUs busy (continuous batching, inference components) and scale to zero when idle. Scaling mechanics are in [4.2.5](02-task-4.2-application-performance.md#skill-425--efficient-resource-allocation-for-fm-workloads) (4.1.3).
- **Choose the cache by what repeats:** identical public GETs → edge · identical requests → exact-match hash · paraphrases → semantic cache · a long stable prefix → **Bedrock prompt caching** (4.1.4).
- **Caches fail quietly:** a prompt-cache prefix below the minimum size caches nothing, and an app cache without tenant and version in the key leaks or serves stale answers (4.1.4).

## Notes

### Skill 4.1.1 — Token efficiency systems

> *Develop token efficiency systems to reduce FM costs while maintaining effectiveness (for example, by using token estimation and tracking, context window optimization, response size controls, prompt compression, context pruning, response limiting).*

#### Why tokens are the cost unit
- Bedrock on-demand pricing is charged **per 1,000 input tokens** plus **per 1,000 output tokens**, and the rates differ by model. Output tokens are usually **3–5× more expensive** than input tokens for text LLMs (check the pricing page for each model).
- **Tokens/minute quotas** (TPM) and **requests/minute quotas** (RPM) limit throughput. Fewer tokens per request therefore cuts cost **and** fits more requests within your quota.
- Images, documents and tool definitions also count as input tokens. Tool schemas are sent on **every** turn, so a large tool catalogue costs you on every call.
- Reasoning or extended thinking tokens are billed as **output** tokens. Turn extended thinking on only for tasks that need it.

#### Estimating tokens before the call — `CountTokens`
- The `CountTokens` API (bedrock-runtime) returns the **exact** token count the model would bill for that input. It's **free** and model-specific, because each model has its own tokenizer. It needs the IAM permission `bedrock:CountTokens`.
- The `input` field takes either a **Converse** body (`messages` + `system`) or an **InvokeModel** body as a JSON string.
- Some Claude models that are offered only through cross-Region inference don't support CountTokens on `bedrock-runtime`. For those, use Anthropic's `count_tokens` path on the **bedrock-mantle** endpoint instead. Check the model card.

```python
import boto3
brt = boto3.client("bedrock-runtime")

resp = brt.count_tokens(
    modelId="anthropic.claude-3-5-haiku-20241022-v1:0",
    input={"converse": {
        "system":   [{"text": "You are a concise support agent."}],
        "messages": [{"role": "user", "content": [{"text": user_question}]}],
    }},
)
if resp["inputTokens"] > BUDGET:      # e.g. reject, summarise or trim before calling
    ...
```

- Rough rule of thumb when you can't call the API: **1 token ≈ 4 English characters ≈ 0.75 words**. It varies by language and is worse for code and non-Latin scripts.

#### Tracking tokens after the call

| Where | What you get | Use it for |
| --- | --- | --- |
| Converse / ConverseStream response `usage` | `inputTokens`, `outputTokens`, `totalTokens`, `cacheReadInputTokens`, `cacheWriteInputTokens` | Per-request metering in app code. Emit as a custom metric with dimensions such as tenant and feature |
| InvokeModel response headers | `X-Amzn-Bedrock-Input-Token-Count`, `X-Amzn-Bedrock-Output-Token-Count` | Same as above when you call InvokeModel |
| CloudWatch `AWS/Bedrock` metrics | `InputTokenCount`, `OutputTokenCount`, `Invocations`, `InvocationLatency`, throttles. Dimension: `ModelId` | Fleet-wide dashboards and alarms |
| Model invocation logging (CloudWatch Logs / S3) | Full request and response plus token counts and identity | Find which prompts are expensive and who is calling (Logs Insights) |
| Application inference profiles + cost allocation tags | Cost per tagged profile in Cost Explorer / CUR | Chargeback and showback per team, app or tenant |
| AWS Budgets / Cost Anomaly Detection | Alerts on spend thresholds and unusual spend | Guard against runaway agents or token bursts (see [Task 4.3 · Skill 4.3.2](03-task-4.3-monitoring-observability.md#skill-432--comprehensive-genai-monitoring-and-kpis)) |

- ⚠️ When **prompt caching** is in use, `inputTokens` counts **only non-cached** tokens. Total input = `inputTokens + cacheReadInputTokens + cacheWriteInputTokens`.
- **Application inference profile pattern:** create one profile per app or team that wraps a model or a system cross-Region profile, tag it (`CostCenter=claims`), activate the tag in Billing, and have each app call **its own profile ARN** as the `modelId`.

```python
bedrock = boto3.client("bedrock")
bedrock.create_inference_profile(
    inferenceProfileName="claims-summariser",
    modelSource={"copyFrom": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-lite-v1:0"},
    tags=[{"key": "CostCenter", "value": "claims"}],
)
```

#### Context window optimisation
A bigger context window costs more. **Every token in the prompt is paid for on every call.** Techniques, roughly from cheapest to most involved:

| Technique | How | Saves | Risk |
| --- | --- | --- | --- |
| **Tighten the system prompt** | Remove repetition, filler and examples you don't need. Move long policy text to KB retrieval | Fixed overhead on every call | Behaviour regressions, so run a regression suite ([Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md)) |
| **Fewer few-shot examples** | 1–2 targeted examples, or pick examples dynamically by similarity | Hundreds to thousands of tokens | Format drift |
| **Retrieve fewer, better chunks** | Lower `numberOfResults`, add a **reranker** and keep the top 3, use metadata filters | Big for RAG | Lower recall, so check with retrieval evaluation |
| **Smaller or hierarchical chunks** | Match small child chunks. Return the parent only when needed | Medium | Context fragmentation |
| **Sliding-window history** | Keep only the last N turns | Linear growth → constant | Loses older facts |
| 🎯 **PQ19** **Summarise history** | Every K turns, compress older turns into a running summary with a cheap model | Large for long chats | Summary drops details. Store key facts (entities, IDs) in structured state |
| **Structured state instead of transcript** | Keep slots and entities in DynamoDB and inject a compact JSON state | Large | More engineering |
| 🎯 **PQ19** **Prompt compression** | Use a small model (or LLMLingua-style token pruning) to rewrite long context before sending it | 2–5× on verbose docs | Lossy, and the extra call adds latency and cost |
| **Trim tool catalogue** | Expose only the tools relevant to the current intent, with short descriptions | Per-turn overhead | The agent may miss a tool |
| **Strip boilerplate in preprocessing** | Remove HTML, headers/footers, signatures and duplicates before embedding or prompting ([Task 1.3](../domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md)) | Medium | Little |

- **Context pruning** means deliberately removing low-value content (old turns, low-score chunks, superseded tool results) **before** each call. Agents should truncate or summarise large tool outputs before putting them back into the loop.
- **"Lost in the middle":** long contexts also **reduce quality**, because models attend less to content in the middle. Shorter context is cheaper **and** often better.
- Bedrock Agents and Strands also manage the context window. For example, Strands has conversation managers such as sliding-window or summarising managers (see [Task 2.1 · Skill 2.1.1](../domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md#skill-211--autonomous-systems-with-memory--state-management)).

> [!IMPORTANT]
> **🎯 Practice question 19 — long, repetitive chat histories push up tokens and cost; no major architecture change**
>
> - Add a **pre-processing Lambda function** before the Bedrock call that:
>   - **prunes** repeated or low-value messages,
>   - **summarises** older history with a **small, cheap model**,
>   - sets **max input and output token limits** on the request (`maxTokens`).
> - This reduces input tokens, keeps answer quality (the key facts survive in the summary) and fits the existing API Gateway → Lambda architecture.
> - ❌ **Index all conversations in OpenSearch:** retrieved content adds **more** input tokens, and more complexity.
> - ❌ **A bigger-context model:** accepts more tokens and doesn't reduce them.
> - ❌ **Send the full history every time:** the opposite of what's needed.

Trimming the input is half the job. The other half is bounding what the model writes back.

#### Response size controls / response limiting
- **`inferenceConfig.maxTokens`** sets a hard cap on output tokens. If the output hits the cap, the response has `stopReason = "max_tokens"` and is truncated. Handle this case: ask the model to continue, or reduce scope.
- **`stopSequences`** end generation early at a delimiter (`"</answer>"`, `"\n\nHuman:"`).
- **Prompt-level length instructions** ("max 50 words", "return only JSON") are soft limits. Combine them with `maxTokens`.
- **Structured outputs / JSON schema** (tool-use forcing or native structured output) stop the model writing prose around the answer.
- **Classification-style tasks:** ask for a label only (`maxTokens` = 5–10).
- **Extended thinking budget:** set `budget_tokens` (Claude) low or turn reasoning off for simple tasks.

```python
resp = brt.converse(
    modelId=PROFILE_ARN,
    system=[{"text": "Answer in at most 3 bullet points."}],
    messages=msgs,
    inferenceConfig={"maxTokens": 300, "temperature": 0.2, "stopSequences": ["</answer>"]},
)
if resp["stopReason"] == "max_tokens":
    log_truncation(resp["usage"])   # track truncation rate as a quality signal
```

#### Worked example — where the money goes
A support chatbot handles 1M requests/month. Each request has a 2,000-token system prompt + 1,500 tokens of RAG chunks + 500 tokens of history + 100 tokens of question = **4,100 input** tokens, and **400 output** tokens.

Illustrative prices: $X per 1K input tokens and $4X per 1K output tokens.
- Input: 4.1 × X × 1M = 4.1M·X. Output: 0.4 × 4X × 1M = 1.6M·X. **Total ≈ 5.7M·X.**
- Cache the 2,000-token system prompt (assume a 90% hit rate and cache reads at about 10% of the input price; verify in docs): 2.0 → about 0.38 effective → saves about 1.6M·X.
- Rerank and keep 3 chunks instead of 6: 1,500 → 750 tokens → saves 0.75M·X.
- Cap output at 250 tokens: 1.6 → 1.0M·X → saves 0.6M·X.
- **Result ≈ 2.75M·X, a cut of about 50%**, without changing the model. Moving routine traffic to a cheaper model saves more on top (Skill 4.1.2).

---

### Skill 4.1.2 — Cost-effective model selection frameworks

> *Create cost-effective model selection frameworks (for example, by using cost-capability tradeoff evaluation, tiered FM usage based on query complexity, inference cost balancing against response quality, price-to-performance ratio measurement, efficient inference patterns).*

#### Cost-capability trade-off evaluation
1. **Define the quality bar** for the task: accuracy, faithfulness, format compliance and latency SLO, measured on a **golden dataset** that represents real traffic.
2. **Shortlist candidates** of different sizes: e.g. Nova Micro / Lite / Pro, Claude Haiku / Sonnet, Llama 8B / 70B. FM selection criteria are covered in [Task 1.2 · Skill 1.2.1](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md#skill-121--assess-and-choose-fms).
3. **Evaluate each candidate** with **Bedrock Model Evaluation** (automatic, LLM-as-a-judge, or human). For each one record the quality score, p50/p95 latency, and average input and output tokens.
4. **Compute cost per successful task**, not just cost per token:

```
cost_per_task = (avg_in_tokens × in_price + avg_out_tokens × out_price) / success_rate
price_performance = quality_score / cost_per_task
```

5. **Pick the cheapest model that meets the quality bar**, not the "best" model. Repeat when new models are released, since prices fall and smaller models improve.

- ⚠️ A cheaper model that needs **longer prompts, more retries or more few-shot examples** may cost more overall. Always compare **cost per completed task**.
- A model that writes long answers costs more even when its per-token price is the same. Compare **average output tokens**.

#### Tiered FM usage based on query complexity

| Pattern | How it works | When to use | Trade-off |
| --- | --- | --- | --- |
| **Static routing by use case** | Each feature or endpoint is hard-wired (in config or AppConfig) to a model tier | Clearly separate workloads (classification vs long-form writing) | No per-query adaptivity |
| **Classifier router** | A cheap model or Comprehend custom classifier labels complexity or intent, and a rule picks the model | Mixed traffic with recognisable query types | Extra hop. Misroutes reduce quality |
| 🎯 **PQ57** **Cascade / escalation** | Try the small model first and escalate if confidence is low, validation fails or the user asks again | Most queries are easy | Hard queries pay twice and take longer |
| **Bedrock Intelligent Prompt Routing** | Managed router between **two models in the same family**. Predicts the quality of each and sends the request to the cheaper one unless the quality difference exceeds `responseQualityDifference` | You want routing with no code, within one family | Limited model pairs, English-focused. Can't learn from your own feedback |
| 🎯 **PQ1** **Distilled model** | Bedrock **Model Distillation**: a large teacher generates training data and a small student is fine-tuned on it | High-volume narrow task where the small model isn't quite good enough | Training cost. Bedrock custom models may need PT or on-demand custom model deployment (verify in docs) |
| **Fine-tuned small model** | SFT or LoRA on your own data (SageMaker or Bedrock) | Domain-specific format or style at high volume | Data and lifecycle cost |

```python
# Cascade sketch
small = call(model=NOVA_LITE, prompt=p, max_tokens=300)
if not passes_schema(small) or self_rated_confidence(small) < 0.7:
    return call(model=SONNET, prompt=p, max_tokens=600)   # escalate
return small
```

- Cascading as a pattern is covered in [Task 2.2 · Skill 2.2.3](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-223--optimise-fm-deployment-for-performance-and-resource-use). Routing mechanics (Step Functions Choice states, API Gateway transforms, metric-based routing) are in [Task 2.4 · Skill 2.4.4](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md#skill-244--intelligent-model-routing-systems). Dynamic model switching with AppConfig is in [Task 1.2 · Skill 1.2.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md#skill-122--flexible-architecture-for-dynamic-model-selection--provider-switching).

> [!IMPORTANT]
> **🎯 Practice question 57 — moderation cascade: small model first, escalate confidence <0.65 asynchronously, spiky load, decoupled**
>
> - **SQS queue 1 → Fargate workers → small model.** High-confidence results come back in **near real time**.
> - If **confidence < 0.65**, put the request on **SQS queue 2**, where workers call the **large model asynchronously**.
> - SQS **buffers spikes** and **decouples** the parts. The large model runs **only when needed**, which controls cost.
> - ❌ **Running both models in parallel every time:** pays for the large model on every request.
> - ❌ **Synchronous API Gateway chain:** not decoupled, and low-confidence requests aren't asynchronous.
> - ❌ **EC2 + keyword heuristics:** infrastructure to manage, and routing by guesswork instead of confidence.

#### Inference cost balancing against response quality
- Ways to reduce cost and their quality effect:

| Lever | Cost effect | Quality effect |
| --- | --- | --- |
| Smaller model | ↓↓ | ↓ on complex reasoning |
| Fewer RAG chunks | ↓ | ↓ recall if overdone |
| Shorter output | ↓ | ↓ completeness |
| Prompt caching | ↓ (repeated prefix) | None |
| Batch inference | ↓ about 50% | None (but no real time) |
| Flex tier | ↓ | None (higher latency, may be deprioritised) |
| Extended thinking off | ↓ | ↓ on multi-step reasoning |

- **Use A/B tests in production** to confirm a cheaper configuration doesn't hurt business KPIs such as CSAT, deflection rate or escalation rate. Prompt Management variants and canaries are covered in [Domain 5](../README.md).

#### Price-to-performance ratio measurement
- Track, per model or configuration: **$ per 1K requests**, **$ per successful task**, **quality score**, **p95 latency** and **tokens per request**. Put them together on a CloudWatch dashboard.
- **Latency-to-quality ratio** and **token efficiency** (quality per 1K tokens) are listed as evaluation dimensions in Task 5.1.
- Get costs from CUR or Cost Explorer filtered by application inference profile tag, and quality from evaluation jobs or user feedback. Join them in Athena or QuickSight.

#### Efficient inference patterns (cost lens)
- **Batch inference** for anything that doesn't need an answer within seconds: nightly summarisation, back-filling embeddings, classifying documents, generating evaluation datasets.
- **Asynchronous processing** with SQS and workers that run at a controlled rate, which smooths spikes and avoids paying for over-provisioned capacity.
- **Pre-computation**: generate answers to predictable queries offline (FAQ answers, product descriptions) and serve them from a cache or database. Latency side: [Task 4.2 · Skill 4.2.1](02-task-4.2-application-performance.md#skill-421--responsive-ai-systems-latency-vs-cost-trade-offs).
- **Use non-LLM tools where they're enough**: regex, Comprehend, Textract or a SQL query are often cheaper and more deterministic than an FM call.
- **Embeddings**: choose smaller dimensions (Titan V2 at 512 or 256), embed in batches, and only re-embed **changed** documents.

---

### Skill 4.1.3 — High-performance FM systems: utilisation and throughput

> *Develop high-performance FM systems to maximize resource utilization and throughput for GenAI workloads (for example, by using batching strategies, capacity planning, utilization monitoring, auto-scaling configurations, provisioned throughput optimization).*

Idle capacity and one-call-per-item processing are the two big sources of waste. This skill is about paying for capacity in the cheapest way that fits the traffic, and then keeping it busy.

#### Choosing the inference purchase option (cost view)
How each option works is in [Task 2.2 · Skill 2.2.1](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-221--deploy-fms-based-on-application-needs-and-performance-requirements). This is the cost comparison:

| Option | Pay for | Best when | Watch out |
| --- | --- | --- | --- |
| **On-demand (Standard tier)** | Tokens used | Unpredictable or spiky traffic, POCs | Throttling at quota limits |
| **Flex tier** | Lower token price (verify in docs) | Latency-tolerant work that can be deprioritised (evaluations, enrichment, agent background steps) | Higher latency, may be throttled first |
| **Priority tier** | Premium token price | Latency-critical, customer-facing traffic | Costs more than Standard |
| **Reserved tier** | Committed TPM capacity (1 or 3 months), with overflow to Standard | Steady, predictable high volume that needs guaranteed capacity | Pay for reserved capacity whether you use it or not |
| **Provisioned Throughput (model units)** | Hourly per model unit. No commitment, 1-month or 6-month terms (verify in docs) | Custom or fine-tuned models that need it, or steady high utilisation | **Hourly cost even when idle.** Size from measured TPM. Delete unused PT |
| **Batch inference** | About 50% of on-demand (verify in docs) | Offline jobs | Hours to complete |
| **Cross-Region (global) inference** | Same as or lower than in-Region (global ≈ 10% saving reported; verify in docs) | More throughput and resilience | Data residency, so use **geographic** profiles if needed |

**When does Provisioned Throughput or Reserved pay off?**
Break-even utilisation = `hourly commitment cost / (on-demand cost per hour at full MU capacity)`. If the measured average utilisation over 24×7 is **below** break-even, on-demand is cheaper. Typical exam answers:
- "Steady 24×7 high volume, need guaranteed capacity" → Reserved or PT.
- "Business-hours only, idle at night" → on-demand (or PT without commitment, deleted after hours, if the model is custom).
- "Using a fine-tuned Bedrock model" → PT or custom model on-demand deployment, depending on support (verify in docs).

**Capacity planning (cost view):** size any commitment from **measured** peak TPM, worked out with the method in [Skill 4.2.5](02-task-4.2-application-performance.md#skill-425--efficient-resource-allocation-for-fm-workloads). If peak demand is over quota, try the options that need no commitment first: **cross-Region inference**, a **quota increase**, and flattening the peak with **SQS + backoff**. Buy Reserved or PT only for load that is steady.

#### Batching strategies
Once the purchase option is chosen, batching raises how much work each call or instance does.

| Level | Mechanism | Notes |
| --- | --- | --- |
| **Bedrock batch inference** | `CreateModelInvocationJob` with JSONL in S3 (`recordId` + `modelInput`, InvokeModel or Converse format) and output to S3 | About **50% cheaper** than on-demand for supported models (verify in docs). **Not for provisioned models.** Converse-type jobs can't use tool calling (use the InvokeModel type). Job quotas, structured-output support and EventBridge job events → [Skill 2.2.1](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-221--deploy-fms-based-on-application-needs-and-performance-requirements) |
| **OpenAI-compatible Batch API** | Bedrock also accepts batches in OpenAI Batch API format for OpenAI models | Same idea |
| **SageMaker Batch Transform** | Offline inference over S3 data on transient instances | Pay only while the job runs |
| **SageMaker async endpoint** | Queue up to 1 GB payloads, scales to 0 | Intermittent heavy jobs |
| **Request batching in the server** | Continuous or dynamic batching in LMI, vLLM or TGI containers | Raises GPU utilisation for self-hosted models ([Task 2.2 · Skill 2.2.2](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-222--address-the-llm-specific-challenges-of-deployment)) |
| **Client-side micro-batching** | Group many small items (e.g. classify 20 short texts) into **one prompt** with structured output | Fewer calls, so less repeated system-prompt overhead. Risk: one bad item breaks the batch |
| **Embedding batching** | Send many texts per call where the model supports it, or use a batch job | Titan V2 accepts one text per InvokeModel call, so use batch jobs for large backfills (verify in docs) |

```jsonl
{"recordId": "r-0001", "modelInput": {"messages": [{"role":"user","content":[{"text":"Summarise: ..."}]}], "inferenceConfig": {"maxTokens": 200}}}
{"recordId": "r-0002", "modelInput": {"messages": [{"role":"user","content":[{"text":"Summarise: ..."}]}], "inferenceConfig": {"maxTokens": 200}}}
```

#### Utilisation monitoring and Provisioned Throughput right-sizing
- **Bedrock:** use the `Invocations`, `InputTokenCount`/`OutputTokenCount`, `InvocationThrottles` and latency metrics. Compare actual TPM with quota or with the PT capacity. Use the CloudWatch `ResolvedServiceTier` dimension to see which tier actually served the request.
- **SageMaker:** use `CPUUtilization`, `GPUUtilization`, `GPUMemoryUtilization`, `InvocationsPerInstance`, `ModelLatency` and `ConcurrentRequestsPerModel`. Low GPU utilisation means right-size the instance (e.g. move to a smaller g6 or Inferentia2), pack models with **inference components**, or consolidate onto **multi-model endpoints**.
- **Cost views:** Cost Explorer grouped by usage type and tags, CUR in Athena, **Cost Anomaly Detection** for spikes, and **AWS Budgets** alerts per app tag.

**Provisioned Throughput checklist:**
- Size PT from **measured** peak TPM plus headroom, not guesses. Start on on-demand and measure first.
- Use **no-commitment PT** for testing a custom model and delete it afterwards.
- **Under-used:** if usage stays under about 50% of the provisioned model units, you're over-provisioned. Unused MUs are pure waste, so reduce MUs at renewal or move bursty traffic back to on-demand.
- **Saturated:** add MUs, or combine PT with **on-demand or CRIS overflow** in app logic (or rely on the Reserved tier's built-in overflow to Standard).
- PT is per model **version**, so plan the migration when you upgrade a model.

#### Auto scaling (cost view, self-hosted models)
The cost goal is to match capacity to load so you don't pay for idle GPUs. Scaling metrics, policy types, cooldowns and a worked example are in [Skill 4.2.5](02-task-4.2-application-performance.md#skill-425--efficient-resource-allocation-for-fm-workloads), and LLM serving specifics in [Task 2.2 · Skill 2.2.2](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-222--address-the-llm-specific-challenges-of-deployment).
- **Scale to zero** for dev/test and idle models: **inference components** (each model copy scales independently and can scale to zero; verify in docs) or **async endpoints**.
- **Serverless Inference** for low, sporadic traffic, but it's CPU only with up to 6 GB of memory, so small models only.
- **Scheduled scaling** to shrink capacity outside predictable peaks.
- **Spot** works for **training and batch** (Managed Spot Training), not for real-time endpoints.

---

### Skill 4.1.4 — Intelligent caching systems

> *Create intelligent caching systems to reduce costs and improve response times by avoiding unnecessary FM invocations (for example, by using semantic caching, result fingerprinting, edge caching, deterministic request hashing, prompt caching).*

#### The caching layers

```
Client ─► CloudFront / API GW cache ─► App ─► [exact-match cache] ─► [semantic cache] ─► Bedrock (prompt cache) ─► model
          (identical public GETs)        (hash key → DynamoDB/ElastiCache) (embedding k-NN)   (reused prefix tokens)
```

| Layer | Avoids | Hit criterion | Store | Best for |
| --- | --- | --- | --- | --- |
| **Edge cache** | The whole request | Identical URL, query string or headers | CloudFront, API Gateway REST stage cache | Public, non-personalised, GET-style responses (e.g. a product description generated once) |
| **Exact-match / deterministic hash** | The FM call | `SHA-256(normalised prompt + model + params + prompt version)` | ElastiCache (Redis/Valkey), DynamoDB with TTL | Repeated identical requests, idempotent retries, batch dedup |
| **Result fingerprinting** | Re-processing unchanged inputs | Hash of the **input content** (document bytes, record) | DynamoDB, S3 object metadata | Don't re-summarise or re-embed documents that haven't changed |
| **Semantic cache** | The FM call for paraphrased questions | Cosine similarity ≥ threshold between embeddings | ElastiCache/MemoryDB vector search, OpenSearch k-NN | FAQ-like traffic with many paraphrases |
| **Bedrock prompt caching** | Re-processing the **prefix** inside the model | Exact prefix match up to a cache checkpoint | Managed by Bedrock | Long, stable system prompts, documents, tool definitions, multi-turn chats, agent loops |

The sections below work from the layer inside the model outwards: prompt caching, then the semantic and exact-match caches in your app, then the edge, and finally how to keep all of them fresh.

#### Bedrock prompt caching — details
- **Two types:**
  - **Implicit:** automatic and best effort, no request changes. Used by Nova and by OpenAI models.
  - **Explicit:** you add **cache checkpoints**. Claude models need explicit caching; requests without cache controls aren't cached.
- 🎯 **PQ3** **Checkpoint placement:** `cachePoint` blocks go in `tools`, `system` and/or `messages` (Converse), or `cache_control: {"type":"ephemeral"}` on content blocks (InvokeModel for Claude).
  - Sections are processed and chained in the order **`tools → system → messages`**. Changing an earlier section invalidates the cache for later sections.
  - Put **stable content first** and **variable content last**.
- 🎯 **PQ3** **Minimum tokens:** the prefix before a checkpoint must reach the model minimum, currently **512–4,096 tokens** depending on the model. If it's shorter, the call still succeeds but **nothing is cached**. The minimum counts **cumulatively** across tools, system and messages.
- **Maximum checkpoints:** **4** for Claude and Nova models.
- **Claude automatic caching:** one top-level `cache_control` field moves the checkpoint forward as the conversation grows. It uses one of the 4 slots and is available on newer Claude models.
- **TTL:** the default is **5 minutes** and **resets on every hit**. Some Claude models also support **`"ttl": "1h"`**. Sending `ttl` to a model that only supports 5 minutes gives a `ValidationException`. In a mixed request, longer-TTL entries must come before shorter ones.
- **Billing:**
  - **Cache write** tokens are billed at the cache-write rate, which can be **higher** than normal input (e.g. 1.25× for some models).
  - **Cache read** tokens are billed at a **discounted** rate (often around 10% of the input price; verify in docs).
  - All other tokens are billed at the normal rate.
  - Cache hits on Claude are **not counted against rate limits**, which also helps throughput.
- **Response fields:** `cacheReadInputTokens`, `cacheWriteInputTokens` and `cacheDetails` (TTL). Total input = `inputTokens + cacheRead + cacheWrite`.
- **No hit is guaranteed:**
  - The cache is held on the server that processed the request.
  - Cross-Region inference can cause **more cache writes**.
  - Requests sent before the first request has started its response won't hit the cache.
- **Not supported with batch inference.** It works with Converse, ConverseStream, InvokeModel, InvokeModelWithResponseStream, CRIS and Prompt Management (an option on the prompt).
- **Also lowers latency:** time to first token drops for long prefixes. Cost and latency both improve.

```python
resp = brt.converse(
    modelId="us.anthropic.claude-sonnet-4-6",          # example ID; check the model card for caching support
    system=[{"text": LONG_POLICY_MANUAL},               # ≥ min tokens, stable
            {"cachePoint": {"type": "default"}}],
    messages=[{"role": "user", "content": [{"text": question}]}],
)
u = resp["usage"]
print(u.get("cacheReadInputTokens"), u.get("cacheWriteInputTokens"), u["inputTokens"])
```

**When does prompt caching pay off?** When the **same prefix** is reused **more than about twice within the TTL**. The first call pays the write premium and later calls get the read discount. For a one-off prompt, a premium write rate makes caching **cost more**.

> [!IMPORTANT]
> **🎯 Practice question 3 — uploaded docs/images, accurate but slow, cost-effective fix**
>
> - Users upload a document or image and then ask **several questions about it**. The same large context is reprocessed on every Converse call → **enable Bedrock prompt caching**.
> - Put **`cachePoint` in `system` and `messages`** (after the uploaded document or image). Use the **lowest minimum token count** the model allows so the content actually gets cached. This lowers both **latency and input-token cost**.
> - ❌ **Caching the prompt in ElastiCache:** the model still has to **reprocess the full context**. It only adds infrastructure.
> - ❌ **Provisioned Throughput:** adds capacity, not lower per-request latency, and costs a lot per hour.
> - ❌ **Chunking + OpenSearch vector index:** that's RAG ingestion and doesn't help with per-request uploads.

#### Semantic caching — design 🎯 PQ23
1. Embed the incoming query with a cheap embedding model (Titan Text Embeddings V2 at 256–512 dims).
2. Run a k-NN lookup (k=1) in the cache index, **filtered by tenant, locale and prompt version**.
3. If similarity ≥ **threshold** (e.g. 0.90–0.95 cosine), return the cached answer and record a hit metric.
4. Otherwise call the FM, store `{embedding, query, answer, metadata, ttl}`, and return the answer.

| Store | Notes |
| --- | --- |
| **ElastiCache (Valkey/Redis OSS) vector search** | In-memory, microsecond to millisecond lookups. Native TTL. Common choice for semantic caching (verify engine and version support in docs) |
| **MemoryDB vector search** | Durable in-memory store with vector search. Use when the cache must survive restarts or be the system of record |
| **OpenSearch (Serverless or managed) k-NN** | When you already run OpenSearch. Slower than in-memory, but has rich filtering |
| **DynamoDB** | Exact-match only (no vector search). Pair it with one of the above |

- **Choosing the threshold:**
  - Too low causes **false hits**: wrong answers to questions that only look similar ("cancel order" vs "cancel subscription").
  - Too high gives few hits.
  - Tune it on labelled paraphrase pairs, and track hit rate together with user feedback on cached answers.
- **What not to cache:**
  - personalised answers (unless the user or tenant ID is in the key or filter)
  - time-sensitive data (prices, stock levels), unless the TTL is short
  - high-temperature creative output
  - responses blocked by guardrails
- **Security:** treat the cache as sensitive data. Encrypt it, isolate tenants, and apply the same guardrails or PII filtering **before** storing (see [Task 3.2 · Skill 3.2.2](../domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md#skill-322--privacy-preserving-systems-pii-detection-bedrock-privacy-guardrails-retention)).

> [!IMPORTANT]
> **🎯 Practice question 23 — 40% of queries are rephrased versions of the same questions: fewer calls, consistent answers, low latency**
>
> - **Semantic cache:** embed each query with **Bedrock (Titan Embeddings)** and store **vector + response** in **MemoryDB for Valkey** (in-memory vector search). Use a **vector RANGE query** (similarity within a radius or threshold) to find an equivalent earlier question and return its answer.
> - This means **fewer FM calls**, **the same answer for equivalent questions**, and **millisecond latency**.
> - ❌ **DAX / DynamoDB key lookups, `LIKE`, stemming, GSI on normalised text:** all are **exact or lexical matches**, so rephrased questions miss the cache.
> - ❌ **OpenSearch k-NN over raw text pairs:** k-NN needs **vector embeddings**, not text.

#### Deterministic request hashing & result fingerprinting
- **Normalise before hashing:** lowercase, trim whitespace, sort JSON keys, and **include** the model ID, inference parameters, prompt template version, KB version and tenant. Otherwise you'll serve stale or cross-tenant answers.
- Key = `sha256(json.dumps(canonical, sort_keys=True))`. Store it in ElastiCache (fast, with TTL) or DynamoDB (serverless, with TTL attribute).
- **Determinism:** only cache results from low-temperature or deterministic configurations. Otherwise identical requests are *expected* to give different results.
- **Fingerprinting for pipelines:** store `content_hash` per document (e.g. in DynamoDB or S3 metadata). On re-ingestion, skip summarising or embedding when the hash hasn't changed. Bedrock KB **sync** already only processes added, changed or deleted files (incremental).
- **Idempotency:** the same hash lets a retried request after a timeout reuse the earlier result instead of paying twice.

```python
import hashlib, json
def cache_key(tenant, model, params, prompt_ver, user_input):
    canon = {"t": tenant, "m": model, "p": params, "v": prompt_ver,
             "q": " ".join(user_input.lower().split())}
    return hashlib.sha256(json.dumps(canon, sort_keys=True).encode()).hexdigest()
```

#### Edge caching
- **CloudFront** in front of API Gateway or a Lambda function URL suits **GET** endpoints with cacheable, non-personalised generated content (e.g. generated product copy or translated static pages). Control it with `Cache-Control` headers and cache policies (cache key = path + selected query strings).
- **API Gateway REST API stage caching** uses a TTL (default 300 s, maximum 3600 s) and per-method cache keys. It isn't available on HTTP APIs. **Response streaming mode disables caching** (see [Task 2.4 · Skill 2.4.2](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md#skill-242--real-time-ai-interaction-systems--pq24)).
- Streaming chat responses and POST requests with personalised context are usually **not** edge-cacheable. Use the app-level caches instead.

#### Cache invalidation & freshness
| Trigger | Action |
| --- | --- |
| KB data source re-synced or document updated | Bump the `kb_version` in cache keys, or purge entries tagged with that source |
| Prompt template version changed | Include the prompt version in keys so old entries miss naturally |
| Model upgraded | Include the model ID in keys. Re-run the cached answers' quality checks |
| Time-sensitive content | Short TTL (minutes). Exclude from the cache |
| Negative feedback on a cached answer | Evict that entry and lower its confidence or threshold |

- **Metrics to emit:** `CacheHitRate` (exact and semantic separately), `TokensSaved`, `CostSaved` and `CacheLatency`. Prompt-cache effectiveness is `cacheReadInputTokens / total input tokens`.

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| "Estimate token cost before sending the request" / "check prompt fits the context window" | **CountTokens API** (free, exact per model) |
| "Show Bedrock spend per team / application / tenant" | **Application inference profiles + cost allocation tags** → Cost Explorer / CUR |
| Answers too long and expensive | `maxTokens`, `stopSequences`, length instructions, structured output |
| `stopReason = max_tokens` and answers cut off | Raise `maxTokens` or ask for shorter output / continuation; monitor the truncation rate |
| Chat cost grows with conversation length | **Sliding window + summarisation of history**, structured state in DynamoDB |
| RAG prompts are huge | Fewer chunks + **reranker**, metadata filters, smaller or hierarchical chunks |
| Agent with 50 tools, high per-turn input tokens | Trim or select tools dynamically, short tool descriptions, cache the tool definitions |
| Sudden spend spike from a runaway agent loop | **Cost Anomaly Detection + Budgets alerts**, max-iteration limits, per-tenant token quotas |
| Most queries are simple, a few are complex; cut cost without hurting quality | **Tiered models / cascade / classifier router** or **Bedrock Intelligent Prompt Routing** |
| 🎯 **PQ1** Need a small, cheap model that matches a large model on one narrow task | **Model distillation** (teacher → student) |
| Compare models fairly on cost | **Cost per successful task** (tokens × price / success rate), not price per token |
| Spiky or unpredictable traffic, minimise cost | **On-demand** (+ CRIS for capacity) |
| Background enrichment that can tolerate delay, real-time API | **Flex service tier** |
| Steady 24×7 high volume, need guaranteed capacity | **Reserved tier / Provisioned Throughput** |
| PT model units mostly idle | Over-provisioned → reduce MUs / move to on-demand / delete no-commit PT |
| Using a custom fine-tuned model on Bedrock | **Provisioned Throughput** (or custom model on-demand deployment where supported) |
| Millions of documents to classify overnight, cheapest option | **Bedrock batch inference** (≈50% cheaper, JSONL in S3) |
| Batch job with Converse format fails on records with `toolConfig` | Converse batch doesn't support tool calling → use the **InvokeModel** invocation type |
| Self-hosted LLM GPU utilisation low | Right-size, **inference components** / multi-model packing, continuous batching |
| SageMaker LLM endpoint scales too late (long requests) | Scale on **concurrency** metrics, lower target, faster model loading, scheduled scaling |
| Dev/test LLM endpoint idle most of the day | Scale to zero (inference components / async endpoint) or serverless for small models |
| Long, identical system prompt or document sent on every request; reduce cost and TTFT | **Bedrock prompt caching** (cachePoint after the stable prefix) |
| Prompt caching enabled but `cacheReadInputTokens` always 0 | Prefix below the model's **minimum tokens**, dynamic content placed **before** the checkpoint, or a gap between calls longer than the TTL (5 min) |
| Identical requests repeated; need deterministic reuse | **Exact-match cache** with a SHA-256 request hash (ElastiCache/DynamoDB TTL) |
| Users ask the same question in many different phrasings | **Semantic cache**: embeddings + vector lookup in ElastiCache/MemoryDB with a similarity threshold |
| Avoid re-summarising or re-embedding documents that haven't changed | **Result fingerprinting** (content hash) / incremental KB sync |
| Public, non-personalised generated content served to many users | **CloudFront** / API Gateway cache |

## Flashcards

<details>
<summary><b>Q:</b> What does the CountTokens API cost, and why use it rather than a characters ÷ 4 estimate?</summary>

It's free. It returns the exact count from the model's own tokenizer, which is the number you'd be billed for. That makes it reliable for budget checks and context-window checks.

*D4 · Task 4.1 · Token estimation*
</details>

<details>
<summary><b>Q:</b> With prompt caching on, how do you compute the total input tokens of a Converse request?</summary>

`inputTokens + cacheReadInputTokens + cacheWriteInputTokens`. When caching is in use, `inputTokens` counts only the non-cached tokens.

*D4 · Task 4.1 · Token tracking*
</details>

<details>
<summary><b>Q:</b> How do you attribute Bedrock costs to individual teams that share the same model?</summary>

Create an application inference profile for each team, tag it with a cost allocation tag (and activate the tag), and have each team call its own profile ARN as the modelId.

*D4 · Task 4.1 · Cost attribution*
</details>

<details>
<summary><b>Q:</b> Why is output-length control often the single biggest token saving?</summary>

Output tokens are usually priced several times higher than input tokens and also drive latency. Capping `maxTokens`, using stop sequences and asking for structured or short output cuts both.

*D4 · Task 4.1 · Response limiting*
</details>

<details>
<summary><b>Q:</b> Name three ways to stop chat context costs growing linearly with conversation length.</summary>

A sliding window of the last N turns, periodic summarisation of older turns with a cheap model, and keeping structured state (entities and slots in DynamoDB) instead of the full transcript.

*D4 · Task 4.1 · Context pruning*
</details>

<details>
<summary><b>Q:</b> How does a reranker reduce cost in RAG?</summary>

It reorders the retrieved chunks by relevance so you can pass only the top few (e.g. 3 instead of 10) to the FM. That cuts input tokens and often improves answer quality.

*D4 · Task 4.1 · Context optimisation*
</details>

<details>
<summary><b>Q:</b> How do you detect a runaway agent loop burning tokens before the bill arrives?</summary>

Per-tenant or per-app token metrics with CloudWatch alarms, AWS Cost Anomaly Detection and Budgets alerts, plus hard limits on agent iterations and maxTokens.

*D4 · Task 4.1 · Cost guardrails*
</details>

<details>
<summary><b>Q:</b> How does Bedrock Intelligent Prompt Routing decide between models?</summary>

It predicts response quality for two models in the same family and sends the request to the cheaper model, unless the predicted quality gap exceeds the configured response-quality difference. Otherwise it uses the stronger model.

*D4 · Task 4.1 · Tiered models*
</details>

<details>
<summary><b>Q:</b> What metric should you use to compare models on cost?</summary>

Cost per successful task: (average input tokens × input price + average output tokens × output price) ÷ success rate. Then look at quality against that cost (price-performance).

*D4 · Task 4.1 · Price-performance*
</details>

<details>
<summary><b>Q:</b> When does Provisioned Throughput make financial sense?</summary>

When steady, measured utilisation is high enough that the hourly model-unit cost is less than the equivalent on-demand token spend. It's also needed for custom models that require PT. Idle PT is pure waste.

*D4 · Task 4.1 · Provisioned Throughput*
</details>

<details>
<summary><b>Q:</b> A latency-tolerant enrichment workload must stay on the real-time API, but cost matters. Which Bedrock option fits?</summary>

The Flex service tier. It has a lower price in exchange for lower priority and higher latency. Batch fits only if the workload can run fully offline.

*D4 · Task 4.1 · Service tiers*
</details>

<details>
<summary><b>Q:</b> Which Bedrock option is about 50% cheaper than on-demand for large offline workloads, and what are two limitations?</summary>

Batch inference (JSONL in and out of S3). It isn't supported for provisioned models, and Converse-type batch jobs don't support tool calling (use the InvokeModel type instead).

*D4 · Task 4.1 · Batch inference*
</details>

<details>
<summary><b>Q:</b> Which scaling metric works better than invocations-per-instance for SageMaker LLM endpoints, and why?</summary>

Concurrency-based metrics (concurrent requests per model or copy). LLM requests run for a long and variable time, so invocation counts lag behind the real load.

*D4 · Task 4.1 · Auto scaling*
</details>

<details>
<summary><b>Q:</b> GPU utilisation on a self-hosted multi-model setup is 15%. What are two fixes?</summary>

Pack several models onto shared instances with SageMaker inference components (or a multi-model endpoint), and right-size to a smaller or Inferentia instance. Enable continuous batching in the serving container.

*D4 · Task 4.1 · Utilisation*
</details>

<details>
<summary><b>Q:</b> In what order are prompt cache checkpoints evaluated in Converse, and why does it matter?</summary>

`tools → system → messages`. The sections are chained, so a change in an earlier section (e.g. tools) invalidates the cache for every later section. Put stable content first.

*D4 · Task 4.1 · Prompt caching*
</details>

<details>
<summary><b>Q:</b> You add a cachePoint after a 300-token system prompt, but nothing is ever cached. Why?</summary>

The prefix is below the model's minimum tokens per checkpoint (512–4,096 depending on the model). The request still succeeds but isn't cached.

*D4 · Task 4.1 · Prompt caching*
</details>

<details>
<summary><b>Q:</b> What is the default prompt cache TTL, and what extends it?</summary>

5 minutes. It resets on every cache hit. Some Claude models also support a 1-hour TTL (`"ttl": "1h"`).

*D4 · Task 4.1 · Prompt caching*
</details>

<details>
<summary><b>Q:</b> When can prompt caching increase cost?</summary>

When a prefix is written but rarely or never reused within the TTL. Cache-write tokens can be billed above the normal input rate, so one-off prompts pay a premium and get no reads back.

*D4 · Task 4.1 · Prompt caching economics*
</details>

<details>
<summary><b>Q:</b> Can you use prompt caching with Bedrock batch inference?</summary>

No. Prompt caching works only with on-demand inference (Converse, InvokeModel, their streaming variants, and CRIS). It isn't supported with the batch inference API.

*D4 · Task 4.1 · Prompt caching limits*
</details>

<details>
<summary><b>Q:</b> Semantic cache vs exact-match cache: what's the key difference and risk?</summary>

An exact-match cache hits only on identical normalised requests (a hash key), so it's deterministic and safe. A semantic cache hits on similar meaning (an embedding plus a similarity threshold). It catches paraphrases, but it can return the wrong answer if the threshold is too low.

*D4 · Task 4.1 · Semantic caching*
</details>

<details>
<summary><b>Q:</b> Which AWS stores fit a low-latency semantic cache?</summary>

ElastiCache (Valkey/Redis OSS) with vector search, or MemoryDB vector search if it needs to be durable. Use OpenSearch k-NN if you already run it. DynamoDB works for exact-match caching only.

*D4 · Task 4.1 · Semantic caching*
</details>

<details>
<summary><b>Q:</b> What must be part of a GenAI cache key to avoid stale or leaked answers?</summary>

Tenant or user scope, model ID, inference parameters, prompt template version and KB/data version, together with the normalised query.

*D4 · Task 4.1 · Cache key design*
</details>

<details>
<summary><b>Q:</b> What is result fingerprinting used for in GenAI pipelines?</summary>

Hash each input's content, and skip re-summarising or re-embedding an item when its hash hasn't changed. This saves FM calls on re-ingestion.

*D4 · Task 4.1 · Fingerprinting*
</details>

<details>
<summary><b>Q:</b> Why can't you usually edge-cache a streaming chat endpoint?</summary>

The responses are personalised POST responses streamed in real time. API Gateway response-streaming mode also disables caching. Use app-level exact or semantic caches instead.

*D4 · Task 4.1 · Edge caching*
</details>

---
[← Back to Domain 4](README.md)
