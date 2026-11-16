[← Back to Domain 1](README.md)

# 🧭 Task 1.1 · Analyze requirements and design GenAI solutions

> **Domain 1** · Foundation Model Integration, Data Management, and Compliance (31%)
>
> - **Skill 1.1.1** Create comprehensive architectural designs that align with specific business needs and technical constraints (for example, by using appropriate FMs, integration patterns, deployment strategies).
> - **Skill 1.1.2** Develop technical proof-of-concept implementations to validate feasibility, performance characteristics, and business value before proceeding to full-scale deployment (for example, by using Amazon Bedrock).
> - **Skill 1.1.3** Create standardized technical components to ensure consistent implementation across multiple deployment scenarios (for example, by using the AWS Well-Architected Framework, AWS WA Tool Generative AI Lens).

## Key points
- **Requirements first, model second.** Turn the business goal into measurable **functional** requirements (task, modality, languages, output format) and **non-functional** ones (latency SLO, throughput/TPM, cost per request, data residency, compliance, availability). Then pick the model, the integration pattern and the deployment option.
- **Customisation ladder** (cheapest/fastest first): **prompt engineering → RAG → agents/tool use → fine-tuning / distillation → continued pre-training → train from scratch**. Need fresh or private facts → **RAG**. Need a style, format or domain *behaviour* → **fine-tune**. Need a smaller, cheaper model that keeps a big model's quality → **distillation**.
- **Deployment options:** **Bedrock on-demand** (serverless, pay per token) · **Bedrock batch** (async S3 JSONL jobs, cheaper, for work nobody is waiting on) · **Provisioned Throughput** (reserved model units; needed for most custom models) · **cross-Region inference profiles** (more burst capacity) · **Custom Model Import / Bedrock Marketplace** · **SageMaker AI endpoints** (real-time, serverless, async, batch transform: full control, open-weight or self-trained models).
- **Integration patterns:** sync request/response (API Gateway + Lambda + Converse) · **streaming** (ConverseStream via WebSocket API, Lambda response streaming or AppSync) · **async/queue** (SQS/EventBridge → Lambda/Step Functions) · **batch** (S3 → Bedrock batch inference) · **RAG** (Knowledge Bases) · **agentic** (Bedrock Agents / AgentCore + tools).
- **Use the Converse API** for model-agnostic code: one request/response shape across providers, plus tool use, system prompts, guardrails and document/image inputs. Use `InvokeModel` only when you need model-native request bodies (for example, embeddings or image generation) - The Converse API is Amazon Bedrock’s unified, model-agnostic API for chat-style interactions with foundation models. Instead of writing a different request format for each model provider (Anthropic, Meta, Amazon Nova, Mistral, Cohere and so on), you send one standard request shape and Bedrock translates it for whichever model you pick. Why it exists? The older InvokeModel API requires each model’s own native JSON body, so switching from Claude to Llama meant rewriting your payload and response parsing. With Converse you change only the modelId.

- **Use a PoC to reduce risk.** Fix the success criteria **before** you build: quality metrics, latency (TTFT/E2E), cost per transaction, business KPI. Build on **Bedrock** (playgrounds → Converse → Knowledge Bases/Agents) with a **golden dataset**, compare models with **Bedrock Model Evaluation** (automatic, human, LLM-as-a-judge, RAG evaluation), then make a go/no-go decision.
- **PoC → production gap:** quotas and throttling, guardrails, IAM least privilege, VPC endpoints (PrivateLink), observability (invocation logging, CloudWatch, X-Ray), cost controls (application inference profiles + tags), CI/CD, evaluation in the pipeline.
- **Standardise with reusable building blocks:** IaC (CDK / CloudFormation, **Generative AI CDK Constructs**), **AWS Service Catalog** products, shared **Prompt Management** templates, central **Guardrails**, a shared **LLM gateway** (API Gateway + Lambda), Lambda layers/SDK wrappers, tagging standards, Organizations **SCPs**.
- **Well-Architected Generative AI Lens** applies the **6 pillars** (Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimisation, Sustainability) across the GenAI lifecycle: **scoping → model selection → customisation → development & integration → deployment → continuous improvement**. To use it, **import it into the WA Tool as a custom lens** (JSON from the AWS samples GitHub repo).
- **WA Tool workflow:** define the workload → apply lenses → answer questions → get **High/Medium Risk Issues (HRI/MRI)** → **improvement plan** → save **milestones** and re-review. Use **review templates** and **profiles** to make reviews repeatable across teams.
- **Generative AI Security Scoping Matrix:** Scope **1** consumer app · **2** enterprise app · **3** pre-trained models (for example, Bedrock FMs) · **4** fine-tuned models · **5** self-trained models. A higher scope means **you own more of the security responsibility**.

## Notes

### Skill 1.1.1 — Architectural designs aligned to business needs and technical constraints

> Create comprehensive architectural designs that align with specific business needs and technical constraints (for example, by using appropriate FMs, integration patterns, deployment strategies).

#### 1. Requirements analysis: what to capture before drawing boxes
- **Business requirements**
  - Problem statement + **measurable outcome** (for example, "cut average handle time by 20%", "deflect 30% of tier-1 tickets").
  - Users and channels (internal staff, external customers, API consumers), plus expected **volume** (requests/day, peak RPS, concurrent sessions).
  - **Value vs risk**: what happens if the model is wrong? Low risk (marketing draft) → light controls. High risk (medical, financial, legal advice) → human-in-the-loop, grounding, citations, guardrails.
  - Budget: **cost per request/conversation**, monthly cap, ROI threshold.
- **Functional requirements**
  - Task type: generation, summarisation, Q&A, classification, extraction, code, translation, reasoning, multi-step actions.
  - **Modality**: text in/out, image understanding, image generation, audio (speech → Transcribe; speech-to-speech → Nova Sonic), video, documents (PDF/Office).
  - **Knowledge needs**: general knowledge vs **private/up-to-date** enterprise data (→ RAG) vs live system data/actions (→ tools/agents).
  - Output contract: free text vs **structured JSON** (tool use / JSON schema), citations, specific tone or format.
  - Languages, context length (long documents, multi-turn memory).
- **Non-functional requirements (NFRs)**
  - **Latency**: interactive chat → optimise **TTFT** and stream the output. Backend/document jobs → **end-to-end latency**. Offline → batch.
  - **Throughput**: tokens per minute (TPM) and requests per minute (RPM) against Bedrock **service quotas**.
  - **Availability / resilience**: multi-AZ is built into Bedrock. Add cross-Region inference, fallback models and graceful degradation (see Task 1.2).
  - **Security & compliance**: data classification, PII, **data residency** (choose a *geographic* inference profile such as `eu.` / `us.` rather than *global*), encryption (KMS CMKs), private connectivity (VPC interface endpoints), audit (CloudTrail).
  - **Responsible AI**: toxicity, bias, prompt-injection exposure → Bedrock Guardrails, evaluation, human review.
  - **Operability**: observability, versioning of prompts and models, rollback.
  - **Sustainability**: right-size the model, prefer serverless/managed, avoid unnecessary training.

#### 2. Mapping requirements to the customisation approach

| Need | Approach | Why / gotcha |
| --- | --- | --- |
| Better instructions, format, few examples | **Prompt engineering** (+ Prompt Management) | Cheapest and fastest; always try first |
| Answers must use private, current or changing data, with citations | **RAG** (Bedrock Knowledge Bases) | No retraining; data is fresh after a sync; reduces hallucination |
| Must act on systems / call APIs / multi-step | **Agents / tool use** (Bedrock Agents, AgentCore, Converse tool use) | Adds latency and cost per step; needs IAM scoping |
| Consistent style/tone/format, domain jargon, a narrow task at scale | **Fine-tuning** (Bedrock or SageMaker; LoRA/PEFT) | Needs labelled data; Bedrock custom models usually need **Provisioned Throughput** or on-demand custom model deployment where supported (verify in docs per model) |
| Big-model quality at small-model cost/latency | **Model distillation** (Bedrock) | Teacher → student; only the supported model pairs |
| Large unlabelled domain corpus (new vocabulary) | **Continued pre-training** | Expensive; rarely the exam's answer |
| Totally novel domain/data, full control | **Train from scratch** (SageMaker HyperPod) | Very expensive; Security Scoping Matrix scope 5 |

- ⚠️ A classic exam trap: "knowledge changes weekly" → **RAG, not fine-tuning**. Fine-tuning bakes facts in at training time, so they go stale and cannot be cited.
- Combine approaches: a **fine-tuned model + RAG** works well when the model needs domain *behaviour* and also *current facts*.

#### 3. Choosing the FM at design time (detail in Task 1.2)
- Filter by **capability** (modality, context window, tool use, languages, reasoning) → **quality** (your own eval set, not only public benchmarks) → **latency** → **cost** → **compliance** (Region availability, data residency, licence/EULA).
- Model families on Bedrock (examples): Anthropic Claude, Amazon Nova (Micro/Lite/Pro/Premier, Canvas, Reel, Sonic), Meta Llama, Mistral, Cohere (Command, Embed, Rerank), AI21, DeepSeek, Qwen, OpenAI open-weight (gpt-oss), Amazon Titan (embeddings, image). Availability varies by Region, so check the model catalogue.
- Don't hard-code a single model. Put the model ID in config (AppConfig / Parameter Store / Prompt Management) so you can switch models without changing code (Skill 1.2.2).

#### 4. Integration patterns

| Pattern | Typical AWS build | Use when | Watch out |
| --- | --- | --- | --- |
| **Synchronous request/response** | API Gateway (REST/HTTP) → Lambda → Bedrock `Converse` | Short generations, simple APIs | API Gateway integration timeout (29 s default; regional/private REST APIs can request a higher limit, so verify in docs). Long generations → stream or go async |
| **Streaming** | `ConverseStream` / `InvokeModelWithResponseStream` via **API Gateway WebSocket**, **Lambda response streaming** (function URL), **AppSync** subscriptions, or a container on ECS/App Runner | Chat UX; perceived latency matters (TTFT) | Guardrails can run on streams in sync or async mode. Async = lower latency but harmful text may show before it's caught |
| **Asynchronous / queue-based** | Client → API → **SQS** → Lambda worker → Bedrock → result in S3/DynamoDB → notify (SNS/EventBridge/WebSocket) | Spiky load, long tasks, protecting quotas | Need idempotency and DLQs; the client polls or gets a callback |
| **Event-driven** | S3 upload / DynamoDB stream / EventBridge rule → Lambda or Step Functions → Bedrock | Document processing, enrichment on arrival | Fan-out can exceed TPM quotas, so throttle with SQS and reserved concurrency |
| **Orchestrated workflow** | **Step Functions** (direct Bedrock `InvokeModel` integration, Map state, retries, human approval with task tokens) | Multi-step pipelines, retries, circuit breakers, HITL | Standard vs Express workflows (duration/cost trade-off) |
| **Batch** | Input JSONL in S3 → **Bedrock batch inference** job (`CreateModelInvocationJob`) → output JSONL in S3 | Large offline jobs (summarise 1M records), cost-sensitive | Not real-time; job completion is signalled via EventBridge |
| **RAG** | Bedrock **Knowledge Bases** (`RetrieveAndGenerate` or `Retrieve`) + vector store (OpenSearch Serverless, S3 Vectors, Aurora pgvector, etc.) | Grounded Q&A over enterprise content | Chunking/embedding choices are fixed at KB creation (see Tasks 1.4/1.5) |
| **Agentic** | **Bedrock Agents** (action groups = Lambda/OpenAPI, KBs) or **AgentCore** (Runtime, Gateway, Memory, Identity, Observability) with Strands/LangGraph/CrewAI | Goal-driven tasks across tools/APIs | Many more calls per user request, so plan quotas, cost and guardrails |
| **LLM gateway / proxy** | API Gateway + Lambda (or a container) in front of Bedrock + other providers; central auth, logging, quotas, routing | Many teams/apps sharing GenAI access | Becomes a shared platform component (Skill 1.1.3) |

```python
# Model-agnostic call with the Converse API (boto3)
import boto3, os
brt = boto3.client("bedrock-runtime")
resp = brt.converse(
    modelId=os.environ["MODEL_ID"],          # e.g. an inference profile ID like "eu.<model-id>"
    system=[{"text": "You are a concise support assistant."}],
    messages=[{"role": "user", "content": [{"text": "Summarise ticket #123"}]}],
    inferenceConfig={"maxTokens": 512, "temperature": 0.2},
    guardrailConfig={"guardrailIdentifier": "gr-abc123", "guardrailVersion": "3"},
)
print(resp["output"]["message"]["content"][0]["text"], resp["usage"])
```

#### 5. Deployment strategies (where the model runs)

| Option | What it is | Best for | Trade-offs |
| --- | --- | --- | --- |
| **Bedrock on-demand** | Serverless, pay per input/output token | Most apps, variable traffic, fastest start | Subject to TPM/RPM quotas → throttling (`ThrottlingException`) |
| **Bedrock cross-Region inference (inference profiles)** | Routes requests across Regions in a geography (`us.`, `eu.`, `apac.`…) or **global** | Burst capacity, higher throughput, models only available in some Regions | Data may be processed in another Region of the geography; global profiles may break residency rules |
| **Bedrock Provisioned Throughput** | Reserved **model units** (commitment terms or no-commitment, verify current terms) | Steady high volume, predictable latency, **custom models** | Pay whether you use it or not; can't buy it for Legacy models |
| **Bedrock batch inference** | Async job over S3 JSONL | Large offline workloads at a discount vs on-demand (commonly ~50% for supported models, verify) | Hours, not seconds; quotas on records per job |
| **Bedrock service tiers / latency-optimised inference** | Priority/Standard/Flex tiers and latency-optimised options for selected models (verify current availability) | Tuning cost vs latency for each workload | Model/Region specific |
| **Bedrock Custom Model Import** | Bring your own open-weight model weights (for example, Llama/Mistral/Qwen-based fine-tunes) and serve them serverlessly on Bedrock | Use your SageMaker-fine-tuned model behind the Bedrock API | Only supported architectures |
| **Bedrock Marketplace** | 100+ specialised models deployed onto managed endpoints, accessed through Bedrock APIs | Niche/specialist models | You choose the instance type and pay for it |
| **SageMaker AI real-time endpoint** | Dedicated instances (GPU/Inferentia), autoscaling, **inference components** (several models per endpoint) | Full control, custom containers, LoRA adapters, self-trained models | You manage scaling and cost of idle capacity |
| **SageMaker serverless / async / batch transform** | Serverless = scale to zero (no GPU) · async = large payloads, long processing, queue + scale to zero · batch transform = offline | Intermittent or heavy-payload workloads | Cold starts (serverless); not interactive (batch) |
| **Self-managed (EKS/EC2 + vLLM, Inferentia/Trainium)** | DIY hosting | Extreme customisation, existing K8s platform | Highest operational burden |

- Rule of thumb for the exam: **"least operational overhead" → Bedrock (serverless)**. "Must run our own fine-tuned open-source model with custom container / full control" → **SageMaker AI**. "Already fine-tuned an open-weight model, want serverless Bedrock APIs" → **Custom Model Import**.

#### 6. Reference architecture: enterprise RAG chatbot (worked example)
1. **Front end**: Amplify-hosted web app → **Cognito** auth (JWT) → **API Gateway WebSocket** API.
2. **Orchestration**: Lambda (or an AgentCore Runtime agent) reads the conversation history from **DynamoDB** (session ID partition key, TTL for expiry).
3. **Prompt**: pulls a versioned template from **Bedrock Prompt Management**; model ID from **AppConfig**.
4. **Retrieval**: Bedrock **Knowledge Base** `Retrieve` with **metadata filters** (department, ACL tag) → optional **reranker**.
5. **Generation**: `ConverseStream` via a **cross-Region inference profile** with a **guardrail** attached (PII masking, denied topics, contextual grounding check).
6. **Data plane**: documents in **S3** (SSE-KMS) → KB sync triggered via **EventBridge** on S3 change → vector store (**OpenSearch Serverless** for hybrid search or **S3 Vectors** for lowest cost).
7. **Security**: VPC interface endpoints for `bedrock-runtime`, `bedrock-agent-runtime`; least-privilege IAM role per Lambda; KMS CMKs; CloudTrail.
8. **Observability**: Bedrock **model invocation logging** → CloudWatch Logs/S3; CloudWatch metrics (`Invocations`, `InvocationLatency`, `InputTokenCount`, `OutputTokenCount`, `InvocationThrottles`); X-Ray tracing; a user-feedback loop (thumbs up/down → DynamoDB → evaluation dataset).
9. **Cost**: **application inference profiles** tagged per app/team → Cost Explorer allocation; **prompt caching** for long, repeated system prompts/context.

#### 7. Architecture trade-off cheat sheet
- **Latency ↓**: smaller model, streaming, prompt caching, latency-optimised inference, fewer RAG chunks, rerank to top-k, put the app in the same Region, avoid agent loops.
- **Cost ↓**: smaller/distilled model, **intelligent prompt routing** (same family, router decides per prompt), batch inference, prompt caching, cap `maxTokens`, semantic cache, cheaper vector store (S3 Vectors).
- **Quality ↑**: better prompts → RAG with reranking/hybrid search → bigger or reasoning model → fine-tuning; evaluate every change.
- **Compliance ↑**: geographic (not global) inference profiles, VPC endpoints, KMS, guardrails, invocation logging, HITL. Bedrock doesn't use your prompts/outputs to train base models and doesn't share them with model providers.

---

### Skill 1.1.2 — Technical proof-of-concept (PoC) implementations

> Develop technical proof-of-concept implementations to validate feasibility, performance characteristics, and business value before proceeding to full-scale deployment (for example, by using Amazon Bedrock).

#### 1. Why a PoC and what it must prove
- **Feasibility**: can an FM do the task at all to an acceptable quality with the available data?
- **Performance characteristics**: latency (TTFT, E2E), throughput, token usage per request, behaviour at quota limits.
- **Business value**: does it move the KPI enough to justify the run cost? (cost per transaction × volume vs savings/revenue).
- **Risk discovery**: hallucination rate, harmful content, prompt-injection exposure, data-quality gaps, compliance blockers.
- Exit with a **go / no-go / pivot** decision backed by numbers, not demos.

#### 2. PoC lifecycle (practical steps)
1. **Scope narrowly**: one use case, one user group, a time-box (for example, 2–6 weeks).
2. **Define success criteria up front** (written and agreed with stakeholders):

| Dimension | Example metric | Example threshold |
| --- | --- | --- |
| Quality | Correctness / faithfulness / helpfulness scores (LLM-as-a-judge or human), ROUGE/BERTScore for summarisation, exact-match for extraction | ≥ 85% judged correct on a golden set |
| Grounding | Faithfulness / citation precision (RAG evaluation) | ≥ 90% faithful |
| Latency | p50/p95 TTFT, E2E | TTFT < 1.5 s p95 |
| Cost | $ per conversation (input + output tokens + retrieval) | < $0.02 |
| Safety | Guardrail intervention rate, red-team pass rate | 0 critical failures |
| Business | Deflection rate, handle time, user CSAT | +20% deflection |

3. **Build a golden dataset**: realistic prompts plus reference answers/expected facts, including edge cases, adversarial inputs and out-of-scope questions. Keep it versioned in S3.
4. **Prototype fast on Bedrock**
   - **Bedrock console playgrounds** (chat/text/image): try models, compare outputs side by side, tune inference parameters, see token counts and latency. You can **export the call as API code**.
   - **Amazon Bedrock in SageMaker Unified Studio** (formerly Bedrock Studio/IDE): build chat agents, flows and KB apps collaboratively without managing infrastructure.
   - **PartyRock**: no-code playground for business stakeholders to explore ideas (not for production or private data).
   - **Bedrock Knowledge Bases** quick create (managed vector store) to test RAG in hours; **Bedrock Agents** / **AgentCore** + Strands Agents for agent PoCs.
   - **Bedrock Prompt Management** to version prompts during iteration; **Prompt optimisation** to auto-rewrite prompts for a target model.
   - SageMaker JumpStart / Studio notebooks if you need to try open-weight models on your own endpoints.
5. **Evaluate systematically**: **Amazon Bedrock Evaluations**
   - **Automatic model evaluation**: built-in or custom datasets; metrics such as accuracy, robustness, toxicity.
   - **Human evaluation**: your own work team (or AWS-managed) rates helpfulness, style, brand voice.
   - **LLM-as-a-judge**: an evaluator model scores correctness, completeness, faithfulness, harmfulness, etc. Scales close to human-like judgement at lower cost.
   - **RAG evaluation** (Knowledge Base evaluation): retrieval-only (context relevance/coverage) or retrieve-and-generate (correctness, faithfulness, citation quality). Works for Bedrock KBs or your own RAG (bring your own inference responses).
   - Compare **2–3 candidate models** on the **same dataset** with the same prompt (adapted per model if needed).
6. **Measure performance**: log `usage` (input/output tokens) and `metrics.latencyMs` from Converse responses; CloudWatch `InvocationLatency`, `TimeToFirstToken` (where available, verify), `InvocationThrottles`; run a small load test to find the quota ceiling.
7. **Estimate production cost**: tokens/request × requests/month × price (use the AWS Pricing Calculator), plus vector store, KB ingestion (embedding tokens), Lambda/API Gateway, logging. Compare on-demand vs batch vs Provisioned Throughput.
8. **Pilot with real users** (small group) → collect feedback (thumbs/comments) → iterate → **decision gate**.
9. **Document the path to production**: the gaps list (below), target architecture, cost model, risks.

#### 3. PoC → production readiness gap list (frequent exam distractors)
- **Quotas**: request TPM/RPM increases (Service Quotas), add cross-Region inference, backoff with jitter, consider Provisioned Throughput.
- **Security**: least-privilege IAM (resource-level `bedrock:InvokeModel` on specific model/inference-profile ARNs), VPC endpoints, KMS CMKs, secrets in Secrets Manager, Guardrails, input validation, prompt-injection defences.
- **Model access**: production accounts need model access enabled (AWS Marketplace subscription for some third-party models), and possibly SCP allow-lists.
- **Observability**: model invocation logging (off by default, so enable it), dashboards and alarms, tracing, cost allocation tags.
- **Quality gates**: automated evaluation in CI/CD (regression on the golden set before a prompt/model change ships). See Task 1.6.4 and Domain 5.
- **Resilience**: fallback model, circuit breaker, graceful degradation (Task 1.2.3).
- **Data**: production-grade ingestion pipelines, data quality checks (Task 1.3), KB sync schedule (Task 1.4.5).
- **Governance**: prompt approval workflow, model cards / AI service cards, responsible-AI review, audit trails.

#### 4. PoC anti-patterns
- Judging on a handful of hand-picked demo prompts → no statistical confidence.
- Over-engineering (fine-tuning, custom infra) before trying prompting + RAG.
- Ignoring cost/latency until after the decision.
- Using production PII in an unsecured sandbox or in PartyRock.
- Treating the PoC code as production code (no IaC, no tests, no guardrails).
- Locking into one model's request format (`InvokeModel` bodies) rather than **Converse**.

#### 5. Example: quick model comparison script for a PoC
```python
import boto3, time, json
brt = boto3.client("bedrock-runtime")
CANDIDATES = ["<model-or-profile-id-A>", "<model-or-profile-id-B>"]
golden = [json.loads(l) for l in open("golden.jsonl")]   # {"prompt":..., "reference":...}
for model in CANDIDATES:
    for row in golden:
        t0 = time.time()
        r = brt.converse(modelId=model,
                         messages=[{"role": "user", "content": [{"text": row["prompt"]}]}],
                         inferenceConfig={"maxTokens": 400, "temperature": 0})
        print(model, r["usage"]["inputTokens"], r["usage"]["outputTokens"],
              r["metrics"]["latencyMs"], round(time.time() - t0, 2))
        # store output → score later with Bedrock Evaluations (LLM-as-a-judge) or a custom judge
```

---

### Skill 1.1.3 — Standardised technical components (Well-Architected, WA Tool, Generative AI Lens)

> Create standardized technical components to ensure consistent implementation across multiple deployment scenarios (for example, by using the AWS Well-Architected Framework, AWS WA Tool Generative AI Lens).

#### 1. Why standardise
- Many teams building GenAI apps → risk of inconsistent security, duplicated effort, untracked cost, ungoverned prompts and models.
- Standard components give: **consistency** (same guardrails, logging, IAM patterns), **speed** (paved road / golden path), **governance** (central visibility), **portability** across environments (dev/test/prod) and accounts.

#### 2. AWS Well-Architected Framework: 6 pillars applied to GenAI

| Pillar | GenAI-specific focus (Generative AI Lens) | Example AWS implementations |
| --- | --- | --- |
| **Operational excellence** | Consistent model output quality, operational health monitoring, traceability, automated lifecycle management, deciding when to customise | Invocation logging, CloudWatch dashboards, Prompt Management versions, CI/CD with evaluation gates, SageMaker Model Registry |
| **Security** | Protect GenAI endpoints, mitigate harmful output and **excessive agency**, monitor and audit, secure prompts, remediate **model/data poisoning** | IAM least privilege, VPC endpoints, Guardrails, CloudTrail, KMS, agent tool scoping (AgentCore Identity/Gateway), input validation |
| **Reliability** | Handle throughput needs, reliable component communication, observability, graceful failure, **version artifacts**, distribute inference, verify distributed tasks complete | Cross-Region inference, retries + backoff, SQS buffering, Step Functions retries/circuit breakers, versioned prompts/models/KBs |
| **Performance efficiency** | Capture and improve model performance, maintain acceptable performance, optimise compute, **improve data retrieval performance** | Evaluations, model selection, latency-optimised inference, prompt caching, reranking, vector index tuning |
| **Cost optimisation** | **Cost-optimised model selection**, balance inference cost and performance, **engineer prompts for cost**, optimise vector stores and agent workflows | Intelligent prompt routing, batch inference, distillation, maxTokens caps, application inference profiles + tags, S3 Vectors |
| **Sustainability** | Minimise compute for training/customisation/hosting/data processing/storage; model efficiency techniques; serverless | Prefer pre-trained FMs + RAG over training, smaller models, serverless Bedrock, efficient Regions, lifecycle/delete unused data and endpoints |

- Generative AI Lens **lifecycle phases**: **scoping → model selection → model customisation → development & integration → deployment → continuous improvement**.
- **Responsible AI** runs through the whole lens, with shared responsibility between **model producers, providers and consumers**.
- Best practices are named per pillar with prefixes such as **GENOPS**, **GENSEC**, **GENREL**, **GENPERF**, **GENCOST**, **GENSUS** (for example, GENSEC01, verify exact IDs in the lens).
- Scope of the lens: Bedrock, SageMaker AI (customer-managed models) and Amazon Q. For **traditional ML** use the **Machine Learning Lens**. Related lenses: **Responsible AI Lens**, Serverless Lens, SaaS Lens.

#### 3. AWS Well-Architected Tool (WA Tool): how it works
- **Workload**: the thing you review (name, environment, Regions/accounts, architecture link).
- **Lenses**: the **Well-Architected Framework lens** is always applied, plus catalogue lenses (Serverless, SaaS, ML…) and **custom lenses**.
  - The **Generative AI Lens** is provided as **custom lens JSON** in the AWS samples GitHub repo (`sample-well-architected-custom-lens`). You **import** it into the WA Tool, **publish** a version, then apply it to workloads.
  - **Custom lenses** let you encode **your own organisation's standards** (for example, "Is a guardrail attached to every production invocation?") as questions with best-practice choices, risk rules and improvement links. You can version them and **share** them with other accounts or AWS Organizations.
- **Review**: answer the questions → the tool flags **High Risk Issues (HRI)** and **Medium Risk Issues (MRI)**.
- **Improvement plan**: prioritised list of HRIs/MRIs with links to guidance. Track it in the console or export it, and integrate with ticketing (Jira connector / AWS Systems Manager OpsCenter integrations, verify).
- **Milestones**: snapshot the review state at a point in time (for example, "pre-prod", "post-launch") to track progress.
- **Review templates**: pre-filled answers/notes for common architectures → consistent reviews across many workloads.
- **Profiles**: capture business context/priorities so the tool prioritises the questions that matter most.
- **Sharing / Organizations**: share workloads and lenses across accounts; a central team can see risk posture across the organisation.
- **APIs**: the WA Tool API (`CreateWorkload`, `ImportLens`, `AssociateLenses`, `UpdateAnswer`, `CreateMilestone`, `ListLensReviewImprovements`…) lets you automate reviews in pipelines.

```bash
# Import the Generative AI Lens JSON as a custom lens and apply it
aws wellarchitected import-lens --json-string file://generative-ai-lens.json
aws wellarchitected create-lens-version --lens-alias <lens-arn> --lens-version "1.0" --is-major-version
aws wellarchitected associate-lenses --workload-id <id> --lens-aliases <lens-arn>
```

#### 4. Standardised technical components: what to build once and reuse

| Component | Implementation | Benefit |
| --- | --- | --- |
| 🎯 **PQ27** **Infrastructure templates** | **AWS CDK** constructs (including **AWS Generative AI CDK Constructs**: Bedrock KB, agents, guardrails, OpenSearch vector index patterns), CloudFormation modules/StackSets, Terraform modules | Repeatable, reviewed, secure-by-default stacks across accounts/Regions |
| **Self-service catalogue** | **AWS Service Catalog** portfolios/products (for example, "RAG chatbot stack", "batch summarisation pipeline") with constraints and launch roles | Teams deploy approved patterns without broad IAM |
| **Central LLM gateway** | API Gateway + Lambda/ECS proxy: auth, per-team quotas/usage plans, model allow-list, logging, routing/fallback | One control point for all FM traffic |
| **Prompt library** | **Bedrock Prompt Management** (versioned, parameterised templates, variants) + S3/Git repo of templates with approval workflow | Consistent instructions; rollback; audit (Task 1.6) |
| **Guardrail policies** | Central, versioned **Bedrock Guardrails** referenced by ID/version (plus organisation-level guardrail enforcement where available, verify) | Uniform responsible-AI controls |
| **Shared SDK / Lambda layer** | Wrapper around Converse with retries/backoff, tracing, token logging, PII redaction, standard error handling | Avoid each team re-implementing resilience |
| **Configuration** | **AWS AppConfig** / Parameter Store for model IDs, inference params, feature flags | Switch models without code changes (Task 1.2.2) |
| **Data components** | Standard ingestion pipeline (S3 → validation → KB sync), metadata schema, chunking config | Consistent retrieval quality (Tasks 1.3–1.5) |
| **Evaluation harness** | Golden datasets + Bedrock Evaluations jobs in CI/CD | Same quality bar everywhere |
| **Observability baseline** | Invocation logging config, CloudWatch dashboard/alarm templates, X-Ray, cost allocation tags, **application inference profiles** | Comparable metrics and cost per app/team |
| 🎯 **PQ22** **Account guardrails** | AWS Organizations **SCPs** (for example, deny non-approved models or Regions with `bedrock:InvokeModel` conditions), **Control Tower** controls, AWS Config rules | Enforce policy centrally |

> [!IMPORTANT]
> **🎯 Practice question 27 — reusable approved components across business units, Bedrock + SageMaker, consistent security and observability**
>
> - **A standardised AWS CDK construct library** (prompt management, security controls such as IAM, KMS, VPC endpoints and guardrails, and observability), **packaged as a shared library** that every business unit uses in both dev and prod. AWS Generative AI CDK Constructs are a good starting point.
> - ❌ **One account, multi-tenant:** a bigger blast radius, and nothing reusable.
> - ❌ **A custom SageMaker pipeline + endpoint + monitoring per business unit:** fragmented and duplicated.
> - ❌ **A separate CloudFormation template per business unit:** the templates drift apart and nothing is shared.

- **Multi-environment / multi-account**: separate dev/test/prod accounts. Promote the *same* IaC and *versioned* prompts/guardrails/models through pipelines (CodePipeline/GitHub Actions). Parameterise per environment (model ID, throughput type, log retention).
- **Multi-deployment scenarios**: the same component library should work for real-time chat, batch jobs and agents. Design components to be **composable** (retrieval module, generation module, guardrail module).
- **Tagging standard**: `app`, `team`, `env`, `cost-centre`, `data-classification`, applied to inference profiles, KBs, buckets and endpoints.

#### 5. Generative AI Security Scoping Matrix (supporting framework)

| Scope | Description | Example | Your responsibility |
| --- | --- | --- | --- |
| 1 · Consumer app | Use a public third-party GenAI app | Public chatbot | Usage policy, data you paste in |
| 2 · Enterprise app | Use a SaaS app with GenAI features | Amazon Q Business, SaaS with GenAI | Contracts, data access, identity |
| 3 · Pre-trained models | Build an app on a provider's FM via API | **Bedrock** FMs + RAG | App security, prompts, guardrails, data, IAM |
| 4 · Fine-tuned models | Fine-tune an existing FM with your data | Bedrock/SageMaker fine-tuning | + training data governance, model access |
| 5 · Self-trained models | Train from scratch | SageMaker HyperPod | + everything: data, training, model hosting |

- Use it in design reviews to decide which controls (governance, legal, risk, controls, resilience) are needed for a solution.

#### 6. Putting it together: governance loop
1. Platform team publishes **standard components** (CDK constructs, Service Catalog products, guardrails, prompt library, gateway).
2. App teams build from the paved road.
3. Each workload is reviewed in the **WA Tool** with the **Framework + Generative AI Lens + organisation custom lens**.
4. HRIs → improvement plan → fixes → **milestone**.
5. Lessons learned feed back into the standard components (continuous improvement).

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| "Least operational overhead" to add GenAI to an app | **Amazon Bedrock** (serverless) + Converse API |
| Knowledge changes frequently / must cite internal documents | **RAG** with Bedrock Knowledge Bases (not fine-tuning) |
| Consistent brand tone/format; lots of labelled examples | **Fine-tuning** (consider LoRA/PEFT) |
| Big-model quality, need lower latency and cost | **Model distillation** or a smaller model; **intelligent prompt routing** |
| Need to switch between models from different providers with one code path | **Converse API** + model ID in AppConfig/Parameter Store |
| Long-running generation behind API Gateway times out | Use **streaming** (WebSocket / Lambda response streaming) or an **async** pattern (SQS + callback/polling) |
| Summarise millions of records overnight, cheapest | **Bedrock batch inference** (S3 JSONL) |
| Steady, high, predictable volume; custom model | **Provisioned Throughput** |
| Throttling at peaks / model not available in home Region | **Cross-Region inference profile** (+ backoff, quota increase) |
| Data must stay in the EU | **Geographic (EU) inference profile**, not global; Region choice; VPC endpoints; KMS |
| Fine-tuned an open-weight model elsewhere, want serverless Bedrock APIs | **Bedrock Custom Model Import** |
| Need full control of container, custom model, multi-model endpoint | **SageMaker AI endpoints** (inference components) |
| Validate feasibility before full investment | **PoC on Bedrock** with predefined success criteria + golden dataset |
| Compare candidate models objectively at scale | **Bedrock Evaluations**: automatic / **LLM-as-a-judge** / human |
| Evaluate a RAG system's retrieval and grounding | **Bedrock RAG evaluation** (KB evaluation; faithfulness, context relevance) |
| Business stakeholders want to explore ideas without code | **PartyRock** / Bedrock playgrounds |
| Teams need a collaborative builder environment for GenAI apps | **Amazon Bedrock in SageMaker Unified Studio** |
| Review a GenAI workload against AWS best practices | **WA Tool** + **Generative AI Lens** (imported as a custom lens) |
| Encode company-specific GenAI standards as a repeatable review | **WA Tool custom lens**, shared across accounts |
| Track review progress over time | WA Tool **milestones** |
| Consistent, approved GenAI stacks deployed by many teams | **Service Catalog** products / **CDK constructs** (Generative AI CDK Constructs) |
| Central control of model access, quotas and logging for all teams | **LLM gateway** (API Gateway + Lambda) + SCPs |
| Restrict which models can be used org-wide | **SCPs** / IAM conditions on model ARNs |
| Track GenAI cost per application/team | **Application inference profiles** + cost allocation tags |
| Traditional ML (not GenAI) best-practice review | **Machine Learning Lens** |
| Decide how much security responsibility you own | **Generative AI Security Scoping Matrix** (scopes 1–5) |

## Flashcards

<details>
<summary><b>Q:</b> List the customisation ladder from cheapest to most expensive.</summary>

Prompt engineering → RAG → agents/tool use → fine-tuning / distillation → continued pre-training → training from scratch. Always start at the top and move down only when evaluation shows you need to.

*D1 · Task 1.1 · Customisation ladder*
</details>

<details>
<summary><b>Q:</b> Product data changes daily and answers must cite sources. Should you use fine-tuning or RAG?</summary>

RAG (for example, Bedrock Knowledge Bases). Fine-tuning fixes knowledge at training time, so it goes stale and can't give citations. RAG retrieves current documents at query time.

*D1 · Task 1.1 · RAG vs fine-tuning*
</details>

<details>
<summary><b>Q:</b> Which non-functional requirements should you capture before choosing an FM?</summary>

Latency (TTFT vs end-to-end), throughput (TPM/RPM vs quotas), cost per request, availability/resilience, security and compliance (data residency, encryption, private connectivity), responsible-AI controls, operability and sustainability.

*D1 · Task 1.1 · NFRs*
</details>

<details>
<summary><b>Q:</b> Why use the Converse API rather than InvokeModel in a new design?</summary>

Converse gives one consistent request/response format across model providers, with built-in multi-turn messages, system prompts, tool use, guardrails, document/image inputs, token usage and latency metrics. You can switch models by changing the model ID only.

*D1 · Task 1.1 · Converse API*
</details>

<details>
<summary><b>Q:</b> A chat generation takes longer than the API Gateway integration timeout. Give two fixes.</summary>

1) Stream the response (ConverseStream via API Gateway WebSocket, Lambda response streaming, or AppSync). 2) Make it asynchronous: queue the request in SQS, process it with a worker, store the result, and notify or let the client poll.

*D1 · Task 1.1 · Integration patterns*
</details>

<details>
<summary><b>Q:</b> When is Bedrock batch inference the right deployment choice?</summary>

For large offline workloads where nobody is waiting on the result (for example, summarising millions of records overnight). The input/output is JSONL in S3, the cost is lower than on-demand, and you get the job status via EventBridge.

*D1 · Task 1.1 · Batch inference*
</details>

<details>
<summary><b>Q:</b> When should you choose SageMaker AI endpoints over Bedrock?</summary>

When you need full control: custom containers, self-trained or unsupported open-weight models, specific GPU/Inferentia instances, multi-model inference components or LoRA adapter hosting. You accept more operational overhead in return.

*D1 · Task 1.1 · Deployment strategy*
</details>

<details>
<summary><b>Q:</b> You fine-tuned an open-weight Llama model on SageMaker and want to serve it serverlessly through Bedrock APIs. What feature do you use?</summary>

Amazon Bedrock Custom Model Import (for supported architectures).

*D1 · Task 1.1 · Custom Model Import*
</details>

<details>
<summary><b>Q:</b> What must be defined before a GenAI PoC starts?</summary>

Measurable success criteria agreed with stakeholders: quality (for example, judged correctness or faithfulness on a golden set), latency (p95 TTFT/E2E), cost per transaction, safety thresholds and the business KPI. Plus scope and a time-box.

*D1 · Task 1.1 · PoC success criteria*
</details>

<details>
<summary><b>Q:</b> What evaluation options does Amazon Bedrock offer for a PoC?</summary>

Automatic model evaluation (built-in/custom datasets and metrics), human evaluation (your own or an AWS-managed work team), LLM-as-a-judge (evaluator model scores quality and responsible-AI metrics), and RAG/Knowledge Base evaluation (retrieval and retrieve-and-generate).

*D1 · Task 1.1 · Bedrock Evaluations*
</details>

<details>
<summary><b>Q:</b> Name five PoC-to-production gaps that are often missed.</summary>

Service quotas/throttling, least-privilege IAM and VPC endpoints, guardrails, model invocation logging and monitoring, cost allocation, automated evaluation in CI/CD, fallback/resilience, production data pipelines, prompt/model governance.

*D1 · Task 1.1 · Production readiness*
</details>

<details>
<summary><b>Q:</b> What is PartyRock, and when should you not use it?</summary>

A no-code Bedrock-powered playground for exploring GenAI app ideas. Don't use it for production workloads or with confidential/PII data.

*D1 · Task 1.1 · Prototyping tools*
</details>

<details>
<summary><b>Q:</b> What are the six pillars covered by the Well-Architected Generative AI Lens?</summary>

Operational excellence, security, reliability, performance efficiency, cost optimisation and sustainability.

*D1 · Task 1.1 · Generative AI Lens*
</details>

<details>
<summary><b>Q:</b> Which lifecycle phases does the Generative AI Lens cover?</summary>

Scoping → model selection → model customisation → development & integration → deployment → continuous improvement.

*D1 · Task 1.1 · Generative AI Lens lifecycle*
</details>

<details>
<summary><b>Q:</b> How do you start a Generative AI Lens review in the AWS WA Tool?</summary>

Download the Generative AI Lens JSON from the AWS samples custom-lens GitHub repo, import it as a custom lens, publish a version, associate it with the workload and answer the questions. Then work through the HRIs/MRIs in the improvement plan.

*D1 · Task 1.1 · WA Tool*
</details>

<details>
<summary><b>Q:</b> What are WA Tool milestones, review templates and custom lenses for?</summary>

Milestones snapshot a review to track progress over time. Review templates pre-fill answers for consistent reviews across many workloads. Custom lenses encode your organisation's own best-practice questions and can be versioned and shared.

*D1 · Task 1.1 · WA Tool features*
</details>

<details>
<summary><b>Q:</b> Which Security pillar risks does the Generative AI Lens call out specifically?</summary>

Protecting GenAI endpoints, harmful outputs, excessive agency (agents with too much power), monitoring/auditing events, securing prompts, and model/data poisoning.

*D1 · Task 1.1 · Generative AI Lens security*
</details>

<details>
<summary><b>Q:</b> Give four standardised components a platform team might provide to GenAI app teams.</summary>

CDK constructs / Service Catalog products for approved stacks, a central LLM gateway, a Prompt Management template library, central versioned Guardrails, a shared SDK/Lambda layer with retries and logging, an AppConfig model configuration, an evaluation harness, and an observability baseline.

*D1 · Task 1.1 · Standard components*
</details>

<details>
<summary><b>Q:</b> How can you allocate Bedrock on-demand costs to individual applications or teams?</summary>

Create application inference profiles, tag them (app/team/cost-centre), invoke models through the profile ARN, and activate the tags as cost allocation tags in Billing/Cost Explorer.

*D1 · Task 1.1 · Cost allocation*
</details>

<details>
<summary><b>Q:</b> How can an organisation stop teams from using unapproved foundation models?</summary>

Use AWS Organizations SCPs or IAM policies that allow `bedrock:InvokeModel*` only on approved model/inference-profile ARNs (and approved Regions). Combine with a central gateway and model-access controls.

*D1 · Task 1.1 · Model governance*
</details>

<details>
<summary><b>Q:</b> In the Generative AI Security Scoping Matrix, which scope is an app built on Bedrock FMs, and which is a fine-tuned model?</summary>

Scope 3 = pre-trained models (for example, Bedrock FMs via API). Scope 4 = fine-tuned models. Scope 5 = self-trained models. Responsibility grows with scope.

*D1 · Task 1.1 · Security Scoping Matrix*
</details>

<details>
<summary><b>Q:</b> Interactive chat vs nightly report generation: which latency metric matters for each?</summary>

Interactive chat: time to first token (TTFT), so stream the output. Nightly/batch report: end-to-end latency (or just throughput and cost).

*D1 · Task 1.1 · Latency metrics*
</details>

<details>
<summary><b>Q:</b> The requirement is "data must not leave the EU" and you need more burst capacity. Which inference option should you use?</summary>

An EU geographic cross-Region inference profile (`eu.` prefix). It routes only within EU Regions. Avoid global inference profiles.

*D1 · Task 1.1 · Data residency*
</details>

<details>
<summary><b>Q:</b> Which Well-Architected lens should you use for a traditional ML (non-GenAI) model on SageMaker?</summary>

The Machine Learning Lens. The Generative AI Lens is for FM-based workloads (Bedrock, SageMaker-hosted FMs, Amazon Q).

*D1 · Task 1.1 · Lens selection*
</details>

---
[← Back to Domain 1](README.md)
