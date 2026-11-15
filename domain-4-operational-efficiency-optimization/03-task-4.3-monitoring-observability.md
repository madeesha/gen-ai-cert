[← Back to Domain 4](README.md)

# 📊 Task 4.3 · Implement monitoring systems for GenAI applications

> **Domain 4 · Operational Efficiency and Optimization for GenAI Applications (12%)**
> - Skill 4.3.1: Holistic observability (operational metrics, performance tracing, FM interaction tracing, business-impact dashboards)
> - Skill 4.3.2: GenAI monitoring and KPIs (token usage, prompt effectiveness, hallucination rate, response quality, anomaly detection for token bursts and response drift, invocation logs, benchmarks, cost anomalies)
> - Skill 4.3.3: Integrated observability with actionable insights (operational and business dashboards, compliance monitoring, forensic traceability, user interaction and model-behaviour tracking)
> - Skill 4.3.4: Tool performance frameworks (call patterns, per-tool metrics, tool-calling and multi-agent coordination tracing, usage baselines)
> - Skill 4.3.5: Vector store operational management (vector DB performance monitoring, automated index optimisation, data quality validation)
> - Skill 4.3.6: FM-specific troubleshooting frameworks (golden datasets, output diffing, reasoning-path tracing, specialised observability pipelines)

## Key points
- **Three layers of GenAI telemetry:**
  - **Metrics** (`AWS/Bedrock`: `Invocations`, `InvocationLatency`, `TimeToFirstToken`, `InputTokenCount`, `OutputTokenCount`, `InvocationThrottles`, `InvocationClientErrors`, `InvocationServerErrors`, `EstimatedTPMQuotaUsage`, `CacheReadInputTokenCount`/`CacheWriteInputTokenCount`)
  - **Logs** (model invocation logging gives full prompts and responses; it's **off by default**)
  - **Traces** (X-Ray / OpenTelemetry → CloudWatch **Transaction Search** → **GenAI observability** dashboard)
- **Throttled requests are not counted** in `Invocations` or the error metrics. Requests rejected for exceeding a quota count as **client errors**, not throttles. Metrics are published both **with and without** the `ModelId` dimension, so filtering by `ModelId` can **undercount errors**.
- `TimeToFirstToken` only exists for **streaming** APIs (`ConverseStream`, `InvokeModelWithResponseStream`). **OTPS = OutputTokenCount / (InvocationLatency − TimeToFirstToken) × 1000**. Alarm on **OTPS dropping**, not on raw latency, to tell model-side slowdown apart from longer answers.
- **AgentCore Observability:** OTEL-compatible spans, traces and sessions plus built-in metrics in the `bedrock-agentcore` namespace. Runtime-hosted agents are **auto-instrumented**. Agents hosted elsewhere need **ADOT** (`aws-opentelemetry-distro` + `opentelemetry-instrument`, or the Lambda OTel layer). You must enable **Transaction Search** once per account.
- **Guardrails metrics** are in `AWS/Bedrock/Guardrails`: `InvocationsIntervened` and `TextUnitCount` by `GuardrailPolicyType` and `GuardrailContentSource` (Input/Output). Use these for a **block-rate** KPI and policy-violation trends.
- 🎯 **PQ17** **Quality KPIs are custom metrics:** hallucination rate, groundedness, thumbs up/down, refusal rate. Emit them with **EMF** from Lambda or with `PutMetricData`, then use **anomaly detection** (needs about 2 weeks of history) for drift and token bursts.
- **Cost:** **Cost Anomaly Detection** (ML, per service, cost category or tag) + **AWS Budgets** (threshold and forecast alerts, budget actions). Use **application inference profiles + tags** for per-team cost allocation. Token metrics are the leading indicator; the bill lags behind.
- **Forensics:** CloudTrail (who called which model, when, from where) + invocation logs (what was said) + traces (what happened) + **immutable storage** (S3 Object Lock, CloudTrail log file validation, KMS).
- **Tool observability:** track per-tool call count, latency, error rate and argument-validation failures from agent traces or spans. Set **usage baselines** and alert on deviations such as tool loops or a sudden spike in one tool.
- **Vector stores:** for **OpenSearch Serverless** (`AWS/AOSS`) watch `SearchRequestLatency`, `SearchRequestErrors`, `SearchOCU`, `IndexingOCU`, `IngestionDocumentErrors`, `SearchableDocuments` and `DeletedDocuments`. For **managed OpenSearch**, watch k-NN graph memory and the circuit breaker. For **KB ingestion jobs**, watch scanned, indexed and failed document counts.
- **GenAI-specific failure modes** are probabilistic: hallucination, drift, inconsistency and reasoning errors. Detect them with **golden datasets**, **output diffing** (semantic similarity between runs or versions), **reasoning-path tracing** (agent traces, extended thinking), and a **Firehose → S3 → Athena/QuickSight** analytics pipeline.

## Notes

### Skill 4.3.1 — Holistic observability systems

> Create holistic observability systems to provide complete visibility into FM application performance (for example, by using operational metrics, performance tracing, FM interaction tracing, business impact metrics with custom dashboards).

#### Why GenAI observability is different
| Traditional app / ML | GenAI application |
| --- | --- |
| Latency + error rate + saturation | Also **TTFT**, **tokens/sec**, **tokens per request**, **cost per request** |
| Deterministic output → assert equality | **Non-deterministic** output → quality must be *scored* (groundedness, relevance, toxicity) |
| Model drift = feature distribution drift | Also **prompt drift** (users ask new things), **response drift** (model version changes, prompt edits), **retrieval drift** (stale KB) |
| One inference = one call | One request = **many spans** (retrieve → rerank → guardrail → LLM → tool → LLM …) |
| Cost ≈ compute hours | Cost ≈ **tokens × price** that varies per model, tier and cache hit |

#### The four observability pillars for FM apps
1. **Operational metrics** (CloudWatch): volume, latency, errors, throttles and tokens per model. These are free and built in.
2. **Performance tracing** (X-Ray / OTel spans): where the time goes across API Gateway → Lambda → KB `Retrieve` → `Converse` → tools.
3. **FM interaction tracing**: the *content* of the interaction (prompt, retrieved chunks, model output, tool calls, guardrail decisions). Sources are model invocation logs, agent traces (`enableTrace=true`) and GenAI spans.
4. **Business impact metrics**: deflection rate, resolution rate, CSAT, conversion, time saved and **cost per resolved conversation**. These are custom metrics on custom dashboards.

#### AWS building blocks
| Need | Service / feature |
| --- | --- |
| Built-in FM metrics | CloudWatch `AWS/Bedrock` namespace (runtime), `AWS/Bedrock/Guardrails`, agents metrics (verify in docs for the exact agents namespace) |
| Full request/response capture | **Model invocation logging** → CloudWatch Logs and/or S3 (setup: [Task 2.5](../domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md)) |
| Distributed tracing | **AWS X-Ray** / **ADOT** → CloudWatch **Transaction Search** (spans stored in the `aws/spans` log group) |
| Agent tracing | **AgentCore Observability** (sessions, traces, spans, `bedrock-agentcore` metrics); Bedrock Agents `trace` events |
| Curated GenAI view | CloudWatch **Generative AI observability** page (Overview, Sessions, Traces, Spans tabs) — covers model invocations and agents on AgentCore Runtime *or elsewhere* |
| App-level SLOs | CloudWatch **Application Signals** (service maps, SLOs) |
| Synthetic probes | CloudWatch **Synthetics** canaries that call the chat API with a fixed prompt |
| Real user monitoring | CloudWatch **RUM** for front-end latency perceived by users |
| Business dashboards | CloudWatch dashboards (ops) · **Amazon QuickSight** (business, built on Athena over logs) · Amazon Managed Grafana |

#### Instrumenting an FM call with OpenTelemetry (GenAI semantic conventions)
The OTel GenAI semantic conventions define standard span attributes, so every backend can understand the spans:

```python
from opentelemetry import trace
tracer = trace.get_tracer("chat-service")

with tracer.start_as_current_span("chat claude-sonnet") as span:
    span.set_attribute("gen_ai.system", "aws.bedrock")
    span.set_attribute("gen_ai.request.model", model_id)
    span.set_attribute("gen_ai.request.max_tokens", 1024)
    span.set_attribute("gen_ai.request.temperature", 0.2)
    resp = bedrock.converse(modelId=model_id, messages=msgs,
                            inferenceConfig={"maxTokens": 1024, "temperature": 0.2})
    usage = resp["usage"]
    span.set_attribute("gen_ai.usage.input_tokens", usage["inputTokens"])
    span.set_attribute("gen_ai.usage.output_tokens", usage["outputTokens"])
    span.set_attribute("gen_ai.response.finish_reasons", [resp["stopReason"]])
    span.set_attribute("app.tenant_id", tenant_id)      # custom business dimension
```
- ADOT auto-instrumentation (`opentelemetry-instrument python app.py`) already instruments **botocore** Bedrock calls, Strands and many frameworks. Add custom attributes such as tenant, feature and prompt version for business slicing.
- ⚠️ Filter sensitive data out of span attributes. Prompt text in spans is effectively a second copy of your logs, with different retention and access rules.

#### Dashboard design (layered)
| Layer | Audience | Widgets |
| --- | --- | --- |
| **Executive / business** | Product owners | Conversations/day, deflection %, CSAT, cost per conversation, cost trend vs budget |
| **Service health** | On-call | p50/p90/p99 `InvocationLatency`, `TimeToFirstToken`, OTPS, throttles, 4xx/5xx, guardrail intervention rate |
| **Model & token** | Platform / FinOps | Input vs output tokens per model, cache read/write tokens, `EstimatedTPMQuotaUsage` vs quota, tokens per tenant |
| **Quality** | ML / app team | Groundedness score, hallucination rate, thumbs-down %, refusal rate, eval-suite pass rate |
| **Dependencies** | On-call | KB retrieve latency, OpenSearch `SearchRequestLatency`, tool error rates |

- Use **cross-account observability** (CloudWatch monitoring account linked to source accounts through OAM) when GenAI workloads span many accounts. AgentCore also supports cross-account monitoring.

---

### Skill 4.3.2 — Comprehensive GenAI monitoring and KPIs

> Implement comprehensive GenAI monitoring systems to proactively identify issues and evaluate key performance indicators specific to FM implementations (for example, by using CloudWatch to track token usage; prompt effectiveness; hallucination rates; and response quality, anomaly detection for token burst patterns and response drift, Amazon Bedrock Model Invocation Logs to perform detailed request and response analysis, performance benchmarks, cost anomaly detection).

#### Amazon Bedrock runtime metrics (`AWS/Bedrock`), verified
| Metric | Unit | Meaning & gotchas |
| --- | --- | --- |
| `Invocations` | Count | Requests accepted for processing (Converse, ConverseStream, InvokeModel, InvokeModelWithResponseStream, and the OpenAI-compatible APIs). **Includes** requests that later fail. **Excludes** throttled requests and requests rejected at input validation. |
| `InvocationLatency` | ms | From the request being sent to the **last** token being received |
| `TimeToFirstToken` | ms | **Streaming APIs only.** The key UX metric for chat. |
| `InvocationClientErrors` | Count | 4xx-type errors. **Includes quota-exceeded rejections**, which are *not* counted as throttles. |
| `InvocationServerErrors` | Count | 5xx-type AWS-side errors |
| `InvocationThrottles` | Count | Throttled requests. Not counted in Invocations or the error metrics. The value **depends on SDK retry settings**, because retries hide or reveal throttles. |
| `InputTokenCount` / `OutputTokenCount` | Count | Token usage. Output tokens usually cost more per token. |
| `EstimatedTPMQuotaUsage` | Count | **Approximate** TPM quota consumption. Throttling is based on an upfront reservation of **input tokens + `max_tokens`**, so don't use this metric alone for capacity planning. |
| `CacheReadInputTokenCount` | Count | Tokens served from the prompt cache. Charged at a reduced rate and **don't count toward TPM**. |
| `CacheWriteInputTokenCount` | Count | Tokens written to the prompt cache. These **count toward TPM**. |
| `OutputImageCount` | Count | Image generation models only |
| `LegacyModelInvocations` | Count | Calls to models in **Legacy** lifecycle status. Alarm on this to catch deprecation risk. |
| `ModelInvocationLogs{CloudWatch,S3}Delivery{Success,Failure}`, `ModelInvocationLargeDataS3Delivery*` | Count | Health of invocation-log delivery. Alarm on failures, because missing logs are a compliance gap. |

**Dimensions:** `ModelId` (all metrics), `ModelId + ContextWindow` (value `1M`, only for requests billed at the long-context rate; a *subset* of the `ModelId` series), and `ModelId + ImageSize + BucketedStepSize` for images.

⚠️ **Gotcha:** if a request fails **before Bedrock resolves the model**, its metrics are published *without* `ModelId`. To count **all** errors, query the metric **with no dimension**.

#### Diagnosing latency with OTPS (output tokens per second)
- `InvocationLatency = TimeToFirstToken + (OutputTokenCount / OTPS) × 1000`
- **Prefill** (processing the whole prompt) drives TTFT and scales with input length. **Decode** (one token per forward pass) scales with output length.
- **Latency up + OTPS stable** → the workload changed: longer prompts or answers, a new system prompt, or a model update that writes more. Fix with `maxTokens`, prompt changes or caching.
- **Latency up + OTPS down** → model-side throughput degraded. Escalate, switch to CRIS, Priority tier or a different model.

```text
# CloudWatch metric math (p50, 5-minute period, same ModelId)
m1 = InvocationLatency   m2 = OutputTokenCount   m3 = TimeToFirstToken
OTPS = m2 / (m1 - m3) * 1000
Alarm: OTPS < 80% of baseline, 3 of 5 datapoints   (or anomaly band "Lower than", 2–3 std dev)
```

#### GenAI KPIs and how to get them
| KPI | Source | How |
| --- | --- | --- |
| Token usage per model / tenant / feature | `AWS/Bedrock` token metrics; `usage` block in each response | Built-in per model. For per-tenant figures, emit EMF with a `Tenant` dimension, or use application inference profiles. |
| Cost per request / conversation | Tokens × price table | Lambda computes it from `usage` and emits an EMF metric. Reconcile with CUR. |
| Prompt effectiveness | Prompt version (Prompt Management ARN/version) + outcome | Tag each call with `prompt_version`. Compare thumbs-up rate, task success and tokens per version. |
| Hallucination rate | Guardrail contextual grounding results, LLM-as-a-judge on a sample, citation-coverage checks | Async evaluator (Lambda/Step Functions) scores a sample of the logs → `PutMetricData` |
| Groundedness / relevance | Guardrails **contextual grounding** filter (grounding and relevance scores, thresholds 0–0.99) | Track `InvocationsIntervened` where `GuardrailPolicyType=ContextualGroundingPolicy` |
| Guardrail block rate | `AWS/Bedrock/Guardrails` → `InvocationsIntervened / Invocations` | Split by `GuardrailContentSource` (Input = attacks or bad requests, Output = model misbehaviour) |
| Response quality (user) | Thumbs up/down, ratings, escalations to a human | Front end → API → EMF metric `Feedback` with dimension `Rating` |
| Refusal rate | Response text classifier or `stopReason` | Pattern match "I can't help" or use an LLM classifier on a sample |
| Response length drift | `OutputTokenCount` average per request | Anomaly detection on `OutputTokenCount / Invocations` |

#### Emitting a custom quality metric with EMF
EMF (Embedded Metric Format) is a structured log line that CloudWatch turns into a metric automatically. There's no `PutMetricData` call, so there's no extra API cost or throttling.

```json
{
  "_aws": {
    "Timestamp": 1763200000000,
    "CloudWatchMetrics": [{
      "Namespace": "GenAI/Quality",
      "Dimensions": [["Application", "PromptVersion"]],
      "Metrics": [
        {"Name": "Hallucinated", "Unit": "Count"},
        {"Name": "GroundingScore", "Unit": "None"}
      ]
    }]
  },
  "Application": "support-bot",
  "PromptVersion": "7",
  "Hallucinated": 1,
  "GroundingScore": 0.42,
  "requestId": "c0ffee-1234"
}
```
- Hallucination **rate** = `SUM(Hallucinated) / SampleCount` in metric math.
- ⚠️ High-cardinality dimensions such as `userId` or `requestId` create a metric per value, which gets expensive. Keep those as **log fields** (queryable in Logs Insights), not dimensions.

#### Anomaly detection for token bursts and drift 🎯 PQ17
- **CloudWatch anomaly detection** builds an expected band from history (it accounts for hourly and daily seasonality). Give it about **2 weeks** of data. For new workloads, start with static thresholds.
- **Token bursts:** anomaly alarm on `InputTokenCount` Sum (period 5 min), "Greater than the band". Typical causes are prompt-injection floods, a runaway agent loop, a bad deploy that stuffs the context, or abuse by a single API key.
- **Response drift:** anomaly alarms on average output tokens, refusal rate, grounding score and thumbs-down rate. A step change right after a model or prompt version change is drift. Tie it back to the deploy.
- **Semantic drift** (meaning changes, not just length) needs embeddings. Embed a daily sample of responses (or answers to fixed golden prompts) and track the **centroid cosine distance** against a baseline. Emit it as a custom metric.

> [!WARNING]
> **🎯 Practice question 17 — EC2 app calling Bedrock: detect drift from baselines, alert with correlation data within 10 min**
>
> - **CloudWatch Application Insights** sets up monitoring for the application's resources (EC2 and the services around it). It **detects anomalies** and **correlates metric anomalies with log errors**, creating "problems" with the related evidence. That's the **correlation data** the question asks for.
> - **Custom metrics through the CloudWatch embedded metric format (EMF)**: recommendation quality or relevance score, token usage and latency, with **dimensions** such as request type and user segment, to see *which* users get bad recommendations.
> - **CloudWatch anomaly detection** on those metrics → learns **dynamic baselines** and alarms when they're breached.
> - **CloudWatch Logs Insights** → log pattern analysis.
> - ❌ **Container Insights:** for **ECS/EKS/Kubernetes containers**, not EC2 applications, and it has no baseline anomaly detection.
> - ❌ **OpenSearch Observability + Kinesis + PPL:** a pipeline you'd have to build and maintain.
> - ❌ **Logs + CloudTrail + QuickSight:** CloudTrail doesn't capture token or latency metrics, and QuickSight is BI, not real-time alerting.

```bash
aws cloudwatch put-anomaly-detector --namespace AWS/Bedrock --metric-name InputTokenCount \
  --dimensions Name=ModelId,Value=anthropic.claude-sonnet-4-5 --stat Sum
# then an alarm with ThresholdMetricId pointing to ANOMALY_DETECTION_BAND(m1, 3)
```

#### Model invocation logs for detailed analysis
- Captures input body, output body, model ID, token counts, identity (caller ARN) and request ID. Goes to **CloudWatch Logs** and/or **S3**. Large or binary payloads go to S3. **Off by default**, and it is **per account per Region**. Setup and schema are in [Task 2.5](../domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md).
- Typical Logs Insights analyses:

```sql
-- Top callers by output tokens (who is driving cost?)
fields identity.arn, output.outputTokenCount
| stats sum(output.outputTokenCount) as outTok, count(*) as calls by identity.arn
| sort outTok desc | limit 20

-- Requests that hit the max_tokens limit (possible truncation)
fields @timestamp, modelId, output.outputBodyJson.stopReason as stop
| filter stop = "max_tokens"
| stats count(*) by modelId, bin(1h)

-- p95 input size per model (context bloat detection)
stats pct(input.inputTokenCount, 95) as p95In by modelId, bin(1d)
```
(Field paths differ slightly between InvokeModel and Converse bodies; check against a real log line.)

#### Performance benchmarks
- Keep a **baseline** of TTFT, OTPS, p95 latency and cost per request for each model and prompt version, measured with a fixed benchmark prompt set. Rerun it on every model or prompt change and alarm on regressions.
- Synthetic canaries (CloudWatch Synthetics) call the endpoint every N minutes with a fixed prompt. They catch outages and latency regressions even when there's no user traffic.

#### Cost anomaly detection
| Tool | Use |
| --- | --- |
| **AWS Cost Anomaly Detection** | ML-based monitors per **AWS service** (for example Bedrock), **linked account**, **cost category** or **cost allocation tag**. Sends alerts by SNS or email (immediate or daily/weekly digest), with root-cause hints. |
| **AWS Budgets** | Fixed or forecast threshold alerts. **Budget actions** can apply an IAM policy or SCP, or stop resources, when a threshold is crossed. |
| **Cost Explorer / CUR 2.0 + Athena** | Per-model, per-tag drill-down. Bedrock line items show input and output token usage types. |
| **Application inference profiles** | Tag a profile per team or app. Costs show up per tag, and the profile also gives per-profile CloudWatch metrics. |
| **Token metrics** | A **real-time** leading indicator: alarm on tokens per hour versus budget well before the bill shows it |

---

### Skill 4.3.3 — Integrated observability with actionable insights

> Develop integrated observability solutions to provide actionable insights for FM applications (for example, by using operational metric dashboards, business impact visualizations, compliance monitoring, forensic traceability and audit logging, user interaction tracking, model behavior pattern tracking).

#### From data to action
Observability is only *actionable* when every signal has an owner and a response:

| Signal | Alarm | Automated action |
| --- | --- | --- |
| Throttles > threshold | `InvocationThrottles` Sum | SNS → on-call; Lambda flips the AppConfig flag to the fallback model or CRIS profile |
| OTPS degradation | OTPS metric-math alarm | Route to another Region or model; open a support case |
| Guardrail input block spike | `InvocationsIntervened` (Input) anomaly | Security Hub finding; WAF rate-based rule on the offending IP or key |
| Invocation log delivery failure | `ModelInvocationLogs*DeliveryFailure` > 0 | Page compliance on-call (audit gap) |
| Legacy model usage | `LegacyModelInvocations` > 0 | Ticket to migrate before end of life |
| Cost anomaly | Cost Anomaly Detection | SNS → FinOps; Budget action to restrict the IAM policy |
| Quality drop | Thumbs-down rate anomaly | Roll back the prompt version (Prompt Management alias/version); open an investigation |

- **EventBridge** is the glue: CloudWatch alarm state changes → EventBridge rule → Lambda, Step Functions or SSM Automation runbooks.
- **CloudWatch investigations** (AI-assisted operational investigations, verify in docs for current naming) can suggest root-cause hypotheses from metrics, logs and traces.

#### Business impact visualisations
- **Ops** dashboards in CloudWatch. **Business** dashboards in **QuickSight** on top of Athena over S3 (invocation logs, feedback events, business outcomes joined by `conversationId`).
- Typical business views: cost per resolved ticket by channel, deflection trend after a prompt change, adoption by department, ROI (hours saved × rate − FM cost).
- **Amazon Q in QuickSight** lets stakeholders ask natural-language questions of these datasets.

#### Compliance monitoring
- **AWS Config** rules and conformance packs: invocation logging enabled, KMS encryption on log groups and S3, VPC endpoints present, no public buckets for training or KB data.
- **CloudTrail**: Bedrock control-plane API calls (CreateGuardrail, CreateKnowledgeBase, PutModelInvocationLoggingConfiguration…) are recorded. Runtime invocations appear as events too (management vs data event classification for some runtime APIs: verify in docs). An alert on `DeleteModelInvocationLoggingConfiguration` or guardrail deletion catches tampering.
- **Security Hub** aggregates findings. **Audit Manager** maps evidence to frameworks (governance depth is in [Task 3.3](../domain-3-ai-safety-security-governance/03-task-3.3-governance-compliance.md)).

#### Forensic traceability and audit logging
To reconstruct "what did the model tell customer X on date Y, and why?", you need:
1. **Correlation ID** propagated end to end: API Gateway request ID → Lambda → Bedrock `requestId` → trace ID → conversation ID in DynamoDB.
2. **Who:** CloudTrail (`userIdentity`, source IP, role session). Use **trusted identity propagation** or session tags so the *end user*, not just the app role, is recorded.
3. **What:** invocation logs (prompt and response), retrieved chunks with source URIs (KB citations), guardrail assessments (trace).
4. **Why:** agent traces (rationale, tool calls), prompt version, model ID and version, inference parameters.
5. **Integrity:** S3 **Object Lock** (compliance mode) for log archives, CloudTrail **log file integrity validation**, KMS CMKs, separate log-archive account, restrictive bucket policies.
6. **Retention:** CloudWatch Logs retention policies (default is *never expire*, so set them explicitly), S3 Lifecycle to Glacier for long-term retention.

#### User interaction tracking
- Track session length, turns per conversation, abandonment, rephrasing ("user asked the same thing again" is a strong implicit signal of a bad answer), copy or click-through on citations, and escalation to a human.
- Store conversation events in DynamoDB or S3 and stream them with **Kinesis Data Firehose** to S3 for analytics.
- **CloudWatch RUM** captures front-end performance: page load and perceived streaming latency.
- ⚠️ This is user data, so apply PII redaction (Comprehend or Guardrails sensitive-info masking) before analytics storage. See [Task 3.2](../domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md).

#### Model behaviour pattern tracking
- Distribution of `stopReason` (`end_turn`, `max_tokens`, `tool_use`, `guardrail_intervened`, `content_filtered`). A rising `max_tokens` share means truncation.
- Output length distribution, refusal rate, language mix, tool-use frequency, citation count per answer.
- Compare behaviour **per model version and per prompt version**. Behaviour changes right after a deploy are the most common root cause.

---

### Skill 4.3.4 — Tool performance frameworks 🎯 PQ39

> Create tool performance frameworks to ensure optimal tool operation and utilization for FMs (for example, by using call pattern tracking, performance metric collection, tool calling observability and multi-agent coordination tracking, usage baselines for anomaly detection).

> [!IMPORTANT]
> **🎯 Practice question 39 — token surges with steady traffic: find which tool integration causes it, thresholds adapt automatically**
>
> - **Model invocation logging → CloudWatch Logs.**
> - **CloudWatch metric filters** on the log JSON pull out **per-tool metrics** (for example `toolUse.name` with input and output token counts) as **custom metrics with a tool dimension**.
> - **CloudWatch anomaly detection alarms** on each tool's metric → the **ML baseline adapts** to hourly, daily and weekly patterns with no manual threshold changes.
> - ❌ **S3 + Lambda + manually updated alarm thresholds:** not adaptive.
> - ❌ **Glue + scheduled Athena reports:** not real time, and thresholds don't adapt.
> - ❌ **Static alarms per tool:** fixed thresholds that don't follow traffic changes.

#### What to measure per tool
| Metric | Why |
| --- | --- |
| **Call count** (per tool, per agent, per session) | Utilisation. A tool never used means its description may be poor. A tool always used may be redundant. |
| **Latency** p50/p95 | Tool latency often dominates agent latency more than the LLM does |
| **Error rate** (exceptions, timeouts, 4xx/5xx from the backend) | Reliability. Errors feed back into the LLM and cause retries or loops. |
| **Argument validation failures** | The model sends bad parameters: a schema or description problem, or a hallucinated parameter |
| **Calls per task / per session** | Efficiency. A rising count means looping or poor planning. |
| **Selection accuracy** (right tool chosen) | Measured offline with labelled scenarios (Domain 5 covers agent evals) |
| **Tool result size (tokens)** | Large tool outputs inflate context and cost. Truncate or summarise them. |

#### Where the data comes from
- **AgentCore Observability:** spans for each model call and **tool execution** (auto-instrumented on Runtime, and via ADOT for Strands and LangChain elsewhere). Built-in metrics for runtime, **Gateway** (tool invocations, latency, errors through the MCP gateway) and Memory resources. AgentCore Gateway also logs API activity to **CloudTrail**, including data events.
- **Bedrock Agents:** `InvokeAgent` with `enableTrace=true` returns **orchestration traces** (rationale, `invocationInput` for action groups and KBs, `observation` with the tool output) plus pre/post-processing, guardrail and failure traces. Persist them to CloudWatch Logs for analysis.
- **Lambda-backed tools:** standard Lambda metrics (`Duration`, `Errors`, `Throttles`, `ConcurrentExecutions`) plus EMF custom metrics for business-level tool outcomes.
- **MCP servers on ECS/Lambda:** instrument with OTel. Add attributes `tool.name`, `mcp.server`, `agent.name`, `session.id`.

```sql
-- Logs Insights over exported agent spans/traces: per-tool reliability
fields attributes.tool.name as tool, durationNano/1e6 as ms, status.code as status
| filter ispresent(tool)
| stats count(*) as calls, avg(ms) as avgMs, pct(ms,95) as p95Ms,
        sum(status = "ERROR") as errors by tool
| sort calls desc
```
(Attribute names depend on your instrumentation; Strands/OTel GenAI conventions use `gen_ai.tool.name`, so verify against your spans.)

#### Multi-agent coordination tracking
- One **trace ID** across supervisor and sub-agents, with each agent as a child span. Set `session.id` through **OTel baggage** so all agent runs in a conversation correlate (`baggage.set_baggage("session.id", session_id)`).
- Track **handoff count**, **delegation depth**, which sub-agent handled what, time spent per agent, and **cycles** (A → B → A ping-pong).
- Strands **Graph/Swarm** and Bedrock **multi-agent collaboration** both emit per-agent spans. Visualise them as a trajectory or timeline in the GenAI observability **Traces** tab.

#### Baselines and anomaly detection
- Build per-tool baselines for calls per hour, error rate and p95 latency. Use CloudWatch anomaly detection, or static thresholds from load tests.
- Alert patterns:
  - **Loop detection**: the same tool with the same arguments more than N times in a session. Enforce a cap in code (max iterations) and alert on cap hits.
  - **Sudden new tool usage**, such as a destructive tool called in production for the first time. This may be prompt injection. Send it to security.
  - **Error-rate spike for one tool**: usually a backend outage. Open the circuit breaker so the agent degrades gracefully.
- Safeguard mechanics (circuit breakers, timeouts, stopping conditions) are in [Task 2.1](../domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md).

---

### Skill 4.3.5 — Vector store operational management

> Create vector store operational management systems to ensure optimal vector store operation and reliability for FM augmentation (for example, by using performance monitoring for vector databases, automated index optimization routines, data quality validation processes).

#### OpenSearch Serverless metrics (`AWS/AOSS`), verified
| Metric | Watch for |
| --- | --- |
| `SearchRequestLatency` (avg/min/max ms) | Retrieval latency SLO; rising latency → OCU scaling, index bloat |
| `SearchRequestErrors`, `SearchRequestRate` | Errors per minute; traffic pattern |
| `SearchOCU`, `IndexingOCU` | Capacity and **cost** (account or collection-group level). Compare to your **max OCU** cap. |
| `IngestionDocumentErrors`, `IngestionRequestErrors` | Failed embeddings or documents at write time |
| `IngestionRequestLatency`, `IngestionDocumentRate`, `IngestionDataRate` | Bulk-load throughput during KB syncs |
| `SearchableDocuments`, `DeletedDocuments` | Data completeness, and a growing deleted count (needs a merge or cleanup) |
| `StorageUsedInS3`, `StorageUsedInHot` | Storage growth / cost |
| `VectorIndexBuildAccelerationOCU` | OCUs used for GPU-accelerated vector index builds |
| `2xx/3xx/4xx/5xx` | HTTP-level health per collection |

- All metrics are at **60-second** frequency and are kept for **15 months**.
- Serverless scales OCUs automatically within the **maximum OCU limit** you set. Hitting the cap shows up as rising latency or throttling, so either raise the cap or reduce load.

#### Managed OpenSearch Service (domains), k-NN specifics (verify in docs for exact metric names)
| Metric | Meaning |
| --- | --- |
| `KNNGraphMemoryUsage` / percentage | Native memory used by HNSW graphs (FAISS/NMSLIB engines load graphs off-heap) |
| `KNNCircuitBreakerTriggered` | Graph memory limit hit, so k-NN queries fail or degrade. Scale memory, use quantisation, or reduce dims. |
| `KNNHitCount` / `KNNMissCount` / `KNNCacheCapacityReached` | Graph cache efficiency; misses mean graphs are loaded from disk (slow) |
| `KNNQueryRequests`, `KNNGraphQueryErrors` | Query volume and errors |
| Standard: `CPUUtilization`, `JVMMemoryPressure`, `SearchLatency`, `IndexingLatency`, `ClusterStatus.red/yellow`, `FreeStorageSpace` | General cluster health |

- **Memory sizing rule of thumb (HNSW):** about `1.1 × (4 × d + 8 × M) × num_vectors` bytes (d = dimensions, M = HNSW connections), plus replicas. Lower it with byte or binary vectors, **product quantisation** or **disk-based vector search** (on-disk mode with compression). Verify in docs for current mode names.

#### Automated index optimisation routines
| Routine | Purpose | How |
| --- | --- | --- |
| **Force merge** (managed OpenSearch) | Fewer segments → fewer HNSW graphs searched → lower latency; purges deleted docs | `POST /idx/_forcemerge?max_num_segments=1` during off-peak hours, via an EventBridge Scheduler → Lambda job. Don't run it on indexes that are actively written. |
| **Refresh interval tuning** | Faster bulk ingest | Set `refresh_interval: -1` (or 30s) during big syncs, then restore it |
| **Warm-up** | Avoid cold first queries | `GET /_plugins/_knn/warmup/idx` loads graphs into memory after deploys or restarts |
| **Reindex / blue-green index** | Change dims, engine or HNSW params (`m`, `ef_construction`), or switch embedding models | Build a new index → validate recall → swap the **alias** atomically |
| **Shard rebalancing** | Hot shards | Monitor shard sizes (aim for about 10–50 GB per shard); split or reindex |
| **ISM policies** | Lifecycle for time-based indexes | Rollover, delete old indexes |

- Index design (sharding, multi-index, HNSW parameters, engines) is in [Task 1.4](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md). Search-side tuning is in [Task 4.2](02-task-4.2-application-performance.md).

#### Bedrock Knowledge Base ingestion monitoring
- Each `StartIngestionJob` has **statistics**: documents scanned, new indexed, modified indexed, deleted, metadata-scanned, and **failed**, plus `failureReasons`. Poll with `GetIngestionJob`, or subscribe to the job status change events. Alarm on `failed > 0` or `status = FAILED`.
- KB **logging** (application logs delivered to CloudWatch Logs, S3 or Firehose) gives per-document ingestion status and the reason a file was skipped, for example unsupported format, file too large or a metadata error (verify in docs for delivery options).
- Common failures: embedding dimension mismatch with the index, a file over the size limit, malformed `.metadata.json`, permission errors on S3 or KMS, throttling on the embedding model.

#### Data quality validation processes
| Check | How |
| --- | --- |
| **Completeness** | `SearchableDocuments` (or the KB indexed count) ≈ the source document count from an S3 inventory or the CMS API; flag drift |
| **Freshness** | Max `last_updated` metadata in the index vs the source; alarm if the last successful sync is older than the SLA |
| **Embedding validity** | Correct dimension count, no all-zero or NaN vectors, norm within the expected range |
| **Duplicate / near-duplicate chunks** | Cosine similarity > 0.98 between chunks → dedupe (noise in top-k) |
| **Orphans** | Vectors whose source doc was deleted (sync drift) → reconcile job |
| **Metadata integrity** | Required filter fields present (tenant, ACL, date). A missing tenant field is a **data-leak risk** in multi-tenant filtering. |
| **Retrieval canary** | Golden queries with known expected doc IDs, run on a schedule. Track **hit rate@k** and **MRR** as custom metrics. A drop means index or embedding drift. |

```python
# Scheduled retrieval canary (EventBridge Scheduler → Lambda)
hits = 0
for q in golden_queries:                      # [{"q": "...", "expected": "doc-123"}, ...]
    r = agent_rt.retrieve(knowledgeBaseId=KB_ID, retrievalQuery={"text": q["q"]},
          retrievalConfiguration={"vectorSearchConfiguration": {"numberOfResults": 5}})
    uris = [x["location"]["s3Location"]["uri"] for x in r["retrievalResults"]]
    hits += any(q["expected"] in u for u in uris)
cloudwatch.put_metric_data(Namespace="GenAI/Retrieval",
    MetricData=[{"MetricName": "HitRateAt5", "Value": hits / len(golden_queries), "Unit": "None"}])
```

---

### Skill 4.3.6 — FM-specific troubleshooting frameworks

> Develop FM-specific troubleshooting frameworks to identify unique GenAI failure modes that are not present in traditional ML systems (for example, by using golden datasets to detect hallucinations, output diffing techniques to conduct response consistency analysis, reasoning path tracing to identify logical errors, specialized observability pipelines).

#### GenAI-specific failure modes
| Failure mode | Symptom | Detection |
| --- | --- | --- |
| **Hallucination** | Fluent but false or unsupported claims | Golden datasets, grounding checks, citation verification, LLM-as-a-judge |
| **Inconsistency** | Same question → different answers | Output diffing across N runs; semantic similarity variance |
| **Prompt / response drift** | Behaviour changes after a model or prompt update | Golden-set regression + behaviour metrics per version |
| **Reasoning errors** | Right facts, wrong conclusion; wrong tool order | Reasoning-path tracing (agent traces, extended thinking) |
| **Context overflow / truncation** | `max_tokens` stop reason, cut-off answers, ignored instructions in long contexts | Logs Insights on `stopReason`, input token p95 |
| **Retrieval misses** | Answer says "I don't know", or answers from model priors instead of the KB | Retrieval canaries, retrieved-chunk logging, relevance scores |
| **Format breakage** | Invalid JSON, missing fields | Schema validation failure metric |
| **Tool misuse / loops** | Repeated calls, wrong arguments | Tool spans, loop counters |
| **Safety over-blocking** | Legitimate requests refused | Guardrail intervention review, refusal-rate metric |

#### Golden datasets for hallucination detection
- A curated set of **(question, reference answer, supporting sources)** examples, versioned in S3 and owned by domain experts. Include edge cases, adversarial prompts and "unanswerable" questions where the right output is "I don't know".
- Run it **on a schedule** (detects drift in a managed model) and **on every change** (prompt, model, KB, chunking). Score with exact or semantic match, a Bedrock evaluation job (LLM-as-a-judge: correctness, faithfulness) or RAG evaluation. Domain 5 covers evaluation mechanics.
- Publish the pass rate and hallucination rate as CloudWatch metrics so they appear next to operational metrics and can be alarmed.

#### Output diffing for consistency analysis
- **Within a version:** send the same prompt N times (temperature > 0) and compute pairwise **embedding cosine similarity** of the outputs. High variance means unstable behaviour, so lower the temperature, tighten instructions or add examples.
- **Across versions:** run golden prompts on version A and version B, then diff:
  - **lexical** (exact diff, ROUGE)
  - **semantic** (embedding similarity, flag below a threshold such as 0.85)
  - **structural** (JSON keys and types)
  - **judge-based** (an LLM decides "same meaning?" and explains the difference)
- Use the diffs as a **deployment gate**: block a release if more than X% of golden answers semantically diverge without an approved reason.

```python
def semantic_diff(a: str, b: str) -> float:
    ea, eb = embed(a), embed(b)                     # Titan Text Embeddings V2, normalize=True
    return sum(x*y for x, y in zip(ea, eb))         # cosine (vectors already normalised)

flagged = [p for p in golden if semantic_diff(old[p], new[p]) < 0.85]
```

#### Reasoning-path tracing
- **Bedrock Agents trace**: the orchestration trace includes `rationale` (the model's reasoning text), `invocationInput` (which tool or KB and which arguments), `observation` (the result) and `finalResponse`. Walk the trace to find the step where logic went wrong: a wrong tool, a misread observation, or a missing step.
- **AgentCore / Strands spans**: a timeline of each model call, tool call and its inputs and outputs. The **trajectory view** in GenAI observability shows the path.
- **Extended thinking / reasoning content** (models that support it): log the reasoning blocks (with care, since they may contain sensitive content) to see *why* the model concluded something.
- Typical findings and fixes:
  - wrong tool chosen → improve tool descriptions
  - correct retrieval but the answer ignores it → instruction ordering or context placement (lost in the middle)
  - premature final answer → add a verification step or reflection

#### Specialised observability pipelines
```
Bedrock invocation logs ─┐
Agent traces / OTel spans ├─► CloudWatch Logs ──subscription filter──► Kinesis Data Firehose
App feedback events ─────┘                                              │ (optional Lambda transform:
                                                                        │  PII redaction, flatten JSON)
                                                                        ▼
                                                     S3 (Parquet, partitioned by date/model/app)
                                                                        │
                                       Glue Data Catalog ◄── crawler ───┤
                                                                        ▼
                                       Athena (SQL forensics, cohort analysis) ──► QuickSight dashboards
                                                                        │
                                       Async evaluator (Step Functions/Lambda + LLM-as-a-judge on samples)
                                                                        ▼
                                       Custom CloudWatch metrics (hallucination rate, grounding score) ──► alarms
```
- Why do this instead of only using CloudWatch? The cost of long retention, SQL joins across logs, feedback and business outcomes, ML-friendly Parquet, and BI dashboards.
- Sampling: evaluate 1–10% of production traffic with LLM-as-a-judge to control evaluator cost. Evaluate 100% of flagged items (thumbs-down, guardrail interventions).
- Link from metric → log → trace with a shared **request ID / trace ID** so an alarm can be followed to the specific conversation.

#### Troubleshooting runbook (GenAI)
1. **Is it the platform?** Check throttles, server errors, OTPS, Health Dashboard, the quota metric `EstimatedTPMQuotaUsage`.
2. **Is it the input?** Look at the input token p95, `stopReason=max_tokens`, guardrail input interventions, and a change in user-query mix.
3. **Is it retrieval?** Check the retrieval canary hit rate, KB ingestion failures, the retrieved chunks for a failing example, and vector store latency.
4. **Is it the prompt or model version?** Compare golden-set results across versions (output diffing). Check the deploy history (prompt version, model ID, inference parameters).
5. **Is it reasoning or tools?** Walk the agent trace and look at tool error rates.
6. **Fix → add the failing case to the golden dataset** so it never regresses.

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Need TTFT for a chat app on Bedrock | `TimeToFirstToken` metric, **streaming APIs only** (ConverseStream / InvokeModelWithResponseStream) |
| Latency rose; is the model slower or are answers longer? | Compute **OTPS** = OutputTokenCount / (InvocationLatency − TTFT) × 1000; alarm on an OTPS drop |
| Error counts look too low when filtered by model | Errors before model resolution have **no `ModelId` dimension**; query without a dimension |
| Quota-exceeded requests aren't in `InvocationThrottles` | They are counted as `InvocationClientErrors` |
| Capacity planning for TPM quota | Throttling reserves **input + max_tokens** upfront; `EstimatedTPMQuotaUsage` is only an estimate. Lower `max_tokens`. |
| Verify the prompt cache is working | `CacheReadInputTokenCount` rising (cache reads don't count toward TPM) |
| Detect usage of soon-to-be-deprecated models | `LegacyModelInvocations` metric alarm |
| Capture full prompts and responses for analysis | **Model invocation logging** → CloudWatch Logs / S3 (off by default) |
| Alert if audit logs stop arriving | `ModelInvocationLogs*DeliveryFailure` metrics |
| Sudden spike in tokens with unknown cause | CloudWatch **anomaly detection** alarm on `InputTokenCount`; then Logs Insights by `identity.arn` |
| Anomaly detection on a brand-new workload | Needs about **2 weeks** of history; start with a static threshold |
| Track hallucination rate on a dashboard | Async evaluator → **custom metric (EMF / PutMetricData)**; guardrail contextual-grounding interventions |
| Measure guardrail block rate split by input vs output | `AWS/Bedrock/Guardrails` `InvocationsIntervened` by `GuardrailContentSource` |
| Per-team Bedrock cost visibility + alerts | **Application inference profiles + cost allocation tags**; Budgets / Cost Anomaly Detection |
| Unexpected Bedrock bill spike detection | **AWS Cost Anomaly Detection** monitor (service / tag) → SNS |
| End-to-end trace across API Gateway, Lambda, KB, model, tools | **X-Ray / ADOT** → CloudWatch **Transaction Search**; GenAI observability dashboard |
| Observe a Strands agent running on EKS (not AgentCore Runtime) | **ADOT** (`aws-opentelemetry-distro`, `opentelemetry-instrument`), `AGENT_OBSERVABILITY_ENABLED=true`, Transaction Search enabled |
| Agent on AgentCore Runtime — what instrumentation is needed? | None extra; **auto-instrumented** with OTel; metrics in `bedrock-agentcore` namespace |
| Correlate traces across multiple agent runs in one conversation | OTel **baggage** `session.id` |
| Find which tool is slowing down or breaking the agent | Per-tool spans / agent trace → per-tool latency & error metrics; Gateway metrics |
| Agent stuck calling the same tool repeatedly | Loop detection on tool-call counts per session + max-iteration cap |
| Vector search latency rising on OpenSearch Serverless | `SearchRequestLatency`, `SearchOCU` vs max OCU cap |
| k-NN queries failing on managed OpenSearch | `KNNCircuitBreakerTriggered` / graph memory → scale memory, quantise, fewer segments |
| Speed up k-NN after heavy updates | **Force merge** off-peak + warm-up API |
| Change embedding model without downtime | New index → validate recall → **alias swap** (blue/green) |
| KB answers missing new documents | Ingestion job stats (`failed`, failure reasons) + freshness check + sync schedule |
| Detect silent retrieval quality degradation | Scheduled **retrieval canary** with golden queries → hit rate@k metric |
| Detect hallucination regressions after a model update | **Golden dataset** regression run + **output diffing** (semantic similarity) as a gate |
| Same question gives different answers | Consistency analysis: N runs, embedding-similarity variance → lower temperature / tighten prompt |
| Agent reached wrong conclusion despite correct data | **Reasoning-path tracing** (agent orchestration trace rationale / spans) |
| Long-term, SQL-queryable GenAI analytics + BI | CloudWatch Logs → **Firehose** → S3 (Parquet) → Glue → **Athena** → **QuickSight** |
| Prove what the model told a customer a year ago | Invocation logs in S3 with **Object Lock**, CloudTrail with integrity validation, correlation IDs |

## Flashcards

<details>
<summary><b>Q:</b> Which Bedrock metric measures time to the first streamed token, and when is it published?</summary>

`TimeToFirstToken`, in milliseconds. It is published only for the streaming APIs, ConverseStream and InvokeModelWithResponseStream.

*D4 · Task 4.3 · Bedrock metrics*
</details>

<details>
<summary><b>Q:</b> Are throttled requests included in the `Invocations` metric?</summary>

No. Throttled requests appear only in `InvocationThrottles`. They are not counted in `Invocations`, `InvocationClientErrors` or `InvocationServerErrors`.

*D4 · Task 4.3 · Bedrock metrics*
</details>

<details>
<summary><b>Q:</b> A request is rejected for exceeding a service quota. Which metric counts it?</summary>

`InvocationClientErrors`, not `InvocationThrottles`.

*D4 · Task 4.3 · Bedrock metrics*
</details>

<details>
<summary><b>Q:</b> Why can filtering Bedrock error metrics by `ModelId` undercount errors?</summary>

Requests that fail before Bedrock works out the target model are published without the `ModelId` dimension. Query the metric with no dimension to see the true total.

*D4 · Task 4.3 · Metric dimensions*
</details>

<details>
<summary><b>Q:</b> Give the OTPS formula and say what an OTPS drop means.</summary>

OTPS = OutputTokenCount / (InvocationLatency − TimeToFirstToken) × 1000. A drop in OTPS means model-side throughput has degraded. If latency rises while OTPS stays stable, the responses or prompts have just got longer.

*D4 · Task 4.3 · Latency diagnosis*
</details>

<details>
<summary><b>Q:</b> Why shouldn't `EstimatedTPMQuotaUsage` be the only input to capacity planning?</summary>

It's an approximation. Throttling is based on an upfront reservation of input tokens plus `max_tokens`, which can be much higher than what the estimate shows.

*D4 · Task 4.3 · Quota monitoring*
</details>

<details>
<summary><b>Q:</b> Do prompt-cache read tokens count toward the TPM quota?</summary>

No. `CacheReadInputTokenCount` tokens are charged at a reduced rate and don't count toward TPM. Cache write tokens (`CacheWriteInputTokenCount`) do count.

*D4 · Task 4.3 · Prompt caching metrics*
</details>

<details>
<summary><b>Q:</b> How do you get alerted if Bedrock invocation logs stop being delivered?</summary>

Alarm on `ModelInvocationLogsCloudWatchDeliveryFailure` / `ModelInvocationLogsS3DeliveryFailure` (and the LargeData variant) in `AWS/Bedrock`.

*D4 · Task 4.3 · Invocation logging*
</details>

<details>
<summary><b>Q:</b> Which namespace and metric give you the guardrail intervention rate, split by input and output?</summary>

`AWS/Bedrock/Guardrails` → `InvocationsIntervened`, using the `GuardrailContentSource` dimension (Input or Output). `GuardrailPolicyType` breaks it down further by policy.

*D4 · Task 4.3 · Guardrail metrics*
</details>

<details>
<summary><b>Q:</b> What one-time account setup does AgentCore Observability need before spans appear?</summary>

Turn on CloudWatch Transaction Search. It sends X-Ray spans to CloudWatch Logs (the `aws/spans` log group). `agentcore deploy` can turn it on for you.

*D4 · Task 4.3 · AgentCore Observability*
</details>

<details>
<summary><b>Q:</b> A Strands agent runs on EKS. How do you get its traces into the CloudWatch GenAI observability dashboard?</summary>

Use ADOT. Add `aws-opentelemetry-distro`, set the OTEL env vars (including `AGENT_OBSERVABILITY_ENABLED=true` and `OTEL_RESOURCE_ATTRIBUTES=service.name=...`), and run with `opentelemetry-instrument`. On Lambda, use the ADOT Lambda layer instead.

*D4 · Task 4.3 · AgentCore Observability*
</details>

<details>
<summary><b>Q:</b> How do you link traces from several agent invocations in the same user conversation?</summary>

Put a session ID in OpenTelemetry baggage (`baggage.set_baggage("session.id", session_id)`).

*D4 · Task 4.3 · Tracing*
</details>

<details>
<summary><b>Q:</b> Why emit custom GenAI quality metrics with EMF rather than `PutMetricData`?</summary>

EMF is a structured log line that CloudWatch turns into metrics asynchronously. There's no extra API call or API throttling, and the full log context stays queryable. Keep high-cardinality fields such as requestId as log fields, not dimensions.

*D4 · Task 4.3 · Custom metrics*
</details>

<details>
<summary><b>Q:</b> How much history does CloudWatch anomaly detection need, and what should you do for a new workload?</summary>

About two weeks to build a reliable model. Until then, use static thresholds.

*D4 · Task 4.3 · Anomaly detection*
</details>

<details>
<summary><b>Q:</b> How do you track semantic response drift, beyond changes in length?</summary>

Embed a sample of responses (or the answers to fixed golden prompts) and track their cosine distance from a baseline centroid as a custom metric. Alarm on step changes, especially after model or prompt deploys.

*D4 · Task 4.3 · Drift*
</details>

<details>
<summary><b>Q:</b> Which two services give cost alerting for Bedrock, and how do they differ?</summary>

**Cost Anomaly Detection** uses ML to spot unusual spend by service, account, cost category or tag. **AWS Budgets** alerts on fixed or forecast thresholds and can run budget actions. Tag application inference profiles so costs can be broken down per team.

*D4 · Task 4.3 · Cost monitoring*
</details>

<details>
<summary><b>Q:</b> What do you need to reconstruct, for an audit, exactly what a model said to a customer and why?</summary>

You need:
- an end-to-end correlation ID
- CloudTrail for who made the call
- invocation logs for the prompt and response
- KB citations and guardrail trace
- the agent trace and prompt/model version
- tamper-proof storage (S3 Object Lock, CloudTrail log file validation, KMS) with explicit retention

*D4 · Task 4.3 · Forensics*
</details>

<details>
<summary><b>Q:</b> Name four per-tool metrics an agent tool performance framework should collect.</summary>

Call count, p95 latency, error/timeout rate, and argument-validation failures. Calls per task and tool result size are also useful.

*D4 · Task 4.3 · Tool observability*
</details>

<details>
<summary><b>Q:</b> Which OpenSearch Serverless metrics show retrieval latency and capacity?</summary>

`SearchRequestLatency` for latency, and `SearchOCU` / `IndexingOCU` for capacity in `AWS/AOSS`. Compare OCU usage against the maximum OCU cap.

*D4 · Task 4.3 · Vector store ops*
</details>

<details>
<summary><b>Q:</b> On managed OpenSearch, k-NN queries start failing under load. Which signal do you check, and what are the fixes?</summary>

Check `KNNCircuitBreakerTriggered` and k-NN graph memory usage. Fixes: add memory or nodes, quantise the vectors (byte, binary or PQ), use disk-based mode, reduce dimensions, or force merge to cut down segments.

*D4 · Task 4.3 · Vector store ops*
</details>

<details>
<summary><b>Q:</b> How do you swap embedding models on a production vector index without downtime?</summary>

Build a new index with the new embeddings, check recall with golden queries, then swap the index alias atomically. This is a blue/green reindex.

*D4 · Task 4.3 · Index maintenance*
</details>

<details>
<summary><b>Q:</b> How do you spot silent retrieval quality degradation in a RAG system?</summary>

Run a scheduled retrieval canary of golden queries with known expected documents. Publish hit rate@k (and MRR) as a CloudWatch metric and alarm when it drops.

*D4 · Task 4.3 · Retrieval monitoring*
</details>

<details>
<summary><b>Q:</b> What is output diffing, and how is it used as a deployment gate?</summary>

Output diffing compares responses across runs or versions. The comparison can be lexical, semantic (embedding similarity), structural (schema) or judge-based. If too many golden answers diverge semantically, the new prompt or model version is blocked.

*D4 · Task 4.3 · Troubleshooting*
</details>

<details>
<summary><b>Q:</b> An agent had the correct retrieved facts but reached the wrong conclusion. Which technique helps diagnose this?</summary>

Reasoning-path tracing. Walk the agent's orchestration trace (rationale, invocationInput, observation) or its spans and trajectory to find the step where the logic went wrong.

*D4 · Task 4.3 · Troubleshooting*
</details>

<details>
<summary><b>Q:</b> Sketch a specialised GenAI observability pipeline for long-term, SQL-queryable analysis.</summary>

CloudWatch Logs (invocation logs, traces, feedback) → subscription filter → Kinesis Data Firehose (optional Lambda for PII redaction) → S3 in Parquet → Glue Data Catalog → Athena → QuickSight. An async LLM-as-a-judge sampler writes quality metrics back to CloudWatch.

*D4 · Task 4.3 · Observability pipelines*
</details>

---
[← Back to Domain 4](README.md)
