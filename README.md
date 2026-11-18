# AWS Certified Generative AI Developer – Professional (AIP-C01) · Detailed Notes

These are detailed, skill-by-skill notes that follow the **official AIP-C01 exam guide** outline. The companion high-level revision notes are in [study-notes](<../study-notes/Certifications/AWS/AIP-C01 - Gen AI Pro/README.md>).

> **Unofficial.** These are independent study notes. They are not affiliated with or endorsed by Amazon Web Services. Check the [official exam guide](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01.html) and AWS documentation for the latest details.

## Exam domains

| # | Domain | Weight | Notes |
| --- | --- | --- | --- |
| 1 | Foundation Model Integration, Data Management, and Compliance | 31% | [Domain 1](domain-1-fm-integration-data-compliance/README.md) |
| 2 | Implementation and Integration | 26% | [Domain 2](domain-2-implementation-integration/README.md) |
| 3 | AI Safety, Security, and Governance | 20% | [Domain 3](domain-3-ai-safety-security-governance/README.md) |
| 4 | Operational Efficiency and Optimization for GenAI Applications | 12% | [Domain 4](domain-4-operational-efficiency-optimization/README.md) |
| 5 | Testing, Validation, and Troubleshooting | 11% | [Domain 5](domain-5-testing-validation-troubleshooting/README.md) |

## 🎯 Practice-question highlights

Parts of the notes tested by a practice question set (PQ1–PQ75) are marked like this:

- **`🎯 PQ#`** tags on the key lines, table rows and headings.
- **Coloured callout boxes** after the relevant block, with the scenario, the right answer and why each distractor is wrong:
  - 🟣 `[!IMPORTANT]`: the notes already covered this content.
  - 🟠 `[!WARNING]`: this content was missing and has been added (PQ4 AgentCore limits, PQ5 cross-account access, PQ9 text-to-SQL validation, PQ13 video pipeline, PQ14 OpenSearch Serverless permissions, PQ17 Application Insights, PQ20 user-scoped agent access, PQ22 boundary vs SCP split, PQ24 Transcribe streaming, PQ29 Step Functions vs Flows, PQ32 streaming APIs, PQ38 webhook auth, PQ52 SageMaker Neo, PQ53 Lake Formation cross-account, PQ67 image invocation logging, PQ72 SCP vs boundary conflict).

Run `grep -rn "🎯" .` from the repo root to list all highlights.

| PQ | Topic | Where |
| --- | --- | --- |
| 1 | Distillation (Nova Pro → Nova Lite, on-demand) | [Task 1.2](domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md) · [Task 4.1](domain-4-operational-efficiency-optimization/01-task-4.1-cost-optimization-resource-efficiency.md) |
| 2 | Semantic chunking (buffer size, breakpoint percentile) | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) |
| 3 | Bedrock prompt caching for uploaded docs/images | [Task 4.1](domain-4-operational-efficiency-optimization/01-task-4.1-cost-optimization-resource-efficiency.md) |
| 4 | AgentCore Runtime (8 h), Memory, Identity (Okta), Observability | [Task 2.1](domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md) |
| 5 | Cross-account AssumeRole + session tags; RAM/SCP traps | [Task 2.3](domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md) · [Task 3.2](domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md) |
| 6 | Glue lineage + Guardrails PII + CloudTrail + Macie | [Task 3.3](domain-3-ai-safety-security-governance/03-task-3.3-governance-compliance.md) · [Task 3.2](domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md) |
| 7 | Hybrid search (BM25 + vector) + weighted fusion + rerank | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) · [Task 4.2](domain-4-operational-efficiency-optimization/02-task-4.2-application-performance.md) |
| 8 | Cross-encoder reranking (retrieve 50 → top 10) | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) · [Task 4.2](domain-4-operational-efficiency-optimization/02-task-4.2-application-performance.md) |
| 9 | Text-to-SQL: Guardrails + SQL parser + RDS Data API | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 10 | Temperature + templates + caching + guardrails + backoff | [Task 4.2](domain-4-operational-efficiency-optimization/02-task-4.2-application-performance.md) |
| 11 | Hierarchical chunking + hybrid search + score threshold (textbooks) | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) |
| 12 | Prompt Management variants + Guardrails (one threshold per guardrail) | [Task 1.6](domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) · [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 13 | Video metadata: frames + Transcribe timestamps + Claude + unified JSON | [Task 1.3](domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md) |
| 14 | `bedrock:InvokeAgent` + `aoss:APIAccessAll` + pattern-based data access policy | [Task 1.4](domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md) · [Task 5.2](domain-5-testing-validation-troubleshooting/02-task-5.2-troubleshooting.md) |
| 15 | KB RetrieveAndGenerate citations + session IDs for audit traceability | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) · [Task 3.3](domain-3-ai-safety-security-governance/03-task-3.3-governance-compliance.md) |
| 16 | Step Functions + DynamoDB circuit breaker (error-rate window, auto-resume) | [Task 1.2](domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md) |
| 17 | CloudWatch Application Insights + EMF + anomaly detection | [Task 4.3](domain-4-operational-efficiency-optimization/03-task-4.3-monitoring-observability.md) |
| 18 | Long transcripts: dynamic chunking + continuity + truncation diagnostics | [Task 5.2](domain-5-testing-validation-troubleshooting/02-task-5.2-troubleshooting.md) |
| 19 | Prune + summarise chat history in Lambda + token limits | [Task 4.1](domain-4-operational-efficiency-optimization/01-task-4.1-cost-optimization-resource-efficiency.md) |
| 20 | AgentCore Runtime + Identity on-behalf-of + Cognito groups → IAM roles → Lake Formation | [Task 3.2](domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md) · [Task 2.1](domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md) |
| 21 | Deployment validation: Bedrock evals + Synthetics + embedding consistency + auto-rollback | [Task 5.1](domain-5-testing-validation-troubleshooting/01-task-5.1-evaluation-systems.md) |
| 22 | Permissions boundary (`bedrock:GuardrailIdentifier`) + SCP (approved models) + StackSets guardrail | [Task 2.3](domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md) · [Task 1.1](domain-1-fm-integration-data-compliance/01-task-1.1-requirements-solution-design.md) |
| 23 | Semantic cache: Bedrock embeddings + MemoryDB for Valkey vector RANGE query | [Task 4.1](domain-4-operational-efficiency-optimization/01-task-4.1-cost-optimization-resource-efficiency.md) · [Task 1.4](domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md) |
| 24 | Transcribe streaming (~100 ms chunks) + InvokeModelWithResponseStream + DynamoDB ratings | [Task 2.4](domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md) |
| 25 | Glue crawlers + S3 object tags + invocation logs with Object Lock + CloudTrail integrity validation | [Task 3.3](domain-3-ai-safety-security-governance/03-task-3.3-governance-compliance.md) |
| 26 | Guardrails denied topic + sensitive info filters + invocation logs in S3 | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) · [Task 1.6](domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) |
| 27 | Shared AWS CDK construct library | [Task 1.1](domain-1-fm-integration-data-compliance/01-task-1.1-requirements-solution-design.md) |
| 28 | US Provisioned Throughput + EU geographic inference profile + failover | [Task 1.2](domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md) |
| 29 | Step Functions Standard + Map + InvokeModel + BDA with a back-off loop | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) · [Task 2.5](domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md) |
| 30 | Golden Q&A set + contextual grounding + Bedrock evaluations | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) · [Task 5.1](domain-5-testing-validation-troubleshooting/01-task-5.1-evaluation-systems.md) |
| 31 | Textract structure + Comprehend PII redaction before the FM | [Task 1.3](domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md) |
| 32 | HTTP API + Lambda response streaming + token limits + retries (⚠️ REST streaming now exists) | [Task 2.4](domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md) |
| 33 | Zero-touch prompt promotion: Prompt Management versions + LLM-judge evals + Flow in CI/CD | [Task 1.6](domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) |
| 34 | AppConfig feature flags + Lambda extension + linear 20% rollout + alarm rollback | [Task 1.2](domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md) |
| 35 | OpenSearch multi-node k-NN + index tuning + memory circuit breakers | [Task 4.2](domain-4-operational-efficiency-optimization/02-task-4.2-application-performance.md) |
| 36 | OpenSearch k-NN + Bedrock embeddings vs pgvector or KB (least dev effort at scale) | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) |
| 37 | Golden-set regression: EventBridge schedule + Lambda semantic similarity + CloudWatch metrics and alarms | [Task 5.1](domain-5-testing-validation-troubleshooting/01-task-5.1-evaluation-systems.md) |
| 38 | Webhooks: REST API + Lambda authorisers (multiple auth methods) + DynamoDB global tables | [Task 2.3](domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md) |
| 39 | Invocation logs → metric filters per tool → anomaly detection | [Task 4.3](domain-4-operational-efficiency-optimization/03-task-4.3-monitoring-observability.md) |
| 40 | KB metadata filtering to narrow the search scope | [Task 1.4](domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md) |
| 41 | Titan Text Embeddings V2 multilingual (no translation) + semantic chunking | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) |
| 42 | Strands on AgentCore Runtime + KB + Gateway MCP tools + Observability | [Task 2.1](domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md) |
| 43 | Agent OrchestrationTrace (modelInvocationInput/Output) + API-schema boundary checks | [Task 3.4](domain-3-ai-safety-security-governance/04-task-3.4-responsible-ai.md) |
| 44 | Glue Data Quality + Data Wrangler + Lambda alerts + CloudWatch | [Task 1.3](domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md) |
| 45 | Invocation logging to S3 + Prompt Management parameterised templates, versions, approval | [Task 1.6](domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) |
| 46 | Custom Model Import eviction: scheduled keep-warm + exponential backoff | [Task 5.2](domain-5-testing-validation-troubleshooting/02-task-5.2-troubleshooting.md) · [Task 1.2](domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md) |
| 47 | ApplyGuardrail pre/post checks + streaming with guardrail + SQS review + IAM GuardrailIdentifier | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 48 | Context overflow: map-reduce chunking + KB RAG (the context window is fixed) | [Task 5.2](domain-5-testing-validation-troubleshooting/02-task-5.2-troubleshooting.md) |
| 49 | Strands supervisor with agents-as-tools on AgentCore + Observability | [Task 2.1](domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md) |
| 50 | Lambda response streaming + 30 s timeout + S3 10-year retention | [Task 2.4](domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md) |
| 51 | Structured RDS data → text-to-SQL + SQL validation + confidence scoring and semantic similarity | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 52 | SageMaker real-time + Neo (hybrid/edge) + Bedrock Provisioned Throughput (batch) | [Task 2.2](domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md) |
| 53 | Lake Formation per account + registered S3 + LF-TBAC columns + cross-account roles | [Task 3.2](domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md) |
| 54 | Semantic-boundary chunking + re-embed (not more dimensions) | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) |
| 55 | Comprehend custom entity recogniser + DynamoDB session state (encrypted, TTL) | [Task 1.3](domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md) |
| 56 | Step Functions circuit breaker + cross-Region inference | [Task 2.4](domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md) |
| 57 | SQS + Fargate model cascade (second queue for low-confidence requests) | [Task 4.1](domain-4-operational-efficiency-optimization/01-task-4.1-cost-optimization-resource-efficiency.md) |
| 58 | API Gateway usage plans per tier + method-level throttling + API keys | [Task 2.4](domain-2-implementation-integration/04-task-2.4-fm-api-integrations.md) |
| 59 | OpenSearch hybrid + Bedrock reranker (legal) | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) |
| 60 | Comprehend toxicity + prompt safety + DetectPiiEntities in parallel | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 61 | Strands SDK: Bedrock provider + session managers + native MCP + supervisor pattern | [Task 2.1](domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md) |
| 62 | Guardrail on **responses** (HIGH, BLOCK) in the flow + lower temperature | [Task 1.6](domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) |
| 63 | One guardrail: MEDIUM content filters + denied topic + mask PII out / block financial in | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 64 | KB on OpenSearch Serverless + web crawler + Guardrails PII | [Task 1.4](domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md) |
| 65 | Zero-shot CoT with fact checks + RAG with semantic chunking | [Task 1.6](domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md) |
| 66 | Bedrock evaluation jobs + CodePipeline quality gate | [Task 2.3](domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md) · [Task 5.1](domain-5-testing-validation-troubleshooting/01-task-5.1-evaluation-systems.md) |
| 67 | Guardrails PII + denied topic + invocation logging with image delivery to S3 | [Task 2.5](domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md) |
| 68 | Guardrails PHI masking + Bedrock Evaluations to compare FMs | [Task 1.3](domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md) |
| 69 | Step Functions: Lambda sanitise → Guardrails → KB → Lambda validate | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 70 | BDA parser + KB structured data retrieval | [Task 1.5](domain-1-fm-integration-data-compliance/05-task-1.5-retrieval-mechanisms.md) |
| 71 | Outposts (on-premises residency) + Wavelength (<100 ms at the edge) | [Task 2.3](domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md) |
| 72 | Guardrails + SCP enforcement + StackSets (⚠️ conflicts with PQ22) | [Task 3.1](domain-3-ai-safety-security-governance/01-task-3.1-input-output-safety-controls.md) |
| 73 | In-session memory via InvokeAgent `promptSessionAttributes`, cleared at session end | [Task 2.1](domain-2-implementation-integration/01-task-2.1-agentic-ai-tool-integrations.md) |
| 74 | Made-up PII in outputs → Guardrails output sensitive information filter | [Task 3.2](domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md) |
| 75 | Comprehend PII (EN/ES) redaction before the call + Guardrails on output + S3 audit logs | [Task 3.2](domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md) |

## Format of each task file

1. **Key points**: a quick summary of the task.
2. **Notes**: one section per official skill (for example, Skill 1.2.3), with services, patterns, decision tables and gotchas.
3. **Exam triggers**: a table that pairs a scenario cue with the answer it points to.
4. **Flashcards**: click-to-reveal `<details>` cards.
