[← Back to Domain 3](README.md)

# ⚖️ Task 3.4 · Implement responsible AI principles

> **Domain 3 · AI Safety, Security, and Governance (20%)**
> - Skill 3.4.1 · Develop transparent AI systems in FM outputs
> - Skill 3.4.2 · Apply fairness evaluations to ensure unbiased FM outputs
> - Skill 3.4.3 · Develop policy-compliant AI systems to ensure adherence to responsible AI practices

## Key points
- **Frame every question by responsible AI dimension** (fairness, explainability, privacy & security, safety, controllability, veracity & robustness, governance, transparency). "Users don't trust answers" → transparency/explainability · "treats groups differently" → fairness · "must follow company policy" → governance/controllability.
- **Transparency = show your working:** reasoning (extended thinking, agent `rationale`), evidence (KB **citations**), uncertainty (confidence proxies → **CloudWatch**), and disclosure that content is AI-generated → 3.4.1.
- 🎯 **PQ43** **Bedrock Agents trace (`enableTrace=True`)** is for developers and auditors: pick the trace type by question (why rejected → PreProcessing, why this tool → Orchestration, wrong sub-agent → RoutingClassifier, why blocked → Guardrail). Users get a **curated summary**, never the raw trace → 3.4.1.
- **No single confidence number exists:** build it from retrieval, grounding, judge and self-consistency scores; publish with `PutMetricData` or **EMF**, alarm, and route low confidence to a human → 3.4.1.
- **Fairness testing:** counterfactual prompts + **BOLD** / **RealToxicityPrompts** benchmarks + LLM-as-a-judge **`Builtin.Stereotyping` / `Harmfulness` / `Refusal`**, sliced per group with the dataset **`category`** key → 3.4.2.
- **Prove a bias fix with A/B tests:** Prompt Management versions/variants run side by side in Flows, scored offline and online, compared per segment in CloudWatch → 3.4.2.
- **Clarify** for tabular bias metrics and bias drift; **Bedrock evaluations** for new FM fairness work (Clarify is closed to new customers) → 3.4.2.
- **Policy-compliant systems:** map each policy clause to a **guardrail policy**, document limits in **model cards** (home: Task 3.3) and AI Service Cards, and run **Lambda compliance checks** before deployment and after each response → 3.4.3.
- **Human-in-the-loop** is the answer for **high-stakes, low-confidence** decisions (credit, hiring, medical): Step Functions `waitForTaskToken` or **A2I**.

## Notes

### Responsible AI dimensions — the frame for every Task 3.4 question

| Dimension | What it means for an FM app | Typical AWS mechanism |
| --- | --- | --- |
| **Fairness** | Outputs don't systematically disadvantage groups | Bedrock evaluation (BOLD, `Builtin.Stereotyping`), Clarify bias metrics, per-group CloudWatch metrics |
| **Explainability** | Humans can understand *why* an output was produced | Agent trace `rationale`, reasoning content, SHAP (Clarify) for classic ML |
| **Privacy & security** | Data used and exposed appropriately | Guardrails PII filters, Macie, KMS, VPC endpoints ([Task 3.2](02-task-3.2-data-security-privacy.md)) |
| **Safety** | Prevent harmful output & misuse | Guardrails content filters / denied topics ([Task 3.1](01-task-3.1-input-output-safety-controls.md)) |
| **Controllability** | Mechanisms to monitor & steer behaviour | System prompts, Prompt Management, guardrails, HITL, stop conditions |
| **Veracity & robustness** | Correct output, even with unexpected/adversarial input | RAG grounding, contextual grounding check, Automated Reasoning checks, robustness evaluation (perturbed prompts) |
| **Governance** | Responsible AI practices built into the supply chain | Model cards, Model Registry approval, AWS Audit Manager, CloudTrail ([Task 3.3](03-task-3.3-governance-compliance.md)) |
| **Transparency** | Stakeholders can make informed choices about the system | AI Service Cards, AI-generated content disclosure, citations, documented limitations |

- **Responsible AI lifecycle:** define the use case and stakeholders → assess risks (likelihood × severity per dimension) → choose mitigations → **evaluate before release** → **monitor after release** → feed back. The exam likes "**continuous**" more than "one-off".
- **AWS AI Service Cards** = AWS-published transparency documents for AWS AI services and Amazon models, e.g. Amazon Nova and Titan models, Rekognition, Textract, Transcribe. They cover intended use cases, limitations, design choices, and deployment/performance best practices. **Model cards** (SageMaker) are what *you* write for *your* models.
- Third-party model providers (Anthropic, Meta, etc.) publish their own model cards or system cards. Read them for **known limitations** before choosing a model (links to [Task 1.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md)).
- Standards you may see named: **ISO/IEC 42001** (AI management system; AWS is certified), the **NIST AI RMF**, and the **EU AI Act** (risk tiers; high-risk systems need transparency, human oversight and logging). → Full detail: [Task 3.3 · Skill 3.3.1](03-task-3.3-governance-compliance.md#skill-331--compliance-frameworks-for-fm-deployments).

---

### Skill 3.4.1 — Transparent AI systems

> *Develop transparent AI systems in FM outputs (for example, by using reasoning displays to provide user-facing explanations, CloudWatch to collect confidence metrics and quantify uncertainty, evidence presentation for source attribution, Amazon Bedrock agent tracing to provide reasoning traces).*

#### 1. Four building blocks of transparency

| Building block | Answers the user question | Implementation |
| --- | --- | --- |
| **Reasoning display** | "How did you get that?" | Extended thinking / `reasoningContent`, CoT summary, agent `rationale` |
| **Evidence presentation** | "Where did that come from?" | KB `citations`, `retrievedReferences`, S3 URIs → presigned links, page numbers in metadata |
| **Uncertainty quantification** | "How sure are you?" | Retrieval scores, grounding scores, judge scores → CloudWatch metrics + UI badges |
| **Disclosure** | "Am I talking to an AI?" | UI banner, watermarking (Titan/Nova image watermark, C2PA content credentials where supported, verify in docs), output labelling |

#### 2. Reasoning displays

- **Extended thinking / reasoning models** (e.g. Claude with extended thinking via Converse `additionalModelRequestFields`). The response contains `reasoningContent` blocks alongside `text`.
  ```python
  resp = brt.converse(
      modelId="us.anthropic.claude-sonnet-4-5-20250929-v1:0",
      messages=[{"role": "user", "content": [{"text": q}]}],
      inferenceConfig={"maxTokens": 4096},
      additionalModelRequestFields={"thinking": {"type": "enabled", "budget_tokens": 2048}},
  )
  for block in resp["output"]["message"]["content"]:
      if "reasoningContent" in block:
          reasoning = block["reasoningContent"]["reasoningText"]["text"]   # show as "How I reasoned" (collapsed)
      elif "text" in block:
          answer = block["text"]
  ```
  - Thinking tokens are billed as **output tokens**, which adds latency and cost. Use them where explanation adds value.
  - Some models return a **summarised** or **redacted** thinking block (`redactedContent`) instead of the full reasoning. Don't build UX that depends on the full raw reasoning (verify in docs for per-model behaviour).
- **Prompted CoT with a structured output:** ask for `{"answer": ..., "reasoning_summary": ..., "assumptions": [...], "sources": [...]}` and enforce it with a JSON Schema or tool use (see [Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md)). Display `reasoning_summary`, not raw CoT.
- **Best practice:** reasoning for **users** = short, plain-language and faithful to the evidence. Reasoning for **auditors/devs** = full trace, logged in CloudWatch/S3. A rationale that sounds right but **isn't faithful** to what the model actually did is a transparency risk. Pair the rationale with **evidence**.

#### 3. Evidence presentation and source attribution

- **`RetrieveAndGenerate`** response structure (KB):
  ```json
  {
    "output": {"text": "Refunds are processed within 14 days..."},
    "citations": [{
      "generatedResponsePart": {"textResponsePart": {"text": "Refunds are processed within 14 days", "span": {"start": 0, "end": 36}}},
      "retrievedReferences": [{
        "content": {"text": "...customers receive refunds within 14 days..."},
        "location": {"type": "S3", "s3Location": {"uri": "s3://kb-docs/policies/refunds-2026.pdf"}},
        "metadata": {"x-amz-bedrock-kb-source-uri": "s3://...", "page": 4, "owner": "finance"}
      }]
    }],
    "sessionId": "..."
  }
  ```
  - Use `span` to **highlight** which sentence is supported by which source → inline footnotes `[1]`.
  - Turn S3 URIs into **presigned URLs** (Lambda) so users can open the source, but only if they are **authorised** for that document (respect KB metadata filters or ACL filtering).
- **`Retrieve`** (you generate the answer yourself): ask the model to cite chunk IDs (`[doc-3]`) and map them back. Validate in post-processing that **every cited ID exists** in the retrieved set. Hallucinated citations are a classic failure mode.
- **Agents:** the `knowledgeBaseLookupOutput.retrievedReferences` in the orchestration trace observation, plus the `citations` in the `InvokeAgent` response chunk attribution, give the same evidence for agent answers.
- **Metadata you need for good attribution:** `title`, `source_url`, `page`, `last_updated`, `owner`. Add these at ingestion (sidecar `.metadata.json`; see [Task 1.4](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md)).
- **Grounding check as transparency:** the guardrail **contextual grounding check** scores **grounding** (supported by the source?) and **relevance** (answers the query?). Besides blocking or flagging, the scores can be shown as a confidence indicator. → Full detail: [Task 3.1 · Skill 3.1.3](01-task-3.1-input-output-safety-controls.md#skill-313--accuracy-verification-systems-to-reduce-hallucinations).

#### 4. Bedrock Agents tracing

- Enable per call: `InvokeAgent(..., enableTrace=True)`. Each streamed `chunk` comes with a `trace` event (`TracePart`):
  ```json
  {
    "agentId": "...", "agentAliasId": "...", "sessionId": "...", "agentVersion": "3",
    "collaboratorName": "billing-agent",
    "callerChain": [{"agentAliasArn": "arn:aws:bedrock:...:agent-alias/SUPERVISOR/..."}],
    "trace": { "orchestrationTrace": { "rationale": {"text": "The user wants their last invoice; call getInvoice."},
                                        "invocationInput": {"invocationType": "ACTION_GROUP", "actionGroupInvocationInput": {"actionGroupName": "billing", "function": "getInvoice", "parameters": [...] , "executionType": "LAMBDA"}},
                                        "observation": {"type": "ACTION_GROUP", "actionGroupInvocationOutput": {"text": "{...}"}} } }
  }
  ```

| Trace type | What it shows | Use for |
| --- | --- | --- |
| **PreProcessingTrace** | `parsedResponse.isValid` + `rationale` (is the input valid/in scope?) | Why a request was rejected as invalid |
| 🎯 **PQ43** **OrchestrationTrace** | `rationale` (reasoning) → `invocationInput` (action group / KB / collaborator / FINISH) → `observation` (results, `REPROMPT`, `ASK_USER`, `FINISH`) | Main **reasoning trace**, i.e. the ReAct steps |
| **PostProcessingTrace** | Final response shaping (`parsedResponse.text`) | Output formatting issues |
| **RoutingClassifierTrace** | Supervisor routing decision in multi-agent collaboration | "Why did it go to the wrong sub-agent?" |
| **CustomOrchestrationTrace** | Steps of a custom orchestration Lambda | Custom orchestrator debugging |
| **GuardrailTrace** | `action: GUARDRAIL_INTERVENED \| NONE` + input/output assessments (topic, content filter with confidence, words, PII, regex) | Explaining **why** content was blocked |
| **FailureTrace** | `failureReason` | Errors (Lambda failure, parsing failure) |

- Every trace except Failure includes `modelInvocationInput`: the prompt text, `type`, `inferenceConfiguration`, `promptCreationMode`/`parserMode` (`DEFAULT` vs `OVERRIDDEN`) and `overrideLambda`. This is how you see **what the model actually saw**, and whether advanced prompt templates are in use.
- `modelInvocationOutput.metadata.usage` gives **input/output tokens per step**, which feeds cost transparency.
- **`callerChain`** shows the path from the end user through the supervisor to the collaborator, which gives accountability in multi-agent systems.
- **Persist traces:** stream them to CloudWatch Logs or Firehose → S3 → Athena for audit. Use invocation logging for raw model I/O (see [Task 2.5](../domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md)).
- **AgentCore / Strands equivalent:** OpenTelemetry traces and spans → **CloudWatch GenAI observability** / X-Ray. Use these for Strands agents and agents hosted on AgentCore Runtime.
- ⚠️ Raw traces can contain **PII, tool outputs and internal prompts**. Don't stream them to end users. Redact before logging where policy requires it.

> [!IMPORTANT]
> **🎯 Practice question 43 — 8% of multi-agent handoffs fail with HTTP 200: find where validation breaks, validate consistently**
>
> - **Turn on Bedrock Agents tracing** (`enableTrace`) and inspect **`OrchestrationTrace`**: **`modelInvocationInput` / `modelInvocationOutput`** at each handoff (with `callerChain` and `collaboratorName`). This shows **exactly where** a parameter changed type or precision.
> - Add **parameter boundary checks in each agent's action-group API schema** (OpenAPI types, formats, `minimum`/`maximum`, decimal precision, required fields) → **consistent validation** across agents that use different FMs.
> - ❌ **Logs Insights + Lambda normalisers:** looks at logs after the fact and doesn't capture model invocation inputs and outputs.
> - ❌ **EventBridge + Step Functions validation layer:** a re-architecture, and it still doesn't pinpoint the failure.
> - ❌ **Correlation IDs + pre-validation Lambda:** tracks requests but doesn't diagnose the FM-to-FM parameter issues.

#### 5. Confidence metrics and uncertainty in CloudWatch

- **Signal sources:**

| Signal | How to get it | Notes |
| --- | --- | --- |
| Retrieval similarity score | `Retrieve` → `retrievalResults[].score` | Low top-score → "I don't have enough information" |
| Contextual grounding / relevance | Guardrail assessment (`contextualGroundingPolicy.filters[].score`) | Thresholds 0–0.99; below = blocked/flagged |
| Judge score | Second FM rates the answer (0–1 or 1–5) | Extra cost/latency, so sample a share of traffic |
| Self-consistency | Generate N answers at temperature > 0 and measure agreement | N× cost, so use for high-stakes calls |
| Token log-probs | Only some models/hosts expose them (e.g. self-hosted on SageMaker) | Not generally available via Bedrock Converse (verify in docs) |
| Model-stated confidence | Ask for a `confidence` field in the JSON | Poorly calibrated, so treat it as weak evidence only |

- **Publish custom metrics:**
  ```python
  cw.put_metric_data(
      Namespace="GenAI/Assistant",
      MetricData=[
          {"MetricName": "GroundingScore", "Value": grounding, "Unit": "None",
           "Dimensions": [{"Name": "UseCase", "Value": "refunds"}, {"Name": "ModelId", "Value": model_id}]},
          {"MetricName": "LowConfidenceResponse", "Value": 1 if grounding < 0.7 else 0, "Unit": "Count",
           "Dimensions": [{"Name": "UseCase", "Value": "refunds"}]},
      ],
  )
  ```
  - Or use **Embedded Metric Format (EMF)** in Lambda logs. That gives metrics without extra API calls, so you're **not throttled by PutMetricData** at high volume.
  - **Alarms:** `p10(GroundingScore) < 0.6` or `Sum(LowConfidenceResponse)/Sum(Requests) > 5%` → SNS → on-call, or trigger a review workflow.
  - **Dashboards:** trend confidence by model version, by prompt version and by user segment. A shift after a prompt change = regression (see [Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md)).
- **Uncertainty in the UX:** show a confidence badge, say "I'm not certain", offer the sources, or **abstain**. Abstaining beats a confident wrong answer, and the `Builtin.Refusal` metric helps check that the model refuses when it should and not when it shouldn't.
- **Routing on confidence:** below the threshold → escalate to a human (Step Functions `waitForTaskToken`, A2I, or a Connect agent queue) or to a stronger model (cascade; see [Task 2.4](../domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md)).

#### 6. Decision table

| Requirement | Pick |
| --- | --- |
| Users must see *which document* supports each sentence | KB `RetrieveAndGenerate` citations + `span` highlighting |
| Devs must see *why the agent called tool X* | `enableTrace=True` → orchestration `rationale` / `invocationInput` |
| Explain *why content was blocked* | GuardrailTrace / guardrail `trace` (`assessments`) |
| Quantify and alert on answer uncertainty | Grounding/retrieval/judge scores → `PutMetricData`/EMF → CloudWatch alarm |
| Users want a plain-language explanation | Structured `reasoning_summary`, or a summary of extended thinking |
| Audit full multi-agent path | Trace `callerChain` + `collaboratorName`, persisted to S3 |

---

### Skill 3.4.2 — Fairness evaluations

> *Apply fairness evaluations to ensure unbiased FM outputs (for example, by using pre-defined fairness metrics in CloudWatch, Amazon Bedrock Prompt Management and Amazon Bedrock Prompt Flows to perform systematic A/B testing, Amazon Bedrock with LLM-as-a-judge solutions to perform automated model evaluations).*

#### 1. Bias in FMs: where it comes from

- **Training data bias** (internet text carries stereotypes) → the FM over- or under-represents groups.
- **Prompt bias:** few-shot examples skewed to one group, or loaded wording ("as a male engineer…").
- **Retrieval bias:** the KB corpus is unbalanced (e.g. only one region's policies), so answers favour one group.
- **Feedback-loop bias:** thumbs-up data from a non-representative user base used in fine-tuning or RLHF.
- **Types of harm:** **allocational** (unequal decisions such as loans or hiring) vs **representational** (stereotypes, offensive depictions). FM chat apps mostly risk **representational** harm. FMs used for decisions risk **allocational** harm, which also needs HITL.

#### 2. Fairness evaluation approaches

| Approach | How | Good for |
| --- | --- | --- |
| **Counterfactual / paired prompts** | Same prompt, swap only the protected attribute (name, gender, ethnicity, age) → compare outputs | Detecting disparate treatment; cheap and targeted |
| **Benchmark datasets** | **BOLD** (bias in open-ended generation), **RealToxicityPrompts** (toxicity) | Baseline comparison across candidate models |
| **LLM-as-a-judge** | `Builtin.Stereotyping`, `Builtin.Harmfulness`, `Builtin.Refusal`, or custom fairness rubric | Scalable scoring of your own prompts |
| **Human evaluation** | Diverse reviewers rate outputs (thumbs, Likert, ranking) | Nuanced/cultural judgement, judge calibration |
| **Production monitoring** | Per-group metrics in CloudWatch (refusal rate, sentiment, length, escalations by segment) | Bias **drift** after launch |
| **Classic ML bias metrics** | SageMaker Clarify (DPL, DI, etc.) on FM-driven decisions turned into labels | Allocational decisions (approve/deny) |

#### 3. Amazon Bedrock model evaluation for fairness

Bedrock evaluations are explained in full in Domain 5 (job types, built-in datasets, judge metrics, RAG evaluation). → Full detail: [Task 5.1 · Skill 5.1.2](../domain-5-testing-validation-troubleshooting/01-task-5.1-evaluation-systems.md#skill-512--systematic-model-evaluation-to-find-optimal-configurations) and [Skill 5.1.5](../domain-5-testing-validation-troubleshooting/01-task-5.1-evaluation-systems.md#skill-515--multi-perspective-assessment-rag-eval-llm-as-a-judge-human-feedback). The fairness angle:

| Job type | Fairness use | Facts to remember here |
| --- | --- | --- |
| **Automatic (programmatic)** | Benchmark candidate models on toxicity (and robustness) | Built-in datasets each **down-sampled to 100 prompts**; custom JSONL **≤ 1,000 prompts** |
| **Human** | Nuanced, cultural judgement by diverse reviewers | Your own team or an **AWS-managed team**; output S3 bucket needs **CORS** |
| **LLM-as-a-judge** | Score your own prompts for stereotyping at scale | Generator & evaluator in the **same Region**; can **bring your own responses** from non-Bedrock models |

- **Fairness datasets:** **BOLD** (bias in open-ended generation) covers **5 fairness domains**: profession, gender, race, religious ideologies, political ideologies. **RealToxicityPrompts** (with BOLD) gives the toxicity score for general text generation.
- **Custom dataset JSONL** (automatic):
  ```json
  {"prompt": "Describe a typical nurse.", "referenceResponse": "...", "category": "gender-profession"}
  ```
  - `prompt` is required. `referenceResponse` is required for accuracy/robustness and for Q&A. **`category` is optional and gives per-category scores, which is your built-in slicing for fairness**, e.g. one category per demographic counterfactual set.

#### 4. LLM-as-a-judge for fairness

- **Responsible AI built-in metrics:** **`Builtin.Harmfulness`** (harmful content), **`Builtin.Stereotyping`** (stereotypes of any kind, **positive or negative**), **`Builtin.Refusal`** (declines / rejects the request). They sit alongside the quality metrics (Correctness, Completeness, Faithfulness, Helpfulness, Coherence, Relevance, FollowingInstructions, ProfessionalStyleAndTone), so one job can show a bias fix didn't hurt quality.
- **Custom metrics:** write your own judge prompt and rating scale, e.g. "Does the response treat the applicant's nationality as relevant? 0/1". Custom metrics use a **subset of evaluator models**.
- **Judge bias** (position, verbosity, self-preference) can itself skew fairness scores: use a judge from a **different model family**, calibrate against a **human-labelled** sample, and randomise order in pairwise comparisons.
- **IAM:** the evaluation service role needs access to **both the generator and evaluator models** + the S3 input/output buckets.
- Job creation sketch:
  ```python
  bedrock.create_evaluation_job(
      jobName="fairness-v3-vs-v4",
      roleArn=role_arn,
      applicationType="ModelEvaluation",
      evaluationConfig={"automated": {
          "datasetMetricConfigs": [{
              "taskType": "General",
              "dataset": {"name": "counterfactuals", "datasetLocation": {"s3Uri": "s3://eval/cf.jsonl"}},
              "metricNames": ["Builtin.Stereotyping", "Builtin.Harmfulness", "Builtin.Refusal", "Builtin.Helpfulness"],
          }],
          "evaluatorModelConfig": {"bedrockEvaluatorModels": [{"modelIdentifier": "amazon.nova-pro-v1:0"}]},
      }},
      inferenceConfig={"models": [{"bedrockModel": {"modelIdentifier": "anthropic.claude-haiku-4-5-20251001-v1:0"}}]},
      outputDataConfig={"s3Uri": "s3://eval/results/"},
  )
  ```
  (Shape is indicative, so verify in docs for exact field names.)
- **RAG evaluation** (retrieve-and-generate jobs) reports the same **responsible AI metrics** (harmfulness, stereotyping, refusal), so you can test a RAG app for bias too (→ Skill 5.1.5).

#### 5. Systematic A/B testing with Prompt Management and Flows

- **Why:** a fairness or bias fix (new system prompt wording, removing skewed few-shot examples) must be **proven** to help, and must not hurt quality.
- **Pattern:**
  1. Store prompt **A** and **B** as **versions** (or **variants** within a prompt; Prompt Management lets you compare variants side by side in the console).
  2. Build a **Flow** (or app-level splitter): input → **Prompt node A** and **Prompt node B** in parallel, or a **Condition node** on a hash of the user ID for an online split → collector → output. Log which variant served each request.
  3. Score offline with an **evaluation job** on a counterfactual dataset (per-`category` scores), **and** online with sampled LLM-as-a-judge + user feedback.
  4. Publish **per-variant, per-segment** metrics to CloudWatch (`Dimensions: PromptVersion, Segment`).
  5. Promote the winner by pointing the **alias/app config** at the new prompt version (AppConfig; see [Task 1.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md)). Keep the old version for **rollback**.
- **Fairness metric ideas for CloudWatch** ("pre-defined fairness metrics"):
  - `RefusalRate` by segment (over-refusing one group = unfair)
  - `StereotypingScore` (judge) by segment
  - `AvgResponseLength` / `SentimentScore` (Comprehend) by segment, since big gaps = differential treatment
  - `EscalationRate` and `ThumbsDownRate` by segment
  - **Disparity ratio** = metric(group d) / metric(group a). Alarm when it is outside **0.8–1.25**. This is the "four-fifths" rule-of-thumb from employment testing.
- ⚠️ **Segment data is sensitive:** collect demographics only with a lawful basis. Use **synthetic counterfactuals** offline instead of profiling real users where possible.

#### 6. SageMaker Clarify and FMEval

- **Pre-training (data) bias metrics:** **CI** class imbalance · **DPL** difference in proportions of labels · **KL** Kullback-Leibler · **JS** Jensen-Shannon · **LP** Lp-norm · **TVD** total variation distance · **KS** Kolmogorov-Smirnov · **CDD** conditional demographic disparity.
- **Post-training (model) bias metrics** (classification-style), e.g. **DPPL** (difference in positive proportions in predicted labels), **DI** (disparate impact), **DCAcc/DAR/DRR**, **recall/accuracy difference**, **TE** (treatment equality), **FT** (flip test, i.e. counterfactual) (verify in docs for the full list).
- **Explainability:** **SHAP** feature attributions for tabular/NLP models, plus **Model Monitor** integration for **bias drift** and **feature attribution drift** on endpoints (links to [Task 3.3](03-task-3.3-governance-compliance.md) Skill 3.3.4).
- **FMEval** (open-source `fmeval` library, also surfaced in SageMaker Studio): evaluates FMs for **accuracy, semantic robustness, factual knowledge, prompt stereotyping, toxicity**. Prompt stereotyping uses the **CrowS-Pairs** dataset (verify in docs). Works with SageMaker JumpStart, Bedrock or custom models.
- ⚠️ **Availability change:** AWS docs now state SageMaker Clarify is **no longer open to new customers** (existing customers continue, no new features). For new FM fairness work, AWS steers you to **Bedrock evaluations**. The exam guide may still mention Clarify, so know both.

| Need | Use |
| --- | --- |
| Compare Bedrock FMs on bias/toxicity with standard benchmarks | Bedrock automatic evaluation (BOLD, RealToxicityPrompts) |
| Score your own prompts for stereotyping at scale | Bedrock LLM-as-a-judge `Builtin.Stereotyping` |
| Nuanced cultural review | Bedrock human evaluation (AWS-managed or own team) |
| Bias in tabular training data / approve-deny decisions | SageMaker Clarify pre/post-training metrics |
| Bias **drift** on a live SageMaker endpoint | Clarify + Model Monitor bias drift |
| Bias drift on Bedrock app | Custom per-segment CloudWatch metrics + scheduled evaluation jobs |

---

### Skill 3.4.3 — Policy-compliant AI systems

> *Develop policy-compliant AI systems to ensure adherence to responsible AI practices (for example, by using Amazon Bedrock guardrails based on policy requirements, model cards to document FM limitations, Lambda functions to perform automated compliance checks).*

#### 1. From written policy to enforced control

| Policy clause (example) | Guardrail policy / control |
| --- | --- |
| "Never give investment advice" | **Denied topic** (name + definition + sample phrases), input & output |
| "No hate, insults, sexual, violent content; resist jailbreaks" | **Content filters** (HATE, INSULTS, SEXUAL, VIOLENCE, MISCONDUCT, **PROMPT_ATTACK**) with strength LOW/MEDIUM/HIGH; Standard tier adds code-domain coverage (verify in docs) |
| "Don't mention competitor names / internal codenames" | **Word filters** (custom words + managed profanity list) |
| "Never reveal customer SSNs; mask emails" | **Sensitive information filters**: PII types with **BLOCK** or **ANONYMIZE** (mask), plus custom **regex** |
| "Answers must be grounded in approved policy docs" | **Contextual grounding check** (grounding + relevance thresholds) |
| "Responses must comply with eligibility rules (e.g. HR leave policy)" | **Automated Reasoning checks**: formal logic policy built from the policy document → Valid / Invalid / Satisfiable findings |
| "Every app in the org must use the baseline guardrail" | **Guardrail enforcement at account/org level** (IAM condition `bedrock:GuardrailIdentifier`, org-level enforced guardrails via resource policy; see [Task 2.3](../domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md)) |

- **Policy → guardrail traceability:** tag each guardrail with `PolicyId`/`PolicyVersion`, keep the guardrail definition in **IaC** (CloudFormation `AWS::Bedrock::Guardrail`), and publish immutable **guardrail versions**. Applications pin a version, and policy changes produce a new version that goes through review and approval.
- **Custom blocked messaging:** configure `blockedInputMessaging` / `blockedOutputsMessaging` to explain the refusal in policy-aligned language. This is transparency again.
- **Detect mode / testing:** test guardrails in the console or with **`ApplyGuardrail`** against a policy test set before release, and record which assessments triggered.
- **Beyond guardrails:** the system prompt states the policy (role, scope, refusal behaviour) via Prompt Management. Human review handles policy exceptions.

#### 2. Model cards: documenting FM limitations

Model card mechanics (schema, versioning, status lifecycle, PDF export, Model Registry linkage, programmatic creation) → Full detail: [Task 3.3 · Skill 3.3.1](03-task-3.3-governance-compliance.md#skill-331--compliance-frameworks-for-fm-deployments). For responsible AI, focus on the **content**:
- **Intended uses** and **out-of-scope uses** ("not for medical diagnosis"), and **known limitations** (languages, knowledge cutoff, hallucination rates on eval set, weaker performance on dialects).
- **Evaluation results**, including **fairness** (per-category scores from Bedrock eval / Clarify), toxicity and robustness.
- **Risk rating** with **mitigations** (guardrail IDs, HITL steps), **ethical considerations**, training data provenance, and **owners/approvers**.
- For Bedrock FMs you don't train, write the card for **your application/system** (FM + prompt + KB + guardrail versions), and link the provider's model card and the **AWS AI Service Card**.
- Gate deployment on an **Approved** card in CI/CD (see [Task 2.3](../domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md) Skill 2.3.5).

#### 3. Automated compliance checks with Lambda

- **Two kinds of compliance check:**
  1. **Pre-deployment / config checks:** a Lambda in CodePipeline or an **AWS Config custom rule** verifies:
     - every Bedrock call path has a guardrail attached
     - the guardrail version ≥ the policy-mandated version
     - model invocation logging is enabled
     - an approved model card exists
     - only allow-listed model IDs are used
     - the KB data sources carry classification tags
  2. **Runtime / post-response checks:** a Lambda after generation checks:
     - mandatory **disclaimers** are present ("This is not financial advice")
     - no banned phrases or PII (regex, or Comprehend `DetectPiiEntities`)
     - output matches the required JSON Schema
     - citations are present for factual claims
     - **tone/sentiment** passes (Comprehend)
     - an optional judge-model compliance score
- **Pattern:**
  ```
  API GW → Lambda (pre-check: input policy, Comprehend PII/toxicity)
         → Bedrock Converse (guardrailConfig: id+version)
         → Lambda (post-check: disclaimer, schema, citations, banned terms)
            ├─ pass → return to user
            └─ fail → redact/regenerate (max N) → else safe fallback message
                     + EventBridge "ComplianceViolation" → SNS / Security Hub / ticket
                     + CloudWatch metric PolicyViolationCount{Policy=...}
  ```
  ```python
  def lambda_handler(event, _):
      text = event["output"]
      violations = []
      if "not financial advice" not in text.lower() and event["useCase"] == "investing":
          violations.append("MISSING_DISCLAIMER")
      if comprehend.detect_pii_entities(Text=text[:5000], LanguageCode="en")["Entities"]:
          violations.append("PII_IN_OUTPUT")
      if violations:
          events.put_events(Entries=[{"Source": "genai.compliance", "DetailType": "ComplianceViolation",
                                      "Detail": json.dumps({"requestId": event["requestId"], "violations": violations})}])
      return {"compliant": not violations, "violations": violations}
  ```
- **Orchestrate multi-step checks** with Step Functions: parallel checks → Choice → HITL approval for borderline cases (`waitForTaskToken`). Use **Step Functions Distributed Map** for batch audits of logged responses.
- **Evidence for auditors:** check results → S3 (Object Lock if required) → **AWS Audit Manager** custom framework mapping controls to evidence. CloudTrail records guardrail and config changes ([Task 3.3](03-task-3.3-governance-compliance.md)).

#### 4. Putting it together: a responsible AI control stack

| Layer | Control | Dimension |
| --- | --- | --- |
| Design | Use-case risk assessment, model card (intended use, limits) | Governance, transparency |
| Model choice | Bedrock evaluation incl. BOLD / Stereotyping across candidates | Fairness, veracity |
| Prompt | Prompt Management versioned system prompt with policy rules; A/B tested | Controllability |
| Input | Guardrail (prompt attack, denied topics, PII), Comprehend pre-filter | Safety, privacy |
| Generation | RAG + citations, contextual grounding, Automated Reasoning | Veracity, transparency |
| Output | Guardrail output filters, Lambda compliance check, disclaimers | Safety, governance |
| Human | HITL on low confidence or high-stakes | Controllability, fairness |
| Monitor | CloudWatch confidence & per-segment fairness metrics, alarms, periodic eval jobs | All (continuous) |

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Users must see which source document supports each sentence of the answer | KB **RetrieveAndGenerate citations** (`generatedResponsePart.span` + `retrievedReferences.location`) |
| Developers need to see why the agent chose a particular action group | `InvokeAgent` with **`enableTrace=True`** → **OrchestrationTrace `rationale`** |
| Need to know why an agent rejected input as out of scope | **PreProcessingTrace** (`isValid`, `rationale`) |
| Wrong collaborator chosen in multi-agent collaboration | **RoutingClassifierTrace** + `callerChain` |
| Explain to user why the response was blocked | **GuardrailTrace / guardrail assessments** + custom blocked message |
| Quantify answer uncertainty and alert on drops | Grounding / retrieval / judge scores → **CloudWatch custom metrics (PutMetricData/EMF) + alarm** |
| High-volume Lambda publishing metrics, avoid API throttling | **Embedded Metric Format (EMF)** |
| Show a plain-language explanation of model reasoning | **Extended thinking / reasoning summary** or structured `reasoning_summary` field |
| Compare candidate FMs for bias in open-ended generation | Bedrock automatic evaluation with **BOLD** |
| Evaluate toxicity tendency of a model | Bedrock automatic evaluation **toxicity** with **RealToxicityPrompts** |
| Score your own prompts for stereotypes at scale, no humans | **LLM-as-a-judge `Builtin.Stereotyping`** (+ Harmfulness, Refusal) |
| Fairness rubric specific to your business | LLM-as-a-judge **custom metric** |
| Need nuanced cultural judgement of outputs | Bedrock **human evaluation** (own team or AWS-managed) |
| Per-group scores from one evaluation job | Use the dataset **`category`** key |
| Evaluate a non-Bedrock model's outputs with a Bedrock judge | LLM-as-a-judge with **bring your own inference responses** |
| LLM judge seems to favour its own outputs | Use a **different model family** as judge + calibrate with human labels |
| Prove a revised system prompt reduces bias without hurting quality | **Prompt Management versions/variants + Flows** A/B test + evaluation job + per-variant CloudWatch metrics |
| Model refuses requests from one dialect/group more often | Monitor **RefusalRate by segment**; `Builtin.Refusal` eval; fix prompt / examples |
| Bias metrics on tabular training data before training | **SageMaker Clarify pre-training metrics** (CI, DPL…) |
| Bias drift on a live SageMaker endpoint | **Clarify + Model Monitor bias drift** |
| AWS-published documentation of an AWS AI service's intended use and limits | **AWS AI Service Cards** |
| Document your FM app's limitations, intended use and eval results for auditors | **SageMaker Model Card** |
| Policy says "never discuss topic X" | Guardrail **denied topic** |
| Policy requires answers consistent with formal rules (eligibility, HR) | Guardrail **Automated Reasoning checks** |
| Policy requires every response to include a disclaimer | **Lambda post-processing compliance check** (+ prompt instruction) |
| Ensure all apps use the mandated guardrail | IAM **`bedrock:GuardrailIdentifier` condition** / org-level enforced guardrail |
| Compliance violations must alert security and create audit evidence | Lambda check → **EventBridge → SNS/Security Hub**; evidence to S3 + **Audit Manager** |
| High-stakes decision with low confidence | **Human-in-the-loop** (Step Functions `waitForTaskToken` / **A2I**) |

## Flashcards

<details>
<summary><b>Q:</b> Name the eight AWS responsible AI dimensions.</summary>

Fairness, explainability, privacy & security, safety, controllability, veracity & robustness, governance, transparency.

*D3 · Task 3.4 · RAI dimensions*
</details>

<details>
<summary><b>Q:</b> How do you get a reasoning trace from a Bedrock agent?</summary>

Call `InvokeAgent` with `enableTrace=True`. Each streamed chunk carries a `trace` (TracePart). The OrchestrationTrace contains `rationale`, `invocationInput` and `observation`.

*D3 · Task 3.4 · Agent trace*
</details>

<details>
<summary><b>Q:</b> List the Bedrock Agents trace types.</summary>

PreProcessing, Orchestration, PostProcessing, RoutingClassifier, CustomOrchestration, Guardrail, Failure.

*D3 · Task 3.4 · Agent trace*
</details>

<details>
<summary><b>Q:</b> Which trace type tells you why the agent judged an input invalid?</summary>

PreProcessingTrace. Its `parsedResponse` has `isValid` and `rationale`.

*D3 · Task 3.4 · Agent trace*
</details>

<details>
<summary><b>Q:</b> Which trace field shows the path of agents in a multi-agent request?</summary>

`callerChain`, the list of agent alias ARNs between the end user and the agent that published the trace. `collaboratorName` identifies the collaborator.

*D3 · Task 3.4 · Multi-agent tracing*
</details>

<details>
<summary><b>Q:</b> Why show users a curated reasoning summary instead of the raw agent trace?</summary>

Raw traces can include internal prompts, tool outputs and PII, and they are noisy. Users need short, faithful explanations plus evidence. Full traces go to secured logs for devs and auditors.

*D3 · Task 3.4 · Transparency*
</details>

<details>
<summary><b>Q:</b> Which RetrieveAndGenerate fields enable sentence-level source attribution?</summary>

`citations[].generatedResponsePart.textResponsePart.span` (which part of the answer) and `citations[].retrievedReferences[]` (content + `location.s3Location.uri` + metadata).

*D3 · Task 3.4 · Evidence presentation*
</details>

<details>
<summary><b>Q:</b> Bedrock doesn't give a single "confidence" number. How do you quantify uncertainty?</summary>

Combine proxies: retrieval scores, contextual grounding/relevance scores, LLM-as-a-judge scores and self-consistency. Publish them to CloudWatch (PutMetricData/EMF), alarm on them, and escalate low-confidence answers.

*D3 · Task 3.4 · Uncertainty*
</details>

<details>
<summary><b>Q:</b> Why prefer EMF over PutMetricData in a high-traffic Lambda?</summary>

EMF writes metrics as structured log lines that CloudWatch extracts asynchronously. There are no extra API calls, so no PutMetricData throttling and lower latency.

*D3 · Task 3.4 · CloudWatch metrics*
</details>

<details>
<summary><b>Q:</b> What does the BOLD dataset evaluate, and across which domains?</summary>

Bias in open-ended language generation, across profession, gender, race, religious ideologies and political ideologies.

*D3 · Task 3.4 · Fairness datasets*
</details>

<details>
<summary><b>Q:</b> What are the three metrics in Bedrock automatic (programmatic) evaluation?</summary>

Accuracy, robustness and toxicity. Toxicity uses RealToxicityPrompts/BOLD for general text generation.

*D3 · Task 3.4 · Automatic evaluation*
</details>

<details>
<summary><b>Q:</b> How many prompts do Bedrock's built-in datasets use, and what is the custom dataset limit for automatic evaluation?</summary>

Built-in datasets are down-sampled to 100 prompts each. Custom JSONL datasets can have up to 1,000 prompts per automatic job.

*D3 · Task 3.4 · Evaluation limits*
</details>

<details>
<summary><b>Q:</b> What are the three responsible-AI built-in metrics for LLM-as-a-judge?</summary>

`Builtin.Harmfulness`, `Builtin.Stereotyping` (positive or negative stereotypes) and `Builtin.Refusal`.

*D3 · Task 3.4 · LLM-as-a-judge*
</details>

<details>
<summary><b>Q:</b> How do you get per-demographic results from a single Bedrock evaluation job?</summary>

Put the optional `category` key on each JSONL record (e.g. one category per counterfactual group). Scores are then reported per category.

*D3 · Task 3.4 · Fairness slicing*
</details>

<details>
<summary><b>Q:</b> Can a Bedrock LLM-as-a-judge job evaluate a model hosted outside Bedrock?</summary>

Yes. Put your own inference responses in the prompt dataset. Bedrock skips inference and the judge scores the supplied responses.

*D3 · Task 3.4 · LLM-as-a-judge*
</details>

<details>
<summary><b>Q:</b> Name two judge-model biases and their mitigations.</summary>

Self-preference: use a judge from a different model family. Position or verbosity bias: randomise order and calibrate against human-labelled samples.

*D3 · Task 3.4 · LLM-as-a-judge*
</details>

<details>
<summary><b>Q:</b> Which evaluation type needs CORS on the S3 output bucket?</summary>

Human evaluation jobs. Automatic and LLM-as-a-judge jobs don't.

*D3 · Task 3.4 · Human evaluation*
</details>

<details>
<summary><b>Q:</b> What is counterfactual fairness testing for an FM?</summary>

Send pairs of prompts that differ only in a protected attribute (name, gender, age, ethnicity) and compare the outputs, e.g. sentiment, refusal, length or judge scores. Systematic differences indicate bias.

*D3 · Task 3.4 · Fairness testing*
</details>

<details>
<summary><b>Q:</b> How do Prompt Management and Flows support fairness A/B testing?</summary>

Store prompt variants/versions in Prompt Management and run them side by side or with a traffic split in a Flow. Score them with evaluation jobs or a judge, and compare per-variant, per-segment metrics in CloudWatch. Promote the winner by version or alias, and keep rollback.

*D3 · Task 3.4 · A/B testing*
</details>

<details>
<summary><b>Q:</b> Name four SageMaker Clarify pre-training bias metrics.</summary>

Any four of: Class Imbalance (CI), Difference in Proportions of Labels (DPL), KL divergence, Jensen-Shannon (JS), Lp-norm, Total Variation Distance (TVD), Kolmogorov-Smirnov (KS), Conditional Demographic Disparity (CDD).

*D3 · Task 3.4 · Clarify*
</details>

<details>
<summary><b>Q:</b> What availability caveat applies to SageMaker Clarify?</summary>

AWS now says Clarify is no longer open to new customers. Existing customers continue, but no new features are planned. Use Bedrock evaluations for new FM fairness work.

*D3 · Task 3.4 · Clarify*
</details>

<details>
<summary><b>Q:</b> What's the difference between an AWS AI Service Card and a SageMaker Model Card?</summary>

AI Service Cards are published by AWS for its own AI services and models (intended use, limitations, best practices). Model Cards are written by you for your own models or applications (intended use, limitations, evaluation results, risk rating).

*D3 · Task 3.4 · Documentation*
</details>

<details>
<summary><b>Q:</b> A policy requires answers about leave eligibility to follow formal HR rules exactly. Which guardrail feature fits?</summary>

Automated Reasoning checks. They validate responses against a formal logic policy built from the HR document.

*D3 · Task 3.4 · Policy-compliant guardrails*
</details>

<details>
<summary><b>Q:</b> What does a Lambda post-response compliance check typically verify?</summary>

Required disclaimers, no PII or banned terms, schema conformance, citations present and acceptable tone. Violations are remediated (redact/regenerate/fallback) and emitted to EventBridge with a CloudWatch metric.

*D3 · Task 3.4 · Compliance checks*
</details>

---
[← Back to Domain 3](README.md)
