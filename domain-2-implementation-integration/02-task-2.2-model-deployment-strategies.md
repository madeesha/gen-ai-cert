[← Back to Domain 2](README.md)

# 🚀 Task 2.2 · Implement model deployment strategies

> **Exam domain:** D2 · Implementation and Integration (26%)
>
> **Skills covered**
> - **2.2.1** Deploy FMs based on application needs and performance requirements
> - **2.2.2** Deploy FM solutions that address the LLM-specific challenges that differ from traditional ML deployments
> - **2.2.3** Optimise FM deployment to balance performance against resource requirements
>
> **Related:** FM selection, Cross-Region Inference resilience and the customisation lifecycle (LoRA, Model Registry, rollback) are in [Task 1.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md). Runtime model routing is in [Task 2.4](04-task-2.4-fm-api-integrations.md). The high-level notes are in [02 · Foundation models, Bedrock APIs & inference](<../../study-notes/Certifications/AWS/AIP-C01 - Gen AI Pro/02-foundation-models-bedrock-apis-inference.md>).

## Key points
- **First decision: managed API or self-hosted?** Bedrock unless you need your own engine, container, GPU choice or a model Bedrock can't serve → SageMaker AI endpoints (2.2.1).
- **Bedrock: pick the inference mode by traffic shape.** Spiky / unknown → **on-demand**; steady and must not throttle → **Provisioned Throughput** or the **Reserved tier**; can wait → **Flex** or **batch**; latency-critical without a reservation → **Priority** (2.2.1).
- **Own open-weight model but want Bedrock's serverless API → Custom Model Import.** It doesn't take embedding models or batch jobs (2.2.1).
- **SageMaker: pick the option by latency and payload.** Interactive → real-time; small CPU model with idle gaps → serverless (**no GPUs**); big payloads or slow jobs with scale-to-zero → async; offline datasets → Batch Transform (2.2.1).
- **Lambda is glue, not a host** for FMs (2.2.1).
- **LLMs are bound by GPU memory, not CPU:** weights + KV cache decide the instance and the tensor-parallel degree (2.2.2).
- **Throughput comes from the serving engine** (LMI / vLLM / TGI continuous batching, quantisation) and **cold starts from model loading**; scale on concurrency, not CPU (2.2.2).
- **Optimise by right-sizing first:** the smallest model that passes evaluation, then cascading, caching and the cheapest purchase option (2.2.3).

## Notes

### Skill 2.2.1 — Deploy FMs based on application needs and performance requirements

> *Deploy FMs based on specific application needs and performance requirements (for example, by using Lambda functions for on-demand invocation, Amazon Bedrock provisioned throughput configurations, SageMaker AI endpoints to implement hybrid solutions).*

#### The deployment decision: managed API vs self-hosted

| Requirement | Choose | Why |
| --- | --- | --- |
| Fastest time to value, pay per token, no infrastructure | **Bedrock on-demand** | Serverless; quotas are tokens per minute (TPM) and requests per minute (RPM) per model per Region |
| Steady high volume, guaranteed throughput, predictable latency | **Bedrock Provisioned Throughput** or the **Reserved tier** | Dedicated or reserved capacity, not shared on-demand pools |
| Spiky traffic that occasionally exceeds Region quota | **Cross-Region inference profile** (`us.`, `eu.`, `apac.`, `global.`) | Spreads load across Regions (see [Task 1.2](../domain-1-fm-integration-data-compliance/02-task-1.2-select-configure-fms.md)) |
| Millions of offline prompts (classification, summarisation backfill) | **Bedrock batch inference** | Asynchronous S3 → S3 job at a discounted price |
| Customer-facing, latency-critical, no reservation | **Priority tier** | Requests are prioritised ahead of Standard and Flex |
| Evaluations, summarisation, agent background jobs that can wait | **Flex tier** | Discount in exchange for longer processing |
| Open-weight model you fine-tuned elsewhere, but you want Bedrock's serverless API | **Custom Model Import** | Bedrock manages hosting; you call it with InvokeModel |
| Model not on Bedrock, custom container, custom engine, specific GPU, VPC isolation, data capture | **SageMaker AI real-time endpoint** | Full control |
| Model from the **Bedrock Marketplace** | Deploys to a **SageMaker-managed endpoint** that you size yourself | You still call it through Bedrock APIs |
| Data must stay on-premises or at the edge | Outposts, Wavelength or Local Zones | See Task 2.3 |

#### Amazon Bedrock inference modes in detail

**On-demand**
- Billed per input and output token (image and embedding models price differently).
- **Quotas are per account, per Region, per model.** Going over them returns `ThrottlingException` (HTTP 429). Handle this with retries plus exponential backoff and jitter (see Task 2.4), a cross-Region inference profile, or a quota increase.
- APIs: `Converse` / `ConverseStream` (one schema for every model, recommended) and `InvokeModel` / `InvokeModelWithResponseStream` (each model's native body).

**Provisioned Throughput (PT)**
- You buy **model units (MUs)**. Each MU gives a set number of tokens per minute; the figure depends on the model (verify in docs).
- Terms: **no commitment** (hourly, can be deleted at any time), or **1-month** / **6-month** commitments at lower rates.
- You invoke it with the **provisioned model ARN** as `modelId`.
- Historically, PT was **required to use fine-tuned (custom) Bedrock models**. Some customised models now support **on-demand** inference (verify in docs for each model).
- **Batch inference is not supported for provisioned models.**

**Service tiers** (verified from the Bedrock User Guide)

| Tier | `service_tier` value | Behaviour | Price |
| --- | --- | --- | --- |
| **Reserved** | `reserved` | Reserved input/output TPM for 1 or 3 months. Overflow goes to Standard. Targets **99.5%** uptime. Minimum **100K input TPM / 10K output TPM**. Set up through your AWS account team. | Fixed price per 1K TPM, billed monthly |
| **Priority** | `priority` | Prioritised over Standard and Flex; fastest responses | Premium over Standard |
| **Standard** | `default` (or leave it out) | Consistent performance for everyday work | Standard on-demand |
| **Flex** | `flex` | Longer processing time is acceptable | Discounted |

- Priority, Standard and Flex **share one on-demand quota**. Reserved capacity is **separate**.
- CloudWatch has `ServiceTier` and **`ResolvedServiceTier`** dimensions. ResolvedServiceTier shows the tier that *actually* served the request, for example after Reserved overflowed to Standard.
- Tokens written to the prompt cache (`CacheWriteInputTokens`) count towards Reserved TPM. Include them when you size a reservation.
- IAM can restrict which tiers a principal may use.

```json
{
  "modelId": "us.anthropic.claude-...",
  "messages": [{"role": "user", "content": [{"text": "Summarise this ticket"}]}],
  "service_tier": "flex"
}
```

**Latency-optimised inference**
- Some models offer a latency-optimised option (e.g. through supported cross-Region profiles) for lower TTFT, set per request with `performanceConfig: {"latency": "optimized"}`. Check which models support it (verify in docs).
- → Full detail: [Task 4.2 · Skill 4.2.1](../domain-4-operational-efficiency-optimization/02-task-4.2-application-performance.md#skill-421--responsive-ai-systems-latency-vs-cost-trade-offs).

**Batch inference**
- Create the job with `CreateModelInvocationJob`. Input is **JSONL** in S3, with one `{"recordId": "...", "modelInput": {...}}` per line. Output is JSONL written to an S3 prefix.
- `modelInput` can follow either the InvokeModel or the Converse format. With the **Converse** invocation type, **tool calling is not supported**: records containing `toolConfig` fail with a 400 error, so use the InvokeModel type with each model's native format instead. Structured outputs work with both types.
- Price is roughly **50% below on-demand** for supported models (verify on the pricing page). There is a **minimum record count per job**, and quotas limit job size (verify in docs).
- Use **EventBridge** job state-change events instead of polling.
- Not available for provisioned models or Custom Model Import models.

**Custom Model Import (CMI)** (verified)
- Brings in your **fine-tuned**, **domain- or language-adapted**, or **trained-from-scratch** open-weight models.
- Required files in S3: `*.safetensors`, `config.json`, `tokenizer_config.json`, `tokenizer.json`, `tokenizer.model`. These are Hugging Face formats, built with transformers 4.51.3.
- Supported architectures: Mistral, Mixtral, Flan-T5, Llama 2/3/3.1/3.2/3.3 and Mllama, GPTBigCode, Qwen2/2.5/2-VL/2.5-VL/Qwen3, and GPT-OSS (20B and 120B, us-east-1 only).
- Limits:
  - Weights under **200 GB** for text models and under **100 GB** for multimodal models.
  - Context under **128K**.
  - **No embedding models.**
  - No **batch inference** and no **CloudFormation**.
  - Qwen3 models don't support **Converse**.
- Regions: us-east-1, us-east-2, us-west-2 and eu-central-1 (verify as Regions get added).
- Invoke with `InvokeModel` / `InvokeModelWithResponseStream` on demand.
- Billing is in **Custom Model Units**, charged in **5-minute windows** while copies are active. Copies **scale to zero** when idle, so the **first call after an idle period has a cold start** (verify the exact window on the pricing page).
- Exam cue: *"fine-tuned Llama in SageMaker; want serverless Bedrock invocation, Guardrails and Knowledge Bases without managing endpoints"* → **CMI**.

#### SageMaker AI inference options 🎯 PQ52

| Option | Latency | Payload / time | Scale to 0 | GPU | Use for |
| --- | --- | --- | --- | --- | --- |
| **Real-time endpoint** | Milliseconds | ~**6 MB** request, **60 s** for `InvokeEndpoint` (verify; streaming responses can run longer) | No (inference components can, verify) | ✅ | Interactive chat, steady traffic |
| **Serverless Inference** | Milliseconds plus a **cold start** | 1–6 GB memory, 10 GB image, 5 GB disk | ✅ | ❌ **no GPUs** | Small models (embeddings, classifiers) with spiky or idle traffic |
| **Asynchronous Inference** | Near real time (queued) | **1 GB** payload, **1 hour** processing | ✅ (scale instance count to 0) | ✅ | Long prompts or documents, slow generations, bursty queues |
| **Batch Transform** | Offline | Whole datasets in S3 | No endpoint | ✅ | Scheduled bulk scoring or generation |

**Serverless Inference** facts (verified):
- Memory options are 1024–6144 MB in 1 GB steps. vCPUs scale with the memory you choose.
- Max concurrency is **200 per endpoint**, with **50 serverless endpoints per Region**. The account-wide concurrency pool is 1000 or 500 depending on the Region.
- **Provisioned Concurrency** keeps instances warm and can be autoscaled with Application Auto Scaling on a target metric or a schedule.
- The cold-start metric is **`OverheadLatency`**.
- **Not supported:** GPUs, Marketplace model packages, private Docker registries, multi-model endpoints, **VPC configuration**, network isolation, **data capture**, multiple production variants, Model Monitor and inference pipelines.
- A serverless endpoint can be converted to real-time (one-way). A real-time endpoint **cannot** be converted to serverless (you get a `ValidationError`).
- ⚠️ Exam trap: *"serverless SageMaker for a 7B LLM"* is wrong. There is no GPU and only 6 GB of memory. Use Bedrock on-demand, CMI, or async with scale-to-zero instead.

**Asynchronous Inference** facts (verified):
- Set `AsyncInferenceConfig` in the endpoint config. **The endpoint then only accepts async calls.**
- Call `InvokeEndpointAsync` with an **S3 input location**. It returns an output location right away. Results land in S3, and success or error notifications can go to **SNS**.
- Autoscale to **0 instances** when idle, using the `ApproximateBacklogSizePerInstance` metric (verify the metric name). This is the cheapest SageMaker way to host a GPU LLM for bursty internal jobs.

**Streaming on SageMaker:**
- `InvokeEndpointWithResponseStream` returns an `EventStream` of `PayloadPart` events.
- Payload parts can split or merge JSON chunks, so **buffer and parse line by line**.
- **Bidirectional streaming** uses `InvokeEndpointWithBidirectionalStream` over HTTP/2 on port 8443, through the experimental new Python SDK. A connection can stay open for up to **30 minutes**. It suits voice and real-time conversations.
- The AWS CLI doesn't support streaming.

> [!WARNING]
> **🎯 Practice question 52 — real-time on-demand FMs + steady high-throughput batch + hybrid cloud/on-prem for residency (choose 2)**
>
> - ✅ **SageMaker AI real-time endpoints** for low-latency, on-demand inference, **+ SageMaker Neo** to **compile and optimise models for edge and on-premises hardware** (the hybrid part). **Lambda** orchestrates calls across both.
> - ✅ **Bedrock Provisioned Throughput** → **reserved, consistent capacity** for heavy batch workloads.
> - ❌ **Bedrock on-demand "auto scaling":** elastic, but **no reserved capacity** and no on-premises option.
> - ❌ **SageMaker *asynchronous* endpoints for low latency:** they queue requests, so they're for delayed responses.
> - ❌ **SageMaker JumpStart:** a way to deploy models in the cloud, with no hybrid orchestration.
> - ℹ️ **Neo** compiles a trained model for a target platform (cloud instance types or edge devices). Other ways to run on-premises: **Outposts** (see Task 2.3). SageMaker Edge Manager has been discontinued, so check current edge tooling in the docs (verify in docs).

#### Multi-model, multi-container and inference components
Once one model per endpoint wastes GPUs, these options share a fleet.

| Feature | What it does | When |
| --- | --- | --- |
| **Multi-model endpoint (MME)** | Many models in **one container type**, loaded on demand from S3 and cached; GPU MMEs use Triton | Thousands of small or tenant-specific models with uneven traffic (cold load on cache miss) |
| **Multi-container endpoint** | Up to 15 different containers on one endpoint, called directly or chained serially (verify the limit) | Different frameworks behind one endpoint |
| **Inference components (IC)** | Each model is a component with its own **accelerator count, CPU and memory** reservation and **its own copy count and autoscaling**; many FMs or LoRA adapters share one endpoint's instances | **Packing several LLMs onto a GPU fleet** for better utilisation and lower cost; scale models separately; IC copies can scale to zero (verify) |

```python
# Inference component: give 2 GPUs to one model on a shared endpoint
sm.create_inference_component(
    InferenceComponentName="llama-8b-ic",
    EndpointName="shared-llm-endpoint",
    VariantName="AllTraffic",
    Specification={
        "ModelName": "llama-8b",
        "ComputeResourceRequirements": {
            "NumberOfAcceleratorDevicesRequired": 2,
            "MinMemoryRequiredInMb": 32768,
        },
    },
    RuntimeConfig={"CopyCount": 1},
)
```

#### Lambda for on-demand invocation
- Lambda is **glue code**, not the host. It calls Bedrock (or a SageMaker endpoint), adds the prompt template, validation, auth context and guardrails, and returns the result.
- Limits: max **15-minute** timeout, **10,240 MB** memory, **no GPU**, 6 MB synchronous request/response payload.
- **Response streaming** works through **Lambda function URLs** or the `InvokeWithResponseStream` API (Node.js managed runtimes natively; other runtimes through custom runtimes or the web adapter). Check the streamed size limit in the docs.
- API Gateway REST APIs have a **29-second** default integration timeout. It can be raised for Regional and private REST APIs, at the cost of throttle quota (verify in docs). For long generations, stream (WebSocket API, Lambda URL) or go asynchronous (SQS + polling or callback).
- Pattern: `API Gateway → Lambda (validate, template, auth) → Bedrock Converse → Lambda → client`, with the model ID read from **AppConfig** or Parameter Store (see Task 1.2.2).
- Cold starts: Lambda's own init time is small next to model latency. Use **provisioned concurrency** or **SnapStart** (Java, Python, .NET) if the p99 matters.

#### Hybrid solutions
- **Bedrock + SageMaker hybrid:**
  - General reasoning and chat → a Bedrock serverless FM.
  - A domain-specific fine-tuned or proprietary model → a SageMaker endpoint.
  - A small façade (Lambda or API Gateway) routes between them, so callers see one API.
- **Bedrock features on SageMaker-hosted models:**
  - **Guardrails**: the `ApplyGuardrail` API works on *any* model's text input and output, including SageMaker and third-party models.
  - **Knowledge Bases**: the `Retrieve` API returns chunks you can pass to a SageMaker model.
  - **Bedrock Marketplace** models run on SageMaker infrastructure but are called through Bedrock.
- **Cloud + on-premises hybrid:** sensitive inference runs on Outposts or self-managed GPUs, and non-sensitive work goes to Bedrock (Task 2.3.4).
- **Cost hybrid:** a Bedrock PT or Reserved baseline for steady load, with on-demand or cross-Region overflow for peaks.

---

### Skill 2.2.2 — Address the LLM-specific challenges of deployment

> *Deploy FM solutions by addressing unique challenges of large language models (LLMs) that differ from traditional ML deployments (for example, by implementing container-based deployment patterns that are optimized for memory requirements, GPU utilization, and token processing capacity, by following specialized model loading strategies).*

#### Why LLM serving is different from classic ML

| Aspect | Traditional ML model | LLM / FM |
| --- | --- | --- |
| Size | MB to a few GB | **Tens to hundreds of GB** |
| Bottleneck | CPU compute | **GPU memory (HBM) capacity and bandwidth** |
| Inference | One forward pass | **Autoregressive**: one pass per output token (prefill + decode) |
| Latency metric | p50/p99 per request | **Time to first token (TTFT)**, **inter-token latency / tokens per second**, end-to-end time |
| Throughput metric | Requests per second | **Tokens per second** (input and output counted separately) |
| State during a request | None | **KV cache** grows with every token |
| Cold start | Seconds | **Minutes** (download and load 100+ GB of weights) |
| Scaling unit | Small CPU instance | Multi-GPU instance (expensive, capacity-limited) |
| Payload | Fixed features | Variable-length prompts, so **token limits** and context windows matter |

#### GPU memory sizing (exam-friendly arithmetic)
- **Weights** ≈ parameters × bytes per parameter:

  | Precision | Bytes/param | 7–8B model | 13B | 70B |
  | --- | --- | --- | --- | --- |
  | FP32 | 4 | ~28–32 GB | ~52 GB | ~280 GB |
  | FP16 / BF16 | 2 | ~14–16 GB | ~26 GB | **~140 GB** |
  | INT8 / FP8 | 1 | ~7–8 GB | ~13 GB | ~70 GB |
  | INT4 (AWQ/GPTQ) | 0.5 | ~4 GB | ~7 GB | ~35 GB |

- **KV cache** ≈ 2 (K and V) × layers × KV heads × head dimension × sequence length × batch size × bytes. It grows **linearly with context length and with concurrent requests**. Long-context chat at high concurrency can need more memory than the weights do.
- Rule of thumb: leave about **20–30% headroom** over the weights for KV cache, activations and CUDA overhead.
- Sizing examples:
  - An 8B model in FP16 fits on one 24 GB GPU (g5, g6) with modest context.
  - A 70B model in FP16 needs about 140 GB+, so **≥ 4 × 80 GB** or **8 × 40 GB** GPUs, with tensor parallelism across them.

#### Accelerated instance families (SageMaker/EC2)

| Family | Accelerator | Memory per device | Typical LLM role |
| --- | --- | --- | --- |
| **g5** | NVIDIA A10G | 24 GB (up to 8 GPUs = 192 GB on g5.48xlarge) | 7–13B inference, cost-effective |
| **g6** | NVIDIA L4 | 24 GB | Cheaper small-model inference |
| **g6e** | NVIDIA L40S | 48 GB | Mid-size models, FP8 support |
| **p4d / p4de** | NVIDIA A100 | 40 GB / 80 GB × 8 | Large models (70B), training |
| **p5 / p5e** | NVIDIA H100 / H200 | 80 GB / 141 GB × 8 | Largest models, highest throughput, FP8 |
| **inf2** | AWS **Inferentia2** | 32 GB per chip (up to 12 chips) | **Lowest cost per inference** for supported models; needs **Neuron SDK** compilation |
| **trn1 / trn2** | AWS **Trainium** | — | Cost-efficient **training** and fine-tuning (trn2 also serves inference) |

- Exam cues:
  - *"lowest inference cost; willing to compile; supported architecture"* → **Inferentia2 (inf2) + Neuron**.
  - *"lowest training cost"* → **Trainium**.
- GPU capacity can be scarce. Reserve it with **SageMaker training plans** or EC2 Capacity Blocks for ML (verify which apply to inference), or use Bedrock so AWS handles capacity.

#### Parallelism strategies

| Strategy | What is split | When |
| --- | --- | --- |
| **Tensor parallelism (TP)** | Each layer's weight matrices across GPUs **within one instance** (fast NVLink) | Model doesn't fit on one GPU. The standard choice for inference. `OPTION_TENSOR_PARALLEL_DEGREE=8` |
| **Pipeline parallelism (PP)** | Groups of layers on different GPUs or nodes | Very large models across nodes; adds pipeline "bubbles" |
| **Data parallelism / replicas** | Whole model copies | Scale throughput once one copy fits; multiple IC copies or instances |
| **Expert parallelism** | MoE experts across devices | Mixtral-style MoE models |

#### Inference engines and containers on SageMaker

| Container | Engines / features | Notes |
| --- | --- | --- |
| **LMI (Large Model Inference) DLC** (DJL Serving) | **vLLM**, **TensorRT-LLM**, **LMI-Dist**, **Transformers NeuronX** (Inferentia/Trainium) | AWS-maintained. Set up with `serving.properties` or `OPTION_*` environment variables |
| **Hugging Face TGI** (Text Generation Inference) DLC | Continuous batching, TP, quantisation, streaming | `HF_MODEL_ID`, `SM_NUM_GPUS`, `MAX_INPUT_TOKENS` / `MAX_TOTAL_TOKENS` environment variables |
| **Triton Inference Server** | Multi-framework, ensembles, GPU MMEs | Mixed model types |
| **Bring your own container (BYOC)** | Anything | Must implement `/invocations` and `/ping` on port 8080 |

Common LMI settings (environment variable form):

```properties
HF_MODEL_ID=s3://my-bucket/llama-3-70b/         # or a Hugging Face Hub ID
OPTION_TENSOR_PARALLEL_DEGREE=8                  # split across 8 GPUs ("max" = all)
OPTION_ROLLING_BATCH=vllm                        # continuous batching engine (vllm | lmi-dist | trtllm)
OPTION_MAX_ROLLING_BATCH_SIZE=64                 # concurrent sequences in a batch
OPTION_MAX_MODEL_LEN=8192                        # caps context, which caps KV-cache memory
OPTION_QUANTIZE=awq                              # or fp8 / gptq (engine-dependent)
OPTION_DTYPE=fp16
```

#### Throughput techniques (GPU utilisation and token capacity)
- **Continuous (rolling / in-flight) batching:**
  - New requests join the running batch **at each decode step** instead of waiting for the whole batch to finish.
  - This greatly increases tokens per second and GPU utilisation compared with static batching.
- **PagedAttention** (vLLM) stores the KV cache in fixed-size *pages*. This avoids fragmentation, so more concurrent sequences fit in the same memory.
- **Quantisation** (AWQ, GPTQ, FP8, INT8) means less memory, fewer GPUs and higher throughput, with a small quality loss. Always **evaluate accuracy** after quantising.
- **Speculative decoding:** a small draft model proposes tokens and the large model verifies them in one pass. This lowers latency without changing output quality.
- **Prefix (prompt) caching** reuses the KV cache for shared system prompts and documents, cutting TTFT and cost. On Bedrock this is **prompt caching** with cache checkpoints.
- **Cap max tokens:** set `max_tokens`/`maxTokens` and a maximum model length to bound memory and latency per request.
- **SageMaker inference optimisation toolkit:** applies speculative decoding, quantisation and compilation (e.g. for Neuron or TensorRT-LLM) as an optimisation job before deployment (verify current options).

#### Model loading strategies (cold start and scale-out)
Loading 100+ GB of weights dominates startup time. The fixes:

| Technique | Effect |
| --- | --- |
| **Store weights in S3 in the same Region** (not the Hugging Face Hub at runtime) | No internet dependency, faster and private (works with VPC-only setups) |
| **Pre-sharded / pre-compiled weights** (shard for the TP degree ahead of time; pre-compile TensorRT-LLM engines or Neuron artifacts) | No sharding or compilation at startup |
| **SageMaker Fast Model Loader** | **Streams weights from S3 straight to GPU memory** in parallel, instead of download → disk → load. Much faster scale-out for large models (verify the exact speed-up and supported containers, LMI) |
| **Container caching** | Container images are pre-cached on instances, so scale-out skips the image pull |
| **`safetensors` format** | Fast, memory-mapped and safe (no pickle code execution) |
| **Minimum instance or copy count ≥ 1** and **scheduled scaling** | Avoid scale-from-zero for latency-sensitive paths; pre-warm before known peaks |
| **Health-check / startup timeouts** | Raise `ContainerStartupHealthCheckTimeoutInSeconds` and `ModelDataDownloadTimeoutInSeconds` for large models, otherwise endpoint creation fails while weights are still loading |
| **Uncompressed model artifacts** (`ModelDataSource` with S3 prefix and `CompressionType: None`) | No tar.gz extraction of a 100 GB archive |

- ⚠️ Gotcha: endpoint creation fails with a health-check timeout on a large model → **raise the container startup health-check timeout** and the model download timeout. Don't jump to a bigger instance.
- ⚠️ Gotcha: out-of-memory (OOM) under load, but not at startup → the **KV cache is growing**. Lower the maximum batch size or model length, quantise, or raise the TP degree.

#### Autoscaling LLM endpoints
With loading under control, the last question is when to add or remove capacity. SageMaker real-time endpoints scale with **Application Auto Scaling**.
- **Pick the right signal.** CPU utilisation is a **poor** signal for LLMs, and even invocations per instance is weak: one request may take 0.5 s or 30 s depending on output length, so **concurrency** reflects load better.

  | Scope | Predefined target-tracking metrics |
  | --- | --- |
  | Endpoint variant | `SageMakerVariantInvocationsPerInstance`, **`SageMakerVariantConcurrentRequestsPerModelHighResolution`** |
  | Inference component | `SageMakerInferenceComponentInvocationsPerCopy`, **`SageMakerInferenceComponentConcurrentRequestsPerCopyHighResolution`** |

  - The **high-resolution concurrency** metrics enable **faster scale-out** on bursts (verify metric names in docs).
  - **GPU utilisation / GPU memory** → publish as **custom CloudWatch metrics** and use target tracking or step scaling (they are not predefined).
- **Policy types:**
  - **target tracking** (the default choice)
  - **step scaling** (aggressive reaction to big spikes)
  - **scheduled scaling** (diurnal or business-hours traffic: pre-warm before 9 am, scale in at night)
- **Cooldowns** (default 300 s for target tracking), and `DisableScaleIn` for scale-out-only policies.
- **Scale out before demand.** Scaling out is slow (minutes, because of the container pull and weight loading), so use scheduled scaling before predictable peaks, a higher minimum capacity or one warm copy, and the loading fixes above.
- **Inference components** scale model **copies** independently on a shared fleet and can scale idle models to zero copies (verify current support).
- **Spiky or idle-heavy traffic:** SageMaker Serverless (CPU only, small models) or Async (scale to zero, queue-backed); Bedrock on-demand is the "no capacity management" default.
- Deploy safely with blue/green, canary or linear **deployment guardrails** and auto-rollback on CloudWatch alarms. The lifecycle side is covered in Task 1.2.4.

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace sagemaker \
  --resource-id endpoint/llm-endpoint/variant/AllTraffic \
  --scalable-dimension sagemaker:variant:DesiredInstanceCount \
  --policy-name concurrency-tt --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 4,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "SageMakerVariantConcurrentRequestsPerModelHighResolution"
    },
    "ScaleInCooldown": 600, "ScaleOutCooldown": 60
  }'
```

---

### Skill 2.2.3 — Optimise FM deployment for performance and resource use

> *Develop optimized FM deployment approaches to balance performance and resource requirements for GenAI workloads (for example, by selecting appropriate models, by using smaller pre-trained models for specific tasks, by using API-based model cascading to perform routine queries).*

#### Right-size the model
- **Start with the smallest model that meets the quality bar.** Measure it with Bedrock **model evaluation** or your own eval set (Domain 5), and don't default to the flagship model.
- Rough size tiers and what they're good for:

  | Need | Typical choice |
  | --- | --- |
  | Classification, extraction, routing, short summaries | **Small/fast models**: Amazon Nova Micro/Lite, Claude Haiku-class, Llama 8B, Mistral Small |
  | General chat and RAG answers | Mid-tier models |
  | Complex reasoning, coding, multi-step agents | Frontier models (Claude Sonnet/Opus-class, Nova Pro/Premier) |
  | Embeddings | Dedicated **embedding** models (Titan Text Embeddings V2, Cohere Embed). Never use a generation model for this |

- **Task-specific small models** (e.g. a fine-tuned 8B classifier) are often more accurate *and* 10× cheaper than a general large model on narrow tasks.
- **Distillation** (**Bedrock Model Distillation**) trains a smaller "student" model from a large "teacher" model's responses, giving teacher-like quality at student cost and latency. See Task 1.2 for the customisation mechanics.
- If the model is right but too slow, Bedrock **latency-optimised inference** (Skill 2.2.1) can lower TTFT.

#### Model cascading (API-based)
Send every request to the **cheapest capable model first** and **escalate** only when needed.

```
request ──► Tier 1: small model (Nova Micro / Haiku)
              │  confidence OK / passes validation? ── yes ──► return
              ▼ no
            Tier 2: mid model ── passes? ── yes ──► return
              ▼ no
            Tier 3: frontier model ──► return (+ log for analysis)
```

- **Ways to decide when to escalate:**
  - the model self-reports confidence in a structured output field
  - validation fails (schema or regex mismatch, empty answer, "I don't know")
  - log-probability thresholds (only on self-hosted models)
  - a guardrail or grounding-check fail
  - a separate small judge model
- **Cascading vs routing:**
  - **Cascading** runs tiers *in sequence*. The worst case costs more than one call, but the average cost drops if most queries are routine.
  - **Routing** picks *one* model up front using classification or complexity scoring. Examples are **Amazon Bedrock Intelligent Prompt Routing** (routes between models in one family, based on predicted response quality) and Step Functions content-based routing. Routing is covered in [Task 2.4](04-task-2.4-fm-api-integrations.md).
- Implementation: a Lambda function or **Step Functions** Choice state, with model IDs and thresholds in **AppConfig** so you can tune them without redeploying.
- Measure **cost per resolved request** and **escalation rate**. If the escalation rate is high, cascading wastes money, so route directly or fine-tune the small model.

#### Cost and performance levers summary

| Lever | Saves | Trade-off |
| --- | --- | --- |
| Smaller or distilled model | Cost and latency | Possible quality loss; needs evaluation |
| Cascading or routing | Average cost | Complexity, extra latency on escalations |
| **Prompt caching** (Bedrock) | Cost and TTFT for repeated prefixes | Only helps stable prefixes; cache entries have a TTL |
| **Batch inference** or **Flex tier** | ~Discount on large or deferrable work | Not real time |
| Provisioned Throughput or Reserved | Predictable cost and capacity at steady load | Pay even when idle; commitment |
| On-demand + cross-Region | No idle cost; absorbs bursts | Quotas; data may be processed in other Regions in the geography |
| Quantisation (self-hosted) | Fewer or cheaper GPUs | Small accuracy drop |
| Inferentia2 | Cost per token | Neuron compilation; supported models only |
| Inference components / MME | GPU utilisation on shared fleets | Noisy neighbours, cold loads |
| Async endpoint with scale-to-zero | Idle GPU cost | Latency from queueing and cold start |
| Limit `maxTokens`, trim context, compress RAG chunks | Output and input tokens | Risk of truncated answers |
| Semantic cache (e.g. ElastiCache or MemoryDB vector search) | Repeat-question cost | Staleness; needs a similarity threshold |

#### Benchmark before you pick
- **SageMaker Inference Recommender** load-tests a model across instance types and reports latency, throughput and cost per instance, so you can right-size before production.
- Watch **TTFT**, **output tokens per second**, **p99 end-to-end latency** and **cost per 1K tokens** under *realistic* prompt and output lengths and concurrency.
- On Bedrock, use CloudWatch metrics `InvocationLatency`, `InputTokenCount`, `OutputTokenCount` and `InvocationThrottles`. Some models also report time to first token (verify `TimeToFirstToken` availability).

---

## Exam triggers

| Scenario cue | Answer |
| --- | --- |
| Fastest way to add an FM, pay per token, no infrastructure | **Bedrock on-demand** (Converse API) |
| Steady high volume; must avoid throttling; guaranteed throughput | **Bedrock Provisioned Throughput** (model units) or the **Reserved tier** |
| Mission-critical; reserved TPM for 1–3 months; overflow if exceeded | **Reserved tier** (overflows to Standard) |
| Customer-facing; needs fastest responses; no capacity reservation | **Priority tier** (`service_tier: "priority"`) |
| Check which tier actually served a request (Reserved overflow?) | CloudWatch **`ResolvedServiceTier`** dimension |
| Evaluations or summaries that can wait; cut cost | **Flex tier** or **batch inference** |
| Score millions of records overnight with an FM | **Bedrock batch inference** (JSONL in S3, `CreateModelInvocationJob`) |
| Batch job with Converse format and `toolConfig` fails with 400 | Tool calling isn't supported with Converse batch; use the **InvokeModel** invocation type |
| Fine-tuned Llama/Mistral/Qwen weights; want serverless Bedrock invocation | **Custom Model Import** (safetensors + config + tokenizer from S3) |
| Import a custom **embedding** model into Bedrock | ❌ CMI doesn't support embedding models → host on **SageMaker** |
| Need a custom inference engine, specific GPU, VPC isolation, data capture | **SageMaker real-time endpoint** |
| Small model, infrequent traffic, tolerates cold start, no GPU needed | **SageMaker Serverless Inference** |
| Serverless SageMaker for a 7B LLM? | ❌ No GPUs and only 6 GB memory → real-time or async endpoint, or Bedrock |
| Large documents (hundreds of MB) or generations taking ~30 min; scale to zero | **SageMaker Asynchronous Inference** (1 GB, 1 hr, S3 + SNS) |
| Several LLMs underusing GPUs on separate endpoints | **Inference components** on one shared endpoint |
| Thousands of small per-tenant models with sparse traffic | **Multi-model endpoint** |
| API Gateway times out on long LLM generations | Stream (WebSocket / Lambda URL), raise the integration timeout, or go **async** (SQS + callback) |
| Run a fine-tuned proprietary model + general FM behind one API | **Hybrid**: Bedrock for general + SageMaker endpoint, behind a Lambda/API Gateway façade |
| Guardrails for a model hosted on SageMaker | **`ApplyGuardrail` API** (model-independent) |
| 70B model doesn't fit on one GPU | **Tensor parallelism** (`OPTION_TENSOR_PARALLEL_DEGREE`) on a multi-GPU instance (p4d/p5), or **quantise** |
| Lowest cost per inference; supported architecture; can compile | **Inferentia2 (inf2)** + Neuron SDK |
| Lowest cost for training or fine-tuning on AWS silicon | **Trainium (trn1/trn2)** |
| Low GPU utilisation and throughput; requests wait for the batch | **Continuous (rolling) batching**: LMI with vLLM, or TGI |
| OOM errors only under high concurrency or long prompts | **KV cache** growth → lower max batch or model length, quantise, raise TP |
| Large model endpoint creation fails with a health-check timeout | Raise **container startup health-check** and **model download timeouts**; pre-shard; Fast Model Loader |
| Slow scale-out because weights take minutes to load | **SageMaker Fast Model Loader** (stream S3 → GPU), container caching, keep warm minimum |
| Most queries are routine; a few are complex; cut average cost | **Model cascading**: small model first, escalate when validation or confidence fails |
| Large-model quality at small-model cost for one narrow task | **Bedrock Model Distillation** or fine-tuning a small model |
| Same long system prompt or document in every request | **Prompt caching** |
| Which instance type should a model use? Benchmark cost vs latency | **SageMaker Inference Recommender** |

## Flashcards

<details>
<summary><b>Q:</b> What are the four Amazon Bedrock service tiers, and how do you select one?</summary>

**Reserved**, **Priority**, **Standard** (`default`) and **Flex**. Set the optional `service_tier` parameter on the runtime request. If you leave it out, the request goes to Standard.

*D2 · Task 2.2 · Service tiers*
</details>

<details>
<summary><b>Q:</b> What happens when traffic exceeds a Reserved tier reservation?</summary>

It automatically overflows to the Standard tier, so operations continue. The CloudWatch `ResolvedServiceTier` dimension shows which tier actually served each request.

*D2 · Task 2.2 · Reserved tier*
</details>

<details>
<summary><b>Q:</b> Which Bedrock tiers share the on-demand quota?</summary>

Priority, Standard and Flex share it. Reserved capacity is separate from the on-demand quota.

*D2 · Task 2.2 · Service tiers*
</details>

<details>
<summary><b>Q:</b> Bedrock batch inference: input format and key limitation with the Converse invocation type?</summary>

JSONL in S3 with one `{"recordId", "modelInput"}` per line, started with `CreateModelInvocationJob`. With the Converse type, tool calling (`toolConfig`) isn't supported, so use the InvokeModel type. Batch also isn't supported for provisioned models.

*D2 · Task 2.2 · Batch inference*
</details>

<details>
<summary><b>Q:</b> What files and format does Bedrock Custom Model Import need?</summary>

Hugging Face format in S3: `.safetensors` weights, `config.json`, `tokenizer_config.json`, `tokenizer.json` and `tokenizer.model`. Supported architectures include Llama, Mistral/Mixtral, Qwen, Flan-T5, GPTBigCode and GPT-OSS.

*D2 · Task 2.2 · Custom Model Import*
</details>

<details>
<summary><b>Q:</b> Name three things Custom Model Import does NOT support.</summary>

Embedding models, batch inference and CloudFormation. Also, Qwen3 models can't use the Converse API.

*D2 · Task 2.2 · Custom Model Import*
</details>

<details>
<summary><b>Q:</b> Why is SageMaker Serverless Inference a bad fit for hosting a 7B LLM?</summary>

It has no GPUs and at most 6 GB of memory. A 7B model needs about 14 GB in FP16 plus a KV cache.

*D2 · Task 2.2 · Serverless Inference*
</details>

<details>
<summary><b>Q:</b> Which CloudWatch metric shows cold-start time for a SageMaker serverless endpoint, and how do you reduce it?</summary>

`OverheadLatency`. Reduce it with **Provisioned Concurrency**, which can be autoscaled on a target metric or a schedule.

*D2 · Task 2.2 · Cold starts*
</details>

<details>
<summary><b>Q:</b> What are the payload and time limits for SageMaker Asynchronous Inference?</summary>

Payloads up to 1 GB and processing time up to 1 hour. Input and output go through S3, notifications can go to SNS, and the endpoint can autoscale to zero instances.

*D2 · Task 2.2 · Async Inference*
</details>

<details>
<summary><b>Q:</b> What problem do SageMaker inference components solve?</summary>

They pack several models onto one endpoint's instances. Each model gets its own accelerator, CPU and memory allocation, copy count and autoscaling, which improves GPU utilisation and cuts cost.

*D2 · Task 2.2 · Inference components*
</details>

<details>
<summary><b>Q:</b> What is Lambda's role in "on-demand invocation" of FMs, and what limits it?</summary>

It's stateless glue code that calls Bedrock or a SageMaker endpoint (templates, validation, auth). It can't host LLMs: no GPU, 10 GB memory and a 15-minute timeout. Use streaming or async for long generations.

*D2 · Task 2.2 · Lambda*
</details>

<details>
<summary><b>Q:</b> How can a model hosted on SageMaker still use Bedrock Guardrails?</summary>

Call the `ApplyGuardrail` API on the input and output text. It works without Bedrock model invocation.

*D2 · Task 2.2 · Hybrid*
</details>

<details>
<summary><b>Q:</b> Roughly how much GPU memory do the weights of a 70B model need in FP16, and what does that imply?</summary>

About 140 GB (70B × 2 bytes), plus KV cache and overhead. It needs several GPUs with tensor parallelism (e.g. p4de or p5 with 8 × 80 GB), or quantisation.

*D2 · Task 2.2 · Memory sizing*
</details>

<details>
<summary><b>Q:</b> What is the KV cache, and why does it cause out-of-memory errors under load?</summary>

It stores the attention keys and values for every token in every active sequence. It grows with context length × concurrent requests, so high concurrency or long prompts can exhaust GPU memory even when the weights fit.

*D2 · Task 2.2 · KV cache*
</details>

<details>
<summary><b>Q:</b> When do you choose Inferentia2 over NVIDIA GPUs?</summary>

When you want the lowest cost per inference, the model architecture is supported by the Neuron SDK, and you can accept the compilation step.

*D2 · Task 2.2 · Accelerators*
</details>

<details>
<summary><b>Q:</b> Tensor parallelism vs pipeline parallelism?</summary>

Tensor parallelism splits each layer's matrices across GPUs within a node (the usual choice for inference). Pipeline parallelism places groups of layers on different devices or nodes, which adds pipeline bubbles.

*D2 · Task 2.2 · Parallelism*
</details>

<details>
<summary><b>Q:</b> What is continuous (rolling) batching?</summary>

New requests join the running batch at each decode step instead of waiting for the whole batch to finish. This greatly raises GPU utilisation and tokens per second. In LMI it's `OPTION_ROLLING_BATCH` (vLLM, LMI-Dist, TensorRT-LLM); TGI also has it.

*D2 · Task 2.2 · Continuous batching*
</details>

<details>
<summary><b>Q:</b> Name four ways to cut LLM endpoint cold-start or scale-out time.</summary>

1. Pre-sharded or pre-compiled weights in same-Region S3 (uncompressed, safetensors).
2. SageMaker Fast Model Loader, which streams from S3 to the GPU.
3. Container caching.
4. A warm minimum instance count or scheduled scaling.

*D2 · Task 2.2 · Model loading*
</details>

<details>
<summary><b>Q:</b> Endpoint creation for a large LLM fails with a health-check failure while the weights are still loading. Fix?</summary>

Raise `ContainerStartupHealthCheckTimeoutInSeconds` and `ModelDataDownloadTimeoutInSeconds`, and speed up loading with pre-sharded weights or Fast Model Loader.

*D2 · Task 2.2 · Deployment gotcha*
</details>

<details>
<summary><b>Q:</b> Model cascading vs model routing?</summary>

Cascading tries the cheapest model first and escalates in sequence when validation or confidence fails. Routing picks one model up front from a classification, as Bedrock Intelligent Prompt Routing or Step Functions content-based routing does.

*D2 · Task 2.2 · Cascading*
</details>

<details>
<summary><b>Q:</b> When does cascading increase cost instead of reducing it?</summary>

When the escalation rate is high. Many requests then pay for multiple model calls. Route directly or improve the small model (fine-tune or distil) instead.

*D2 · Task 2.2 · Cascading*
</details>

<details>
<summary><b>Q:</b> Which tool load-tests a model across instance types to recommend the best cost/latency option?</summary>

SageMaker Inference Recommender.

*D2 · Task 2.2 · Benchmarking*
</details>

<details>
<summary><b>Q:</b> Which LLM serving metrics matter more than requests per second?</summary>

Time to first token (TTFT), inter-token latency or output tokens per second, end-to-end p99 latency, and cost per 1K tokens, measured at realistic prompt lengths and concurrency.

*D2 · Task 2.2 · Metrics*
</details>

---
[← Back to Domain 2](README.md)
