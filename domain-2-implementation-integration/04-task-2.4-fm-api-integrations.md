[← Back to Domain 2](README.md)

# 🔌 Task 2.4 · Implement FM API integrations

> **Exam domain:** D2 · Implementation and Integration (26%)
>
> - **Skill 2.4.1**: Create flexible model interaction systems
> - **Skill 2.4.2**: Develop real-time AI interaction systems to provide immediate feedback from FMs
> - **Skill 2.4.3**: Create resilient FM systems to ensure reliable operations
> - **Skill 2.4.4**: Develop intelligent model routing systems to optimize model selection

## Key points
- **Five ways to call a Bedrock model**:
  - **InvokeModel** is synchronous and uses a model-specific body.
  - **Converse** is synchronous and uses one unified format for all models (tools, system prompt, documents, guardrails).
  - **ConverseStream** / **InvokeModelWithResponseStream** stream tokens back.
  - **StartAsyncInvoke** is asynchronous and writes its output to S3 (long jobs such as Nova Reel video).
  - **CreateModelInvocationJob** is batch inference: JSONL in S3 in, JSONL in S3 out, at a discounted price.
- **Default to Converse** for new code. You can swap models by changing only the `modelId`, and tool use and multi-turn chat work the same way for every model. Use InvokeModel only when you need a model-specific parameter that Converse doesn't expose (pass it in `additionalModelRequestFields`) or a non-chat model (embeddings, images).
- **Async decoupling**: API Gateway → **SQS** → Lambda worker → results in DynamoDB or S3 → notify the client (WebSocket, SNS or polling). SQS absorbs spikes. Cap the worker concurrency so you stay under the Bedrock TPM/RPM quotas. Use a **DLQ** for poison messages.
- **API Gateway REST** gives you **request validators** (JSON Schema models), **usage plans + API keys** (per-client throttle and quota), stage/method throttling, WAF and caching. **HTTP APIs** are cheaper and simpler but have **no request validation and no usage plans**.
- **Timeouts**: REST integrations default to **29 s**. Regional and private REST APIs can request a higher limit (this may lower your account throttle quota). HTTP API maximum is **30 s**. WebSocket integrations: **29 s**. Lambda maximum: **15 min**. The botocore default **read timeout is 60 s**, so raise it for long generations.
- **Streaming options**:
  - 🎯 **PQ32** **API Gateway REST response streaming** (`responseTransferMode = STREAM`, proxy integrations only): up to **15 min**, beats the 29 s timeout and the 10 MB payload limit, but **no VTL, caching or content encoding**.
  - 🎯 **PQ32** **Lambda function URL** with `RESPONSE_STREAM`.
  - **API Gateway WebSocket API** for two-way chat.
  - **SSE** (`text/event-stream`) for one-way token push.
- **Retryable errors**:
  - 429 **ThrottlingException** (you hit an account quota)
  - 503 **ServiceUnavailable** (capacity, not your quota)
  - 500 **InternalFailure**
  - **ModelNotReadyException** (imported model still loading)
  - **ModelTimeoutException**
  - 529 **overloaded_error**
- **Don't retry** ValidationException, AccessDenied or ResourceNotFound.
- **Exponential backoff with jitter** is the AWS-recommended retry approach. botocore `retries={'mode': 'standard' | 'adaptive', 'max_attempts': N}`. **adaptive** adds client-side rate limiting on top of standard.
- **Degrade gracefully** in this order: retry → Cross-Region Inference profile → fallback model (smaller or different provider) → cached or templated answer → async "we'll email you". See [Task 1.2 · Skill 1.2.3](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md) for CRIS, circuit breakers and AppConfig model switching.
- **X-Ray** traces requests across API Gateway → Lambda → Bedrock / SQS / DynamoDB. Turn on active tracing and instrument the SDK so each FM call shows up as a timed subsegment.
- **Routing options**, from simplest to smartest:
  1. **Static** config map in code or AppConfig.
  2. **Content-based**: classify the request, then a Step Functions **Choice** state picks a specialised FM.
  3. **Bedrock Intelligent Prompt Routing**: managed, picks between **2 models in the same family** using predicted quality, with a fallback model and a `responseQualityDifference` setting. **English only**.
  4. **Metric-based**: CloudWatch latency, error or cost data drives the routing weights.
  5. **API Gateway mapping templates** or stage variables, for edge-level request rewriting.

> [!WARNING]
> **🎯 Practice question 32 — API must stream, enforce token limits, retry on timeouts/partial responses, LEAST overhead**
>
> - **Answer key:** **API Gateway HTTP API → Lambda** (using **Lambda response streaming** through a function URL / HTTP proxy integration) → Bedrock `ConverseStream`. Enforce **token limits (`maxTokens`, input size checks) in Lambda**, and put **retry logic** for model timeouts in Lambda, with API Gateway timeouts as the outer bound.
> - ❌ **WebSocket API + ECS inference server:** you run the containers yourself, and task restarts work per task, not per request.
> - ❌ **HTTP API straight to Bedrock with client polling + frontend token limits:** not real streaming, and limits aren't enforced on the server.
> - ⚠️ **The answer key is out of date:** its rationale says *REST APIs can't stream Lambda responses*. API Gateway now has **REST API response streaming** (`responseTransferMode: STREAM`, see the bullet above, checked against AWS docs). If an exam option contrasts REST and HTTP for streaming, **go with the exam's framing (HTTP API + Lambda streaming)**, but know that REST streaming exists now.

## Notes

### Skill 2.4.1 — Flexible model interaction systems

> *"Create flexible model interaction systems (for example, by using Amazon Bedrock APIs to manage synchronous requests from various compute environments, language-specific AWS SDKs and Amazon SQS for asynchronous processing, API Gateway to provide custom API clients with request validation)."*

#### Bedrock runtime API surface

| API | Mode | Request format | Use when | Notes |
| --- | --- | --- | --- | --- |
| `InvokeModel` | Sync | **Model-native JSON body** (differs per provider) | Embeddings, image models, provider-specific features | `contentType`/`accept` = `application/json`; body is bytes |
| `Converse` | Sync | **Unified** `messages` / `system` / `inferenceConfig` / `toolConfig` | Chat, tool use, multi-turn, swapping models | `additionalModelRequestFields` for model-specific params (e.g., `top_k`) |
| `InvokeModelWithResponseStream` | Stream | Model-native | Streaming with native body | Event stream of `chunk` bytes |
| `ConverseStream` | Stream | Unified | Streaming chat UX | Typed events (see 2.4.2) |
| `StartAsyncInvoke` / `GetAsyncInvoke` / `ListAsyncInvokes` | Async | Model-native, `outputDataConfig` → S3 | Long-running generation (e.g., **Nova Reel video**) | Returns `invocationArn`; poll or react to S3 event |
| `CreateModelInvocationJob` (control plane `bedrock`) | **Batch** | JSONL in S3: `{"recordId": "...", "modelInput": {...}}` | Large offline workloads (classify 1M tickets overnight) | Discounted vs on-demand (≈50% — verify in docs); has min record count and job quotas — verify in docs |
| `CountTokens` | Sync | Same as Converse/InvokeModel | Pre-flight token budgeting | Model support varies — verify in docs |

- Two clients in boto3: `bedrock` (control plane: models, jobs, guardrails, provisioned throughput, prompt routers) and `bedrock-runtime` (data plane: invoke/converse). Agents use `bedrock-agent-runtime`; AgentCore has its own clients.
- `modelId` can be a **foundation model ID**, an **inference profile ID/ARN** (e.g., `us.anthropic...` for CRIS, or an application inference profile for cost tags), a **provisioned throughput ARN**, an imported/custom model ARN, a **prompt router ARN**, or a **Prompt Management prompt ARN** (Converse only).

```python
import boto3
from botocore.config import Config

brt = boto3.client(
    "bedrock-runtime",
    region_name="us-east-1",
    config=Config(
        retries={"mode": "adaptive", "max_attempts": 6},
        read_timeout=300,        # default 60 s is too short for long outputs
        connect_timeout=10,
        tcp_keepalive=True,      # see gotcha on NAT/VPCE 350 s idle timeout
    ),
)

resp = brt.converse(
    modelId="us.amazon.nova-pro-v1:0",           # CRIS inference profile
    system=[{"text": "You are a concise support assistant."}],
    messages=[{"role": "user", "content": [{"text": "Reset my MFA device"}]}],
    inferenceConfig={"maxTokens": 512, "temperature": 0.2},
)
text = resp["output"]["message"]["content"][0]["text"]
usage = resp["usage"]            # inputTokens / outputTokens / totalTokens
stop  = resp["stopReason"]       # end_turn | max_tokens | tool_use | guardrail_intervened | ...
```

#### Calling Bedrock from different compute environments

| Compute | Pattern | Watch out for |
| --- | --- | --- |
| **Lambda** | SDK in handler; client created **outside** handler for reuse | 15 min max; concurrency vs Bedrock quotas; package newer boto3 if runtime's bundled SDK lacks new APIs |
| **ECS/Fargate, EKS** | Long-running services, streaming servers, MCP servers | Task role (IRSA / EKS Pod Identity); NAT/VPCE idle timeout 350 s |
| **EC2** | Instance profile | Same keep-alive gotcha |
| **On-prem / other cloud** | IAM Roles Anywhere (X.509 → temp creds), or federation | Never long-lived access keys |
| **Browser / mobile** | **Don't call Bedrock directly**. Front with API Gateway + Cognito, or Cognito identity pool temp creds with tightly scoped role | Keeps prompts, guardrails, quotas server-side |
| **Step Functions** | Optimised integration `arn:aws:states:::bedrock:invokeModel` | Payload limit 256 KB in state; use S3 for large inputs/outputs |
| **Private network** | **Interface VPC endpoint** (`com.amazonaws.<region>.bedrock-runtime`) via PrivateLink + endpoint policy | Private DNS; endpoint policy can restrict models |

- **Language SDKs** (Python boto3, JavaScript v3 `@aws-sdk/client-bedrock-runtime`, Java v2, .NET, Go v2, etc.) all expose the same operations. They all sign requests with SigV4, retry, and parse event streams for you. Use the SDK rather than calling the REST API by hand.
- **Bedrock API keys** (short- or long-term, bearer token) exist for quick exploration. For production, AWS recommends IAM roles and temporary credentials. (verify in docs for the current guidance on API keys.)

#### Asynchronous processing with SQS

Why go async: generation takes longer than your front-door timeout, traffic spikes past Bedrock quotas, the caller doesn't need an immediate answer (document summaries, email drafting), or you want to smooth out TPM usage.

```
Client ──POST /jobs──► API Gateway ──(AWS service integration: SendMessage)──► SQS queue ──► Lambda worker ──► Bedrock
   ▲                         │ returns 202 + jobId                          │  DLQ (maxReceiveCount)        │
   │                         ▼                                              ▼                               ▼
   └── poll GET /jobs/{id} / WebSocket push / SNS / EventBridge ◄── DynamoDB (status, result) / S3 (large output)
```

- **API Gateway → SQS directly** (REST AWS service integration, no Lambda in front) is cheaper and can't hit Lambda cold starts or concurrency limits. Use a mapping template to turn the body into `Action=SendMessage&MessageBody=...`.
- **Lambda event source mapping (ESM) settings**:
  - `BatchSize`
  - `MaximumBatchingWindowInSeconds`
  - **`ScalingConfig.MaximumConcurrency`** (2–1000), which caps concurrent workers so you don't flood Bedrock
  - `ReportBatchItemFailures`, so you retry only the failed messages
- **Visibility timeout** must be **at least 6× the function timeout**, which is the AWS guidance for Lambda ESMs. If it's shorter, a message can be processed twice.
- **DLQ / redrive policy**: after `maxReceiveCount` attempts the message moves to the DLQ. Alarm on DLQ depth.
- **Idempotency**: SQS standard queues are **at-least-once**, so the same message can arrive twice. Store a `jobId` and use a DynamoDB conditional write (or Powertools idempotency) so you don't generate or bill twice. Use **FIFO** queues when per-conversation order matters (`MessageGroupId` = conversationId). FIFO throughput is lower.
- **Priority**: use separate queues (premium vs free tier) with different worker concurrency.
- **Bedrock batch inference vs SQS workers**: batch suits huge, latency-tolerant jobs where cost matters most (results in hours). SQS workers suit near-real-time async work (results in seconds to minutes) with per-item control.
- **StartAsyncInvoke** is a Bedrock-native async call that writes to S3. Use it when the model *requires* async (video generation). Use SQS when *you* want async around any synchronous model.

#### API Gateway as the custom client front door

| Feature | REST API | HTTP API | WebSocket API |
| --- | --- | --- | --- |
| Request validation (JSON Schema models) | ✅ | ❌ | ✅ (route request models) |
| Usage plans + API keys | ✅ | ❌ | ❌ |
| Per-stage / per-method throttling | ✅ | ✅ (route-level) | ✅ (route-level) |
| Mapping templates (VTL) | ✅ | ❌ (parameter mapping only) | ✅ |
| AWS WAF | ✅ | ❌ (front with CloudFront + WAF) | ❌ |
| Caching | ✅ | ❌ | ❌ |
| Response streaming | ✅ (`STREAM`, proxy only) | ❌ | Bi-directional by nature |
| Private endpoint | ✅ | ❌ (VPC link to private backends only) | ❌ |
| Authorizers | IAM, Cognito, Lambda | IAM, **JWT**, Lambda | IAM, Lambda |
| Integration timeout | 29 s default; Regional/private can be raised | 30 s max | 29 s |

**Request validation** rejects bad input *before* you pay for any tokens:

```json
// Model (JSON Schema draft 4) attached to POST /chat with a "Validate body" request validator
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "required": ["sessionId", "message"],
  "properties": {
    "sessionId": { "type": "string", "pattern": "^[a-f0-9-]{36}$" },
    "message":   { "type": "string", "minLength": 1, "maxLength": 8000 },
    "modelTier": { "type": "string", "enum": ["fast", "balanced", "best"] }
  },
  "additionalProperties": false
}
```

- Validators can check the **body**, the **parameters** (required query strings and headers), or both. If validation fails, API Gateway returns **400** and the backend is never called.
- A `maxLength` on the prompt is a cheap first guard on token cost. Do a precise token check in Lambda (`CountTokens`, or roughly 4 characters per token for English).
- 🎯 **PQ58** **Usage plans**: an API key belongs to a usage plan, which sets a **throttle** (rate + burst) and a **quota** (requests per day, week or month) per client. ⚠️ API keys **identify** clients; they **don't authenticate** them. Pair them with IAM, Cognito or a Lambda authorizer.
- **Account limit**: **10,000 RPS** per Region, with **5,000 burst** (token bucket). Some newer Regions start at 2,500/1,250. You can raise it through Service Quotas.
- **Request counts ≠ tokens**: API Gateway throttles requests, but Bedrock quotas are measured in **tokens per minute** and requests per minute. To budget tokens per tenant, track usage yourself (DynamoDB counter, reading `usage` from each response), or use an application inference profile per tenant together with CloudWatch metrics.
- The **SDK generator** produces typed client SDKs (JavaScript, Java, Android, iOS) from a REST API stage. You can also export an **OpenAPI** spec (more on this in [Task 2.5](05-task-2.5-app-integration-dev-tools.md)).

> [!IMPORTANT]
> **🎯 Practice question 58 — premium vs standard tiers, protect the critical document-analysis endpoint from 429s, audit metrics**
>
> - **Separate usage plans** for premium and standard, each with its **own throttle and quota**. Give each customer an **API key** in the right plan.
> - **Method-level throttling inside each usage plan** → **higher limits for the document-analysis method**. Image generation is in the premium plan only.
> - API Gateway sends **throttling and 4XX metrics to CloudWatch** automatically, ready for audits.
> - ❌ **One usage plan / one key + stage-level throttling:** no tiers and no per-method priority.
> - ❌ **WAF rate-based rules:** blocking for DDoS protection, not tiered throttling.
> - ❌ **Client-side throttling + custom metrics:** lots of custom work, and unreliable.

### Skill 2.4.2 — Real-time AI interaction systems 🎯 PQ24

> *"Develop real-time AI interaction systems to provide immediate feedback from FM (for example, by using Amazon Bedrock streaming APIs for incremental response delivery, WebSockets or server-sent events to generate text in real time, API Gateway to implement chunked transfer encoding)."*

> [!WARNING]
> **🎯 Practice question 24 — live call assist: speech-to-text <500 ms, GenAI suggestions, supervisors rate them live, store everything**
>
> - **Amazon Transcribe *streaming* API** with **small, uniform audio chunks (~100 ms)** → lowest-latency real-time transcription.
> - **Bedrock `InvokeModelWithResponseStream`** (or `ConverseStream`) → tokens arrive **as they're generated**, so suggestions appear immediately.
> - **DynamoDB** → low-latency writes for **live supervisor ratings**. Store the call audio, transcripts and suggestions in **S3** for the audit record.
> - ❌ **Transcribe batch:** post-call only.
> - ❌ **SQS between steps:** queuing adds delay.
> - ❌ **Non-streaming `InvokeModel`:** waits for the full response.
> - ❌ **Bedrock batch inference:** asynchronous, not real time.
> - ❌ **CloudWatch for ratings:** it's logs and metrics, not an application data store.

#### Why stream
- **Time to first token (TTFT)** is what users feel. A 20-second answer that starts after 400 ms feels fast; one that appears all at once after 20 s feels broken.
- Streaming also gets around front-door timeouts (if the transport supports it) and lets users cancel early, which saves output tokens.

#### Bedrock streaming APIs

`ConverseStream` sends typed events in this order:

| Event | Contains |
| --- | --- |
| `messageStart` | `role` |
| `contentBlockStart` | Start of a block, for example a `toolUse` with name and id |
| `contentBlockDelta` | `delta.text`, `delta.toolUse.input` (partial JSON string) or `delta.reasoningContent` |
| `contentBlockStop` | Block finished |
| `messageStop` | `stopReason` |
| `metadata` | `usage` (tokens), `metrics.latencyMs`, guardrail `trace` |
| Error events | e.g. `modelStreamErrorException`, `throttlingException` **mid-stream** |

```python
stream = brt.converse_stream(
    modelId="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
    messages=[{"role": "user", "content": [{"text": prompt}]}],
    inferenceConfig={"maxTokens": 1024},
)
for event in stream["stream"]:
    if "contentBlockDelta" in event:
        delta = event["contentBlockDelta"]["delta"]
        if "text" in delta:
            yield delta["text"]                    # push to client immediately
    elif "messageStop" in event:
        stop_reason = event["messageStop"]["stopReason"]
    elif "metadata" in event:
        usage = event["metadata"]["usage"]         # log tokens for cost tracking
```

- `InvokeModelWithResponseStream` returns `chunk.bytes` in the provider's own format, which you decode per model. Converse streaming is the same for every model.
- **Tool use while streaming**: the tool input arrives as **partial JSON** across several deltas. Buffer it until `contentBlockStop`, then parse it.
- **Guardrails with streaming**: `guardrailConfig.streamProcessingMode` can be `sync` or `async`:
  - **sync** (default) buffers chunks and checks them before sending, which adds some latency but nothing unsafe reaches the user.
  - **async** sends chunks straight away and checks in the background, so it's faster but unsafe text can reach the user before the guardrail intervenes.
- **Errors can arrive mid-stream** after you've already sent HTTP 200. Your client protocol needs an in-band error event (for example an SSE `event: error`).

#### Transport options to the client

| Transport | Direction | AWS implementation | Best for | Limits / gotchas |
| --- | --- | --- | --- | --- |
| **SSE** (`text/event-stream`) | Server → client | REST API response streaming, Lambda function URL streaming, ALB → ECS | Token streaming chat UI, one-way | Uses plain HTTP; browser `EventSource` only does GET, so use `fetch` + `ReadableStream` for POST |
| **Chunked transfer encoding** | Server → client | REST API `STREAM` mode, ALB/ECS, Lambda URL | Any incremental HTTP body | Client must read the body as a stream |
| **WebSocket** | **Bi-directional** | API Gateway WebSocket API (`$connect`, `$disconnect`, `$default`, custom routes) + `@connections` POST-back; AppSync Events / AppSync subscriptions | Interactive agents, interrupt or cancel, server push of async job results | Integration timeout **29 s** per route call (return quickly, then stream back through `PostToConnection`); connection max **2 h**, idle **10 min**; message size limits (verify in docs); you manage the connection table (DynamoDB) |
| **Polling** | Client pull | REST GET /jobs/{id} | Simple async jobs | Wastes requests; adds latency |

#### API Gateway REST response streaming (checked against the docs)
- Set the integration's **`responseTransferMode` to `STREAM`**. The default is `BUFFERED`.
- Works only with **`HTTP_PROXY` and `AWS_PROXY`** integrations (Lambda proxy, plus private integrations over HTTP_PROXY). **REST APIs only**, all endpoint types.
- A stream can last **up to 15 minutes**. The **idle timeout** is **5 min** for Regional and private endpoints and **30 s** for edge-optimised ones. To get a longer idle timeout, put your own CloudFront distribution in front and raise its response timeout.
- You get past the **29 s timeout** and the **10 MB payload limit** without asking for a quota increase. After the first 10 MB, bandwidth is capped at **2 MB/s**.
- **Not supported** in STREAM mode: **endpoint caching**, **content encoding** (compression) and **VTL response transformation**. **Request** streaming isn't supported either.
- If the client disconnects, **the Lambda function may keep running**, so check for cancellation and stop calling Bedrock.
- Response streaming costs extra (see API Gateway pricing).

#### Lambda response streaming 🎯 PQ50
- Set the **Lambda function URL** `InvokeMode` to **`RESPONSE_STREAM`** (the default is `BUFFERED`). You can also invoke with `InvokeWithResponseStream`, or front it with API Gateway in STREAM mode.
- **Node.js managed runtimes** support it natively through `awslambda.streamifyResponse()`. Other languages need a custom runtime or the **Lambda Web Adapter** (verify in docs for the current runtime list).
- Lambda can stream bigger responses than it can buffer (6 MB). The streaming limit is a soft limit (verify in docs for the current number).

> [!IMPORTANT]
> **🎯 Practice question 50 — responses over 4 MB taking 15–20 s, first response in under 10 s, keep records 10 years**
>
> - **Lambda response streaming** (function URL `RESPONSE_STREAM`) + Bedrock streaming → the **first tokens arrive within seconds**, even though the full answer takes 15–20 s. Streamed responses can also be larger than the 6 MB buffered-response limit.
> - **Lambda timeout of 30 s**, which leaves headroom over 20 s of processing.
> - **Store the complete responses in S3** with a **lifecycle policy (for example to Glacier) for 10-year retention**. Add **Object Lock** if records must be immutable.
> - ❌ **SQS async:** the customer gets nothing until they poll.
> - ❌ **API Gateway polling + ElastiCache:** no streaming, so the first byte waits the full 15–20 s.
> - ❌ **Streaming with a 15 s timeout:** the function is **killed before it finishes**.

```javascript
// Node.js Lambda — SSE over function URL (InvokeMode: RESPONSE_STREAM)
export const handler = awslambda.streamifyResponse(async (event, responseStream) => {
  responseStream = awslambda.HttpResponseStream.from(responseStream, {
    statusCode: 200, headers: { "Content-Type": "text/event-stream" } });
  const out = await brt.send(new ConverseStreamCommand({ modelId, messages }));
  for await (const ev of out.stream) {
    const t = ev.contentBlockDelta?.delta?.text;
    if (t) responseStream.write(`data: ${JSON.stringify({ t })}\n\n`);
  }
  responseStream.write("event: done\ndata: {}\n\n");
  responseStream.end();
});
```

#### WebSocket chat pattern

```
Browser ⇄ API GW WebSocket ── $connect (Lambda authorizer, store connectionId in DynamoDB)
                         └── sendMessage route → Lambda (async invoke / SQS) → ConverseStream
                                 for each delta → apigatewaymanagementapi.post_to_connection(ConnectionId, Data)
                         └── $disconnect → delete connectionId
```

- The route Lambda should **hand off quickly**: call another Lambda asynchronously or queue the work. Otherwise the 29 s route timeout applies. The worker then streams tokens back with `PostToConnection`. A `GoneException` (410) means the client disconnected, so stop generating.
- Batch tokens into small groups (for example every 50 ms or every few tokens) to cut down on `PostToConnection` calls.

#### Choosing a transport

| Requirement | Pick |
| --- | --- |
| Simple token streaming for a chat UI, keep your API Gateway auth, WAF and usage plans | **REST API + Lambda proxy, `STREAM` mode + SSE** |
| Server-to-server or minimal infrastructure | **Lambda function URL `RESPONSE_STREAM`** (IAM or no auth; put CloudFront + OAC in front if needed) |
| Full duplex: user can interrupt, voice or agent sends progress events, push results of async jobs | **API Gateway WebSocket** (or AppSync Events) |
| Long-lived streaming server, custom protocols, gRPC | **ECS/Fargate behind ALB** (configure the idle timeout) |
| Real-time speech-to-speech | Bidirectional streaming API (`InvokeModelWithBidirectionalStream`, for example Nova Sonic). verify in docs |

### Skill 2.4.3 — Resilient FM systems

> *"Create resilient FM systems to ensure reliable operations (for example, by using the AWS SDK for exponential backoff, API Gateway to manage rate limiting, fallback mechanisms for graceful degradation, AWS X-Ray to provide observability across service boundaries)."*

#### Bedrock error codes and how to handle them (checked against the docs)

| Error | HTTP | Meaning | Retry? | Action |
| --- | --- | --- | --- | --- |
| `ThrottlingException` | 429 | Exceeded **your account quota** (RPM/TPM) | ✅ backoff + jitter | Spread load (CRIS), request a quota increase, Provisioned Throughput / Reserved tier, client-side rate limiting |
| `ServiceUnavailable` | 503 | **Service capacity**, not your quota | ✅ | Backoff, CRIS, try another Region |
| `overloaded_error` | 529 | Model temporarily lacks serving capacity | ✅ | Backoff + jitter; honour **`Retry-After`**; avoid synchronised retries |
| `InternalFailure` / `InternalServerException` | 500 | Server error | ✅ | Backoff |
| `ModelNotReadyException` | 429 (verify) | Imported / custom model not loaded yet (scaled to zero) | ✅ | The SDK retries this automatically (verify in docs for the count); warm the model up first |
| `ModelTimeoutException` | 408 | Model took too long | ✅ (maybe with a shorter output) | Lower `maxTokens`, stream, or go async |
| `ModelErrorException` | 424 | Model-side processing error | ⚠️ limited | Log, fallback |
| `ModelStreamErrorException` | — | Error **during** a stream | ⚠️ | Resend or fall back; tell the client in-band |
| `ValidationException` | 400 | Bad input (too many tokens, bad params) | ❌ | Fix the request; truncate or summarise context |
| `AccessDeniedException` | 403 | IAM/SCP or model access not granted | ❌ | Fix permissions or model access |
| `ResourceNotFoundException` | 404 | Wrong model ID or Region | ❌ | **Fall back** to another model; keep the model catalogue in sync |
| `ServiceQuotaExceededException` | 400 | Hard quota | ❌ | Request an increase |

**Network gotcha (checked against the docs)**: NAT gateways, interface VPC endpoints and NLBs **silently drop TCP connections idle for more than 350 s**. Long streams, extended thinking, or a pooled connection reused after a quiet spell can reset or hang. The fix needs **both** `Config(tcp_keepalive=True)` **and** an OS keep-alive below 350 s (for example `sysctl net.ipv4.tcp_keepalive_time=45`).

#### Retries in the SDK

| botocore `retries.mode` | Behaviour |
| --- | --- |
| `legacy` | Older default in many SDK versions; limited set of retryable errors |
| **`standard`** | Exponential backoff **with jitter**, broader retryable errors (throttles, transient 5xx), default `max_attempts` = **3** (counts the first call), retry quota (token bucket) to stop retry storms |
| **`adaptive`** | Standard + **client-side rate limiting** that slows the send rate when throttled. Best for one busy client hitting a throttled API. AWS calls it experimental, so behaviour may change |

```python
Config(retries={"mode": "standard", "max_attempts": 5})
# or env: AWS_RETRY_MODE=adaptive  AWS_MAX_ATTEMPTS=8
```

- **Full jitter**: `sleep = random(0, min(cap, base * 2**attempt))`. Jitter stops a whole fleet from retrying at the same moment ("thundering herd").
- **Retry budget and timeout chain**: client timeout > API Gateway timeout > Lambda timeout > SDK read timeout × attempts. If the SDK keeps retrying after API Gateway has already returned 504, the work is wasted. Give each inner layer a shorter timeout than the layer outside it.
- **Retry only idempotent work**, or use idempotency keys. Retrying an agent turn that already ran a side-effecting tool can repeat that action.
- **Don't retry 4xx** errors other than 429.
- **Step Functions** `Retry` on a Bedrock task: `ErrorEquals` (for example `Bedrock.ThrottlingException`; verify in docs for the exact names, or use `States.TaskFailed`), `IntervalSeconds`, `BackoffRate`, `MaxAttempts`, `MaxDelaySeconds`, `JitterStrategy: FULL`. Then add a `Catch` that goes to a fallback state.

#### Rate limiting at the edge (API Gateway)
- **Layers**:
  - account limit (10k RPS / 5k burst default)
  - stage and method throttling
  - **usage plan** per API key (rate, burst, quota)
  - **AWS WAF rate-based rules** (per IP or per header/key; REST only)
- Callers above the limit get **429 Too Many Requests**. Tell clients to back off.
- Protect Bedrock quotas **in front** of the model: edge throttling + SQS buffer + capped Lambda concurrency (reserved concurrency or ESM `MaximumConcurrency`) + client `adaptive` mode.
- Token-aware limiting: keep a per-tenant token bucket in DynamoDB or ElastiCache, check it in a Lambda authorizer or at the start of the handler, and update it from `usage` after each response.

#### Fallbacks and graceful degradation

```
try primary model (Converse, CRIS profile)
  └─ throttled/5xx after N retries → secondary model (cheaper/other provider; same Converse call, different modelId)
       └─ fails → semantic cache hit? (ElastiCache / OpenSearch / MemoryDB vector cache) → return cached answer
            └─ miss → static/templated response ("Here are help articles…") or queue for async reply
```

- Converse makes switching models a one-line change. Keep the fallback chain in **AppConfig** so you can change it without redeploying ([Task 1.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md)).
- Degrade **features**, not just models: turn off reranking, use fewer retrieved chunks, lower `maxTokens`, skip optional tool calls.
- 🎯 **PQ56** Use a **circuit breaker** (DynamoDB or Step Functions state) to skip a failing model for a cooldown period instead of retrying it every time. Task 1.2 covers this in detail.
- Use **health signals** to decide when to fail over: CloudWatch `InvocationThrottles`, `InvocationServerErrors`, `InvocationLatency` and `InvocationClientErrors` from the `AWS/Bedrock` namespace.

> [!IMPORTANT]
> **🎯 Practice question 56 — global medical imaging app: MOST resilient, intelligent routing, no degradation when latency varies (choose 2)**
>
> - ✅ **Step Functions circuit breaker** (state in DynamoDB, Lambda checks) watches **Bedrock response times**. When thresholds are breached, it **routes to alternative models** and **degrades gracefully** to rule-based processing at peak.
> - ✅ **Bedrock cross-Region inference** → requests are **routed across Regions** automatically for throughput and availability, with the same FMs in several Regions.
> - ❌ **Route 53 health checks + weighted DNS:** DNS level, so it can't react to FM latency or apply circuit-breaker logic.
> - ❌ **SQS DLQs + EventBridge:** **reactive**, only after a failure.
> - ❌ **API Gateway with several backends + Lambda backoff:** retries alone, no proactive circuit breaking.

#### Observability across service boundaries (X-Ray)
- Turn on **active tracing** for API Gateway (stage setting), Lambda (`TracingConfig: Active`) and Step Functions. Instrument the SDK, for example with `aws_xray_sdk.core.patch_all()` / `patch(['boto3'])`, Powertools Tracer, or **ADOT / OpenTelemetry**. Each `bedrock-runtime` call then appears as a **subsegment** with its latency and errors.
- The **trace map** shows client → API GW → Lambda → Bedrock / DynamoDB / SQS, so you can find which hop adds latency or throws errors. SQS → Lambda traces are linked.
- Add **annotations** (indexed: `modelId`, `tenantId`, `route`) and **metadata** (not indexed: token counts). You can then filter traces by model.
- Use **CloudWatch Application Signals** and **Transaction Search** for service-level SLOs on top of X-Ray spans. For GenAI- and agent-specific views, use **CloudWatch GenAI observability** and AgentCore Observability (Task 4.x / 5.x).
- **Bedrock model invocation logging** goes to CloudWatch Logs and/or S3 and records the full prompt and response (an account-level setting). X-Ray gives you timing; invocation logs give you content. You usually need both.

### Skill 2.4.4 — Intelligent model routing systems

> *"Develop intelligent model routing systems to optimize model selection (for example, by using application code to implement static routing configurations, Step Functions for dynamic content-based routing to specialized FMs, intelligent model routing based on metrics, API Gateway with request transformations for routing logic)."*

#### Routing approaches compared

| Approach | Decision based on | Effort | Flexibility | Example |
| --- | --- | --- | --- | --- |
| **Static config** | Task type, tenant, tier | Low | Change config, not code | `{"summarise":"nova-lite","code":"claude-sonnet"}` in AppConfig |
| **Rule-based content routing** | Keywords, length, language, attachments | Low–Med | Brittle | `len(prompt) > 20k` → long-context model |
| **Classifier-based (Step Functions)** | ML or LLM classification of intent or complexity | Med | High | Comprehend custom classifier or Nova Micro labels the intent → Choice → specialist FM |
| **Bedrock Intelligent Prompt Routing** | Managed prediction of response quality | **Lowest** | Same family only, 2 models | Nova Lite ↔ Nova Pro, Claude Haiku ↔ Sonnet |
| **Metric-based / adaptive** | Live latency, errors, cost, user feedback | High | Highest | EventBridge + Lambda adjust weights in AppConfig when p95 latency breaches |
| **Edge transformation** | Header, path, API key, stage variable | Low | Coarse | API GW mapping template injects `modelId` per usage plan |

#### Static routing in application code
- Keep a model map **outside the code** (AppConfig hosted configuration or feature flags; Parameter Store works for simple cases). Cache it in the Lambda (the AppConfig Lambda extension polls and caches locally).
- Use **application inference profiles** per route or tenant. You get cost allocation tags and per-profile CloudWatch metrics, and the route stays stable even when the underlying model changes.

```json
{
  "routes": {
    "classify":  { "modelId": "us.amazon.nova-micro-v1:0", "maxTokens": 50 },
    "summarise": { "modelId": "us.amazon.nova-lite-v1:0",  "maxTokens": 800 },
    "reason":    { "modelId": "us.anthropic.claude-sonnet-4-5-20250929-v1:0", "maxTokens": 4000 }
  },
  "fallback": "us.amazon.nova-pro-v1:0"
}
```

#### Dynamic content-based routing with Step Functions

```json
{
  "StartAt": "ClassifyIntent",
  "States": {
    "ClassifyIntent": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "amazon.nova-micro-v1:0",
        "Body": { "messages": [{ "role": "user", "content": [{ "text.$": "States.Format('Classify as CODE, LEGAL or GENERAL: {}', $.query)" }]}],
                  "inferenceConfig": { "maxTokens": 5 } }
      },
      "ResultSelector": { "label.$": "$.Body.output.message.content[0].text" },
      "ResultPath": "$.route",
      "Next": "Route"
    },
    "Route": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.route.label", "StringMatches": "*CODE*",  "Next": "CodeModel"  },
        { "Variable": "$.route.label", "StringMatches": "*LEGAL*", "Next": "LegalRAG"   }
      ],
      "Default": "GeneralModel"
    }
  }
}
```

- The classifier should be **cheap and fast** (Nova Micro, Haiku, or Comprehend custom classification / Comprehend intent detection). Its cost and latency are added to every request.
- Each branch can run a different model, a RAG flow (Knowledge Base `RetrieveAndGenerate`) or an agent. Add `Retry` and `Catch` on each branch so it falls back to the general model.
- **Express workflows** suit high-volume, short (≤5 min) synchronous routing. Start them synchronously from API Gateway with `StartSyncExecution`. **Standard workflows** suit long, auditable runs, human approval steps and exactly-once execution.
- Also called the **"router" agent design pattern**. You can build the same thing in code with Strands or LangGraph, or as Bedrock Flows condition nodes (Task 1.6).

#### Bedrock Intelligent Prompt Routing (checked against the docs)
- One **serverless endpoint** (the router ARN) that sends each request to the **better of the models in the same family**, based on **predicted response quality**, to balance quality against cost.
- **Default routers** are pre-configured for selected Nova and Llama families, depending on the Region. **Configured routers** (`CreatePromptRouter`) let you pick **exactly two models in the same family**, a **fallback model** and the **routing criteria** `responseQualityDifference`.
- Example: fallback = Nova Lite and quality difference = 10%. The router only sends a request to Nova Pro if Nova Pro's predicted quality is **more than 10% better**. A higher threshold sends more traffic to the cheap fallback model.
- Call it by passing the **router ARN as `modelId`** to Converse or InvokeModel. The **response tells you which model was used**, so log it for cost attribution.
- **Limitations**:
  - **optimised for English prompts only**
  - **can't learn from your application-specific performance data**
  - may not route well for specialised use cases
  - same family only, so no cross-provider routing
- Exam cue: *"route simple prompts to a cheaper model and complex ones to a stronger model, minimal operational overhead"* → **Intelligent Prompt Routing**. If the question needs cross-provider routing, your own quality signals, or non-English prompts → build a **custom router** (classifier + Step Functions or code).

#### Metric-based routing
- Inputs: CloudWatch Bedrock metrics (`InvocationLatency`, throttles, errors per `ModelId`), cost per request (from `usage`), and quality signals (thumbs-up/down feedback, online evaluation scores).
- Loop: a **CloudWatch alarm** or scheduled **EventBridge** rule triggers a Lambda that recomputes routing weights → **AppConfig deployment** (gradual, with rollback on alarm) → routers pick up the new weights.
- Use **weighted / canary** routing to A/B test a new model on, say, 5% of traffic, then compare quality, latency and cost before promoting it.
- **Latency-aware routing**: if the primary model's p95 is above its SLO, send interactive traffic to the faster model and keep batch traffic on the primary.

#### API Gateway with request transformations
- **REST API mapping templates (VTL)** on a non-proxy integration can rewrite the request body, for example to inject `modelId` from a header, stage variable or `$context.identity.apiKey` (usage plan tier), or to build a Bedrock body for a **direct AWS service integration** (API Gateway → `bedrock-runtime` InvokeModel with no Lambda).
- **Stage variables** let `dev` and `prod` stages point at different models or Lambda aliases. Use **canary release** on a stage to send a percentage of traffic to a new deployment.
- **HTTP API parameter mapping** can add, overwrite or remove headers, query strings and paths, but it can't transform the body.
- **Custom domain routing rules** can route by header or path to different APIs or stages. This is a newer feature, so verify in docs.
- ⚠️ VTL templates **can't** be used with response `STREAM` mode, and routing logic in VTL is hard to test. Use it for simple header- or tier-based routing; put content-aware routing in Lambda or Step Functions.

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Same code must work across Claude, Nova, Llama; swap model with config only | **Converse API** + `modelId` from AppConfig |
| Need provider-specific parameter not in Converse | `additionalModelRequestFields` (or InvokeModel) |
| Generate embeddings / images | **InvokeModel** with model-native body |
| Video generation, output takes minutes | **StartAsyncInvoke** → S3 |
| Score 2 million records overnight, minimise cost | **Batch inference** (`CreateModelInvocationJob`, JSONL in S3) |
| Spiky traffic overwhelms Bedrock quota; responses can be delayed | **API GW → SQS → Lambda** (ESM MaximumConcurrency) + DLQ |
| Duplicate generations from SQS redelivery | Idempotency key + DynamoDB conditional write; check visibility timeout ≥ 6× function timeout |
| Reject malformed prompts before invoking the model | **REST API request validator** + JSON Schema model |
| Per-customer request quotas and throttling on a public GenAI API | **Usage plans + API keys** (+ real auth) |
| Chatbot answers feel slow even though total time is fine | **Stream** (ConverseStream) to cut TTFT |
| Stream tokens through API Gateway with existing REST auth/WAF | REST API **`responseTransferMode: STREAM`** + Lambda proxy (SSE) |
| Stream needs > 29 s, response > 10 MB, no quota request | **REST response streaming** (up to 15 min) |
| Need response caching or VTL transformation *and* streaming | Not possible in STREAM mode; redesign |
| Bi-directional chat, user can interrupt, server pushes async results | **API Gateway WebSocket API** + `PostToConnection` |
| WebSocket route times out on long generation | Hand off async (SQS / async Lambda), stream back with `@connections` |
| Simplest Lambda token streaming without API Gateway | **Lambda function URL, `RESPONSE_STREAM`** |
| Intermittent 429 ThrottlingException | **Exponential backoff + jitter** (SDK standard/adaptive), CRIS, quota increase, Provisioned Throughput |
| 503 ServiceUnavailable despite being under quota | Capacity issue → backoff, **Cross-Region Inference**, other Region |
| One busy client keeps getting throttled; want SDK to self-pace | botocore **`adaptive`** retry mode |
| Long streams on EKS hang/reset after idle via NAT/VPC endpoint | **TCP keep-alive** in SDK **and** OS < 350 s |
| Long completions fail with read timeout in boto3 | Increase `read_timeout` (default 60 s) or stream |
| ModelNotReadyException on imported model | Model scaling from zero; SDK retries; warm-up |
| Primary model down; must keep serving | Fallback chain: alt model → semantic cache → static response; circuit breaker |
| Find which hop (API GW, Lambda, Bedrock, DynamoDB) adds latency | **AWS X-Ray** active tracing + SDK instrumentation |
| Route simple vs complex prompts within a model family, least ops | **Bedrock Intelligent Prompt Routing** |
| Non-English prompts or cross-provider routing based on content | **Custom router**: classifier + **Step Functions Choice** |
| Route by task type, change without redeploy | **Static routing config in AppConfig** |
| Shift traffic away from a model when its latency spikes | **Metric-based routing**: CloudWatch alarm → Lambda → AppConfig weights |
| Premium API-key tier gets a better model, no Lambda | API GW **mapping template** / stage variable injects `modelId` |

## Flashcards

<details>
<summary><b>Q:</b> What is the main advantage of Converse over InvokeModel?</summary>

Converse uses one request and response format for every model, including messages, system prompt, tool use, documents and guardrails. You can swap models by changing only the `modelId`. InvokeModel needs a body specific to each provider.

*D2 · Task 2.4 · Converse vs InvokeModel*
</details>

<details>
<summary><b>Q:</b> How do you pass a model-specific inference parameter (e.g. top_k) through Converse?</summary>

Put it in `additionalModelRequestFields`. Common parameters go in `inferenceConfig`.

*D2 · Task 2.4 · Converse params*
</details>

<details>
<summary><b>Q:</b> When should you use StartAsyncInvoke rather than an SQS + Lambda worker?</summary>

When the model itself has to run asynchronously and writes its output to S3, such as Nova Reel video generation. SQS + Lambda wraps async processing around any synchronous model call.

*D2 · Task 2.4 · Async invoke*
</details>

<details>
<summary><b>Q:</b> In an SQS → Lambda → Bedrock worker, how do you stop Lambda from exceeding the Bedrock quotas?</summary>

Cap concurrency with the event source mapping `ScalingConfig.MaximumConcurrency` (or reserved concurrency). Add SDK backoff, and a DLQ for messages that keep failing.

*D2 · Task 2.4 · SQS worker*
</details>

<details>
<summary><b>Q:</b> What's the recommended SQS visibility timeout for a Lambda event source mapping?</summary>

At least 6× the function timeout. Otherwise messages can become visible again while they're still being processed.

*D2 · Task 2.4 · SQS visibility*
</details>

<details>
<summary><b>Q:</b> Which API Gateway type supports request validation and usage plans?</summary>

REST APIs. HTTP APIs support neither.

*D2 · Task 2.4 · API Gateway types*
</details>

<details>
<summary><b>Q:</b> Do API keys authenticate callers?</summary>

No. API keys identify clients for usage plans (throttle and quota). Use IAM, Cognito or a Lambda authorizer for authentication.

*D2 · Task 2.4 · Usage plans*
</details>

<details>
<summary><b>Q:</b> How do you turn on response streaming for an API Gateway REST API, and which integrations does it work with?</summary>

Set the integration's `responseTransferMode` to `STREAM` (the default is `BUFFERED`). It only works with `HTTP_PROXY` and `AWS_PROXY` integrations, such as Lambda proxy.

*D2 · Task 2.4 · REST streaming*
</details>

<details>
<summary><b>Q:</b> Which API Gateway features stop working in STREAM mode?</summary>

Endpoint caching, content encoding (compression) and VTL response transformation. Request streaming isn't supported either.

*D2 · Task 2.4 · Streaming limits*
</details>

<details>
<summary><b>Q:</b> How long can an API Gateway REST response stream last, and what are the idle timeouts?</summary>

Up to 15 minutes. The idle timeout is 5 minutes for Regional and private endpoints and 30 seconds for edge-optimised ones.

*D2 · Task 2.4 · Streaming limits*
</details>

<details>
<summary><b>Q:</b> A WebSocket route Lambda times out while a long answer is generated. What's the fix?</summary>

Return from the route quickly and hand the work off (SQS or an async Lambda). The worker streams tokens back to the client with the `@connections` `PostToConnection` API. The WebSocket integration timeout is 29 s.

*D2 · Task 2.4 · WebSocket*
</details>

<details>
<summary><b>Q:</b> Which ConverseStream event tells you the token usage?</summary>

The `metadata` event. It contains `usage` (input and output tokens) and `metrics.latencyMs`.

*D2 · Task 2.4 · ConverseStream events*
</details>

<details>
<summary><b>Q:</b> What's the difference between guardrail streamProcessingMode sync and async?</summary>

Sync buffers and checks chunks before sending them, so it's safer but adds some latency. Async sends chunks immediately and checks them in the background, so it's faster but unsafe text may reach the user first.

*D2 · Task 2.4 · Guardrails streaming*
</details>

<details>
<summary><b>Q:</b> What's the difference between ThrottlingException (429) and ServiceUnavailable (503)?</summary>

429 means you exceeded your account quota. 503 means the service lacks capacity and isn't related to your quota. Retry both with backoff and jitter. For 429, also consider a quota increase or Provisioned Throughput. For 503, consider CRIS or another Region.

*D2 · Task 2.4 · Error handling*
</details>

<details>
<summary><b>Q:</b> Which Bedrock errors should you not retry?</summary>

ValidationException, AccessDeniedException, ResourceNotFoundException and ServiceQuotaExceededException. These are client or configuration errors, so fix the request or fall back to another model.

*D2 · Task 2.4 · Error handling*
</details>

<details>
<summary><b>Q:</b> How does botocore adaptive retry mode differ from standard?</summary>

Adaptive does everything standard does (exponential backoff with jitter, retry quota) and adds client-side rate limiting, which slows the send rate when the client is being throttled.

*D2 · Task 2.4 · SDK retries*
</details>

<details>
<summary><b>Q:</b> Bedrock calls from EKS through a NAT gateway hang after idle periods. What's the cause and the fix?</summary>

NAT gateways, VPC endpoints and NLBs silently drop TCP connections that are idle for more than 350 s. Turn on `tcp_keepalive=True` in the SDK and set the OS `tcp_keepalive_time` below 350 s. You need both.

*D2 · Task 2.4 · Networking*
</details>

<details>
<summary><b>Q:</b> Why order timeouts so that client > API Gateway > Lambda > SDK?</summary>

So each inner layer gives up before the layer outside it. Otherwise retries keep running (and spending tokens) after the caller has already received a timeout.

*D2 · Task 2.4 · Timeout budget*
</details>

<details>
<summary><b>Q:</b> What does X-Ray add over Bedrock model invocation logging?</summary>

X-Ray traces timing and errors across service boundaries (API GW → Lambda → Bedrock / SQS / DynamoDB). Invocation logging records the content of prompts and responses. They complement each other.

*D2 · Task 2.4 · Observability*
</details>

<details>
<summary><b>Q:</b> What are the main limitations of Bedrock Intelligent Prompt Routing?</summary>

It only routes between two models in the same family, it's optimised for English prompts only, and it can't learn from your application's own performance data.

*D2 · Task 2.4 · Prompt routing*
</details>

<details>
<summary><b>Q:</b> What does responseQualityDifference control in a configured prompt router?</summary>

How much better the other model's predicted quality must be than the fallback model's before the router switches to it. Raising the value keeps more traffic on the fallback (usually cheaper) model.

*D2 · Task 2.4 · Prompt routing*
</details>

<details>
<summary><b>Q:</b> How do you call an intelligent prompt router?</summary>

Pass the router ARN as the `modelId` in Converse or InvokeModel. The response shows which model handled the request.

*D2 · Task 2.4 · Prompt routing*
</details>

<details>
<summary><b>Q:</b> How do you build content-based routing to specialised FMs with Step Functions?</summary>

Classify the request with a cheap model or Comprehend, then use a Choice state to branch to model-specific tasks (or RAG or agent branches). Each branch has Retry and Catch that fall back to a general model. Use Express workflows for high-volume synchronous routing.

*D2 · Task 2.4 · Step Functions routing*
</details>

<details>
<summary><b>Q:</b> How can API Gateway pick a model per customer tier without a Lambda?</summary>

On a REST non-proxy integration, use a VTL mapping template (or a stage variable) to inject the `modelId` based on the API key or usage plan, and call Bedrock through a direct AWS service integration.

*D2 · Task 2.4 · Edge routing*
</details>

---
[← Back to Domain 2](README.md)
