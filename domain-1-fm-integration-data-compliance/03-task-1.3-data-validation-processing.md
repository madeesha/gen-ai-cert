[← Back to Domain 1](README.md)

# 🧪 Task 1.3 · Implement data validation and processing pipelines for FM consumption

> **Exam domain:** D1 · Foundation Model Integration, Data Management, and Compliance (31%)
>
> - **Skill 1.3.1**: Data validation workflows (AWS Glue Data Quality, SageMaker Data Wrangler, custom Lambda, CloudWatch metrics)
> - **Skill 1.3.2**: Processing complex data types: text, image, audio, tabular (Bedrock multimodal models, SageMaker Processing, Amazon Transcribe, multimodal pipelines)
> - **Skill 1.3.3**: Formatting input for FM inference (JSON for Bedrock API requests, structured payloads for SageMaker AI endpoints, conversation formatting)
> - **Skill 1.3.4**: Improving input quality (Bedrock reformatting, Amazon Comprehend entity extraction, Lambda normalisation)

## Key points
- **Garbage in, garbage out.** Answer quality is capped by input quality (RAG documents, fine-tuning data, prompts). Validate **before** you embed, train or invoke; fixing it afterwards means re-embedding, retraining and lost trust.
- **Pick the validation tool by data type and user:** **Glue Data Quality** (DQDL rules, rule recommendations, ML anomaly detection) for tabular pipelines · **Data Wrangler** for data scientists · **DataBrew** for no-code analysts · **custom Lambda** for unstructured and GenAI-specific checks · **CloudWatch** metrics and alarms to track it all (Skill 1.3.1).
- **Non-text data: convert to text, or process natively** with multimodal models and embeddings. Many designs do both (1.3.2).
- **Route by modality:** audio → **Transcribe** · scanned docs/forms/tables → **Textract** or **Bedrock Data Automation** · images → **multimodal FM** · video → **BDA** / Nova or a frame + transcript pipeline · tables → **text-to-SQL**, not embedded rows (1.3.2).
- **Scale out preprocessing:** heavy or custom work (GPU, Spark, >15 min) → **SageMaker Processing**; millions of S3 objects → **Step Functions Distributed Map** (1.3.2).
- **Converse is the default request shape** (one schema, content blocks). Use **InvokeModel** for embeddings, image generation and model-native bodies. On **SageMaker**, the **container** defines the payload and you may have to apply the chat template yourself (1.3.3).
- **Conversation formatting:** FMs are stateless, so re-send history every turn (persist it in DynamoDB), start with `user` and alternate roles, answer each `toolUse` with a matching `toolResult`, and trim with a sliding window + summary (1.3.3).
- **Improve input in order of cost:** deterministic **Lambda** normalisation first → **Comprehend** for entities, language, PII and classification → a small **Bedrock** model only for what rules can't do (1.3.4).
- **Redact PII before embedding or fine-tuning.** Once it's in a vector store or model weights it's very hard to remove (1.3.4).

## Notes

### Where validation and processing fit in a GenAI data pipeline
- Three consumers of prepared data, each with different quality needs:

| Consumer | What "good data" means | Typical failure if skipped |
| --- | --- | --- |
| **RAG ingestion** (KB / vector store) | Clean text, correct metadata, no duplicates, current, no PII/secret leakage, parseable | Irrelevant/duplicate chunks, stale answers, citation to wrong doc, PII surfaced to users |
| **Fine-tuning / distillation datasets** | Correct JSONL schema, balanced, labelled correctly, deduplicated, no PII, no toxic content, within token limits | Job fails validation, model learns bias/errors, memorises PII |
| **Real-time inference input** | Valid request schema, within context/token limits, normalised, safe | `ValidationException`, truncated context, inconsistent answers, prompt injection |

- Reference pipeline (batch ingestion):
  1. **Land** raw data in S3 (raw zone). Trigger with **S3 Event Notifications → EventBridge**.
  2. **Validate**: Glue Data Quality (structured), Lambda (files/JSON/token limits). Failed records go to a **quarantine prefix** (or a DLQ) + SNS alert.
  3. **Extract/convert by modality**: Transcribe, Textract/BDA, multimodal FM for images, SageMaker Processing for custom work.
  4. **Enrich & normalise**: Comprehend entities → metadata, Lambda normalisation, Bedrock rewriting.
  5. **Write** to a curated zone in S3 (+ `.metadata.json` sidecars) → **KB sync** / embedding job / training job.
  6. **Monitor**: CloudWatch metrics (records processed/failed, DQ score) → alarms → dashboards.
- Orchestration options: **Step Functions** (serverless, visual, retries/catch, Distributed Map for millions of S3 objects) · **Glue workflows** (Glue-centric ETL) · **SageMaker Pipelines** (ML-centric, lineage, ties to Model Registry) · **MWAA** (existing Airflow estates).
- Medallion-style zones (raw → validated/clean → curated/enriched) make reprocessing and audits easy. Keep raw **immutable** and use S3 versioning.
- ⚠️ Validating **after** embedding means you must re-embed and re-sync to fix issues, so validate upstream. Forgetting to **re-sync the KB** after changing documents or metadata sidecars leaves the KB serving stale data.

### Skill 1.3.1 — Data validation workflows 🎯 PQ44
> *Create comprehensive data validation workflows to ensure data meets quality standards for FM consumption (for example, by using AWS Glue Data Quality, SageMaker Data Wrangler, custom Lambda functions, Amazon CloudWatch metrics).*

#### Data quality dimensions to validate
| Dimension | Example check for GenAI data |
| --- | --- |
| **Completeness** | No empty `body`/`text` fields; every training record has `prompt` and `completion` |
| **Validity / conformity** | JSONL parses; field types correct; dates ISO 8601; language = expected |
| **Uniqueness** | No duplicate documents/records (duplicates bias retrieval and fine-tuning) |
| **Consistency** | Same entity spelled the same way; categories from an allowed list |
| **Accuracy** | Labels correct (sample + human review, e.g. SageMaker Ground Truth / A2I) |
| **Timeliness / freshness** | `last_updated` within N days, so RAG doesn't serve stale policy |
| **Size / length** | Text length within chunk/token limits; files within service size limits |
| **Safety / compliance** | No PII, secrets, or toxic content; licence/consent metadata present |

#### Decision table — which validation tool?
The tools below overlap, so choose by data type and by who does the work. Each is explained in the sections that follow.

| Situation | Pick |
| --- | --- |
| Rule-based checks on tabular data in Glue ETL/catalog, alerts, scale | **Glue Data Quality** |
| Analysts, no code, profile + clean | **Glue DataBrew** |
| Data scientist exploring and preparing a fine-tuning/ML dataset visually, exporting to a pipeline | **SageMaker Data Wrangler** |
| Unstructured files, JSONL schema, token limits, PII, language checks | **Custom Lambda** (+ Comprehend) |
| Track quality over time, alert on degradation | **CloudWatch metrics + alarms** (+ Glue DQ anomaly detection) |
| Human verification of labels/ground truth | **SageMaker Ground Truth / Amazon A2I** |
| Discover PII sitting in S3 buckets before ingestion | **Amazon Macie** |

> [!IMPORTANT]
> **🎯 Practice question 44 — 5 TB a day of structured data: schema, anomalies, nulls, metrics, custom alerts, rules, LEAST complexity**
>
> - **AWS Glue Data Quality** (serverless, scales) → **DQDL rules** for schema, completeness (nulls), uniqueness and value ranges, plus **ML anomaly detection**. Results are **published as metrics**.
> - **SageMaker Data Wrangler** → explore and analyse the features.
> - **Lambda** → **custom alerting logic**. **CloudWatch** → metrics, thresholds and alarms.
> - ❌ **SageMaker Processing + Lambda + Step Functions:** batch jobs you have to orchestrate and tune.
> - ❌ **Data Wrangler + Lambda without Glue DQ:** no managed rules engine, so it's custom code.
> - ❌ **+ Firehose → OpenSearch dashboards:** extra infrastructure (shards, indexes) for no benefit.

#### AWS Glue Data Quality
- Built on the open-source **Deequ** engine. Serverless and pay per DPU-hour.
- **Two modes**:
  - **Data Catalog** checks validate **data at rest** in catalogued tables (S3, etc.) on a schedule.
  - **ETL job** checks validate **data in flight** using the `EvaluateDataQuality` transform in Glue Studio or a script. Records can be routed by pass/fail.
- **DQDL (Data Quality Definition Language)** is declarative rules in a ruleset:

```text
Rules = [
    IsComplete "doc_id",
    IsUnique "doc_id",
    Completeness "body" > 0.98,
    ColumnLength "body" between 50 and 20000,
    ColumnValues "language" in ["en", "fr", "de"],
    ColumnValues "department" in ["hr", "legal", "finance", "it"],
    DataFreshness "last_updated" <= 30 days,
    RowCount > 1000,
    CustomSql "select count(*) from primary where body like '%lorem ipsum%'" = 0
]
```

- Other rule types you should recognise: `Uniqueness`, `IsPrimaryKey`, `ColumnExists`, `ColumnDataType`, `ColumnCorrelation`, `Entropy`, `Mean`/`StandardDeviation`/`Sum`, `ReferentialIntegrity`, `DatasetMatch`, `DistinctValuesCount`, `SchemaMatch`, `AggregateMatch`. *(Exact list: verify in docs.)*
- **Rule recommendations** profile the data and suggest a starter ruleset, so you don't have to write rules from scratch.
- **ML-based anomaly detection / dynamic rules** learn from historical statistics and flag unusual drift (e.g. row count drops 60%), e.g. `RowCount > avg(last(10)) * 0.8`.
- **Outputs**: a data quality **score** (% rules passed) plus per-rule results. Publishes **CloudWatch metrics** and **EventBridge events**, so you can alarm, page, or stop a pipeline. Row-level results let you **quarantine bad rows**.
- Pick it for **structured/semi-structured tabular** data (CSV, Parquet, JSON tables) at scale, which includes metadata tables for RAG and fine-tuning datasets in tabular form.

#### AWS Glue DataBrew
- **No-code visual** data prep for analysts. **250+ prebuilt transforms**, **data profiling** (distributions, missing values, outliers), data quality rules, and PII detection/masking transforms. Recipes can be versioned and scheduled as jobs.
- Exam cue: "business analysts, no code, clean and profile data". Data engineers who need pipelines-as-code use Glue Data Quality.

#### Amazon SageMaker Data Wrangler
- **Visual, low-code** data preparation, now part of **SageMaker Canvas** (and SageMaker Unified Studio). Imports from S3, Athena, Redshift, Snowflake, EMR, Databricks…
- **300+ built-in transforms** (encode, impute, dedupe, text ops, time-series) + custom PySpark/pandas/SQL.
- **Data Quality and Insights report**: missing values, duplicates, outliers, class imbalance, target leakage, feature correlation. Use it to spot fine-tuning dataset problems quickly.
- **Export** the flow to a **SageMaker Processing job**, **SageMaker Pipelines** step, Python code, or a **Feature Store** feature group. This takes it from exploration to a repeatable production pipeline.
- Canvas can also **prepare fine-tuning datasets** and use natural-language data prep (via Amazon Q / GenAI assistance). *(Verify current UI naming in docs.)*
- Glue DQ vs Data Wrangler: **Glue DQ** = rule-based checks inside data engineering pipelines (ETL, catalog). **Data Wrangler** = data scientists exploring, transforming, and profiling data for ML/FM.

#### Custom Lambda validators
Use them for what managed DQ tools don't do (especially unstructured and GenAI-specific checks):
- File checks: MIME type / extension allow-list, **file size** within limits (e.g. KB S3 data source files **≤ 50 MB**), not encrypted/corrupt, page count.
- **JSON / JSONL schema validation** (e.g. `jsonschema`) for fine-tuning records and batch inference input.
- **Token counting**: estimate tokens (Bedrock `CountTokens` API for supported models, or tokenizer libs) and reject or split oversized inputs. *(CountTokens availability per model: verify in docs.)*
- **Language detection** (Comprehend `DetectDominantLanguage`) to confirm content matches the embedding model's supported languages.
- **PII / secrets detection** (Comprehend `ContainsPiiEntities` / `DetectPiiEntities`, regex for keys) → block or redact.
- **Toxicity / safety** (Comprehend toxicity detection, or `ApplyGuardrail` to screen text without calling a model).
- Dedup via **content hash** (SHA-256 of normalised text) stored in DynamoDB.
- Lambda limits that matter: **15-min max timeout**, **up to 10,240 MB memory**, **/tmp up to 10 GB**, **6 MB synchronous invoke payload** (256 KB async). Bigger or longer work goes to Glue, SageMaker Processing, Fargate/Batch, or Step Functions Distributed Map.

```python
# Lambda: validate a fine-tuning JSONL record & emit metrics (sketch)
import json, boto3
cw = boto3.client("cloudwatch")
REQUIRED = {"prompt", "completion"}

def handler(event, _):
    bad = 0
    for line in event["records"]:
        try:
            rec = json.loads(line)
            if not REQUIRED <= rec.keys() or not rec["completion"].strip():
                raise ValueError("missing fields")
        except Exception:
            bad += 1
    cw.put_metric_data(Namespace="GenAI/DataQuality", MetricData=[
        {"MetricName": "InvalidRecords", "Value": bad, "Unit": "Count"},
        {"MetricName": "TotalRecords", "Value": len(event["records"]), "Unit": "Count"}])
    return {"invalid": bad}
```

#### CloudWatch for data quality monitoring
- **Custom metrics** via `PutMetricData`, or **Embedded Metric Format (EMF)**, which writes structured logs from Lambda that turn into metrics with no extra API call and are cheaper at high volume.
- Useful metrics: `InvalidRecords`, `QuarantinedFiles`, `DQScore`, `AvgTokensPerDoc`, `PIIDetected`, `DuplicateRate`, `IngestionLagMinutes`.
- **Alarms** on thresholds or **metric math** (e.g. `InvalidRecords / TotalRecords > 2%`) → SNS / EventBridge → stop the pipeline (Step Functions `Choice` / fail state) or open a ticket.
- **CloudWatch anomaly detection** alarms catch unexpected changes in volume or quality.
- **Logs Insights** queries over validator logs help with root-cause analysis. **Dashboards** give pipeline health at a glance.
- KB ingestion jobs also report **per-document failures** (e.g. unsupported file, too large). Check the ingestion job statistics/logs, and enable KB logging to CloudWatch Logs. *(Verify log delivery options in docs.)*

### Skill 1.3.2 — Processing complex data types (text, image, audio, tabular)
> *Create data processing workflows to handle complex data types including text, image, audio, and tabular data with specialized processing requirements for FM consumption (for example, by using Amazon Bedrock multimodal models, SageMaker Processing, AWS Transcribe, advanced multimodal pipeline architectures).*

#### Two strategies for non-text data
1. **Convert to text, then use a text pipeline.** Transcribe audio, OCR/parse documents, caption images. This is simple, works with any text FM or embedding model, and gives cheap, searchable text. The downside is that some information is lost (layout, visual nuance, tone).
2. **Process natively with multimodal models/embeddings.** Send images/documents/video directly to a multimodal FM, or embed with a **multimodal embedding model** (Titan Multimodal Embeddings G1, Cohere Embed v4, Nova Multimodal Embeddings) for cross-modal search. This keeps visual meaning, but costs more and has fewer model choices.
- Many production designs **combine both**: transcript/caption text for keyword + semantic search, and native images for answer generation.

#### Modality → service map
| Modality | Extraction / processing | Notes |
| --- | --- | --- |
| **Plain text / HTML / Markdown** | Lambda/Glue cleaning, KB default parser | Strip boilerplate, keep headings for structural chunking |
| **PDF / scanned docs / forms** | **Amazon Textract** (OCR, `FORMS`, `TABLES`, `QUERIES`, `LAYOUT`), **Bedrock Data Automation**, **KB FM parsing** | Multi-page PDFs use **async** `StartDocumentAnalysis` (results via SNS) |
| **Images** | **Bedrock multimodal FMs** (Claude, Nova Lite/Pro, Llama vision) for captions/extraction · **Rekognition** (labels, text-in-image, faces, moderation) · **BDA** | Caption → text embedding, or multimodal embedding |
| **Audio** | **Amazon Transcribe** (batch or streaming) · **BDA** (audio summaries/transcripts) | Speaker diarization, custom vocabulary, PII redaction |
| 🎯 **PQ13** **Video** | **BDA** (scene summaries, transcripts, chapters, logos, text-in-video) · **Nova Lite/Pro** video understanding · Rekognition Video | Frame sampling + transcript if building custom |
| **Tabular / structured** | Glue/SageMaker Processing to serialise · **KB structured data (NL→SQL over Redshift)** · Athena | Don't blindly embed numeric rows; use text-to-SQL for aggregation |
| **Multilingual** | **Comprehend** language detection → **Amazon Translate** | Or use a multilingual embedding model (Cohere Embed multilingual / v4). Embedding with a model that doesn't support the language gives poor retrieval |
| **Medical text/audio** | **Comprehend Medical**, **Transcribe Medical** / HealthScribe | Domain entities (medications, conditions), HIPAA-eligible |

> [!WARNING]
> **🎯 Practice question 13 — 10,000 h of video: scenes, timestamped transcripts, on-screen text, emotion, all in sync and searchable**
>
> **Pattern: split the video into time-aligned pieces, analyse them with a multimodal model, and store one timeline.**
> 1. **SageMaker Processing job** extracts **frames at fixed intervals** and attaches a **timestamp** to each frame.
> 2. **Amazon Transcribe** with timestamps (word-level time markers, plus speaker labels) handles the audio.
> 3. A **Claude model in Bedrock** (image + text input) analyses **frame + transcript pairs** for the same time window. It returns the scene description, on-screen text and **emotional context**.
> 4. Write everything to a **unified JSON structure keyed by timestamp**, which keeps the time relationships. Index it (for example in OpenSearch) so users can search any field and **jump to that moment**.
>
> Traps:
> - ❌ **The whole video in one Claude request:** Claude takes **images and text, not video files**, and a full video would exceed payload and token limits.
> - ❌ **Rekognition + Comprehend for emotion:** Comprehend is **text-only NLP**. It can't read emotion from visuals or music.
> - ❌ **Separate S3 buckets per content type:** loses the synchronisation, so users can't jump to a moment.
> - ℹ️ **Bedrock Data Automation** (video scene summaries, transcripts, text-in-video) is a managed option too, but the exam answer here is the frame + Transcribe + Claude pipeline.

#### Amazon Transcribe (audio → text)
- **Batch** (`StartTranscriptionJob`, file in S3) vs **streaming** (HTTP/2 or WebSocket, real-time).
- Features that matter for FM pipelines:
  - **Speaker diarization** (who said what) lets you build `Agent:` / `Customer:` turns for summarisation.
  - **Channel identification** (separate stereo channels in call centres).
  - **Custom vocabulary** (product names, jargon) and **custom language models** (domain-trained). These are the answer when "domain terms are mis-transcribed".
  - **Automatic language identification** (including multi-language audio).
  - **PII redaction** in transcripts (redact before the FM sees the text).
  - **Vocabulary filtering** (mask profanity), **toxicity detection** (audio).
  - **Transcribe Call Analytics**: sentiment, interruptions, talk time, issue detection, and **generative call summarisation**.
- Output is **JSON** with the transcript, word-level **timestamps** and confidence, and speaker labels. Keep the timestamps as chunk metadata, so answers can cite "minute 12:40".
- Pattern: S3 upload → EventBridge → Step Functions → Transcribe job → Lambda formats transcript into turns → Comprehend enrichment → Bedrock summarise / KB ingest.

#### Amazon Textract & Bedrock Data Automation (documents)
- **Textract APIs**: `DetectDocumentText` (OCR lines/words) · `AnalyzeDocument` with `FORMS` (key-value pairs), `TABLES` (cells/merged cells), `QUERIES` (ask "What is the policy number?"), `SIGNATURES`, `LAYOUT` (titles, headers, paragraphs, lists, which helps with structure-aware chunking) · `AnalyzeExpense` (invoices/receipts) · `AnalyzeID` (identity docs) · `AnalyzeLending`.
- Synchronous APIs are for single-page images. **Multi-page PDFs/TIFFs** need the **async** `Start*` APIs + SNS completion notification.
- ⚠️ Lambda timeouts on large PDFs or long audio: use async Textract/Transcribe jobs + Step Functions waits (`.sync` / callback) instead of polling inside Lambda.
- **Bedrock Data Automation (BDA)** handles **documents, images, audio and video** with one API: **standard output** (predefined insights per modality, e.g. document text/markdown with layout, audio transcripts, video chapters and summaries) or **custom output with blueprints** (your schema of fields, e.g. invoice number, total, line items; from the catalogue or generated with natural language). It can also be the **KB parser**. Exam cue: "extract structured fields from many doc types / multimodal content **without managing multiple services or prompts**" → **BDA**. → Full detail: [Task 2.5 · Skill 2.5.3](../domain-2-implementation-integration/05-task-2.5-app-integration-dev-tools.md#skill-253--business-system-enhancements)
- **KB parsing options**: **default parser** (text only, fastest/cheapest) · **Bedrock Data Automation** parser · **FM as parser** (e.g. Claude/Nova reads pages including tables/figures, with a customisable parsing prompt). FM/BDA parsing **costs more** but keeps table and figure meaning.

#### Images with Bedrock multimodal FMs
- Send image bytes as a Converse `image` content block (formats: **png, jpeg, gif, webp**) alongside the text instruction. Common pipeline uses:
  - **Captioning / alt-text** → store the caption as text for embedding/search.
  - **Structured extraction** from charts, diagrams, screenshots, product photos → JSON.
  - **Classification / tagging** → metadata for filtering.
  - **Quality checks** (blurry, wrong product, inappropriate). Combine with Rekognition moderation or the guardrail image filter.
- Size limits apply per image and per request (dimensions and MB). **Resize/compress before sending** to cut tokens and latency. *(Exact per-model image limits: verify in docs.)*
- Image tokens count toward input cost, so downscale to the smallest resolution that still answers the question.

#### Tabular data
- LLMs are **bad at arithmetic over large tables** and embeddings of numeric rows retrieve poorly.
- Options:
  1. **Small tables in context**: serialise to **Markdown table / CSV / JSON** and include headers + units. Markdown/CSV is usually token-efficient.
  2. **Row-to-text ("textualisation")**: `"Product X, region EMEA, Q3 revenue $1.2M"`. This lets semantic search work on descriptive rows.
  3. **Text-to-SQL / structured retrieval**: the FM generates SQL against **Redshift/Athena**. **Bedrock Knowledge Bases structured data stores** support NL → SQL over Redshift (and Glue Data Catalog via Redshift). This is the right answer for **aggregations, filters, joins, exact numbers**.
  4. **Code interpreter / tool use**: an agent runs Python (e.g. Bedrock Agents code interpretation, AgentCore Code Interpreter) for calculations.
- Pre-processing: fix types, consistent units/currency, explicit column descriptions (schema comments are crucial for text-to-SQL accuracy), drop irrelevant columns, and handle nulls.

#### SageMaker Processing
- Managed jobs that run **your container** or built-in **scikit-learn / Spark / PyTorch / Hugging Face** processors on ephemeral instances (CPU or **GPU**), with S3 input/output. You pay only while the job runs.
- Use it for: large-scale text cleaning, dedup (MinHash), tokenisation, **batch embedding generation with an open-source model**, image resizing/feature extraction, audio feature extraction, building fine-tuning datasets. It's also good for **evaluation** jobs.
- Integrates as a step in **SageMaker Pipelines** (with lineage). Data Wrangler flows export to it.
- Choose it over Lambda when work is **long-running (>15 min), memory/GPU heavy, or needs custom libraries/distributed Spark**. Choose it over Glue when the team is ML-centric or needs GPUs/custom ML libraries.

#### Advanced multimodal pipeline architecture (reference)
```text
S3 (raw) ──EventBridge──► Step Functions
   │                         ├─ Choice on content-type
   │                         │   ├─ audio/* ─► Transcribe ─► Lambda (turns + timestamps)
   │                         │   ├─ pdf/tiff ─► Textract async / BDA ─► Lambda (markdown + tables)
   │                         │   ├─ image/*  ─► Bedrock multimodal FM (caption + JSON tags)
   │                         │   ├─ video/*  ─► BDA (chapters, transcript, summary)
   │                         │   └─ csv/parquet ─► Glue DQ ─► Redshift (NL→SQL path)
   │                         ├─ Comprehend (entities, language, PII) ─► redact / metadata
   │                         ├─ Glue DQ / Lambda validators ─► quarantine on fail
   │                         └─ write curated text + .metadata.json ─► S3 (curated)
   └──────────────────────────────────────────────────► KB StartIngestionJob / embeddings
                                       CloudWatch metrics + alarms throughout
```
- **Distributed Map** in Step Functions iterates over **millions of S3 objects** with high parallelism. Use it for backfills.
- Use **SQS** between stages to buffer and absorb throttling (e.g. Bedrock/Transcribe quotas). Add **retry with exponential backoff** on `ThrottlingException`.
- **Bedrock batch inference** (JSONL in S3 → JSONL out, asynchronous, **~50% cheaper than on-demand** for supported models) is the right choice for **bulk offline** captioning/summarisation/extraction. *(Discount and minimum record counts: verify in docs.)* → Batch inference detail: [Task 2.2 · Skill 2.2.1](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-221--deploy-fms-based-on-application-needs-and-performance-requirements)

### Skill 1.3.3 — Formatting input data for FM inference
> *Format input data for FM inference according to model-specific requirements (for example, by using JSON formatting for Amazon Bedrock API requests, structured data preparation for SageMaker AI endpoints, conversation formatting for dialog-based applications).*

#### Bedrock: Converse vs InvokeModel
| | **Converse / ConverseStream** | **InvokeModel / InvokeModelWithResponseStream** |
| --- | --- | --- |
| Request shape | **Unified** across models | **Provider-specific JSON body** |
| Switching models | Change `modelId` only | Rewrite body + response parsing |
| Multimodal, documents, tool use | Standard content blocks | Provider-specific formats |
| Model-specific params | `additionalModelRequestFields` | Native in body |
| Use when | Default for chat/apps, model portability | Model features not exposed in Converse, embeddings, image generation models |
- **Embedding models** (Titan Embeddings, Cohere Embed) and **image generation models** are called with **InvokeModel**, not Converse.

#### Converse request anatomy
```json
{
  "modelId": "anthropic.claude-...",
  "system": [{ "text": "You are a claims assistant. Answer only from the provided context." }],
  "messages": [
    { "role": "user", "content": [
        { "text": "Summarise this claim and list missing documents." },
        { "document": { "format": "pdf", "name": "claim-123", "source": { "bytes": "<bytes>" } } },
        { "image": { "format": "png", "source": { "bytes": "<bytes>" } } }
    ]}
  ],
  "inferenceConfig": { "maxTokens": 800, "temperature": 0.2, "topP": 0.9, "stopSequences": ["</answer>"] },
  "additionalModelRequestFields": { "top_k": 50 },
  "guardrailConfig": { "guardrailIdentifier": "gr-abc", "guardrailVersion": "1" },
  "toolConfig": { "tools": [ { "toolSpec": { "name": "get_policy", "inputSchema": { "json": { "type": "object" } } } } ] }
}
```
- **Content block types**: `text`, `image` (png/jpeg/gif/webp), `document` (e.g. pdf, csv, doc, docx, xls, xlsx, html, txt, md), `video` (supported models), `toolUse`, `toolResult`, `guardContent` (marks which text the guardrail should evaluate), `reasoningContent` (models with extended thinking), `cachePoint` (prompt caching on supported models).
- Image/document sources can be **raw bytes** (SDKs base64-encode them for you) or, for some models, an **S3 location**. There are per-request limits on number and size of images/docs. *(Exact limits: verify in docs.)*
- `document.name` has restrictions (allowed characters only). An unsanitised user filename can trigger a `ValidationException`, and the name is also a **prompt-injection vector**, so use a neutral name.
- **Response**: `output.message.content[]`, `stopReason` (`end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `guardrail_intervened`, `content_filtered`), `usage` (input/output tokens), `metrics.latencyMs`.
- `stopReason = max_tokens` means the output was truncated. Raise `maxTokens` or ask for a shorter format.
- ⚠️ Converse `ValidationException` causes: non-alternating roles, first message not `user`, unsupported document format/name, image too large, `maxTokens` above model max, or a `toolUse` without a matching `toolResult`.

#### InvokeModel bodies (recognise the shapes)
```json
// Anthropic Claude (Messages API on Bedrock)
{ "anthropic_version": "bedrock-2023-05-31", "max_tokens": 512,
  "system": "You are concise.",
  "messages": [ { "role": "user", "content": [ { "type": "text", "text": "Hello" } ] } ],
  "temperature": 0.3 }

// Amazon Titan Text Embeddings V2
{ "inputText": "Refund policy for damaged goods", "dimensions": 512, "normalize": true }

// Cohere Embed (distinguish documents vs queries!)
{ "texts": ["Refund policy for damaged goods"], "input_type": "search_document" }

// Amazon Nova (messages-style schema)
{ "schemaVersion": "messages-v1",
  "system": [ { "text": "You are concise." } ],
  "messages": [ { "role": "user", "content": [ { "text": "Hello" } ] } ],
  "inferenceConfig": { "maxTokens": 300, "temperature": 0.3 } }
```
- Call it with `contentType="application/json"`, `accept="application/json"`, and `body=json.dumps(...)`. The response `body` is a stream you `json.loads(response["body"].read())`.
- **Cohere `input_type` gotcha**: use `search_document` when indexing and `search_query` when querying. If you mix them up, retrieval quality drops.
- Titan V2 `dimensions` must **match the vector index dimension** (256 / 512 / 1024).
- **Batch inference** input is **JSONL**, one record per line: `{"recordId": "r1", "modelInput": { ...InvokeModel body... }}`. Input and output both live in S3, and a service role needs S3 access.

#### SageMaker AI endpoints — structured payloads
- `InvokeEndpoint(EndpointName, ContentType, Accept, Body)`. **The container defines the schema.** JSON is common, but CSV, `application/x-npy`, image bytes, and so on are possible.
- **Hugging Face TGI / LMI (DJL Serving, vLLM)** containers for LLMs:
```json
{ "inputs": "<s>[INST] Summarise: ... [/INST]",
  "parameters": { "max_new_tokens": 256, "temperature": 0.2, "top_p": 0.9, "do_sample": true, "stop": ["</s>"] } }
```
- Some JumpStart/LMI containers also accept an **OpenAI-style Messages API** (`{"messages": [...]}`) and apply the model's **chat template** server-side. If you send raw `inputs`, **you must apply the model's chat template yourself** (e.g. Llama `<|begin_of_text|><|start_header_id|>user<|end_header_id|>...`, Mistral `[INST] ... [/INST]`). If the template is wrong, you get rambling or poor-quality output.
- Custom models use **inference scripts** (`input_fn` / `predict_fn` / `output_fn`) to deserialise, validate, and serialise.
- **Payload limits decide the inference option**: **real-time** takes small payloads (**~6 MB**, **~60 s** response timeout); **asynchronous** takes the input from S3 (**up to 1 GB**, up to **1 hour**, SNS notification); **batch transform** is S3 → S3 offline; **serverless** is small payloads with no GPUs; token streaming uses `InvokeEndpointWithResponseStream`. *(Limits change, so verify in docs. Real-time limits in particular have been raised for some configurations.)* → Full detail: [Task 2.2 · Skill 2.2.1](../domain-2-implementation-integration/02-task-2.2-model-deployment-strategies.md#skill-221--deploy-fms-based-on-application-needs-and-performance-requirements)
- **Tabular input for classic ML endpoints**: same feature order and encoding as training (reuse the Data Wrangler/Processing transforms, or **SageMaker Feature Store** for consistent online/offline features).

#### Conversation formatting for dialog applications
- FMs are **stateless**, so you must **re-send history** each turn. In the Converse API:
  - `messages` must **start with `user`** and **alternate** `user` / `assistant`. Merge consecutive same-role messages or you get a `ValidationException`.
  - Put the **system prompt** (persona, rules, output format) in `system`, not as a fake user turn.
  - **Tool use loop**: the model returns `stopReason: tool_use` with a `toolUse` block (`toolUseId`, `name`, `input`). Your app runs the tool, appends the assistant message as-is, then a **user** message containing a `toolResult` with the **same `toolUseId`** (status `success`/`error`), then calls Converse again.
- **History storage**: **DynamoDB** (partition key `session_id`, sort key `timestamp`/turn number, **TTL** to expire old sessions). For agents, use Bedrock Agents session state/memory or **AgentCore Memory** (short- and long-term).
- **Managing the context window** (cost + latency + the "lost in the middle" problem):
  - **Sliding window**: keep the last N turns.
  - **Summarise older turns** with a cheap model and keep the summary + recent turns.
  - **Token budgeting**: reserve tokens for system prompt + retrieved context + expected output.
  - **Prompt caching** (`cachePoint`) for long static prefixes (system prompt, large docs) on supported models cuts cost and latency.
- **Prefill** (Anthropic): end with a partial `assistant` message (e.g. `{`) to force JSON output. Check model support before relying on it.
- **Multi-turn RAG**: rewrite the follow-up question into a **standalone query** using history before retrieval ("What about for contractors?" → "What is the parental leave policy for contractors?"). `RetrieveAndGenerate` handles this when you pass a `sessionId`.

```python
# Converse multi-turn with DynamoDB history (sketch)
history = load_history(session_id)                 # [{"role":"user","content":[{"text":...}]}, ...]
history.append({"role": "user", "content": [{"text": user_input}]})
resp = bedrock.converse(modelId=MODEL, system=[{"text": SYSTEM}],
                        messages=trim_to_budget(history, max_tokens=6000),
                        inferenceConfig={"maxTokens": 600, "temperature": 0.3})
history.append(resp["output"]["message"])         # keep assistant turn verbatim
save_history(session_id, history)                 # DynamoDB with TTL
```

### Skill 1.3.4 — Improving input quality
> *Enhance input data quality to improve FM response quality and consistency (for example, by using Amazon Bedrock to reformat text, Amazon Comprehend to extract entities, Lambda functions to normalize data).*

#### Why enhance input?
- Cleaner, more consistent input leads to **better retrieval** (embeddings of noisy text cluster badly), **more consistent outputs**, **fewer tokens** (lower cost and latency), and **less hallucination** (the context is unambiguous).
- Apply it at **two points**: **ingestion time** (documents for RAG / training data) and **request time** (user queries and inputs).

#### Decision table — which enhancement tool?
Pick by need; each tool is explained in the sections below.

| Need | Pick |
| --- | --- |
| Deterministic cleanup: dates, units, Unicode, whitespace, dedup | **Lambda** (or Glue for batch scale) |
| Standard entities, key phrases, language, sentiment, PII, at scale, cheap | **Amazon Comprehend** |
| Domain-specific entities/classes with labelled data | **Comprehend custom entity recognition / custom classification** |
| Clinical text | **Comprehend Medical** |
| Messy → clean/structured, arbitrary schema, summarisation | **Amazon Bedrock FM** (small model, batch inference for bulk) |
| Bulk language conversion | **Amazon Translate** |
| Extract fields from multimodal docs with a schema | **Bedrock Data Automation blueprints** |

#### Lambda normalisation (deterministic, cheap, fast)
| Problem | Normalisation |
| --- | --- |
| Mixed encodings, smart quotes, ligatures, zero-width chars | **Unicode normalisation (NFKC)**, strip control/zero-width characters (these also hide prompt injection) |
| HTML/boilerplate, nav menus, footers, cookie banners | Strip tags; remove repeated headers/footers/page numbers |
| Inconsistent dates/numbers/units | Convert to **ISO 8601**, standard units/currency, consistent decimal separators |
| Whitespace, broken line hyphenation from PDFs | Collapse whitespace, re-join hyphenated words, fix line breaks |
| Duplicates/near-duplicates | Hash-based exact dedup, MinHash/embedding similarity for near-dups |
| Abbreviations / synonyms ("PTO", "annual leave") | Domain dictionary expansion → consistent terminology |
| Casing / codes | Normalise product codes and IDs (`sku-001` → `SKU-001`) |
| Oversized input | Truncate/split by token budget at sentence/section boundaries |
- Deterministic rules come **first** (cheap, predictable). Use an FM only for what rules can't handle.

#### Amazon Comprehend (NLP enrichment without training)
- **Built-in APIs**: `DetectEntities` (PERSON, ORGANIZATION, LOCATION, DATE, QUANTITY, COMMERCIAL_ITEM, EVENT, TITLE, OTHER) · `DetectKeyPhrases` · `DetectDominantLanguage` · `DetectSentiment` / **targeted sentiment** · `DetectSyntax` · `DetectPiiEntities` / `ContainsPiiEntities` · **toxicity detection** · **prompt safety classification** (flags unsafe or injection-style prompts). Batch versions and async jobs exist for large S3 datasets. *(Toxicity/prompt-safety API names: verify in docs.)*
- 🎯 **PQ55** **Custom**: **custom classification** (e.g. route tickets by department) and **custom entity recognition** (e.g. policy numbers, part numbers) trained on your labelled data.
- 🎯 **PQ68** **Comprehend Medical**: medical entities, RxNorm/ICD-10-CM/SNOMED linking, PHI detection.
- How enrichment improves FM results:
  - Entities/key phrases go into **metadata** (`.metadata.json`) for **KB metadata filtering**, e.g. filter by `product = "X"`.
  - **Language detection** lets you **route** to Translate or the right model/embedding.
  - **Intent/classification** **routes** the request to the right prompt template, KB, or agent.
  - **Entity grounding**: add extracted entities explicitly into the prompt ("Customer: ACME; Order: 12345").
  - 🎯 **PQ31** **PII redaction** before the FM, logs, the vector store or fine-tuning (compliance). Once PII is in a vector store it can be retrieved and shown to users, and PII in training data can be **memorised** by the model; both are very hard to remove. Use Comprehend `DetectPiiEntities` or Transcribe PII redaction, with Macie to discover PII in S3.
- Comprehend vs FM: Comprehend is **cheaper, faster, deterministic, and scalable** for standard NLP tasks. An FM is more flexible (arbitrary schemas, reasoning) but costs more and its output varies.

```python
# Ingestion-time enhancement: Comprehend entities -> KB metadata sidecar (sketch)
ents = comprehend.detect_entities(Text=text[:5000], LanguageCode="en")["Entities"]  # mind per-call size limits
orgs = sorted({e["Text"] for e in ents if e["Type"] == "ORGANIZATION" and e["Score"] > 0.9})
meta = {"metadataAttributes": {"organizations": orgs, "language": "en", "doc_type": "contract"}}
s3.put_object(Bucket=B, Key=f"{key}.metadata.json", Body=json.dumps(meta))
```

> [!IMPORTANT]
> **🎯 Practice question 31 — messy financial documents + PII to redact → consistent FM inputs, LEAST overhead**
>
> - **Amazon Textract** → analyses document **structure** (tables, forms, key-value pairs) and returns **structured, machine-readable text + metadata**, which fixes the formatting inconsistencies.
> - **Amazon Comprehend PII detection and redaction** (built in, **no training**) → meets the PII policy before data reaches the FM.
> - Use that **structured metadata in the prompt** → better reasoning and fewer hallucinations.
> - ❌ **Macie:** *discovers and classifies* sensitive data in S3, but it's **not a document pre-processing or redaction step**.
> - ❌ **Comprehend *custom* entity recognition:** you have to train and maintain models, when built-in PII detection already covers it.
> - ❌ **Guardrails PII at inference:** filters during or after the call and **doesn't clean the input data**, which is the real cause.

> [!IMPORTANT]
> **🎯 Practice question 68 — summarise medical records: remove PHI, keep the medical terms, compare FMs, finish in 3 weeks**
>
> - **Bedrock Guardrails sensitive information filters** → **detect and mask PHI during generation**. Use custom regex for medical record numbers if needed.
> - **Bedrock Evaluations** → **automatically score and compare several FMs** against a **medical terminology dataset** to find the most consistent one.
> - ❌ **Comprehend Medical + a custom SageMaker summarisation model:** too slow to build in 3 weeks, and no automated comparison of FMs.
> - ❌ **Playground + manual review:** not systematic.
> - ❌ **Comprehend Medical *after* generation + custom Lambda scoring in DynamoDB:** finds PHI after the fact, and it's custom work.
> - 🔁 **Comprehend Medical** *is* right when the task is **extracting medical entities or ontology codes**. For **removing PHI inline with the least effort**, the answer is **Guardrails**.

> [!IMPORTANT]
> **🎯 Practice question 55 — track which product each customer is discussing (several per chat), high concurrency, encrypted, low cost**
>
> - **Comprehend *custom* entity recogniser** trained on **your product taxonomy** → reliably identifies *which* product is being discussed.
> - **Session state in DynamoDB** (current product, preferences) → single-digit-ms latency, **auto scaling / on-demand**, **encrypted at rest by default** (KMS), and **TTL** to expire sessions cheaply.
> - Inject the **session context** into each Bedrock prompt.
> - ❌ **Send the whole history in every prompt:** token costs climb, and nothing is stored encrypted.
> - ❌ **Lex intents per category + Lambda fallback:** Lex doesn't track context across multiple products, and it adds latency.
> - ❌ **Built-in `COMMERCIAL_ITEM` entities + only the latest entity:** not trained on your catalogue, and it loses earlier products.

#### Amazon Bedrock for reformatting and enhancement
- Use an FM (often a **small/cheap one**, e.g. Nova Micro/Lite or Claude Haiku) to:
  - **Rewrite messy text** (OCR noise, chat logs, emails) into clean, well-structured prose or Markdown.
  - **Convert unstructured → structured JSON** using a schema (tool use/function calling with a JSON schema gives the most reliable structure).
  - **Summarise/condense** long documents before they are included in context (contextual compression).
  - **Generate metadata**: titles, summaries, keywords, questions answered (hypothetical questions per chunk improve retrieval).
  - **Standardise terminology/tone**, and **translate** (or use Amazon Translate for deterministic, cheaper bulk translation).
  - **Query rewriting/expansion** at request time (details in Task 1.5).
- Run it at **ingestion via batch inference** (cheaper, offline) instead of on every request where possible.
- Risks: the FM can **alter facts or hallucinate** during rewriting. Use low temperature, instruct it to "preserve all facts and numbers", validate the output schema with Lambda, and spot-check with evaluation.

#### Request-time input enhancement pipeline
```text
User input ─► Lambda: normalise + length/token check
           ─► Comprehend: language, PII (redact), entities, intent/custom classifier
           ─► (optional) Bedrock small model: rewrite to standalone query / extract slots
           ─► Guardrails (ApplyGuardrail / inline): prompt-attack + denied topics
           ─► Route to prompt template / KB / agent ─► FM
```

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Rule-based quality checks (completeness, uniqueness, freshness) on tabular data in ETL, with alerts | **AWS Glue Data Quality** (DQDL rules → CloudWatch/EventBridge) |
| "Don't know which rules to write" / detect unexpected drift in data volume | Glue DQ **rule recommendations** / **ML anomaly detection** |
| Data scientists visually explore + clean a fine-tuning dataset, then productionise | **SageMaker Data Wrangler** → export to Processing job / Pipelines |
| Business analysts, no-code profiling and cleaning | **AWS Glue DataBrew** |
| Validate JSONL schema, token length, file type before ingestion | **Custom Lambda** validator (+ quarantine prefix) |
| Track invalid-record rate and alert on degradation | **CloudWatch custom metrics (PutMetricData/EMF) + alarms** |
| Call recordings must feed a summarisation FM | **Amazon Transcribe** (diarization) → Bedrock |
| Product names consistently mis-transcribed | Transcribe **custom vocabulary** / custom language model |
| Remove customer PII from call transcripts before the LLM | Transcribe **PII redaction** (or Comprehend DetectPiiEntities) |
| Scanned PDFs with forms and tables | **Textract** (FORMS/TABLES, async for multi-page) or **Bedrock Data Automation** |
| One service to extract insights from docs, images, audio, video with custom schema | **Bedrock Data Automation** (blueprints) |
| RAG over docs whose tables/charts matter | KB **FM parsing** or **BDA parser** (not default parser) |
| Need captions/structured info from images | **Bedrock multimodal FM** via Converse `image` block |
| Search images with text queries | **Multimodal embeddings** (Titan Multimodal G1 / Nova Multimodal / Cohere v4) |
| Questions require sums/aggregations over sales tables | **Text-to-SQL** (KB structured data store on Redshift / Athena), not embedding rows |
| Heavy custom preprocessing (GPU, Spark, > 15 min) | **SageMaker Processing** |
| Process millions of S3 files in parallel | **Step Functions Distributed Map** |
| Bulk offline captioning/summaries at lower cost | **Bedrock batch inference** (JSONL in S3) |
| Swap Bedrock models without changing request code | **Converse API** |
| Call an embedding model on Bedrock | **InvokeModel** with model-specific body |
| Cohere embeddings retrieve poorly | Wrong **`input_type`**: `search_document` for indexing, `search_query` for queries |
| Open-source LLM on SageMaker gives rambling output | Apply the **model's chat template** (or use Messages API container) |
| Large payloads / long-running inference on SageMaker | **Asynchronous Inference** (S3 input, up to 1 GB) |
| Chatbot forgets earlier turns | Re-send history in `messages`; persist in **DynamoDB** (TTL) |
| Long conversations hit context limits / cost rises | **Sliding window + summarisation**, prompt caching |
| Tool call loop errors | Return `toolResult` with the **same `toolUseId`** in the next **user** message |
| Add entities to documents for metadata filtering | **Comprehend DetectEntities** → `.metadata.json` → KB sync |
| Domain-specific entities (policy numbers) | **Comprehend custom entity recognition** |
| Inconsistent date/unit formats confuse the model | **Lambda normalisation** (ISO 8601, units) |
| OCR'd text is noisy and unstructured | **Bedrock (small FM) reformat** to clean Markdown/JSON, low temperature |
| Multilingual corpus, English-only embedding model | Comprehend language detection → **Amazon Translate**, or multilingual embedding model |

## Flashcards

<details>
<summary><b>Q:</b> What language do you use to write AWS Glue Data Quality rules, and name four rule types.</summary>

DQDL (Data Quality Definition Language). Examples: `IsComplete`, `IsUnique`, `Completeness`, `ColumnValues`, `ColumnLength`, `RowCount`, `DataFreshness`, `CustomSql`.

*D1 · Task 1.3 · Glue Data Quality*
</details>

<details>
<summary><b>Q:</b> What are the two places Glue Data Quality can run, and what does each validate?</summary>

Data Catalog checks validate data **at rest** in catalogued tables. ETL job checks (`EvaluateDataQuality` transform) validate data **in flight** and can route rows by pass/fail.

*D1 · Task 1.3 · Glue Data Quality*
</details>

<details>
<summary><b>Q:</b> How do Glue Data Quality results trigger alerts or stop a pipeline?</summary>

They publish metrics to CloudWatch and events to EventBridge. Use CloudWatch alarms or EventBridge rules to notify via SNS or fail the Step Functions workflow.

*D1 · Task 1.3 · DQ monitoring*
</details>

<details>
<summary><b>Q:</b> When would you choose SageMaker Data Wrangler over Glue Data Quality?</summary>

When data scientists need visual, low-code exploration and transformation (300+ transforms, Data Quality and Insights report) of an ML/fine-tuning dataset, and then export the flow to a SageMaker Processing job or Pipeline. Glue DQ is for rule-based checks inside data engineering pipelines.

*D1 · Task 1.3 · Data Wrangler*
</details>

<details>
<summary><b>Q:</b> Name four GenAI-specific checks that typically need a custom Lambda validator.</summary>

JSONL/JSON schema validation, token-count limits, PII/secret detection, language detection, file type/size checks, content-hash deduplication.

*D1 · Task 1.3 · Lambda validation*
</details>

<details>
<summary><b>Q:</b> What is the cheapest way to publish high-volume custom metrics from Lambda?</summary>

CloudWatch Embedded Metric Format (EMF). You write structured logs and CloudWatch extracts the metrics, with no `PutMetricData` calls.

*D1 · Task 1.3 · CloudWatch*
</details>

<details>
<summary><b>Q:</b> Call-centre audio must be summarised by an FM, with speaker turns and no PII. Which service and features?</summary>

Amazon Transcribe with speaker diarization (or channel identification) and PII redaction. Format the output into Agent/Customer turns and send it to Bedrock. Transcribe Call Analytics can also generate summaries.

*D1 · Task 1.3 · Transcribe*
</details>

<details>
<summary><b>Q:</b> Transcripts misspell internal product names. What is the fix?</summary>

Transcribe custom vocabulary, or a custom language model trained on domain text.

*D1 · Task 1.3 · Transcribe*
</details>

<details>
<summary><b>Q:</b> What is Bedrock Data Automation and what are its two output types?</summary>

A managed service that extracts insights from documents, images, audio and video with one API. It has standard output (predefined insights per modality) and custom output defined by blueprints (your extraction schema). It can also act as a Knowledge Base parser.

*D1 · Task 1.3 · Bedrock Data Automation*
</details>

<details>
<summary><b>Q:</b> Which Textract features would you use for key-value pairs, tables, natural-language field questions, and document structure?</summary>

`FORMS`, `TABLES`, `QUERIES`, `LAYOUT` (via `AnalyzeDocument`). Multi-page PDFs need the async `StartDocumentAnalysis`.

*D1 · Task 1.3 · Textract*
</details>

<details>
<summary><b>Q:</b> Users ask "total revenue by region last quarter" over large sales tables. Embed the rows or not?</summary>

Don't rely on embeddings. Use text-to-SQL / structured retrieval (e.g. a Bedrock Knowledge Base structured data store over Redshift, or Athena) for exact aggregations.

*D1 · Task 1.3 · Tabular data*
</details>

<details>
<summary><b>Q:</b> When should preprocessing move from Lambda to SageMaker Processing?</summary>

When jobs run longer than 15 minutes, need more than 10 GB memory or GPUs, need custom ML libraries or distributed Spark, or need to be a SageMaker Pipelines step with lineage.

*D1 · Task 1.3 · SageMaker Processing*
</details>

<details>
<summary><b>Q:</b> List the main Converse API content block types.</summary>

`text`, `image`, `document`, `video`, `toolUse`, `toolResult`, `guardContent`, and on supported models `reasoningContent` and `cachePoint`.

*D1 · Task 1.3 · Converse format*
</details>

<details>
<summary><b>Q:</b> Give three causes of a Converse `ValidationException` related to message formatting.</summary>

First message not from `user`, roles not alternating, `toolUse` without a matching `toolResult`, unsupported document format or invalid document name, image too large, `maxTokens` above the model's limit.

*D1 · Task 1.3 · Converse format*
</details>

<details>
<summary><b>Q:</b> How does the tool-use loop work in Converse?</summary>

The model returns `stopReason: tool_use` with a `toolUse` block (`toolUseId`, `name`, `input`). The app runs the tool, appends the assistant message, then sends a `user` message with a `toolResult` using the same `toolUseId`, and calls Converse again.

*D1 · Task 1.3 · Tool use formatting*
</details>

<details>
<summary><b>Q:</b> Which Bedrock API do you use for Titan or Cohere embeddings, and what Cohere parameter is commonly misused?</summary>

InvokeModel with a model-specific body. Cohere's `input_type` must be `search_document` when indexing and `search_query` when querying.

*D1 · Task 1.3 · InvokeModel bodies*
</details>

<details>
<summary><b>Q:</b> What does a Bedrock batch inference input record look like?</summary>

One JSON object per line in a JSONL file in S3: `{"recordId": "...", "modelInput": { ...model-specific InvokeModel body... }}`. Output is written as JSONL to S3.

*D1 · Task 1.3 · Batch inference*
</details>

<details>
<summary><b>Q:</b> An open-source LLM on a SageMaker TGI endpoint gives rambling, low-quality answers to raw prompts. Likely cause?</summary>

The prompt doesn't use the model's chat template (special tokens like `[INST]` or Llama header tokens). Apply the template, or use a container that accepts the Messages API and applies it server-side.

*D1 · Task 1.3 · SageMaker payloads*
</details>

<details>
<summary><b>Q:</b> Inference payloads are hundreds of MB and processing takes minutes. Which SageMaker option?</summary>

Asynchronous Inference. Input is in S3 (up to ~1 GB), processing up to ~1 hour, with an SNS notification and scale to zero.

*D1 · Task 1.3 · SageMaker inference options*
</details>

<details>
<summary><b>Q:</b> How do you keep long chat conversations within the context window and budget?</summary>

Store history in DynamoDB (with TTL). Send a sliding window of recent turns plus a summary of older turns, budget tokens for context and output, and use prompt caching for static prefixes.

*D1 · Task 1.3 · Conversation formatting*
</details>

<details>
<summary><b>Q:</b> How can Amazon Comprehend improve RAG retrieval precision?</summary>

Extract entities and key phrases (and custom entities) at ingestion and write them to `.metadata.json` sidecars. Then use KB metadata filters at query time. Language detection and classification can also route queries.

*D1 · Task 1.3 · Comprehend enrichment*
</details>

<details>
<summary><b>Q:</b> Comprehend vs a Bedrock FM for entity extraction: when do you pick each?</summary>

Comprehend for standard entity types or trained custom entities: cheap, fast, deterministic, scalable. A Bedrock FM for arbitrary schemas, reasoning, or messy text needing restructuring, at higher cost and with variable output.

*D1 · Task 1.3 · Input enhancement*
</details>

<details>
<summary><b>Q:</b> Name five Lambda normalisation steps that improve FM input consistency.</summary>

Unicode NFKC and stripping zero-width/control characters, HTML/boilerplate removal, dates to ISO 8601 and standard units, whitespace and hyphenation fixes, dedup by content hash, abbreviation/synonym expansion, token-budget truncation at sentence boundaries.

*D1 · Task 1.3 · Normalisation*
</details>

<details>
<summary><b>Q:</b> What are the risks of using an FM to reformat ingestion text, and how do you mitigate them?</summary>

The FM may alter facts or hallucinate. Use low temperature, instruct it to preserve all facts and numbers, enforce a JSON schema (tool use) and validate it with Lambda, run bulk jobs via batch inference, and spot-check with evaluation.

*D1 · Task 1.3 · Bedrock reformatting*
</details>

<details>
<summary><b>Q:</b> Why redact PII before embedding or fine-tuning rather than after?</summary>

PII in a vector store can be retrieved and shown to users, and PII in training data can be memorised by the model weights. Both are hard to remove. Redact upstream with Comprehend/Transcribe, and use Macie to discover PII in S3.

*D1 · Task 1.3 · PII handling*
</details>

---
[← Back to Domain 1](README.md)
