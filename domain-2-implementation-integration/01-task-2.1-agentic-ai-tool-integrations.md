[← Back to Domain 2](README.md)

# 🕹️ Task 2.1 · Implement agentic AI solutions and tool integrations

> **Domain 2 · Implementation and Integration (26%)**
> - Skill 2.1.1: Autonomous systems with memory and state management (Strands Agents, AWS Agent Squad, MCP)
> - Skill 2.1.2: Structured reasoning for complex problems (Step Functions ReAct, chain-of-thought)
> - Skill 2.1.3: Safeguarded AI workflows (stopping conditions, timeouts, IAM boundaries, circuit breakers)
> - Skill 2.1.4: Model coordination systems (specialised FMs, ensembles, model selection frameworks)
> - Skill 2.1.5: Collaborative human + AI systems (Step Functions approvals, API Gateway feedback, human augmentation)
> - Skill 2.1.6: Intelligent, reliable tool integrations (Strands API, standard function definitions, Lambda validation)
> - Skill 2.1.7: Model extension frameworks (MCP servers on Lambda vs ECS, MCP client libraries)

## Key points
- **Agent = FM + tools + memory + loop**, iterating until the model stops asking for tools or a guard fires. Use the **least autonomy** that solves the problem (2.1.1).
- **Platform choice:** managed, low-code → **Bedrock Agents**; code-first with native MCP → **Strands**; production hosting for any framework → **AgentCore**; classifier routing across specialist agents → **Agent Squad**; deterministic and auditable → **Step Functions** (2.1.1).
- **Memory is tiered:** short-term session context, long-term summaries / preferences, and external state in DynamoDB / S3 (2.1.1).
- **Reasoning pattern:** pure reasoning → CoT; needs tools mid-reasoning → ReAct; long tasks → plan-and-execute. Step Functions makes the ReAct loop explicit and auditable (2.1.2).
- **Safeguards stack in layers:** stopping conditions, timeouts, least-privilege IAM, circuit breakers and guardrails (2.1.3).
- **Model coordination:** a specialised FM per sub-task, ensembles for accuracy, config-driven model selection (2.1.4).
- **Human approval → Step Functions Standard + `.waitForTaskToken`.** Express workflows can't wait for humans (2.1.5).
- **The tool description is the prompt.** Tight schemas, validate model-generated args, return structured errors so the model can self-correct, make side effects idempotent (2.1.6).
- **MCP hosting:** stateless and lightweight → **Lambda**; stateful or heavy → **ECS / Fargate**; existing APIs → **AgentCore Gateway** (2.1.7).

## Notes

### Skill 2.1.1 — Autonomous systems with memory & state management

> Develop intelligent autonomous systems with appropriate memory and state management capabilities (for example, by using Strands Agents and AWS Agent Squad for multi-agent systems, MCP for agent-tool interactions).

#### What makes a system "agentic"
- **Autonomy**: the FM decides *which* step comes next. A workflow, by contrast, has code-defined steps.
- **Tool use**: the FM emits structured tool calls and the runtime executes them.
- **Memory**: context persists across turns (short-term) and across sessions (long-term).
- **Goal-directed loop**: the agent keeps iterating until the goal is met or a stop condition fires.
- Spectrum: *single prompt → prompt chain → routed workflow → single agent with tools → multi-agent system*. **Use the least autonomy that solves the problem**: more autonomy means more cost, latency, non-determinism and risk.

#### Choosing the agent platform

| Need | Choose |
| --- | --- |
| Managed, low-code agent with action groups, KBs, guardrails, and console tracing | **Amazon Bedrock Agents** |
| Code-first, model-agnostic, full control of the loop, native MCP, multi-agent patterns | **Strands Agents** (open-source SDK) |
| Production hosting for *any* framework (Strands, LangGraph, CrewAI, LlamaIndex) with session isolation, managed memory, identity and tool gateway | **Amazon Bedrock AgentCore** |
| Route each user message to the best of N specialist agents (intent classification) with shared conversation storage | **AWS Agent Squad** (formerly *Multi-Agent Orchestrator*) |
| Fully deterministic multi-step process with audit, retries and human approval | **AWS Step Functions** (optionally *calling* agents) |

The sections below take each platform in turn, then look at where state lives outside the agent.

#### Strands Agents: the essentials 🎯 PQ61
- Open-source, code-first SDK (Python / TypeScript).
- **Model-driven agent loop.** The `Agent` sends the system prompt, conversation and tool specs to the model. If the model returns `tool_use`, Strands runs the tool, appends the `toolResult` and calls the model again, until `end_turn`.
- The default model provider is **Amazon Bedrock**. Other providers are pluggable (Anthropic, OpenAI, Ollama, LiteLLM, SageMaker and more).
- **Tools:**
  - Python functions with the `@tool` decorator: the **docstring becomes the description** and the **type hints become the JSON schema**.
  - Class-based tools.
  - The `strands-agents-tools` package (http_request, file ops, calculator, `use_aws`, and others).
  - **MCP tools** via `MCPClient`.

```python
from strands import Agent, tool
from strands.tools.mcp import MCPClient
from mcp import stdio_client, StdioServerParameters

@tool
def get_order_status(order_id: str) -> dict:
    """Return the shipping status for an order. Use when the user asks where their order is.

    Args:
        order_id: Order identifier in the form ORD-123456.
    """
    ...

docs_mcp = MCPClient(lambda: stdio_client(
    StdioServerParameters(command="uvx", args=["awslabs.aws-documentation-mcp-server@latest"])))

with docs_mcp:
    agent = Agent(
        model="us.anthropic.claude-sonnet-...",          # Bedrock model / inference profile ID
        system_prompt="You are a support agent...",
        tools=[get_order_status, *docs_mcp.list_tools_sync()],
    )
    result = agent("Where is ORD-123456?")
```

- **Short-term memory = conversation manager:**
  - `SlidingWindowConversationManager` is the default. It keeps the last N messages and trims safely around tool-use / tool-result pairs.
  - `SummarizingConversationManager` summarises older turns.
  - `NullConversationManager` keeps everything (verify the default window size in docs).
- **Persistence = session manager:** `FileSessionManager` and `S3SessionManager` save the agent's messages and state, so a stateless compute instance can resume a session. For managed long-term memory, use the **AgentCore Memory** integration.
- **Agent state:** `agent.state` is a key-value store that is **not sent to the model**. Use it for app data such as a user ID or cart. `invocation_state` passes config to tools and hooks without exposing it to the LLM.
- **Hooks, structured output and direct tool calls** customise the loop. See Skill 2.1.6.
- **Multi-agent patterns** (verified at strandsagents.com):

| Pattern | How it works | Control | Safety knobs | Exam cue |
| --- | --- | --- | --- | --- |
| 🎯 **PQ49** **Agents as tools** | An orchestrator agent gets specialist agents in its `tools` list | Orchestrator LLM decides | Orchestrator's own loop limits | "One coordinator delegating to experts" |
| **Swarm** (`Swarm`) | A pool of peers *hand off* to each other autonomously, with shared context | Emergent (agents decide) | Max handoffs / iterations, execution and node timeouts, repetitive-handoff detection (verify exact param names) | "Unclear which specialist is needed; agents should collaborate and pass work along" |
| **Graph** (`GraphBuilder`) | A flowchart of agent nodes. Edges can be conditional, and **cycles are allowed** | Developer defines the topology, the LLM picks the path | `max_iterations`, `node_timeout`, error-routing edges | "Known process with branches / loops" |
| **Workflow** (`Workflow`) | A fixed **DAG** of tasks. Independent tasks run in parallel, and the whole thing can be exposed as one tool | Deterministic | No cycles; a failure halts dependent tasks | "Fixed pipeline with parallel steps" |
| **A2A** | Agents run as separate services and talk over the Agent-to-Agent protocol | Cross-process / cross-team | Network and auth boundaries | — |

> [!IMPORTANT]
> **🎯 Practice question 61 — autonomous support: high throughput, multi-step reasoning, persistent sessions, MCP tools, supervisor delegation**
>
> - **Strands Agents SDK** with the **Amazon Bedrock model provider** (`BedrockModel`):
>   - **built-in session managers** keep the agent's state and memory between interactions
>   - **native MCP** (`MCPClient`) connects external tools
>   - **supervisor → collaborator** patterns (agents-as-tools) handle delegation
>   - the agent loop does the multi-step reasoning
> - ❌ **Agent Squad + EventBridge:** no native MCP or session managers (per the answer key), and event-driven delegation adds latency.
> - ❌ **LangChain + DynamoDB + custom MCP servers + custom orchestration:** lots of custom work.
> - ❌ **Step Functions + ElastiCache + Lambda tools:** orchestration overhead, no MCP, and session memory is all custom.

> [!IMPORTANT]
> **🎯 Practice question 49 — student advisor delegates to scholarship, internship and mental-health agents, keeps context, metrics, minimal code**
>
> - **Strands Agents supervisor** with **agents-as-tools**: register the specialist agents as tools of the supervisor. Strands' **built-in orchestration** decides when to delegate, and **conversation context stays with the supervisor's session and memory**.
> - **Deploy on AgentCore Runtime** (serverless, session isolation) + **AgentCore Observability** → usage metrics for each specialist.
> - ❌ **Separate agents behind API Gateway + a Lambda router + DynamoDB context:** custom infrastructure.
> - ❌ **Supervisor + Lambda classifier + Step Functions routing:** duplicates what Strands already does.
> - ❌ **Frontend keyword routing:** brittle, and it doesn't understand context.

#### AWS Agent Squad
- Open-source framework (Python and TypeScript) from awslabs. It was formerly **Multi-Agent Orchestrator**.
- **Flow:** user input → **Classifier** (LLM-based, e.g. a Bedrock or Anthropic classifier) picks the best agent using agent *descriptions* and **conversation history** → the selected **Agent** responds → **Storage** saves the conversation per user, session and agent.
- **Built-in agents:** Bedrock LLM agent, **Amazon Bedrock Agents** wrapper, **Amazon Lex bot**, **Lambda** agent, Anthropic / OpenAI agents, Chain agent (verify the current list).
- **Storage:** in-memory (dev), **DynamoDB** (prod), SQL (verify). This is what gives **multi-turn context across agents**: a follow-up like "and what about Tuesday?" is routed correctly because the classifier sees the history.
- **`SupervisorAgent`:** a lead agent coordinates a *team* using an **agent-as-tools** architecture, with **parallel** sub-agent queries and shared context.

#### Amazon Bedrock Agents: memory and state
- **Session (short-term):** a `sessionId` keeps conversation context. `idleSessionTTLInSeconds` ends idle sessions (verify the default and maximum).
  - **`sessionAttributes`** persist for the whole session and are *not* sent to the model by default. They are passed to action-group Lambdas, e.g. a user ID or auth context.
  - 🎯 **PQ73** **`promptSessionAttributes`** are injected into the **prompt** for *one turn only*, e.g. the current date or time zone.
- **Long-term memory:** enable `memoryConfiguration` with type **`SESSION_SUMMARY`**. The agent summarises each session and recalls the summaries in later sessions for the same **`memoryId`**. Retention is set by `storageDays` (verify the max in docs).
- **Multi-agent collaboration:**
  - A **supervisor** agent coordinates **collaborator** agents (each with its own alias).
  - **Supervisor mode** breaks the task down, delegates and synthesises the answer.
  - **Supervisor with routing** sends simple requests *directly* to one collaborator (lower latency) and falls back to full supervision for complex ones.
  - Conversation history can optionally be shared with collaborators.
- **Agent versions and aliases:** use an immutable version plus an alias for deployment and rollback. `TSTALIASID` is the working draft.
- Deep dive: [07 · Agents & multi-agent](<../../study-notes/Certifications/AWS/AIP-C01 - Gen AI Pro/07-agents-multi-agent-frameworks.md>).

> [!IMPORTANT]
> **🎯 Practice question 73 — 25k concurrent sessions <500 ms, remember context within a session only, nothing kept afterwards, hand-off to humans**
>
> - Keep the conversation history **in application memory** and **send it with every `InvokeAgent` call** in `sessionState.promptSessionAttributes`. Use a **unique `sessionId` per customer interaction**.
> - **Clear the in-memory history when the session ends**, which meets the *no retention beyond the session* rule. **Encrypt** the session attributes when handing context to a human agent.
> - This uses only built-in API parameters, with **no extra data store**, so it's the lowest latency and has the least infrastructure.
> - ❌ **Lambda + DynamoDB + TTL:** extra hops and infrastructure, and TTL deletion isn't immediate.
> - ❌ **DynamoDB Streams → archive to S3:** **keeps** conversations, which breaks the policy.
> - ❌ **ElastiCache Redis:** another cluster plus custom code to clear it.
> - ⚠️ The question says "AgentCore", but the mechanism (`InvokeAgent`, `sessionState`) belongs to **Bedrock Agents**. Remember: `promptSessionAttributes` applies to **one turn**, so the app has to **send it again each turn**. `sessionAttributes` lasts for the whole session.

#### Amazon Bedrock AgentCore (brief; details in [08 · AgentCore](<../../study-notes/Certifications/AWS/AIP-C01 - Gen AI Pro/08-bedrock-agentcore.md>))

| Component | Role in memory / state / tools |
| --- | --- |
| 🎯 **PQ4** **Runtime** | Serverless hosting for any framework and model, with **per-session microVM isolation** and long-running sessions: **up to 8 hours on microVMs (up to 14 days on the Instances compute type)**. **Framework-agnostic** (Strands, LangGraph, CrewAI, custom) and **model-agnostic** (Bedrock, Anthropic, Gemini, **OpenAI**). Can also host **MCP servers** and **A2A** agents |
| 🎯 **PQ4** **Memory** | **Short-term** = raw conversation *events* per actor and session. **Long-term** = *strategies* extract **semantic facts, summaries, user preferences** (or custom ones) asynchronously into namespaces, then retrieve them semantically |
| 🎯 **PQ42** **Gateway** | Converts **Lambda, OpenAPI, Smithy** targets (and existing MCP servers) into a single **MCP endpoint**, with **semantic tool search**, inbound auth and outbound credentials |
| 🎯 **PQ20** 🎯 **PQ4** **Identity** | Inbound auth with corporate IdPs (**Okta**, Microsoft Entra ID, Amazon Cognito, any OIDC IdP), plus outbound OAuth / API keys to third-party tools through a **token vault**, on behalf of the user |
| **Code Interpreter / Browser** | Sandboxed code execution and managed headless browser tools |
| 🎯 **PQ4** **Observability** | OpenTelemetry traces of agent steps into CloudWatch |

> [!IMPORTANT]
> **🎯 Practice question 42 — support PoC: use 10k tickets + 500 docs, take resolution actions, low latency, real-time metrics**
>
> - **Strands Agents agent on AgentCore Runtime** → reasoning **plus actions**.
> - **Knowledge base** with the product docs **and** the historical tickets → RAG.
> - **AgentCore Gateway** exposes the **ticket-management APIs as MCP tools** → the agent can update tickets and start processes. That's what **cuts resolution time**.
> - **AgentCore Observability + CloudWatch** → resolution time and accuracy compared with the manual baseline.
> - ❌ **KB + FM only:** answers questions but **can't take actions**.
> - ❌ **Prompt template only:** no RAG over the docs or tickets.
> - ❌ **Multi-model voting ensemble:** high latency.

> [!WARNING]
> **🎯 Practice question 4 — Strands + OpenAI agent, >1 h runs, Okta, memory, MOST operationally efficient**
>
> - Keep the **existing Strands + OpenAI agent as it is** and **deploy it to AgentCore Runtime**. Add **AgentCore Memory** (context), **AgentCore Identity** (Okta) and the built-in **AgentCore Observability**.
> - Why (see the table above): Runtime is framework- and model-agnostic, so **no refactor**; its long sessions cover **"> 1 hour"**; Identity supports **Okta** for inbound auth.
> - ❌ **Switching to Anthropic models or CrewAI** forces a refactor.
> - ❌ **ECS/Fargate** means managing containers and scaling yourself.
> - ❌ **Lambda + custom Okta and memory code** hits the 15-minute limit and adds a lot of custom work.
> - *Source: AgentCore Runtime developer guide, checked 2026-11-15.*

#### MCP for agent-tool interactions
Whichever platform runs the agent, MCP is how it reaches shared tools.
- MCP standardises how agents **discover** (`tools/list`) and **call** (`tools/call`) tools. You write a tool once, and any MCP client can use it: Strands, Amazon Q Developer, Claude Desktop, LangChain adapters, and others.
- It decouples the **agent** (reasoning) from the **tool** (implementation), so each can be versioned, secured and scaled independently. See Skill 2.1.7.

#### State management outside the agent
Framework memory covers the conversation; everything else needs an explicit store.

| State | Store |
| --- | --- |
| Conversation history (custom apps) | **DynamoDB**: PK `sessionId`, SK `timestamp#turn`, **TTL** for expiry. See [Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) |
| Long-running workflow progress | **Step Functions** execution state, or DynamoDB checkpoints |
| Large artefacts (documents, generated files) | **S3**, with a pointer in state (the Step Functions payload limit is **256 KB**) |
| Semantic long-term memory | **AgentCore Memory**, or a vector store (OpenSearch / S3 Vectors) of past interactions |
| Low-latency session cache | **ElastiCache / MemoryDB** |

### Skill 2.1.2 — Structured reasoning: ReAct & chain-of-thought

> Create advanced problem-solving systems to give FMs the ability to break down and solve complex problems by following structured reasoning steps (for example, by using Step Functions to implement ReAct patterns and chain-of-thought reasoning approaches).

#### Reasoning patterns

| Pattern | Idea | When |
| --- | --- | --- |
| **Chain-of-thought (CoT)** | "Think step by step"; the model writes its intermediate reasoning before the answer | Maths, logic, multi-constraint decisions. Pure reasoning, no tools |
| **ReAct** (Reason + Act) | Interleaves *Thought → Action (tool) → Observation*, then repeats | You need external info or actions mid-reasoning |
| **Plan-and-execute** | A planner LLM writes a full plan, then an executor runs the steps (and can re-plan) | Long tasks; cheaper models can execute; easier to audit the plan |
| **Decomposition / least-to-most** | Split the question into sub-questions and solve them in order | Multi-hop Q&A, complex RAG (query decomposition, [Task 1.5](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md)) |
| **Reflection / self-critique** | Generate → critique → revise | Quality-sensitive output (code, reports) |
| **Tree-of-thought / self-consistency** | Sample several reasoning paths, then vote or search | High-stakes reasoning; costs N× more |
| **Extended thinking** | Built-in model reasoning budget (e.g. Claude extended thinking via `additionalModelRequestFields`) | Hard reasoning without hand-built CoT prompts. Raises latency and cost |

- Modern Bedrock Agents orchestration *is* ReAct-style by default. Trace `rationale` shows the "thought", `invocationInput` the "action", and `observation` the result.

#### ReAct as a Step Functions state machine
Why Step Functions instead of an in-code loop:
- **explicit, auditable steps**
- built-in **retries and catch**
- **max-iteration guard**
- per-step timeouts
- parallel tool calls with **Map**
- easy insertion of **human approval**
- visual debugging in execution history

```
Init (iteration=0, messages=[user])
  → CallModel   (arn:aws:states:::bedrock:invokeModel  — or Lambda calling Converse with toolConfig)
  → CheckStop   (Choice)
       stopReason == "end_turn"          → Succeed (final answer)
       iteration >= MaxIterations        → Fallback / Fail ("max reasoning steps")
       stopReason == "tool_use"          → ExecuteTools
  → ExecuteTools (Map over toolUse blocks → Lambda per tool; Retry + Catch → error toolResult)
  → AppendResults (Pass/Lambda: add assistant msg + toolResult msgs; iteration += 1)
  → CallModel  (loop)
```

```json
"CheckStop": {
  "Type": "Choice",
  "Choices": [
    { "Variable": "$.iteration", "NumericGreaterThanEquals": 8, "Next": "MaxStepsReached" },
    { "Variable": "$.model.stopReason", "StringEquals": "tool_use", "Next": "ExecuteTools" }
  ],
  "Default": "ReturnAnswer"
}
```

- **Payload limit 256 KB:** keep long conversations in S3 or DynamoDB and pass a pointer.
- **CoT inside a step:** the prompt asks for reasoning inside `<thinking>` tags and the answer inside `<answer>` tags. A Lambda or `Pass` step parses out only the answer for the user, and the reasoning is logged for audit.
- **Plan-and-execute in Step Functions:**
  1. A Planner step returns a JSON array of steps.
  2. A **Map** state runs the independent steps in parallel.
  3. An optional re-plan step runs on failure.
  4. A Synthesise step builds the final answer.
- **Prompt chaining** (fixed sequence, no autonomy) is the simpler sibling. Use Bedrock Prompt Flows or Step Functions. See [Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) and Task 2.5.5.

#### Gotchas
- CoT **raises token use and latency**. Use it only where accuracy matters, and consider a *smaller* model plus CoT versus a *larger* model without it.
- Don't show raw reasoning to end users if it may contain sensitive intermediate data. Apply guardrails to the *final* output.
- ReAct loops can **oscillate**, calling the same tool with the same args. Detect repeats and break out (see 2.1.3).

### Skill 2.1.3 — Safeguarded AI workflows

> Develop safeguarded AI workflows to ensure controlled FM behavior (for example, by using Step Functions to implement stopping conditions, Lambda functions to implement timeout mechanisms, IAM policies to enforce resource boundaries, circuit breakers to mitigate failures).

#### Failure modes to guard against
- Infinite or long loops; repeated identical tool calls; runaway **token / cost** spend.
- Tool hangs or slow downstream APIs; throttling cascades.
- **Over-privileged** agents (prompt injection → destructive action); **excessive agency**.
- Bad outputs: hallucinated tool args, policy-violating content.

#### Stopping conditions (Step Functions and frameworks)

| Condition | Implementation |
| --- | --- |
| **Max iterations / steps** | Counter in state + `Choice` (`NumericGreaterThanEquals`); Strands Graph `max_iterations`; Swarm max handoffs |
| **Goal reached** | `stopReason == end_turn`, or a validator Lambda confirms the output meets the schema / criteria |
| **Wall-clock limit** | State machine `TimeoutSeconds` (whole execution), per-Task `TimeoutSeconds`, `HeartbeatSeconds` for long callbacks |
| **Token / cost budget** | Sum `usage.inputTokens + outputTokens` from each Converse response in state, then stop at the budget |
| **Repetition** | Hash `(toolName, args)`. If seen *k* times, stop or escalate (Strands Swarm has repetitive-handoff detection) |
| **Confidence / safety** | Guardrail intervention (`stopReason = guardrail_intervened`) or low confidence → stop and **route to a human** |

#### Timeouts with Lambda
- Lambda's maximum timeout is **900 s (15 min)**. Set the function timeout *below* the caller's timeout. With API Gateway → Lambda the integration timeout is **29 s by default**; it can be raised for Regional/private REST APIs via a quota request (verify).
- **Inside the function:** use `context.get_remaining_time_in_millis()` to abort gracefully. Set **SDK client timeouts and retries** with botocore `Config(read_timeout=..., connect_timeout=..., retries={"max_attempts":..., "mode":"adaptive"})` so a slow model call fails fast instead of consuming the whole Lambda timeout (retry config detail: [Task 2.4 · Skill 2.4.3](04-task-2.4-fm-api-integrations.md#skill-243--resilient-fm-systems)).
- **Long FM jobs** (> 15 min) → Step Functions Standard, Bedrock **batch inference** (async), or the SageMaker **async** endpoint, rather than one long Lambda.

#### IAM resource boundaries
The agent angle: assume a prompt-injected agent will try anything its role allows, so the role is the real boundary.
- **Least privilege:** the agent role may invoke **only specific model or inference-profile ARNs**. Each tool Lambda gets its own narrowly scoped role; give the agent read-only roles where possible.
- **Permission boundaries** cap the *maximum* permissions of roles that developers or agents can create. This is essential if an agent can create IAM entities or deploy resources. **SCPs** back this up org-wide (e.g. deny unapproved Regions or model IDs).
- 🎯 **PQ20** **Session policies / scoped-down STS credentials:** the agent assumes a role *on behalf of the user*, with a session policy or **ABAC tags** (`aws:PrincipalTag/tenant`) so it can reach only that tenant's data.
- **Resource policies:** the Lambda resource policy allows invocation only by the specific Bedrock agent ARN (`aws:SourceArn`) to prevent confused-deputy attacks.
- **High-impact actions:** require **user confirmation** or human approval (2.1.5), never full autonomy.
- → Full detail: [Task 2.3 · Skill 2.3.3](03-task-2.3-enterprise-integration-architectures.md#skill-233--secure-access-frameworks) (Bedrock least-privilege policies) and [Task 3.3 · Skill 3.3.3](../domain-3-ai-safety-security-governance/03-task-3.3-governance-compliance.md#skill-333--organisational-governance-systems) (SCPs, multi-account governance).

#### Circuit breakers
- Wrap flaky **tools and models** the agent depends on in a circuit breaker (**closed → open → half-open**, state in **DynamoDB**, checked by Step Functions or a Lambda wrapper). This stops retry storms and cascading latency inside the agent loop.
- When the circuit is open, take a **fallback** path: alternate model or Region, cached answer, a **no-tool answer**, human handoff, or a "try later" message. Combine with backoff + jitter and cross-Region inference.
- → Full detail: [Task 1.2 · Skill 1.2.3](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md#skill-123--resilient-ai-systems) (circuit breaker, CRIS, graceful degradation).

#### Other safeguards
- **Guardrails** on input (prompt attacks, denied topics) and output (PII, grounding). Apply them to Bedrock Agents via the agent config, or call `ApplyGuardrail` from any framework.
- **Allow-listed tools per agent**: an agent cannot call what it isn't given. With AgentCore Gateway, expose only the tools each agent needs.
- **Dry-run / simulation mode** for destructive tools; **idempotency keys**; **audit logs** (CloudTrail plus model invocation logging).

### Skill 2.1.4 — Model coordination systems

> Create sophisticated model coordination systems to optimize performance across multiple capabilities (for example, by using specialized FMs to perform complex tasks, custom aggregation logic for model ensembles, model selection frameworks).

#### Specialised FMs per sub-task

| Sub-task | Typical model choice |
| --- | --- |
| Intent classification, routing, extraction | Small, fast model (e.g. Nova Micro/Lite, Claude Haiku), or **Amazon Comprehend** custom classifier |
| Complex reasoning, planning, code | Frontier model (e.g. Claude Sonnet/Opus class, Nova Premier) |
| Embeddings | Titan Text Embeddings V2, Cohere Embed, Nova multimodal embeddings |
| Image generation / editing | Nova Canvas, Stability models |
| Video | Nova Reel |
| Speech | Amazon Transcribe (STT), Amazon Polly (TTS), Nova Sonic (speech-to-speech) |
| Document extraction | **Bedrock Data Automation**, Textract |
| Reranking | Cohere Rerank / Amazon Rerank on Bedrock |

- **Coordination topology:**
  - A **router / orchestrator** (code, Step Functions, or a supervisor agent) dispatches to specialists and **aggregates** the results.
  - Use the cheapest model that meets the quality bar for each step. This is **model cascading**: try a small model first and escalate on low confidence. See Task 2.2.3.

#### Ensemble aggregation logic

| Technique | How | Use |
| --- | --- | --- |
| **Majority / plurality vote** | N models (or N samples) classify, then take the mode | Classification, yes/no, extraction with discrete answers |
| **Weighted vote** | Weight by historical accuracy or confidence | Models with known strengths |
| **Self-consistency** | Same model, temperature > 0, N samples, then vote on the final answer | Reasoning accuracy |
| **LLM-as-judge / ranker** | A judge model scores the candidate answers and picks the best | Open-ended generation |
| **Mixture-of-agents / fusion** | An aggregator model *synthesises* the candidates into one answer | Long-form, research |
| **Agreement check** | Answers disagree → flag low confidence → human review | High-stakes outputs |

- Implement with **Step Functions `Parallel` / `Map`**: fan out to models, then an aggregation Lambda. Watch the **N× cost** and set timeouts per branch (the slowest model determines total latency, so consider "first k of N").

#### Model selection frameworks
- **Static config** maps a task type to a model ID in **AWS AppConfig** or Parameter Store, so you can switch models without redeploying. AppConfig gives validation, gradual rollout and rollback. See Skill 1.2.2.
- **Bedrock Intelligent Prompt Routing:** one *prompt router* ARN routes each request **between models in the same family** based on predicted response quality, to cut cost (you set a fallback model and a quality-difference threshold; verify the supported families).
- **Dynamic / metric-based:** the router reads CloudWatch metrics (latency, error rate, cost per request) or evaluation scores, then picks a model. Content-based routing with Step Functions `Choice` is covered in Task 2.4.4.
- Underpin the choice with an **offline evaluation**: Bedrock model evaluation, LLM-as-judge, or RAG evaluation. Record the selection rationale (model cards).

### Skill 2.1.5 — Collaborative AI: human + FM

> Develop collaborative AI systems to enhance FM capabilities with human expertise (for example, by using Step Functions to orchestrate review and approval processes, API Gateway to implement feedback collection mechanisms, human augmentation patterns).

#### Human augmentation patterns

| Pattern | Description | Example |
| --- | --- | --- |
| **Human-in-the-loop (approval gate)** | The AI proposes and a human must approve before the action | Refund over $500, sending a contract, a production change |
| **Human-on-the-loop** | The AI acts and humans monitor and can intervene or roll back | Auto-replies with sampled QA |
| **Confidence-based escalation** | Below a threshold → human review; otherwise automatic | Document extraction (A2I), claim triage |
| **Human as tool** | The agent can "ask a human / expert" as a tool | Ambiguous request → clarify with an SME |
| **Feedback loop** | Users rate outputs and the data improves prompts, evals and fine-tuning | Thumbs up/down → DPO dataset |
| **Co-pilot / draft-and-edit** | The AI drafts and the human edits the final version | Marketing copy, code (Q Developer) |

#### Step Functions approval workflow (`.waitForTaskToken`)

```
GenerateDraft (Lambda → Bedrock)
  → RequestApproval  Resource: arn:aws:states:::lambda:invoke.waitForTaskToken
       Payload: { draft, taskToken: $$.Task.Token }  → Lambda stores token (DynamoDB) + notifies reviewer (SNS email / Slack / UI)
       HeartbeatSeconds / TimeoutSeconds  (e.g. 48h → escalate)
  Reviewer clicks Approve/Reject → API Gateway → Lambda → SendTaskSuccess(taskToken, output) | SendTaskFailure
  → Choice(approved?) → Execute action | Revise (loop back with reviewer comments)
```

- Requires a **Standard** workflow, which can wait **up to 1 year**. **Express workflows don't support `.waitForTaskToken` or `.sync`** and are limited to **5 min**, so they can't wait for humans.
- Store **who approved what and when** for audit (DynamoDB + CloudTrail).
- `.waitForTaskToken` also works with **SQS, SNS, ECS** and other integrations: you put the token in the message.

#### Managed human review services
- **Amazon Augmented AI (A2I):** human review **workflows (flow definitions)** with **activation conditions**, e.g. confidence < 0.8.
  - Built-in integrations for **Textract** (AnalyzeDocument) and **Rekognition** (content moderation). **Custom task types** cover anything else, including FM outputs.
  - Work teams can be **private** (your staff), **vendor**, or **Mechanical Turk**.
- **SageMaker Ground Truth:** labelling and **RLHF-style preference data** collection.
- **Bedrock model evaluation with human workers:** human judges rate model outputs (your team or AWS-managed).
- **Bedrock Agents:**
  - **User confirmation** (`requireConfirmation: ENABLED` on a function): the agent asks the user to confirm before invoking the action.
  - **`AMAZON.UserInput`**: the agent can ask the user clarifying questions.
  - **Return of control** (Skill 2.1.6): your app can insert human approval before executing.

#### Feedback collection with API Gateway

```
UI 👍/👎 + comment → API Gateway (POST /feedback, request validation, Cognito auth)
  → Lambda → DynamoDB (requestId, promptVersion, modelId, rating, comment)
           → Kinesis Data Firehose → S3 (analytics: Athena/QuickSight)
  → CloudWatch custom metric: thumbsDownRate per promptVersion → alarm
  → Curated negatives → eval dataset / prompt fixes / fine-tuning (DPO) data
```

- Log a **correlation ID** (`requestId`) with each model invocation so feedback can be joined to the prompt, retrieved context and model version.

### Skill 2.1.6 — Intelligent & reliable tool integrations

> Implement intelligent tool integrations to extend FM capabilities and to ensure reliable tool operations (for example, by using the Strands API to implement custom behaviors, standardized function definitions, Lambda functions to implement error handling and parameter validation).

#### Standardised function definitions (Converse API)

```json
"toolConfig": {
  "tools": [{
    "toolSpec": {
      "name": "create_refund",
      "description": "Create a refund for a delivered order. Only use after the user confirms the amount. Amount must not exceed the order total.",
      "inputSchema": { "json": {
        "type": "object",
        "properties": {
          "order_id": { "type": "string", "pattern": "^ORD-[0-9]{6}$" },
          "amount":   { "type": "number", "minimum": 0.01 },
          "reason":   { "type": "string", "enum": ["damaged", "late", "not_received", "other"] }
        },
        "required": ["order_id", "amount", "reason"]
      }}
    }
  }],
  "toolChoice": { "auto": {} }
}
```

- **`toolChoice`:**
  - `auto`: the model decides.
  - `any`: the model must call *some* tool.
  - `tool: {name}`: force a specific tool. A handy trick for **guaranteed structured JSON output**.
- **Flow:**
  1. The response has `stopReason: "tool_use"` and `toolUse` blocks (`toolUseId`, `name`, `input`).
  2. You execute the tool.
  3. You send a user message with `toolResult` (`toolUseId`, `content` [json/text], `status: "success" | "error"`).
- **Good definitions:**
  - Clear verb-noun names.
  - Descriptions that say **when to use and when not to**.
  - Tight schemas (enums, patterns, required fields).
  - **Few tools per agent**: too many similar tools confuse selection. Use AgentCore Gateway semantic search, or split the tools across agents.
- **Bedrock Agents action groups:**
  - Define with an **OpenAPI schema** or **function details** (simpler).
  - The executor is a **Lambda** function or **`RETURN_CONTROL`**.
  - The Lambda gets `actionGroup`, `function` / `apiPath`, `parameters` and session attributes, and must return the documented response structure.

#### Return of control (verified in the Bedrock docs)
- Set `actionGroupExecutor.customControl = "RETURN_CONTROL"`. The `InvokeAgent` response then contains `returnControl.invocationInputs` (function or API details plus parameters) and an **`invocationId`**.
- Your app runs the action (in your own environment, perhaps after a human approval or via a private-network API) and calls `InvokeAgent` again with `sessionState.returnControlInvocationResults` and the **same `invocationId`**. `inputText` is ignored when results are included.
- Use it when the action **must run client-side**, needs **user approval**, needs credentials the agent shouldn't hold, or the call takes longer than a Lambda can run.

#### Lambda error handling and parameter validation
- **Validate** before acting:
  - Use a JSON Schema check (Powertools for AWS Lambda `validation`, or Pydantic models) for types, ranges, enums and business rules (e.g. amount ≤ order total).
  - **Never trust model-generated args.** Treat them like untrusted user input: they could come from prompt injection.
- **Return structured, model-readable errors** instead of throwing, so the model can **self-correct**:
  ```json
  {"toolResult": {"toolUseId": "t1", "status": "error",
    "content": [{"json": {"error": "INVALID_ORDER_ID", "message": "order_id must match ORD-123456 format", "retryable": true}}]}}
  ```
- Classify errors:
  - **validation** → tell the model what to fix
  - **transient** (throttling, timeouts) → retry with backoff *inside* the tool
  - **permanent / auth** → stop and escalate
- **Idempotency** for side-effecting tools (Powertools `idempotent` decorator + DynamoDB) protects against model or loop retries creating duplicate refunds or orders.
- **Timeouts** on downstream calls. **DLQ / on-failure destinations** for async invocations. Structured logging (Powertools logger / tracer → **X-Ray**).

#### Strands API for custom behaviours
- Tools are `@tool` functions or class-based tools (2.1.1). A tool can receive a `ToolContext` / `invocation_state` for request-scoped data (user ID, credentials) **not visible to the LLM**.
- **Hooks** (before/after invocation, model call and tool call) let you:
  - **validate or rewrite tool inputs**
  - **block tools** (policy) and add guardrails
  - add approval prompts
  - log or emit metrics
  - cap tool-call counts
  (verify the hook event class names).
- **Structured output:** `agent.structured_output(PydanticModel, prompt)` returns a validated object.
- **Retries / model config:** set the Bedrock model's boto config (retries, timeouts), temperature and max tokens per agent.
- **Direct tool calls:** `agent.tool.<name>(...)` invokes a tool deterministically, without the LLM. Useful for tests and fixed steps.

### Skill 2.1.7 — Model extension frameworks with MCP

> Develop model extension frameworks to enhance FM capabilities (for example, by using Lambda functions to implement stateless MCP servers that provide lightweight tool access, Amazon ECS to implement MCP servers that provide complex tools, MCP client libraries to ensure consistent access patterns).

#### MCP fundamentals (verified on modelcontextprotocol.io)
- **Architecture:** a **host** (the AI app) runs **MCP clients**, each connected 1:1 to an **MCP server**. Messages are **JSON-RPC 2.0** (UTF-8).
- **Server primitives:**
  - **Tools** are model-controlled actions (`tools/list`, `tools/call`).
  - **Resources** are app-controlled read-only context (files, records) addressed by URI.
  - **Prompts** are user-controlled reusable templates.
- **Client primitives:** **sampling** (the server asks the client's LLM for a completion), **roots**, **elicitation** (the server asks the user for input).
- **Transports:**

| Transport | How | Use |
| --- | --- | --- |
| **stdio** | The client launches the server as a **subprocess** and exchanges newline-delimited JSON-RPC on stdin / stdout (logs on stderr) | Local tools: IDE, desktop, a dev's machine |
| **Streamable HTTP** | A single **MCP endpoint** (e.g. `/mcp`) supports POST (client → server) and GET (optional server → client **SSE** stream). The server may return JSON *or* an SSE stream. An optional **`Mcp-Session-Id`** header gives stateful sessions. A `MCP-Protocol-Version` header is required | Remote / shared servers on AWS |
| ~~HTTP + SSE~~ | The 2024-11-05 transport. **Replaced** by Streamable HTTP in 2025 | Legacy compatibility only |

- **Security (spec):** validate the `Origin` header (DNS rebinding), bind local servers to 127.0.0.1, and **authenticate** every connection (the spec uses OAuth 2.1 for HTTP authorisation).

#### Lambda vs ECS (and other hosts) for MCP servers

| Factor | **Lambda** (stateless MCP) | **ECS / Fargate** (stateful / complex MCP) |
| --- | --- | --- |
| Session model | **Stateless** Streamable HTTP (no `Mcp-Session-Id`, every request self-contained) | Long-lived sessions, SSE streams, server-initiated notifications |
| Workload | Lightweight lookups, CRUD wrappers, API proxies | Heavy deps (headless browser, ML libs, large models), connection pools, caches, long operations |
| Duration | ≤ 15 min per invocation | Unbounded |
| Scaling / cost | Scale-to-zero, pay per request | Always-on tasks behind an **ALB** (sticky sessions if stateful); Service Auto Scaling |
| Front door | **Function URL** or **API Gateway** (+ Cognito / IAM / Lambda authoriser) | ALB (+ Cognito / OIDC), or API Gateway via VPC Link |

- **Alternatives:**
  - **AgentCore Gateway** (see the 2.1.1 table): point it at existing **Lambda functions / OpenAPI / Smithy** APIs and it *becomes* an MCP server, with **inbound OAuth**. Lowest effort for "expose our existing APIs as MCP tools".
  - **AgentCore Runtime**: hosts MCP servers with session isolation.
  - **EKS**: if the platform standard is Kubernetes.
- **AWS-published MCP servers** (awslabs/mcp: AWS docs, CDK, pricing, Bedrock KB retrieval, and more) can be run locally over stdio or hosted.

#### MCP client libraries for consistent access
- Official **MCP SDKs** (Python, TypeScript, Java, and others) give a `ClientSession` with `initialize`, `list_tools` and `call_tool`.
- **Framework adapters:**
  - **Strands `MCPClient`**: list the MCP tools and pass them as agent tools.
  - **LangChain / LangGraph MCP adapters**, LlamaIndex.
  - **Amazon Q Developer** as an MCP host.
- **Pattern:** a **shared internal client wrapper** library that standardises:
  - auth (token fetch from Cognito / AgentCore Identity)
  - retries and timeouts
  - tool allow-lists
  - logging and tracing
  Every team then calls tools the same way. Combine it with a **central MCP registry / gateway** for discovery and governance.

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async with streamablehttp_client("https://tools.example.com/mcp",
                                 headers={"Authorization": f"Bearer {token}"}) as (r, w, _):
    async with ClientSession(r, w) as s:
        await s.initialize()
        tools = await s.list_tools()
        res = await s.call_tool("get_order_status", {"order_id": "ORD-123456"})
```

#### MCP vs other tool mechanisms

| Mechanism | Best when |
| --- | --- |
| **Converse `toolConfig`** (inline tools) | A single app owns both the tools and the loop |
| **Bedrock Agents action groups** | A managed agent; tools are Lambda / OpenAPI |
| **MCP server** | Tools **shared across many agents, teams or clients**; vendor-neutral; discoverable |
| **A2A** | Agent ↔ agent delegation (peers with their own reasoning), not tool calls |

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Managed agent with tools, KB and guardrails, minimal code | **Bedrock Agents** (action groups + KB + guardrail) |
| Code-first, model-agnostic agent SDK with native MCP and multi-agent patterns | **Strands Agents** |
| Run an existing LangGraph/CrewAI/Strands agent in production with session isolation | **AgentCore Runtime** |
| Expose existing internal REST APIs / Lambda functions as MCP tools with OAuth, without rewriting them | **AgentCore Gateway** |
| Agent must call third-party SaaS on behalf of the user with the user's OAuth token | **AgentCore Identity** (token vault, outbound OAuth) |
| Remember user preferences across sessions (managed) | **AgentCore Memory** long-term strategies, or Bedrock Agents **`SESSION_SUMMARY`** memory |
| Pass a user ID to the action Lambda without sending it to the model | Bedrock Agents **`sessionAttributes`** (Strands: `invocation_state` / agent state) |
| Inject today's date into the prompt for one turn | **`promptSessionAttributes`** |
| Route each message to billing/tech/sales specialist agents, keeping context across turns | **Agent Squad** classifier + DynamoDB storage (or Bedrock multi-agent **supervisor with routing**) |
| Peer agents should hand off work to each other autonomously | Strands **Swarm** |
| Known process with conditional branches and loops between agents | Strands **Graph** |
| Fixed pipeline, independent steps in parallel, exposed as one tool | Strands **Workflow** |
| Auditable ReAct loop with retries and a max-steps limit | **Step Functions**: Choice on `stopReason`, iteration counter, Map for tools |
| Agent loops forever calling the same tool | Max-iteration counter + repetition detection → stop or escalate |
| Third-party tool API keeps failing and causing retry storms | **Circuit breaker** (DynamoDB state + Step Functions fallback) + backoff with jitter |
| Agent role must never exceed a defined permission set, even if it creates roles | **IAM permission boundary** (+ SCP org-wide) |
| Restrict the agent to approved models only | IAM / SCP `Resource` limited to specific **model / inference-profile ARNs** |
| Combine three model answers to improve classification accuracy | Step Functions **Parallel** → majority-vote aggregation Lambda |
| Cut cost by routing simple prompts to a smaller model in the same family automatically | **Bedrock Intelligent Prompt Routing** |
| Switch the model for a task without redeploying | Model ID in **AWS AppConfig** (validated, gradual rollout) |
| Human must approve the action before execution; may take days (or: an Express workflow needs an approval step) | Step Functions **Standard** + **`.waitForTaskToken`** + `SendTaskSuccess` from an API Gateway approval endpoint. ❌ Express doesn't support it |
| Review low-confidence Textract extractions by humans | **Amazon A2I** human review workflow with an activation condition |
| Bedrock agent must ask before executing a booking | Function **user confirmation** (`requireConfirmation`) |
| Collect thumbs up/down to improve prompts | **API Gateway** → Lambda → DynamoDB/Firehose → S3; CloudWatch metric per prompt version |
| Force the model to return JSON matching a schema | Define a tool with that `inputSchema` + **`toolChoice: {tool: {name}}`** |
| Action must run in the client app / on-premises after approval | **Return of control** (`RETURN_CONTROL`, reply with `returnControlInvocationResults` + same `invocationId`) |
| Model generates invalid tool args | Validate in the Lambda (JSON Schema / Powertools) → return `toolResult` with **`status: "error"`** and a fix-hint |
| Duplicate refunds when the agent retries | **Idempotency** (Powertools + DynamoDB idempotency key) |
| Local developer tool for an IDE assistant | MCP over **stdio** |
| The HTTP+SSE MCP transport | Deprecated → **Streamable HTTP** (`Mcp-Session-Id`) |
| Lightweight, stateless MCP tools with scale-to-zero | **MCP server on Lambda** (Streamable HTTP, stateless) behind a Function URL / API Gateway |
| MCP server needs a headless browser, large deps, persistent sessions | **MCP server on ECS Fargate** behind an ALB (or AgentCore Runtime) |

## Flashcards

<details>
<summary><b>Q:</b> What are the four Strands multi-agent orchestration patterns (besides A2A), and when do you use each?</summary>

- **Agents as tools**: an orchestrator delegates to specialist agents.
- **Swarm**: peers hand off to each other autonomously.
- **Graph**: a developer-defined flowchart with conditional edges and loops.
- **Workflow**: a fixed DAG with parallel independent tasks.

*D2 · Task 2.1 · Strands patterns*
</details>

<details>
<summary><b>Q:</b> In Strands, how does a Python function become a tool, and where do its description and schema come from?</summary>

Add the `@tool` decorator. The docstring becomes the tool description (including the Args section) and the type hints become the JSON input schema.

*D2 · Task 2.1 · Strands tools*
</details>

<details>
<summary><b>Q:</b> What handles short-term memory in a Strands agent, and what handles persistence across invocations?</summary>

A **conversation manager** handles short-term memory. The default is the sliding window; there are also summarising and null managers. A **session manager** (File / S3) handles persistence. For managed long-term memory, use AgentCore Memory.

*D2 · Task 2.1 · Strands memory*
</details>

<details>
<summary><b>Q:</b> What is AWS Agent Squad and how does it pick which agent answers?</summary>

It's an open-source multi-agent orchestrator, formerly Multi-Agent Orchestrator. A classifier (usually LLM-based) uses the agents' descriptions plus the conversation history to choose the agent. Conversations are stored per user, session and agent, e.g. in DynamoDB. It also has a `SupervisorAgent` that coordinates a team as agents-as-tools.

*D2 · Task 2.1 · Agent Squad*
</details>

<details>
<summary><b>Q:</b> Bedrock Agents: what's the difference between `sessionAttributes` and `promptSessionAttributes`?</summary>

`sessionAttributes` persist for the session and are passed to action-group Lambdas, but not into the prompt. `promptSessionAttributes` are injected into the prompt for a single turn.

*D2 · Task 2.1 · Bedrock Agents state*
</details>

<details>
<summary><b>Q:</b> How do you give a Bedrock agent memory across sessions?</summary>

Enable `memoryConfiguration` with `SESSION_SUMMARY` and pass the same `memoryId` per user. The agent stores session summaries for `storageDays` and recalls them in later sessions.

*D2 · Task 2.1 · Bedrock Agents memory*
</details>

<details>
<summary><b>Q:</b> Bedrock multi-agent collaboration: supervisor vs supervisor-with-routing?</summary>

**Supervisor** breaks the task down, delegates to collaborators and synthesises the answer. **Supervisor with routing** sends simple requests straight to the right collaborator for lower latency, and falls back to full supervision for complex ones.

*D2 · Task 2.1 · Multi-agent collaboration*
</details>

<details>
<summary><b>Q:</b> Chain-of-thought vs ReAct vs plan-and-execute?</summary>

- **CoT**: step-by-step reasoning with no tools.
- **ReAct**: interleaved thought → tool action → observation.
- **Plan-and-execute**: plan everything first, then run the steps (parallel or cheaper), re-planning on failure.

*D2 · Task 2.1 · Reasoning patterns*
</details>

<details>
<summary><b>Q:</b> Sketch a ReAct loop in Step Functions.</summary>

CallModel (Converse with tools) → Choice: if `stopReason` is `end_turn`, return the answer; if the iteration count is at or above the max, take the fallback path; if it's `tool_use`, go to the Map of tool Lambdas (with Retry/Catch) → append the tool results and increment the iteration → loop back to CallModel.

*D2 · Task 2.1 · ReAct*
</details>

<details>
<summary><b>Q:</b> List five stopping conditions for an agent loop.</summary>

1. Max iterations or handoffs.
2. Goal reached (`end_turn` or a validator passes).
3. Wall-clock timeout (`TimeoutSeconds`).
4. Token or cost budget exceeded.
5. Repeated identical tool calls, a guardrail intervention, or low confidence → escalate to a human.

*D2 · Task 2.1 · Stopping conditions*
</details>

<details>
<summary><b>Q:</b> Describe the three states of a circuit breaker.</summary>

- **Closed**: normal; failures are counted.
- **Open**: after N failures, calls are skipped and the fallback runs immediately.
- **Half-open**: after a cool-down, one trial call is allowed. Success → closed; failure → open.

On AWS: circuit state in DynamoDB, with Step Functions or a Lambda wrapper choosing the fallback.

*D2 · Task 2.1 · Circuit breaker*
</details>

<details>
<summary><b>Q:</b> An agent can create IAM roles for deployments. How do you guarantee it can never escalate privileges?</summary>

Require a **permission boundary** on any role it creates, and back it with SCPs. Scope the agent role itself to least privilege with specific resource ARNs.

*D2 · Task 2.1 · IAM boundaries*
</details>

<details>
<summary><b>Q:</b> Name four ways to aggregate outputs from a model ensemble.</summary>

1. Majority or weighted voting.
2. Self-consistency (N samples, then vote).
3. LLM-as-judge picking the best answer.
4. Mixture-of-agents fusion, where an aggregator synthesises the answers.

Also: an agreement check that sends disagreements to a human.

*D2 · Task 2.1 · Ensembles*
</details>

<details>
<summary><b>Q:</b> What's the difference between Bedrock Intelligent Prompt Routing and an AppConfig-driven model selection?</summary>

**Intelligent Prompt Routing** is managed and decides per request, choosing between models in the same family based on predicted quality. **AppConfig** is a static, config-driven mapping from task to model, which you can switch without redeploying (with validation and gradual rollout).

*D2 · Task 2.1 · Model selection*
</details>

<details>
<summary><b>Q:</b> Why can't a human approval step use an Express workflow?</summary>

Express workflows don't support the `.waitForTaskToken` callback (or `.sync`) and run for at most 5 minutes. Use a Standard workflow instead, which can wait up to 1 year.

*D2 · Task 2.1 · Human approval*
</details>

<details>
<summary><b>Q:</b> How does a reviewer's click resume a waiting Step Functions execution?</summary>

The Task state passes `$$.Task.Token` to a Lambda, which stores it and notifies the reviewer. The reviewer's Approve/Reject goes through API Gateway to a Lambda that calls `SendTaskSuccess` or `SendTaskFailure` with that token.

*D2 · Task 2.1 · waitForTaskToken*
</details>

<details>
<summary><b>Q:</b> What does `toolChoice` control in the Converse API?</summary>

- `auto`: the model decides whether to use a tool.
- `any`: the model must call some tool.
- `tool: {name}`: the model must call that tool. This is useful for forcing schema-conformant JSON.

*D2 · Task 2.1 · Function definitions*
</details>

<details>
<summary><b>Q:</b> How does Bedrock Agents return of control work end to end?</summary>

1. Set the action group executor to `customControl: RETURN_CONTROL`.
2. `InvokeAgent` returns `returnControl.invocationInputs` plus an `invocationId`.
3. Your app executes the action.
4. Call `InvokeAgent` again with `sessionState.returnControlInvocationResults` and the same `invocationId` (`inputText` is ignored).

*D2 · Task 2.1 · Return of control*
</details>

<details>
<summary><b>Q:</b> A tool Lambda receives invalid arguments from the model. What should it return?</summary>

A `toolResult` with `status: "error"` and a structured, model-readable message saying what to fix (and whether it's retryable), so the model can self-correct. Don't throw an unhandled exception.

*D2 · Task 2.1 · Tool error handling*
</details>

<details>
<summary><b>Q:</b> Why add idempotency to agent tools that have side effects?</summary>

Model loops, Step Functions retries and network retries can call the same tool twice. An idempotency key (e.g. Powertools + DynamoDB) stops duplicate refunds or orders.

*D2 · Task 2.1 · Tool reliability*
</details>

<details>
<summary><b>Q:</b> What are the three MCP server primitives?</summary>

- **Tools**: model-controlled actions.
- **Resources**: app-controlled read-only context, addressed by URI.
- **Prompts**: user-controlled reusable templates.

*D2 · Task 2.1 · MCP primitives*
</details>

<details>
<summary><b>Q:</b> Which MCP transports are current, and what replaced HTTP+SSE?</summary>

**stdio** (a local subprocess using stdin/stdout) and **Streamable HTTP**: a single endpoint taking POST/GET, with optional SSE streaming and the `Mcp-Session-Id` header. Streamable HTTP replaced the 2024-11-05 HTTP+SSE transport.

*D2 · Task 2.1 · MCP transports*
</details>

<details>
<summary><b>Q:</b> When should an MCP server run on Lambda vs ECS Fargate?</summary>

**Lambda**: stateless, lightweight tools with short calls and scale-to-zero. **ECS Fargate**: stateful sessions, heavy dependencies (e.g. a browser), long operations, connection pools or caches, behind an ALB.

*D2 · Task 2.1 · MCP hosting*
</details>

<details>
<summary><b>Q:</b> Fastest way to make 30 existing internal REST APIs available as MCP tools to many agents, with OAuth?</summary>

**AgentCore Gateway** with OpenAPI (or Lambda) targets. It exposes one MCP endpoint with inbound auth, outbound credentials and semantic tool search.

*D2 · Task 2.1 · AgentCore Gateway*
</details>

---
[← Back to Domain 2](README.md)
