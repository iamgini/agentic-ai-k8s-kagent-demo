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
# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# kagent CLI
curl https://raw.githubusercontent.com/kagent-dev/kagent/refs/heads/main/scripts/get-kagent | bash
```


## Setup Ollama 

(on another machine for demo)

```shell
# 1. Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# 2. Start Ollama listening on all interfaces (not just localhost)
OLLAMA_HOST=0.0.0.0:11434 ollama serve &

# or edit the systemd
sudo systemctl edit ollama

# Add override 
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"

sudo systemctl daemon-reload
sudo systemctl restart ollama

# verify it's now listening on all interfaces
ss -tlnp | grep 11434

```

```shell
# 3. Pull the model (do this before talk day — takes a few minutes)
ollama pull qwen2.5:32b

# 4. Quick test
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:32b",
  "prompt": "Why would a Kubernetes pod be in ImagePullBackOff?",
  "stream": false
}'
```

## Setup

```shell
kubectl create secret generic kagent-openai \
  --from-literal=OPENAI_API_KEY=<your-key> \
  -n kagent
```  

### 1. Create the kind cluster
```bash
kind create cluster --name kagent-demo --config kind-cluster.yaml
kubectl cluster-info --context kind-kagent-demo
```

```shell
export OPENAI_API_KEY=<your-key>
```

### 2. Install kagent
```bash
kagent install --profile demo
kubectl get pods -n kagent   # wait until all pods are Running
```


### 3. Use OpenAI (faster, fallback)
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
