# AKS GPU Lab 

## What you should know before starting with this project?

1. Kubernetes
    * 1.1 Should know k8s/AKS
    * 1.2 How taint and tolerations work
    * 1.3 CRDs
    * 1.4 Deployments, Services
2. GPUs
    * 2.1 How GPUs work
    * 2.2 Internal components of GPU such as SM
    * 2.3 Memory hierarchy of GPU and internal working
    * 2.4 CUDA Cores
    * 2.5 Tensor cores
    * 2.6 GPU time slicing
3. LLM and vLLM
4. Pytorch Basics


### We are deploying a Standard_NC4as_T4_v3 VM having 1x NVIDIA T4 GPU.
#### The GPU will serve the LLM model TinyLlama 1.1B (FP16) which has following specs - 
- Model: TinyLlama 1.1B (FP16)
- VRAM requirement: ~5.5 GB
- Fits in T4 (15GB)?: Fits comfortably
- Tokens/sec on T4: ~20 tok/s
- **Cost Running the T4 GPU:** A `Standard_NC4as_T4_v3` GPU node costs ~$0.50/hr.

#### About the model TinyLlama:

- TinyLlama is a 1.1 billion parameter language model released by researchers at Singapore University. 
- It was trained on 3 trillion tokens using the same architecture as Meta's Llama 2 — but compressed down to a tiny size.

#### The objective is to deploy the LLM model to GPU over AKS and test the results. End-to-end GPU infrastructure on Azure Kubernetes Service — covering time-slicing, inference serving, benchmarking, LoRA fine-tuning, and production observability.

**Stack:** AKS · Ubuntu 22.04 · NVIDIA T4 · vLLM · PyTorch · DCGM · Prometheus · Grafana

---

## Phase 1 — Infrastructure

### VM Size Selection

| VM Size | GPU | VRAM | Cost (approx) |
|---|---|---|---|
| Standard_NC4as_T4_v3 | 1x NVIDIA T4 | 16 GB | ~$0.50/hr |
| Standard_NC6s_v3 | 1x V100 | 16 GB | ~$3.00/hr |

### Create the AKS Cluster

```bash
az login

az group create --name gpu-aks-rg --location centralindia

az aks create \
  --resource-group gpu-aks-rg \
  --name gpu-aks \
  --node-count 1 \
  --node-vm-size Standard_D2s_v3 \
  --nodepool-name systempool \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --network-plugin azure
```

### Add the GPU Node Pool

```bash
az aks nodepool add \
  --resource-group gpu-aks-rg \
  --cluster-name gpu-aks \
  --name gpupool \
  --node-count 1 \
  --node-vm-size Standard_NC4as_T4_v3 \
  --node-taints sku=gpu:NoSchedule \
  --aks-custom-headers UseGPUDedicatedVHD=true
```

`--node-taints sku=gpu:NoSchedule` prevents regular pods from landing on the expensive GPU node.

`UseGPUDedicatedVHD=true` tells AKS to use a GPU-optimised node image with NVIDIA drivers pre-installed — no manual driver installation needed.

> **Do NOT install NVIDIA drivers manually.** Azure pre-installs them via the GPU-dedicated VHD.

**When do you need the GPU Operator on AKS?**

Use AKS native (this lab) when:
- Single workload per GPU
- You're happy with whatever driver version Azure ships
- No MIG partitioning needed
- Simple dev/test setup

Add the GPU Operator when:
- Multi-tenant inference (multiple teams sharing GPUs)
- MIG partitioning on A100/H100 nodes
- Controlled driver versions (specific CUDA dependency)
- Heterogeneous cluster with different GPU types
- Production SLA requirements

### Connect kubectl

```bash
az aks get-credentials --resource-group gpu-aks-rg --name gpu-aks
kubectl get nodes -o wide
```

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Phase 2 — GPU Time-Slicing

Time-slicing is a node-level configuration that must be set up before any workload pods are scheduled. It changes what the GPU node advertises to Kubernetes (`nvidia.com/gpu: 4` instead of `1`).

| | Time-Slicing | MIG |
|---|---|---|
| Isolation | Soft — time-shared, no memory isolation | Hard — dedicated HBM slice per instance |
| Supported GPUs | All NVIDIA GPUs including T4 | A100, H100 only |
| Memory | All pods share full GPU memory pool | Each instance has fixed private memory |

### Create Time-Slicing ConfigMap

Apply `time-slicing-configmap.yaml` (see `/manifests` folder).

### Deploy Device Plugin with Time-Slicing

> **Critical:** Always use `https://nvidia.github.io/k8s-device-plugin` — NOT the NGC repo (`helm.ngc.nvidia.com/nvidia`). 
- The NGC repo hosts an abandoned v0.9.0 chart with no config file support.
- I HAVE SPENT ALMOST 3 HOURS DEBUGGING THIS SMALL ISSUE.

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update

helm install nvidia-device-plugin nvdp/nvidia-device-plugin \
  --namespace kube-system \
  --set 'tolerations[0].key=sku' \
  --set 'tolerations[0].operator=Equal' \
  --set 'tolerations[0].value=gpu' \
  --set 'tolerations[0].effect=NoSchedule' \
  --set 'nodeSelector.accelerator=nvidia' \
  --set config.name=time-slicing-config \
  --set nfd.enabled=false \
  --set gfd.enabled=false
```

`nfd.enabled=false` and `gfd.enabled=false` disable NFD-based node affinity — required on AKS to avoid needing manual node labels after every scale-up.

### Verify Time-Slicing is Active

```bash
kubectl describe node <gpu-node-name> | grep "nvidia.com/gpu"
```

Expected output:
```
nvidia.com/gpu:     4   # Capacity
nvidia.com/gpu:     4   # Allocatable
```

> **After node pool scale-up:** If the device plugin pod shows `DESIRED: 0`, the new node may be missing NFD labels. Add them manually:
> ```bash
> kubectl label node <new-node-name> \
>   "feature.node.kubernetes.io/pci-10de.present=true" \
>   "nvidia.com/gpu.present=true"
> ```
> This is avoided permanently by using `nfd.enabled=false` in the Helm install above.

---

## Phase 3 — Verification

### nvidia-smi Inside a Pod

Apply `nvidia-smi-pod.yaml` (see `/manifests` folder).

```bash
kubectl logs nvidia-smi-pod
```

Expected: Full nvidia-smi table showing Tesla T4, Driver Version, 16384 MiB VRAM.

![smi](screenshots/smi.png)

### Deploy Persistent dcgmi Observer

Apply `dcgmi-observer.yaml` (see `/manifests` folder).

```bash
kubectl logs dcgmi-observer -f
```

Expected: Streaming table of GPU metrics — SM clock, memory clock, temperature — every 500ms.

![dcgmi-observer](screenshots/dcgmi-observer.png)

---

## Phase 4 — vLLM Inference

### Why TinyLlama

TinyLlama 1.1B in FP16 = ~2.2 GB VRAM. Fits comfortably in the T4's 15.6 GB, leaving room for KV cache and other workloads.

### Deploy vLLM TinyLlama

Apply `vllm-tinyllama.yaml` (see `/manifests` folder).

Wait for the model to load — watch for `Uvicorn running on http://0.0.0.0:8000` in the logs:

```bash
kubectl logs -l app=vllm-tinyllama -f
```

![vllama](screenshots/vllama.png)


### Send Inference Request

```bash
kubectl port-forward svc/vllm-tinyllama 8000:8000 &

curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "TinyLlama/TinyLlama-1.1B-Chat-v1.0",
    "prompt": "Explain what a GPU is:",
    "max_tokens": 150
  }'
```

Inference screenshot:

![llm-inference](screenshots/llm-inference.png)


### What to Watch in dcgmi

| Metric | Idle | During Inference |
|---|---|---|
| SM Clock | 300 MHz | ~1590 MHz |
| Memory Clock | 405 MHz | ~5000 MHz |
| GPU Util | 0% | 60–100% |
| Tensor Core Util | 0% | 38–41% |

---

## Phase 5 — Observability Pipeline

### Data Flow

```
T4 GPU hardware
      ↓  (DCGM reads hardware counters)
dcgm-exporter pod
      ↓  (exposes metrics at :9400/metrics)
Prometheus
      ↓  (scrapes every 30s, stores time-series)
Grafana
      ↓  (queries Prometheus, renders dashboards)
```

### Step 1: Install Prometheus Stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.scrapeInterval="30s" \
  --set grafana.adminPassword="admin123"
```

Wait for all pods to be Running:

```bash
kubectl get pods -n monitoring --watch
```

### Step 2: Install dcgm-exporter

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update

helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace monitoring \
  --set "tolerations[0].key=sku" \
  --set "tolerations[0].operator=Equal" \
  --set "tolerations[0].value=gpu" \
  --set "tolerations[0].effect=NoSchedule" \
  --set "nodeSelector.accelerator=nvidia" \
  --set "serviceMonitor.enabled=true"
```

### Step 3: Fix ServiceMonitor Label

The `kube-prometheus-stack` Prometheus operator only scrapes ServiceMonitors with the `release=prometheus` label. Apply it:

```bash
kubectl label servicemonitor dcgm-exporter -n monitoring release=prometheus
```

### Step 4: Verify Prometheus is Scraping

```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
```

Open `http://localhost:9090` → Status → Targets → confirm `dcgm-exporter` shows state `UP`.

Also verify raw metrics:

```bash
curl -s "http://localhost:9090/api/v1/query?query=DCGM_FI_DEV_GPU_UTIL" | python3 -m json.tool
```

### Step 5: Import Grafana Dashboard

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &
```

Open `http://localhost:3000` — login: `admin` / `admin123`

Import dashboard via API (avoids datasource variable issues):

```bash
curl -s https://grafana.com/api/dashboards/12239/revisions/latest/download -o nvidia-dashboard.json

python3 -c "
import json
with open('nvidia-dashboard.json') as f:
    d = json.load(f)
content = json.dumps(d).replace('\${DS_PROMETHEUS}', 'prometheus')
with open('nvidia-dashboard-fixed.json', 'w') as f:
    f.write(content)
"

curl -s -X POST \
  http://admin:admin123@localhost:3000/api/dashboards/import \
  -H 'Content-Type: application/json' \
  -d "{\"dashboard\": $(cat nvidia-dashboard-fixed.json), \"overwrite\": true, \"folderId\": 0, \"inputs\": [{\"name\": \"DS_PROMETHEUS\", \"type\": \"datasource\", \"pluginId\": \"prometheus\", \"value\": \"prometheus\"}]}"
```

### Key Dashboard Panels

| Panel | Metric | What it shows |
|---|---|---|
| GPU Utilization | `DCGM_FI_DEV_GPU_UTIL` | % of time GPU was computing |
| Tensor Core Utilization | `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | Tensor Core pipeline activity |
| SM Clock | `DCGM_FI_DEV_SM_CLOCK` | Compute clock — jumps during workload |
| VRAM Used | `DCGM_FI_DEV_FB_USED` | Memory consumed by workloads |
| Power Usage | `DCGM_FI_DEV_POWER_USAGE` | Watts drawn — correlates with compute |
| Temperature | `DCGM_FI_DEV_GPU_TEMP` | GPU core temperature |

> **Stale series fix:** If power or utilization panels show inflated values, edit the panel query to use `last_over_time(DCGM_FI_DEV_POWER_USAGE{gpu="0"}[2m])` to filter out stale label sets from completed pods.

**Initial Grafana Dashboard with few inference requests:**

![grafana-initial01](screenshots/grafana-initial01.png)

---

## Phase 6 — Batched Inference

### Send 8 Concurrent Requests

```bash
for i in {1..8}; do
  curl -s http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "TinyLlama/TinyLlama-1.1B-Chat-v1.0",
      "prompt": "What is machine learning?",
      "max_tokens": 100
    }' &
done
wait
```

### What to Watch in dcgmi

Tensor Core utilization should jump from ~38–41% (single inference) to ~72–78% with 8 concurrent requests. This validates the theory that Tensor Cores become more efficient with larger batch sizes — the matrix multiplications in the attention layers grow large enough to fully saturate the Tensor Core pipelines.

---

## Phase 7 — PyTorch Benchmark

- This code in the `args` in `pytorch-benchmark.yaml` does the GPU stress test. It tests the physical hardware (the NVIDIA GPU).
- The args block runs a raw Python script that forces the GPU to work at 100% capacity.
- **The Math:** It creates two massive matrices (4096 x 4096) filled with random numbers.
- **The Precision:** It uses float16, which matches the data type you set for vLLM earlier.
- **The Loop:** It runs a while loop for exactly 180 seconds (3 minutes).Inside, it performs torch.matmul(a, b) (Matrix Multiplication) over and over.
- **torch.cuda.synchronize():** GPU tasks are "lazy" (asynchronous). This command forces the script to wait until the GPU actually finishes the last calculation before printing "Done."

Apply `pytorch-benchmark.yaml` (see `/manifests` folder).

```bash
kubectl logs pytorch-benchmark -f
```

### Expected Results

| Test | Time | Speedup |
|---|---|---|
| CPU (FP32) | ~300s | baseline |
| GPU (FP32) | ~0.56s | ~537x vs CPU |
| GPU (FP16) | ~0.16s | ~3.5x vs FP32 |

The FP32 → FP16 speedup confirms Tensor Cores are active. FP16 matmuls are routed to Tensor Cores; FP32 uses CUDA cores.

### What to Watch in dcgmi During Benchmark

- SM Clock spikes to ~1590 MHz
- Tensor Core Utilization reaches 85–90%
- Power draw rises to 60–70W
- GPU Utilization hits 100%

I have covered all the benchmark reports in the Grafana itself.

**Grafana Dashboard with pytorch bennchmark and batched inference requests:**

![grafana-load-test](screenshots/grafana-load-test.png)

---

## Phase 8 — OOM Stress Test

Apply `oom-test.yaml` (see `/manifests` folder).

```bash
kubectl logs oom-test -f
```

### What Happens

The test allocates progressively larger tensors until VRAM is exhausted. You will see `torch.OutOfMemoryError: CUDA out of memory` in the logs.

### Recovery

```bash
kubectl delete pod oom-test
```

The GPU resets automatically when the pod exits. Verify:

```bash
kubectl logs dcgmi-observer | tail -5
# VRAM should return to baseline
```

---

## Phase 9 — How a Pod Gets to the GPU

The complete 5-stage journey from YAML to Tensor Core:

**Stage 1 — The Pod YAML Is the Contract**

The `toleration` says "I am permitted on the GPU node." The `nvidia.com/gpu: 1` resource limit says "I need one GPU time-slice." Without both, the pod either gets rejected at scheduling or lands on the wrong node.

**Stage 2 — The Kubernetes Scheduler**

Two independent checks must pass: the taint/toleration check (does this pod have the key to the GPU node's lock?), and the resource availability check (are any of the 4 time-slice slots free?).

**Stage 3 — kubelet Calls the Device Plugin**

kubelet sees the `nvidia.com/gpu: 1` request and calls the NVIDIA device plugin via gRPC. The device plugin responds: inject `NVIDIA_VISIBLE_DEVICES=0` into the container environment. That index is how the container knows which physical GPU it's allowed to talk to.

**Stage 4 — Container Toolkit Mounts the GPU**

containerd starts the container and triggers the NVIDIA container toolkit hook. The toolkit reads `NVIDIA_VISIBLE_DEVICES=0` and mounts `/dev/nvidia0` into the container filesystem plus NVIDIA driver libraries. This is why `nvidia-smi` inside the pod works.

**Stage 5 — vLLM Uses the GPU**

- Load model weights into HBM — TinyLlama 1.1B in FP16 = ~2.2 GB via PCIe
- Allocate KV cache — vLLM pre-allocates remaining HBM for past token attention
- Token generation loop — attention matmul hits Tensor Cores; softmax, GELU, FFN hit CUDA cores

---

## Phase 10 — LoRA Fine-Tuning
***Note to self:  This is not done! Not planning to do until I fully understand it.***

### Why Training Uses More Memory Than Inference

| Stage | Memory Used |
|---|---|
| Inference (model weights only) | ~2.2 GB |
| Training (weights + gradients + optimizer states + activations) | ~5+ GB |

LoRA reduces this significantly by freezing the base model and only training small adapter matrices (~1M params vs 1.1B total).

---

## Cleanup

> GPU node = ~$0.50/hr. Overnight = ~$12. One week = ~$84.

```bash
# Delete workload pods
kubectl delete pod nvidia-smi-pod dcgmi-observer pytorch-benchmark oom-test lora-finetune

# Uninstall Helm releases
helm uninstall nvidia-device-plugin -n kube-system
helm uninstall dcgm-exporter -n monitoring
helm uninstall prometheus -n monitoring

# Option A: Scale GPU pool to 0 (keep cluster, stop GPU billing)
az aks nodepool scale \
  --resource-group gpu-aks-rg \
  --cluster-name gpu-aks \
  --name gpupool \
  --node-count 0

# Option B: Delete everything
az group delete --name gpu-aks-rg --yes --no-wait
```

---