[← Back to Domain 3](README.md)

# 🔐 Task 3.2 · Implement data security and privacy controls

> **Domain 3 · AI Safety, Security, and Governance (20%)**
> - **Skill 3.2.1**: Develop protected AI environments to ensure comprehensive security for FM deployments
> - **Skill 3.2.2**: Develop privacy-preserving systems to protect sensitive information during FM interactions
> - **Skill 3.2.3**: Create privacy-focused AI systems to protect user privacy while maintaining FM utility and effectiveness

## Key points
- **Network isolation**: use **interface VPC endpoints (PrivateLink)** for `bedrock-runtime`, `bedrock-agent-runtime` and the others, so traffic never crosses the internet. Attach **endpoint policies** to limit which principals, actions and models can be used through each endpoint. In IAM and S3 policies, **`aws:SourceVpce`** / **`aws:SourceVpc`** enforce "only from my VPC". SageMaker uses **VPC-only Studio**, **network isolation** (`EnableNetworkIsolation`) and **inter-container traffic encryption**.
- **Bedrock native privacy**: prompts and completions are **not used to train** AWS or third-party models and are **not shared with model providers**. Models run in **Model Deployment Accounts** owned by the Bedrock team, and **providers have no access** to them, to logs, or to prompts and completions. **TLS 1.2 is required** (1.3 recommended) and data is encrypted at rest. **Invocation logging is off by default (opt-in).**
- **KMS customer managed keys (CMKs)** can encrypt Knowledge Bases (transient ingestion data and the data sources), agents, guardrails, custom or fine-tuned models, invocation-log S3 buckets and CloudWatch log groups. Use a CMK when the scenario needs **key control, rotation, revocation or a CloudTrail audit of key use**.
- **Lake Formation** gives **fine-grained access** (database, table, column, row and cell) through **data filters** and **LF-tags (LF-TBAC)**. It's the answer when an FM pipeline or agent must see only *some* columns or rows of data-lake tables.
- **Know which tool detects which PII**:
  - **Macie** discovers PII **at rest in S3** (automated discovery plus discovery jobs, with managed and custom data identifiers).
  - 🎯 **PQ75** **Comprehend** detects PII **in text**: `DetectPiiEntities` returns offsets, `ContainsPiiEntities` returns labels only. Real-time calls take up to **100 KB** in **English or Spanish**, and **redaction needs an async job**.
  - **Guardrails sensitive info filters** act on **live prompts and responses**.
- 🎯 **PQ74** 🎯 **PQ6** **Guardrails PII actions** are `BLOCK`, `ANONYMIZE` (mask, which replaces the value with `{NAME}`, `{EMAIL}` and so on) or `NONE` (detect only). Input and output can have **separate actions**. **Custom regex** patterns can be up to 500 characters and **don't support lookaround**.
- **Guardrails masking doesn't protect three things**: (1) the **invocation logs**, where the `input` field always holds the original text, so use **CloudWatch Logs data protection**; (2) the **trace `match` field**, which shows the raw PII; (3) **tool call arguments, tool results and tool definitions**.
- **Redact before you embed.** Any PII that gets into a vector store comes back out in retrieval. Scrub it with Comprehend, the **Glue Detect PII** transform or `ApplyGuardrail` **before chunking and embedding**.
- **Retention**: **S3 Lifecycle** handles expiration and tiering of logs, transcripts and training data. **S3 Object Lock** (Governance or Compliance mode) gives WORM immutability for audit logs. Conversation history in DynamoDB expires with **TTL**.
- **Privacy techniques**:
  - **masking**: irreversible, keeps the format
  - **tokenisation**: reversible through a token vault
  - **pseudonymisation**: reversible with a key, and *still personal data under GDPR*
  - **anonymisation**: irreversible, and no longer personal data
  - **differential privacy**: statistical noise; AWS Clean Rooms supports it
- **Multi-tenant RAG**: inject the **tenant metadata filter server-side** (never trust a filter supplied by the client), or use **separate KBs or indexes per tenant** for strict isolation.

> [!IMPORTANT]
> **🎯 Practice question 75 — 20k notes a day, 100/min, sub-second pre-processing, remove PII (EN + ES) before Bedrock, block identifiers in output, audit**
>
> - **Amazon Comprehend PII detection** (real time, **English and Spanish**) → **redact** the detected spans before calling Bedrock. Call `DetectPiiEntities`, replace the returned offsets in code, and log each redaction.
> - **Bedrock Guardrails** on the **output** → block medical identifiers in the summaries.
> - **Store the redacted input and output logs in S3** → audit visibility for every redacted field.
> - ❌ **Glue DataBrew one-time masking + Lake Formation:** batch, misses each day's new notes, and LF can't filter FM output.
> - ❌ **Comprehend + a fine-tuned Bedrock model as a second detector:** extra latency, and Comprehend already covers EN and ES.
> - ❌ **Macie + WAF + DynamoDB:** Macie isn't a real-time detector, and WAF can't filter FM output.
> - 🔁 **Compare with PQ60:** there, *flag* PII (no redaction) in a real-time gate. Here, *remove* it, which you do yourself from the **real-time detection offsets**. Comprehend's built-in **redaction mode runs as an asynchronous batch job**.

> [!IMPORTANT]
> **🎯 Practice question 74 — the input is de-identified, but 3–5% of outputs still contain names and record numbers that weren't in the input**
>
> - The model is **making up** identifiers that look real. It isn't leaking them from the input, so fixing the **input** won't help.
> - → **Guardrails sensitive information filters on the *output*** (block or mask NAME, custom regex for medical record numbers, and so on). Guardrail traces and invocation logs give the **audit record of PII decisions**, and it fits the <500 ms budget.
> - ❌ **Session isolation / clearing history:** the problem isn't data carried over between sessions.
> - ❌ **Better *input* detection (regex + custom entities):** the identifiers weren't in the input.
> - ❌ **Removing medical detail from inputs:** defeats the purpose and doesn't stop made-up identifiers.
> - 🔁 **Cue:** *"PII appears that wasn't in the input"* → **filter the output**.

## Notes

### Skill 3.2.1 — Protected AI environments (network, IAM, Lake Formation, monitoring)

> *Develop protected AI environments to ensure comprehensive security for FM deployments (for example, by using VPC endpoints to isolate networks, IAM policies to enforce secure data access patterns, AWS Lake Formation to provide granular data access, CloudWatch to monitor data access).*

#### Defence layers for an FM workload

| Layer | Control | AWS mechanism |
| --- | --- | --- |
| Network | Private connectivity, no internet path | Interface VPC endpoints (PrivateLink), gateway endpoint for S3/DynamoDB, security groups, no IGW/NAT for sensitive subnets |
| Endpoint | Limit what can be done through the endpoint | VPC endpoint policies |
| Identity | Least privilege on models, KBs, agents, data | IAM identity policies, permission boundaries, SCPs, resource-based policies (guardrails, S3, KMS) |
| Data | Fine-grained data-lake access | Lake Formation permissions, data filters, LF-tags |
| Encryption | At rest + in transit, customer key control | KMS CMKs, TLS 1.2+ (Bedrock requires TLS 1.2, recommends 1.3), FIPS endpoints |
| Detection | Who accessed what, when | CloudTrail (management + data events), CloudWatch metrics/alarms/Logs, Macie, GuardDuty, Security Hub |

#### VPC endpoints (PrivateLink) for Bedrock and SageMaker
- Bedrock has **separate interface endpoints for each API family**: control plane (`bedrock`), runtime (`bedrock-runtime`), agents build-time (`bedrock-agent`) and agents runtime (`bedrock-agent-runtime`), plus newer ones. The full list and the FIPS variants are in [Task 2.3](../domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md). Create **only the endpoints your app calls**. Most apps need **`bedrock-runtime`**, and RAG or agent apps also need **`bedrock-agent-runtime`**.
- **Private DNS** is on by default, so the normal SDK hostname resolves to the endpoint's private IPs and the code doesn't change. If private DNS is off, set `endpoint_url` explicitly.
- **Bedrock still reaches some resources over the AWS network on your behalf**, for example reading S3 for a KB sync or calling an OpenSearch Serverless collection. Those resources need their **own** private access configuration:
  - OpenSearch Serverless **network policy**: VPC access, or access from AWS services
  - S3 **bucket policy**
  - KMS **key policy**
- **SageMaker**:
  - **Studio domain in `VpcOnly` mode**: all Studio traffic goes through your VPC. Add endpoints for `sagemaker.api`, `sagemaker.runtime`, S3, CloudWatch, STS and ECR.
  - **`EnableNetworkIsolation=True`** on training jobs or models: the container gets **no outbound network access at all**, not even to S3. SageMaker copies data in and out for it.
  - **Inter-container traffic encryption** encrypts traffic between nodes in distributed training.
  - **VPC config** on endpoints and jobs places the ENIs in your private subnets.

**Endpoint policy example**: only allow invoking one approved model family through this endpoint, and only from principals in your organisation:

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
    "Resource": [
      "arn:aws:bedrock:*::foundation-model/anthropic.*",
      "arn:aws:bedrock:*:111122223333:inference-profile/*"
    ],
    "Condition": { "StringEquals": { "aws:PrincipalOrgID": "o-exampleorgid" } }
  }]
}
```

> ⚠️ The Converse and ConverseStream APIs are authorised with the **`bedrock:InvokeModel` / `bedrock:InvokeModelWithResponseStream` IAM actions** (verify in docs for any new actions). If you call through a **cross-Region inference profile**, the policy must allow **both the inference-profile ARN and the foundation-model ARN in every destination Region**.

#### IAM data access patterns
- **Least privilege on models**: scope `bedrock:InvokeModel` to specific foundation-model or inference-profile ARNs, and don't use `Resource: "*"`. If a team may only use approved models, deny everything else with an SCP.
- **Force calls to come from the VPC**: deny when the request doesn't arrive through your endpoint. In an identity policy, use `aws:SourceVpce` for the specific endpoint or `aws:SourceVpc` for the VPC.

```json
{
  "Effect": "Deny",
  "Action": "bedrock:*",
  "Resource": "*",
  "Condition": { "StringNotEquals": { "aws:SourceVpce": "vpce-0abc123example" } }
}
```

> ⚠️ `aws:SourceVpce` is **only present on requests that go through a VPC endpoint**. Requests from AWS services acting on your behalf (for example, a KB sync reading S3) don't carry it, so a blanket deny can break them. Add exceptions such as `aws:ViaAWSService` or `aws:CalledVia`, or principal-based exceptions.

- **S3 data protection for training data and KB source data**:
  - Bucket policy denies unless the request comes through `aws:SourceVpce`.
  - Deny `s3:PutObject` without SSE-KMS (`s3:x-amz-server-side-encryption`).
  - Deny any request where `aws:SecureTransport = false`.
  - The KB service role gets read access only to the specific prefix.
- **Service roles** for KB, agent, customisation and batch jobs: add **confused-deputy protection** to the trust policy using `aws:SourceAccount` and `aws:SourceArn`.
- **ABAC**: tag models, KBs and agents (for example `team=claims`) and use `aws:ResourceTag` / `aws:PrincipalTag` conditions. This scales better than one policy per team. Application inference profiles also give cost allocation per tenant.
- **Agent tool access**: give action-group Lambdas their **own narrow execution roles**. The agent itself shouldn't hold broad data permissions (see [study-notes 11](<../../study-notes/Certifications/AWS/AIP-C01 - Gen AI Pro/11-security-identity-agent-security.md>)).

#### AWS Lake Formation: granular data access for GenAI pipelines 🎯 PQ20
- Lake Formation is a central permission layer over the **Glue Data Catalog** and S3 data lakes. Services that respect it include Athena, Redshift Spectrum, EMR, Glue and SageMaker.
- **Permission levels**: database, table, **column** (include or exclude), **row** (row filter expression) and **cell** (row and column together), using **data filters**.
- 🎯 **PQ53** **LF-TBAC (tag-based access control)**: put LF-tags such as `sensitivity=pii` or `domain=hr` on databases, tables and columns, then grant on tag expressions. This scales to thousands of tables.
- **GenAI use cases**:
  - **Text-to-SQL agents** or tools that query the lake through Athena. The agent's role sees only permitted columns and rows, so the FM can't leak a salary column it never receives.
  - **Building training or fine-tuning datasets** from the lake while excluding PII columns.
  - 🎯 **PQ5** **Cross-account data sharing** to an ML account through LF grants or RAM.
- **Lake Formation vs IAM/S3 policies**: S3 policies work at **object or prefix** granularity. Lake Formation understands **tables, columns and rows**. If the scenario says "only certain columns" or "only rows for region X", the answer is **Lake Formation**.

> [!WARNING]
> **🎯 Practice question 53 — app in Account C needs data from A (transactions) and B (PII), 10 accounts soon, only the needed columns**
>
> 1. **Set up Lake Formation in each source account** and **register the S3 locations** with Lake Formation, so LF takes over access control from plain S3 permissions.
> 2. Grant **column-level access with LF-tags (LF-TBAC)**, for example `pii=false` columns only. New tables and columns pick up permissions **from their tags**, which scales to new accounts.
> 3. Create **IAM roles in each source account** for the paths that are needed. The **SageMaker AI execution role in Account C assumes these cross-account roles** and reads **through Lake Formation**, so it only sees the allowed columns.
>
> Traps:
> - ❌ **IAM Identity Center permission sets + a Lambda authoriser to strip PII:** no column-level data control.
> - ❌ **S3 object tags:** **object-level only**, so you can't restrict **columns** inside an object.
> - ❌ **S3 bucket policies + identity policies "using LF-TBAC":** LF-TBAC only works once the buckets are **registered with Lake Formation** and grants go through **LF permissions**. VPC endpoints are network controls, not column filters.
> - ℹ️ Lake Formation also supports **cross-account grants** (LF-tag sharing or RAM) straight to the consumer account. Know both patterns.

> [!WARNING]
> **🎯 Practice question 20 — co-pilot agent must inherit each human rep's Lake Formation permissions; open source frameworks (choose 3)**
>
> 1. ✅ **AgentCore Python SDK + AgentCore Runtime.** Build with **any open source framework** (Strands, LangGraph, CrewAI) and host it **serverless**, with no infrastructure to run.
> 2. ✅ **AgentCore Identity on-behalf-of (impersonation) flow.** The agent reads data **using the signed-in rep's identity and permissions**, not its own broad role, so access matches the human's exactly and is audited per user.
> 3. ✅ **Cognito user pool groups → IAM roles.** Put each rep in a group for their organisational role. **Associate an IAM role with each group**, and grant **Lake Formation fine-grained permissions** (database, table, column, row) **to those IAM roles**.
>
> Traps:
> - ❌ **A workload identity with full read access:** the agent then sees more than the human, which breaks the privacy policy.
> - ❌ **Cognito groups → IAM *groups*:** IAM groups can't be assumed, and Lake Formation grants go to **IAM roles or users**. The working chain is user pool group → **IAM role**.
> - ❌ **Docker on EC2 + Auto Scaling:** you run the infrastructure yourself, which isn't least overhead.

#### Monitoring data access
- **CloudTrail management events** (on by default for 90 days in Event history) record control-plane calls such as CreateKnowledgeBase, CreateGuardrail and PutModelInvocationLoggingConfiguration. Whether runtime calls (InvokeModel, Converse, Retrieve, InvokeAgent) count as management or data events differs by API, so verify in docs. Enable **data events** for Bedrock resource types where they're offered, and for **S3 object-level access** to training and KB buckets.
- **CloudTrail Lake** or Athena over trail logs answers questions like "who called this model, from where, how often".
- **Bedrock model invocation logging** (opt-in) records the full request and response bodies plus token counts to **CloudWatch Logs and/or S3**. Bodies up to **100 KB** go inline; anything larger or binary goes to S3. This is your record of *what data was sent to the FM*. **Treat these logs as sensitive data in their own right** and protect them with a KMS CMK, tight IAM, CloudWatch Logs data protection and retention settings.
- **CloudWatch**: Bedrock runtime metrics (`Invocations`, `InvocationClientErrors`, `InputTokenCount` and others, by `ModelId`). Use **metric filters and alarms** on unusual volumes, for example a spike in invocations from one principal, which could be exfiltration through prompts.
- **CloudWatch Logs data protection policies** detect and **mask** sensitive data such as PII and credentials in log groups as it's ingested. Only principals with `logs:Unmask` see the raw values. This is the fix for PII in invocation logs.
- **Macie**: continuous discovery of sensitive data in the S3 buckets that feed RAG or training.
- **GuardDuty**: S3 protection and anomalous API detection, for example credential misuse calling Bedrock from an unusual location. Verify in docs which Bedrock-specific GuardDuty findings exist.
- **Security Hub / AWS Config**: posture checks, for example Config rules to confirm buckets are encrypted and endpoints exist.

#### Gotchas
- **A VPC endpoint isn't data protection on its own.** Without an endpoint policy and `aws:SourceVpce` conditions, any principal with credentials can still call Bedrock over the public endpoint from anywhere.
- **Network isolation breaks containers that download models at runtime** (for example, pulling from Hugging Face). Package the model artefacts in S3 or ECR instead.
- **Cross-Region inference** keeps traffic on the AWS network, but processing happens in other Regions. For residency, use **geographic profiles** (US, EU, APAC) rather than the global profile. See [Task 2.3](../domain-2-implementation-integration/03-task-2.3-enterprise-integration-architectures.md).

---

### Skill 3.2.2 — Privacy-preserving systems (PII detection, Bedrock privacy, Guardrails, retention)

> *Develop privacy-preserving systems to protect sensitive information during FM interactions (for example, by using Amazon Comprehend and Amazon Macie to detect personally identifiable information [PII], Amazon Bedrock native data privacy features, Amazon Bedrock guardrails to filter outputs, Amazon S3 Lifecycle configurations to implement data retention policies).*

#### Where PII appears in a GenAI system, and which control fits

| Location | Risk | Control |
| --- | --- | --- |
| S3 data lake / KB source docs / training data | PII embedded into vectors or learned by fine-tuned model | **Macie** discovery → quarantine/redact (Comprehend, Glue Detect PII) **before ingestion** |
| User prompt (runtime) | User pastes card numbers, SSNs | **Guardrails sensitive info filter** (input: BLOCK or ANONYMIZE), or Comprehend pre-filter |
| Retrieved context | KB chunk contains another customer's data | Redact at ingestion + **metadata access filtering** (Skill 3.2.3) |
| Model response | FM echoes/infers PII | **Guardrails output** ANONYMIZE/BLOCK; Lambda post-processing |
| Tool calls (agents) | Model passes PII into tool args | ⚠️ Guardrails **don't** inspect tool args/results → validate in the tool Lambda |
| Logs (invocation logs, app logs, traces) | Raw prompts stored | **CloudWatch Logs data protection**, KMS, retention, restricted IAM |
| Conversation history (DynamoDB) | Long-lived personal data | Redact before storing, **TTL**, KMS |
| Audio / images / scanned docs | Spoken or printed PII | **Transcribe PII redaction** for audio; Textract → Comprehend for scanned docs |

#### Amazon Bedrock native data privacy features (checked against the AWS docs)
- **Shared responsibility**: AWS secures the infrastructure. You control your content, IAM, encryption choices and logging.
- **No training on your data**: Bedrock doesn't use your prompts and completions to train AWS or third-party models, and doesn't share them with model providers.
- **Model Deployment Accounts**: there's **one per model provider in each Region**, **owned and operated by the Bedrock service team**. Bedrock makes a deep copy of the provider's inference and training software into these accounts. **Model providers have no access** to them, so they **can't see Bedrock logs or customer prompts and completions**.
- **Encryption in transit**: **TLS 1.2 required, TLS 1.3 recommended**. **FIPS 140-3 endpoints** are available for regulated workloads.
- **Encryption at rest**: on by default with AWS-owned or AWS managed keys. You can use **customer managed KMS keys** for guardrails, KBs, agents, custom models and the logging destinations.
- **Fine-tuning and customisation**: the customised model copy is **private to your account**. The base model isn't changed and your training data isn't used to improve the base model. The custom model can be encrypted with your CMK.
- **Invocation logging is opt-in**, so by default Bedrock doesn't put your prompt bodies anywhere you can see them. If you turn it on, you take on protecting those logs.
- **Abuse detection**: Bedrock runs automated abuse detection. Some models may retain data for a limited time for safety reasons; check each model's terms.
- **Data retention controls (newer feature, from 2026)**: account-level or project-level **data retention modes**. They are set **per Region**, through the API only (no console at launch).
  - The modes, from least to most permissive: `none` (zero data retention) < `default` (the model's own policy) < `aws_review` (AWS may retain and human-review the data, which never leaves the AWS boundary) < `provider_data_share` (legacy).
  - `inherit` passes the decision up the hierarchy.
  - A model that requires a higher mode returns a **`ValidationException`** on `bedrock-runtime`.
  - You can enforce the mode across an organisation with an **SCP condition on `bedrock:DataRetentionMode`**.
  - This is new, so it may not appear on the exam. If you see "guarantee zero retention of prompts", it's the answer.
- **Compliance**: Bedrock is in scope for common programmes, including HIPAA eligibility, SOC, ISO and GDPR support. Check AWS Artifact for current scope (verify in docs for specific programmes).
- Don't put sensitive data in **tags or free-text Name fields**. They can appear in billing and diagnostic logs.

#### Amazon Comprehend: PII detection in text (checked against the AWS docs)

| API | Mode | Returns | Use |
| --- | --- | --- | --- |
| `DetectPiiEntities` | Real-time, single doc ≤ **100 KB** UTF-8 | Entity `Type`, `Score`, `BeginOffset`/`EndOffset` | Locate PII → mask it yourself in Lambda |
| `ContainsPiiEntities` | Real-time | **Labels only** (entity types present + score), no offsets | Cheap yes/no gate: "does this prompt contain PII?" → route/block |
| `StartPiiEntitiesDetectionJob` | **Async batch** over S3 | Offsets *or* **redacted copies** of documents | Bulk redaction of a corpus before KB ingestion / training |

- **Languages**: **English and Spanish** for PII.
- **Redaction** of documents is done with an **async job**. Set `Mode=ONLY_REDACTION` plus a `RedactionConfig` with `MaskMode` set to `MASK` (a mask character such as `*`) or `REPLACE_WITH_PII_ENTITY_TYPE`, which inserts `[NAME]`, `[SSN]` and so on. `ONLY_OFFSETS` returns locations only. Verify in docs for the exact parameter names.
- **Entity types**:
  - Universal: `NAME`, `ADDRESS`, `EMAIL`, `PHONE`, `AGE`, `DATE_TIME`, `CREDIT_DEBIT_NUMBER`/`CVV`/`EXPIRY`, `PIN`, `PASSWORD`, `USERNAME`, `IP_ADDRESS`, `MAC_ADDRESS`, `URL`, `AWS_ACCESS_KEY`, `AWS_SECRET_KEY`, `DRIVER_ID`, `LICENSE_PLATE`, `VEHICLE_IDENTIFICATION_NUMBER`, `INTERNATIONAL_BANK_ACCOUNT_NUMBER`, `SWIFT_CODE`
  - Country-specific: US (`SSN`, `PASSPORT_NUMBER`, `BANK_ACCOUNT_NUMBER`, `BANK_ROUTING`, ITIN), UK (NHS, NINO, UTR), Canada (health number, SIN), India (Aadhaar, PAN, NREGA, voter ID)
- **Use the confidence score**: pick a threshold such as `Score >= 0.8` for redaction. A low threshold over-redacts and hurts usefulness; a high one misses PII.
- **PHI** (health data): use **Amazon Comprehend Medical `DetectPHI`**, which is tuned for clinical text.

```python
import boto3
comprehend = boto3.client("comprehend")

def redact(text: str, threshold: float = 0.8) -> str:
    ents = comprehend.detect_pii_entities(Text=text, LanguageCode="en")["Entities"]
    # Replace from the end so earlier offsets stay valid
    for e in sorted(ents, key=lambda e: e["BeginOffset"], reverse=True):
        if e["Score"] >= threshold:
            text = text[:e["BeginOffset"]] + f"[{e['Type']}]" + text[e["EndOffset"]:]
    return text
```

#### Amazon Macie: PII discovery at rest in S3
- Macie is a managed service that uses ML and pattern matching to **discover sensitive data in S3** and to assess bucket security (public access, encryption, sharing).
- **Automated sensitive data discovery** continuously samples objects across the estate and produces a sensitivity score for each bucket.
- **Sensitive data discovery jobs** are targeted scans, one-off or scheduled, of chosen buckets or prefixes.
- **Managed data identifiers** are built in (PII, financial data, credentials). **Custom data identifiers** use regex plus keywords, for example employee IDs. **Allow lists** suppress known false positives.
- Findings go to **EventBridge** and **Security Hub**, so you can automate a response: EventBridge → Lambda/Step Functions to quarantine the object, tag it, or block KB ingestion of that prefix.
- **Exam split**: **Macie = S3 at rest, finds where PII lives**. **Comprehend = arbitrary text in the pipeline or at runtime, finds and redacts.** **Guardrails = live FM prompts and responses.**

**Pattern: block PII from reaching the KB**

```
S3 landing bucket ──► Macie job / EventBridge(Object Created) ──► Step Functions
     │                                                              ├─ Comprehend PII (or Glue Detect PII for tabular)
     │                                                              ├─ redact → write to S3 "clean" bucket
     │                                                              └─ Macie/Comprehend finding → quarantine + SNS alert
     ▼
S3 clean bucket ──► Bedrock KB data source ──► StartIngestionJob ──► vector store
```

- **AWS Glue sensitive data detection (Detect PII transform)** in Glue Studio ETL finds PII in **tabular or columnar** data and can redact it, hash it or report it. Use it for structured data feeding text-to-SQL or fine-tuning datasets.

#### Bedrock Guardrails: sensitive information filters (checked against the AWS docs)
The full guardrail design (content filters, denied topics, word filters, contextual grounding, Automated Reasoning) is in [Task 3.1](01-task-3.1-input-output-safety-controls.md). For privacy, these are the parts to know:
- The filter is **probabilistic and ML-based**, so it uses context. It works better with more surrounding text and worse on single words. It detects PII in **natural language and in code** (variable names, hard-coded credentials, comments).
- **Actions**:
  - `BLOCK`: return the configured blocked message instead of the content.
  - `ANONYMIZE` (shown as "Mask" in the console): replace the value with `{NAME}`, `{EMAIL}` and so on.
  - `NONE` (detect mode): take no action but return detection info in the trace. Use it for dry runs and tuning.
- **Separate input and output behaviour** with `inputAction`/`outputAction` and `inputEnabled`/`outputEnabled`. A common setup is input `ANONYMIZE` (let users ask about their own account without the PII reaching the model) and output `BLOCK` for high-risk types such as credentials.
- **Built-in PII types** are grouped as General, Finance, IT, and country-specific (US, Canada, UK).
- **Custom regex** handles things like booking IDs or internal account numbers:
  - name: 1–100 characters
  - **pattern: 1–500 characters**
  - description: up to 1,000 characters
  - **no lookaround** support
- **Limits on masking (often tested)**:
  1. **Invocation logs**: the `input` field always holds the **original, unmasked request**. Use **CloudWatch Logs data protection** for those logs.
  2. The **trace `match` field** in `GuardrailPiiEntityFilter` contains the **raw PII value**. This is by design, but don't log the trace without protecting it.
  3. **Tool use is not evaluated**: PII in `toolUse.input`, `toolResult` or `toolSpec` passes through untouched. Validate inside the tool Lambda.
- **KMS CMK** on the guardrail, and **cross-Region inference for the guardrail** using guardrail profiles.
- **`ApplyGuardrail` API**: run the guardrail on **any text, without calling a model**. Examples: redact text before it's written to DynamoDB history, scrub documents before embedding, or apply the guardrail to output from a SageMaker or third-party model.

```python
bedrock_rt = boto3.client("bedrock-runtime")
resp = bedrock_rt.apply_guardrail(
    guardrailIdentifier="gr-abc123", guardrailVersion="3",
    source="INPUT",                       # or OUTPUT
    content=[{"text": {"text": user_text}}])
if resp["action"] == "GUARDRAIL_INTERVENED":
    safe_text = resp["outputs"][0]["text"]   # masked text (if ANONYMIZE) or blocked message
```

#### Data retention with S3 Lifecycle, Object Lock and TTL
- **S3 Lifecycle** **transitions** objects to cheaper classes (Standard-IA, Glacier Instant/Flexible Retrieval, Deep Archive) and **expires** them (deletes current and noncurrent versions, and cleans up incomplete multipart uploads). It enforces "keep prompt logs for 90 days, then delete".

```json
{
  "Rules": [{
    "ID": "invocation-logs-retention",
    "Filter": { "Prefix": "AWSLogs/bedrock/" },
    "Status": "Enabled",
    "Transitions": [{ "Days": 30, "StorageClass": "GLACIER_IR" }],
    "Expiration": { "Days": 365 },
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
  }]
}
```

- **S3 Object Lock** gives WORM storage for **audit evidence** that must not be deleted early.
  - **Governance mode**: users with special permission can override it.
  - **Compliance mode**: nobody can override it, not even root, until retention ends.
  - **Legal hold**: indefinite, until removed.
  - Object Lock and Lifecycle **pull in opposite directions**: Lifecycle can't delete a locked version before its retention ends.
- **CloudWatch Logs retention** is set per log group (1 day up to 10 years, or never expire). Pair invocation-log groups with an explicit retention period.
- **DynamoDB TTL** expires conversation history and session state automatically. Deletion isn't instant (usually within days; verify in docs).
- **Right to erasure (GDPR "right to be forgotten")**: you need to be able to **find and delete one person's data** in S3 sources, vector stores, conversation history and logs. Store a `user_id` in the metadata, delete the source document, then **re-sync the KB** so its vectors are removed too. You **can't remove data from fine-tuned model weights**, so don't fine-tune on personal data you may have to delete.

---

### Skill 3.2.3 — Privacy-focused AI systems that stay useful (masking, anonymisation, access filtering)

> *Create privacy-focused AI systems to protect user privacy while maintaining FM utility and effectiveness (for example, by using data masking techniques, Amazon Comprehend PII detection, anonymization strategies for sensitive information, Amazon Bedrock guardrails).*

#### The core trade-off
Redact too much and the FM loses the context it needs ("summarise the complaint from [NAME] about [PRODUCT] on [DATE]"). Redact too little and you leak data. The aim is to **keep the semantics while removing the identity**.

#### Privacy techniques compared

| Technique | Reversible? | Keeps format/semantics? | Still "personal data" (GDPR)? | Typical AWS implementation | Use when |
| --- | --- | --- | --- | --- | --- |
| **Redaction / masking** (`****`, `[NAME]`) | ❌ | Type label kept (`[EMAIL]`) | No (if all identifiers removed) | Comprehend redaction, Guardrails ANONYMIZE, CloudWatch Logs data protection | Logs, KB ingestion, outputs to users who shouldn't see PII |
| **Entity-type replacement** (`{NAME}`) | ❌ | ✅ model knows *a name was here* | No | Guardrails ANONYMIZE, Comprehend `REPLACE_WITH_PII_ENTITY_TYPE` | Summaries, classification where identity doesn't matter |
| **Tokenisation** (`CUST_8f3a…` ↔ real value in vault) | ✅ via vault lookup | ✅ consistent token = same entity across turns | Yes (re-identifiable) | Lambda + **DynamoDB token vault** encrypted with **KMS**; deterministic HMAC | FM must reason about "the same customer" and the answer must be **re-hydrated** with real values for authorised users |
| **Pseudonymisation** (consistent fake names "Alice → Person_17") | ✅ with key/mapping | ✅ natural-sounding | **Yes** | Lambda + mapping table, or synthetic replacements | Fine-tuning/evaluation datasets that need realistic text |
| **Generalisation / bucketing** (age 37 → 30–40, ZIP → region) | ❌ | Partially | Depends (k-anonymity) | Glue ETL, SQL | Analytics/text-to-SQL results surfaced through FM |
| **Anonymisation** (irreversible removal of all identifiers incl. quasi-identifiers) | ❌ | Reduced | **No** | Combination of the above + review | Sharing data externally, long-term retention |
| **Synthetic data** | n/a | ✅ statistical | No | Bedrock FM-generated synthetic records, SageMaker | Dev/test, training without real PII |
| **Differential privacy** (calibrated noise on aggregates) | ❌ | Aggregates only | No | **AWS Clean Rooms** differential privacy | Multi-party analytics/collaboration on sensitive data |

> **Exam cue**: "the model must still know that two mentions refer to the same person" or "restore the real values in the final answer for authorised users" → **tokenisation or pseudonymisation with a secured mapping**, not one-way masking.

#### Pattern: tokenise → FM → re-hydrate

```
User prompt ─► Lambda (Comprehend DetectPiiEntities)
                 ├─ replace each PII span with token (HMAC(value, KMS-derived key)) 
                 └─ store token→value in DynamoDB vault (KMS CMK, TTL, strict IAM)
            ─► Bedrock (sees only tokens) ─► response with tokens
            ─► Lambda: if caller authorised (IAM/Cognito claim) → swap tokens back; else leave masked
            ─► Guardrails output filter as a final backstop
```
- **Deterministic tokens** (HMAC) keep the same token for the same value across a conversation, so the model can still reason about it.
- **The vault is now the sensitive asset.** Protect it with a separate KMS key, restricted IAM (only the re-hydration Lambda can read it), TTL and CloudTrail data events.

#### Privacy-safe RAG
1. **Redact or tokenise before chunking and embedding.** Embeddings of PII can leak through retrieval, and arguably through the vectors themselves. The tools are Comprehend async redaction jobs, Glue Detect PII and `ApplyGuardrail`.
2. **Use metadata for access control, not just relevance**:
   - Tag each document with `tenant_id`, `department` and `classification` in a `.metadata.json` sidecar (see [Task 1.4](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md)).
   - **Inject the filter on the server side** from the authenticated identity (a Cognito claim or the IAM principal tag). **Never take the filter from the client request.**

```python
agent_rt.retrieve(
  knowledgeBaseId=KB_ID,
  retrievalQuery={"text": question},
  retrievalConfiguration={"vectorSearchConfiguration": {
      "numberOfResults": 5,
      "filter": {"andAll": [
          {"equals": {"key": "tenant_id", "value": claims["custom:tenant_id"]}},
          {"in": {"key": "classification", "value": allowed_levels(claims)}}
      ]}}})
```

3. **Choose the isolation model per tenant**:

| Model | Isolation | Cost/ops | Use when |
| --- | --- | --- | --- |
| Shared KB + **metadata filter** | Logical (filter bug = leak) | Lowest | Many small tenants, moderate sensitivity |
| **Index/collection per tenant** behind one app | Stronger | Medium | Regulated tenants, different retention |
| **KB (and account) per tenant** | Strongest, IAM-enforced | Highest | Strict contractual/regulatory separation |

   OpenSearch adds **fine-grained access control** with **document-level and field-level security** for self-managed RAG (verify in docs for Serverless support of the same features).
4. **Put a Guardrails output filter on the generated answer** as a backstop in case a chunk still contained PII.
5. **Minimise context**: retrieve fewer and more relevant chunks (reranking), and don't stuff whole customer records into the prompt.

#### Keeping the FM useful while protecting privacy
- **Mask with type labels** (`{NAME}`, `[DATE]`) instead of blanks, so the model keeps the sentence structure.
- **Apply masking selectively** by entity type. Mask `SSN` and `CREDIT_DEBIT_NUMBER`, but perhaps keep `DATE_TIME` for a timeline summary. Configure this per guardrail or per Comprehend entity list.
- **Tune the confidence threshold** and use Guardrails **detect mode (`NONE`)** in a pilot to measure false positives before switching to ANONYMIZE or BLOCK.
- **Asymmetric policies**: input ANONYMIZE keeps the conversation going, while output BLOCK on credentials stops leaks.
- **Use different guardrail versions for different audiences**, for example an internal analyst who is allowed to see names versus a public chatbot.
- **Evaluate the impact**: compare answer quality on masked and unmasked evaluation sets (Bedrock model evaluation, LLM-as-a-judge) so you can show that privacy controls haven't broken the task. See Domain 5 notes.
- **Fine-tuning data**: pseudonymise or generate synthetic data, document it in model cards (Task 3.3), and don't train on personal data you might have to erase.

#### Anti-patterns to spot in answer choices
- ❌ "Instruct the model in the system prompt not to reveal PII". This is probabilistic and can be overridden by prompt injection. Use a deterministic filter.
- ❌ Relying only on the output filter while raw PII sits in the vector store and the logs.
- ❌ Letting the client send the KB `tenant_id` filter.
- ❌ Fine-tuning on raw customer conversations "because the model is private to your account". You still can't erase the data, and the model may memorise and regurgitate it.
- ❌ Turning on invocation logging to S3 with no KMS, no lifecycle rule and broad read access.

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Bedrock calls must not traverse the public internet | **Interface VPC endpoint (PrivateLink)** for `bedrock-runtime` (+ `bedrock-agent-runtime` for KB/agents) |
| Only allow a specific model to be invoked through the VPC endpoint | **VPC endpoint policy** scoped to the model/inference-profile ARN |
| Deny Bedrock API calls unless they come from the corporate VPC | IAM/SCP **Deny with `aws:SourceVpce` / `aws:SourceVpc`** condition |
| SageMaker training container must have no outbound network access | **`EnableNetworkIsolation`** (+ VPC config) |
| Studio users must not reach the internet | SageMaker domain in **VpcOnly** mode + VPC endpoints |
| Agent queries data lake; must not see salary column or other regions' rows | **Lake Formation** column/row-level **data filters** (LF-tags at scale) |
| Thousands of tables, permission by sensitivity classification | **LF-TBAC (LF-tags)** |
| Find which S3 buckets feeding the KB contain PII | **Amazon Macie** (automated discovery / discovery job) |
| Detect and redact PII in text documents in bulk before ingestion | **Comprehend async PII job** (`ONLY_REDACTION`) or **Glue Detect PII** for tabular |
| Quick yes/no "does this prompt contain PII?" with no offsets needed | **Comprehend `ContainsPiiEntities`** |
| Locate PII spans in a 50 KB text in real time | **Comprehend `DetectPiiEntities`** (≤100 KB, EN/ES) |
| Clinical notes – detect PHI | **Comprehend Medical `DetectPHI`** |
| Call-centre audio contains card numbers | **Transcribe PII redaction** before passing transcript to FM |
| Mask PII in prompts/responses at runtime, replace with entity type | **Guardrails sensitive info filter – ANONYMIZE** |
| Block response if it contains AWS secret keys | Guardrails PII `AWS_SECRET_KEY` **outputAction BLOCK** |
| Detect internal booking IDs | Guardrails **custom regex** (no lookaround, ≤500 chars) |
| Measure PII hits without affecting users during pilot | Guardrails action **NONE (detect mode)** |
| PII still visible in CloudWatch invocation logs despite guardrail masking | **CloudWatch Logs data protection policy** (masking is not applied to logs) |
| PII passed as agent tool arguments not masked | Guardrails don't evaluate tool use → **validate/redact in tool Lambda** |
| Redact text without invoking an FM (e.g., before storing history) | **`ApplyGuardrail` API** |
| Concern that model providers can see prompts | **Model Deployment Accounts** – providers have no access; no training on customer data |
| Regulator requires customer-controlled, revocable encryption keys | **KMS customer managed keys** on KB, agents, guardrails, custom models, log buckets |
| Organisation must guarantee zero retention of prompts/completions | Bedrock **data retention mode `none`** enforced with SCP on `bedrock:DataRetentionMode` |
| Keep prompt logs 90 days then delete automatically | **S3 Lifecycle expiration** (+ CloudWatch Logs retention) |
| Audit evidence must be immutable for 7 years | **S3 Object Lock – Compliance mode** |
| Conversation history must expire automatically | **DynamoDB TTL** |
| FM must reason about "same customer" across turns; real values restored for authorised staff | **Tokenisation** with KMS-protected DynamoDB vault + re-hydration Lambda |
| Share aggregate insights across organisations without exposing individuals | **AWS Clean Rooms differential privacy** |
| Multi-tenant RAG – tenant A must never retrieve tenant B's chunks | **Server-side metadata filter** from auth claims; KB/index per tenant for strict isolation |
| Right to be forgotten for RAG data | Delete source doc → **re-sync KB**; avoid PII in fine-tuning data |

## Flashcards

<details>
<summary><b>Q:</b> Which Bedrock VPC endpoints does a RAG chatbot that calls RetrieveAndGenerate and Converse need?</summary>

`bedrock-agent-runtime` (Retrieve and RetrieveAndGenerate) and `bedrock-runtime` (Converse and InvokeModel). Add `bedrock-agent` and `bedrock` only if the app also calls the build-time or control-plane APIs.

*D3 · Task 3.2 · VPC endpoints*
</details>

<details>
<summary><b>Q:</b> What's the difference between a VPC endpoint policy and an <code>aws:SourceVpce</code> condition?</summary>

An endpoint policy limits what can be done *through that endpoint*. An `aws:SourceVpce` condition in an IAM, SCP or bucket policy limits *where requests may come from*. To force all traffic through the VPC you need the SourceVpce condition, because without it callers can still use the public endpoint.

*D3 · Task 3.2 · Network isolation*
</details>

<details>
<summary><b>Q:</b> What does SageMaker <code>EnableNetworkIsolation</code> do, and what breaks?</summary>

The container gets no outbound network access at all, not even to S3, and SageMaker moves the data in and out for it. Anything that downloads models or packages at runtime (for example, from Hugging Face) breaks, so package the artefacts ahead of time.

*D3 · Task 3.2 · SageMaker isolation*
</details>

<details>
<summary><b>Q:</b> Do model providers on Bedrock have access to customer prompts?</summary>

No. Models run in Model Deployment Accounts, one per provider per Region, owned and operated by the Bedrock team. Providers have no access to those accounts, the logs, or prompts and completions. Customer data isn't used to train the models.

*D3 · Task 3.2 · Bedrock privacy*
</details>

<details>
<summary><b>Q:</b> What TLS version does Bedrock require?</summary>

TLS 1.2 is required and TLS 1.3 is recommended. FIPS 140-3 endpoints exist for regulated workloads.

*D3 · Task 3.2 · Encryption in transit*
</details>

<details>
<summary><b>Q:</b> An agent queries an Athena table but must not see the <code>ssn</code> column or rows outside the EU. What should you use?</summary>

AWS Lake Formation data filters: column exclusion plus a row filter expression, granted to the agent's role. At scale, use LF-tags (LF-TBAC).

*D3 · Task 3.2 · Lake Formation*
</details>

<details>
<summary><b>Q:</b> Macie vs Comprehend vs Guardrails for PII: when do you use each?</summary>

Macie discovers PII at rest in S3 buckets. Comprehend detects and redacts PII in any text inside a pipeline, in real time or with async jobs. Guardrails sensitive information filters act on live FM prompts and responses.

*D3 · Task 3.2 · PII tool choice*
</details>

<details>
<summary><b>Q:</b> <code>DetectPiiEntities</code> vs <code>ContainsPiiEntities</code>?</summary>

`DetectPiiEntities` returns each entity's type, score and begin/end offsets, so you can mask the spans. `ContainsPiiEntities` returns only the labels of the PII types present, which suits a quick gate. Both are real-time calls, up to 100 KB, English or Spanish.

*D3 · Task 3.2 · Comprehend PII*
</details>

<details>
<summary><b>Q:</b> How do you get Comprehend to produce redacted copies of 50,000 documents in S3?</summary>

Run an asynchronous `StartPiiEntitiesDetectionJob` with `Mode=ONLY_REDACTION` and a `RedactionConfig`, using `MaskMode` set to `MASK` or `REPLACE_WITH_PII_ENTITY_TYPE`. Real-time APIs only locate PII; they don't redact.

*D3 · Task 3.2 · Bulk redaction*
</details>

<details>
<summary><b>Q:</b> What are the three Guardrails sensitive information actions?</summary>

`BLOCK` returns the blocked message. `ANONYMIZE` masks the value as `{NAME}`, `{EMAIL}` and so on. `NONE` is detect mode: no action, but the detection is reported. Input and output can have different actions.

*D3 · Task 3.2 · Guardrails PII*
</details>

<details>
<summary><b>Q:</b> Guardrails mask PII in responses, but auditors find raw PII in CloudWatch. Why, and what's the fix?</summary>

Guardrails masking doesn't apply to model invocation logs; the `input` field always holds the original request. Apply a CloudWatch Logs data protection policy to the log group, plus KMS and restricted access.

*D3 · Task 3.2 · Log privacy*
</details>

<details>
<summary><b>Q:</b> Does a guardrail mask PII that the model puts into tool call arguments?</summary>

No. Sensitive information filters don't evaluate `toolUse.input`, `toolResult` or tool definitions. Validate and redact inside the tool Lambda.

*D3 · Task 3.2 · Agent PII*
</details>

<details>
<summary><b>Q:</b> What are the limits on Guardrails custom regex patterns?</summary>

The pattern can be 1–500 characters and the name 1–100 characters. Lookaround isn't supported. Each regex can have its own input and output actions.

*D3 · Task 3.2 · Custom regex*
</details>

<details>
<summary><b>Q:</b> How do you redact text with a guardrail without calling an FM?</summary>

Use the `ApplyGuardrail` API with `source` set to INPUT or OUTPUT. It returns the masked text when the guardrail intervenes. This is useful before saving conversation history or embedding documents, and for models outside Bedrock.

*D3 · Task 3.2 · ApplyGuardrail*
</details>

<details>
<summary><b>Q:</b> Why redact PII before ingesting into a Knowledge Base, rather than only filtering outputs?</summary>

Whatever is embedded can come back out through retrieval to any user who can query the KB. Output filters are probabilistic backstops. Redacting at the source removes the risk and keeps vectors and logs clean.

*D3 · Task 3.2 · Privacy-safe RAG*
</details>

<details>
<summary><b>Q:</b> In multi-tenant RAG, how should you apply the tenant filter?</summary>

On the server side. Take `tenant_id` from the authenticated identity (a Cognito claim or principal tag) and inject it into the Retrieve metadata filter. Never accept it from the client. For strict isolation, use a separate KB or index per tenant.

*D3 · Task 3.2 · Multi-tenant*
</details>

<details>
<summary><b>Q:</b> Masking vs tokenisation: which lets the final answer show real values to authorised users?</summary>

Tokenisation. Tokens map back to the real values through a secure vault (for example, DynamoDB plus KMS), so a re-hydration step can restore them for authorised callers. Masking is one-way.

*D3 · Task 3.2 · Tokenisation*
</details>

<details>
<summary><b>Q:</b> Is pseudonymised data still personal data under GDPR?</summary>

Yes. It can be re-identified with the key or mapping. Only properly anonymised data, where re-identification isn't reasonably possible, falls outside GDPR.

*D3 · Task 3.2 · Anonymisation*
</details>

<details>
<summary><b>Q:</b> Which AWS service offers differential privacy for collaborative analytics?</summary>

AWS Clean Rooms, through its differential privacy feature.

*D3 · Task 3.2 · Differential privacy*
</details>

<details>
<summary><b>Q:</b> Prompt logs in S3 must be kept for 1 year and then deleted, and audit records must be immutable for 7 years. What should you use?</summary>

An S3 Lifecycle expiration rule for the logs (optionally with transitions to Glacier). S3 Object Lock in Compliance mode for the audit records.

*D3 · Task 3.2 · Retention*
</details>

<details>
<summary><b>Q:</b> How can an organisation ensure Bedrock never retains prompts or completions?</summary>

Set the Bedrock data retention mode to `none` for each Region and enforce it with an SCP that denies `PutAccountDataRetention` unless `bedrock:DataRetentionMode` is `none`. Models that require retention then become unavailable and return a `ValidationException`. This is a newer feature.

*D3 · Task 3.2 · Data retention*
</details>

<details>
<summary><b>Q:</b> Why is "tell the model not to reveal PII in the system prompt" a weak control?</summary>

It's probabilistic and can be bypassed with prompt injection or jailbreaks. Use deterministic controls instead: redaction at ingestion, Guardrails sensitive information filters, post-processing validation and IAM data scoping.

*D3 · Task 3.2 · Anti-patterns*
</details>

<details>
<summary><b>Q:</b> A user asks to have their data deleted, and it was used for RAG and fine-tuning. What's the problem?</summary>

For RAG, you delete the source documents and re-sync the KB to remove the vectors. A fine-tuned model can't have specific data removed from its weights; you'd have to retrain without that data. So avoid fine-tuning on personal data.

*D3 · Task 3.2 · Right to erasure*
</details>

<details>
<summary><b>Q:</b> Which Bedrock resources can you encrypt with a customer managed KMS key?</summary>

Guardrails, Knowledge Bases (transient ingestion data and data sources), agents, custom or fine-tuned models, and the S3 buckets and CloudWatch log groups used for invocation logging. Batch and customisation job outputs in S3 can be encrypted too.

*D3 · Task 3.2 · KMS*
</details>

---
[← Back to Domain 3](README.md)
