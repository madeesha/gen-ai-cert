[← Back to Domain 5](README.md)

# 🩺 Task 5.2 · Troubleshoot GenAI applications

> **Domain 5 · Testing, Validation, and Troubleshooting (11%)**
> - Skill 5.2.1: Resolve content handling issues (context window overflow, dynamic chunking, prompt design, truncation analysis)
> - Skill 5.2.2: Diagnose and resolve FM integration issues (error logging, request validation, response analysis)
> - Skill 5.2.3: Troubleshoot prompt engineering problems (prompt testing frameworks, version comparison, systematic refinement)
> - Skill 5.2.4: Troubleshoot retrieval system issues (relevance analysis, embedding diagnostics, drift, vectorisation, chunking remediation, vector search performance)
> - Skill 5.2.5: Troubleshoot prompt maintenance issues (template testing, CloudWatch Logs, X-Ray prompt observability, schema validation, refinement workflows)

This file is about **diagnosis**: reading a symptom, finding the root cause and picking the fix. How each fix is built lives elsewhere:
- **Retry mechanics:** [Task 2.4](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md)
- **Logging setup:** [Task 2.5](../domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md)
- **Chunking and embeddings:** [Task 1.5](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md)
- **Monitoring pipelines:** [Task 4.3](../domain-4-operational-efficiency-optimization/03-task-4.3-monitoring-observability.md)
- **Evaluation:** [Task 5.1](01-task-5.1-evaluation-systems.md)

## Key points
- **Incomplete answer → read `stopReason` first.** `max_tokens` = the *output* cap was hit (raise `maxTokens`, ask for less, continue). `model_context_window_exceeded` or an input-length `ValidationException` = the *input* is too big. → [5.2.1](#skill-521--resolve-content-handling-issues--pq48)
- **Input bigger than the window → don't throw data away.** Map-reduce over chunks or RAG over a KB; a bigger-context model or truncation only delays or loses data. → [5.2.1](#skill-521--resolve-content-handling-issues--pq48)
- **Errors: classify, then retry or fix.** 4xx validation / access / not-found → fix the request, **never retry**. 429 `ThrottlingException` = **your quota**; 503 / 529 = **service capacity**; both → backoff + jitter, CRIS. → [5.2.2](#skill-522--diagnose-and-resolve-fm-integration-issues)
- **Newer model fails with the bare model ID → use the inference profile ID.** AccessDenied → walk the tree: action, resource (CRIS destination Regions), SCP, model access, KMS, endpoint policy. → [5.2.2](#skill-522--diagnose-and-resolve-fm-integration-issues)
- **Timeouts: find the layer that cut you off** (API Gateway 29 s → stream or go async; SDK read timeout; 350 s idle on NAT/endpoints → keep-alive in SDK **and** OS). → [5.2.2](#skill-522--diagnose-and-resolve-fm-integration-issues)
- **Prompt problems → controlled experiments.** Change one variable, compare Prompt Management versions on the same golden set; format drift → schema + structured output or forced tool use. → [5.2.3](#skill-523--troubleshoot-prompt-engineering-problems)
- **RAG triage: was the right chunk retrieved?** No → retrieval problem (chunking, embeddings, filters, sync, hybrid). Yes → generation problem (prompt, context order, model). → [5.2.4](#skill-524--troubleshoot-retrieval-system-issues)
- **Query and document embeddings must come from the same model and dimension.** Changing the embedding model means re-embedding everything (a new KB). → [5.2.4](#skill-524--troubleshoot-retrieval-system-issues)
- **Prompt maintenance is an observability problem:** invocation logs + `requestMetadata` + Logs Insights, X-Ray spans per stage, a schema-violation metric, and regression tests before switching the alias. → [5.2.5](#skill-525--troubleshoot-prompt-maintenance-issues)

## Notes

### Skill 5.2.1 — Resolve content handling issues 🎯 PQ48

> Resolve content handling issues to ensure that necessary information is processed completely in FM interactions (for example, by using context window overflow diagnostics, dynamic chunking strategies, prompt design optimization, truncation-related error analysis).

#### Two kinds of "incomplete" — input vs output
Converse `stopReason` values: `end_turn | tool_use | max_tokens | stop_sequence | guardrail_intervened | content_filtered | malformed_model_output | malformed_tool_use | model_context_window_exceeded`. Branch on it before trusting any response.

| Symptom | Where it breaks | Signal | Typical fix |
| --- | --- | --- | --- |
| Request rejected outright | **Input** too large for the context window | `ValidationException` (400) with a message about input length or too many tokens; or `stopReason = model_context_window_exceeded` | Trim, summarise or chunk the input; use a model with a larger context window |
| Answer stops mid-sentence or mid-JSON | **Output** hit the token cap | `stopReason = max_tokens`; `usage.outputTokens == maxTokens` | Raise `maxTokens`, ask for concise output, or **continue** in a follow-up turn |
| Answer ignores part of a long document | **Attention**: the content fitted, but the model under-weighted it | No error; eval shows missed facts in the *middle* of the context | Reorder (most relevant first or last), cut irrelevant context, map-reduce |
| Late conversation turns "forget" early facts | **History truncation** by the app or framework | Conversation manager dropped old turns | Summarise history; put key facts in the system prompt or memory |
| Stops early on an unexpected token | **Stop sequence** too broad | `stopReason = stop_sequence` | Narrow `stopSequences` |
| Output blanked or replaced | **Guardrail** | `stopReason = guardrail_intervened`, guardrail `trace` | See [Task 3.1](../domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |

- Token budget: **input tokens + `maxTokens` must fit within the model's context window**. Some models reject the request when this doesn't fit, even if the input alone does (verify in docs per model family).
- `usage.inputTokens` / `outputTokens` / `totalTokens` come back on every Converse response (and in the `metadata` event for ConverseStream). Log them and alarm when input tokens approach about 80% of the window.
- Estimate before sending. The **CountTokens** API gives an exact count for supported models (verify in docs which ones). Otherwise use the model family's tokenizer or a rough rule of ~4 characters per token for English. Don't trust that rule for code, CJK text or JSON.

#### Handling `max_tokens` truncation (continuation pattern)
```python
resp = brt.converse(modelId=MODEL, messages=msgs,
                    inferenceConfig={"maxTokens": 2048, "temperature": 0})
parts = [resp["output"]["message"]["content"][0]["text"]]
while resp["stopReason"] == "max_tokens" and len(parts) < 5:   # hard cap on loops
    msgs += [resp["output"]["message"],
             {"role": "user", "content": [{"text": "Continue exactly where you stopped. Do not repeat."}]}]
    resp = brt.converse(modelId=MODEL, messages=msgs,
                        inferenceConfig={"maxTokens": 2048, "temperature": 0})
    parts.append(resp["output"]["message"]["content"][0]["text"])
full = "".join(parts)
```
- For **structured output**, continuation is fragile because the JSON breaks at the seam. It's better to:
  1. make `maxTokens` big enough
  2. split the task, for example one object per call
  3. ask for **less**, such as IDs instead of full text
- Validate before trusting the result: `json.loads` plus a schema check (see 5.2.5).
- **Never** treat a `max_tokens` response as complete. Branch on `stopReason` in code.

#### Context window overflow diagnostics
1. Log `inputTokens` per request, broken down by component: system prompt, history, retrieved chunks, tool results, user query. A Logs Insights query on `input.inputTokenCount` in the invocation logs (see 5.2.2), or app-side counters, shows **which part grew**.
2. Common culprits:
   - unbounded **chat history**
   - **too many or too large RAG chunks** (for example `numberOfResults` = 50 on 1,000-token chunks)
   - **verbose tool results**, such as a whole API JSON response
   - **base64 images or documents**
   - **agent loops** that pile up traces
3. Fixes, cheapest first:
   - cap history to N turns or a token budget (Strands `SlidingWindowConversationManager`, or a summarising manager)
   - **summarise older turns** into a running summary
   - lower `numberOfResults` and **add a reranker** to keep only the top-k
   - shrink tool outputs (project only the needed fields)
   - **prompt caching** for a large static prefix (this cuts cost and latency, *not* the context limit)
   - switch to a model with a larger context window

#### Dynamic chunking strategies for long inputs
When trimming isn't enough because every part of the input matters, split the work across several calls:

| Pattern | How | Use when |
| --- | --- | --- |
| 🎯 **PQ18** **Map-reduce** | Split the doc → summarise or extract per chunk (in parallel) → combine the partial results in a final call | Summarising or extracting from documents larger than the window |
| **Refine** (iterative) | Running answer + next chunk → updated answer | Order matters (narratives); slower and sequential |
| **Hierarchical summarisation** | Summarise sections → summarise the summaries | Very long corpora, books |
| **Adaptive chunk size** | Size chunks from the *remaining* token budget (window − system − question − `maxTokens`) | Prompts of varying size; multi-model support |
| **Structure-aware split** | Split on headings, pages or table boundaries, never mid-table | Tables and code blocks getting cut in half |
| **RAG instead of stuffing** | Retrieve only the relevant chunks | Q&A over large docs: cheaper and more accurate than whole-doc stuffing |

- Orchestrate map-reduce with a **Step Functions Distributed Map** (or inline Map) of Lambda + Bedrock calls, then a reduce step. That gives you parallelism, retries per chunk and a visual history of which chunk failed.
- Chunk **overlap** (10–20%) stops facts from being split across a boundary.
- These are runtime patterns for **one oversized input**. Choosing a KB's ingestion chunking strategy (fixed, hierarchical, semantic, custom Lambda) is a different decision. → Full detail: [Task 1.5 · Skill 1.5.1](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md#skill-151--document-segmentation-chunking-approaches)

> [!IMPORTANT]
> **🎯 Practice question 48 — huge customer histories: context-length errors + ignored input; use ALL the data, with KBs (choose 2)**
>
> - ✅ **Chunk + map-reduce:** split the customer data into segments, process each one, then make a **final call that combines the results**. Every part of the data is used.
> - ✅ **RAG with a Bedrock knowledge base:** index the customer data as embeddings and **retrieve only the relevant parts** for each request. You get the whole dataset without overflowing the context.
> - ❌ **Raising the context window through `additionalModelRequestFields`:** that field only passes **model-specific inference parameters**. The context window is **fixed by the model**.
> - ❌ **Put key information first and last + truncate the rest:** **throws data away**.
> - ❌ **A bigger-context model + longer Lambda timeout:** puts off the failure, and doesn't use KBs.

> [!IMPORTANT]
> **🎯 Practice question 18 — multi-hour transcripts get truncated: process them fully, reliably across many FM calls**
>
> - ✅ **Dynamic chunking** into coherent segments that fit the context window + **prompts that carry continuity** between segments (a running summary or the previous segment's key points, with **enough overlap**) + **truncation diagnostics** (check `stopReason == "max_tokens"`, compare input token counts with the limit).
> - ❌ **Same, but "limit overlap for lower latency":** leaves **context gaps at segment boundaries** and lowers reliability. Watch for this near-identical distractor.
> - ❌ **A bigger context-window model:** multi-hour transcripts can still be too long, and it costs more.
> - ❌ **Summarise or compress first:** compression **loses details**, so insights get missed.

#### Prompt design to keep long inputs effective
- **Lost in the middle:** models attend best to the **start and end** of long contexts. So:
  - put **instructions in the system prompt**
  - put **documents first**, then the **question last**
  - order retrieved chunks by relevance (best at the edges)
- Wrap each document in tags (`<doc id="3" source="...">…</doc>`) so the model can cite and separate them.
- Ask the model to **quote the relevant passages first, then answer**. This improves focus on long documents.
- Remove boilerplate (headers, footers, navigation, HTML) **before** sending. This is a preprocessing fix; see [Task 1.3](../domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md).

#### Truncation-related error analysis checklist
- Truncated **JSON** → `stopReason`? `max_tokens` → raise the cap or shrink the schema. `end_turn` with broken JSON → a format problem, not truncation (see 5.2.5).
- **Silent upstream truncation** by your own code. Look for:
  - string slicing (`text[:4000]`)
  - Lambda payload limits (**6 MB** sync request/response)
  - API Gateway payload limit (**10 MB**)
  - SQS message limit (**256 KiB**; verify in docs, as the limit may have increased)
  - Step Functions state payload (**256 KiB**)

  Large payloads → pass **S3 pointers** instead of content.
- **Document parser truncation**: a scanned PDF with no text layer gives empty or partial text. Use FM parsing or Bedrock Data Automation, or Textract.
- **Streaming cut-off**: the client disconnected or hit an idle timeout before `messageStop`. Check for `messageStop.stopReason` in the client.

### Skill 5.2.2 — Diagnose and resolve FM integration issues

> Diagnose and resolve FM integration issues to identify and fix API integration problems specific to GenAI services (for example, by using error logging, request validation, response analysis).

#### Bedrock runtime error taxonomy (Converse / ConverseStream / InvokeModel)
| Exception | HTTP | Meaning | Retry? | Fix |
| --- | --- | --- | --- | --- |
| `ValidationException` | 400 | Request breaks constraints: bad body schema, parameter out of range, input too long, unsupported feature for the model | ❌ | Fix the request; check model-specific parameters; validate before sending |
| `AccessDeniedException` | 403 | IAM, SCP, KMS, model access or Marketplace agreement | ❌ | See the AccessDenied decision tree below |
| `ResourceNotFoundException` | 404 | Wrong model ID, ARN, Region, endpoint or prompt ARN | ❌ | `ListFoundationModels`, `ListInferenceProfiles`; use the profile ID; check the Region |
| `ModelTimeoutException` | 408 | Processing exceeded the model's timeout | ⚠️ Retry with a smaller request | Reduce input or `maxTokens`; stream; split the task |
| `ModelErrorException` | 424 | The model failed while processing (has `originalStatusCode`, `resourceName`) | ⚠️ Limited | Common with **Custom Model Import / Marketplace** endpoints; inspect the original code |
| 🎯 **PQ46** `ModelNotReadyException` | 429 | Model not ready to serve (for example, an imported model scaled to zero is warming up) | ✅ SDK auto-retries **up to 5×** | Wait or retry; keep it warm with periodic traffic if latency matters |
| `ThrottlingException` | 429 | **Account quota** exceeded (RPM/TPM) | ✅ Backoff + jitter | Quota increase, CRIS, PT, Reserved tier, smooth traffic (SQS) |
| `ServiceQuotaExceededException` | 400 | Control-plane quota hit (for example, too many resources) (verify in docs for runtime) | ❌ | Request a quota increase |
| `InternalServerException` | 500 | Server-side error | ✅ Backoff + jitter | Contact Support if it persists |
| `ServiceUnavailableException` | 503 | **Service capacity**, *not* your quota | ✅ Backoff + jitter | CRIS / another Region / Reserved tier or PT |
| `overloaded_error` | 529 | Model temporarily over capacity | ✅ Respect `Retry-After` | Backoff, avoid synchronised retries, CRIS |
| `modelStreamErrorException` (stream event) | 424 | Error **mid-stream** after HTTP 200 | ✅ Re-issue the request | Handle error events inside the stream loop |

Rule of thumb: **4xx client errors → fix the request, don't retry; 429 and 5xx → backoff with jitter.** A 429 `ThrottlingException` is *your* quota; a 503 is the *service's* capacity.

Other errors from the troubleshooting page:
- **`FTUFormNotFilled` (404):** the Anthropic use-case form hasn't been submitted.
- **`MPAgreementBeingCreated` (403):** wait about 15 minutes.
- **`IncompleteSignature` / `InvalidClientTokenId` / `RequestExpired`:** credential, signature or clock-skew problems.

Retry mechanics (standard vs adaptive modes, jitter, circuit breakers) are in [Task 2.4](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md).

> [!IMPORTANT]
> **🎯 Practice question 46 — imported model throws ModelNotReadyException after idle periods; keep it warm at peak, cheap off-peak**
>
> - **Custom Model Import models are evicted (scaled to zero) after a period of inactivity.** The next call gets `ModelNotReadyException` while the model is restored.
> - **EventBridge Scheduler → Lambda → small invoke at a modest interval** (the answer key uses **every 45 minutes**, during peak hours only) **+ exponential backoff retries** (boto3 `Config(retries={'mode': 'standard' or 'adaptive', ...})`). Use **CloudWatch Logs** to find availability patterns.
> - ❌ **Invoking every minute:** wasted calls and cost.
> - ❌ **Constant Step Functions warm-up with parallel paths:** costly and complex.
> - ❌ **Warm-up 15 minutes before peak + *linear* backoff:** gaps during the peak, and AWS recommends **exponential backoff with jitter**.
> - *Check the exact inactivity window before eviction in the Custom Model Import docs (verify in docs).*

#### AccessDeniedException decision tree
1. **Is the action right?**
   - Converse needs `bedrock:InvokeModel`.
   - ConverseStream needs `bedrock:InvokeModelWithResponseStream`.
   - 🎯 **PQ14** Agents need `bedrock:InvokeAgent`.
   - KBs need `bedrock:Retrieve` / `bedrock:RetrieveAndGenerate`.
   - Prompt Management ARNs need `bedrock:RenderPrompt`.
   - Guardrails on the call need `bedrock:ApplyGuardrail`.
2. **Is the resource right?**
   - For **CRIS**, allow the **inference-profile ARN** *and* the **foundation-model ARN in every destination Region** (or `arn:aws:bedrock:*::foundation-model/...`).
   - **SCPs** must not deny any destination Region. Global profiles need `aws:RequestedRegion = "unspecified"` allowed; see [Task 2.3](../domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md).
3. **Model access / agreements:** first-time Anthropic use-case form; Marketplace subscription pending; model not offered in the Region.
4. **KMS:** customer managed key on a KB, agent, custom model or log bucket → the caller or service role needs `kms:Decrypt` / `kms:GenerateDataKey`, and the **key policy** must allow it.
5. **Service roles:** the KB role can't read S3 or write to the vector store; the agent role can't invoke the action-group Lambda. The error appears in the **ingestion job** or **agent trace**, not as a direct 403 to the caller.
6. **VPC endpoint policy** on `bedrock-runtime` doesn't allow the principal, action or model → 403 even though IAM allows it. Check the **endpoint policy** and any `aws:SourceVpce` conditions on bucket or KMS policies.
7. **Expired temporary credentials** → refresh the session.
8. Diagnose with **CloudTrail**, which records `errorCode` / `errorMessage` and the principal. IAM **Policy Simulator** and **IAM Access Analyzer** help too.

#### ResourceNotFound and "on-demand throughput isn't supported"
- Many newer models are **only invocable via an inference profile**. Calling the bare `anthropic.claude-...` model ID gives a `ValidationException` like *"Invocation of model ID … with on-demand throughput isn't supported. Retry your request with the ID or ARN of an inference profile"*. Fix it by using `us.anthropic.…` / `eu.…` / `global.…` (exact wording: verify in docs).
- The model isn't available in the client's Region → `ResourceNotFound` or a validation error. Check the **Region of the SDK client** (a common mistake: Lambda in `eu-west-1` calling a model that is only available in `us-east-1`).
- Custom models: call the **custom model deployment ARN** (on-demand) or the **Provisioned Throughput ARN**, not the custom model ARN.
- Prompt resource: `arn:...:prompt/ID:VERSION`. If you pass `inferenceConfig`, `system`, `toolConfig` or `additionalModelRequestFields` with a prompt ARN, the request is **rejected**, because those must be defined in Prompt Management.

#### Malformed request bodies (InvokeModel vs Converse)
| | InvokeModel | Converse |
| --- | --- | --- |
| Body | **Model-family-specific JSON** (Anthropic Messages body with `anthropic_version`, Titan `inputText` + `textGenerationConfig`, Llama `prompt`, Nova `messages` + `inferenceConfig`) | **Unified** `messages`, `system`, `inferenceConfig`, `toolConfig` |
| Typical 400 causes | Wrong family schema (sending a Titan body to Claude); missing `anthropic_version`; `max_tokens` vs `maxTokenCount` naming | Roles not alternating user/assistant; first message not `user`; empty content; `toolResult` without the matching `toolUseId`; unsupported content type (doc or image) for the model |
| Extra params | In the body | `additionalModelRequestFields` (for example `top_k`); a bad JSON Pointer in `additionalModelResponseFieldPaths` → 400 |
| Switching models | Rewrite the body | Change `modelId` (mostly) |

- **Exam pattern:** errors after swapping models with InvokeModel → migrate to **Converse** for a model-agnostic request format. Request anatomy and full bodies → [Task 1.3 · Skill 1.3.3](../domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md#skill-133--formatting-input-data-for-fm-inference).
- Tool use errors: a `toolResult` must come in the **next user message**, reference the same `toolUseId`, and the tool's **`inputSchema` must be valid JSON Schema**. `stopReason = malformed_tool_use` means the model produced invalid tool input. Retry with a clearer schema, `toolChoice`, or a stronger model.
- Images and documents: check the format and size limits, and the number of attachments per request (verify in docs for current limits).

#### Request validation (fail fast, before calling the FM)
- **API Gateway request validators** with a JSON Schema **model** reject malformed client requests with **400** before they reach Lambda or Bedrock.
- Lambda-side validation (Pydantic / jsonschema):
  - required fields
  - prompt length (in tokens) below the budget
  - `maxTokens` within the model's maximum
  - allowed `modelId` values (allowlist via AppConfig)
  - attachment MIME types and sizes
- Return clear 4xx errors to callers. Never forward raw model errors containing internal ARNs.

#### Streaming-specific failures
- ConverseStream returns **HTTP 200 and then events**. Errors can arrive **inside the stream**: `internalServerException`, `modelStreamErrorException`, `throttlingException`, `validationException`, `serviceUnavailableException`. **Handle these in the event loop.** A try/except around the initial call isn't enough.
- No `messageStop` → the stream was interrupted. Treat the output as partial and retry or resume.
- The AWS **CLI doesn't support streaming** operations for Bedrock. Test with an SDK.
- Check that the model supports streaming: `GetFoundationModel` → `responseStreamingSupported`.
- Proxies or load balancers buffering SSE (for example, a misconfigured CloudFront/ALB, or a missing `Content-Type: text/event-stream`) → the client receives everything at once. Fix with no-buffer settings and the streaming integration.

#### Timeouts across the stack
| Layer | Limit | Symptom | Fix |
| --- | --- | --- | --- |
| API Gateway REST (default integration timeout) | **29 s** (can be raised for Regional/private APIs via a quota increase; verify in docs) | **504** Gateway Timeout | **REST response streaming**, async (SQS + job ID + polling, or WebSocket), shorter outputs |
| API Gateway HTTP API | 30 s max | 503/504 | Same as above |
| Lambda | 15 min (900 s) max (function setting may be lower) | `Task timed out after X seconds` | Raise the timeout, stream, offload to Step Functions or async |
| SDK read timeout | botocore default **60 s** `read_timeout` | `ReadTimeoutError` on long generations | Raise `read_timeout` in `Config` for long or extended-thinking calls; stream |
| NAT GW / interface endpoint / NLB idle | **350 s** fixed | Connection reset; 70 s+ first call after idle | `tcp_keepalive=True` **and** OS `tcp_keepalive_time` < 350 (for example 45) |

```python
from botocore.config import Config
cfg = Config(read_timeout=300, connect_timeout=10, tcp_keepalive=True,
             retries={"mode": "adaptive", "max_attempts": 8})
brt = boto3.client("bedrock-runtime", config=cfg)
```

#### VPC endpoint / DNS issues
- In a private subnet with **no NAT and no interface endpoint** for `bedrock-runtime`, calls hang and then time out. Add the **interface endpoint** (and `bedrock-agent-runtime` for KB/agents).
- With **private DNS turned off** on the endpoint, the SDK still resolves the public hostname. Either turn on private DNS or set `endpoint_url` to the VPCE DNS name.
- **Security group** on the endpoint must allow **443 inbound** from the caller's SG/subnets.
- The endpoint **policy** may restrict actions or models, which gives a 403 (see the AccessDenied tree).
- Using a control-plane endpoint for runtime calls is a common mix-up: `bedrock` (control plane) vs `bedrock-runtime` (inference) vs `bedrock-agent-runtime` (KB / agents runtime).

#### Error logging and response analysis
- **Model invocation logging** (off by default) logs the request and response body, token counts and model ID to CloudWatch Logs and/or S3. Setup is in [Task 2.5 · Skill 2.5.6](../domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md#skill-256--troubleshooting-fm-applications).
- Put `requestMetadata` (up to **16** key/value pairs) on Converse calls, for example `{"tenant": "acme", "promptVersion": "7", "feature": "summarise"}`. You can then **filter invocation logs** by them.
- **CloudTrail** records the API call, principal, `errorCode` and, for CRIS, `additionalEventData.inferenceRegion`. It doesn't record prompt content.
- CloudWatch `AWS/Bedrock` metrics: `Invocations`, `InvocationLatency`, `InvocationClientErrors`, `InvocationServerErrors`, `InvocationThrottles`, `InputTokenCount`, `OutputTokenCount`. Alarm on error rate and throttles. Full metric list → [Task 4.3 · Skill 4.3.2](../domain-4-operational-efficiency-optimization/03-task-4.3-monitoring-observability.md#skill-432--comprehensive-genai-monitoring-and-kpis).
- Response analysis checklist: `stopReason`, `usage` tokens, `metrics.latencyMs`, guardrail `trace`, `trace.promptRouter.invokedModelId` (which model the router picked), `serviceTier` actually used.

```sql
-- Logs Insights: errors & throttles by model in the invocation log group
fields @timestamp, modelId, errorCode, identity.arn
| filter ispresent(errorCode)
| stats count(*) as errors by modelId, errorCode
| sort errors desc
```

(Field names follow the invocation log schema. Verify in docs for your log version.) A query for requests approaching the context limit (filter on `input.inputTokenCount`, add `requestMetadata.feature` to see which feature grew) and other invocation-log queries → [Task 2.5 · Skill 2.5.6](../domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md#skill-256--troubleshooting-fm-applications).

### Skill 5.2.3 — Troubleshoot prompt engineering problems

> Troubleshoot prompt engineering problems to improve FM response quality and consistency beyond basic prompt adjustments (for example, by using prompt testing frameworks, version comparison, systematic refinement).

#### Symptom → cause → fix
| Symptom | Likely cause | Diagnosis | Fix |
| --- | --- | --- | --- |
| Different answers to the same input | High `temperature` / `topP`; ambiguous instructions | Run the same input N=10 times and measure variance | `temperature` 0–0.2 for deterministic tasks; tighten the instructions; give examples |
| Ignores an instruction | Instruction buried in a long user message; conflicting instructions; instruction after the data | Move the instruction and retest | Put rules in the **system prompt**; one clear rule per line; data in `<context>` tags, **question last** |
| Wrong format (prose instead of JSON, extra commentary) | No schema; examples contradict the instruction | Schema-validation failure rate | **Structured output** (`outputConfig.textFormat` with a JSON Schema where supported) or **forced tool use** (`toolChoice: {tool: {name}}`) + validation; prefill the assistant turn (`{`) where supported |
| Makes up facts | No grounding; prompt invites guessing | Faithfulness / grounding eval | RAG + "answer only from the context; otherwise say *I don't know*"; contextual grounding guardrail |
| Too verbose or too short | No length guidance; `maxTokens` | Output token distribution | Explicit length ("≤ 3 bullets"); adjust `maxTokens` |
| Refuses benign requests | Over-cautious system prompt; guardrail too strict | Guardrail `trace`; test without the guardrail | Reword the role; tune filter strengths or denied-topic definitions |
| Good on the old model, poor on the new one | Prompt tuned to the old model's quirks | Same golden set on both versions | Re-tune per model (each model has its own prompt guide); keep a prompt **variant per model** |
| Works in testing, fails on real users | Test set not representative | Error analysis on production logs | Add failing production cases to the golden set |
| Ignores few-shot style | Too few or inconsistent examples | Ablation: remove or add examples | 3–5 diverse, consistent examples in tags |
| Chain-of-thought leaks into the output | CoT asked for but not separated | Output contains reasoning | Ask for reasoning in `<thinking>` and the answer in `<answer>`; parse only `<answer>`; or use the model's reasoning mode |

#### Prompt testing frameworks
- **Golden dataset**: inputs + expected outputs or rubrics, covering typical, edge and adversarial cases. Version it in S3 or Git.
- **Assertions**:
  - deterministic: regex, JSON Schema, contains / not-contains, exact match for classification
  - semantic: embedding similarity to the reference
  - LLM-as-a-judge: a rubric score
- **Tooling**:
  - Bedrock **Prompt Management** test window and **compare variants** side by side
  - **Bedrock Evaluations** with a custom prompt dataset and LLM-as-a-judge (see [Task 5.1](01-task-5.1-evaluation-systems.md))
  - open-source tools (promptfoo, DeepEval, RAGAS) run in CodeBuild as a CI quality gate
- **Step Functions Map** over the golden set → invoke → score in Lambda → aggregate → pass or fail. Orchestration patterns are in [Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md).

#### Version comparison
- Prompt Management keeps **immutable versions** (`prompt/ID:1`, `:2`, …) plus a working **DRAFT**. Compare **v_n vs v_n+1** on the **same dataset, model and parameters**.
- Record per version: pass rate, mean judge score, format-violation rate, input and output tokens, latency. Promote only if quality ≥ baseline **and** cost and latency stay within budget.
- Roll back by pointing the app (AppConfig parameter or alias) at the previous version ARN, with no code deploy.
- `diff` the prompt text between versions to tie a regression to a specific edit.

#### Systematic refinement loop
1. **Reproduce**: capture the failing input, full rendered prompt, model ID, parameters and output from the invocation logs.
2. **Classify** the failure: format, factual, instruction-following, tone, refusal, truncation.
3. **Hypothesise** one cause and change **one variable** (an instruction, an example, ordering, temperature, model).
4. **Test** on the failing case **and** the full regression set, so you don't fix one case and break five.
5. **Version** the change in Prompt Management with a description; promote through the approval workflow.
6. **Monitor** in production (format-violation metric, user feedback).

### Skill 5.2.4 — Troubleshoot retrieval system issues

> Troubleshoot retrieval system issues to identify and resolve problems that affect information retrieval effectiveness for FM augmentation (for example, by using model response relevance analysis, embedding quality diagnostics, drift monitoring, vectorization issue resolution, chunking and preprocessing remediation, vector search performance optimization).

#### First split: retrieval vs generation
```
Wrong / missing answer
├── Call Retrieve (not RetrieveAndGenerate) with the same query → inspect chunks + scores
│   ├── Correct chunk NOT in top-k  → RETRIEVAL problem (below)
│   └── Correct chunk IS in top-k   → GENERATION problem
│        ├── chunk buried mid-context / too many chunks → rerank, fewer chunks, reorder
│        ├── prompt doesn't force grounding → "answer only from context", citations
│        └── model too weak / temperature high → stronger model, lower temperature
```
- Use **Bedrock RAG evaluation** (retrieve-only: context relevance and coverage; retrieve-and-generate: faithfulness, correctness, citation precision) to measure the problem rather than guess. Details in [Task 5.1](01-task-5.1-evaluation-systems.md).

#### Debugging a Bedrock KB query
```python
r = bar.retrieve(
    knowledgeBaseId=KB_ID,
    retrievalQuery={"text": q},
    retrievalConfiguration={"vectorSearchConfiguration": {
        "numberOfResults": 10,
        "overrideSearchType": "HYBRID",                       # compare SEMANTIC vs HYBRID
        "filter": {"equals": {"key": "department", "value": "hr"}}}})
for x in r["retrievalResults"]:
    print(round(x["score"], 3), x["location"], x.get("metadata", {}).get("department"), x["content"]["text"][:120])
```
- Run the same query with and without the filter, and with SEMANTIC vs HYBRID. Look at whether the expected document appears and at which rank.
- Check **ingestion jobs**: `ListIngestionJobs` / `GetIngestionJob` → `statistics` (scanned, indexed, failed, deleted) and `failureReasons`. Turn on **KB logging** (delivered via CloudWatch vended logs; verify in docs) for per-document ingestion status.

With the failing stage located, match the symptom:

#### Retrieval symptom table
| Symptom | Likely cause | Diagnosis | Fix |
| --- | --- | --- | --- |
| **Nothing relevant retrieved** for exact terms (SKUs, error codes, names) | Pure semantic search misses lexical matches | Query is a code or ID; the keyword appears in the doc | **Hybrid search** (OpenSearch Serverless KB: `overrideSearchType: HYBRID`) |
| Relevant doc exists but never comes back | Doc **not ingested**: sync not run, file > size limit, unsupported format, parser failed | KB **ingestion job** stats (`numberOfDocumentsFailed`), failure reasons; KB logs | Re-sync; fix the format or size; use FM parser / BDA for scans and tables; check S3 prefix and inclusion filters |
| New docs missing, old content returned | **Stale index**: no sync after S3 update | Last ingestion-job time vs S3 `LastModified` | EventBridge/S3 event → `StartIngestionJob`; scheduled sync; see [Task 1.4](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md) |
| Chunks are fragments with no context | Chunks **too small**; split mid-table or mid-section | Inspect the retrieved text | Bigger chunks plus overlap; **hierarchical** (match child, return parent); semantic chunking |
| Chunks long and noisy; answer diluted | Chunks **too large** | Chunk token lengths; low precision | Smaller chunks; rerank |
| Right doc, wrong section | Poor chunk boundaries | Chunk inspection | Structure-aware or semantic chunking; custom Lambda chunker |
| Filtered queries return **zero** results | **Metadata filter mistakes**: wrong key or case, type mismatch (string `"2024"` vs number), sidecar not named `<file>.metadata.json`, no re-sync after adding metadata, field not filterable (S3 Vectors non-filterable keys) | Run without the filter and compare; inspect the metadata of returned chunks | Fix the key names and types; re-sync; mark the field filterable |
| Results from the wrong tenant | Missing **mandatory tenant filter** | Audit queries | Enforce the filter server-side, never from client input |
| Scores all similar or low; semantically odd matches | **Embedding model mismatch**: query embedded with a different model or version than the docs; wrong normalisation or distance metric | Check the KB config vs the custom pipeline; cosine vs L2 | Use the **same model** for queries and documents; re-embed the corpus on a model change |
| Ingestion fails: dimension error | Index dimension ≠ embedding dimension (for example, a 1024-d index fed 512-d vectors) | Ingestion error message | Recreate the index with the matching dimension (the KB embedding model and vector config are **fixed at creation**) |
| Quality degrades over months | **Drift**: new vocabulary or products, changed query mix | Rising "no answer" rate, falling eval scores, embedding-distribution drift | Refresh the corpus; add synonyms or query expansion; re-evaluate the embedding model; possibly fine-tune |
| Multi-part questions half-answered | One embedding can't cover several intents | Look at the query | **Query decomposition** (KB orchestration option or Lambda) |
| Non-English queries fail | Embedding model's language coverage | Test per language | Multilingual embedding model (Cohere Embed Multilingual, Titan V2 multilingual) |
| Tables or images ignored | Text-only parser | Chunk text empty or garbled | FM parsing / BDA; multimodal embeddings |

#### Embedding quality diagnostics
- **Known-pair test**: build query → expected-doc pairs and measure **recall@k / MRR / nDCG**. A drop after a change means the change broke retrieval.
- **Nearest-neighbour inspection**: for sample docs, inspect the top neighbours. If unrelated docs cluster together, the embeddings aren't capturing your domain.
- **Similarity distribution**: plot the cosine scores of relevant vs irrelevant pairs. Heavy overlap means weak separation. Try another embedding model, add domain context to chunks, or use **contextual chunk headers** (prepend the doc title or section).
- **Normalisation**: cosine similarity on unnormalised vectors with an L2 index (or the reverse) gives odd rankings. Titan V2's `normalize: true` and a matching space type fix this.
- **Dimensionality trade-off**: Titan V2 at 256-d is cheaper but slightly less accurate than 1024-d. If quality dropped after shrinking, that's the reason.
- **Truncated inputs at embed time**: chunks longer than the embedding model's maximum input get truncated silently (verify in docs per model), so the tail of each chunk is never embedded. Keep chunk size ≤ the model's maximum (Cohere Embed ~512 tokens; Titan V2 8,192).

#### Vectorisation issue resolution
- **Empty or garbage text** fed to the embedder (OCR failure, binary files, boilerplate-only pages) gives junk vectors that match everything weakly. Validate text length and quality before embedding ([Task 1.3](../domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md)).
- **Duplicates** crowd the top-k with copies. Deduplicate on a content hash before indexing.
- **Partial batch failures** in custom pipelines: throttled embedding calls drop docs silently. Make it idempotent with retries, a DLQ, and a reconciliation count (docs in S3 vs vectors in the index).
- **Mixed embedding versions in one index** after a partial migration: the scores aren't comparable. Re-embed fully into a **new index** and swap with an alias (blue/green).

#### Chunking and preprocessing remediation
Strategy options and their KB settings live in [Task 1.5 · Skill 1.5.1](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md#skill-151--document-segmentation-chunking-approaches). The troubleshooting angle:
- The Bedrock KB **chunking strategy and parser are fixed after the data source is created** (verify in docs; changing them means a new data source or KB and re-ingestion). Plan experiments on a copy.
- Remediation order:
  1. **clean** the text (remove headers, footers, navigation)
  2. **parse** properly (FM parser or BDA for PDFs with tables)
  3. choose a **strategy** (fixed → hierarchical/semantic)
  4. tune **size and overlap**
  5. add **metadata**
  6. re-evaluate
- **Custom transformation Lambda** for bespoke splits (by clause number, by Q&A pair, by code function).

#### Vector search performance
| Symptom | Cause | Fix |
| --- | --- | --- |
| High query latency | Too many results or large `k`; heavy filters on HNSW (post-filter); under-provisioned OCUs or shards; reranking many candidates | Lower `k`/`numberOfResults`; efficient pre-filtering; scale OCUs, shards or replicas; rerank only the top ~20–50 |
| Recall too low with ANN | HNSW `ef_search` too low; aggressive quantisation | Raise `ef_search` (latency trade-off); less aggressive quantisation; rescoring |
| Latency spikes during ingestion | Indexing competes with search | Separate collections (OpenSearch Serverless splits indexing and search compute); off-peak bulk loads |
| Memory pressure or circuit breaker in OpenSearch (managed) | HNSW graphs in memory exceed the heap or native memory | Bigger or memory-optimised nodes; quantisation (fp16, byte, binary); disk-based vector search |
| Slow end-to-end RAG but fast search | Time spent in generation or reranking | X-Ray spans per stage; reduce context; stream |

Deep OpenSearch tuning is in [Task 1.4](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md) and [Task 4.2](../domain-4-operational-efficiency-optimization/02-task-4.2-application-performance.md).

### Skill 5.2.5 — Troubleshoot prompt maintenance issues

> Troubleshoot prompt maintenance issues to continuously improve the performance of FM interactions (for example, by using template testing and CloudWatch Logs to diagnose prompt confusion, X-Ray to implement prompt observability pipelines, schema validation to detect format inconsistencies, systematic prompt refinement workflows).

#### What "prompt maintenance" problems look like
| Symptom | Root cause | Detect with |
| --- | --- | --- |
| Quality dropped with **no code change** | Model version upgraded or alias moved; upstream data or retrieval changed; user query mix drifted | Eval scores over time; `modelId` in logs; deploy timeline |
| **Template rendering bugs**: literal `{{customer_name}}` in the prompt; empty variables; double-escaped JSON | Variable missing or misnamed; null values | Template unit tests; log the **rendered** prompt; Logs Insights search for `{{` |
| **Prompt confusion**: the model answers the wrong task, mixes instructions with data, follows instructions found *inside* documents | Data and instructions not separated; injected text; conflicting rules added over time (prompt bloat) | Invocation logs: read the full rendered prompts of failing requests |
| **Format inconsistency**: JSON keys renamed, extra prose, wrong enum values | No schema enforcement; temperature; new model behaves differently | **Schema validation** failure metric |
| Rising token cost per request | Prompt grew (more examples, rules, history) | `InputTokenCount` per prompt version |
| Behaviour differs between environments | Different prompt version, model or parameters per stage | Log `promptVersion`, `modelId` and params on every call |

#### Template testing
- Treat templates as code. Unit-test **rendering** with fixtures for every variable, including null, very long, unicode and injection-like values. Assert that no unresolved `{{…}}` remains and that the size is within the token budget.
- Golden-output tests per template version run in **CodeBuild** before promotion (quality gate).
- Prompt Management: variables are declared per prompt. A request with a missing variable fails or renders blank (verify in docs for the behaviour). Test with the **Test** panel and through `Converse` with `promptVariables`.

#### Diagnosing prompt confusion with CloudWatch Logs
```sql
-- Find unrendered template variables reaching the model
fields @timestamp, modelId, requestMetadata.promptVersion
| filter @message like /\{\{[a-zA-Z_]+\}\}/
| stats count(*) by requestMetadata.promptVersion
```
```sql
-- Compare format-failure rate by prompt version (app emits structured log line)
fields promptVersion, schemaValid
| filter event = "fm_response"
| stats sum(schemaValid = 0) / count(*) * 100 as pctInvalid by promptVersion
```
- The app logs a structured JSON line per call containing `promptVersion`, `modelId`, `templateId`, `schemaValid`, `stopReason`, `inputTokens`, `outputTokens`, `latencyMs` and a correlation ID. This makes the regressions above queryable.
- Use **metric filters** on these logs to create CloudWatch metrics (`SchemaViolations`, `UnrenderedVariables`), then alarms → SNS / incident.
- PII: keep invocation logs encrypted (KMS) with restricted access, or redact prompts before app-level logging (see [Task 3.2](../domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md)).

#### X-Ray prompt observability pipeline
- Instrument the request path with **AWS X-Ray / ADOT (OpenTelemetry)**. Give each stage a subsegment or span:
  `api → auth → retrieve (KB) → rerank → render prompt (template id, version) → bedrock.converse (model, tokens, stopReason) → validate schema → guardrail → respond`.
- Add **annotations** (indexed, filterable: `promptVersion`, `modelId`, `tenant`, `schemaValid`) and **metadata** (not indexed: token counts, chunk IDs). Then filter traces with expressions like `annotation.promptVersion = "7" AND responsetime > 5`.
- The **service map** shows which hop is slow or failing. Trace IDs link to the logs (CloudWatch **Transaction Search** / Application Signals; GenAI observability for agents in **AgentCore** — verify in docs for the current naming).
- Strands Agents and AgentCore emit OpenTelemetry spans for agent cycles, model calls and tool calls. Export them to CloudWatch / X-Ray to trace multi-step reasoning paths.

#### Schema validation to detect format inconsistencies
```python
import json, jsonschema
SCHEMA = {"type": "object", "required": ["category", "confidence"],
          "properties": {"category": {"enum": ["billing", "tech", "other"]},
                         "confidence": {"type": "number", "minimum": 0, "maximum": 1}},
          "additionalProperties": False}

def parse(text):
    try:
        obj = json.loads(text)
        jsonschema.validate(obj, SCHEMA)
        return obj, True
    except (json.JSONDecodeError, jsonschema.ValidationError) as e:
        log.warning({"event": "schema_violation", "err": str(e)[:200]})
        cw.put_metric_data(Namespace="GenAIApp", MetricData=[{"MetricName": "SchemaViolations", "Value": 1}])
        return None, False
```
- On failure:
  1. **retry once** with an error-feedback message ("Your output failed validation: … Return only valid JSON")
  2. fall back to a stronger model or a deterministic default
  3. never pass invalid output downstream
- Prevent the problem at the source:
  - **forced tool use**, where the tool `inputSchema` is the output schema
  - **structured outputs** (`outputConfig.textFormat` with a JSON Schema, on supported models; verify in docs)
  - low temperature
  - an example of the exact output in the prompt
- `stopReason = malformed_model_output` or `malformed_tool_use` means the model itself produced unparseable output. Count these per model version.

#### Model upgrade regressions
- **Pin model IDs** (versioned IDs, not "latest") and prompt **versions** in configuration (AppConfig).
- Upgrade flow:
  1. run the **golden set** on the new model with the existing prompt
  2. if quality drops, create a **model-specific prompt variant**
  3. **canary or shadow** traffic (Step Functions or app-level split)
  4. compare metrics
  5. promote or roll back by flipping the config
- Watch for **deprecation / end-of-life** notices on Bedrock models (legacy → EOL). Plan migrations before the EOL date, or calls will start failing.

#### Systematic prompt refinement workflow (production loop)
1. **Collect** signals: thumbs-down feedback, schema violations, guardrail interventions, low judge scores, escalations.
2. **Triage** weekly: cluster failures by type (Logs Insights, or an LLM-based clustering job).
3. **Fix** in a new prompt version (Skill 5.2.3 loop); add the failing cases to the golden set.
4. **Gate** in CI: regression suite + LLM-as-a-judge thresholds + cost and token budget.
5. **Approve and release** through the Prompt Management versioning / approval workflow ([Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md)).
6. **Monitor** with the dashboards and alarms from [Task 4.3](../domain-4-operational-efficiency-optimization/03-task-4.3-monitoring-observability.md), then loop back to step 1.

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Responses cut off mid-sentence; `stopReason = max_tokens` | Raise `maxTokens`, request a shorter output, or continue in a follow-up turn; branch on `stopReason` |
| `ValidationException`: input is too long for the model | Reduce input: trim or summarise history, fewer or reranked chunks, map-reduce, or a larger-context model |
| Summarise a document larger than the context window | Map-reduce chunk summaries (Step Functions Map + Lambda + Bedrock) |
| Long-context answers miss facts in the middle | Lost in the middle → reorder (key info at start or end), cut context, rerank |
| Chatbot forgets early details in long sessions | Summarising conversation manager / memory; key facts in the system prompt |
| Intermittent 429 `ThrottlingException` at peak | Backoff + jitter, smooth with SQS, quota increase, CRIS, Provisioned Throughput / Reserved tier |
| 503 `ServiceUnavailable` but quota not exceeded | Service capacity → backoff + jitter, cross-Region inference, another Region |
| "On-demand throughput isn't supported" for a new model | Invoke via the inference profile ID (`us.` / `eu.` / `global.`) |
| AccessDenied only when using a CRIS profile | IAM must allow the profile ARN **and** the model ARN in all destination Regions; SCP must not block them |
| AccessDenied on a KB with a customer managed key | KMS key policy / grants for the KB service role |
| AccessDenied for an Anthropic model in a new account | Submit the first-time use-case form (`FTUFormNotFilled`); check model access |
| Requests fail after switching models with InvokeModel | Body is model-family-specific → use the Converse API |
| `ModelNotReadyException` on an imported model after idle | Model scaling from zero; SDK retries up to 5×; keep warm or retry |
| `ModelErrorException` 424 from a Custom Model Import model | Inspect `originalStatusCode`; model container or format issue |
| API Gateway returns 504 after ~29 s on long generations | REST response streaming or async (SQS / WebSocket); or raise the integration timeout |
| Stream sometimes ends without a final message | Handle in-stream error events (`modelStreamErrorException`); check for `messageStop`; retry |
| Connection resets / 70 s first call after idle, behind a NAT gateway | 350 s idle timeout → `tcp_keepalive=True` + OS keep-alive < 350 s |
| Lambda in a private subnet hangs calling Bedrock | Add a `bedrock-runtime` interface VPC endpoint (private DNS, SG 443) |
| Need to filter invocation logs by tenant / feature | `requestMetadata` on Converse + Logs Insights |
| Same input, different answers each time | Lower temperature / topP; tighter instructions; examples |
| Model returns prose around the JSON | Forced tool use or structured output + JSON Schema validation + retry with error feedback |
| Prompt worked on the old model version, regressed on the new one | Golden-set comparison; model-specific prompt variant; canary before switching |
| Need to find which prompt edit caused the regression | Compare Prompt Management versions on the same dataset; diff the versions |
| RAG answers wrong; is it retrieval or generation? | Call **Retrieve** alone and check whether the right chunk is in the top-k |
| Exact product codes not found by the KB | Hybrid search (keyword + vector) |
| Newly uploaded S3 docs not in answers | Ingestion job / sync not run → event-driven `StartIngestionJob` |
| Metadata filter returns nothing | Wrong key or type, sidecar naming, no re-sync after adding metadata |
| Ingestion fails with a dimension mismatch | Index dimension ≠ embedding model dimension → recreate the index / KB |
| Retrieval quality poor after switching the embedding model for queries only | Query and docs must use the same model → re-embed the corpus |
| Tables in PDFs retrieved as garbage | FM parsing / Bedrock Data Automation parser; structure-aware chunking |
| KB answers degrade over months | Content or query drift → refresh the corpus, monitor eval scores, re-evaluate the embedding model |
| Literal `{{variable}}` text appears in model inputs | Template rendering bug → template unit tests; Logs Insights search on the rendered prompts |
| Trace which RAG stage is slow or failing | X-Ray / ADOT spans per stage with annotations (promptVersion, modelId) |

## Flashcards

<details>
<summary><b>Q:</b> A Converse response looks incomplete. Which field should you check first, and which two values point to different fixes?</summary>

`stopReason`. `max_tokens` means the **output** cap was hit: raise `maxTokens` or continue in a follow-up turn. `model_context_window_exceeded` (or a ValidationException about input length) means the **input** is too large: trim, summarise or chunk it.

*D5 · Task 5.2 · stopReason*
</details>

<details>
<summary><b>Q:</b> Need to summarise a 500-page manual with a 200K-token model. Pattern?</summary>

**Map-reduce**: chunk the manual, summarise each chunk in parallel (Step Functions Map + Lambda + Bedrock), then combine the partial summaries in a final call. Use hierarchical summarisation if needed.

*D5 · Task 5.2 · Long documents*
</details>

<details>
<summary><b>Q:</b> Two Bedrock errors share HTTP 429. What are they and how do you handle each?</summary>

`ThrottlingException` means the account quota was exceeded. Use backoff and jitter, a quota increase, CRIS or PT. `ModelNotReadyException` means the model isn't ready yet, for example an imported model warming up. The SDK automatically retries it up to 5 times.

*D5 · Task 5.2 · Error codes*
</details>

<details>
<summary><b>Q:</b> How does a 503 ServiceUnavailableException differ from a 429 ThrottlingException?</summary>

A 503 is service-side capacity pressure, not your quota. Handle it with backoff and jitter, cross-Region inference or another Region. A 429 means **your account quota** was exceeded.

*D5 · Task 5.2 · Error codes*
</details>

<details>
<summary><b>Q:</b> Which Bedrock errors should you NOT retry?</summary>

4xx client errors such as `ValidationException` (400), `AccessDeniedException` (403) and `ResourceNotFoundException` (404). Fix the request, permissions or identifier instead.

*D5 · Task 5.2 · Retry policy*
</details>

<details>
<summary><b>Q:</b> A call to a new Claude model with its base model ID fails with "on-demand throughput isn't supported". Fix?</summary>

Invoke it through a cross-Region **inference profile** ID or ARN (for example the `us.` / `eu.` / `global.` prefix) instead of the bare model ID.

*D5 · Task 5.2 · Inference profiles*
</details>

<details>
<summary><b>Q:</b> IAM allows `bedrock:InvokeModel` on the inference profile, but CRIS calls still get AccessDenied. Two likely causes?</summary>

1. The policy doesn't also allow the **foundation-model ARN in every destination Region**.
2. An **SCP** denies one of the destination Regions. For global profiles, the SCP must allow `aws:RequestedRegion = "unspecified"`.

*D5 · Task 5.2 · CRIS permissions*
</details>

<details>
<summary><b>Q:</b> Which IAM actions do Converse and ConverseStream need?</summary>

Converse needs `bedrock:InvokeModel`. ConverseStream needs `bedrock:InvokeModelWithResponseStream`. To deny inference on a model, deny both.

*D5 · Task 5.2 · IAM actions*
</details>

<details>
<summary><b>Q:</b> A team swaps from a Titan model to Claude with InvokeModel and gets ValidationExceptions. Why, and what's the durable fix?</summary>

The InvokeModel body is model-family-specific, so a Titan body is invalid for Claude. Move to the **Converse** API, which uses a unified message format across models.

*D5 · Task 5.2 · Converse vs InvokeModel*
</details>

<details>
<summary><b>Q:</b> Your ConverseStream code wraps the initial call in try/except, but users still see half-finished answers with no error. What's missing?</summary>

Errors can arrive **inside the stream** after HTTP 200 (`modelStreamErrorException`, `throttlingException` and others). Handle these event types in the loop, and treat a missing `messageStop` as a partial response.

*D5 · Task 5.2 · Streaming errors*
</details>

<details>
<summary><b>Q:</b> Long generations behind API Gateway REST fail at about 29 seconds with 504. Options?</summary>

- Use **REST API response streaming**.
- Go async: SQS plus a job ID and polling, or WebSocket push.
- Request a higher integration timeout (Regional/private APIs).
- Shorten the outputs.

*D5 · Task 5.2 · Timeouts*
</details>

<details>
<summary><b>Q:</b> ECS tasks calling Bedrock through a NAT gateway get connection resets and 70-second first calls after idle periods. Fix?</summary>

The NAT gateway, interface endpoint or NLB drops idle connections after 350 s. Set `tcp_keepalive=True` in the SDK config **and** lower the OS `net.ipv4.tcp_keepalive_time` below 350 s (for example 45). Both settings are required.

*D5 · Task 5.2 · Networking*
</details>

<details>
<summary><b>Q:</b> A Lambda in a private subnet with no NAT times out calling Bedrock. Fix?</summary>

Create an **interface VPC endpoint** for `bedrock-runtime` (plus `bedrock-agent-runtime` for KBs and agents). Turn on private DNS and allow 443 from the Lambda's security group.

*D5 · Task 5.2 · VPC endpoints*
</details>

<details>
<summary><b>Q:</b> How do you tag Bedrock calls so you can later filter invocation logs by tenant and prompt version?</summary>

Pass `requestMetadata` key/value pairs (up to 16) on Converse / ConverseStream, then filter in CloudWatch Logs Insights.

*D5 · Task 5.2 · Logging*
</details>

<details>
<summary><b>Q:</b> What's the right way to refine a misbehaving prompt?</summary>

1. Reproduce from the logs.
2. Classify the failure.
3. Change **one variable**.
4. Test on the failing case **and** the full regression set.
5. Save as a new Prompt Management version.
6. Gate in CI, release through approval, and monitor.

*D5 · Task 5.2 · Refinement*
</details>

<details>
<summary><b>Q:</b> A RAG chatbot gives wrong answers. What's the first diagnostic step?</summary>

Call **Retrieve** on its own with the same query and check whether the correct chunk is in the top-k. If not, it's a retrieval problem (chunking, embeddings, filters, sync, hybrid search). If it is, it's a generation problem (prompt, context ordering, model).

*D5 · Task 5.2 · RAG triage*
</details>

<details>
<summary><b>Q:</b> Users search for exact part numbers and the KB returns unrelated results. Fix?</summary>

Turn on **hybrid search** (keyword plus vector), for example `overrideSearchType: HYBRID` with an OpenSearch Serverless vector store.

*D5 · Task 5.2 · Hybrid search*
</details>

<details>
<summary><b>Q:</b> After adding metadata sidecar files, filtered KB queries return nothing. Likely causes?</summary>

- The sidecar isn't named `<file>.metadata.json`.
- Keys or types don't match the filter.
- No **re-sync** was run after adding the metadata.
- On S3 Vectors, the field was set as non-filterable.

*D5 · Task 5.2 · Metadata filters*
</details>

<details>
<summary><b>Q:</b> KB ingestion fails with a vector dimension error. Cause and fix?</summary>

The index dimension doesn't match the embedding model's output dimension. Recreate the index (or the KB) with the matching dimension. The embedding model and vector store configuration can't be changed after KB creation.

*D5 · Task 5.2 · Embeddings*
</details>

<details>
<summary><b>Q:</b> A team switched the query embedding model to a newer one but kept the existing index. Retrieval got worse. Why?</summary>

Query and document vectors must come from the **same model**, otherwise they live in incomparable vector spaces. Re-embed the whole corpus into a new index and swap it in blue/green.

*D5 · Task 5.2 · Embedding mismatch*
</details>

<details>
<summary><b>Q:</b> Literal `{{customer_name}}` text appears in prompts sent to the model. How do you detect and prevent it?</summary>

Detect it with a Logs Insights regex search for `{{…}}` in the rendered prompts, plus a metric filter and alarm. Prevent it with template unit tests in CI that check every variable resolves, including null cases.

*D5 · Task 5.2 · Template testing*
</details>

<details>
<summary><b>Q:</b> How do you find which stage of a RAG request (retrieve, rerank, render, invoke, validate) causes latency or errors?</summary>

Instrument with **X-Ray / ADOT**: one span per stage, annotations such as `promptVersion` and `modelId` for filtering, and the service map to locate the slow or failing hop.

*D5 · Task 5.2 · X-Ray*
</details>

<details>
<summary><b>Q:</b> Model output intermittently includes prose around the JSON and sometimes renames keys. Best combination of fixes?</summary>

Enforce the schema at generation time with **forced tool use** or **structured output** and low temperature. Validate with JSON Schema in Lambda, emit a `SchemaViolations` metric, and retry once with error feedback.

*D5 · Task 5.2 · Schema validation*
</details>

<details>
<summary><b>Q:</b> Quality dropped with no code change. What should you check?</summary>

- Whether the **model version** or alias changed.
- Whether the **prompt version** changed.
- Whether retrieved content or the KB sync state changed.
- Whether the user query mix drifted.

Compare golden-set scores before and after, using `modelId` and `promptVersion` from the logs.

*D5 · Task 5.2 · Regressions*
</details>

---
[← Back to Domain 5](README.md)
