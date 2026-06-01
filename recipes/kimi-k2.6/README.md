# Kimi-K2.6 Turbo Recipes

Turbo recipes for **moonshotai/Kimi-K2.6**.

## Configurations

Dynamo + vLLM deployment profiles across two GPU SKUs and two target workloads:

|                          | B200 chat                             | H200 chat                             | B200 agentic                         | H200 agentic                         |
|--------------------------|---------------------------------------|---------------------------------------|--------------------------------------|--------------------------------------|
| **GPU** (per worker)                  | 4x B200                               | 8x H200                               | 4x B200                              | 8x H200                              |
| **Mode**                 | aggregated                            | aggregated                            | aggregated                           | aggregated                           |
| **Framework**            | vLLM 0.21.0                           | vLLM 0.21.0                           | vLLM 0.21.0                          | vLLM 0.21.0                          |
| **Precision**            | NVFP4 + FP8 KV                        | INT4                                  | NVFP4 + FP8 KV                       | INT4                                 |
| **Parallelism**          | TP4                                   | TP8                                   | TP4                                  | TP8                                  |
| **MoE backend**          | FLASHINFER_TRTLLM                     | MARLIN                                | FLASHINFER_TRTLLM                    | MARLIN                               |
| **Attention backend**    | TOKENSPEED_MLA                        | FLASH_ATTN_MLA                        | TOKENSPEED_MLA                       | FLASH_ATTN_MLA                       |
| **AllReduce backend**    | NCCL symmetric memory                 | NCCL                                  | NCCL symmetric memory                | NCCL                                 |
| **All2All backend**      | N/A                                   | N/A                                   | N/A                                  | N/A                                  |
| **Routing**              | KV-aware                              | KV-aware                              | KV-aware                             | KV-aware                             |
| **Speculative decoding** | EAGLE3 MLA (DL=3, SpeedBench AL=2.49) | EAGLE3 MLA (DL=3, SpeedBench AL=2.49) | EAGLE3 MLA (DL=3, SpeedBench AL=2.49) | EAGLE3 MLA (DL=3, SpeedBench AL=2.49) |
| **KV cache offloading**  | LMCache CPU                           | LMCache CPU                           | LMCache CPU                          | LMCache CPU                          |


## Supported features

- Modalities: Text + Image
- Reasoning
- Tool calling


## Prerequisites

1. **Dynamo Platform installed** — see [Kubernetes Deployment Guide](../../docs/kubernetes/README.md).
2. **HuggingFace token** with access to `nvidia/Kimi-K2.6-NVFP4`, `moonshotai/Kimi-K2.6` and `lightseekorg/kimi-k2.6-eagle3-mla`:
   ```bash
   export NAMESPACE=your-namespace
   kubectl create secret generic hf-token-secret \
     --from-literal=HF_TOKEN="your-token" \
     -n ${NAMESPACE}
   ```


## Quick Start

### 1. Create Storage

> **Note:** Edit `model-cache/model-cache.yaml` first and update `storageClassName` to match your cluster (`kubectl get storageclass`).

```bash
kubectl apply -f model-cache/model-cache.yaml -n ${NAMESPACE}
```

### 2. Download model + EAGLE3 head

> **Note:** Edit `model-cache/model-download.yaml` first and remove the `hf download` lines that do not apply to your deployment.

```bash
kubectl apply -f model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s
```


### 3. Deploy the DGD

Deploy the target DGD:

```bash
SKU=b200 # or h200
USECASE=chat # or agentic

kubectl apply -f vllm/turbo_kimi_k26_agg_${SKU}_${USECASE}.yaml -n ${NAMESPACE}
```


### 4. Benchmark

See [`perf/README.md`](perf/README.md) for the full benchmark workflow — trace staging on the PVC, running the AIPerf trace-replay Job ([`perf/perf.yaml`](perf/perf.yaml)), running a concurrency sweep, and fetching artifacts.


## Optimization targets

Recipes are optimized for the following configurations, at the target user interactivity:

| Workload                 | Median ISL | Median OSL | KV cache hit rate | User output tok/s |
|------------------------|------------|------------|----------------|------------|
| Chat                   |      1k      | 1k        |  70%       | 50        |
| Agentic                  |      64k      | 400        | 90%        | 50        |


Modified Mooncake traces are provided to showcase the value of KV-aware routing and CPU offloading, see [perf/README.md](./perf/README.md) for details.


## Performance results

| Recipe                 | SKU | # workers | Concurrency | User output tok/s | System output tok/s/gpu | TTFT |
|------------------------|------------|------------|----------------|------------|------------|------------|
| Chat                   |      B200      | 4        |         |         |  |  |
| Agentic                  |      B200      | 4        |         |         | | |


## Known issues

1. Guided decoding and tool calling with `tool_choice` = `required` will raise 500 errors. Add `--no-async-scheduling` to the worker command to enable these features at the cost of low-latency performance.
2. Dynamo's KV cache router does not support all LMCache KV events, so routing can be sub-optimal
3. Some 400 HTTP errors from the workers on invalid inputs can be raised as 500 errors through the frontend
4. Disabling reasoning with the current container requires both flags in the request's `chat_template_kwargs`:
   ```json
   "chat_template_kwargs": {"thinking": false, "enable_thinking": false}
   ```
