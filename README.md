# kagent Demo — Agentic AI on Kubernetes: The Cloud Native Way

Demo repo for the CNCF Cloud Native Singapore Meetup talk.

**Speaker:** Gineesh Madapparambath | Architect, Red Hat  
**Talk:** Agentic AI on Kubernetes: The Cloud Native Way

## What this demo shows

A Kubernetes-native AI agent (powered by [kagent](https://kagent.dev)) autonomously
diagnosing a broken Kubernetes deployment using natural language — no scripts, no
hardcoded logic, just an agent with tools.

## Demo scenario

1. A `broken-app` deployment is applied with a bad container image → `ImagePullBackOff`
2. You ask the agent in plain English: *"Why is the broken-app pod failing and how do I fix it?"*
3. The agent calls `k8s_get_resources` → `k8s_get_events` → `k8s_describe_resource` in sequence
4. It returns a plain-language diagnosis and fix — all inside Kubernetes, all open source

## Repo structure

```
kagent-demo/
├── README.md                   # this file
├── kind-cluster.yaml           # kind cluster config
├── ollama-in-cluster.yaml      # Ollama deployment inside kind (Option A)
├── modelconfig-ollama.yaml     # kagent ModelConfig for Ollama
├── modelconfig-openai.yaml     # kagent ModelConfig for OpenAI (Option B fallback)
├── agent.yaml                  # the k8s-troubleshooter Agent CRD
├── broken-app.yaml             # intentionally broken deployment (demo target)
└── demo-script.md              # exact on-stage commands
```

## Prerequisites

```bash
brew install kind helm kubectl
brew install kagent
```

For **Option A (Ollama — 100% open source):**
```bash
brew install ollama
```

For **Option B (OpenAI — faster, fallback):**
```bash
export OPENAI_API_KEY="your-key-here"
```

## Setup (run once before the talk)

### 1. Create the kind cluster
```bash
kind create cluster --name kagent-demo --config kind-cluster.yaml
kubectl cluster-info --context kind-kagent-demo
```

### 2. Install kagent
```bash
kagent install --profile demo
kubectl get pods -n kagent   # wait until all pods are Running
```

### 3a. Option A — Deploy Ollama inside the cluster (recommended)
```bash
kubectl apply -f ollama-in-cluster.yaml
kubectl rollout status deployment/ollama -n ollama

# Pull the model into the running pod
kubectl exec -n ollama deploy/ollama -- ollama pull llama3.1

# Apply the Ollama ModelConfig
kubectl apply -f modelconfig-ollama.yaml
```

### 3b. Option B — Use OpenAI (faster, fallback)
```bash
kubectl apply -f modelconfig-openai.yaml
```

### 4. Deploy the agent
```bash
kubectl apply -f agent.yaml
kubectl get agent -n kagent   # should show k8s-troubleshooter
```

### 5. Deploy the broken app
```bash
kubectl apply -f broken-app.yaml
kubectl get pods -n default   # should show ImagePullBackOff after ~30s
```

### 6. Open the dashboard
```bash
kagent dashboard
# Opens http://localhost:8082
```

## On-stage demo flow

See [`demo-script.md`](./demo-script.md) for the exact commands and talking points.

## References

- kagent docs: https://kagent.dev/docs/kagent
- kagent GitHub: https://github.com/kagent-dev/kagent
- CNCF project page: https://www.cncf.io/projects/kagent/
