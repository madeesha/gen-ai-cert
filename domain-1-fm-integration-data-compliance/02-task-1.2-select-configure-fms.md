[← Back to Domain 1](README.md)

# 🤖 Task 1.2 · Select and configure FMs

> **Exam domain:** D1 · Foundation Model Integration, Data Management, and Compliance (31%)
>
> **Skills covered**
> - **1.2.1** Assess and choose FMs for business use cases and technical requirements
> - **1.2.2** Flexible architecture patterns for dynamic model selection and provider switching without code changes
> - **1.2.3** Resilient AI systems that keep running during service disruptions
> - **1.2.4** FM customisation deployment and lifecycle management

## Key points
- **Choosing an FM is about trade-offs.** Weigh modality, quality on *your* task, context window, latency (TTFT and tokens/sec), cost per token, languages, customisation support, Regional availability, compliance, and licence/provider. **Benchmarks only shortlist models. Evaluate on your own data** (Bedrock Model Evaluation: automatic, human, LLM-as-a-judge).
- **Pick the smallest model that meets the quality bar.** If request complexity varies, route requests: use **Bedrock Intelligent Prompt Routing** (within one model family) or a custom router in Lambda (a classifier sends each request to a small or large model).
- 🎯 **PQ34** **Swapping models without code changes:** use the **Converse API** (one request/response shape for every model). Keep the model ID out of code: put it in **AWS AppConfig** (feature flags or free-form config, validators, gradual rollout with automatic rollback) or Parameter Store. Put the Lambda behind **API Gateway** as a stable façade. **InvokeModel** uses a model-specific body, so switching providers means changing code.
- **For multiple providers**, use an **adapter/strategy pattern** in Lambda: one internal contract, with one adapter per provider (Bedrock, SageMaker endpoint, or an external API with its key in Secrets Manager).
- **Throttling or limited Regional capacity:** use **Cross-Region Inference (CRIS)**. Call an **inference profile** ID (`us.`, `eu.`, `apac.` geographic, or `global.`) instead of the base model ID. No code changes, and no extra routing charge (you pay the source-Region price). The IAM policy must allow the model in **every destination Region**, and SCPs must not block those Regions.
- **Which errors to retry:** retry `ThrottlingException` (429), `ServiceUnavailableException` (503), `ModelTimeoutException` (408), `InternalServerException` (500) and `ModelNotReadyException` with **exponential backoff and jitter**. **Never retry** `ValidationException` (400) or `AccessDeniedException` (403); fix the request or permissions instead.
- **Circuit breaker:** **Step Functions** reads the circuit state from **DynamoDB**. CLOSED: call the primary model. OPEN: go straight to the fallback. HALF-OPEN: send a probe request. Use `Retry` (with `BackoffRate`, `MaxAttempts`, `JitterStrategy: FULL`) plus `Catch`, which leads to the fallback state.
- **Graceful degradation ladder:** primary model → same model in another Region (CRIS or multi-Region deployment) → smaller or other-provider model → cached or semantic-cache answer → static or templated response → queue the request for asynchronous processing ("we'll email you").
- **Customisation path:** prompt engineering → RAG → fine-tuning (PEFT/**LoRA**) → continued pre-training → train from scratch. Cost, data needs and effort rise at each step. **LoRA** trains small low-rank matrices; adapters are only MBs and can be swapped onto one shared base model.
- **SageMaker AI deployment:** use LMI containers (vLLM/TensorRT-LLM) on real-time endpoints. **Inference components** pack many models or LoRA adapters onto one endpoint, each scaling independently. Use **deployment guardrails** (blue/green: all-at-once, canary or linear) with **CloudWatch-alarm auto-rollback**. Use **shadow tests** to compare models on live traffic with zero user impact.
- **SageMaker Model Registry:** a model package group holds versioned model packages. Each has an approval status (`PendingManualApproval` → `Approved` / `Rejected`). A status change emits an **EventBridge** event that triggers the CI/CD deployment pipeline. Roll back by re-approving or deploying the previous version.
- **Bedrock model lifecycle:** Active → **Legacy** (≥6 months' notice, no new Provisioned Throughput) → **EOL** (requests fail, no automatic migration). Bedrock custom models run on **Provisioned Throughput** (some support on-demand; verify in docs). **Custom Model Import** brings open-weight fine-tunes (for example Llama or Mistral architectures) into Bedrock as a serverless deployment.

> [!IMPORTANT]
> **🎯 Practice question 34 — switch models with no redeploy, roll out to 20% gradually, auto-rollback if errors >1%, 3 Regions**
>
> - **AWS AppConfig feature flags** hold the model selection. The **AppConfig Agent Lambda extension** fetches and caches the config at runtime, so **there are no code changes or redeploys**.
> - A **linear deployment strategy** (for example a 20% step) rolls the new model out gradually.
> - Use **validators** plus a **CloudWatch alarm on the error rate (>1%) as the deployment monitor** → AppConfig **rolls back automatically**.
> - ❌ **DynamoDB global tables + custom lookup code:** a code change, with no built-in gradual rollout or rollback.
> - ❌ **Parameter Store + custom selection logic:** a code change, with no rollout or rollback.
> - ❌ **Lambda versions + weighted aliases + CodeDeploy:** every model switch is a **code deployment**.

## Notes

### Skill 1.2.1 — Assess and choose FMs

> *Assess and choose FMs to ensure optimal alignment with specific business use cases and technical requirements (for example, by using performance benchmarks, capability analysis, limitation evaluation).*

#### Selection criteria (the full checklist)

| Dimension | Questions to ask | Typical exam signal |
| --- | --- | --- |
| **Task / modality** | Text, chat, code, image generation, image or video understanding, embeddings, speech? | "Analyse product photos" → multimodal (e.g. Claude, Nova Pro/Lite) · "Generate images" → Nova Canvas / Stable Diffusion / Titan Image |
| **Quality on your task** | Accuracy, reasoning, instruction-following, tool use, faithfulness | "Complex multi-step reasoning" → larger, reasoning-capable model |
| **Context window** | How many tokens of input (documents + history + RAG chunks)? | "Summarise 300-page contracts in one pass" → long-context model |
| **Latency** | TTFT and output tokens/sec, synchronous or async | "Real-time chat / autocomplete / mobile" → small, fast model + streaming + latency-optimised inference |
| **Throughput / scale** | RPM/TPM quotas, burst patterns | "Spiky traffic, throttling" → CRIS; "sustained high volume" → Provisioned Throughput |
| **Cost** | Price per 1K input/output tokens; output usually costs more | "High-volume, simple classification" → smallest adequate model, batch inference |
| **Languages** | Multilingual support and quality, embedding language support | "Japanese + English support bot" → check model's supported languages |
| **Customisation** | Fine-tuning, distillation, continued pre-training, Custom Model Import | "Must fine-tune on proprietary data in Bedrock" → pick a model that supports Bedrock customisation |
| **Regional availability** | Is the model in the required Region? | "Model not available in eu-west-x" → CRIS (EU profile keeps data in EU) or another Region |
| **Compliance / data residency** | Data must stay in a geography; certifications (HIPAA eligible, etc.) | "Data must not leave the EU" → **EU geographic** inference profile, *not* global |
| **Responsible AI / safety** | Built-in safety, provider's AI service card, bias | "Need documented intended use & limits" → AWS AI Service Cards / model cards |
| **Licence / openness** | Proprietary API vs open weights (self-host, modify) | "Need full control over weights / run in own VPC on GPUs" → open-weight on SageMaker AI |
| **Features** | Tool use, structured output (JSON), system prompts, prompt caching, guardrail support, streaming | "Agent must call tools" → model with tool use in Converse API |

#### Performance benchmarks — use them, but know the limits
- Public benchmarks (e.g. MMLU for knowledge, HumanEval for code, GSM8K for maths, HELM for holistic evaluation, MT-Bench/Arena-style for chat) are useful for building a **shortlist**.
- ⚠️ Limitations: benchmarks may be contaminated (in the training data), may not represent your domain, can be gamed, and say nothing about your latency or cost.
- **Correct approach (exam):** shortlist from benchmarks, then run an evaluation on **your own representative dataset** with task-specific metrics, then compare quality, latency and cost together.

#### Capability analysis & limitation evaluation
- **Capabilities to test:** instruction following, output-format compliance (JSON), tool calling, reasoning depth, long-context recall ("needle in a haystack"), multilingual quality, refusal behaviour.
- **Limitations to evaluate:**
  - Knowledge cutoff → use RAG for current data
  - Hallucination rate → grounding, contextual grounding checks (Guardrails)
  - Context window and degradation in the middle of long contexts ("lost in the middle")
  - Max output tokens
  - Token cost at scale
  - Bias/toxicity
  - Rate limits / quotas per Region
  - Determinism (temperature 0 still isn't perfectly deterministic)

#### Amazon Bedrock Model Evaluation (key tool for 1.2.1)

| Evaluation type | How it works | Use when |
| --- | --- | --- |
| **Automatic (programmatic)** | Built-in or custom datasets; metrics such as accuracy, robustness, toxicity for tasks like summarisation, Q&A, classification, open-ended generation | Fast, cheap, repeatable comparison |
| **Human** | Your own work team (or AWS-managed team); rate on custom criteria (helpfulness, brand voice, style) | Subjective quality, tone, brand alignment |
| **LLM-as-a-judge** | A judge model scores outputs on correctness, completeness, faithfulness, helpfulness, harmfulness, etc. | Near-human-quality evaluation at scale, lower cost than humans |
| **RAG / Knowledge Base evaluation** | Evaluates retrieval (context relevance and coverage) and retrieve-and-generate (correctness, faithfulness, citation quality) | Comparing KB configs or generator models for RAG |

- Results are written to **S3**. Evaluation jobs need an IAM service role with access to the dataset bucket and models.
- You can also evaluate **bring-your-own inference responses** (outputs from models outside Bedrock). This lets you compare Bedrock models against external or SageMaker models on the same scale (verify the current feature set in the docs).
- SageMaker alternative: **SageMaker Clarify FM evaluations** (based on the open-source `fmeval` library) for models on SageMaker / JumpStart.

#### Right-sizing & routing
- **Principle:** the smallest/cheapest model that meets the quality bar. A small model with a good prompt or RAG often beats a big model with a generic prompt.
- **Bedrock Intelligent Prompt Routing:** a single prompt-router ARN routes each request between models **in the same family** (e.g. a smaller and a larger Claude, Llama or Nova model), based on predicted response quality. It optimises cost while keeping quality. Use the default routers or configure your own (pick the models and a response-quality-difference threshold).
- **Custom routing:** for different families or providers, or rules based on business logic (tenant tier, task type), build a router in Lambda. For example, Comprehend or a small LLM classifies the request and a routing table in AppConfig maps it to a model.
- **Model cascade:** try the cheap model first. If a confidence or validation check fails (e.g. invalid JSON, low self-reported confidence), escalate to the larger model.

#### Decision shortcuts
- Need **embeddings** → Titan Text Embeddings V2 / Cohere Embed / Nova multimodal embeddings (see Task 1.5).
- Need **document/image understanding** → a multimodal model, or **Bedrock Data Automation** for structured extraction.
- Need **lowest cost for offline bulk** → **batch inference** (roughly 50% cheaper than on-demand for supported models; verify in docs).
- Need **agentic tool use** → a model that supports tool use in Converse.
- Need **open weights in your own account** → SageMaker JumpStart / Custom Model Import.

---

### Skill 1.2.2 — Flexible architecture for dynamic model selection & provider switching

> *Create flexible architecture patterns to enable dynamic model selection and provider switching without requiring code modifications (for example, by using AWS Lambda, Amazon API Gateway, AWS AppConfig).*

#### Why
- Models change fast: new versions, deprecations (Legacy/EOL), price drops and outages.
- Hard-coding a model ID means a redeploy for every change. The goal is to **externalise the choice** so ops or product teams can switch models through configuration, with safe rollout and rollback.

#### Building block 1 — Converse API (model-agnostic contract)
- `Converse` / `ConverseStream` provide one message format (`messages`, `system`, `inferenceConfig`, `toolConfig`, `guardrailConfig`) across Bedrock models.
- Switching models is then just changing `modelId`, which can be a base model ID, an **inference profile** ID/ARN, a **provisioned model ARN**, a **prompt router ARN**, or a **Prompt Management prompt ARN** (verify which IDs each API supports).
- Model-specific parameters go into `additionalModelRequestFields` (keep these in config too).
- `InvokeModel` uses a **provider-specific JSON body** (e.g. the Anthropic Messages format vs the Titan format), so switching providers there requires code changes.

```python
import boto3, json
brt = boto3.client("bedrock-runtime")
appcfg = get_config()                    # e.g. from AppConfig Lambda extension (localhost:2772)
resp = brt.converse(
    modelId=appcfg["chat"]["modelId"],   # swap in config, not code
    system=[{"text": appcfg["chat"]["systemPrompt"]}],
    messages=[{"role": "user", "content": [{"text": user_input}]}],
    inferenceConfig={"maxTokens": appcfg["chat"]["maxTokens"],
                     "temperature": appcfg["chat"]["temperature"]},
)
```

#### Building block 2 — AWS AppConfig (dynamic configuration)
- **Feature flags** and **free-form configuration** (JSON/YAML/text) stored in an AppConfig hosted store, S3, Parameter Store, SSM documents, etc.
- **Validators:** JSON Schema or a Lambda validator. These stop a typo like `"modelId": "antropic..."` from being deployed.
- **Deployment strategies:** linear or exponential rollout over time, with **bake time**. Link **CloudWatch alarms** for **automatic rollback** if error rate or latency rises after the model switch.
- **AWS AppConfig Agent Lambda extension:** caches the config locally and polls in the background, so it isn't fetched on every invocation (lower latency and cost).
- **Multi-variant feature flags:** serve different values to different segments (e.g. 10% of users get the new model, or premium tenants get the larger model). Use this for **A/B testing models** (verify the current capability in the docs).

Example configuration:
```json
{
  "routes": {
    "summarise": { "provider": "bedrock", "modelId": "us.anthropic.claude-...", "maxTokens": 800, "temperature": 0.2 },
    "classify":  { "provider": "bedrock", "modelId": "amazon.nova-micro-v1:0", "maxTokens": 50,  "temperature": 0 },
    "legacy-ft": { "provider": "sagemaker", "endpointName": "llama-ft-prod" }
  },
  "fallbackOrder": ["summarise", "classify"]
}
```

#### Building block 3 — API Gateway as a stable façade
- Clients call `POST /v1/generate` and never see the model or provider.
- **Stage variables** can point at different Lambda aliases or backends (dev/test/prod). **Lambda aliases with weighted routing** support canary releases of new router code.
- API Gateway also handles **auth** (Cognito/IAM/Lambda authoriser), **throttling and usage plans** (per-tenant quotas), request validation and caching.

#### Building block 4 — Lambda router / adapter (strategy pattern)
- One internal interface `generate(request) → response`, with an **adapter per provider**:
  - Bedrock adapter: Converse
  - SageMaker adapter: `InvokeEndpoint` with a container-specific payload
  - External provider adapter: HTTPS call, API key from **Secrets Manager**
- The router reads AppConfig and picks a route by task type, tenant, cost tier or A/B bucket. It then normalises the response (text, token usage, stop reason) and emits metrics (`model`, `latency`, `tokens`) to **CloudWatch** (EMF) so models can be compared.
- Alternatives: **Bedrock Prompt Management** (model + inference params stored with a prompt version, changed without code); **LiteLLM**/LangChain abstractions; **Bedrock AgentCore Gateway** for tools (not model routing).

#### Reference architecture
```
Client → API Gateway (auth, throttling, usage plans)
       → Lambda "model router" ──reads──> AppConfig (routes, flags, fallback order)
            ├─ Bedrock Converse (base model | inference profile | prompt router)
            ├─ SageMaker AI endpoint (fine-tuned / open-weight)
            └─ External provider (Secrets Manager key)
       → CloudWatch metrics per model (latency, tokens, errors) → alarms → AppConfig rollback
```

#### Gotchas
- Different models have **different tokenisers, context limits, and prompt sensitivities**. A config-only swap still needs **regression testing** of prompts (see Task 1.6.4).
- Keep **guardrails and inference parameters per model** in config. A prompt tuned for one model may degrade on another.
- **IAM:** the Lambda role needs `bedrock:InvokeModel` (also used for Converse) for all model and inference-profile ARNs it may switch to. Scope permissions to an allow-list rather than `*` for governance.

---

### Skill 1.2.3 — Resilient AI systems

> *Design resilient AI systems to ensure continuous operation during service disruptions (for example, by using AWS Step Functions circuit breaker patterns, Amazon Bedrock Cross-Region Inference for models that have limited regional availability, cross-Region model deployment, graceful degradation strategies).*

#### Failure modes in GenAI systems

| Failure | Signal | Mitigation |
| --- | --- | --- |
| Throttling / quota exhaustion | `ThrottlingException` (429) | Backoff + jitter, CRIS, quota increase, Provisioned Throughput, request queueing (SQS) |
| Model capacity / Regional outage | `ServiceUnavailableException` (503), elevated latency | CRIS, multi-Region deployment, fallback model |
| Model timeout | `ModelTimeoutException` (408) | Shorter outputs (`maxTokens`), streaming, async pattern, retry |
| Model not ready (e.g. imported/custom model cold) | `ModelNotReadyException` | Retry with backoff (SDK retries automatically) |
| Bad request | `ValidationException` (400) | **Do not retry**: fix input (token limit, format) |
| Permissions / model access | `AccessDeniedException` (403) | **Do not retry**: fix IAM / model access / SCP |
| Model deprecated | Legacy/EOL notices, failures after EOL | Lifecycle management (1.2.4), config-driven switch (1.2.2) |
| Downstream dependency (KB, vector store, tool API) down | Errors / timeouts in retrieval | Circuit breaker, answer without RAG with a disclaimer, cached results |

#### Retries done right
- AWS SDK **retry modes:** `legacy`, `standard` (default in newer SDKs, 3 attempts), `adaptive` (client-side rate limiting, good for throttling-heavy workloads).
  ```python
  from botocore.config import Config
  cfg = Config(retries={"max_attempts": 8, "mode": "adaptive"}, read_timeout=120)
  brt = boto3.client("bedrock-runtime", config=cfg)
  ```
- Use **exponential backoff with jitter** so retries don't all arrive at once (thundering herd). Cap total retry time below the user's timeout budget (e.g. the API Gateway integration timeout, 29 s default for REST).
- Make operations **idempotent** where side effects exist (agent tool calls).

#### CRIS (Cross-Region Inference)
- **What:** call an **inference profile** instead of a model ID. Bedrock routes the request from your **source Region** to an optimal Region within the profile's Region set.
- **Types:**
  - 🎯 **PQ28** **Geographic** profiles (`us.`, `eu.`, `apac.`, …): stay within that geography. Use these for data-residency requirements.
  - **Global** profile (`global.`): routes across commercial Regions worldwide for maximum throughput and availability (verify pricing differences in docs).
  - **Application inference profiles** (user-created): wrap a model or system profile for **cost allocation tags** and usage tracking per app or tenant.
- **Benefits:** higher throughput, burst handling, access to models with **limited Regional availability**, and no extra routing cost (priced at the source Region).
- **Requirements & gotchas:**
  - The IAM policy must allow the inference profile **and** the underlying foundation model in **all destination Regions**.
  - **SCPs** that deny Regions can break CRIS (use condition exceptions for Bedrock inference profiles; verify the current guidance).
  - **CloudTrail** and invocation logs are recorded in the **source Region**. Data at rest isn't stored in destination Regions, but in-transit processing happens there, so check compliance.
  - Model access must be enabled where required (verify; newer accounts get automatic access for most serverless models).
  - CRIS improves availability **within** a set of Regions. It is **not** a full DR plan for your own app stack (Lambda, KB, vector store) in the source Region.

> [!IMPORTANT]
> **🎯 Practice question 28 — 1,000 TPS, <500 ms, HA during disruptions, EU data must stay in EU Regions**
>
> - **US applications** → **Provisioned Throughput** (`InvokeModel`) for **guaranteed, consistent capacity** at 1,000 TPS and <500 ms.
> - **EU applications** → an **EU geographic inference profile** (`eu.`), so data is **processed only in European Regions**.
> - **Auto scaling** of provisioned capacity on utilisation, plus **EventBridge + Lambda cross-Region failover** for high availability.
> - ❌ **Global inference profile:** it can route EU data **anywhere in the world**, which breaks residency.
> - ❌ **On-demand only, with IAM per Region:** throttling risk, and IAM isn't residency enforcement.
> - ❌ **Per-Region Provisioned Throughput behind an ALB with no cross-Region profile:** no automated cross-Region failover.

#### Cross-Region model deployment (your own stack)
- For **SageMaker AI endpoints** or a full app (KB, vector store, Lambda), deploy to **two or more Regions**. Use active-active or active-passive with **Route 53** health checks/failover (or latency-based routing), and replicate data (S3 CRR, DynamoDB global tables, OpenSearch cross-cluster replication, Aurora Global Database).
- **Bedrock Knowledge Bases** are Regional. Multi-Region RAG means a KB and vector store per Region, each fed from replicated S3 sources.
- **Provisioned Throughput** is per Region. Buying PT in a second Region is the guaranteed-capacity DR option, and it is expensive.

#### Circuit breaker pattern with Step Functions
- **Why:** after repeated failures, stop calling the failing dependency for a while. This avoids wasting retries, adding latency and overloading the dependency. Fail fast and use the fallback.
- **States:** **CLOSED** (normal) → failures exceed a threshold → **OPEN** (skip the primary, use the fallback until the timeout expires) → **HALF-OPEN** (allow a probe request; success closes the circuit, failure reopens it).
- 🎯 **PQ16** **AWS implementation:** a Step Functions workflow and a **DynamoDB** table holding `{serviceName, state, failureCount, expiresAt (TTL)}`.
  1. `GetItem` on the circuit state (direct SDK integration, no Lambda needed).
  2. `Choice`: OPEN and not expired → **fallback branch**.
  3. Otherwise **invoke Bedrock** (Step Functions has an optimised `bedrock:invokeModel` integration).
  4. `Retry` transient errors, then `Catch` → increment the failure count / set OPEN (with TTL) → fallback branch.
  5. On success, reset to CLOSED.

> [!IMPORTANT]
> **🎯 Practice question 16 — error rate >10% in any 60 s → switch to an alternate path, auto-resume, finish in 15 min**
>
> - **Circuit breaker pattern:** a **Step Functions** workflow records **Bedrock invocation failures in DynamoDB** over a rolling 60-second window.
> - **Error rate > 10% → circuit OPEN** → route jobs to the **alternate path**. After a cool-down it goes **HALF-OPEN** and sends a probe request. If that succeeds → **CLOSED**, and normal Bedrock processing **resumes automatically**.
> - ❌ **Retry/Catch on a Task state alone:** handles **individual** errors and doesn't track an **aggregated error rate over time**.
> - ❌ **Lambda polling metrics + staff updating the workflow:** manual, so it's too slow for the 15-minute deadline.
> - ❌ **Several state machines routed by fastest response:** looks at latency, not the 10% error threshold.

```json
"InvokePrimaryModel": {
  "Type": "Task",
  "Resource": "arn:aws:states:::bedrock:invokeModel",
  "Parameters": { "ModelId": "us.anthropic.claude-...", "Body": { "...": "..." } },
  "Retry": [{
    "ErrorEquals": ["Bedrock.ThrottlingException", "Bedrock.ServiceUnavailableException", "Bedrock.ModelTimeoutException"],
    "IntervalSeconds": 2, "BackoffRate": 2.0, "MaxAttempts": 3, "JitterStrategy": "FULL"
  }],
  "Catch": [{ "ErrorEquals": ["States.ALL"], "ResultPath": "$.error", "Next": "OpenCircuitAndFallback" }],
  "Next": "CloseCircuit"
}
```
- **Error names:** Step Functions surfaces service errors with a service prefix (e.g. `Bedrock.ThrottlingException`). Check the exact names in the docs.
- **Alternative without Step Functions:** a circuit-breaker library inside Lambda, with state in DynamoDB or ElastiCache.

#### Graceful degradation strategies (ordered)
1. **Same model, other capacity:** CRIS / another Region / Provisioned Throughput.
2. **Fallback model:** a smaller model in the same family, or another provider via the router (1.2.2). Keep prompts compatible.
3. **Reduced features:** turn off expensive steps (reranking, multi-agent orchestration, long context), shorten `maxTokens`, use RAG-only extractive answers.
4. **Cached responses:** exact-match cache (ElastiCache/DynamoDB) or a **semantic cache** (vector similarity on previous questions).
5. **Static fallback:** FAQ answers, a templated "we're experiencing issues" message, or a hand-off to a human agent (e.g. Amazon Connect).
6. **Async deferral:** put the request in **SQS**, process it later, and notify via SNS/email/WebSocket.
- Always **tell the user** when the response is degraded, and **emit metrics** (`DegradedResponses`) with alarms.

#### Load-levelling & async patterns
- Use **SQS** in front of model calls to absorb spikes (with a DLQ for poison messages). Lambda event source mappings with **maximum concurrency** protect quotas.
- For long generations, use **asynchronous** processing (Step Functions, SageMaker Async Inference, Bedrock batch inference) instead of keeping a synchronous HTTP call open.
- API Gateway WebSocket API or streaming responses (Lambda response streaming) for long outputs.

---

### Skill 1.2.4 — FM customisation deployment & lifecycle management

> *Implement FM customization deployment and lifecycle management (for example, by using Amazon SageMaker AI to deploy domain-specific fine-tuned models, parameter-efficient adaptation techniques such as LoRA and adapters for model deployment, SageMaker Model Registry for versioning and to deploy customized models, automated deployment pipelines to update models, rollback strategies for failed deployments, lifecycle management to retire and replace models).*

#### When to customise (escalation ladder)

| Approach | Changes weights? | Data needed | Use when |
| --- | --- | --- | --- |
| Prompt engineering / few-shot | No | A few examples | Format, tone, simple behaviour |
| RAG | No | Documents | Up-to-date or proprietary **knowledge**, citations |
| **Fine-tuning (supervised, PEFT/LoRA)** | Yes (small subset) | Labelled prompt→completion pairs (hundreds to thousands) | Consistent **style/format/task skill**, domain jargon, shorter prompts |
| 🎯 **PQ1** Distillation | Yes (student) | Prompts (teacher generates the answers) | Cheaper, faster model with near-teacher quality |
| Continued pre-training | Yes | Large **unlabelled** domain corpus | Deep domain language (medical, legal) |
| Reinforcement fine-tuning / RLHF / DPO | Yes | Preference or reward data | Align with human preference or graders |
| Train from scratch | Yes (all) | Massive | Almost never the exam answer |

> [!IMPORTANT]
> **🎯 Practice question 1 — cut cost, keep quality, demand peaks in work hours**
>
> - Lower cost + **same quality** + variable demand → **Bedrock Model Distillation**: a large **teacher** trains a smaller **student in the same model family** (for example **Nova Pro → Nova Lite**). Then serve the student with **on-demand inference**, which is pay per token and scales with the peaks.
> - Training data can come from the teacher's **earlier invocation logs** (model invocation logging stored in S3), so you don't need hand-labelled data.
> - ❌ **Few-shot examples** add tokens to every prompt, which **raises** cost.
> - ❌ **Fine-tuning the same big model + Provisioned Throughput** means paying for fixed hourly capacity, so cost doesn't go down.
> - ❌ **A student from another family deployed on SageMaker JumpStart** means managing infrastructure, and it isn't on-demand for spiky load.

- ⚠️ **Fine-tuning ≠ adding knowledge reliably.** For facts that change, use **RAG**. Fine-tune for behaviour and style.

#### Parameter-efficient fine-tuning (PEFT)
- **Full fine-tuning** updates all weights. It needs lots of GPU memory, produces a full model copy per task, and risks **catastrophic forgetting**.
- **LoRA (Low-Rank Adaptation):** freeze the base weights `W` and learn `ΔW = B·A`, where A and B have low **rank r** (e.g. 8–64). Only A and B are trained, often <1% of the parameters.
  - Hyperparameters: `r` (rank), `lora_alpha` (scaling), `target_modules` (e.g. attention q/v projections), dropout.
  - The **adapter** artifact is small (MBs vs GBs). You can **merge** it into the base weights for deployment (no inference overhead) or keep it **separate** and swap it at runtime.
- **QLoRA:** LoRA on a **quantised (4-bit)** base model. Fine-tune big models on smaller GPUs.
- **Adapters (general):** small trainable modules inserted between layers. **Prefix/prompt tuning** learns virtual tokens. The idea is the same: train only a small part of the model.
- **Deployment benefit:** **one base model + many LoRA adapters** (per tenant, domain or task) on shared GPUs. This is much cheaper than one endpoint per fine-tuned model.

#### Where to fine-tune & host

| Option | Notes |
| --- | --- |
| **Bedrock model customisation** (fine-tuning, continued pre-training, distillation, reinforcement fine-tuning on supported models) | Training data in S3 (JSONL). Output is a **custom model**. Inference typically needs **Provisioned Throughput**; some models support **on-demand inference for custom models** (verify per model). Fully managed, no infrastructure. |
| 🎯 **PQ46** **Bedrock Custom Model Import** | Bring weights fine-tuned elsewhere (e.g. on SageMaker) for **supported open architectures** (Llama, Mistral, Mixtral, Flan-T5, Qwen etc., verify list) in Hugging Face safetensors format from S3. Serverless, billed per **Custom Model Unit** in active 5-minute windows (verify). Can scale to zero, so expect a **cold start** (`ModelNotReadyException`). Uses Bedrock APIs, Guardrails and KBs. |
| **SageMaker AI (JumpStart / training jobs / HyperPod)** | Full control: any open model, custom training scripts (Hugging Face PEFT), distributed training, Spot training. Pick when you need a custom algorithm, a model or architecture Bedrock doesn't support, or control over the infrastructure. |
| **SageMaker AI hosting** | Real-time endpoints with **LMI (Large Model Inference) containers** (DJL Serving with vLLM, TensorRT-LLM, LMI-Dist backends), the Hugging Face TGI container, or a custom container. |

#### SageMaker AI inference options (pick correctly)

| Option | Best for | Limits / notes |
| --- | --- | --- |
| **Real-time endpoint** | Low-latency, steady traffic, GPUs | Always-on instances; auto scaling on invocations/GPU metrics |
| **Inference components** | Many models or **LoRA adapters** on one endpoint; each with its own CPU/GPU/memory allocation and scaling (can scale to zero) | Best for multi-tenant fine-tunes; adapter inference components sit on a base-model component (verify) |
| **Multi-model endpoint (MME)** | Many small models loaded on demand from S3 | Cold load on first request; less suited to large LLMs |
| **Serverless inference** | Intermittent traffic, small models | **No GPU**; cold starts; memory up to ~6 GB |
| **Asynchronous inference** | Large payloads / long processing, queued | Payload up to **1 GB**, processing up to **1 hour**; can scale to zero; SNS notifications |
| **Batch transform** | Offline scoring of datasets in S3 | No persistent endpoint |

#### SageMaker Model Registry — versioning & governance
- **Model package group** = one logical model (e.g. `support-summariser-llama-lora`). **Model package** = a versioned entry (v1, v2…) pointing to the artifacts in S3, the inference image, metrics, and lineage.
- **Approval status:** `PendingManualApproval` → `Approved` / `Rejected`. CI/CD deploys only **Approved** versions.
- Attach **evaluation metrics**, **model cards** (intended use, risk rating, evaluation results) and lineage (training job, dataset) for **auditability**.
- An approval status change emits an **EventBridge** event → triggers CodePipeline or a SageMaker Project deploy pipeline.
- Cross-account: a central registry in a shared-services account (resource policies) deploys to staging and prod accounts.
- **MLflow** model registry on SageMaker (managed MLflow) is an alternative registry. Registered MLflow models can sync with the SageMaker Model Registry (verify).

#### Automated deployment pipeline (MLOps for FMs)
```
Data in S3 → SageMaker Pipeline:
   Process data → Fine-tune (LoRA) → Evaluate (fmeval / Bedrock eval / custom metrics)
   → Condition: metrics ≥ threshold? → Register model (PendingManualApproval)
→ Human approves in Model Registry (or auto-approve in dev)
→ EventBridge → CodePipeline: deploy to staging → integration + prompt regression tests
→ Manual gate → deploy to prod with deployment guardrails (canary/linear) + alarms
```
- **SageMaker Projects** provide MLOps templates (build + deploy pipelines). Infrastructure as code with CloudFormation/CDK.
- Tests include **prompt regression suites**, safety evals (toxicity, jailbreak), latency/cost benchmarks and **golden datasets**.

#### Safe rollout & rollback strategies

| Strategy | How | Rollback |
| --- | --- | --- |
| **Blue/green all-at-once** (deployment guardrails) | New fleet (green) is provisioned; traffic shifts at once; **baking period** | Auto-rollback to blue if **CloudWatch alarms** fire during baking |
| **Canary** | Small % of traffic first (e.g. 10%), bake, then the rest | Auto-rollback on alarm |
| **Linear** | Shift in equal steps (e.g. 20% every N min) | Auto-rollback on alarm |
| **Shadow testing** (SageMaker shadow tests / shadow variants) | Copy of production traffic goes to the new model; **responses not returned to users** | No user impact; promote if metrics are good |
| **Production variants (A/B)** | Weighted traffic across variants on one endpoint | Set the weight back to the old variant |
| **Bedrock / config-driven** | AppConfig gradual rollout of the new model ID / custom model ARN | AppConfig auto-rollback on alarm |
| **Registry-driven rollback** | Re-deploy the previous **Approved** model package version | Fast, auditable |

- Alarm on: 5xx/4xx errors, model latency, GPU memory, **quality signals** (thumbs-down rate, eval score from online sampling), guardrail intervention rate.
- Keep the previous version's artifacts and endpoint config until the new version has baked. **Don't delete blue too early.**

#### Lifecycle management — retire & replace
- **Bedrock base model lifecycle:** **Active** → **Legacy** (at least **6 months' notice** before EOL; no new Provisioned Throughput; new customers may lose access) → **EOL** (calls fail; **no automatic migration**). Each model is available for at least 12 months after launch.
  - Customised models on Provisioned Throughput bought **before the Legacy date** keep working for a period after base-model EOL (existing notes: 12 months; verify in docs).
  - **Action plan:** subscribe to deprecation notices (AWS Health / email). Inventory the model IDs in use (CloudTrail, invocation logs, AppConfig). Evaluate the replacement with the same eval suite. Re-tune prompts. Re-run fine-tuning on the new base if needed. Switch through config with a canary, then remove the old IDs from IAM allow-lists.
- **SageMaker:** mark old model package versions `Rejected` / archive them, delete unused endpoints and endpoint configs (cost), keep artifacts per your **retention policy** (S3 lifecycle), and update **model cards** (status: archived).
- Track **ownership, purpose, data lineage and risk** for each model for governance (links to Domain 3).
- Re-evaluate periodically for **drift**: data drift and changing user behaviour (SageMaker Model Monitor for traditional ML; for LLMs, online evaluation sampling and user feedback).

#### Cost notes for customised models
- Provisioned Throughput: commitment options (no commitment, 1 month, 6 months). Billed **hourly** whether or not it is used.
- Custom Model Import: billed per active model copy in 5-minute windows. Storage is billed monthly (verify).
- SageMaker: instance-hours. Use inference components / LoRA adapters to share GPUs, and auto scaling (including scale-to-zero for inference components) for idle time.

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| "Compare 3 candidate models on our own support transcripts for accuracy and tone" | **Bedrock Model Evaluation**: automatic + human (tone) or **LLM-as-a-judge** |
| "Public benchmark leader performed poorly on our data" | Benchmarks only shortlist → **evaluate on domain dataset** |
| "Mix of simple and complex queries, minimise cost without losing quality" | **Bedrock Intelligent Prompt Routing** (same family) or Lambda router / cascade |
| "Switch model or provider without redeploying code" | **Converse API** + model ID in **AppConfig** (validators, gradual rollout, auto-rollback) behind **API Gateway → Lambda** |
| "A/B test a new model on 10% of users" | AppConfig multi-variant flags / gradual deployment, or SageMaker production variants |
| "Invalid model ID pushed to config broke production" | AppConfig **validator** (JSON Schema / Lambda) + CloudWatch alarm **rollback** |
| "ThrottlingException during spikes; model has limited Regional availability" | **Cross-Region inference profile** (no code change) |
| "Data must stay within the EU, but need more throughput" | **EU geographic** inference profile (not global) |
| "CRIS calls fail with AccessDenied" | IAM must allow model in **all destination Regions**; check **SCP** Region denies |
| "Stop hammering a failing model; fall back fast" | **Step Functions circuit breaker** with **DynamoDB** state + fallback branch |
| "Retry transient Bedrock errors in Step Functions" | `Retry` with `BackoffRate`, `MaxAttempts`, `JitterStrategy: FULL`, then `Catch` |
| "ValidationException is being retried repeatedly" | Don't retry 4xx validation errors; fix input (token limits / format) |
| "Keep serving users when the primary model is down" | Graceful degradation: fallback model → cache → static answer → async queue |
| "Region-wide outage of the whole GenAI app" | Multi-Region deployment + **Route 53 failover**, replicated data (S3 CRR, DynamoDB global tables) |
| "Many tenant-specific fine-tunes, minimise GPU cost" | **LoRA adapters** on one base model with SageMaker **inference components** |
| "Fine-tune a 70B model on limited GPU memory" | **QLoRA** / LoRA (PEFT) |
| "Version, approve and audit fine-tuned models" | **SageMaker Model Registry** (model package groups, approval status, model cards) |
| "Automatically deploy when a model is approved" | Registry approval → **EventBridge** → CodePipeline / SageMaker Projects |
| "Roll out new fine-tuned model gradually with automatic rollback" | SageMaker **deployment guardrails** (canary/linear) + CloudWatch alarms |
| "Test new model on live traffic with zero user impact" | SageMaker **shadow testing** |
| "Fine-tuned Llama on SageMaker; want to use it via Bedrock APIs with Guardrails" | **Bedrock Custom Model Import** |
| "First request to imported model fails with ModelNotReadyException" | Cold start (scaled to zero) → retry with backoff |
| "Can't buy Provisioned Throughput for a model" | Model is **Legacy** → plan migration |
| "Model reaches EOL" | Calls fail, no auto-migration → evaluate replacement, switch through config, canary |
| "Need facts that change weekly" | **RAG**, not fine-tuning |
| "Need consistent output style / domain format" | **Fine-tuning** (PEFT) after prompt engineering is exhausted |
| "Large payloads, minutes-long inference, scale to zero" | SageMaker **Asynchronous Inference** |

## Flashcards

<details>
<summary><b>Q:</b> Why aren't public benchmarks enough to choose an FM?</summary>

They may be contaminated, may not match your domain, and say nothing about your latency or cost. Use them to shortlist, then evaluate on your own representative data (e.g. Bedrock Model Evaluation).

*D1 · Task 1.2 · Benchmarks*
</details>

<details>
<summary><b>Q:</b> What are the Bedrock Model Evaluation types?</summary>

Automatic (programmatic metrics such as accuracy, robustness, toxicity), human (your own or an AWS-managed work team), LLM-as-a-judge, and RAG/Knowledge Base evaluation. Results go to S3.

*D1 · Task 1.2 · Model evaluation*
</details>

<details>
<summary><b>Q:</b> What does Bedrock Intelligent Prompt Routing do, and what is its key constraint?</summary>

It routes each request through one router ARN to the model predicted to give the best quality for the cost. It routes only between models in the same family.

*D1 · Task 1.2 · Prompt routing*
</details>

<details>
<summary><b>Q:</b> Why use the Converse API rather than InvokeModel for provider switching?</summary>

Converse has one request and response format for all Bedrock models, so switching only changes the modelId. InvokeModel needs a provider-specific body.

*D1 · Task 1.2 · Converse API*
</details>

<details>
<summary><b>Q:</b> Which AppConfig features make model switching safe?</summary>

Validators (JSON Schema or Lambda), gradual deployment strategies with bake time, CloudWatch-alarm auto-rollback, feature flags and multi-variant flags, and the Lambda extension for cached retrieval.

*D1 · Task 1.2 · AppConfig*
</details>

<details>
<summary><b>Q:</b> What is the role of API Gateway in a flexible model architecture?</summary>

It is a stable façade (e.g. /v1/generate) that hides the model and provider. It also handles auth, throttling, usage plans per tenant and request validation, while a Lambda router picks the model from config.

*D1 · Task 1.2 · API Gateway façade*
</details>

<details>
<summary><b>Q:</b> What types of Bedrock inference profiles are there?</summary>

System-defined cross-Region profiles: geographic (us., eu., apac., stay in the geography) and global (worldwide routing). Application inference profiles are user-created, for cost tags and usage tracking.

*D1 · Task 1.2 · Inference profiles*
</details>

<details>
<summary><b>Q:</b> What IAM gotcha breaks cross-Region inference?</summary>

The policy must allow the inference profile and the foundation model in every destination Region, and SCPs must not deny those Regions.

*D1 · Task 1.2 · CRIS IAM*
</details>

<details>
<summary><b>Q:</b> Which Bedrock errors should be retried and which shouldn't?</summary>

Retry ThrottlingException (429), ServiceUnavailableException (503), ModelTimeoutException (408), InternalServerException (500) and ModelNotReadyException with backoff and jitter. Don't retry ValidationException (400) or AccessDeniedException (403).

*D1 · Task 1.2 · Error handling*
</details>

<details>
<summary><b>Q:</b> What are the three circuit breaker states?</summary>

CLOSED (normal calls), OPEN (skip the failing dependency and use the fallback until a timeout), HALF-OPEN (allow a probe; success closes the circuit, failure reopens it).

*D1 · Task 1.2 · Circuit breaker*
</details>

<details>
<summary><b>Q:</b> How do you implement a circuit breaker for Bedrock on AWS?</summary>

A Step Functions workflow reads the circuit state from DynamoDB (TTL for the open timeout), chooses the primary or fallback, and invokes Bedrock with Retry plus Catch. On failure it updates the state to OPEN; on success it resets to CLOSED.

*D1 · Task 1.2 · Circuit breaker*
</details>

<details>
<summary><b>Q:</b> List a graceful degradation ladder for a GenAI chat app.</summary>

Same model in another Region (CRIS) → smaller or other-provider model → reduced features (no rerank, shorter output) → cached or semantic-cache answer → static or templated response or human hand-off → async queue with notification.

*D1 · Task 1.2 · Graceful degradation*
</details>

<details>
<summary><b>Q:</b> Is CRIS a full disaster-recovery solution?</summary>

No. It only spreads model inference across Regions. Your own stack (Lambda, KB, vector store, API) in the source Region still needs a multi-Region design (Route 53 failover, replicated data).

*D1 · Task 1.2 · Multi-Region*
</details>

<details>
<summary><b>Q:</b> How does LoRA work, and why is it cheap?</summary>

It freezes the base weights and learns a low-rank update ΔW = B·A (rank r), so it trains a tiny fraction of the parameters. The adapter is only MBs and can be merged into the base model or swapped at runtime.

*D1 · Task 1.2 · LoRA*
</details>

<details>
<summary><b>Q:</b> What is QLoRA?</summary>

LoRA fine-tuning on a base model quantised to 4 bits. It lets you fine-tune large models on smaller or fewer GPUs.

*D1 · Task 1.2 · QLoRA*
</details>

<details>
<summary><b>Q:</b> How do you host dozens of tenant-specific fine-tunes cheaply on SageMaker AI?</summary>

Host one base model and attach many LoRA adapters as SageMaker inference components on a shared endpoint. Each scales independently and can scale to zero.

*D1 · Task 1.2 · Inference components*
</details>

<details>
<summary><b>Q:</b> What are the SageMaker Model Registry building blocks?</summary>

Model package groups (logical model), model packages (versions with artifacts, metrics and lineage), approval status (PendingManualApproval/Approved/Rejected), and model cards. Approval changes emit EventBridge events.

*D1 · Task 1.2 · Model Registry*
</details>

<details>
<summary><b>Q:</b> Which SageMaker feature gives canary or linear rollout with automatic rollback?</summary>

Deployment guardrails (blue/green with all-at-once, canary or linear traffic shifting, plus a baking period and CloudWatch-alarm auto-rollback).

*D1 · Task 1.2 · Deployment guardrails*
</details>

<details>
<summary><b>Q:</b> How do you validate a new model on real production traffic without affecting users?</summary>

SageMaker shadow testing: a copy of the traffic goes to the shadow variant and its responses are logged but not returned to users.

*D1 · Task 1.2 · Shadow testing*
</details>

<details>
<summary><b>Q:</b> When would you use Bedrock Custom Model Import?</summary>

When you fine-tuned a supported open-weight architecture (e.g. Llama, Mistral) elsewhere and want to serve it serverlessly through Bedrock APIs with Guardrails and Knowledge Bases. Expect possible cold starts (ModelNotReadyException).

*D1 · Task 1.2 · Custom Model Import*
</details>

<details>
<summary><b>Q:</b> What happens across the Bedrock model lifecycle?</summary>

Active → Legacy (≥6 months' notice, no new Provisioned Throughput) → EOL (requests fail, no automatic migration). Plan replacement through evaluation and a config-driven canary switch.

*D1 · Task 1.2 · Model lifecycle*
</details>

<details>
<summary><b>Q:</b> Fine-tuning or RAG for product prices that change weekly?</summary>

RAG. Fine-tuning teaches behaviour and style and doesn't reliably keep facts current. RAG retrieves up-to-date data at query time.

*D1 · Task 1.2 · Customisation choice*
</details>

<details>
<summary><b>Q:</b> Which SageMaker inference option suits 500 MB payloads taking 20 minutes to process?</summary>

Asynchronous Inference: payloads up to 1 GB, processing up to 1 hour, queued, can scale to zero, with SNS notifications.

*D1 · Task 1.2 · SageMaker inference options*
</details>

<details>
<summary><b>Q:</b> What does an automated FM deployment pipeline look like?</summary>

SageMaker Pipeline (process → fine-tune → evaluate → condition → register as PendingManualApproval) → approval → EventBridge → CodePipeline deploys to staging with regression and safety tests → prod via canary with alarm-based rollback.

*D1 · Task 1.2 · MLOps pipeline*
</details>

---
[← Back to Domain 1](README.md)
