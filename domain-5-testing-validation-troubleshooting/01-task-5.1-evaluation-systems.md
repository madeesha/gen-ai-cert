[← Back to Domain 5](README.md)

# 🧮 Task 5.1 · Implement evaluation systems for GenAI

> **Domain 5 · Testing, Validation, and Troubleshooting (11%)**
>
> - Skill 5.1.1: Assessment frameworks beyond traditional ML (relevance, factual accuracy, consistency, fluency)
> - Skill 5.1.2: Systematic model evaluation for optimal configurations (Bedrock Model Evaluations, A/B, canary, multi-model, cost-performance)
> - Skill 5.1.3: User-centred evaluation (feedback interfaces, ratings, annotation workflows)
> - Skill 5.1.4: Systematic QA processes (continuous evaluation, regression testing, automated quality gates)
> - Skill 5.1.5: Multi-perspective assessment (RAG evaluation, LLM-as-a-judge, human feedback)
> - Skill 5.1.6: Retrieval quality testing (relevance scoring, context matching, retrieval latency)
> - Skill 5.1.7: Agent performance frameworks (task completion, tool usage, Bedrock Agent evaluations, reasoning quality)
> - Skill 5.1.8: Stakeholder reporting (visualisation, automated reports, model comparisons)
> - Skill 5.1.9: Deployment validation (synthetic user workflows, hallucination rate, semantic drift, consistency checks)

## Key points
- **Mental model:** GenAI output is open-ended and non-deterministic, so you score **quality dimensions** (relevance, correctness, faithfulness, consistency, fluency, safety) with a **layered stack**: deterministic checks → reference metrics → LLM-as-a-judge → humans → business KPIs. → [5.1.1](#skill-511--assessment-frameworks-beyond-traditional-ml)
- **Metric by task:** translation → BLEU · summarisation → ROUGE / BERTScore · extractive Q&A → F1 · fluency → perplexity · open-ended, no single reference → **LLM-as-a-judge** · brand voice / domain nuance → **humans**. → [5.1.1](#skill-511--assessment-frameworks-beyond-traditional-ml)
- **Bedrock Evaluations: choose the job type by who scores.** Built-in algorithms on standard datasets (**programmatic**: accuracy, robustness, toxicity), your own **work team** (human), an **evaluator model** (LLM-as-a-judge), or **RAG evaluation** (retrieve-only vs retrieve-and-generate). Model outside Bedrock or external RAG → **BYOI** (Bedrock only judges). → [5.1.2](#skill-512--systematic-model-evaluation-to-find-optimal-configurations), [5.1.5](#skill-515--multi-perspective-assessment-rag-eval-llm-as-a-judge-human-feedback)
- **Optimal configuration = cheapest model + prompt + parameters that clears every quality threshold** (Pareto), not the top scorer. → [5.1.2](#skill-512--systematic-model-evaluation-to-find-optimal-configurations)
- **Online testing:** A/B for business impact · canary (small %, **auto-rollback** on alarms) for safe release · shadow (mirrored, **not served**) for zero user exposure. → [5.1.2](#skill-512--systematic-model-evaluation-to-find-optimal-configurations)
- **User feedback lives or dies on the join key** (`requestId` shared with the invocation log). Batch expert labelling → **Ground Truth**; production review loops on low confidence → **A2I**. → [5.1.3](#skill-513--user-centred-evaluation-mechanisms)
- **QA = versioned golden dataset + regression tests on every change + CodeBuild quality gate that fails the stage + continuous evaluation on sampled traffic.** → [5.1.4](#skill-514--systematic-quality-assurance-processes--pq37--pq66)
- **RAG: test retrieval separately from generation.** Low context relevance → fix retrieval; high relevance but low faithfulness → fix generation; high faithfulness but low correctness → fix the KB content. → [5.1.5](#skill-515--multi-perspective-assessment-rag-eval-llm-as-a-judge-human-feedback), [5.1.6](#skill-516--retrieval-quality-testing)
- **Agents are scored at three levels** (session goal success, trace quality, tool selection/parameters) with **AgentCore Evaluations**; classic Bedrock Agents → `enableTrace` + scenario suites. → [5.1.7](#skill-517--agent-performance-frameworks)
- **Reporting and release:** QuickSight for stakeholders, CloudWatch dashboards for ops ([5.1.8](#skill-518--stakeholder-reporting)). Deployment validation = judge evals vs baseline (hallucination rate) + Synthetics canaries + embedding drift/consistency + alarm-driven rollback ([5.1.9](#skill-519--deployment-validation-systems--pq21)).

## Notes

### Skill 5.1.1 — Assessment frameworks beyond traditional ML

> Develop comprehensive assessment frameworks to evaluate the quality and effectiveness of FM outputs beyond traditional ML evaluation approaches (for example, by using metrics for relevance, factual accuracy, consistency, and fluency).

#### Why traditional ML metrics aren't enough

| Traditional ML | GenAI / FMs |
| --- | --- |
| Closed label set (class, number) | Open-ended text, images, code, tool calls |
| One correct answer per input | Many acceptable answers; wording varies |
| Deterministic for a fixed model | Stochastic (temperature, top-p); same prompt → different outputs |
| Accuracy, precision, recall, F1, RMSE, AUC | Quality *dimensions* scored by references, judges, humans |
| Test set fixed at training time | Behaviour also depends on **prompt**, **context (RAG)**, **tools**, **model version** |
| Failure = wrong label | Failure = hallucination, unfaithful to context, off-topic, unsafe, wrong format, refusal, verbosity |

#### Core quality dimensions (know the definitions)

| Dimension | Question it answers | Typical measurement |
| --- | --- | --- |
| **Relevance** | Does the answer address the question asked? | LLM judge (`Builtin.Relevance`), embedding similarity of Q↔A |
| **Factual accuracy / correctness** | Is it true (vs ground truth or world knowledge)? | Reference compare (F1, exact match), LLM judge `Correctness`, RWK score |
| **Faithfulness / groundedness** | Is every claim supported by the provided context? | LLM judge `Faithfulness`, Guardrails contextual grounding score |
| **Completeness** | Did it answer every part of the question? | LLM judge `Completeness` (with or without reference) |
| **Consistency** | Same/semantically equivalent input → same meaning out? | Run N times or paraphrase; measure semantic similarity variance |
| **Coherence** | Logical, no contradictions/gaps? | LLM judge `Coherence` / `LogicalCoherence` |
| **Fluency** | Grammatical, natural, readable? | Perplexity (from a reference LM), LLM judge, human rating |
| **Instruction following / format** | Respected constraints (length, JSON schema, tone)? | Deterministic checks (JSON schema validate, regex, length) + `FollowingInstructions` |
| **Safety** | Harmful, toxic, stereotyping, PII leak? | Toxicity classifier, `Harmfulness`, `Stereotyping`, Guardrails |
| **Refusal appropriateness** | Refuses when it should, answers when it should? | `Refusal` metric + labelled should-refuse set |
| **Efficiency** | Tokens, latency, cost per useful answer | CloudWatch `InputTokenCount` / `OutputTokenCount` / `InvocationLatency` |

#### Reference-based (n-gram / embedding) metrics: when each fits

| Metric | What it computes | Good for | Weakness |
| --- | --- | --- | --- |
| **BLEU** | n-gram **precision** vs reference(s), brevity penalty | Machine **translation** | Penalises valid paraphrases; ignores meaning |
| **ROUGE-N / ROUGE-L** | n-gram / longest-common-subsequence **recall** vs reference | **Summarisation** | Same lexical bias; long outputs can game recall |
| **METEOR** | Unigram match with stems + synonyms, recall-weighted | Translation, short text | Language-resource dependent |
| **BERTScore** | Cosine similarity of contextual token embeddings (P/R/F1) | Summaries, paraphrase, Q&A where wording varies | Needs reference; model-dependent |
| **Exact match / F1 (token overlap)** | Token overlap with gold answer | Extractive / short-answer **Q&A** | Brittle for long-form |
| **Perplexity** | How "surprised" an LM is by the text (lower = more fluent/likely) | **Fluency**, LM quality during training | Not correctness; not comparable across tokenisers |
| **Word error rate (WER)** | Edit distance at word level | Speech-to-text; Bedrock uses it for **robustness** deltas | Lexical only |
| **Classification accuracy** | Correct label rate | Classification tasks framed as generation | Needs output parsing |
| **Semantic similarity** | Cosine of sentence embeddings (e.g. Titan Embeddings) | Consistency, drift, paraphrase match | Similar ≠ correct |

> Exam rule of thumb: **translation → BLEU**, **summarisation → ROUGE** (or BERTScore), **extractive Q&A → F1 / exact match**, **fluency → perplexity**, **open-ended quality with no single reference → LLM-as-a-judge**, **subjective/brand/tone → human evaluation**.

#### Reference-free evaluation
- **LLM-as-a-judge**: a strong model scores the output against a rubric (with or without a reference). It scales and explains its scores, but it has known biases. Biases and mitigations are in [5.1.5](#llm-as-a-judge-best-practices).
- **Rules/deterministic checks**: JSON Schema validation, regex for required disclaimers, max length, banned terms, citation present. These are cheap and exact, so run them first.
- **Human evaluation**: the gold standard for nuance, brand voice, domain correctness and safety edge cases. It is slow and expensive, so use it to calibrate the judges and for high-risk releases.

#### A layered evaluation framework (design pattern)
```
Layer 0  Deterministic checks      schema/format, length, PII regex, citation present     (every response, ~free)
Layer 1  Reference metrics         F1 / ROUGE / BERTScore on golden set                   (every build)
Layer 2  LLM-as-a-judge            correctness, faithfulness, helpfulness, harmfulness     (every build + sampled prod)
Layer 3  Human review              SMEs on stratified sample + all low-confidence cases   (release / weekly)
Layer 4  Business KPIs             deflection, CSAT, conversion, escalation rate          (continuous, A/B)
```
- Define **per-dimension thresholds** (for example faithfulness ≥ 0.9, harmfulness = 0, p95 latency < 3 s) and **weight** dimensions by use case. For example, faithfulness matters most for a regulated-advice bot, while creativity and fluency matter most for marketing copy.
- **Segment** results using the `category` field in the dataset (intent, product line, language). An average can hide a failing slice.

---

### Skill 5.1.2 — Systematic model evaluation to find optimal configurations

> Create systematic model evaluation systems to identify optimal configurations (for example, by using Amazon Bedrock Model Evaluations, A/B testing and canary testing of FMs, multi-model evaluation, cost-performance analysis to measure token efficiency, latency-to-quality ratios, and business outcomes).

#### Amazon Bedrock Evaluations: job types

| Job type | Who scores | Needs | Output | Use when |
| --- | --- | --- | --- | --- |
| **Automatic (programmatic)** | Built-in algorithms | Built-in dataset **or** custom JSONL prompt dataset | Accuracy, robustness, toxicity scores | Quick objective baseline, model shortlisting |
| **Human-based** | Your **work team** (employees/SMEs) via a SageMaker Ground Truth-style UI | Custom dataset, work team, S3 with **CORS** | Ratings per metric you define | Subjective quality, brand voice, domain expertise |
| **LLM-as-a-judge (model as judge)** | **Evaluator model** (judge) scores a **generator model** | Prompt dataset (≤ 1,000 prompts), judge model access | Scores **+ natural-language explanations** per prompt | Scalable human-like quality scoring |
| **RAG evaluation** | Evaluator model | Queries (+ ground truth), KB **or** BYOI responses | Retrieval and/or generation metrics | Comparing KBs, chunking, embeddings, generator models |

- Supported **generator models**: Bedrock foundation models, **Marketplace** models, **customised** (fine-tuned/distilled) models, **imported** models (Custom Model Import), **prompt routers**, and **Provisioned Throughput** models. Non-Bedrock models are covered through **BYOI**.
- Supported **judge models** for built-in metrics include Amazon **Nova** (Pro, Premier, Micro, 2 Lite), Anthropic **Claude** (3.x, 4.x Sonnet/Haiku/Opus), **Llama 3.1 70B**, **Mistral Large** and some OpenAI GPT models. **Cross-Region inference profiles** are supported for judges. The list changes often, so check the Bedrock docs for the current one.
- In the console, the **metrics summary card** shows a histogram of scores plus explanations for the **first 5 prompts**. The **full report** goes to your **S3** output location.
- Human jobs need **CORS** on the S3 bucket. LLM-as-a-judge jobs **don't need CORS**.
- Evaluation job create, stop and delete actions are logged as **CloudTrail management events**.

#### Automatic (programmatic) tasks: built-in datasets and algorithms

| Task type | Accuracy | Robustness | Toxicity |
| --- | --- | --- | --- |
| General text generation | **RWK** (real-world knowledge) score, **TREX** | **Word error rate** on perturbed prompts, BOLD / TREX / WikiText2 | **RealToxicityPrompts**, BOLD |
| Text summarisation | **BERTScore**, **Gigaword** | BERTScore + **deltaBERTScore**, Gigaword | Gigaword |
| Question & answer | **NLP-F1**, **BoolQ / NaturalQuestions / TriviaQA** | F1 + **deltaF1** | BoolQ / NQ / TriviaQA |
| Text classification | **Accuracy**, **Women's E-commerce Clothing Reviews** | `classification_accuracy_score` + delta | — |

- **Robustness** = how much the score drops when prompts are slightly perturbed (typos, casing, whitespace). This is the "delta" metrics.
- One **task type per job**.

#### LLM-as-a-judge jobs: metrics, dataset and BYOI

Programmatic jobs give an objective baseline; for open-ended quality you hand scoring to an evaluator model.

- **Built-in judge metrics:** `Builtin.Correctness`, `Completeness`, `Faithfulness`, `Helpfulness`, `Coherence`, `Relevance`, `FollowingInstructions`, `ProfessionalStyleAndTone`, plus the responsible AI metrics `Harmfulness`, `Stereotyping` and `Refusal`. **Custom metrics** (below) use your own judge prompt and rating scale.

**Prompt dataset JSONL**
```json
{"prompt":"What is our refund window for annual plans?","referenceResponse":"30 days from purchase.","category":"billing"}
```
BYOI variant (**bring your own inference responses**: Bedrock skips invocation and only judges, so it works for **SageMaker, on-prem or third-party** models):
```json
{"prompt":"What is our refund window?","referenceResponse":"30 days.","category":"billing",
 "modelResponses":[{"response":"You can get a refund within 30 days.","modelIdentifier":"sagemaker-llama-ft-v3"}]}
```
- `referenceResponse` is optional. When it's present, **Correctness** and **Completeness** use it.
- `category` makes the report break scores down **per category**.
- BYOI allows **one response per prompt** and **one `modelIdentifier` per job**. To compare 3 external models, run 3 jobs and compare the reports.

**Creating the job (boto3 sketch)**
```python
bedrock = boto3.client("bedrock")
bedrock.create_evaluation_job(
    jobName="support-bot-claude-vs-nova-2026-11",
    roleArn="arn:aws:iam::111122223333:role/BedrockEvalRole",
    applicationType="ModelEvaluation",
    evaluationConfig={"automated": {
        "datasetMetricConfigs": [{
            "taskType": "General",
            "dataset": {"name": "golden-support",
                        "datasetLocation": {"s3Uri": "s3://evals/golden/support.jsonl"}},
            "metricNames": ["Builtin.Correctness", "Builtin.Completeness",
                            "Builtin.Faithfulness", "Builtin.Helpfulness",
                            "Builtin.Harmfulness"]}],
        "evaluatorModelConfig": {"bedrockEvaluatorModels": [
            {"modelIdentifier": "us.anthropic.claude-sonnet-4-5-20250929-v1:0"}]}}},
    inferenceConfig={"models": [{"bedrockModel": {
        "modelIdentifier": "amazon.nova-pro-v1:0",
        "inferenceParams": "{\"inferenceConfig\":{\"maxTokens\":512,\"temperature\":0}}"}}]},
    outputDataConfig={"s3Uri": "s3://evals/results/"})
```
> The field names follow the CreateEvaluationJob API shape. Check the exact nesting in the docs (verify in docs) before using this in a pipeline.

**Custom metrics**
- You write the **judge prompt** (with placeholders such as `{{prompt}}` and `{{prediction}}`, plus `{{ground_truth}}` if available) and define a **rating scale** (numerical or categorical with definitions). Use them for business-specific checks such as "mentions the mandatory compliance disclaimer" or "uses our product terminology".
- Custom metrics support a slightly different list of judge models than built-in metrics (verify in docs).

#### Multi-model evaluation workflow
The job types above are the building blocks. Comparing candidates fairly means running them the same way:

1. Build a **representative golden dataset** of 200–1,000 prompts, stratified by intent, difficulty and language, including **edge cases** and **should-refuse** prompts.
2. Run the same dataset through each candidate (Bedrock models directly, others with BYOI) using **identical inference parameters** (temperature 0 for comparability).
3. Score with the same **judge + metrics**, plus deterministic checks.
4. Collect **cost and latency** for each candidate: tokens from the response `usage` or CloudWatch, and latency p50/p95.
5. Compare on a **quality vs cost** and **quality vs latency** plot. Choose the **Pareto-optimal** model that meets every threshold.

#### Cost-performance analysis

| Metric | Formula / source |
| --- | --- |
| Token efficiency | (input + output tokens) per **successful** task; output tokens per answer |
| Cost per request / per 1k requests | tokens × per-token price (on-demand) or amortised PT/Reserved cost |
| Cost per *resolved* task | total cost ÷ tasks meeting quality threshold |
| Latency-to-quality ratio | p95 latency ÷ mean quality score (lower = better) or quality per second |
| Time to first token (TTFT) | streaming UX metric; check the CloudWatch metric name for your API (verify in docs) |
| Business outcome | deflection %, CSAT/NPS, conversion, average handle time, escalation rate |

- **Prompt routers** (Intelligent Prompt Routing) can be evaluated as the generator, which tells you whether routing keeps quality while cutting cost.
- Inference parameters (temperature, top-p, max tokens) and the **prompt version** are part of the *configuration* under test. Evaluate the combination of **model + prompt + parameters + KB settings**, not the model alone.

#### A/B, canary and shadow testing of FMs (online evaluation)
Offline evaluation picks the candidate; online tests confirm it on real traffic.

| Pattern | Traffic | Risk | Measures | AWS mechanisms |
| --- | --- | --- | --- | --- |
| **A/B test** | Split (e.g. 50/50 or 90/10) by user/session hash | Medium | Business KPIs, feedback ratings, judge scores on sampled logs | **AppConfig** feature flag/variant, Lambda router, API Gateway stage variables, Prompt Management versions, **SageMaker production variants** (weights) |
| **Canary** | Small % (1–10%) to new version, ramp up | Low | Error rate, latency, guardrail interventions, judge scores → **auto-rollback** | **AppConfig deployment strategies** (linear/canary + bake time + **CloudWatch alarm rollback**), Lambda alias weighted routing, CodeDeploy for Lambda |
| **Shadow** | 100% mirrored; new version **not served** | Very low | Offline compare (diff, judge), latency, cost | **SageMaker shadow tests / shadow variants**, app-side async duplication to SQS |
| **Interleaving / pairwise** | Same user sees both outputs | Low | User preference | Custom UI |

- Keep the assignment **sticky by session** so a conversation doesn't switch models halfway through.
- Log `variant`, `modelId` and `promptVersion` with each request so every result can be traced back to the configuration that produced it.
- Use **enough sample size** and look at **segments** before declaring a winner. Online results tell you about business impact, while offline evaluation catches quality regressions before release.
- Lambda **alias weighted routing** splits traffic between two Lambda *versions*, each pinned to a different model/prompt config. This is a simple way to run a canary for Bedrock-backed functions.

---

### Skill 5.1.3 — User-centred evaluation mechanisms

> Develop user-centered evaluation mechanisms to continuously improve FM performance based on user experience (for example, by using feedback interfaces, rating systems for model outputs, annotation workflows to assess response quality).

#### Feedback signals

| Type | Example | Strength | Weakness |
| --- | --- | --- | --- |
| **Explicit binary** | 👍 / 👎 | High volume, easy | Low detail, negativity bias |
| **Explicit scale** | 1–5 stars, Likert | Graded signal | Inconsistent scales between users |
| **Explicit free text / reason codes** | "Incorrect", "Outdated", "Unsafe", "Too long" | Actionable root cause | Low response rate |
| **Pairwise preference** | "Which answer is better?" | Great for A/B and preference datasets (DPO) | Needs 2 outputs |
| **Implicit** | Copy/accept rate, regenerate clicks, follow-up rephrase, escalation to human, session abandonment, dwell time | No user effort | Noisy, needs interpretation |
| **Expert annotation** | SMEs label correctness, severity, category | High quality | Expensive, slow |

#### Reference architecture: feedback capture
```
UI (Amplify / web)  --POST /feedback {requestId, sessionId, rating, reasonCode, comment}-->
API Gateway (request validation model, throttling, Cognito authoriser)
  → Lambda (validate, enrich: userSegment, modelId, promptVersion, variant)
      → DynamoDB  Feedback table   PK=requestId  SK=ts   (+ GSI on modelId#promptVersion)
      → Firehose → S3 (Parquet) → Glue Catalog → Athena / QuickSight
      → CloudWatch EMF metric  "ThumbsDownRate" dim {modelId, promptVersion, intent}
Bedrock model invocation logs (S3 / CloudWatch Logs) joined on requestId
```
- The **join key** is everything: store the same `requestId` or trace ID in the app log, the invocation log and the feedback record. Without it you can't link a 👎 to the prompt, context and model that produced it.
- Emit a **custom CloudWatch metric** (EMF) for negative-feedback rate and add **alarms** or **anomaly detection** on it per model version, so a bad release triggers an alert.
- Feedback data has many uses:
  - grow the **golden dataset** from real failures
  - build **preference pairs** for fine-tuning (DPO) or reinforcement fine-tuning
  - add prompt fixes and few-shot examples
  - update the KB when answers are missing or outdated
- Apply **privacy controls** to free-text feedback (it can contain PII): Comprehend PII redaction before storage, and retention lifecycle rules. See [Task 3.2](../domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md).

#### Annotation workflows

| Service | Use |
| --- | --- |
| **Bedrock human evaluation jobs** | Structured side-by-side model comparison by your own work team. You define the metrics and rating methods. |
| **SageMaker Ground Truth** | Labelling jobs with **custom templates** (rate response 1–5, mark hallucinated spans, choose better answer). Workforce types: **private** (your employees), **vendor** (AWS Marketplace), **Mechanical Turk** (public, don't use for sensitive data). Ground Truth Plus is the AWS-managed labelling service. |
| **Amazon Augmented AI (A2I)** | **Human review loops** in production: trigger review when confidence is low, a guardrail intervenes or the case is high-risk. Results go to S3. |
| **Custom review UI + Step Functions** | `waitForTaskToken` for approve/edit before release (see [Task 2.1](../domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md) human-in-the-loop). |

- Rating methods used in human evaluation: **thumbs up/down**, **Likert scale** (individual or comparison), **ordinal ranking**, **choice buttons** (pick the better of two). Check which methods the current Bedrock human evaluation supports (verify in docs).
- Track **inter-annotator agreement** (for example Cohen's kappa) and use **gold questions** to monitor annotator quality. Give annotators a **rubric with examples**.
- Annotated data is also what you use to **calibrate LLM judges**: if judge and human scores correlate well, you can lean on the judge.

---

### Skill 5.1.4 — Systematic quality assurance processes 🎯 PQ37 🎯 PQ66

> Create systematic quality assurance processes to maintain consistent performance standards for FMs (for example, by using continuous evaluation workflows, regression testing for model outputs, automated quality gates for deployments).

#### What can regress (anything below can change outputs)
- the model ID or version (provider updates, migrating to a newer generation)
- inference parameters
- prompt template or system prompt version
- few-shot examples
- the KB: new documents, chunking, embedding model, reranker
- guardrail configuration
- tool definitions or schemas
- the agent's instructions or action groups
- upstream data formats

#### Regression testing for model outputs
- 🎯 **PQ30** A **golden dataset** is a versioned set of inputs, each with expected properties: a reference answer, required facts, a must-cite source, must refuse, a JSON schema, a max length. Store it in **S3 or Git** next to the prompts, and version it with them. Include **paraphrased versions** of key questions so consistency is tested too (PQ30 callout: [Task 3.1 · Skill 3.1.3](../domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md#skill-313--accuracy-verification-systems-to-reduce-hallucinations)).
- Assertions, from strictest to loosest:
  1. **Exact / structural**: valid JSON against the schema, required keys, enum values, tool called with the right parameters.
  2. **Contains / regex**: a required disclaimer or citation is present; banned phrases are absent.
  3. **Semantic**: embedding similarity to the reference ≥ τ (for example 0.85).
  4. **Judge**: correctness ≥ x, faithfulness ≥ y, harmfulness = 0.
- Use **temperature 0** (or a low value) plus several runs to reduce noise. Track the **pass rate per category**, not just single cases.
- Compare against the **baseline** (the current production version). Fail the build on a **statistically meaningful drop**, not a 1-case wobble. A common rule is "pass rate drops > 2 percentage points, or any critical case fails".

#### Automated quality gates (CodePipeline)
```
Source (Git: prompts/, flows/, golden/, infra/)
  → Build (CodeBuild: lint prompt templates, unit tests, schema checks, security scan)
  → Deploy to staging (CDK/CloudFormation: new Prompt version, Lambda alias, KB config)
  → Evaluate (CodeBuild: run golden set via Bedrock eval job or harness; compute scores)
        if score < threshold → exit 1  → stage FAILS → pipeline stops
  → Manual approval (optional, for high-risk changes; reviewers see eval report link)
  → Deploy to prod (canary via AppConfig / Lambda alias weights, CloudWatch alarm rollback)
```
`buildspec.yml` (evaluation stage sketch):
```yaml
version: 0.2
phases:
  build:
    commands:
      - python eval/run_golden.py --dataset s3://evals/golden/v12.jsonl --prompt-arn $PROMPT_ARN --out results.json
      - python eval/gate.py results.json --min-correctness 0.85 --min-faithfulness 0.90 --max-harmful 0 --max-p95-ms 3000
reports:
  genai-eval:
    files: [eval/junit.xml]
    file-format: JUNITXML
artifacts:
  files: [results.json]
```
- If the gate script exits non-zero, the build fails and the pipeline stops before promotion. Publish the results as **CodeBuild test reports** (JUnit) and store them in S3 as an audit trail.
- A Bedrock evaluation job runs **asynchronously**. Either poll `GetEvaluationJob` until it's complete and then parse the S3 results, or use a **Step Functions** stage that waits and polls. For fast per-commit checks, many teams run a small in-house harness and run the full Bedrock evaluation job nightly or before a release.
- Cross-reference: prompt-level QA (Lambda output verifiers, Step Functions edge-case tests, CloudWatch prompt regression) is in [Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md). CI/CD pipeline architecture is in [Task 2.3](../domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md).

#### Continuous evaluation (after deployment)
- **Sample production traffic** (for example 1–5%) from the **model invocation logs**. Use **EventBridge Scheduler** to run a Lambda or Step Functions job each night that scores the sample with an LLM judge or a Bedrock evaluation job using BYOI (the logged responses). Publish the scores as **CloudWatch custom metrics**.
- Alarm on **trends**: faithfulness falling, refusal rate rising, negative-feedback rate rising, guardrail intervention rate rising, tokens per answer rising.
- Re-run the golden set when a **provider updates a model**, on a **schedule**, and on **every config change**.
- AgentCore Evaluations **online evaluation** continuously scores sampled agent traces. Domain 4 [Task 4.3](../domain-4-operational-efficiency-optimization/03-task-4.3-monitoring-observability.md) covers the monitoring side: dashboards, anomaly detection and golden-dataset hallucination checks in operation.

> [!IMPORTANT]
> **🎯 Practice question 37 — 500 → 2,000+ golden examples, semantic similarity, scheduled, dashboards, alerts**
>
> - **EventBridge scheduled rule** → **Lambda** runs the golden set (invoke the FM, embed the output and the expected answer, compute **semantic similarity**) → **CloudWatch custom metrics** (`PutMetricData` / EMF) → **dashboards + alarms**. Lambda fans out, so going from 500 to 2,000+ cases needs **no architecture change**.
> - ❌ **Step Functions + DynamoDB:** no built-in semantic scoring, and lots of state transitions at this volume (per the answer key).
> - ❌ **CloudWatch Synthetics canaries:** **uptime, availability and simple assertions**, not semantic regression on 2,000 cases.
> - ❌ **Comprehend sentiment or key phrases:** says nothing about **semantic equivalence** to the expected answer.
> - 🔁 **Compare with PQ21:** there, canaries **replay user workflows** as one part of deployment validation. Here they can't be the **semantic evaluator**.

---

### Skill 5.1.5 — Multi-perspective assessment (RAG eval, LLM-as-a-judge, human feedback)

> Develop comprehensive assessment systems to ensure thorough evaluation from multiple perspectives for FM outputs (for example, by using RAG evaluation, automated quality assessment with LLM-as-a-Judge techniques, human feedback collection interfaces).

#### Why multiple perspectives?
Each method has blind spots:

| Perspective | Catches | Misses |
| --- | --- | --- |
| Deterministic checks | Format, schema, length, required strings | Meaning, correctness |
| Reference metrics | Drift from known answers | Valid paraphrases, open-ended tasks |
| LLM-as-a-judge | Nuanced quality at scale, explanations | Judge bias, domain subtleties, judge cost |
| Human (SME) | Domain correctness, tone, risk | Scale, consistency, speed |
| End users | Real usefulness, UX | Expert correctness; biased by tone/confidence |
| Business KPIs | Actual impact | Root cause |

#### Bedrock RAG evaluation

Two job types:
- **Retrieve only**: evaluates retrieval quality only, from a Bedrock KB (`Retrieve`) or **your own retrieval results** (BYOI from any RAG system).
- **Retrieve and generate**: evaluates the retrieved context plus the generated answer. It uses a Bedrock KB with a generator model (`RetrieveAndGenerate`) or BYOI from any external RAG pipeline.

| Retrieve-only metric | Meaning |
| --- | --- |
| `Builtin.ContextRelevance` | Are the retrieved passages relevant to the question? |
| `Builtin.ContextCoverage` | Do the retrieved passages cover **all** information in the ground-truth text? (**requires ground truth**) |

| Retrieve-and-generate metric | Meaning |
| --- | --- |
| `Builtin.Correctness` | Response answers the question accurately |
| `Builtin.Completeness` | Resolves all aspects of the question |
| `Builtin.Helpfulness` | Holistic usefulness |
| `Builtin.LogicalCoherence` | No gaps/contradictions |
| `Builtin.Faithfulness` | Avoids **hallucination relative to the retrieved texts** |
| `Builtin.CitationPrecision` | Fraction of cited passages that were **correctly** cited |
| `Builtin.CitationCoverage` | Response is supported by citations; **missing citations** penalised |
| `Builtin.Harmfulness` · `Builtin.Stereotyping` · `Builtin.Refusal` | Responsible AI metrics |

- The RAG dataset contains queries plus **ground truth** (expected retrieved text and/or expected answer). The evaluator models must be ones you have access to.
- Compare **multiple KB configurations** (chunking strategy, embedding model, reranker on or off, top-k) with the **same dataset**. That's the main use of RAG evaluation jobs.
- **RAGAS-style** open-source metrics map closely: *faithfulness*, *answer relevancy*, *context precision* (are relevant chunks ranked high?), *context recall* (does the context cover the ground truth?), *answer correctness*. Bedrock's `ContextRelevance` and `ContextCoverage` are roughly context precision and context recall.

#### Diagnosing RAG by combining metrics
No single RAG metric localises a fault; read them in pairs:

| Low metric | High metric | Likely cause | Fix |
| --- | --- | --- | --- |
| Context relevance | — | Bad retrieval: chunking, embeddings, query | Tune chunking/embedding, hybrid search, reranker, query reformulation ([Task 1.5](../domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md)) |
| Context coverage | Context relevance | Relevant but **incomplete** retrieval (top-k too small, chunks too small) | Increase k, hierarchical chunking, query decomposition |
| Faithfulness | Context relevance | Generator **ignores/goes beyond** context | Stronger grounding instructions, lower temperature, Guardrails contextual grounding, different model |
| Correctness | Faithfulness | Faithful to **wrong/outdated** sources | Fix the KB content / sync / metadata filters |
| Citation coverage | Correctness | Answer right but not attributed | Prompt to cite; use RetrieveAndGenerate citations |

#### LLM-as-a-judge best practices
- Know the three biases: **position bias**, **verbosity / length bias** and **self-preference** (favours its own model family).
- Use a **strong judge from a different family** than the generator to avoid self-preference bias.
- Give **explicit rubrics** with anchored scale definitions, and ask for **reasoning before the score**. Bedrock returns explanations with each score.
- Choose the mode deliberately: **pointwise** scoring is absolute and good for gates, **pairwise** comparison is better at finding which of two is better, and **reference-guided** grading is more accurate when you have ground truth.
- **Swap or randomise positions** in pairwise tests, and watch for **length bias**.
- **Calibrate**: score a human-labelled sample with the judge and check the correlation and agreement before trusting it.
- A judge **costs tokens** as well, so sample production traffic and keep the golden sets reasonably sized.

#### Human evaluation as the third perspective
Human feedback collection interfaces are covered in [5.1.3](#skill-513--user-centred-evaluation-mechanisms). Bedrock **human evaluation jobs** use your own work team, managed through Bedrock or SageMaker Ground Truth private workforces, and need **CORS** on the S3 bucket. Use them for **final sign-off** on high-stakes changes and to build **judge calibration** sets.
> Older materials mention an "AWS-managed work team" option for Bedrock human evaluation. Current docs describe teams of **your own employees or SMEs** (verify in docs whether the AWS-managed option is still offered).

---

### Skill 5.1.6 — Retrieval quality testing

> Implement retrieval quality testing to evaluate and optimize information retrieval components for FM augmentation (for example, by using relevance scoring, context matching verification, retrieval latency measurements).

Test retrieval **independently of generation**. A good generator can hide poor retrieval, and the reverse is also true.

#### Building a retrieval test set
- Each item is a query plus its **relevant document/chunk IDs**, optionally with **graded relevance** (0 = irrelevant, 1 = partial, 2 = highly relevant).
- Sources:
  - real user queries from logs, labelled by SMEs
  - **synthetic queries** generated by an LLM from each chunk ("write 3 questions this passage answers"), which makes that chunk the known positive
  - hard negatives: near-miss documents
- Include **filtered** queries (metadata filters such as tenant, date or product), **multi-hop** queries, keyword-heavy queries (SKUs, error codes) and **multilingual** queries if relevant.

#### Retrieval metrics

| Metric | Formula / idea | Use |
| --- | --- | --- |
| **Precision@k** | relevant in top k ÷ k | Noise in context (cost, distraction) |
| **Recall@k** | relevant in top k ÷ total relevant | Did we fetch what's needed? Most important for RAG |
| **Hit rate@k** | % of queries with ≥ 1 relevant in top k | Simple pass/fail |
| **MRR** | mean of 1 / rank of first relevant | Is the best chunk near the top? |
| **nDCG@k** | discounted cumulative gain with graded relevance, normalised | Ranking quality with graded labels; evaluating rerankers |
| **MAP** | mean of average precision across queries | Overall ranking quality |
| **Context relevance / coverage** (LLM-judged) | Bedrock RAG eval retrieve-only | When no chunk-ID labels exist |
| **Similarity score distribution** | Retrieve API `score` per result | Pick a minimum-score cutoff; detect "no good match" |

#### Context matching verification
- Check that the **expected source** (document ID or S3 URI) appears in the retrieved results or in the response **citations**. RetrieveAndGenerate returns `citations[].retrievedReferences[].location`.
- Check **metadata filter correctness**: a tenant-A query must **never** return tenant-B chunks. Make this a hard security test.
- Check **freshness**: after updating a document and syncing, the new version is returned and the old one isn't. This validates the sync pipeline from [Task 1.4](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md).

#### Retrieval latency measurement
- Time the `Retrieve` call, the vector DB query (OpenSearch `took`, Aurora `EXPLAIN ANALYZE`), the reranker and the embedding of the query **separately**. Report **p50/p95/p99**.
- Load-test at the expected QPS (for example with the **Distributed Load Testing on AWS** solution, Locust or k6). Watch OpenSearch Serverless OCU scaling, search latency and **k-NN memory / circuit breaker** metrics. Domain 4 covers the tuning.
- Trade-offs to measure explicitly:
  - **top-k**: higher gives better recall but more tokens and latency
  - **reranker**: better precision@small k but adds latency
  - **hybrid search**: helps keyword queries but costs more compute
  - **HNSW `ef_search`**: higher gives better recall but slower queries

#### Experiment loop (grid search)
```
for chunking in [fixed-300, fixed-512, hierarchical, semantic]:
  for embed in [titan-v2-1024, titan-v2-512, cohere-v3]:
    for rerank in [off, on]:
       build/point KB → run retrieval test set → record recall@5, nDCG@5, p95 latency, $/1k queries
pick config on the recall/latency/cost frontier → confirm with Bedrock RAG eval (retrieve-and-generate)
```
- Chunking, parser and embedding model are **fixed after KB creation**, so each variant needs its **own KB/index**. Plan for that cost.

---

### Skill 5.1.7 — Agent performance frameworks

> Develop agent performance frameworks to ensure that agents perform tasks correctly and efficiently (for example, by using task completion rate measurements, tool usage effectiveness evaluations, Amazon Bedrock Agent evaluations, reasoning quality assessment in multi-step workflows).

Agents fail in ways a single model call doesn't: wrong tool, wrong parameters, loops, giving up early, overly long trajectories, ignoring tool errors, or claiming success that didn't happen.

#### Agent metric framework

| Level | Metric | How to measure |
| --- | --- | --- |
| **Outcome (session)** | **Task / goal completion rate** | Did the end state match the goal? Check side effects (record created, ticket closed) or judge against expected outcome |
| | Goal success with ground truth | Compare final state/answer to expected |
| | Human escalation / handoff rate | Count escalations |
| **Trajectory (turn/trace)** | Steps / LLM calls / tool calls per task | Traces (Bedrock Agent trace, OTel spans) |
| | Trajectory match | Compare to the reference tool sequence (exact, in-order, any-order, superset) |
| | Loop / repetition rate | Same tool + same args repeated |
| | Reasoning quality | Judge rationale for logical validity, plan adherence, use of observations |
| **Tool (span)** | **Tool selection accuracy** | Right tool chosen for the step (vs labelled expectation or judge) |
| | **Tool parameter accuracy** | Args correct and valid against schema |
| | Tool error rate / retries | Lambda errors, validation failures |
| | Tool latency | Span duration |
| **Response** | Correctness, faithfulness to tool outputs, helpfulness, harmfulness | LLM judge |
| **Efficiency** | Tokens, cost, wall-clock per task | Usage metrics, traces |

#### Amazon Bedrock AgentCore Evaluations
- Managed **LLM-as-a-judge** evaluation for agents. It integrates with **Strands** and **LangGraph** through **OpenTelemetry / OpenInference** instrumentation. Traces are converted to a **unified format** and scored.
- Evaluators have ARNs. Built-in ones look like `arn:aws:bedrock-agentcore:::evaluator/Builtin.Helpfulness`. You can create **custom evaluators** (your own prompt and model), and access is controlled with **resource-based policies**. There are also third-party evaluators.
- Evaluation modes: **on-demand** (score chosen sessions or traces), **online** (continuous sampling of live traffic), **batch**, **dataset** evaluation, and **simulation** (simulated users driving the agent).
- Built-in evaluators by level:

| Level | Built-in evaluators |
| --- | --- |
| **Session** | Goal success rate (± ground truth) |
| **Trace (turn)** | Coherence, Conciseness, Correctness (± ground truth), Faithfulness, Harmfulness, Helpfulness, Instruction following, Refusal, Response relevance, Stereotyping |
| **Tool** | Tool selection accuracy, Tool parameter accuracy (+ skill selection / skill instruction following for skill-loading tools) |

- Built-in evaluator configurations (their judge model and prompt) **can't be modified**. For different behaviour, create a custom evaluator.
- The default quota is **1,000 evaluation configurations per Region**.

#### Bedrock Agents (classic) evaluation approach
- Turn on **`enableTrace`** in `InvokeAgent` to capture **pre-processing, orchestration (rationale, invocation input, observation), post-processing, guardrail and failure traces**. Score these traces with a judge, or check tool calls against expectations.
- Build a **scenario test suite** of conversations with expected tool calls, parameters and final answers. Replay it against an **agent alias** (for example `test`) before moving the `prod` alias to the new version.
- Use **return of control** or mocked action-group Lambdas in tests, so you check what the agent *would* call without side effects.
- Evaluate the agent's final responses with a Bedrock LLM-as-a-judge job using **BYOI** (the logged final answers).

#### Strands / framework-level
- Strands has an evaluation SDK (Strands Evals; verify in docs) for test cases, trajectory assertions and judge-based scoring. The general pattern is to unit-test tools with mocks, integration-test agent trajectories, and judge final answers.
- In multi-agent systems, also measure **routing accuracy** (did the supervisor or classifier pick the right sub-agent), **hand-off success** and **coordination overhead** (extra turns and tokens).

#### Reasoning quality in multi-step workflows
- Judge the **plan** (is it sensible and minimal), each **step's rationale** (does it follow from the observations) and **self-correction** (does it recover after a tool error).
- Detect **unsupported success claims**: the agent says "I've booked it" but no booking tool call succeeded. Compare the claim with the tool results in the trace.
- Set **budgets** (max steps, tokens, time) and treat hitting the budget as a failure in evaluations. See the safeguards in [Task 2.1](../domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md).

---

### Skill 5.1.8 — Stakeholder reporting

> Create comprehensive reporting systems to communicate performance metrics and insights effectively to stakeholders for FM implementations (for example, by using visualization tools, automated reporting mechanisms, model comparison visualizations).

#### Know your audiences

| Stakeholder | Wants | Format |
| --- | --- | --- |
| Executives / product owners | Business impact, cost, risk, trend | QuickSight dashboard, monthly **report card** (green/amber/red) |
| ML / app engineers | Per-metric scores, failing slices, regressions, traces | CloudWatch dashboards, eval reports in S3, Athena queries |
| Risk / compliance | Safety metrics, guardrail interventions, audit trail, human review stats | Scheduled PDF/CSV, Audit Manager evidence, model cards |
| Ops / SRE | Latency, errors, throttles, cost anomalies | CloudWatch dashboards + alarms |

#### Building the reporting pipeline
```
Bedrock eval job results (S3 JSONL)  ┐
Golden-set CI results (S3)           ├─► Glue crawler → Data Catalog → Athena views ─► Amazon QuickSight
Invocation logs (S3) + feedback (DDB→S3) ┘                                           (SPICE, scheduled refresh,
CloudWatch metrics (latency, tokens, guardrail interventions) ─► CloudWatch dashboards   email reports, Q in QuickSight)
EventBridge Scheduler → Lambda: summarise week (Bedrock writes narrative) → SNS / email / Slack
```
- **QuickSight** supports **scheduled email reports**, row-level security per team, and natural-language Q&A over the dashboards (QuickSight Q / Amazon Q in QuickSight). It's the usual answer for business stakeholder dashboards.
- **CloudWatch dashboards** suit operational, near-real-time metrics. They can be shared, and they can embed **Logs Insights** query widgets.
- **Bedrock console evaluation reports** show score histograms and explanations, and you can **compare evaluation jobs** side by side (verify in docs which job types support comparison).
- Use an **LLM to generate the narrative** ("Faithfulness fell 4 points after KB sync on 3 Nov; the cause is outdated policy docs in category billing"), but base it on computed numbers and link to the data.

#### Model comparison visualisations
- **Radar / spider chart**: one axis per quality dimension, one polygon per model.
- **Scatter plot**: quality score vs cost per 1k requests, bubble size = p95 latency. This shows the **Pareto frontier**.
- **Grouped bars**: metric by model × category, which exposes weak slices.
- **Score distribution histograms**: two models can have the same mean but one has a long tail of failures.
- **Trend lines**: metrics over time and releases, with deployment annotations.
- **Confusion-style matrices**: intent classification, routing accuracy.

Always show the **dataset version, judge model, prompt version and date** on a report. Without them, results can't be reproduced or compared.

---

### Skill 5.1.9 — Deployment validation systems 🎯 PQ21

> Create deployment validation systems to maintain reliability during FM updates (for example, by using synthetic user workflows, AI-specific output validation for hallucination rates and semantic drift, automated quality checks to ensure response consistency).

#### Typical FM updates to validate
- moving to a new model version or generation
- a provider's minor update
- switching to Provisioned Throughput or a cross-Region inference profile
- a new fine-tune or distillation
- a prompt or guardrail change
- KB re-index or embedding model change
- agent instruction or tool change

#### Validation stages

| Stage | What runs | Pass criteria |
| --- | --- | --- |
| Pre-deploy (CI) | Golden regression + judge metrics ([5.1.4](#skill-514--systematic-quality-assurance-processes--pq37--pq66)) | Thresholds vs baseline |
| Staging smoke | **Synthetic user workflows** end to end (UI → API → agent → tools → KB) | All journeys pass assertions |
| Shadow (optional) | Mirror prod traffic to new version, compare outputs | Drift and judge scores within bounds |
| Canary in prod | 1–10% traffic + **CloudWatch Synthetics canaries** + live metrics | No alarms during bake time |
| Full rollout | Ramp up; continuous evaluation | KPIs stable |
| Rollback | Automatic on alarm (AppConfig, CodeDeploy, Lambda alias) | Previous version restored |

#### Synthetic user workflows: CloudWatch Synthetics canaries
- Canaries are scripted, scheduled **Node.js or Python (Puppeteer/Selenium/Playwright)** tests that call your API or drive the UI. They publish `SuccessPercent` and `Duration` metrics, which you can alarm on.
- For GenAI, add **semantic assertions** inside the canary: the response contains the expected fact, is valid JSON, includes a citation, or is semantically similar to the reference above a threshold (compute embedding similarity with a Bedrock embeddings call in the canary or a downstream Lambda).
```python
# Synthetics canary step (Python sketch)
resp = call_chat_api("What is the refund window for annual plans?")
assert resp["status"] == 200
answer = resp["body"]["answer"]
assert "30 days" in answer, "required fact missing"
assert resp["body"]["citations"], "no citation returned"
sim = cosine(embed(answer), REF_EMBED)       # Titan Text Embeddings V2
assert sim >= 0.80, f"semantic drift {sim:.2f}"
```
- Run canaries **every few minutes** in each Region. Use a fixed set of **non-sensitive synthetic queries** and tag them so they're excluded from business metrics and feedback stats.

#### AI-specific output validation

| Check | Method | Signal |
| --- | --- | --- |
| **Hallucination rate** | % responses with faithfulness < τ (LLM judge or **Guardrails contextual grounding** score) on golden + sampled traffic; or claims not supported by sources | Rising → block/rollback |
| **Semantic drift** | Embedding cosine distance between **old-version and new-version answers** to the same prompts; distribution shift of production answer embeddings over time | Large mean distance or new clusters → investigate |
| **Response consistency** | Same prompt × N runs (or paraphrases) → pairwise semantic similarity; variance of judge scores; JSON schema validity rate | High variance → lower temperature, tighten prompt |
| **Format compliance** | JSON Schema validation, required fields, length | < 100% for structured outputs → fail |
| **Safety regression** | Guardrail intervention rate, harmfulness metric on red-team set | Any increase on critical set → fail |
| **Refusal shift** | Refusal rate on should-answer and should-refuse sets | Over-refusal or under-refusal |
| **Cost/latency regression** | Tokens per response, p95 latency, TTFT | Over budget → fail |

- 🎯 **PQ21** **Output diffing**: show the old and new answers side by side for the golden set and flag cases where meaning changed (low similarity or judge disagreement). Reviewers only look at the flagged diffs.
- Pin model **versions** explicitly (full model ID, not a floating alias) so an update is a **deliberate** change that passes through this validation.
- **Rollback triggers**: CloudWatch alarms on the canary's `SuccessPercent`, error rate, guardrail intervention rate, negative feedback rate or a hallucination-rate custom metric. Wire them to **AppConfig deployment rollback**, **CodeDeploy** or a **Lambda alias** shift back. For SageMaker endpoints, use **deployment guardrails** (blue/green with canary or linear traffic shifting and **auto-rollback alarms**).

> [!IMPORTANT]
> **🎯 Practice question 21 — sprint releases: pre-release gate + post-deploy monitoring + alarms + auto-rollback**
>
> The CodePipeline pre-deploy stage → **Step Functions** workflow must include **all four** of these:
> 1. **Bedrock model evaluations** (LLM-as-a-judge: faithfulness, correctness, coherence) **against the production baseline** → hallucination rate.
> 2. **CloudWatch Synthetics canaries** → **replay synthetic user workflows** for the key intents.
> 3. **Embedding-based consistency checks** against a **reference prompt set** → cosine similarity between new and baseline answers = **semantic drift**.
> 4. **Automatic approve *or* rollback**, plus **scheduled post-deploy runs (e.g. nightly) with CloudWatch alarms** that trigger rollback.
> - ❌ **Evals + canaries + auto-approve only:** no embedding consistency check (performance metrics ≠ semantic consistency).
> - ❌ **Manual approval / weekly / no alarms:** not automated and not continuous.
> - ❌ **Canaries + consistency only, no model evaluations:** no AI-specific hallucination evaluation.

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Evaluate summarisation quality against reference summaries | **ROUGE / BERTScore** (Bedrock summarisation task uses **BERTScore** on Gigaword) |
| Evaluate translation output against references | **BLEU** |
| Measure fluency of generated text | **Perplexity** (or judge/human fluency rating) |
| Need to compare FMs quickly on accuracy, robustness and toxicity with standard datasets | **Bedrock automatic (programmatic) evaluation job** with built-in datasets |
| Open-ended answers, no single correct answer, need scalable human-like scoring with explanations | **LLM-as-a-judge** model evaluation job |
| Evaluate a SageMaker-hosted or third-party model using Bedrock evaluation | **Bring your own inference responses** (`modelResponses` in JSONL) |
| Need brand-voice / domain-expert judgment on outputs | **Human-based evaluation job** with your own work team |
| Pick a model balancing quality and cost | Cost-performance analysis: **cheapest model meeting quality thresholds**; quality vs $ Pareto |
| Measure business impact of model B vs A | **A/B test** with sticky assignment + KPIs + feedback |
| Release new model to 5% of traffic, auto-revert on errors | **Canary**: AppConfig deployment strategy / Lambda alias weights + **CloudWatch alarm rollback** |
| Compare new model on real traffic without exposing users | **Shadow testing** (SageMaker shadow variants / mirrored requests) |
| Collect thumbs up/down and link to prompts | **API Gateway → Lambda → DynamoDB**, join on **requestId** with invocation logs |
| Route low-confidence answers to human reviewers | **Amazon A2I** human review loop (or Step Functions waitForTaskToken) |
| Experts annotate hallucinated spans in responses | **SageMaker Ground Truth** custom labelling job, private workforce |
| Detect regressions when prompt/model/KB changes | **Regression tests on a versioned golden dataset** |
| Block deployment if eval scores drop | **Automated quality gate** in CodePipeline (CodeBuild eval step exits non-zero) |
| Measure whether the RAG answer contains claims not in the retrieved context | **Faithfulness** metric (RAG retrieve-and-generate) |
| Check retrieved chunks cover everything in the ground-truth answer | **Context coverage** (retrieve-only; needs ground truth) |
| Check retrieved chunks are relevant to the question | **Context relevance** |
| Answers are right but sources not properly cited | **Citation precision / citation coverage** metrics |
| Compare several KB chunking/embedding configurations | **RAG evaluation jobs** on the same dataset, one per KB config |
| Evaluate an external (non-Bedrock) RAG pipeline | RAG eval with **BYOI** retrieval/generation results |
| Judge seems to favour its own model's outputs | **Self-preference bias** → use judge from different family; calibrate with humans |
| Did retrieval fetch all needed documents? | **Recall@k** |
| Is the first relevant chunk near the top? | **MRR** |
| Need ranking-quality metric with graded relevance labels | **nDCG@k** |
| Measure agent success at end-to-end tasks | **Task / goal completion rate** (AgentCore **goal success rate** evaluator) |
| Agent picks the wrong tool or passes bad arguments | **Tool selection accuracy** / **tool parameter accuracy** evaluators; trace analysis |
| Continuously score live agent traffic with LLM-as-a-judge | **AgentCore Evaluations online evaluation** |
| Capture Bedrock Agent reasoning steps for evaluation | **`enableTrace`** in InvokeAgent → orchestration traces |
| Business stakeholders need scheduled dashboards of eval results | **QuickSight** (Athena over S3 results), scheduled email reports |
| Visualise models on quality vs cost | **Scatter / Pareto chart**; radar chart per dimension |
| Continuously verify the chatbot end-to-end every 5 min | **CloudWatch Synthetics canary** with semantic assertions |
| Detect that answers changed meaning after model upgrade | **Semantic drift**: embedding distance old vs new outputs; output diffing |
| Same question gives inconsistent answers | **Response consistency check** (N runs → similarity variance); lower temperature |

## Flashcards

<details>
<summary><b>Q:</b> Why are accuracy and F1 alone insufficient for evaluating FM outputs?</summary>

Outputs are open-ended and non-deterministic, with many valid answers. You need quality dimensions such as relevance, correctness, faithfulness, completeness, coherence, fluency and safety, scored by references, judges and humans.

*D5 · Task 5.1 · GenAI vs ML evaluation*
</details>

<details>
<summary><b>Q:</b> Which metric suits translation, which suits summarisation, and which suits fluency?</summary>

BLEU for translation (precision-oriented), ROUGE for summarisation (recall-oriented) and perplexity for fluency.

*D5 · Task 5.1 · Reference metrics*
</details>

<details>
<summary><b>Q:</b> Name the four Amazon Bedrock evaluation job types.</summary>

Automatic (programmatic) model evaluation, human-based model evaluation, model evaluation with an LLM as a judge, and RAG evaluation (retrieve-only or retrieve-and-generate).

*D5 · Task 5.1 · Bedrock Evaluations*
</details>

<details>
<summary><b>Q:</b> What three metric categories does a Bedrock automatic (programmatic) evaluation compute?</summary>

Accuracy, robustness (the score delta under prompt perturbation) and toxicity.

*D5 · Task 5.1 · Programmatic eval*
</details>

<details>
<summary><b>Q:</b> Which computed metric does Bedrock use for summarisation accuracy, and on which dataset?</summary>

BERTScore, on the Gigaword dataset.

*D5 · Task 5.1 · Built-in datasets*
</details>

<details>
<summary><b>Q:</b> List five built-in LLM-as-a-judge quality metrics in Bedrock, plus the three responsible AI ones.</summary>

Any five of Correctness, Completeness, Faithfulness, Helpfulness, Coherence, Relevance, FollowingInstructions and ProfessionalStyleAndTone. The responsible AI metrics are Harmfulness, Stereotyping and Refusal.

*D5 · Task 5.1 · LLM-as-a-judge metrics*
</details>

<details>
<summary><b>Q:</b> How do you evaluate a model hosted outside Bedrock with Bedrock's LLM-as-a-judge?</summary>

Use bring your own inference responses: add `modelResponses: [{response, modelIdentifier}]` to each JSONL line. Bedrock skips invocation and only judges. One modelIdentifier per job.

*D5 · Task 5.1 · BYOI*
</details>

<details>
<summary><b>Q:</b> What is the maximum number of prompts in an LLM-as-a-judge evaluation dataset?</summary>

1,000 prompts per evaluation job.

*D5 · Task 5.1 · Dataset limits*
</details>

<details>
<summary><b>Q:</b> What does the optional `referenceResponse` field change in a judge evaluation?</summary>

The judge uses it as ground truth when scoring Correctness and Completeness. Both metrics also work without it.

*D5 · Task 5.1 · Prompt dataset*
</details>

<details>
<summary><b>Q:</b> Canary vs shadow vs A/B testing for an FM change: what's the key difference?</summary>

A canary serves a small share of users the new version and rolls back automatically on alarms. Shadow mirrors traffic to the new version without serving its responses. A/B splits users to compare business KPIs.

*D5 · Task 5.1 · Online testing*
</details>

<details>
<summary><b>Q:</b> Which AWS feature gives a gradual config rollout with automatic rollback on a CloudWatch alarm?</summary>

AWS AppConfig deployment strategies (linear or canary with a bake time), with CloudWatch alarms configured as rollback triggers.

*D5 · Task 5.1 · Canary*
</details>

<details>
<summary><b>Q:</b> What's the critical design element when collecting user feedback on FM responses?</summary>

A shared request or trace ID, so you can join the feedback to the invocation log (prompt, context, model, prompt version).

*D5 · Task 5.1 · Feedback*
</details>

<details>
<summary><b>Q:</b> A2I vs Ground Truth for human review of GenAI outputs?</summary>

A2I runs human review loops in production, triggered by low confidence or risk. Ground Truth runs batch labelling and annotation jobs with custom templates and a private, vendor or public workforce.

*D5 · Task 5.1 · Annotation*
</details>

<details>
<summary><b>Q:</b> How do you implement an automated quality gate for a prompt change in CodePipeline?</summary>

Add a CodeBuild stage that runs the golden dataset evaluation and exits non-zero when scores fall below the thresholds. The stage fails, so promotion stops. Publish JUnit test reports as evidence.

*D5 · Task 5.1 · Quality gates*
</details>

<details>
<summary><b>Q:</b> Which RAG evaluation metric needs ground truth, and what does it measure?</summary>

Context coverage, a retrieve-only metric. It measures how much of the ground-truth information the retrieved passages cover.

*D5 · Task 5.1 · RAG eval*
</details>

<details>
<summary><b>Q:</b> Context relevance is high but faithfulness is low. What's wrong?</summary>

Retrieval is fine, but the generator ignores or goes beyond the context. Strengthen the grounding instructions, lower the temperature, add a Guardrails contextual grounding check, or try another model.

*D5 · Task 5.1 · RAG diagnosis*
</details>

<details>
<summary><b>Q:</b> Faithfulness is high but correctness is low. Where is the problem?</summary>

In the knowledge base content. The model faithfully repeats wrong or outdated sources, so fix the data, the sync pipeline or the metadata filters.

*D5 · Task 5.1 · RAG diagnosis*
</details>

<details>
<summary><b>Q:</b> What's the difference between citation precision and citation coverage?</summary>

Citation precision is the share of cited passages that were cited correctly. Citation coverage is how well the response is supported by its citations, and it penalises missing citations.

*D5 · Task 5.1 · Citation metrics*
</details>

<details>
<summary><b>Q:</b> Name three biases of LLM-as-a-judge and how to reduce them.</summary>

Position bias, verbosity bias and self-preference. Randomise or swap the order of answers, use anchored rubrics, use a judge from a different model family, and calibrate against human labels.

*D5 · Task 5.1 · Judge bias*
</details>

<details>
<summary><b>Q:</b> MRR vs nDCG: when would you use each?</summary>

MRR when you care about the rank of the first relevant result. nDCG when you have graded relevance labels and want to evaluate the whole ranking, for example to judge a reranker.

*D5 · Task 5.1 · Retrieval metrics*
</details>

<details>
<summary><b>Q:</b> Why do chunking and embedding experiments need separate knowledge bases?</summary>

The parser, chunking strategy, embedding model and vector store are fixed once a KB is created, so each variant needs its own KB and index.

*D5 · Task 5.1 · Retrieval experiments*
</details>

<details>
<summary><b>Q:</b> Name the three evaluator levels in AgentCore Evaluations, with an example for each.</summary>

Session level (goal success rate), trace level (correctness, faithfulness, helpfulness, harmfulness and others) and tool level (tool selection accuracy, tool parameter accuracy).

*D5 · Task 5.1 · AgentCore Evaluations*
</details>

<details>
<summary><b>Q:</b> What evaluation modes does AgentCore Evaluations support?</summary>

On-demand, online (continuous sampling of live traces), batch, dataset evaluation and simulation. It consumes OpenTelemetry/OpenInference traces from Strands, LangGraph and other frameworks.

*D5 · Task 5.1 · AgentCore Evaluations*
</details>

<details>
<summary><b>Q:</b> How do you check a Bedrock Agent's reasoning and tool use in tests?</summary>

Set `enableTrace` on InvokeAgent and inspect the pre-processing, orchestration (rationale, invocation input, observation) and post-processing traces. Compare tool calls and parameters with expectations, and judge the rationale.

*D5 · Task 5.1 · Bedrock Agents eval*
</details>

<details>
<summary><b>Q:</b> Which tool fits executive-facing, scheduled dashboards of model evaluation results?</summary>

Amazon QuickSight over Athena (Glue Catalog on S3 results), with scheduled email reports. CloudWatch dashboards are for operational metrics.

*D5 · Task 5.1 · Reporting*
</details>

<details>
<summary><b>Q:</b> How do you detect semantic drift after a model upgrade?</summary>

Run the same prompts through the old and new versions, embed the outputs and measure the cosine distance. Flag large distances for diffing or judge review. Also watch for shifts in the distribution of production answer embeddings over time.

*D5 · Task 5.1 · Deployment validation*
</details>

<details>
<summary><b>Q:</b> What does a CloudWatch Synthetics canary add to GenAI deployment validation?</summary>

Scheduled end-to-end synthetic user journeys with assertions (required facts, citations, JSON validity, semantic similarity). It publishes SuccessPercent and Duration metrics that can trigger alarms and rollback.

*D5 · Task 5.1 · Synthetic monitoring*
</details>

---
[← Back to Domain 5](README.md)
