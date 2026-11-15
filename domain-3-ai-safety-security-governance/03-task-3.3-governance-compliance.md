[← Back to Domain 3](README.md)

# 📜 Task 3.3 · Implement AI governance and compliance mechanisms

> **Exam domain:** D3 · AI Safety, Security, and Governance (20%)
>
> - **Skill 3.3.1**: Develop compliance frameworks to ensure regulatory compliance for FM deployments.
> - **Skill 3.3.2**: Implement data source tracking to maintain traceability in GenAI applications.
> - **Skill 3.3.3**: Create organizational governance systems to ensure consistent oversight of FM implementations.
> - **Skill 3.3.4**: Implement continuous monitoring and advanced governance controls to support safety audits and regulatory readiness.

## Key points

- **Document the model → SageMaker Model Cards.** They record intended uses, a **risk rating** (`Unknown` / `Low` / `Medium` / `High`), training details, evaluation results, ethical considerations and caveats. **Every edit except an approval-status change creates a new immutable version.** You can export to **PDF**, share **cross-account**, and the cards are **integrated with Model Registry** model versions.
- **Track where data came from → Glue Data Catalog** (crawlers register sources, and tables carry metadata and tags), plus **lineage**: SageMaker ML Lineage Tracking for training artifacts, and Amazon DataZone / SageMaker Catalog OpenLineage-based lineage for data pipelines.
- **Source attribution in answers → metadata on documents** (S3 object metadata or a KB `.metadata.json` sidecar with `source`, `owner`, `classification` and `effective_date`), plus **KB citations** returned by `RetrieveAndGenerate`.
- **Who called what → CloudTrail.** Bedrock `InvokeModel`, `InvokeModelWithResponseStream`, `Converse` and `ConverseStream` are **management events**, so they're logged by default. `InvokeAgent`, `Retrieve`/`RetrieveAndGenerate`, `InvokeFlow`, `RenderPrompt` and `ApplyGuardrail` are **data events**, which you must turn on with advanced event selectors (and pay extra for).
- **What was said → model invocation logging** (off by default) to CloudWatch Logs and/or S3, which captures the full prompt and response. CloudTrail **does not** contain prompt or response bodies.
- **Configuration compliance → AWS Config** rules and conformance packs. **Audit evidence → Audit Manager "AWS Generative AI Best Practices Framework v2"**, which covers Bedrock and SageMaker AI and has **8 principles**, **72 automated + 38 manual controls** and **8 control sets**. Audit Manager is **closed to new customers**.
- **Organisation-wide guardrails → Organizations SCPs** (for example, deny `bedrock:InvokeModel` except for approved model ARNs, or deny outside approved Regions), **Control Tower** controls, and **Bedrock organisation-level enforced guardrails**.
- **Compliance reports (SOC, ISO, HIPAA eligibility) → AWS Artifact.** Bedrock is in scope for ISO/IEC 42001 (AI management system) certification. Verify in docs for the current scope list.
- **Bias drift in production → SageMaker Clarify + Model Monitor** (bias drift monitor). **Data and quality drift → Model Monitor.** For FMs on Bedrock, run **scheduled evaluations** (Bedrock Evaluations or LLM-as-a-judge) and track guardrail intervention rates in CloudWatch.
- **Automated remediation:** CloudWatch alarm, EventBridge or Config non-compliance → **EventBridge → Lambda / SSM Automation**. Examples: revoke a role, re-attach a guardrail, quarantine an S3 prefix, notify through SNS.
- **Redaction before storage:** use guardrail PII **masking** (`ANONYMIZE`), CloudWatch Logs **data protection policies** (mask PII in log groups), or Comprehend `DetectPiiEntities` in a Lambda before logging. Keep the raw text only in a locked-down, KMS-encrypted bucket if regulation requires it.
- **Regulations at awareness level:** EU AI Act (**risk tiers**: unacceptable / high / limited / minimal, plus general-purpose AI obligations), GDPR (lawful basis, right to erasure, data residency), HIPAA (BAA plus HIPAA-eligible services), **ISO/IEC 42001**, and the **NIST AI RMF** (Govern / Map / Measure / Manage).

## Notes

### Skill 3.3.1 — Compliance frameworks for FM deployments

> *Develop compliance frameworks to ensure regulatory compliance for FM deployments (for example, by using SageMaker AI to develop programmatic model cards, AWS Glue to automatically track data lineage, metadata tagging for systematic data source attribution, CloudWatch Logs to collect comprehensive decision logs).*

#### What a "compliance framework" means in practice

A compliance framework for GenAI combines four kinds of evidence:

| Pillar | Question an auditor asks | AWS mechanism |
| --- | --- | --- |
| **Model documentation** | What is this model for, what are its risks, how was it evaluated? | SageMaker **Model Cards**, Model Registry, Bedrock model evaluation reports |
| 🎯 **PQ6** **Data provenance** | Which data trained or grounded this model, and was it approved? | **Glue Data Catalog**, lineage (SageMaker Lineage, DataZone/SageMaker Catalog), S3 tags and metadata |
| 🎯 **PQ6** **Decision logging** | What did the system say to whom, and why? | Bedrock **model invocation logging**, agent traces, **CloudWatch Logs**, CloudTrail |
| **Control evidence** | Are the required controls in place and working? | **AWS Config**, **Audit Manager**, Security Hub, SCPs, guardrails |

> [!IMPORTANT]
> **🎯 Practice question 6 — end-to-end lineage + real-time PII filtering + audit trail + automatic compliance reports**
>
> - **AWS Glue Data Catalog** → registers every data source and **tracks lineage**.
> - **Bedrock Guardrails sensitive information filters** → **real-time** PII blocking or masking **during generation**.
> - **CloudTrail** (delivered to S3) → **audit trail** of every Bedrock API call.
> - **Amazon Macie** → scans stored S3 data for sensitive information and sends findings to **CloudWatch**, where **dashboards** produce the automatic compliance reports.
> - ❌ **DataSync** = replication, not lineage.
> - ❌ **Session Manager** = shell sessions, not API audit.
> - ❌ **AWS WAF** = filters web requests, not FM output.
> - ❌ **AWS Config** = resource configuration changes, not data lineage.
> - ❌ **Scheduled Comprehend Medical or Textract** = batch after the fact, not real time.
> - ❌ **Rekognition** = images, not text PII.

#### SageMaker Model Cards (in depth)

- **Purpose:** a single, structured, auditable record of an ML or FM model. It supports audit, communicates intended use and supports responsible AI.
- **Sections in the JSON schema:**
  - `model_overview` (description, creator, owner, artifact locations, algorithm and problem type)
  - `intended_uses` (purpose, intended uses, factors affecting efficiency, `risk_rating`, explanation of the risk rating)
  - `business_details` (business problem, stakeholders, line of business)
  - `training_details` (objective function, training job ARN, datasets, container image, metrics, hyperparameters)
  - `evaluation_details` (named evaluations, evaluation job ARN, datasets, metric groups: simple, bar chart, linear graph, matrix)
  - `additional_information` (`ethical_considerations`, `caveats_and_recommendations`, `custom_details`)
- **Risk rating** enum: `High`, `Medium`, `Low`, `Unknown`. Use it to gate production. For example, an approval workflow can require extra sign-off for `High`.
- **Evaluation metrics can be auto-parsed:** upload **Clarify** or **Model Monitor** JSON reports to S3 and give their S3 URI.
- **Versioning and immutability:** any edit **other than an approval-status update** creates a new model card version, which gives an immutable history of model changes. This is a strong audit trail.
- **Status lifecycle** (`ModelCardStatus`): `Draft` → `PendingReview` → `Approved` → `Archived`.
- **Export:** `CreateModelCardExportJob` produces a **PDF** in S3 that you can share with regulators or risk committees.
- **Cross-account support:** a central governance account can view and manage cards from workload accounts, using AWS RAM sharing. Verify in docs for the exact mechanism.
- **Model Registry integration:** when you register a model version, you can attach the model card or its audit details to the model package. That links what was deployed with what was documented.
- **For Bedrock FMs:** Model Cards were designed for SageMaker models, but you can still create a card for a Bedrock-hosted FM or a fine-tuned custom model. Record the provider, model ID and version, evaluation job ARNs (Bedrock Evaluations), guardrail ID, and known limitations, and link the provider's own **AWS AI Service Card** or model card. AWS publishes **AI Service Cards** for its own services and models, such as Amazon Nova, Titan, Rekognition and Textract.

```python
import boto3, json
sm = boto3.client("sagemaker")

content = {
  "model_overview": {"model_description": "Claims summarisation (Claude via Bedrock, fine-tuned prompt)",
                     "model_owner": "claims-ml-team", "problem_type": "Text summarisation"},
  "intended_uses": {"purpose_of_model": "Summarise insurance claim notes for adjusters",
                    "intended_uses": "Internal adjusters only; not for automated claim decisions",
                    "risk_rating": "High",
                    "explanations_for_risk_rating": "Output informs financial decisions about customers"},
  "evaluation_details": [{"name": "bedrock-eval-2026-11",
                          "evaluation_job_arn": "arn:aws:bedrock:us-east-1:111122223333:evaluation-job/abc123",
                          "evaluation_observation": "Faithfulness 0.93; toxicity 0.00"}],
  "additional_information": {"ethical_considerations": "May under-represent non-English notes",
                             "caveats_and_recommendations": "Human review required before customer contact"}
}

sm.create_model_card(
    ModelCardName="claims-summariser",
    Content=json.dumps(content),
    ModelCardStatus="Draft",
    Tags=[{"Key": "data-classification", "Value": "confidential"}],
)

# Later: move to PendingReview → Approved (status change doesn't create a new version)
sm.update_model_card(ModelCardName="claims-summariser", ModelCardStatus="PendingReview")

# Export PDF for the risk committee
sm.create_model_card_export_job(
    ModelCardName="claims-summariser",
    ModelCardExportJobName="claims-summariser-q4",
    OutputConfig={"S3OutputPath": "s3://governance-artifacts/model-cards/"},
)
```

> 💡 **Programmatic model cards** means generating cards automatically in the pipeline, for example in a CodePipeline or SageMaker Pipelines step after evaluation, rather than writing them by hand. Each retrain or prompt change produces a new card version holding the new evaluation metrics.

#### Glue for automatic data lineage

- **Glue crawlers** scan S3, JDBC and DynamoDB sources and register their **schemas, partitions and table properties** in the **Glue Data Catalog**. That gives every source used for RAG or fine-tuning a catalog entry.
- **Glue ETL jobs** that read catalog tables and write outputs create a traceable path from source to transformed data. Lineage across jobs is visualised with **Amazon DataZone / SageMaker Catalog data lineage**, which is OpenLineage-compatible and can capture lineage automatically from Glue and Redshift. Verify in docs for which sources are captured automatically.
- **Glue Data Quality** results can be attached as evidence that the data met its quality bar before it was used (see [Task 1.3](../domain-1-fm-integration-data-compliance/03-task-1.3-data-validation-processing.md)).
- **SageMaker ML Lineage Tracking** covers the ML side. It records **entities** (artifacts, actions, contexts, trial components) and **associations** between datasets → training jobs → model packages → endpoints. SageMaker Pipelines creates lineage automatically. Query it with `LineageQuery` to answer "which dataset produced this endpoint's model?"

#### Metadata tagging for data source attribution 🎯 PQ15

- **Resource tags** (S3 buckets and objects, KBs, models, guardrails): use `data-owner`, `data-classification`, `source-system`, `retention` and `regulatory-scope` (for example `gdpr` or `hipaa`). Tags enable **ABAC**, **cost allocation** and Config rules such as `required-tags`.
- **Document-level metadata** for RAG: a KB `.metadata.json` sidecar or S3 object metadata (`x-amz-meta-source`, `x-amz-meta-owner`, `x-amz-meta-effective-date`). This metadata is returned with retrieved chunks, so the app can show the **source, version and date** next to each claim. See [Task 1.4 · metadata frameworks](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md).

```json
{
  "metadataAttributes": {
    "source_system": "policy-portal",
    "document_owner": "legal@example.com",
    "classification": "internal",
    "effective_date": "2026-07-01",
    "regulatory_scope": "gdpr"
  }
}
```

#### CloudWatch Logs as decision logs

- Turn on **Bedrock model invocation logging**. It is **off by default**, and it is set per account per Region. Deliver to:
  - **CloudWatch Logs** for search, metric filters, Logs Insights and alarms.
  - **S3** for long-term, cheap, immutable retention. Combine with **S3 Object Lock** (compliance mode) for WORM retention, and lifecycle rules to move logs to Glacier.
- Each record holds `requestId`, `timestamp`, `accountId`, `identity.arn`, `modelId`, `input`/`output` bodies (inline up to about 100 KB, larger bodies go to S3), and token counts.
- **App-level decision logs** add what Bedrock doesn't know: the user ID, session ID, retrieved chunk IDs and sources, the guardrail action, the prompt template version, the agent trace, and the final user-visible answer. Use **structured JSON** and a correlation ID that matches the Bedrock `requestId`.
- **Retention:** set a log group retention policy, encrypt with a **KMS CMK**, and restrict access through a resource policy. Mask PII with a **CloudWatch Logs data protection policy** (see Skill 3.3.4).

```
# Logs Insights: high-risk decisions in the last 7 days (app decision log)
fields @timestamp, userId, promptVersion, guardrailAction, modelId
| filter riskTier = "HIGH" and guardrailAction = "INTERVENED"
| stats count(*) as interventions by modelId, promptVersion
| sort interventions desc
```

#### Regulatory awareness (what the exam expects)

| Framework | What to remember | Typical AWS mapping |
| --- | --- | --- |
| **EU AI Act** | Risk-based tiers: **unacceptable** (banned), **high-risk** (conformity assessment, risk management, logging, human oversight, technical documentation), **limited** (transparency: tell users they're talking to AI or that content is AI-generated), **minimal**. Extra obligations for **general-purpose AI** models. | Model Cards (technical documentation), invocation logs (record keeping), human-in-the-loop with A2I or Step Functions, transparency notices in the UI, watermarking (Titan/Nova image watermark) |
| **GDPR** | Lawful basis, data minimisation, purpose limitation, **right to erasure**, data subject access requests, **cross-border transfers** | Region pinning and **geographic CRIS** profiles, SCP Region deny, PII masking, KB re-sync after deletion, S3 lifecycle |
| **HIPAA** | PHI requires a **BAA** and **HIPAA-eligible services** | Bedrock is HIPAA-eligible; use encryption, access logging and minimum necessary access |
| **ISO/IEC 42001** | AI management system standard (the "ISO 27001 for AI") | AWS has 42001 certification for some services (check AWS Artifact) |
| **NIST AI RMF** | Four functions: **Govern, Map, Measure, Manage**, plus a Generative AI Profile (NIST AI 600-1) | The Well-Architected GenAI Lens and the Audit Manager framework map to these |
| **SOC 2 / PCI** | Standard controls; reports come from AWS **Artifact** | Artifact, Config conformance packs |

> ⚠️ **Shared responsibility:** AWS's certifications cover the **service**. You are responsible for **your use** of it: data classification, prompts, outputs, access and human oversight.

---

### Skill 3.3.2 — Data source tracking and traceability

> *Implement data source tracking to maintain traceability in GenAI applications (for example, by using AWS Glue Data Catalog to register data sources, metadata tagging for source attribution in FM-generated content, CloudTrail for audit logging).*

#### End-to-end traceability chain

```
Source system ──► S3 raw (tags: source, owner, classification)
     │                 │ Glue crawler → Data Catalog table (+ DQ results)
     ▼                 ▼
 Glue ETL / Lambda ──► S3 curated (+ .metadata.json per doc)
                           │ KB ingestion job (StartIngestionJob → jobId)
                           ▼
                   Vector store chunks (metadata: x-amz-bedrock-kb-source-uri, custom attrs)
                           │ Retrieve / RetrieveAndGenerate
                           ▼
              Answer + citations (retrievedReferences[].location.s3Location.uri, metadata)
                           │
                           ▼
         Decision log (requestId, user, chunks, model, prompt version) + CloudTrail
```

Every hop leaves an identifier you can join on: **catalog table → S3 URI → ingestion job ID → chunk source URI → citation → requestId → CloudTrail event.**

#### Glue Data Catalog as the source register

- **Register every data source** that feeds the GenAI app, including RAG corpora, fine-tuning datasets and evaluation datasets, as databases and tables, with **table parameters** for owner, PII flag, retention and approval status.
- **Lake Formation** on top adds fine-grained permissions (column, row and cell level) and **LF-Tags** (tag-based access control). For example, only the RAG ingestion role may read `classification=internal` tables. Data access security itself is covered in [Task 3.2](02-task-3.2-data-security-privacy.md).
- **Crawlers on a schedule** detect schema changes and new partitions. Use **EventBridge** events for Glue Data Catalog state changes to trigger governance reviews when a new source appears.

#### Source attribution in FM-generated content

- **KB citations:** `RetrieveAndGenerate` returns `citations[]`. Each one has `generatedResponsePart.textResponsePart` (the span of answer text) and `retrievedReferences[]` (content, `location.s3Location.uri` or a web/Confluence/SharePoint URL, and **metadata**). Show these to users and **log them**.
- **Custom RAG with `Retrieve`:** you must carry the source metadata into the prompt (for example `[doc-17 | policy-portal | 2026-07-01]`) and instruct the model to cite IDs. Then **verify** that each cited ID was actually retrieved, which catches hallucinated citations.
- **Agents:** turn on `enableTrace` to capture which action groups and KB lookups were used. Store the trace alongside the response.
- **Content provenance for generated media:** Titan Image Generator and Nova Canvas add an **invisible watermark**, and Nova adds **C2PA content credentials**. Verify in docs for the current models. Detection is available for watermarked images.

#### CloudTrail for audit logging (precise facts)

| Event category | Bedrock operations | Default? |
| --- | --- | --- |
| **Management events** | All control-plane calls (`CreateGuardrail`, `CreateKnowledgeBase`, `CreateModelCustomizationJob`, `PutModelInvocationLoggingConfiguration`, ...) **plus** runtime `InvokeModel`, `InvokeModelWithResponseStream`, `Converse`, `ConverseStream` | ✅ Logged by default (90-day Event history; create a **trail** for long-term storage) |
| **Data events** | `InvokeAgent` (`AWS::Bedrock::AgentAlias`), `InvokeInlineAgent` (`AWS::Bedrock::InlineAgent`), `Retrieve` / `RetrieveAndGenerate` (`AWS::Bedrock::KnowledgeBase`), `InvokeFlow` (`AWS::Bedrock::FlowAlias`), `RenderPrompt` (`AWS::Bedrock::Prompt`), `ApplyGuardrail` incl. guardrail evaluations during invocation (`AWS::Bedrock::Guardrail`), `StartAsyncInvoke`/`GetAsyncInvoke`, `InvokeModelWithBidirectionalStream` | ❌ Must enable with **advanced event selectors**; **extra cost** |

- CloudTrail records **who, when, from where, which model ID and which Region**. It does **not** record prompt or response text. For content, use **invocation logging**.
- **Streaming calls** produce an extra **stream-completion event** (`eventType: AwsServiceEvent`) carrying `inputTokens`, `outputTokens` and `inferenceRegion`. When you count invocations or check SCP blocks, filter on `eventType = AwsApiCall`, otherwise you double count.
- **Cross-Region inference:** the event is recorded in the **source** Region, and `inferenceRegion` shows where inference actually ran. This is the evidence you need for data residency.
- 🎯 **PQ25** **Organisation trail** (from the management or delegated admin account) gives you all accounts and Regions in one S3 bucket. Turn on **log file integrity validation** (SHA-256 digest files) to prove the logs weren't altered. Combine with Object Lock.
- **CloudTrail Lake:** a managed event data store you query with **SQL**, with long retention (up to 10 years, depending on pricing option; verify in docs). It's useful for audit queries such as "all `DeleteGuardrail` calls in 2026". It can also ingest non-AWS events.
- **GuardDuty** analyses CloudTrail for suspicious Bedrock activity, for example a new-location login followed by guardrail removal or a change to the training-data bucket.

> [!IMPORTANT]
> **🎯 Practice question 25 — central auto-discovered catalog + queryable citations per summary + tamper-evident, immutable I/O logs**
>
> - **Glue Data Catalog + crawlers** → a central catalog that **discovers and updates** data sources automatically.
> - **Summaries in S3 with object tags** (`source_id`, citations) → per-object metadata you can query.
> - **Bedrock model invocation logging → S3 with Object Lock** → **immutable** input/output records for every invocation.
> - **CloudTrail with log file integrity validation** → a **tamper-evident** audit trail of every Bedrock API call.
> - ❌ **DynamoDB item-level tags:** they don't exist. Tags apply to **tables and indexes only**.
> - ❌ **CloudWatch Logs alone:** not immutable or tamper-evident.
> - ❌ **Comprehend as a catalog, or CloudWatch metrics instead of per-call logs:** no catalog and no per-invocation I/O.
> - ❌ **AppConfig flags + a mapping file:** not a catalog and not an audit log.

```bash
aws cloudtrail put-event-selectors --trail-name org-trail --advanced-event-selectors '[
 {"Name":"KB retrievals","FieldSelectors":[
   {"Field":"eventCategory","Equals":["Data"]},
   {"Field":"resources.type","Equals":["AWS::Bedrock::KnowledgeBase"]}]},
 {"Name":"Guardrail evaluations","FieldSelectors":[
   {"Field":"eventCategory","Equals":["Data"]},
   {"Field":"resources.type","Equals":["AWS::Bedrock::Guardrail"]}]}
]'
```

```sql
-- CloudTrail Lake: who invoked which models last month (excluding stream-completion events)
SELECT userIdentity.arn, json_extract_scalar(requestParameters,'$.modelId') AS model, count(*) AS calls
FROM <event_data_store_id>
WHERE eventSource = 'bedrock.amazonaws.com'
  AND eventName IN ('InvokeModel','Converse','ConverseStream','InvokeModelWithResponseStream')
  AND eventType = 'AwsApiCall'
  AND eventTime > '2026-10-01 00:00:00'
GROUP BY 1,2 ORDER BY calls DESC;
```

---

### Skill 3.3.3 — Organisational governance systems

> *Create organizational governance systems to ensure consistent oversight of FM implementations (for example, by using comprehensive frameworks that align with organizational policies, regulatory requirements, and responsible AI principles).*

#### Governance operating model

| Element | What it is | Example |
| --- | --- | --- |
| **AI governance board / council** | Cross-functional body: legal, risk, security, data, business, ML | Approves high-risk use cases and model onboarding |
| **AI use-case intake & risk tiering** | Every GenAI use case is registered and rated (matching the EU AI Act tiers or an internal scale) | Low: internal drafting · High: customer-facing credit advice |
| **Policies & standards** | Acceptable use, approved models list, data classification rules, prompt and output standards, human oversight rules | "PII may not be sent to non-approved models" |
| **RACI** | Who is Responsible, Accountable, Consulted, Informed | Model owner (A), ML team (R), Legal (C), Users (I) |
| **Model inventory** | Single register of models, versions, owners and risk ratings | Model Registry + Model Cards + tags; SageMaker Model Dashboard |
| **Review cadence** | Periodic re-assessment, incident review, sunset decisions | Quarterly evaluation re-run, annual card re-approval |

Responsible AI dimensions that the policies should cover (AWS's list): **fairness, explainability, privacy & security, safety, controllability, veracity & robustness, governance, transparency**. The Well-Architected **Generative AI Lens** and the Audit Manager framework (below) make these operational.

#### Enforcing policy centrally on AWS (multi-account)

- **AWS Organizations + SCPs:** preventive guardrails that no account admin can override.
  - **Allow-list approved FMs** (deny `bedrock:InvokeModel*` / `Converse*` unless the resource is in the approved list).
  - **Region restriction** for data residency. Remember that CRIS needs the destination Regions allowed too. For global CRIS profiles, use `aws:RequestedRegion` = `unspecified` (see [Task 2.3](../domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md)).
  - **Deny disabling governance:** deny `bedrock:DeleteModelInvocationLoggingConfiguration`, `bedrock:DeleteGuardrail`, `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail`, except for a break-glass role.
  - **Deny model customisation** outside the ML accounts.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonApprovedFoundationModels",
      "Effect": "Deny",
      "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      "NotResource": [
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*",
        "arn:aws:bedrock:*::foundation-model/amazon.nova-*",
        "arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0",
        "arn:aws:bedrock:*:*:inference-profile/*",
        "arn:aws:bedrock:*:*:application-inference-profile/*"
      ]
    },
    {
      "Sid": "ProtectGovernanceControls",
      "Effect": "Deny",
      "Action": [
        "bedrock:DeleteModelInvocationLoggingConfiguration",
        "bedrock:DeleteGuardrail",
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*",
      "Condition": {"ArnNotLike": {"aws:PrincipalArn": "arn:aws:iam::*:role/GovernanceBreakGlass"}}
    }
  ]
}
```

> ⚠️ `Converse` and `ConverseStream` are authorised with the **`bedrock:InvokeModel` / `bedrock:InvokeModelWithResponseStream` IAM actions**, so the SCP above covers them too. Verify in docs. Inference profiles route to foundation-model ARNs in other Regions, so allow-lists must cover **both** the profile ARN and the underlying model ARNs.

- **Bedrock guardrail enforcement:** **organisation-level enforced guardrails**, applied automatically to model invocations across accounts. The guardrail needs a **resource-based policy**. CloudTrail shows `guardrailIdentifier: ENFORCED` when no guardrail was passed in the request but an enforced one applied. **IAM condition key `bedrock:GuardrailIdentifier`** lets you deny invocations that don't pass the approved guardrail. Guardrail detail is in [Task 3.1](01-task-3.1-input-output-safety-controls.md).
- **AWS Control Tower:** a landing zone with preventive (SCP), detective (Config) and proactive (CloudFormation hook) **controls**. It can apply governance baselines to every new account through Account Factory.
- **AWS Config:** a resource configuration history plus **managed and custom rules**. Examples: S3 buckets encrypted, logging enabled, SageMaker endpoint KMS-encrypted, SageMaker notebook without direct internet access, required tags. **Conformance packs** bundle rules, and AWS publishes sample packs. Use **custom Lambda rules** for checks such as "Bedrock invocation logging is enabled in this Region" (verify in docs whether a managed rule exists).
- **AWS Audit Manager: AWS Generative AI Best Practices Framework v2** (June 2024 upgrade; v1 is no longer supported for new assessments):
  - Covers **Amazon Bedrock and Amazon SageMaker AI**.
  - **72 automated controls, 38 manual controls, 8 control sets**, built with input from Deloitte.
  - **8 principles:** Responsible, Safe, Fair, Sustainable, Resilience, Privacy, Accuracy, Secure.
  - Evidence comes from Config, CloudTrail, Security Hub and API calls. You must **enable the Config rules** it maps to. Manual evidence can include `InvokeModel` test-prompt results, for example collected by **CloudWatch Synthetics canaries**.
  - ⚠️ **Audit Manager is no longer open to new customers.** Existing customers can continue. On the exam it's still the answer for "continuously collect evidence for a GenAI audit".
- **AWS Artifact:** download AWS compliance reports (SOC 1/2/3, ISO 27001/27017/27018/42001, PCI) and accept agreements such as the **BAA**. This is evidence about **AWS**, not about your workload.
- **Security Hub:** aggregates findings (Config, GuardDuty, Inspector, Macie) against standards. It is your single pane for posture.

#### Governance in the delivery pipeline

- **Approval gates:** a CodePipeline manual approval or SageMaker Model Registry status `PendingManualApproval` → `Approved` before deployment. Prompt changes go through **Prompt Management versions** and a review (see [Task 1.6](../domain-1-fm-integration-data-compliance/06-task-1.6-prompt-engineering-governance.md)).
- **Policy as code:** cfn-guard or Config proactive rules block infrastructure that lacks a guardrail, invocation logging or encryption.
- **Release evidence bundle:** for each release, store the model card version, evaluation report, guardrail version, prompt version and approver identity in S3. This is your audit packet.

---

### Skill 3.3.4 — Continuous monitoring and advanced governance controls

> *Implement continuous monitoring and advanced governance controls to support safety audits and regulatory readiness (for example, by using automated detection for misuse, drift, and policy violations, bias drift monitoring, automated alerting and remediation workflows, token-level redaction, response logging, AI output policy filters).*

#### What to monitor

| Risk | Signal | Where it comes from |
| --- | --- | --- |
| **Misuse / abuse** | Spike in guardrail interventions per user, prompt-attack detections, unusual token volume, calls from new IPs or Regions | Guardrail metrics (`AWS/Bedrock/Guardrails` namespace; verify in docs), invocation logs, CloudTrail, **GuardDuty** |
| **Policy violations** | Denied-topic hits, PII detections, calls to non-approved models, missing guardrail on a call | Guardrail trace, CloudTrail (`AccessDenied`), Config rules |
| **Quality drift** | Falling faithfulness, relevance or correctness scores; more 👎 feedback | Scheduled **Bedrock Evaluations** / LLM-as-a-judge, user feedback stored in DynamoDB, CloudWatch custom metrics |
| **Data drift** (SageMaker models) | Feature distribution changes | **SageMaker Model Monitor**: data quality monitor |
| **Model quality drift** | Accuracy or F1 against ground truth drops | Model Monitor: **model quality** monitor (needs ground-truth labels) |
| **Bias drift** | Pre- or post-training bias metric moves past a threshold | Model Monitor + **SageMaker Clarify**: **bias drift** monitor |
| **Explainability drift** | Feature attribution (SHAP) ranking changes | Model Monitor + Clarify: **feature attribution drift** |
| **Operational** | Latency, throttles, errors, token cost | `AWS/Bedrock` metrics (`Invocations`, `InvocationLatency`, `InvocationThrottles`, `InputTokenCount`, `OutputTokenCount`) |

#### Bias drift monitoring

- **SageMaker Clarify** computes bias metrics. Pre-training examples: **CI** (class imbalance) and **DPL** (difference in proportions of labels). Post-training examples: **DPPL** (difference in positive proportions in predicted labels), **DI** (disparate impact), **DCR** and **AD**. Clarify also computes **SHAP** feature attributions.
- **Model Monitor bias drift** runs Clarify on a schedule against **captured endpoint data** (Data Capture enabled) plus ground truth. It compares the results against a **baseline** and emits **CloudWatch metrics** and violation reports.
- **For LLMs:** Clarify's tabular metrics don't apply directly. Use **FM evaluations** instead: Bedrock or SageMaker model evaluation with built-in datasets for **toxicity, stereotyping and bias** (for example CrowS-Pairs and BOLD), or LLM-as-a-judge with fairness rubrics, run on a schedule and on every model or prompt change. Fairness testing design is covered in [Task 3.4](04-task-3.4-responsible-ai.md).

```python
# Model Monitor bias drift schedule (sketch; SageMaker Python SDK)
from sagemaker.model_monitor import ModelBiasMonitor, BiasAnalysisConfig, CronExpressionGenerator
monitor = ModelBiasMonitor(role=role, instance_count=1, instance_type="ml.m5.xlarge")
monitor.suggest_baseline(data_config=data_cfg, bias_config=bias_cfg, model_config=model_cfg)
monitor.create_monitoring_schedule(
    endpoint_input=endpoint_name,
    ground_truth_input="s3://bucket/ground-truth/",
    analysis_config=monitor.latest_baselining_job_config.analysis_config,
    schedule_cron_expression=CronExpressionGenerator.daily(),
)
```

#### Automated alerting & remediation

Pattern: **detect → route → act → record**

```
Guardrail metric / Logs metric filter / Config non-compliance / GuardDuty finding
        │
        ▼
 CloudWatch Alarm ─┐                 ┌─► SNS / Chatbot (notify owner, security)
 Config rule ──────┼─► EventBridge ──┼─► Lambda (revoke session, disable API key, tag user "suspended")
 GuardDuty ────────┘    rule         ├─► SSM Automation runbook / Config auto-remediation
                                     └─► Step Functions (human review → approve rollback)
        │
        ▼
 Ticket + evidence in S3 / Security Hub finding (audit trail)
```

```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "configRuleName": ["bedrock-invocation-logging-enabled"],
    "newEvaluationResult": {"complianceType": ["NON_COMPLIANT"]}
  }
}
```

- **Config remediation** can call an **SSM Automation document** directly, either automatically or manually, with retry settings.
- **Misuse throttling:** when a user trips N guardrail interventions in M minutes, a Lambda adds that user to a DynamoDB deny list that the API layer checks, or lowers their API Gateway **usage plan** quota.
- **Rollback:** a quality-drift alarm triggers Step Functions, which moves the AppConfig model ID or Prompt Management alias back to the previous version. The pattern is in [Task 1.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md).

#### Token-level redaction

| Where | Mechanism | Notes |
| --- | --- | --- |
| **Model input/output** | Bedrock Guardrails **sensitive information filter**: PII entity types + **custom regex**, action **`BLOCK`** or **`ANONYMIZE`** (mask, for example `{EMAIL}`) | Works on input and output. `ANONYMIZE` keeps utility. Masking applies to the response returned, and the original may still be in invocation logs, so plan for that |
| **Logs** | **CloudWatch Logs data protection policy**: managed data identifiers (emails, credit cards, SSNs, ...) + custom identifiers. Data is **masked at ingestion for display**. Only principals with `logs:Unmask` see the original | Audit findings can go to another log group, S3 or Firehose |
| **Pre-processing** | **Comprehend `DetectPiiEntities` / `ContainsPiiEntities`** in Lambda; replace spans by offset (`BeginOffset`/`EndOffset`) | Fine-grained token or entity replacement before anything is stored or sent |
| **Data at rest (S3)** | **Amazon Macie** discovers PII in buckets, for example RAG corpora | Detection only; remediate through EventBridge → Lambda |
| **Streaming** | Guardrail on streaming in **async mode** may release tokens before the check finishes (**sync mode** buffers them) | Use sync mode when redaction is mandatory |

Detailed PII and anonymisation design is in [Task 3.2](02-task-3.2-data-security-privacy.md).

```json
{
  "name": "mask-pii-in-genai-logs",
  "version": "2021-06-01",
  "statement": [
    {"sid": "audit", "dataIdentifier": ["arn:aws:dataprotection::aws:data-identifier/EmailAddress",
                                        "arn:aws:dataprotection::aws:data-identifier/CreditCardNumber"],
     "operation": {"Audit": {"FindingsDestination": {"S3": {"Bucket": "pii-audit-findings"}}}}},
    {"sid": "redact", "dataIdentifier": ["arn:aws:dataprotection::aws:data-identifier/EmailAddress",
                                         "arn:aws:dataprotection::aws:data-identifier/CreditCardNumber"],
     "operation": {"Deidentify": {"MaskConfig": {}}}}
  ]
}
```

#### Response logging for regulatory readiness

- **Log the response the user actually saw**: post-guardrail and post-redaction. Also keep a pointer to the raw model output if policy requires it, in a locked bucket with tight IAM, KMS and Object Lock.
- Include the **guardrail assessment** (`trace` → `guardrail` → `outputAssessments`), **citations**, **model ID and version**, **prompt version**, **user and session**, and **latency and tokens**.
- **Retention aligned to regulation:** S3 lifecycle (Standard → IA → Glacier → expire), and Object Lock **compliance mode** for legal hold. CloudWatch log group retention can be shorter.
- **Right to erasure vs immutable logs:** design up front. Pseudonymise user IDs in logs and keep the mapping in a separate deletable store. Verify the legal stance with counsel.

#### AI output policy filters

- **Bedrock Guardrails** on output: content filters, denied topics, word filters, PII, **contextual grounding check** (grounding and relevance thresholds), and **Automated Reasoning checks** (formal-logic validation against a policy document). Use a different `blockedOutputsMessaging` per policy.
- **`ApplyGuardrail` API:** applies the same policies to output from **any** model, including SageMaker, self-hosted or third-party models, or to text at any pipeline stage.
- **Custom post-filters:** a Lambda after generation enforces deterministic rules that guardrails can't express. Examples: "never quote a price not in the catalogue", "must include a disclaimer for financial advice", JSON Schema validation.
- **Escalate instead of block** for some categories: route to **human review** (A2I or a Step Functions `waitForTaskToken` step) and log the decision.

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Regulator wants documentation of a model's intended use, risk rating and evaluation results | **SageMaker Model Cards** (export **PDF**) |
| Need an immutable history of every change to model documentation | Model Cards: **every edit except a status change creates a new version** |
| Auto-generate model documentation on every retrain | **Programmatic model cards** (`CreateModelCard` in the pipeline) + Clarify/Model Monitor reports parsed from S3 |
| Link the deployed model version to its documentation and approval | **Model Registry** + Model Card integration, approval status |
| Central register of all data sources feeding RAG / fine-tuning | **Glue Data Catalog** (crawlers) + Lake Formation LF-Tags |
| Trace which dataset produced the model on this endpoint | **SageMaker ML Lineage Tracking** (`LineageQuery`) |
| Visualise data lineage across Glue/Redshift pipelines | **Amazon DataZone / SageMaker Catalog lineage** (OpenLineage) |
| Show users which document each part of the answer came from | KB **citations** (`retrievedReferences` + metadata) |
| Record document owner, date and classification for attribution | KB **`.metadata.json` sidecar** / S3 object metadata + resource tags |
| Who invoked which Bedrock model and when | **CloudTrail** (management event, on by default) |
| Audit `Retrieve`/`RetrieveAndGenerate`, `InvokeAgent` or guardrail evaluations | CloudTrail **data events** (advanced event selectors; extra cost) |
| Auditor needs the actual prompt and response text | **Model invocation logging** (CloudWatch Logs / S3), **not** CloudTrail |
| Counting Bedrock streaming calls from CloudTrail looks doubled | Filter on `eventType = AwsApiCall` (exclude `AwsServiceEvent` stream-completion events) |
| Prove where CRIS inference physically ran | CloudTrail `inferenceRegion` (event logged in the source Region) |
| SQL queries over years of API activity across the organisation | **CloudTrail Lake** |
| 🎯 **PQ25** Prove logs weren't tampered with | CloudTrail **log file integrity validation** + S3 **Object Lock** |
| Only approved FMs may be used in any account | **SCP** deny `bedrock:InvokeModel*` with `NotResource` allow-list |
| Guardrail must apply to every invocation across the organisation | **Organisation-level enforced guardrails** / IAM condition `bedrock:GuardrailIdentifier` |
| Continuously collect audit evidence for GenAI best practices | **Audit Manager: AWS Generative AI Best Practices Framework v2** |
| Download AWS's SOC 2 / ISO 42001 report or accept a BAA | **AWS Artifact** |
| Detect drift in resource configuration (encryption off, logging disabled) | **AWS Config** rules / conformance packs (+ auto-remediation via SSM) |
| Governance baseline for every new account | **AWS Control Tower** controls (preventive, detective, proactive) |
| Bias metric of a SageMaker model drifting in production | **Model Monitor bias drift** (SageMaker **Clarify**) |
| LLM output fairness or toxicity over time | Scheduled **Bedrock model evaluation** / LLM-as-a-judge → CloudWatch metrics |
| Suspicious Bedrock activity (guardrail deleted from a new location) | **GuardDuty** (analyses CloudTrail) |
| Auto-respond when a Config rule becomes non-compliant | **EventBridge → Lambda / SSM Automation** |
| Mask PII in CloudWatch log groups, with unmask for authorised staff only | **CloudWatch Logs data protection policy** (`logs:Unmask`) |
| Mask PII in model responses but keep the answer useful | Guardrails sensitive info filter **`ANONYMIZE`** |
| Apply output policies to a non-Bedrock model | **`ApplyGuardrail` API** |
| EU AI Act high-risk system: record keeping + human oversight | Invocation/decision logs + Model Cards + human-in-the-loop (A2I / Step Functions) |
| Data residency for GDPR with CRIS | **Geographic** (for example EU) inference profile + SCP Region restriction |

## Flashcards

<details>
<summary><b>Q:</b> Which edits to a SageMaker Model Card do NOT create a new version?</summary>

Only approval-status updates. Every other edit creates a new immutable version.

*D3 · Task 3.3 · Model Cards*
</details>

<details>
<summary><b>Q:</b> What are the four risk-rating values in a SageMaker Model Card?</summary>

High, Medium, Low and Unknown.

*D3 · Task 3.3 · Model Cards*
</details>

<details>
<summary><b>Q:</b> How do you share a model card with a regulator who has no AWS access?</summary>

Run a model card export job (`CreateModelCardExportJob`). It writes a PDF to S3, which you can share.

*D3 · Task 3.3 · Model Cards*
</details>

<details>
<summary><b>Q:</b> How can evaluation metrics be added to a Model Card automatically?</summary>

Upload the JSON reports from SageMaker Clarify or Model Monitor to S3 and give their S3 URI. The card parses the metrics.

*D3 · Task 3.3 · Model Cards*
</details>

<details>
<summary><b>Q:</b> Are Bedrock `InvokeModel` and `Converse` calls logged in CloudTrail by default?</summary>

Yes. They are management events, which CloudTrail logs by default. They show the caller, the model ID and the Region, but not the prompt or response text.

*D3 · Task 3.3 · CloudTrail*
</details>

<details>
<summary><b>Q:</b> Which Bedrock operations need CloudTrail data events turned on?</summary>

`InvokeAgent`, `InvokeInlineAgent`, `Retrieve`/`RetrieveAndGenerate`, `InvokeFlow`, `RenderPrompt`, `ApplyGuardrail` (including guardrail evaluations during invocation), async invoke and bidirectional streaming. Turn them on with advanced event selectors; data events cost extra.

*D3 · Task 3.3 · CloudTrail*
</details>

<details>
<summary><b>Q:</b> An auditor asks for the exact prompts and responses from last quarter. Where do they come from?</summary>

Bedrock model invocation logging (off by default), delivered to CloudWatch Logs and/or S3. CloudTrail does not store the content.

*D3 · Task 3.3 · Invocation logging*
</details>

<details>
<summary><b>Q:</b> Why can invocation counts from CloudTrail be double for streaming calls?</summary>

Each streaming call also produces a stream-completion event (`eventType: AwsServiceEvent`) carrying the token counts. Filter on `eventType = AwsApiCall`.

*D3 · Task 3.3 · CloudTrail*
</details>

<details>
<summary><b>Q:</b> With cross-Region inference, how do you prove where inference actually ran?</summary>

CloudTrail logs the event in the source Region. The `inferenceRegion` field shows the Region where inference ran.

*D3 · Task 3.3 · Data residency*
</details>

<details>
<summary><b>Q:</b> What is the AWS Audit Manager framework for GenAI, and what does it cover?</summary>

The AWS Generative AI Best Practices Framework v2. It covers Bedrock and SageMaker AI, with 72 automated and 38 manual controls in 8 control sets.

*D3 · Task 3.3 · Audit Manager*
</details>

<details>
<summary><b>Q:</b> Name the 8 principles in the Audit Manager GenAI best practices framework.</summary>

Responsible, Safe, Fair, Sustainable, Resilience, Privacy, Accuracy and Secure.

*D3 · Task 3.3 · Audit Manager*
</details>

<details>
<summary><b>Q:</b> How do you stop every account in the organisation from using non-approved FMs?</summary>

Use an SCP that denies `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` with a `NotResource` allow-list of approved model and inference-profile ARNs.

*D3 · Task 3.3 · SCPs*
</details>

<details>
<summary><b>Q:</b> AWS Artifact vs Audit Manager: what's the difference?</summary>

Artifact gives you AWS's own compliance reports and agreements (SOC, ISO, BAA). Audit Manager collects evidence about your own workload's controls.

*D3 · Task 3.3 · Compliance services*
</details>

<details>
<summary><b>Q:</b> Which service registers data sources and their schemas for traceability?</summary>

The AWS Glue Data Catalog, populated by Glue crawlers. Add Lake Formation LF-Tags for tag-based governance.

*D3 · Task 3.3 · Data lineage*
</details>

<details>
<summary><b>Q:</b> How do you answer "which training dataset produced the model behind this endpoint?"</summary>

Use SageMaker ML Lineage Tracking. SageMaker Pipelines creates the lineage automatically, and you query it with `LineageQuery`.

*D3 · Task 3.3 · Data lineage*
</details>

<details>
<summary><b>Q:</b> How does a RAG app show users the source of each claim?</summary>

Use KB citations from `RetrieveAndGenerate`. They map answer spans to retrieved references (S3 URI or URL plus metadata). Add document metadata such as owner and date through `.metadata.json` sidecars.

*D3 · Task 3.3 · Source attribution*
</details>

<details>
<summary><b>Q:</b> A SageMaker credit model's disparate impact is creeping up in production. What detects this?</summary>

A SageMaker Model Monitor bias drift monitor, powered by Clarify. It needs Data Capture and ground-truth labels, and emits CloudWatch metrics.

*D3 · Task 3.3 · Bias drift*
</details>

<details>
<summary><b>Q:</b> How do you monitor an LLM's fairness or toxicity over time?</summary>

Run Bedrock or SageMaker model evaluations on a schedule, using built-in toxicity and stereotyping datasets or LLM-as-a-judge. Publish the scores as CloudWatch metrics with alarms.

*D3 · Task 3.3 · Continuous monitoring*
</details>

<details>
<summary><b>Q:</b> How do you mask PII in CloudWatch logs while letting the security team see the originals?</summary>

Use a CloudWatch Logs data protection policy, which masks data at ingestion. Only principals with `logs:Unmask` can see the unmasked values.

*D3 · Task 3.3 · Redaction*
</details>

<details>
<summary><b>Q:</b> Guardrails sensitive information filter: BLOCK vs ANONYMIZE?</summary>

BLOCK rejects the whole input or output and returns the blocked message. ANONYMIZE masks the entity (for example `{EMAIL}`) and returns the rest of the text.

*D3 · Task 3.3 · Output policy filters*
</details>

<details>
<summary><b>Q:</b> How do you apply Bedrock guardrail policies to a model hosted on SageMaker?</summary>

Call the `ApplyGuardrail` API on the input and output text.

*D3 · Task 3.3 · Output policy filters*
</details>

<details>
<summary><b>Q:</b> What's the standard automated remediation pattern for a GenAI policy violation?</summary>

The detector (CloudWatch alarm, Config rule or GuardDuty) feeds an EventBridge rule. The rule calls Lambda or SSM Automation (or Step Functions for human approval) and notifies through SNS. The outcome is recorded as evidence.

*D3 · Task 3.3 · Remediation*
</details>

<details>
<summary><b>Q:</b> What are the EU AI Act risk tiers?</summary>

Unacceptable (banned), high-risk (documentation, logging, human oversight, conformity assessment), limited (transparency duties) and minimal. General-purpose AI models carry extra obligations.

*D3 · Task 3.3 · Regulations*
</details>

<details>
<summary><b>Q:</b> What are the four functions of the NIST AI RMF?</summary>

Govern, Map, Measure and Manage. NIST AI 600-1 is its Generative AI Profile.

*D3 · Task 3.3 · Regulations*
</details>

<details>
<summary><b>Q:</b> How do you prove CloudTrail logs weren't altered?</summary>

Turn on log file integrity validation (digest files), store the logs in S3 with Object Lock, and use an organisation trail.

*D3 · Task 3.3 · Audit logging*
</details>

---
[← Back to Domain 3](README.md)
