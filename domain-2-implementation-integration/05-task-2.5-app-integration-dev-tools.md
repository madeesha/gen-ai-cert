[← Back to Domain 2](README.md)

# 🧰 Task 2.5 · Implement application integration patterns and development tools

> **Domain 2 · Implementation and Integration (26%)**
> - Skill 2.5.1: GenAI-specific FM API interfaces (streaming, token limits, timeout retries)
> - Skill 2.5.2: Accessible AI interfaces (Amplify, OpenAPI API-first, Prompt Flows no-code)
> - Skill 2.5.3: Business system enhancements (CRM via Lambda, document processing via Step Functions, Bedrock Data Automation)
> - Skill 2.5.4: Developer productivity (Amazon Q Developer)
> - Skill 2.5.5: Advanced GenAI applications (Strands, Agent Squad, Step Functions agent patterns, prompt chaining)
> - Skill 2.5.6: Troubleshooting FM applications (CloudWatch Logs Insights, X-Ray, Q Developer)

## Key points
- **GenAI APIs are not ordinary APIs:** slow, long, priced per token and non-deterministic. Choose the front door by how long the answer takes: sync under the **29 s** REST default, **streaming** for live UX, **async job** (202 + job ID) for minutes → Skill 2.5.1.
- **Token budgets are part of the interface:** count before the call, cap `maxTokens`, check `stopReason`, trim history. Context overflow is a `ValidationException`, so fix the input rather than retry → 2.5.1.
- **Ready-made UI:** **Amplify AI Kit**: conversation route (multi-turn, history in DynamoDB, streamed) vs generation route (one synchronous call, structured output) → 2.5.2.
- **Contract first:** one **OpenAPI** spec feeds API Gateway, SDKs, mocks, tests and **Bedrock Agents action groups**; agents choose operations from the **descriptions** → 2.5.2.
- **No-code chains:** **Bedrock Flows** for business users and prompt engineers → 2.5.2 (full detail in Task 1.6).
- **Business systems:** CRM events → **Lambda** → Bedrock → write back; documents → **Step Functions** (Textract → Comprehend → Bedrock → human review); multimodal extraction with the least orchestration → **Bedrock Data Automation** (standard output vs custom **blueprints**) → 2.5.3.
- **Developer productivity:** **Q Developer** `/test`, `/review`, `/doc`, `/transform`, customizations. ⚠️ The IDE plugins reach **end of support on 30 April 2027** (successor **Kiro**), but the exam still names Q Developer → 2.5.4.
- **Orchestration choice:** deterministic, auditable steps → **Step Functions** (agent design patterns as states); visual GenAI chains → **Flows**; open-ended autonomy → **Strands / Bedrock Agents / AgentCore** → 2.5.5.
- **Troubleshooting stack:** **invocation logs** for content (+ Logs Insights), **X-Ray/ADOT** for timing per hop, **CloudTrail** for who called and the error code, `AWS/Bedrock` **metrics** for symptoms → 2.5.6.

## Notes

### Skill 2.5.1 — FM API interfaces for GenAI workloads

> *Create FM API interfaces to address the specific requirements of GenAI workloads (for example, by using API Gateway to handle streaming responses, token limit management, retry strategies to handle model timeouts).*

This skill is about **designing the API your clients call** to reach an FM. The low-level SDK streaming and backoff mechanics are in [Task 2.4](04-task-2.4-fm-api-integrations.md) ([streaming: Skill 2.4.2](04-task-2.4-fm-api-integrations.md#skill-242--real-time-ai-interaction-systems--pq24), [retries: Skill 2.4.3](04-task-2.4-fm-api-integrations.md#skill-243--resilient-fm-systems)). Here the focus is on interface design.

#### What makes GenAI APIs different

| Property | Traditional REST API | GenAI API | Design consequence |
| --- | --- | --- | --- |
| Latency | ms | **seconds → minutes** (long outputs, reasoning models) | Streaming, async jobs, long client timeouts |
| Response size | bounded | **variable, up to max output tokens** | Token caps, pagination/continuation |
| Cost | per request | **per input + output token** | Budgets, per-tenant quotas, caching |
| Determinism | deterministic | **stochastic** | Idempotency keys, response caching only where safe |
| Failure modes | 4xx/5xx | + **throttling**, **model timeout**, **context overflow**, **truncation**, **guardrail intervention** | Typed error mapping, partial-response handling |

#### Choosing the API front door

| Requirement | Pattern | Notes |
| --- | --- | --- |
| Short answers (< ~25 s), simple clients | **API Gateway REST/HTTP → Lambda → `Converse`** | REST integration timeout **29 s default**. A higher limit can be requested for Regional/private REST APIs, at the cost of throttle quota (**verify in docs**). HTTP API max **30 s**. |
| Token-by-token UX in a browser | **Lambda response streaming** (function URL, `InvokeWithResponseStream`) + `ConverseStream` → SSE/chunked to the client | Lambda streaming responses can be far larger than buffered ones (soft limit **20 MB**, **verify**). Avoids the API GW 29 s cap. |
| Bidirectional, chat, server push | **API Gateway WebSocket API** → Lambda → `ConverseStream`, push chunks with `@connections` `PostToConnection` | Idle timeout **10 min**, max connection **2 h**. Store `connectionId` in DynamoDB. |
| Streaming through REST API Gateway | **REST API response streaming** (newer feature; **verify in docs**) | Lets you keep API GW features (auth, WAF, usage plans) while streaming |
| Very long / batch jobs | **Async**: API → SQS → worker → result in S3/DynamoDB → poll `GET /jobs/{id}` or callback/webhook/EventBridge | Return **202 Accepted** + job ID. Use Bedrock **batch inference** for bulk offline jobs. |
| GraphQL front end | **AppSync** → Lambda/Bedrock resolver. **AppSync subscriptions** stream chunks. | This is what Amplify AI Kit uses |

```python
# Lambda (Python) behind a WebSocket API: stream Converse chunks to the client
import boto3, json
brt = boto3.client("bedrock-runtime")
def handler(event, _):
    cid = event["requestContext"]["connectionId"]
    apigw = boto3.client("apigatewaymanagementapi",
        endpoint_url=f"https://{event['requestContext']['domainName']}/{event['requestContext']['stage']}")
    prompt = json.loads(event["body"])["prompt"]
    resp = brt.converse_stream(
        modelId="us.anthropic.claude-sonnet-4-20250514-v1:0",
        messages=[{"role": "user", "content": [{"text": prompt}]}],
        inferenceConfig={"maxTokens": 1024, "temperature": 0.3})
    for ev in resp["stream"]:
        if "contentBlockDelta" in ev:
            apigw.post_to_connection(ConnectionId=cid,
                Data=json.dumps({"delta": ev["contentBlockDelta"]["delta"].get("text", "")}))
        elif "messageStop" in ev:
            apigw.post_to_connection(ConnectionId=cid,
                Data=json.dumps({"done": True, "stopReason": ev["messageStop"]["stopReason"]}))
    return {"statusCode": 200}
```

#### Token limit management

- **Know the three budgets**: model **context window** (input + output), the model's **max output tokens**, and your **cost/latency budget**.
- **Before the call**:
  - Estimate tokens (≈ 4 chars/token for English is a rough heuristic) or call the Bedrock **`CountTokens`** API where the model supports it (**verify model support**).
  - Truncate or **summarise conversation history**: keep the system prompt + last *N* turns + a running summary of older turns (store the summary in DynamoDB).
  - For RAG, limit **numberOfResults** and **rerank** so fewer, better chunks are sent.
  - Reject oversized uploads at the edge: **API Gateway request validation** (JSON schema, `maxLength`), plus Lambda checks.
- **During the call**: set `inferenceConfig.maxTokens` explicitly, never rely on defaults. Use `stopSequences` to end early.
- **After the call**: inspect **`stopReason`**:

| `stopReason` | Meaning | Action |
| --- | --- | --- |
| `end_turn` | Natural finish | Return |
| `max_tokens` | **Truncated** | Ask the model to *continue*, raise `maxTokens`, or request a shorter format |
| `stop_sequence` | Hit a stop sequence | Expected |
| `tool_use` | Model wants a tool | Run the tool, send `toolResult` |
| `guardrail_intervened` | Guardrail blocked/masked | Return the guardrail message; log it |
| `content_filtered` | Filtered by the model provider | Handle as a safe refusal |

- Usage data in the `Converse` response: `usage.inputTokens`, `usage.outputTokens`, `usage.totalTokens`. Return these to the caller or meter them per tenant (API Gateway **usage plans + API keys** for request quotas; token quotas need custom logic, e.g. DynamoDB counters).
- **Context overflow** → `ValidationException` ("input is too long"). **Not retryable**. Shrink the input or use a larger-context model.

#### Retry strategies for model timeouts

Retry mechanics (which errors to retry, SDK `standard`/`adaptive` modes and `read_timeout`, backoff with jitter, the timeout chain, fallbacks) → Full detail: [Task 2.4 · Skill 2.4.3](04-task-2.4-fm-api-integrations.md#skill-243--resilient-fm-systems). What the interface itself must add:
- **Don't retry guardrail interventions**: they are blocked by design.
- **One retry layer, one time budget:** don't let a client retry and a Lambda retry *both* fire. Accept an **idempotency key** header (→ DynamoDB / Powertools idempotency) so a retried `POST /generate` doesn't double-charge or double-write.
- **Streaming + retries:** if a stream drops mid-way you can't simply resume. Either restart (idempotent) or send the partial text back with "continue from here".

---

### Skill 2.5.2 — Accessible AI interfaces

> *Develop accessible AI interfaces to accelerate adoption and integration of FMs (for example, by using AWS Amplify to develop declarative UI components, OpenAPI specifications for API-first development approaches, Amazon Bedrock Prompt Flows for no-code workflow builders).*

"Accessible" here means **easy for developers and business users to adopt**: ready-made UI components, contract-first APIs and no-code builders.

#### AWS Amplify Gen 2 — AI Kit

- Amplify Gen 2 = **TypeScript code-first** backend definitions (`amplify/data/resource.ts`), deployed with CDK under the hood. The **AI Kit** adds GenAI routes to the Amplify **Data** (AppSync) schema.
- **Architecture**: **AppSync** (API, auth, routing) + **Lambda** (conversation handler) + **DynamoDB** (conversation history) + **Bedrock** (Converse API).

| | **Conversation route** (`a.conversation`) | **Generation route** (`a.generation`) |
| --- | --- | --- |
| Style | **Async, multi-turn** chat | **Single synchronous** request–response |
| History | Stored automatically in **DynamoDB**, per user | None |
| Transport | Mutation to send + **AppSync subscription** streams the response | AppSync **query** |
| Output | Free text (+ tool use) | **Structured, typed** data (`.returns(...)`) |
| Typical use | Chatbots, assistants | Summaries, tag generation, form fill, recipe JSON |
| UI | `<AIConversation>` + `useAIConversation` | `useAIGeneration` |

```ts
// amplify/data/resource.ts (Amplify Gen 2 AI Kit — shape; verify exact model names in docs)
const schema = a.schema({
  chat: a.conversation({
    aiModel: a.ai.model("Claude 3.5 Haiku"),
    systemPrompt: "You are a helpful support assistant for Acme.",
    tools: [a.ai.dataTool({ name: "listOrders", description: "List the user's orders",
                            model: a.ref("Order"), modelOperation: "list" })],
  }).authorization((allow) => allow.owner()),

  summarise: a.generation({
    aiModel: a.ai.model("Claude 3.5 Haiku"),
    systemPrompt: "Summarise the text in 3 bullet points.",
  })
  .arguments({ text: a.string() })
  .returns(a.customType({ bullets: a.string().array() }))
  .authorization((allow) => allow.authenticated()),
});
```

```tsx
// React front end — declarative UI component
import { AIConversation, createAIHooks } from "@aws-amplify/ui-react-ai";
const { useAIConversation, useAIGeneration } = createAIHooks(client);
const [{ data: { messages }, isLoading }, sendMessage] = useAIConversation("chat");
<AIConversation messages={messages} isLoading={isLoading} handleSendMessage={sendMessage} />
```

- **Why it's on the exam**: the fastest way to put an **authenticated (Cognito), streaming, history-persisting chat UI** on Bedrock with very little code. **Tools** can be Amplify data queries or custom Lambda functions. Auth: `allow.owner()` keeps conversations private per user.
- Hosting: **Amplify Hosting** (Git-based CI/CD for the front end).

#### OpenAPI specifications — API-first development

- **API-first** = write the **OpenAPI 3.x** contract *before* the code. One spec gives you:
  - **API Gateway** import (`ImportRestApi`), plus **request validation** (JSON schema models) from the spec.
  - **Client SDK generation** (API Gateway SDK export, OpenAPI Generator), **docs** (Swagger UI), **mock integrations** for front-end teams, and **contract tests** in CI.
- **Bedrock Agents action groups** take an **OpenAPI schema** (JSON/YAML, inline or in S3) *or* **function details** (simpler: name, description, parameters). The agent uses the spec's **descriptions** to decide which operation to call and how to fill parameters.
- Executor = **Lambda** (the agent invokes it with `apiPath`, `httpMethod`, `parameters`, `requestBody`) or **Return of control** (the parameters go back to your app, which runs the call itself).

```yaml
openapi: 3.0.0
info: { title: Order API, version: 1.0.0 }
paths:
  /orders/{orderId}:
    get:
      operationId: getOrderStatus
      description: Get the current shipping status of a customer's order. Use when the user asks where their order is.
      parameters:
        - name: orderId
          in: path
          required: true
          description: The order number, e.g. ORD-12345
          schema: { type: string }
      responses:
        "200":
          description: Order status
          content:
            application/json:
              schema:
                type: object
                properties:
                  status: { type: string }
                  eta: { type: string }
```

- ⚠️ Gotchas: vague `description` → the agent picks the wrong API. Keep operations few and focused. The schema must be **valid OpenAPI** or agent creation/preparation fails. The Lambda response must match the expected agent response format (`messageVersion`, `response.actionGroup`, `apiPath`, `httpStatusCode`, `responseBody`).
- Tool-use JSON schemas in `Converse` (`toolConfig.tools[].toolSpec.inputSchema.json`) follow the same idea. Strands/MCP tool definitions are in [Task 2.1](01-task-2.1-agentic-ai-tool-integrations.md).

#### Bedrock Flows (Prompt Flows) — no-code workflow builder

- **The accessibility angle:** a visual canvas in the Bedrock console where **business analysts and prompt engineers** chain prompt, KB, agent, Lambda, condition and iterator nodes without code, and developers get a managed chain without Step Functions plumbing.
- Apps consume a flow like an API: call **`InvokeFlow`** on a versioned **alias** (`flowIdentifier` + `flowAliasIdentifier`) and promote by moving the alias.
- Flows vs Step Functions → [Skill 2.5.5](#skill-255--advanced-genai-applications).
- → Full detail (node types, versions/aliases, branching, reusable components): [Task 1.6 · Skill 1.6.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md#skill-166--complex-prompt-systems-with-bedrock-prompt-flows).

#### Other "accessible" options worth knowing

| Option | Audience | Use |
| --- | --- | --- |
| **Bedrock playgrounds** (chat/text/image) | Anyone | Try models and prompts, compare models side by side |
| **Amazon Bedrock IDE / SageMaker Unified Studio** | Builders across teams | Build chat agents/flows collaboratively (**verify current naming**) |
| **Amazon Q Business** | Business users | Managed enterprise assistant over company data (connectors), no build |
| **Amazon Q Apps** | Business users | No-code apps generated from natural language inside Q Business |
| **PartyRock** | Learners | Playground for building simple GenAI apps (educational) |
| **SageMaker Canvas** | Analysts | No-code ML + FM access |

---

### Skill 2.5.3 — Business system enhancements

> *Create business system enhancements (for example, by using Lambda functions to implement customer relationship management [CRM] enhancements, Step Functions to orchestrate document processing systems, Amazon Bedrock Data Automation to manage automated data processing workflows).*

#### CRM enhancements with Lambda

Typical GenAI add-ons for a CRM (Salesforce, Dynamics, HubSpot, an in-house system):

| Enhancement | Trigger | Flow |
| --- | --- | --- |
| **Case/ticket summarisation** | Case created/updated | CRM event → EventBridge → Lambda → Bedrock (summarise thread) → write summary field back |
| **Draft email/reply** | Agent clicks "Draft" | CRM UI → API GW → Lambda → Bedrock (+ KB for policies) → draft returned (human sends) |
| **Lead scoring / classification** | New lead | Lambda → Bedrock (classify intent/priority, JSON output) → CRM update |
| **Sentiment & next best action** | Call/chat ends | Amazon Connect / Contact Lens transcript → Lambda → Bedrock → CRM task |
| **Data enrichment** | Nightly | EventBridge Scheduler → Lambda/Step Functions → Bedrock batch → CRM bulk API |
| **Natural-language CRM search** | User query | Agent with action group calling CRM APIs (OpenAPI) |

- **Integration plumbing**:
  - **Event sources**: **EventBridge partner event sources** (e.g. Salesforce, Zendesk) or the CRM's webhooks → API GW → Lambda.
  - **Bulk data**: **Amazon AppFlow** (SaaS ↔ S3/Redshift) for KB ingestion.
  - **Credentials**: CRM OAuth tokens in **Secrets Manager**.
  - **Retries**: **SQS** buffer between the event and the Lambda to absorb CRM API rate limits; **DLQ** for failures.
- **Patterns**:
  - Keep the Lambda **stateless and idempotent** (CRM webhooks can arrive more than once).
  - Ask for **structured JSON output** (tool use / JSON schema) so the CRM update is deterministic.
  - Put a **human in the loop** before anything customer-facing is sent.
  - Apply **Guardrails** (PII masking) before text is written to CRM fields.
- ⚠️ Lambda max timeout is **15 min**, so long CRM batch jobs → Step Functions + Map / Bedrock batch inference.

#### Document processing with Step Functions

```
S3 PutObject → EventBridge → Step Functions (Standard)
  ├─ Classify document (Bedrock / Comprehend custom classifier)
  ├─ Choice: invoice | contract | ID | other
  ├─ Textract (AnalyzeDocument FORMS/TABLES/QUERIES, AnalyzeExpense, AnalyzeID)
  │     async: StartDocumentAnalysis → SNS/polling or .waitForTaskToken
  ├─ Comprehend: DetectPiiEntities / DetectEntities (redact, enrich)
  ├─ Bedrock (Converse): normalise to JSON schema, summarise, validate business rules
  ├─ Choice: confidence < threshold?  → Amazon A2I / human review task (waitForTaskToken)
  ├─ Write to DynamoDB / RDS / ERP API
  └─ Catch → SNS alert + DLQ, Retry with backoff on throttling
```

- **Step Functions integrations for Bedrock**:
  - **Optimised integration** `arn:aws:states:::bedrock:invokeModel`. It can read the request body from and write output to **S3**, which avoids the **256 KB** state payload limit. Also `createModelCustomizationJob` (with `.sync`).
  - For **Converse**, use the **AWS SDK integration** `arn:aws:states:::aws-sdk:bedrockruntime:converse` (**verify**).
- **Large volumes**: **Distributed Map** (up to 10,000 parallel child executions) iterating over S3 objects/CSV/manifest.
- **Standard vs Express**: Standard (≤ **1 year**, exactly-once, full history, supports human waits) for document workflows. Express (≤ **5 min**, at-least-once, high volume, cheap) for short synchronous enrichment.
- **Textract features**:
  - `FORMS` (key-value), `TABLES`, `QUERIES` (natural-language questions such as "What is the invoice total?"), `SIGNATURES`, `LAYOUT`.
  - Specialised APIs: `AnalyzeExpense` (invoices/receipts), `AnalyzeID` (identity docs), `AnalyzeLending` (mortgage packages).
  - Synchronous APIs = single-page/image. **Async** (`Start*`) for multi-page PDFs.
- **Comprehend**: entities, key phrases, sentiment, language, **PII detection/redaction**, **custom classification/entity recognition**. **Comprehend Medical** for clinical text.
- **When to add an FM**: Textract gives you raw text/fields; Bedrock **reasons** over them (normalise vendor names, map to your schema, check totals, summarise clauses).

#### Amazon Bedrock Data Automation (BDA)

- **What**: a managed GenAI service that turns **unstructured multimodal content** (documents, images, video, audio) into **structured output** through **one API**. You don't orchestrate multiple models/services yourself. Built-in **confidence scores** and **visual grounding** (bounding boxes).
- **Key concepts**:

| Concept | Meaning |
| --- | --- |
| **Standard output** | Default extraction per modality, no config needed: document summaries/text/tables, image summaries/text, **video scene summaries**, **audio transcripts/summaries**. Tunable via projects. |
| **Custom output** | **Documents, images and audio only**. Defined by **blueprints**. |
| **Blueprint** | Schema of **fields to extract** (name, type, natural-language instruction, e.g. "invoice_total: the total due including tax"). Pick from the **blueprint catalog** (W-2, pay stub, invoice, driver's licence, …) or **create custom** (you can also generate one from a sample document + prompt). |
| 🎯 **PQ29** **Project** | Container for standard-output settings for each modality + custom blueprints. Referenced in **`InvokeDataAutomationAsync`**. |
| **Output** | JSON written to your **S3** output location. Async, so check status or react to EventBridge events (**verify**). |

- **BDA vs DIY (Textract + Comprehend + Bedrock in Step Functions)**:

| Choose **BDA** when | Choose **DIY Step Functions** when |
| --- | --- |
| You want the least orchestration and code | You need fine-grained control over each step |
| Multimodal inputs (incl. **video/audio**) | You have custom business rules, branching, approvals |
| Standard document types, catalog blueprints | You need specific Textract features (e.g. AnalyzeLending) or custom Comprehend models |
| You want confidence scores + grounding built in | You must integrate many downstream systems with complex error handling |

  Combine them: Step Functions calls BDA as one step, then validates/routes and does human review.

- **BDA in RAG**: Bedrock **Knowledge Bases** can use **BDA as the parser** (alongside the default parser and **FM parser**). Use it for multimodal documents with charts, tables and images, and for audio/video content in a KB. See [Task 1.5](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md).

---

### Skill 2.5.4 — Developer productivity with Amazon Q Developer

> *Enhance developer productivity to accelerate development workflows for GenAI applications (for example, by using Amazon Q Developer to generate and refactor code, code suggestions for API assistance, AI component testing, performance optimization).*

- **Amazon Q Developer** = the generative AI coding assistant from AWS. It runs in IDEs (**VS Code, JetBrains, Eclipse, Visual Studio**), the **CLI**, the **AWS Console**, and chat tools (Slack/Teams via chat integrations).
- ⚠️ **Lifecycle note**: AWS has announced **end of support for the Q Developer IDE plugins on 30 April 2027**, pointing users to **Kiro** (agentic IDE with spec-driven development, MCP). The exam guide still lists Q Developer, so learn its capabilities.

| Capability | What it does | Exam cue |
| --- | --- | --- |
| **Inline suggestions** | Real-time code completion, including **AWS SDK/API calls** (e.g. writes the `bedrock-runtime` `converse` call with the correct shape) | "Speed up writing Bedrock/boto3 code" |
| **Chat / inline chat** | Explain code, answer AWS questions, generate snippets, using **workspace context** (`@workspace`, files) | "Ask about the codebase" |
| **Agentic coding** | Multi-file changes, runs shell commands, iterates | "Implement a feature end to end" |
| **`/dev`** (feature dev, now largely folded into agentic chat, **verify**) | Plans and implements a feature across files | |
| **`/test`** | **Generates unit tests** (Java/Python), finds edge cases | "AI component testing", raise coverage of the prompt-handling code |
| **`/review`** | **Code review + security scanning** (SAST, secrets, IaC misconfig, code quality) with fixes | "Find vulnerabilities before merge" |
| **`/doc`** | Generates README/docs and data-flow diagrams for a codebase | "Document a legacy service" |
| **`/transform`** | **Code transformation**: Java 8/11 → **17/21**, .NET Framework → **cross-platform .NET**, SQL conversion (**verify scope**). Large-scale mainframe/VMware/.NET migration → **AWS Transform** | "Upgrade the Java version" |
| **Refactoring & optimisation** | Suggests faster or cleaner code (e.g. reuse the boto3 client, batch embeddings, async calls) | "Performance optimization" |
| **Customizations** (Pro tier) | Suggestions informed by your **private repos/internal libraries** | "Suggestions should use our internal SDK" |
| **MCP servers** | Connect Q to external tools/data via MCP | |
| **Console: "Diagnose with Amazon Q"** | Explains AWS **console errors** (e.g. AccessDenied) and suggests fixes | Troubleshooting (2.5.6) |
| **CloudWatch investigations** | AI-assisted operational investigation of alarms (formerly a Q Developer capability, **verify naming**) | Root cause analysis |
| **Q Developer CLI** | Agentic chat in the terminal, command completions, natural language → shell (**verify current naming; transitioning to Kiro CLI**) | |

- **Tiers**: **Free** (monthly limits) vs **Pro** (per-user subscription: higher limits, customizations, admin/IAM Identity Center management, **IP indemnity**). Pro does not use your content for service improvement (**verify**).
- **GenAI-specific use**:
  - Generate Bedrock `Converse`/tool-use boilerplate and Strands agent skeletons.
  - Generate **test harnesses** for prompts (golden datasets, assertions on JSON output).
  - Explain `ValidationException` payload errors.
  - Refactor synchronous code to streaming.
  - Generate CDK for KB/OpenSearch stacks.
- **Security/governance**: use **Pro** with IAM Identity Center for enterprises. Q's generated code still needs **review** (it's a suggestion, not authority). Reference tracking flags suggestions that resemble open-source code, along with the licence.

---

### Skill 2.5.5 — Advanced GenAI applications

> *Develop advanced GenAI applications to implement sophisticated AI capabilities (for example, by using Strands Agents and AWS Agent Squad for AWS native orchestration, Step Functions to orchestrate agent design patterns, Amazon Bedrock to manage prompt chaining patterns).*

Application-level view only. Strands, Agent Squad and MCP internals are in [Task 2.1](01-task-2.1-agentic-ai-tool-integrations.md).

#### Orchestration options at a glance

| Option | Control style | Best for |
| --- | --- | --- |
| **Strands Agents** (open-source SDK, model-driven agent loop) | The **model decides** steps and tools | Autonomous agents, tool use, multi-agent (agents-as-tools, swarm, graph). Deploy on **AgentCore Runtime**/Lambda/ECS. |
| **AWS Agent Squad** (open-source, formerly Multi-Agent Orchestrator) | A **classifier routes** each request to the best specialist agent | Many specialist agents behind one entry point, with conversation context kept per agent |
| **Bedrock Agents** (managed) | Managed ReAct loop + action groups + KBs; **multi-agent collaboration** (supervisor) | Managed, low-code agents |
| **Bedrock Flows** | **Visual DAG** of prompts/KB/Lambda | Deterministic chains, no code |
| **Step Functions** | **Explicit state machine** | Deterministic, auditable, long-running, human approval, many AWS integrations |

- **Rule of thumb:** the more the path must be predictable and auditable, the further down this table you go; the more the model must decide the next step, the further up.
- **Flows vs Step Functions**: Flows → **GenAI-native nodes**, fast to build, simpler. **Step Functions** → 220+ service integrations, long-running workflows, human-approval waits (`.waitForTaskToken`), rich error handling/retries, Standard-workflow audit history.

#### Agent design patterns implemented with Step Functions

| Pattern | Step Functions construct | Example |
| --- | --- | --- |
| **Prompt chaining** | Sequential `Task` states, each output → next input. Add a **gate** (Choice) between steps. | Outline → draft → edit → translate |
| **Routing** | Classifier Task → **Choice** state → specialised branch/model | Billing vs tech support. Simple → small model, complex → big model |
| **Parallelisation (sectioning / voting)** | **Parallel** state or **Map** → aggregate step | Check a doc for 5 policy types at once. Run 3 models and take a majority vote. |
| **Orchestrator–workers** | Orchestrator LLM emits a sub-task list → **Map** over sub-tasks → synthesiser | Research report across multiple sources |
| **Evaluator–optimizer (reflection)** | Generate → **judge** Task (LLM-as-judge) → Choice (score ≥ threshold?) → loop back with feedback, **max-iteration counter** | Code/SQL generation with self-critique |
| **ReAct / tool loop** | Loop: Converse → Choice on `stopReason == tool_use` → run tool Task → back | Explicit, auditable agent loop (see Task 2.1) |
| **Human-in-the-loop** | `.waitForTaskToken` → notify a reviewer → resume with `SendTaskSuccess` | Approve before sending to a customer |

```json
{
  "Comment": "Evaluator-optimizer loop (simplified)",
  "StartAt": "Generate",
  "States": {
    "Generate": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": { "ModelId": "<model-arn>", "Body": { "...": "prompt + $.feedback" } },
      "ResultPath": "$.draft", "Next": "Evaluate"
    },
    "Evaluate": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": { "ModelId": "<judge-model-arn>", "Body": { "...": "score $.draft 1-10 as JSON" } },
      "ResultPath": "$.score", "Next": "GoodEnough?"
    },
    "GoodEnough?": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.score.value", "NumericGreaterThanEquals": 8, "Next": "Done" },
        { "Variable": "$.iteration", "NumericGreaterThanEquals": 3, "Next": "Done" }
      ],
      "Default": "IncrementAndRetry"
    },
    "IncrementAndRetry": { "Type": "Pass", "Next": "Generate", "Comment": "increment $.iteration, set $.feedback" },
    "Done": { "Type": "Succeed" }
  }
}
```

- **Stopping conditions** (max iterations, timeouts, token budget) are mandatory for loops. See safeguarded workflows in [Task 2.1](01-task-2.1-agentic-ai-tool-integrations.md).
- Use `Retry` on `Bedrock.ThrottlingException` / `Bedrock.ModelTimeoutException` (error names for the optimised integration: **verify in docs**) with `BackoffRate` + `JitterStrategy: FULL` (retry design: [Skill 2.4.3](04-task-2.4-fm-api-integrations.md#skill-243--resilient-fm-systems)).

#### Prompt chaining with Amazon Bedrock

- **Why chain**: one giant prompt is fragile. Smaller prompts are **easier to test, version and debug**, and each step can use a **different model** (cheap model for extraction, strong model for reasoning).
- **Ways to chain**:
  1. **Application code**: sequential `Converse` calls, passing outputs (simplest, most flexible).
  2. **Bedrock Flows**: Prompt nodes wired together, plus Condition/Iterator nodes. No code; versioned via aliases.
  3. **Step Functions**: Task states calling Bedrock. Adds retries, branching, audit, human steps.
  4. **Bedrock Prompt Management**: store each step's prompt as a versioned template with `{{variables}}` and reference it from code/Flows.
- **Tips**:
  - Have each step return **structured JSON** (tool-use schema), so the next step parses it reliably.
  - Validate between steps (Lambda/JSON schema).
  - Cache stable prefixes (**prompt caching**).
  - Log each step's input/output for debugging (2.5.6).

---

### Skill 2.5.6 — Troubleshooting FM applications

> *Improve troubleshooting efficiency for FM applications (for example, by using CloudWatch Logs Insights to analyze prompts and responses, X-Ray to trace FM API calls, Amazon Q Developer to implement GenAI-specific error pattern recognition).*

#### Bedrock model invocation logging (verified) 🎯 PQ67

- **Disabled by default**. Configure per account per Region: console **Settings → Model invocation logging**, or `PutModelInvocationLoggingConfiguration`.
- Covers **`Converse`, `ConverseStream`, `InvokeModel`, `InvokeModelWithResponseStream`** on the `bedrock-runtime` endpoint.
- **Destinations**: **CloudWatch Logs**, **S3**, or **both**. They must be in the **same account and Region**.
- **Modalities to log**: Text, Image, Embedding, Video. The setting applies to *all* models that support the selected modality.
- **Size handling**: bodies **≤ 100 KB** are inline. **Binary data and bodies > 100 KB go to S3 only** (under a data prefix; CloudWatch needs an S3 location configured for large data). The S3 destination writes **gzipped JSON batches**, which you can query with **Athena**/Glue.
- **CloudWatch Logs setup**: an IAM role trusted by `bedrock.amazonaws.com` with `logs:CreateLogStream`/`PutLogEvents`. Log stream: `aws/bedrock/modelinvocations`. **S3 setup**: a bucket policy allowing `bedrock.amazonaws.com` `s3:PutObject`, with `aws:SourceAccount`/`aws:SourceArn` conditions (confused-deputy protection). **SSE-KMS** needs `kms:GenerateDataKey` for Bedrock. Bucket **ACLs must be disabled**.
- **Log record fields**:
  - `timestamp`, `accountId`, `region`, `requestId`, `operation`, `modelId` (model or inference profile)
  - `identity.arn` (caller principal)
  - `requestMetadata` (optional caller-supplied tags, e.g. team/env)
  - `input.inputBodyJson`, `input.inputTokenCount`
  - `output.outputBodyJson`, `output.outputTokenCount`
- ⚠️ The logs contain **full prompts and responses**, which may include PII. Restrict access to the log group/bucket, encrypt with KMS, and consider CloudWatch Logs **data protection policies** (masking).

> [!WARNING]
> **🎯 Practice question 67 — bank assistant: no PII to the model, no investment advice, audit logs that include images and documents, LEAST effort**
>
> - **Guardrail sensitive information policy** → detects and filters PII **in prompts (before the model) and in responses**. **Denied topic policy** → investment advice.
> - **Model invocation logging** (covers Converse and the other runtime APIs) **to S3** → full request and response content. **Turn on the data-delivery options** for each data type: **text, image and embedding** (plus video where supported), so **images and documents sent in conversations are logged too**. Large or binary payloads go to **S3**, not CloudWatch.
> - ❌ **Regex + prompt engineering:** misses unstructured PII such as names and addresses, and prompts don't reliably block topics.
> - ❌ **Macie + CloudTrail:** Macie scans S3 and doesn't filter inline, and CloudTrail has **no conversation content**.
> - ❌ **Lambda + Comprehend redaction + topic modelling + CloudWatch metrics:** custom code. **Topic modelling** groups documents offline and isn't a real-time block, and metrics aren't logs.

#### CloudWatch Logs Insights queries

```sql
-- 1. Token usage by caller (from AWS docs)
fields identity.arn as principal, input.inputTokenCount as inTokens, output.outputTokenCount as outTokens
| stats sum(inTokens) as totalInput, sum(outTokens) as totalOutput, count() as calls by principal
| sort totalInput desc
```

```sql
-- 2. Which models / operations are used most, and their average output size
fields modelId, operation, output.outputTokenCount
| stats count() as calls, avg(output.outputTokenCount) as avgOut by modelId, operation
| sort calls desc
```

```sql
-- 3. Find prompts mentioning a keyword (e.g. investigate a bad answer reported by a user)
fields @timestamp, requestId, identity.arn, modelId
| filter input.inputBodyJson.messages.0.content.0.text like /refund policy/
| sort @timestamp desc
| limit 20
```

```sql
-- 4. Very large prompts (context bloat / cost spikes)
fields @timestamp, modelId, input.inputTokenCount
| filter input.inputTokenCount > 50000
| sort input.inputTokenCount desc
```

```sql
-- 5. Cost by team using requestMetadata tags
fields requestMetadata.team as team, input.inputTokenCount + output.outputTokenCount as tokens
| stats sum(tokens) as totalTokens by team
```

- Also run Logs Insights over **your application logs**: the Lambda structured logs from Powertools (with prompt template version, `stopReason`, latency, guardrail action, retrieved chunk IDs), so you can correlate a bad answer to its **prompt version + retrieved context**.
- For KBs, enable **Knowledge Base logging** (ingestion job/document-level status) to troubleshoot documents that failed to ingest (**verify delivery options**).

Logs tell you *what* was sent and returned. The next two tools tell you *how the system behaved*: metrics for symptoms, traces for where the time went.

#### CloudWatch metrics & GenAI observability

- **Read the `AWS/Bedrock` metrics (by `ModelId`) as symptoms** (`Invocations`, `InvocationLatency`, `InvocationClientErrors`, `InvocationServerErrors`, `InvocationThrottles`, `InputTokenCount` / `OutputTokenCount`):
  - throttle spikes → request a quota increase / use cross-Region inference / Provisioned Throughput
  - rising latency → check output token growth
  - rising client errors → a recent deploy broke the payload
- **Guardrails metrics** (interventions per policy) help tell "model refused" apart from "guardrail blocked".
- **CloudWatch generative AI observability**: out-of-the-box dashboards for **model invocations** and **agent traces** (Bedrock AgentCore, Strands and other frameworks instrumented with **ADOT/OpenTelemetry**), showing latency, token usage, errors and spans per agent step (**verify current feature scope**).
- → Full metric list, dimensions, KPIs and anomaly alarms: [Task 4.3 · Skill 4.3.2](../domain-4-operational-efficiency-optimization/03-task-4.3-monitoring-observability.md#skill-432--comprehensive-genai-monitoring-and-kpis).

#### AWS X-Ray — tracing FM API calls

- **End-to-end trace**: API Gateway (stage setting: enable **active tracing**) → Lambda (`TracingConfig: Active`) → Bedrock / KB / OpenSearch / DynamoDB / SQS as **subsegments** → Step Functions (X-Ray tracing on the state machine). SQS → Lambda traces are linked.
- **Instrumentation**:
  - **AWS Distro for OpenTelemetry (ADOT)** is the recommended path; the X-Ray SDKs are moving to maintenance (**verify**).
  - X-Ray SDK patching, e.g. `aws_xray_sdk.core.patch_all()` / `patch(['boto3'])`, so each `bedrock-runtime` call appears as a subsegment with its latency and errors.
  - Powertools Tracer for Lambda.
  - Add **annotations** (indexed: `modelId`, `promptVersion`, `tenantId`, `route`) and **metadata** (non-indexed: token counts) to filter traces.
- **What it answers**: *where* the latency is (retrieval vs reranking vs generation vs tool call), which downstream call is throwing, and retries/throttles visible as faults. Use the **service map** (trace map) to spot the slow or failing node. **CloudWatch Transaction Search / Application Signals** gives span-level search and service-level SLOs (**verify**). For agents, **AgentCore Observability** adds agent-specific views.
- **X-Ray vs invocation logging**: X-Ray gives you **timing**; invocation logs give you **content**. You usually need both.
- Typical finding: "slow chatbot" = 80% of the time is in **generation**. Fix with streaming (perceived latency), a smaller model, a lower `maxTokens`, prompt caching, or **latency-optimised inference**.

#### CloudTrail

- Bedrock **control-plane** calls (create KB, guardrail, logging config) are management events. **Runtime** calls (`InvokeModel`, `Converse`) are recorded too (**verify management vs data event classification**). Use CloudTrail for **who called what, when, with which error code** (`errorCode`: `AccessDenied`, `ValidationException`, `ThrottlingException`). It **does not include prompt bodies**; use invocation logs for those.

#### Amazon Q Developer for GenAI error pattern recognition

- **"Diagnose with Amazon Q"** in the console explains errors (e.g. `AccessDeniedException` because model access isn't enabled or an IAM/SCP denies `bedrock:InvokeModel` for that model ARN / inference profile).
- In the IDE: paste a stack trace or log excerpt → Q explains it and proposes a fix (e.g. a `ValidationException: messages: roles must alternate between user and assistant` payload bug; a malformed tool result; wrong model ID format for an inference profile; read timeout too short).
- **CloudWatch investigations** (AI-assisted ops) correlates alarms, metrics, logs and deployments to suggest a root cause (**verify naming**).

#### Common GenAI errors → likely cause → fix

| Symptom / error | Likely cause | Fix |
| --- | --- | --- |
| `AccessDeniedException` on InvokeModel | Model access not granted / IAM policy missing model or **inference profile ARN** / SCP deny / Marketplace subscription missing | Enable model access; include both the profile ARN and the underlying model ARNs in each Region the profile routes to |
| `ValidationException: input too long` | Context window exceeded | Trim history, fewer RAG chunks, summarise, larger-context model |
| `ValidationException` (roles/format) | Messages don't alternate user/assistant, empty content, wrong body schema for InvokeModel | Use Converse, validate payloads |
| `ValidationException: on-demand throughput isn't supported` | Model requires an **inference profile** (CRIS) ID | Use the `us.`/`eu.`/`global.` profile ID (**verify exact message**) |
| `ThrottlingException` | RPM/TPM quota exceeded | Backoff + jitter, cross-Region inference, quota increase, Provisioned Throughput, batch |
| `ModelTimeoutException` / client read timeout | Long generation | Stream, raise read timeout, lower `maxTokens`, async |
| API GW **504** after ~29 s | Integration timeout | Streaming, WebSocket, async job, or request a timeout increase |
| Answer cut off mid-sentence | `stopReason=max_tokens` | Raise `maxTokens` / continue |
| Empty/odd answer with a guardrail | `guardrail_intervened` | Check guardrail trace (`trace: enabled`), tune policies |
| RAG answer wrong but model fine | Retrieval issue | Log retrieved chunks; check chunking, metadata filters, KB sync (see Domain 1) |
| Agent loops forever | No stopping condition / ambiguous tool descriptions | Max iterations, better tool descriptions, enable agent **trace** |

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Chat UI must show tokens as they are generated; API GW REST times out at 29 s | `ConverseStream` + **Lambda response streaming** / **WebSocket API** (or REST API response streaming) |
| Generation takes several minutes; client can poll | **Async**: API → SQS → worker → S3/DynamoDB result, return job ID (202) |
| Response ends mid-sentence | `stopReason = max_tokens` → raise `maxTokens` or continue the generation |
| Long chat sessions start failing with "input too long" | **Sliding window + summarise older turns** (history in DynamoDB) |
| Which Bedrock errors should be retried? | Throttling, ModelTimeout, ServiceUnavailable, InternalServer, ModelNotReady. **Not** Validation/AccessDenied. |
| boto3 calls to a reasoning model fail at ~60 s | Increase the SDK **read_timeout** (+ adaptive retries) |
| Retried POST creates duplicate CRM records | **Idempotency key** (Powertools idempotency / DynamoDB) |
| Front-end team needs an authenticated chat UI with stored history, minimal code | **Amplify Gen 2 AI Kit**: `a.conversation` + `<AIConversation>` |
| Return structured JSON from a single prompt in an Amplify app | **Amplify generation route** (`a.generation` + `.returns()`) |
| Multiple teams must build against an API before it exists | **OpenAPI-first**: spec → API GW import, mocks, SDK generation, contract tests |
| Agent must call an existing REST API | **Bedrock Agents action group with an OpenAPI schema** (+ Lambda executor or return of control) |
| Agent picks the wrong API operation | Improve the OpenAPI **descriptions** / operationId, reduce the number of operations |
| Business analysts want to chain prompts + KB without code | **Bedrock Flows** (versions + aliases, `InvokeFlow`) |
| Summarise each new support case in Salesforce automatically | **EventBridge partner event source → Lambda → Bedrock → write back** |
| Invoices arrive in S3; extract fields, validate, human review of low-confidence results | **Step Functions**: Textract `AnalyzeExpense` → Bedrock normalise → Choice → **A2I** / `.waitForTaskToken` |
| Extract fields from documents, images, audio and video with minimal orchestration | **Bedrock Data Automation** (standard output / custom blueprints) |
| Extract a specific set of fields from a custom form type with BDA | **Custom output + custom blueprint** in a **project** |
| BDA output for video | **Standard output only** (custom blueprints are documents/images/audio) |
| KB must understand charts/images inside PDFs, and audio/video | **BDA parser** (or FM parser) for the Knowledge Base |
| Step Functions Bedrock payload exceeds 256 KB | Optimised `bedrock:invokeModel` with **S3 input/output** |
| Thousands of documents to process in parallel | **Distributed Map** (+ Bedrock batch inference for offline jobs) |
| Generate unit tests for prompt-handling code | **Q Developer `/test`** |
| Security scan + code review before merge | **Q Developer `/review`** |
| Upgrade a Java 8 GenAI backend to Java 17/21 | **Q Developer `/transform`** |
| Suggestions should follow internal libraries | **Q Developer Pro customizations** |
| Draft → critique → improve, with bounded loops | **Evaluator-optimizer** in Step Functions (Choice + max iterations) |
| Fixed sequence of LLM steps, each validated | **Prompt chaining** (Step Functions / Flows / code) with gates |
| One entry point routing to many specialist agents | **AWS Agent Squad** (classifier routing) |
| See full prompts and responses for debugging | **Model invocation logging** → CloudWatch Logs Insights / S3 + Athena |
| Token usage per IAM principal or team | Logs Insights `stats sum(input.inputTokenCount) by identity.arn` / `requestMetadata` |
| Find which hop is slow: retrieval, rerank or generation | **X-Ray / ADOT** traces + service map (annotations for modelId) |
| Who invoked the model and got AccessDenied? | **CloudTrail** (`errorCode`) |
| Understand a cryptic console error | **Diagnose with Amazon Q** |
| Invocation log images/large bodies missing in CloudWatch | > 100 KB / binary → **S3 only** (configure an S3 large-data location) |

## Flashcards

<details>
<summary><b>Q:</b> A REST API Gateway → Lambda → Bedrock endpoint returns 504 for long answers. What are the options?</summary>

The REST integration timeout is 29 s by default. Options:
- Stream: Lambda response streaming, a WebSocket API, or REST API response streaming.
- Go async: SQS + a job ID + polling or a callback.
- Request a higher integration timeout, which reduces the throttle quota.

*D2 · Task 2.5 · API timeouts*
</details>

<details>
<summary><b>Q:</b> What does <code>stopReason: "max_tokens"</code> mean and how do you handle it?</summary>

The output was truncated at the `maxTokens` limit. You can:
- raise `maxTokens`,
- ask the model to continue from the partial text, or
- request a more concise output format.

*D2 · Task 2.5 · Token limits*
</details>

<details>
<summary><b>Q:</b> A long-running chat starts throwing ValidationException "input is too long". What's the fix?</summary>

The context window has been exceeded, and the error is not retryable. Fixes:
- Keep the system prompt plus the last N turns, and summarise older turns into a running summary stored in DynamoDB.
- Send fewer RAG chunks (rerank first).

*D2 · Task 2.5 · Context management*
</details>

<details>
<summary><b>Q:</b> Name four retryable and two non-retryable Bedrock runtime errors.</summary>

- **Retryable:** ThrottlingException, ModelTimeoutException, ServiceUnavailableException, InternalServerException (also ModelNotReadyException).
- **Non-retryable:** ValidationException, AccessDeniedException.

*D2 · Task 2.5 · Retry strategy*
</details>

<details>
<summary><b>Q:</b> Amplify AI Kit: what is the difference between a conversation route and a generation route?</summary>

- **Conversation route** (`a.conversation`): async and multi-turn. History is stored in DynamoDB and responses stream over AppSync subscriptions.
- **Generation route** (`a.generation`): a single synchronous AppSync query that returns typed, structured output.

*D2 · Task 2.5 · Amplify AI Kit*
</details>

<details>
<summary><b>Q:</b> Which AWS services does the Amplify AI Kit use under the hood?</summary>

- AWS AppSync (API, auth, routing)
- AWS Lambda (conversation handler)
- Amazon DynamoDB (conversation history)
- Amazon Bedrock (Converse API)

Cognito handles user authentication.

*D2 · Task 2.5 · Amplify architecture*
</details>

<details>
<summary><b>Q:</b> Why is OpenAPI central to an API-first GenAI integration?</summary>

One contract drives all of these:
- API Gateway import and request validation
- client SDK generation, docs and mocks
- contract tests
- Bedrock Agents action groups, where the agent reads the operation and parameter descriptions to decide which call to make

*D2 · Task 2.5 · OpenAPI*
</details>

<details>
<summary><b>Q:</b> What are the two ways to define a Bedrock Agents action group, and the two ways to execute it?</summary>

- **Define:** an OpenAPI schema, or function details.
- **Execute:** a Lambda function, or return of control (the parameters go back to your application, which runs the action itself).

*D2 · Task 2.5 · Action groups*
</details>

<details>
<summary><b>Q:</b> When would you choose Bedrock Flows over Step Functions?</summary>

Choose Flows for no-code or low-code GenAI chains built from prompt, KB, agent, condition and Lambda nodes, with fast iteration and versioned aliases.

Choose Step Functions when you need any of these:
- broad AWS integrations
- long-running workflows
- human approval waits
- rich retry and catch handling
- an audit history

*D2 · Task 2.5 · Flows vs Step Functions*
</details>

<details>
<summary><b>Q:</b> Design an automatic case summary for a CRM.</summary>

1. A CRM event arrives through an EventBridge partner event source (or a webhook to API Gateway).
2. It goes into SQS as a buffer.
3. Lambda calls Bedrock for a JSON summary, with Guardrails masking PII.
4. The summary is written back to the CRM via its API. Credentials are kept in Secrets Manager.
5. The Lambda is idempotent and failures go to a DLQ.

*D2 · Task 2.5 · CRM enhancement*
</details>

<details>
<summary><b>Q:</b> In a Step Functions IDP workflow, how do you add human review for low-confidence extractions?</summary>

1. A Choice state checks the confidence against a threshold.
2. If it's below, an Amazon A2I human loop or a `.waitForTaskToken` task notifies a reviewer.
3. The workflow resumes when `SendTaskSuccess` is called with the corrected data.

Use a Standard workflow for this.

*D2 · Task 2.5 · Document processing*
</details>

<details>
<summary><b>Q:</b> Step Functions passes a large document to Bedrock and fails on payload size. What's the fix?</summary>

The state payload limit is 256 KB. Use the optimised `bedrock:invokeModel` integration with S3 input and output locations, or pass S3 references instead of the content itself.

*D2 · Task 2.5 · Step Functions limits*
</details>

<details>
<summary><b>Q:</b> Bedrock Data Automation: standard output vs custom output?</summary>

- **Standard output:** the default per-modality extraction, such as document summaries, audio transcripts and video scene summaries.
- **Custom output:** defined by blueprints listing the fields to extract. Available for documents, images and audio only, not video.

*D2 · Task 2.5 · BDA*
</details>

<details>
<summary><b>Q:</b> What is a BDA project and how is it used?</summary>

A project groups the standard-output settings for each modality plus custom blueprints. You pass it to `InvokeDataAutomationAsync`, and the results are written as JSON to S3.

*D2 · Task 2.5 · BDA projects*
</details>

<details>
<summary><b>Q:</b> When should you pick BDA over a Textract + Comprehend + Bedrock Step Functions pipeline?</summary>

Pick BDA when:
- you want the least orchestration,
- the inputs are multimodal (including audio and video), and
- catalog or simple custom blueprints are enough, with built-in confidence scores and grounding.

Pick DIY for custom branching and approvals, specific Textract APIs, or custom Comprehend models.

*D2 · Task 2.5 · BDA vs DIY*
</details>

<details>
<summary><b>Q:</b> Which Amazon Q Developer commands generate tests, review code, write docs and upgrade Java?</summary>

- `/test`: unit tests
- `/review`: code review and security scan
- `/doc`: documentation
- `/transform`: Java 8/11 → 17/21, and .NET Framework → cross-platform .NET

*D2 · Task 2.5 · Q Developer*
</details>

<details>
<summary><b>Q:</b> A company wants Q Developer suggestions that use its internal SDKs. Which feature?</summary>

Q Developer Pro customizations, which are informed by the company's private repositories.

*D2 · Task 2.5 · Q Developer customizations*
</details>

<details>
<summary><b>Q:</b> How do you implement an evaluator-optimizer pattern safely in Step Functions?</summary>

1. A Generate task produces a draft.
2. An LLM-as-judge task scores it.
3. A Choice state checks whether the score meets the threshold. If not, loop back to Generate with the feedback.

Always add a max-iteration counter and an overall timeout as stopping conditions.

*D2 · Task 2.5 · Agent patterns*
</details>

<details>
<summary><b>Q:</b> Map these to Step Functions constructs: prompt chaining, routing, parallelisation.</summary>

- **Prompt chaining:** sequential Task states with gates.
- **Routing:** a classifier Task followed by a Choice state.
- **Parallelisation:** a Parallel or Map state followed by an aggregation step.

*D2 · Task 2.5 · Agent patterns*
</details>

<details>
<summary><b>Q:</b> Model invocation logging: what is the default state, which destinations are supported, and what happens to large payloads?</summary>

- **Default:** disabled.
- **Destinations:** CloudWatch Logs, S3, or both, in the same account and Region.
- **Large payloads:** bodies up to 100 KB are logged inline. Anything larger, and any binary data, goes to S3 only.

*D2 · Task 2.5 · Invocation logging*
</details>

<details>
<summary><b>Q:</b> Write a Logs Insights query for token usage per caller.</summary>

`fields identity.arn as p, input.inputTokenCount as i, output.outputTokenCount as o | stats sum(i), sum(o), count() by p | sort sum(i) desc`

To break usage down by team, group on `requestMetadata` tags instead.

*D2 · Task 2.5 · Logs Insights*
</details>

<details>
<summary><b>Q:</b> Users report the chatbot is slow. How do you find where the time goes?</summary>

Turn on X-Ray/ADOT tracing across API Gateway, Lambda and Step Functions, with Bedrock and the KB showing as subsegments. Add annotations such as modelId and promptVersion, then look at the service map and trace timelines to compare retrieval, rerank and generation time.

*D2 · Task 2.5 · X-Ray*
</details>

<details>
<summary><b>Q:</b> You need to know who called InvokeModel and got AccessDenied. Invocation logs or CloudTrail?</summary>

CloudTrail. It records the caller identity, time and `errorCode`, but not prompt bodies. Model invocation logging is where the prompt and response content is recorded.

*D2 · Task 2.5 · CloudTrail vs invocation logs*
</details>

<details>
<summary><b>Q:</b> AccessDeniedException when calling a model through a cross-Region inference profile. What are the common causes?</summary>

- The IAM policy covers the profile ARN but not the underlying foundation-model ARNs in every destination Region (or the reverse).
- Model access hasn't been enabled.
- An SCP blocks one of the Regions.

*D2 · Task 2.5 · Troubleshooting*
</details>

---
[← Back to Domain 2](README.md)
