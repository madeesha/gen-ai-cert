[← Back to Domain 2](README.md)

# 🏢 Task 2.3 · Design and implement enterprise integration architectures

> **Exam domain:** D2 · Implementation and Integration (26%)
>
> - **Skill 2.3.1:** Enterprise connectivity: legacy APIs, event-driven loose coupling, data synchronisation
> - **Skill 2.3.2:** Integrated AI capabilities: API Gateway microservices, Lambda webhooks, EventBridge
> - **Skill 2.3.3:** Secure access frameworks: identity federation, RBAC for models and data, least-privilege FM access
> - **Skill 2.3.4:** Cross-environment and data residency: Outposts, Wavelength, secure hybrid routing
> - **Skill 2.3.5:** CI/CD pipelines and GenAI gateway architectures

## Key points
- **Mental model:** an FM is a slow, variable, quota-bound dependency. Call it synchronously only when a user is waiting; otherwise decouple with **SQS** (buffer), **EventBridge** (route to many consumers) or **Step Functions** (multi-step, approvals) → Skill 2.3.1.
- **Legacy systems → a facade.** The GenAI code calls a clean REST API (API Gateway + **VPC Link**, or a Lambda adapter) and never speaks SOAP or mainframe protocols → 2.3.1.
- **Keeping FM context fresh:** pick the sync tool by source: **DMS CDC** (databases), **AppFlow** (SaaS), **EventBridge Pipes** (stream → enrich → target), **S3 events → KB `StartIngestionJob`** (documents) → 2.3.1.
- **Webhooks:** verify the signature → **ack fast** → enqueue → FM work asynchronously. Many senders with different auth methods → **API Gateway + Lambda authorisers** in one place → 2.3.2.
- **Identity by audience:** employees → **IAM Identity Center**; customers → **Cognito**; workloads → IAM roles; per-user data access → **trusted identity propagation**; cross-account agents → `AssumeRole` with **session tags** → 2.3.3.
- **Least privilege:** scope invoke actions to specific **foundation-model / inference-profile ARNs**, use **ABAC on tagged application inference profiles**, require a guardrail with `bedrock:GuardrailIdentifier`, and keep traffic on **PrivateLink** with `aws:SourceVpce` → 2.3.3.
- **Residency ladder:** global CRIS (cheapest, anywhere) → geographic CRIS (stay in the geography) → in-Region on-demand + SCP (stay in one country) → self-hosted model on **Outposts** (stay on-premises; Bedrock doesn't run there) → 2.3.4.
- 🎯 **PQ71** **Edge:** **Wavelength** = 5G carrier edge (ultra-low latency to mobile devices) · **Local Zones** = metro proximity · **Outposts** = AWS racks in your data centre (residency or local processing).
- **GenAI CI/CD:** version everything (prompts, guardrails, agent aliases, model IDs), gate on **threshold-based evaluations**, and roll back by **repointing an alias** → 2.3.5.
- **GenAI gateway:** one central layer for auth, per-team quotas, model allow-list, guardrails, logging and cost attribution; enforce it by **denying direct `bedrock:Invoke*`** to everyone but the gateway role → 2.3.5.

## Notes

### Skill 2.3.1 — Enterprise connectivity solutions

> *Create enterprise connectivity solutions to seamlessly incorporate FM capabilities into existing enterprise environments (for example, by using API-based integrations with legacy systems, event-driven architectures to implement loose coupling, data synchronization patterns).*

#### Integration styles: what to use when

**Why it matters for GenAI:** FM calls are slow, variable and quota-bound. Decoupling stops one slow model from blocking upstream systems, lets you **buffer bursts** under Bedrock TPM/RPM quotas, and lets you add new AI consumers without touching producers. Pick the style by whether someone is waiting and how many consumers react.

| Style | AWS services | Use when | Watch out for |
| --- | --- | --- | --- |
| **Synchronous request/response** | API Gateway → Lambda/ECS → Bedrock | The user is waiting (chat, inline assist) | 29 s default REST integration timeout (can be raised for Regional/private APIs: *verify in docs*); FM latency; throttling |
| **Asynchronous queue** | SQS (+ DLQ) → Lambda/ECS workers | Bulk summarisation, document processing, smoothing bursts against Bedrock quotas | Visibility timeout must exceed processing time; idempotency |
| **Event-driven pub/sub** | EventBridge bus / SNS | Many consumers react to a business event (order created → summarise, classify, notify) | Event schema governance; at-least-once delivery |
| **Orchestrated workflow** | Step Functions | Multi-step, branching, human approval, retries per step | State size limit (256 KB payload → pass S3 pointers) |
| **Batch** | Bedrock batch inference (S3 JSONL in/out), Glue, EMR | Large offline jobs, no latency need | About 50% cheaper than on-demand (*verify in docs*); job latency in hours |
| **Streaming data** | Kinesis Data Streams / MSK → Lambda/Flink | Real-time enrichment (call transcripts, clickstream) | Ordering per shard or partition; back-pressure |

#### Integrating with legacy systems
- **Facade / anti-corruption layer:** expose a modern REST/JSON API in front of SOAP, mainframe or proprietary protocols. GenAI code only sees the facade.
  - **API Gateway REST API + VPC Link → NLB → on-prem or VPC service** for private integration (HTTP API VPC Link supports ALB, NLB and Cloud Map).
  - Mapping templates (VTL) in REST APIs can convert JSON ↔ XML for SOAP backends.
  - **Lambda as an adapter:** call legacy APIs, normalise the payload, then call the FM.
- **Strangler fig pattern:** route new GenAI features (for example a "summarise case" button) through the facade while the legacy core is unchanged. Migrate functionality gradually.
- **Hybrid reachability:** Lambda or ECS in a VPC → **Direct Connect / Site-to-Site VPN** → on-prem systems. Bedrock is reached privately through a **bedrock-runtime interface endpoint**, so no internet or NAT is needed.
- **Mainframe and file-based systems:** files land in **S3** (Transfer Family SFTP or DataSync) → S3 event → EventBridge → GenAI pipeline. **Amazon MQ** (ActiveMQ or RabbitMQ) bridges legacy JMS/AMQP messaging into AWS without code rewrites.
- **Agent-to-legacy:** expose legacy operations as **tools** (Bedrock Agents action groups with OpenAPI schemas, or MCP servers / AgentCore Gateway converting APIs to MCP tools). See [Task 2.1](01-task-2.1-agentic-ai-tool-integrations.md).

#### Event-driven architecture for loose coupling
- **SQS patterns**
  - Producer → SQS → Lambda (event source mapping, `batchSize`, `maximumConcurrency` to cap concurrent FM calls) → Bedrock.
  - **DLQ** + `maxReceiveCount` for poison messages (for example oversized prompts or content blocked by guardrails).
  - **FIFO** queues when the order per entity matters (per-conversation ordering through `MessageGroupId`).
  - **Max message size 256 KB** → put large documents in S3 and send a pointer (claim-check pattern).
- **EventBridge patterns**
  - Rules use content-based filtering on event fields, so you can route `detail.priority = "high"` to a premium model pipeline and everything else to a cheaper one.
  - **Input transformers** reshape the event before the target. **API destinations** call external HTTP APIs (SaaS) with auth and rate limits.
  - **Archive & replay** reprocesses past events (for example re-summarising after a prompt change).
  - **Schema registry** gives discoverable event contracts.
  - **EventBridge Pipes:** source (SQS, Kinesis, DynamoDB Streams, MSK) → filter → **enrichment** (Lambda, Step Functions, API Gateway, API destination) → target. A good fit for "enrich every DynamoDB change with an FM summary".
  - **EventBridge Scheduler** for scheduled refreshes (nightly KB sync, weekly report generation).
- **SNS:** simple fan-out to several SQS queues (SNS → SQS fan-out), mobile push, email or SMS. It filters on message attributes and lacks EventBridge's rich routing and SaaS sources.

```json
// EventBridge rule: route new support tickets from a SaaS partner bus to the summarisation queue
{
  "source": [{ "prefix": "aws.partner/zendesk.com" }],
  "detail-type": ["Support Ticket: Ticket Created"],
  "detail": { "ticket_event": { "ticket": { "priority": ["high", "urgent"] } } }
}
```

#### Data synchronisation patterns (keeping FM context current)
| Source | Pattern | Service |
| --- | --- | --- |
| Relational DB (Oracle, SQL Server, MySQL, Postgres) | **Change data capture**: full load + ongoing replication into S3, Kinesis or Aurora | **AWS DMS** (CDC) |
| Aurora/RDS → analytics | Near-real-time replication without ETL | **Zero-ETL integrations** (to Redshift; *verify supported pairs*) |
| DynamoDB | Item-level change stream | **DynamoDB Streams** → Lambda/Pipes → re-embed |
| SaaS (Salesforce, ServiceNow, Zendesk, Slack, SAP) | Scheduled, event-triggered or on-demand flows into S3/Redshift | **Amazon AppFlow** |
| File shares / NFS / SMB | Scheduled sync into S3 | **DataSync** |
| Documents in S3 | Object-created/deleted events trigger re-ingestion | S3 Event Notifications / EventBridge → Lambda → Bedrock KB **`StartIngestionJob`** (incremental: only changed docs are re-embedded) |
| SharePoint, Confluence, Salesforce, web | Native connector + sync | **Bedrock Knowledge Base data source connectors** |

- **Freshness trade-offs:** real-time CDC is more complex and expensive. Scheduled batch sync is simpler but stale between runs. Choose from the business tolerance (pricing data = minutes; policy manuals = daily).
- **Consistency:** the vector store is **eventually consistent** with the source. Store `last_updated` metadata and filter or boost on it. Handle **deletes** too (DMS and KB sync both propagate deletions).
- Detail on vector-store maintenance: [Domain 1 · Task 1.4](../domain-1-fm-integration-data-compliance/04-task-1.4-vector-store-solutions.md).

### Skill 2.3.2 — Integrated AI capabilities in existing applications 🎯 PQ38

> *Develop integrated AI capabilities to enhance existing applications with GenAI functionality (for example, by using API Gateway to implement microservice integrations, Lambda functions for webhook handlers, Amazon EventBridge to implement event-driven integrations).*

With the integration style chosen, this skill is about where GenAI plugs into an existing application: as a callable microservice, as a webhook receiver, or as an event subscriber.

#### Adding GenAI as a microservice
- Package each AI capability (summarise, classify, extract, draft reply, semantic search) as its own **microservice** behind API Gateway. Existing apps call `POST /ai/summarise` and never need the Bedrock SDK, model IDs or prompts.
- **API type:** **REST API** for request validation, usage plans, WAF, caching, VTL, private endpoints and response streaming; **HTTP API** for lowest cost and latency with a JWT authoriser (Cognito/OIDC) and simple proxy; **WebSocket API** for two-way server push of tokens. → Full comparison: [Task 2.4 · Skill 2.4.1](04-task-2.4-fm-api-integrations.md#skill-241--flexible-model-interaction-systems); streaming in [Skill 2.4.2](04-task-2.4-fm-api-integrations.md#skill-242--real-time-ai-interaction-systems--pq24); front-door choice in [Task 2.5 · Skill 2.5.1](05-task-2.5-app-integration-dev-tools.md#skill-251--fm-api-interfaces-for-genai-workloads).
- **Backends:** Lambda (spiky, short work: watch the 15-min max and API Gateway timeout), **ECS/Fargate** (long-running, streaming, MCP servers), or direct **AWS service integration** (API Gateway → Step Functions `StartExecution` / SQS `SendMessage` with no Lambda).
- **Async job API pattern:** `POST /jobs` → API Gateway → SQS/Step Functions → return `202 + jobId` → client polls `GET /jobs/{id}` (status in DynamoDB) or gets a callback via WebSocket or webhook. Use it for any FM task that may exceed the API timeout.
- **Sidecar enrichment:** existing CRUD flows emit events. GenAI services subscribe and write results back (for example an `ai_summary` column) without changing the original service's code path.

#### Webhook handlers with Lambda
- Entry options: **Lambda function URL** (simplest, `AuthType NONE` + signature validation in code, or `AWS_IAM`), **API Gateway** (WAF, throttling, custom domain), or **EventBridge API destinations / partner event sources** for supported SaaS.
- Webhook handler checklist:
  1. **Verify the signature** (HMAC with the shared secret from **Secrets Manager**), check the timestamp to block replays.
  2. **Validate the schema** and reject unknown events.
  3. **Idempotency:** dedupe on the event ID (DynamoDB conditional put, or Powertools idempotency utility).
  4. **Ack quickly** with 2xx inside the provider's timeout (often a few seconds).
  5. **Hand off** to SQS or EventBridge → worker Lambda calls Bedrock → callback into the SaaS API (for example post an AI-drafted reply to a ticket).
- Gotcha: calling the FM synchronously inside the webhook leads to timeouts, provider retries and **duplicate processing**.

```python
# Lambda webhook handler (sketch)
import hmac, hashlib, json, os, boto3
sqs = boto3.client("sqs")
def handler(event, _):
    body = event["body"]
    sig = event["headers"].get("x-signature", "")
    expected = hmac.new(os.environ["SECRET"].encode(), body.encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(sig, expected):
        return {"statusCode": 401}
    sqs.send_message(QueueUrl=os.environ["QUEUE_URL"], MessageBody=body,
                     MessageDeduplicationId=json.loads(body)["id"], MessageGroupId="tickets")  # FIFO queue
    return {"statusCode": 200}
```

> [!WARNING]
> **🎯 Practice question 38 — webhooks from several ticketing systems (different auth methods), 100 concurrent requests <500 ms, 99.9% multi-Region**
>
> - **API Gateway REST API (Regional endpoint) → Lambda → Bedrock** for sentiment and the auto-reply.
> - **Lambda authorisers** check **each ticketing system's own webhook auth** (HMAC signatures, API tokens, basic auth, JWT) **in one central place**, so the ticketing systems don't need any changes and you don't get auth failures.
> - **DynamoDB global tables** store the results across Regions for 99.9% availability. API Gateway + Lambda handle spikes of 250k tickets a day.
>
> Traps:
> - ❌ **Lambda function URLs:** support only **one auth type, `AWS_IAM` or `NONE`**. `NONE` + HMAC code in every function means scattered auth logic and a higher risk of failures.
> - ❌ **WebSocket API:** built for persistent two-way connections, not webhook intake, and it has no support for mixed auth methods.
> - ❌ **SQS + Comprehend + Lex:** doesn't use **Bedrock**, and queueing adds latency.

#### Event-driven GenAI integrations with EventBridge
- **Typical flow:** business app → `PutEvents` (custom bus) → rule → Step Functions (call Bedrock via the **optimised `bedrock:invokeModel` integration**) → write result → emit `AnalysisCompleted` event → downstream consumers.
- Step Functions has **direct Bedrock integrations** (`InvokeModel`, plus create/model-customisation tasks), so simple prompt calls need no Lambda.
- **SaaS partner event sources** (Salesforce, Zendesk, Shopify, and others) deliver events straight onto a partner event bus.
- **Cross-account buses:** a central AI platform account receives events from many business accounts (bus resource policy + rule targets).
- Use **Bedrock events in EventBridge** (for example model customisation job or batch job state changes) to trigger next steps.

### Skill 2.3.3 — Secure access frameworks

> *Create secure access frameworks to ensure appropriate security controls (for example, by using identity federation between FM services and enterprise systems, role-based access control for model and data access, least privilege API access to FMs).*

> Integration focus here. Guardrails, encryption, prompt injection and compliance depth are in Domain 3.

#### Identity federation patterns
| Who | Pattern | Service |
| --- | --- | --- |
| **Employees** (console, Bedrock Studio/Unified Studio, Q Business) | Corporate IdP (Okta, Entra ID, Ping) → **SAML 2.0** + **SCIM** provisioning → permission sets → IAM roles across accounts | **IAM Identity Center** |
| Employees, single account / legacy | SAML or OIDC IdP federated directly into IAM roles (`AssumeRoleWithSAML` / `AssumeRoleWithWebIdentity`) | IAM identity providers |
| **External customers** (web/mobile apps) | Sign-in through a **user pool** (social, SAML, OIDC) → JWT → API Gateway authoriser; or **identity pool** → temporary IAM creds (role per group) | **Amazon Cognito** |
| **Workloads/services** | IAM roles (Lambda execution role, ECS task role, EC2 instance profile, IRSA / EKS Pod Identity); on-prem servers through **IAM Roles Anywhere** (X.509) | IAM, STS |
| **End-user identity into data services** | **Trusted identity propagation**: the user's Identity Center identity flows to downstream services (Q Business, Redshift, Lake Formation, S3 Access Grants) for per-user authorisation | IAM Identity Center |
| 🎯 **PQ5** **Agents calling tools on behalf of users** | Inbound auth (OAuth/JWT) + outbound OAuth token vault | **AgentCore Identity** (see [Task 2.1](01-task-2.1-agentic-ai-tool-integrations.md)) |

- **Key principle:** application code should never hold long-term access keys. Use roles and STS temporary credentials everywhere. Bedrock supports temporary credentials and **forward access sessions** (FAS) when it calls downstream services such as S3 for KB ingestion using the caller's context.
- **Bedrock API keys** (short-term and long-term) exist for quick developer access. Prefer short-term keys or IAM roles in production (*verify current guidance in docs*).
- **Per-user data filtering in RAG:** pass user attributes (department, clearance) from the JWT or Identity Center into **KB metadata filters**, or use **Q Business ACL-aware** connectors that respect source permissions. Never rely on the prompt alone to hide data.

> [!WARNING]
> **🎯 Practice question 5 — agent queries DynamoDB across 25 → 40+ accounts as a specific user, LEAST overhead**
>
> **Pattern: cross-account roles assumed through an outbound credential provider, with session tags.**
> 1. In **each business-unit account**, create an IAM role whose **trust policy trusts the agent's execution role** (allow `sts:AssumeRole` and `sts:TagSession`). Give it **least-privilege** `dynamodb:GetItem` / `dynamodb:Query` on that unit's tables only.
> 2. The agent's **outbound credential provider** calls **`sts:AssumeRole`** on the target account's role and gets **temporary credentials** for each request.
> 3. Pass **session tags** (for example `user_id`, `business_unit`) on AssumeRole. The **end-user identity is kept**: it shows in CloudTrail and can be used in policies with `aws:PrincipalTag/...` (for example DynamoDB `dynamodb:LeadingKeys` for row-level limits).
> 4. A new account just needs **one more role**, deployed with **CloudFormation StackSets**. This scales from 25 to 40+ accounts with almost no extra work.
>
> Traps:
> - ❌ **AWS RAM can't share DynamoDB tables.** DynamoDB isn't a RAM-shareable resource type; RAM shares things like subnets, Transit Gateways and License Manager configurations. *(Checked against the RAM shareable-resources list, 2026-11-15.)*
> - ❌ **SCPs never grant permissions.** They only set the **maximum** allowed, so you still need identity or resource policies plus a cross-account trust.
> - ❌ **One agent per account + API Gateway routing** duplicates 25–40 deployments, which means high operational overhead.

#### RBAC and ABAC for model and data access
- **RBAC:** IAM roles per persona (`DataScientist` can fine-tune and evaluate; `AppRuntime` can only invoke approved models; `PlatformAdmin` manages guardrails and Provisioned Throughput).
- **ABAC:** tag resources (application inference profiles, KBs, agents, custom models) with `team=claims`, and tag principals the same. One policy then scales across teams.
- **Data layer:** **Lake Formation** (table/column/row-level, tag-based LF-TBAC), **S3 Access Grants** (map Identity Center users/groups to S3 prefixes), KMS key policies (who can decrypt training data or the KB index), OpenSearch fine-grained access control.

#### Least-privilege FM access: policy examples

```json
// Allow invoking only approved models + one application inference profile
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeApprovedFoundationModels",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      "Resource": [
        "arn:aws:bedrock:eu-west-1::foundation-model/anthropic.claude-*",
        "arn:aws:bedrock:eu-west-1:111122223333:application-inference-profile/abc123xyz"
      ]
    }
  ]
}
```
- `Converse` / `ConverseStream` are authorised by the **`bedrock:InvokeModel` / `bedrock:InvokeModelWithResponseStream`** actions. There is no separate "Converse" IAM action (*verify in docs*).
- **Inference profile gotcha:** to call a **cross-Region inference profile**, the principal needs permission on the **inference profile ARN** *and* the **foundation-model ARN in every destination Region** in that profile. SCPs must allow those Regions (geographic) or `aws:RequestedRegion = "unspecified"` (global).
- **Model access** must be enabled per account and Region (subscription / Marketplace permissions for third-party models). Deny `aws-marketplace:Subscribe` through an SCP to stop teams self-enabling unapproved models.
- 🎯 **PQ22** **Enforce guardrails:** the condition key `bedrock:GuardrailIdentifier` can require a specific guardrail on invoke calls. Bedrock also supports **account-level and organisation-level enforced guardrails** (organisation-level requires a guardrail resource-based policy).
- **ABAC example:** allow invoke only on inference profiles whose tag matches the caller's team.

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "arn:aws:bedrock:*:111122223333:application-inference-profile/*",
  "Condition": { "StringEquals": { "aws:ResourceTag/team": "${aws:PrincipalTag/team}" } }
}
```
- **Network-scoped access:** add `"Condition": {"StringEquals": {"aws:SourceVpce": "vpce-0abc..."}}` to deny Bedrock calls that don't come through your PrivateLink endpoint.

> [!WARNING]
> **🎯 Practice question 22 — many accounts: block topics and proprietary info in prompts + allow only approved models (choose 2)**
>
> - ✅ **Guardrail with a BLOCK policy (denied topics / word filters), deployed to every account with CloudFormation StackSets.** The same guardrail definition goes everywhere.
> - ✅ **Permissions boundary on each centrally managed employee IAM role** that requires `bedrock:GuardrailIdentifier` (the approved guardrail) on `InvokeModel`/`Converse`, **+ an SCP** that allows only **approved model IDs** across the organisation.
> - **Why it's split this way:** **guardrail IDs/ARNs are different in every account**, so the condition belongs in the **per-account role boundary**. The **approved model list is the same everywhere**, so it fits an **account-agnostic SCP**.
>
> Traps:
> - ❌ **SCP requires the guardrail ID:** SCPs need account-agnostic conditions, and guardrail IDs are account-specific.
> - ❌ **SCP for guardrail + boundary for models:** the logic is the wrong way round.
> - ❌ **Guardrail with a MASK policy:** masking applies only to **sensitive information filters**. It can't block denied topics.
> - ℹ️ **Newer option:** Bedrock **organisation-level enforced guardrails** (see the Guardrails notes in Task 3.1). The exam answer for this question is boundary + SCP + StackSets.

#### Private network access to Bedrock (PrivateLink)
Policies decide *who* may call Bedrock; PrivateLink decides *which path* the call takes. (Endpoint policy vs `aws:SourceVpce` and data-access monitoring: [Task 3.2 · Skill 3.2.1](../domain-3-ai-safety-security-governance/02-task-3.2-data-security-privacy.md#skill-321--protected-ai-environments-network-iam-lake-formation-monitoring).)
- Interface endpoint service names: `com.amazonaws.<region>.bedrock` (control plane), `.bedrock-runtime` (inference), `.bedrock-agent` (agent/KB build-time), `.bedrock-agent-runtime` (`InvokeAgent`, `Retrieve`, `RetrieveAndGenerate`), plus `bedrock-fips` / `bedrock-runtime-fips` in selected US/Canada/GovCloud Regions. A `bedrock-mantle` endpoint also exists (*verify purpose in docs*).
- **Private DNS enabled** → SDK calls to `bedrock-runtime.<region>.amazonaws.com` resolve to endpoint ENIs with **no code change**. Disabled → pass `endpoint_url=https://vpce-...bedrock-runtime.<region>.vpce.amazonaws.com`.
- **Endpoint policy** (resource policy on the endpoint) restricts which principals, actions and resources are allowed through it. The default is full access.
- **On-prem → Bedrock privately:** Direct Connect or VPN → VPC → interface endpoint. Use **Route 53 Resolver inbound endpoints** so on-prem DNS resolves the private names.
- Also add endpoints for the dependencies: S3 (gateway endpoint), STS, CloudWatch Logs, Secrets Manager, KMS, OpenSearch Serverless (VPC endpoint), SageMaker runtime.

### Skill 2.3.4 — Cross-environment AI solutions and data compliance

> *Develop cross-environment AI solutions to ensure data compliance across jurisdictions while enabling FM access (for example, by using AWS Outposts for on-premises data integration, AWS Wavelength to perform edge deployments, secure routing between cloud and on-premises resources).*

#### Data residency decision ladder
| Requirement | Design |
| --- | --- |
| Inference can happen anywhere; cost matters | **Global CRIS** (`global.` profile): about 10% savings, routes to any supported commercial Region |
| Processing must stay within a geography (EU, US, APAC) | **Geographic CRIS** (`eu.`, `us.`, `apac.`): routing stays inside that geography's Regions |
| Processing must stay in **one Region / country** | **In-Region on-demand** model ID (no inference profile) + SCP deny on other Regions + no CRIS profile permissions |
| Data must **never leave the premises** / sovereign | Self-host open-weight models on **AWS Outposts** (GPU instances where available: *verify in docs*), on-prem, or **Dedicated Local Zones**; integrate with cloud control planes |
| Very low latency to mobile/5G devices | **AWS Wavelength** zones (compute inside the carrier network) running small or distilled models |
| Low latency to a metro area / no local Region | **AWS Local Zones** |
| Disconnected or intermittent edge | Snowball Edge / IoT Greengrass with small on-device models |

- **CRIS facts that matter for residency (verified)** (CRIS as a resilience tool: [Task 1.2 · Skill 1.2.3](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md#skill-123--resilient-ai-systems)):
  - Data stays on the **AWS network**, encrypted in transit between Regions.
  - **CloudTrail logs the call in the source Region**. Check `additionalEventData.inferenceRegion` to see where it was processed.
  - Price is based on the **source Region**. There is no extra routing charge.
  - CRIS can route to Regions **not enabled** in your account.
  - Inference profiles do **not support Provisioned Throughput**.
- **Data at rest:** prompts and outputs are not stored by Bedrock by default. **Your** invocation logs, KB vector stores, S3 buckets and DynamoDB history stay in the Region where you create them. Place them in the compliant Region.
- **Bedrock does not run on Outposts or in Wavelength.** On-prem or edge inference means a **self-hosted model** (SageMaker-built containers, vLLM/TGI on EC2/ECS/EKS on Outposts). The cloud keeps the training, model registry and observability.
- **Hybrid split pattern:** sensitive data stays on-prem. Run **PII redaction or tokenisation locally** (Outposts/on-prem Comprehend-like models or regex) → send only de-identified text to Bedrock in-Region → re-identify the result on-prem.
- **Guardrails / AWS Organizations controls:**
  - SCP `aws:RequestedRegion` deny-list (remember global services and CRIS `unspecified`).
  - **AWS Control Tower data residency controls**.
  - Allow-list of model IDs.
  - **Config rules** to detect non-compliant resources.

```json
// SCP: only allow Bedrock in EU Regions (geographic EU CRIS stays compliant)
{
  "Effect": "Deny",
  "Action": "bedrock:*",
  "Resource": "*",
  "Condition": { "StringNotEquals": { "aws:RequestedRegion": ["eu-west-1", "eu-central-1", "eu-west-3", "eu-north-1"] } }
}
// Note: this also blocks global CRIS (RequestedRegion = "unspecified"). That is intended here.
// The list must include EVERY destination Region of the EU profile you use (verify the list per profile).
```

#### Secure routing between cloud and on-premises
| Option | Characteristics | Use |
| --- | --- | --- |
| **Site-to-Site VPN** | IPsec over the internet, quick to set up, up to about 1.25 Gbps per tunnel | Fast start, backup path |
| **Direct Connect** | Dedicated private link, consistent latency, 1/10/100 Gbps dedicated; **MACsec** on supported links; not encrypted by default → add VPN over DX or MACsec for encryption | Production hybrid, large data sync |
| **Transit Gateway** | Hub for many VPCs + on-prem attachments | Multi-account AI platform networking |
| **PrivateLink** | Service-level private access, no CIDR overlap issues | Bedrock/SageMaker endpoints; expose an on-prem-fronted service to other VPCs |
| **Client VPN / Verified Access** | User-level secure access without VPN (Verified Access) | Employees reaching internal GenAI apps |

- **Outposts:** an extension of a Region (control plane in the parent Region over the **service link**). Local Gateway connects to the on-prem network. Data on Outposts EBS/S3 on Outposts stays local.
- **Wavelength:** subnets in a Wavelength Zone, traffic through the **carrier gateway**. Suited to AR/VR, connected vehicles and real-time video inference near devices. Large FMs still run in the parent Region (hybrid: small model at the edge, escalate to the Region).

> [!IMPORTANT]
> **🎯 Practice question 71 — patient data must stay on-premises per country, <100 ms diagnostics at remote clinics, cloud FMs for analytics**
>
> - **AWS Outposts** in the countries with residency rules → **store and process patient data on-premises** using the **same AWS APIs and tools**, so development works the same in every environment.
> - **AWS Wavelength** (compute at the 5G carrier edge) near the clinics → **real-time inference under 100 ms**.
> - **Cloud FMs in the Region** analyse **aggregated or non-sensitive** data at scale.
> - ❌ **Global Accelerator + DataSync to S3 + one central endpoint:** copies the data out of the country, and too much latency.
> - ❌ **EC2 GPUs in Regions + copying data to the cloud:** breaks residency, and too much latency.
> - ❌ **Self-managed EC2/EBS per country + Systems Manager + CSV exports:** heavy operations, and no edge.
> - 🔁 **Cue:** *"data must stay on-premises"* → **Outposts**. *"ultra-low latency to mobile or 5G devices"* → **Wavelength**. *"metro proximity"* → **Local Zones**.

### Skill 2.3.5 — CI/CD pipelines and GenAI gateway architectures

> *Implement CI/CD pipelines and GenAI gateway architectures to implement secure and compliant consumption patterns in enterprise environments (for example, by using AWS CodePipeline, AWS CodeBuild, automated testing frameworks for continuous deployment and testing of GenAI components with security scans and rollback support, centralized abstraction layers, observability and control mechanisms).*

#### What is different about GenAI CI/CD
- More artefacts are versioned: **application code, IaC, prompts (Prompt Management versions), guardrail versions, agent versions/aliases, KB configuration, model IDs/inference profiles, evaluation datasets, fine-tuned model versions (SageMaker Model Registry)**.
- Tests are **non-deterministic**. Use thresholds, not exact matches: semantic similarity, LLM-as-judge scores, Bedrock model/RAG evaluation jobs, groundedness/relevance metrics, guardrail block rates, latency and token-cost budgets.
- **Promotion = versions + aliases:** the Bedrock Agent alias → agent version, prompt version in Prompt Management, Lambda alias → version. Rollback = point the alias back to the previous version.

#### Reference pipeline (CodePipeline V2)
1. **Source:** CodeCommit/GitHub (CodeConnections) → trigger on PR merge or tag.
2. **Build (CodeBuild):** unit tests, lint, package, **SAST** (Amazon **CodeGuru Security** / Amazon Q Developer security scan, Semgrep, Bandit), **dependency/SCA** (Amazon Inspector for Lambda/ECR images, `pip-audit`), **IaC scanning** (cfn-lint, cfn-guard, Checkov), **secret scanning**.
3. **Deploy to test/staging:** CloudFormation/CDK/SAM.
4. 🎯 **PQ66** **GenAI quality gate (CodeBuild or Step Functions)** (golden datasets and gate design: [Task 5.1 · Skill 5.1.4](../domain-5-testing-validation-troubleshooting/01-task-5.1-evaluation-systems.md#skill-514--systematic-quality-assurance-processes--pq37--pq66)):
   - Run the golden prompt set against the new prompt/model/KB config.
   - **Bedrock Evaluations** job or a custom LLM-as-judge → fail the build if accuracy, faithfulness or toxicity cross thresholds.
   - Red-team / prompt-injection test suite, guardrail tests (PII blocked, denied topics).
   - Latency / token budget tests.
5. **Manual approval** action (for regulated environments).
6. **Production deploy:** canary or linear (**CodeDeploy** for Lambda/ECS: `Canary10Percent5Minutes`, and similar), shift the agent/prompt alias, or the gateway router sends X% to the new model (shadow / A-B).
7. **Automatic rollback:** CloudWatch alarms on error rate, latency, guardrail interventions, user thumbs-down rate → CodeDeploy rollback or pipeline stage rollback. CodePipeline V2 supports **automatic rollback on stage failure** (*verify in docs*).

```yaml
# buildspec.yml for the GenAI quality gate (sketch)
version: 0.2
env:
  variables: { EVAL_THRESHOLD: "0.85" }
phases:
  install:
    runtime-versions: { python: 3.12 }
    commands: [ pip install -r requirements-test.txt ]
  pre_build:
    commands:
      - bandit -r src/ -ll            # SAST
      - pip-audit                     # dependency vulnerabilities
      - cfn-guard validate -d template.yaml -r rules/   # IaC policy
  build:
    commands:
      - pytest tests/unit
      - python eval/run_golden_set.py --prompt-version $PROMPT_VERSION --threshold $EVAL_THRESHOLD
      - python eval/prompt_injection_suite.py
reports:
  eval-report: { files: [ "eval/results.xml" ], file-format: JUNITXML }
```

- **Cross-account deployment:** a tooling account runs the pipeline and assumes deploy roles in dev/test/prod accounts. Artefacts are encrypted with a KMS key shared across accounts.
- **Model lifecycle pipelines** (fine-tuned models): **SageMaker Pipelines** (train → evaluate → register in **Model Registry** with `PendingManualApproval`) → approval event in EventBridge → CodePipeline deploys the endpoint. See [Domain 1 · Task 1.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md).

> [!IMPORTANT]
> **🎯 Practice question 66 — QA for prompt and model changes: compare templates, detect quality issues, metrics, human feedback, block bad configs**
>
> - **Bedrock evaluation jobs** with **custom prompt datasets** compare the outputs of the different prompt templates and models, give **quantitative metrics** (automatic or LLM-as-a-judge), and support **human evaluation** jobs.
> - **CodePipeline** runs the evaluation jobs whenever a template changes and **deploys only the configurations that score above the threshold**.
> - ❌ **Shadow traffic + Comprehend sentiment:** sentiment ≠ quality, and no template comparison.
> - ❌ **CloudWatch latency and error alarms + manual approval:** operational metrics only, not quality.
> - ❌ **Lambda + S3 + QuickSight + daily manual review:** manual, with no standard metrics.

With releases under control, the remaining enterprise problem is runtime consumption: how every team reaches FMs consistently.

#### GenAI gateway architecture
**Problem:** many teams call FMs directly with their own keys and model choices. Logging is inconsistent, guardrails are missing, costs can't be attributed, and quotas get exhausted by one team.

**Solution:** a centralised **GenAI gateway** (an abstraction layer) that every application uses.

```
Apps / teams ──► API Gateway (custom domain, WAF, auth: IAM / Cognito / Lambda authoriser,
                 usage plans + API keys per team, throttling & quotas)
             ──► Router (Lambda or ECS/Fargate for streaming)
                   • model allow-list & routing config (AppConfig / DynamoDB)
                   • tenant quotas & token budgets (DynamoDB counters)
                   • PII redaction / ApplyGuardrail pre- and post-checks
                   • semantic cache (ElastiCache / OpenSearch / MemoryDB)
                   • fallback & retries (other model / Region)
             ──► Amazon Bedrock (application inference profiles per team) · SageMaker endpoints · external providers
Observability:   Bedrock model invocation logging → CloudWatch Logs / S3 · CloudWatch metrics per tenant ·
                 X-Ray tracing · CloudTrail · cost allocation tags → Cost Explorer / CUR
```

| Gateway capability | AWS mechanism |
| --- | --- |
| Authentication / tenant ID | IAM SigV4, Cognito JWT, Lambda authoriser, API keys (API keys alone are **not** authentication; use them for usage plans) |
| Per-team rate limits / quotas | API Gateway **usage plans** (rate, burst, quota per day/month); token-level budgets in DynamoDB |
| Model governance | Allow-list in config; SCP / IAM restricting direct Bedrock access to the gateway role only |
| Safety | **Bedrock Guardrails** (`guardrailIdentifier` on Converse, or the `ApplyGuardrail` API for any model, including non-Bedrock ones) |
| Cost attribution | **Application inference profiles tagged per team/project** → cost allocation tags; per-tenant token metrics |
| Audit & observability | Model invocation logging (prompts and responses to S3/CloudWatch), CloudTrail (API calls), CloudWatch dashboards (`InputTokenCount`, `OutputTokenCount`, `InvocationLatency`, `InvocationThrottles`), X-Ray |
| Provider abstraction | Converse API as a unified interface; router maps logical names (`"chat-default"`) → model IDs, so models change without client code changes |
| Resilience | CRIS profiles, fallback models, circuit breakers (see [Task 2.4](04-task-2.4-fm-api-integrations.md)) |

- **Enforce gateway use:** deny `bedrock:Invoke*` for all principals except the gateway role (SCP or permission boundary). Optionally restrict to `aws:SourceVpce`.
- **Multi-tenant isolation:** tenant ID from the authoriser → used in metadata filters (KB), DynamoDB partition keys and log fields. Never trust a tenant ID that arrives in the request body.
- **AWS reference:** AWS publishes a "multi-provider generative AI gateway" solution guidance based on LiteLLM on ECS (*verify current name in docs*). AgentCore Gateway does the same job for **tools** (turning APIs and Lambdas into MCP tools), not for model calls.

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| On-prem SOAP or mainframe service must feed data to a GenAI app with minimal changes | API Gateway REST API + **VPC Link** facade (VTL JSON↔XML) or Lambda adapter, reached over DX/VPN |
| Burst of 50,000 documents to summarise, Bedrock throttling errors | **SQS** buffer + Lambda `maximumConcurrency` (or batch inference) |
| New business event must trigger several independent AI consumers | **EventBridge** custom bus + rules (content-based filtering) |
| Enrich each DynamoDB change with an FM summary, minimal glue code | **EventBridge Pipes** (DynamoDB Streams → enrichment → target) |
| Keep a KB in sync with an Oracle database in near real time | **AWS DMS CDC** → S3 → S3 event → KB `StartIngestionJob` |
| Pull Salesforce or ServiceNow records into S3 for RAG on a schedule | **Amazon AppFlow** (or a native KB connector) |
| SaaS sends webhooks; FM processing takes 20 s; provider retries cause duplicates | Lambda URL/API GW → validate HMAC → **ack 200** → SQS/EventBridge → async worker; **idempotency** key |
| Long FM job exceeds the API Gateway timeout | Async pattern: **202 + jobId**, Step Functions/SQS, poll or WebSocket callback |
| Simple prompt call in a workflow with no custom code | **Step Functions optimised Bedrock integration** (`InvokeModel`) |
| Employees from Okta/Entra ID need Bedrock-backed apps across accounts | **IAM Identity Center** (SAML + SCIM, permission sets) |
| Mobile app customers must call a GenAI API securely | **Cognito** user pool → JWT authoriser on API Gateway |
| RAG must only return documents a user is allowed to see | Per-user **metadata filters** from identity claims / ACL-aware connectors (Q Business); **trusted identity propagation** |
| Allow only approved models | IAM `Resource` = specific **foundation-model / inference-profile ARNs**; SCP deny `aws-marketplace:Subscribe` |
| Per-team cost tracking for Bedrock on-demand usage | **Tagged application inference profiles** + cost allocation tags |
| AccessDenied when calling a CRIS profile although the profile is allowed | Also allow the **foundation-model ARN in all destination Regions** + SCP Regions |
| Every invoke must use the company guardrail | `bedrock:GuardrailIdentifier` IAM condition / **enforced guardrails** (account or org level) |
| Bedrock calls must not traverse the internet | **Interface VPC endpoint** `bedrock-runtime` (PrivateLink) + endpoint policy + `aws:SourceVpce` |
| On-prem apps call Bedrock privately | **Direct Connect/VPN** → VPC interface endpoint + Route 53 Resolver inbound endpoint |
| EU data residency, still want burst capacity | **EU geographic CRIS** profile (`eu.` prefix); avoid `global.` |
| Cheapest cross-Region option, no residency constraint | **Global CRIS** (about 10% savings) |
| Data legally cannot leave the customer's data centre | Self-hosted open model on **AWS Outposts** (Bedrock isn't available there) |
| Ultra-low latency inference for 5G connected devices | **AWS Wavelength** with a small or distilled model; escalate to Region for complex queries |
| Where is a CRIS request logged and how do I see the processing Region? | CloudTrail in the **source Region**, `additionalEventData.inferenceRegion` |
| Prompt changes broke quality in prod | CI/CD **evaluation gate** (golden set + thresholds) + prompt **versions/aliases** for rollback |
| Pipeline must scan GenAI code and infra for vulnerabilities | CodeBuild with **CodeGuru Security / Q Developer**, Inspector, cfn-guard/Checkov, secrets scanning |
| Roll back automatically when the new model version raises error rates | **CodeDeploy canary** + CloudWatch alarms (Lambda alias) / alias repoint |
| Many teams with inconsistent FM access, logging and costs | **Centralised GenAI gateway** (API GW usage plans + router + guardrails + invocation logging + tagged inference profiles) |
| Force all FM traffic through the gateway | Deny `bedrock:Invoke*` except the gateway role (SCP/permission boundary) |

## Flashcards

<details>
<summary><b>Q:</b> Why put SQS between an upstream system and Bedrock calls?</summary>

It decouples them, absorbs bursts so you stay under Bedrock TPM/RPM quotas, gives retries and a DLQ for failures, and lets you cap concurrency with Lambda's `maximumConcurrency`.

*D2 · Task 2.3 · Loose coupling*
</details>

<details>
<summary><b>Q:</b> EventBridge vs SNS for GenAI event fan-out?</summary>

EventBridge offers rich content-based filtering on any event field, SaaS partner sources, input transformers, archive & replay, schema registry and API destinations. SNS is simpler, high-throughput fan-out with attribute filtering and push to email, SMS and mobile.

*D2 · Task 2.3 · Event-driven*
</details>

<details>
<summary><b>Q:</b> What is EventBridge Pipes and when does it fit GenAI integration?</summary>

Point-to-point source → filter → enrichment → target, with no glue code. Example: DynamoDB Streams → Lambda or Step Functions enrichment calling an FM → target queue or bus.

*D2 · Task 2.3 · EventBridge Pipes*
</details>

<details>
<summary><b>Q:</b> How do you expose an on-premises legacy service to a cloud GenAI app?</summary>

API Gateway REST API with a VPC Link to an NLB that reaches the on-prem service over Direct Connect or VPN. Use VTL mapping or a Lambda adapter to translate protocols (for example JSON↔SOAP/XML).

*D2 · Task 2.3 · Legacy integration*
</details>

<details>
<summary><b>Q:</b> Which service keeps a vector store current with changes in an on-prem Oracle database?</summary>

AWS DMS with change data capture (CDC) into S3 (or Kinesis). S3 events then trigger an incremental Bedrock KB `StartIngestionJob`.

*D2 · Task 2.3 · Data sync*
</details>

<details>
<summary><b>Q:</b> Which service moves SaaS data (Salesforce, ServiceNow, Zendesk) into S3 on a schedule or by event?</summary>

Amazon AppFlow. Bedrock Knowledge Bases also has native connectors for some sources, such as Salesforce, SharePoint and Confluence.

*D2 · Task 2.3 · SaaS sync*
</details>

<details>
<summary><b>Q:</b> List the steps of a robust webhook handler that triggers FM processing.</summary>

Verify the HMAC signature and timestamp → validate the schema → dedupe on the event ID (idempotency) → return 2xx quickly → enqueue to SQS or EventBridge → an async worker calls the FM and posts results back.

*D2 · Task 2.3 · Webhooks*
</details>

<details>
<summary><b>Q:</b> A summarisation request can take 2 minutes. How should the API be designed?</summary>

Asynchronous job pattern: `POST` returns `202` with a job ID. Step Functions or SQS does the work. The client polls a status endpoint backed by DynamoDB, or gets a WebSocket or webhook callback.

*D2 · Task 2.3 · Async API*
</details>

<details>
<summary><b>Q:</b> What does the ARN of a Bedrock foundation model look like, and what is unusual about it?</summary>

`arn:aws:bedrock:<region>::foundation-model/<model-id>`. It has no account ID because AWS owns foundation models.

*D2 · Task 2.3 · Least privilege*
</details>

<details>
<summary><b>Q:</b> A role can call a cross-Region inference profile ARN but still gets AccessDenied. Why?</summary>

It also needs `bedrock:InvokeModel` on the foundation-model ARN in every destination Region of the profile. SCPs must also allow those Regions (or `aws:RequestedRegion = "unspecified"` for global profiles).

*D2 · Task 2.3 · CRIS permissions*
</details>

<details>
<summary><b>Q:</b> How do you allocate Bedrock on-demand costs per team or application?</summary>

Create application inference profiles per team or app, tag them, activate the cost allocation tags, and have apps invoke through the profile ARN.

*D2 · Task 2.3 · Cost allocation*
</details>

<details>
<summary><b>Q:</b> Does Bedrock support ABAC? Which condition keys apply?</summary>

Yes. Use `aws:ResourceTag/<key>`, `aws:RequestTag/<key>` and `aws:TagKeys`, for example matching `aws:ResourceTag/team` to `${aws:PrincipalTag/team}`.

*D2 · Task 2.3 · ABAC*
</details>

<details>
<summary><b>Q:</b> Workforce SSO vs customer sign-in for GenAI apps: which services?</summary>

Workforce: IAM Identity Center federated with the corporate IdP (SAML 2.0 + SCIM). Customers: Amazon Cognito user pools (JWT), with identity pools when the app needs temporary AWS credentials.

*D2 · Task 2.3 · Identity federation*
</details>

<details>
<summary><b>Q:</b> What is trusted identity propagation?</summary>

An IAM Identity Center capability that passes the signed-in user's identity to downstream AWS services (for example Q Business, Redshift, Lake Formation, S3 Access Grants). Those services can then authorise data access per user, not per application role.

*D2 · Task 2.3 · Identity propagation*
</details>

<details>
<summary><b>Q:</b> Which interface VPC endpoints do you need for private Bedrock inference and RAG?</summary>

`bedrock-runtime` for InvokeModel and Converse, `bedrock-agent-runtime` for Retrieve, RetrieveAndGenerate and InvokeAgent, plus `bedrock` and `bedrock-agent` for control-plane and build-time APIs. Add endpoints for dependencies (S3, STS, Logs, KMS).

*D2 · Task 2.3 · PrivateLink*
</details>

<details>
<summary><b>Q:</b> How do you make sure Bedrock is only called through your VPC endpoint?</summary>

Add a deny condition on `aws:SourceVpce` (or `aws:SourceVpc`) in IAM or SCP policies, and attach a restrictive endpoint policy to the interface endpoint.

*D2 · Task 2.3 · Network controls*
</details>

<details>
<summary><b>Q:</b> Geographic vs global cross-Region inference: residency and cost?</summary>

Geographic (`us.`, `eu.`, `apac.`) keeps processing within that geography at standard pricing. Global (`global.`) routes to any supported commercial Region for about 10% savings, so it is not suitable under residency rules.

*D2 · Task 2.3 · Data residency*
</details>

<details>
<summary><b>Q:</b> Where does CloudTrail record a cross-Region inference call?</summary>

In the source (calling) Region. The `additionalEventData.inferenceRegion` field shows the Region that processed it.

*D2 · Task 2.3 · CRIS auditing*
</details>

<details>
<summary><b>Q:</b> Data may never leave the customer's facility, but they want GenAI. What is the AWS pattern?</summary>

Run a self-hosted open-weight model on AWS Outposts (or on-prem), integrated with the parent Region for management. Bedrock doesn't run on Outposts.

*D2 · Task 2.3 · Outposts*
</details>

<details>
<summary><b>Q:</b> Wavelength vs Local Zones vs Outposts?</summary>

Wavelength: compute inside 5G carrier networks for ultra-low latency to mobile devices. Local Zones: AWS infrastructure in metro areas close to users. Outposts: AWS racks in your own data centre for local data processing or residency.

*D2 · Task 2.3 · Edge options*
</details>

<details>
<summary><b>Q:</b> Is Direct Connect traffic encrypted by default?</summary>

No. Add MACsec on supported dedicated connections, or run Site-to-Site VPN over Direct Connect. Application-level TLS to Bedrock endpoints still applies.

*D2 · Task 2.3 · Hybrid routing*
</details>

<details>
<summary><b>Q:</b> What makes CI/CD for GenAI different from traditional CI/CD?</summary>

You also version prompts, guardrails, agent aliases, KB config, model IDs and evaluation datasets. Tests are non-deterministic, so you need threshold-based evaluation gates (golden sets, LLM-as-judge, Bedrock Evaluations, red-team suites). Rollback happens through version and alias repointing.

*D2 · Task 2.3 · GenAI CI/CD*
</details>

<details>
<summary><b>Q:</b> Which security scans belong in a GenAI CodeBuild stage?</summary>

SAST (CodeGuru Security or Q Developer, Bandit), dependency/SCA (Inspector, pip-audit), IaC policy (cfn-guard, Checkov, cfn-lint), secret scanning, plus prompt-injection and guardrail test suites.

*D2 · Task 2.3 · Security scans*
</details>

<details>
<summary><b>Q:</b> What does a centralised GenAI gateway provide?</summary>

One abstraction layer giving auth and tenant identification, per-team throttling and quotas (usage plans, token budgets), a model allow-list and routing, guardrails, caching, fallback, invocation logging, and cost attribution (tagged inference profiles). Apps stay decoupled from model IDs and providers.

*D2 · Task 2.3 · GenAI gateway*
</details>

<details>
<summary><b>Q:</b> How do you stop teams from bypassing the GenAI gateway?</summary>

Deny `bedrock:InvokeModel*` (and Converse through the same actions) for every principal except the gateway's role, using an SCP or permission boundary. Optionally require `aws:SourceVpce`.

*D2 · Task 2.3 · Gateway enforcement*
</details>

---
[← Back to Domain 2](README.md)
