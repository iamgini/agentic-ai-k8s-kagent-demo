# kagent Demo — Agentic AI on Kubernetes: The Cloud Native Way

Demo repo for the CNCF Cloud Native Singapore Meetup talk.

**Speaker:** Gineesh Madapparambath | Architect, Red Hat  
**Talk:** Agentic AI on Kubernetes: The Cloud Native Way

---

## What this demo shows

A Kubernetes-native AI agent (powered by [kagent](https://kagent.dev)) autonomously
diagnosing a broken Kubernetes deployment using natural language — no scripts, no
hardcoded logic, just an agent with tools.

## Demo scenario

1. A `broken-app` deployment is applied with a bad container image → `ImagePullBackOff`
2. You ask the agent in plain English: *"Why is the broken-app pod failing and how do I fix it?"*
3. The agent calls `k8s_get_resources` → `k8s_get_events` → `k8s_describe_resource` in sequence
4. It returns a plain-language diagnosis — you apply the fix

## Repo structure

```
kagent-demo/
├── README.md                   # this file
├── kind-cluster.yaml           # kind cluster config
├── modelconfig-openai.yaml     # kagent ModelConfig for OpenAI
├── modelconfig-ollama.yaml     # kagent ModelConfig for Ollama (alternative)
├── agent.yaml                  # the k8s-troubleshooter Agent CRD
├── broken-app.yaml             # intentionally broken deployment (demo target)
└── demo-script.md              # exact on-stage commands
```

---

## LLM Provider

kagent is model-agnostic. You can use any LLM provider by changing one field in the `ModelConfig` YAML:

| Provider | File | Notes |
|---|---|---|
| OpenAI | `modelconfig-openai.yaml` | Recommended for demos — fast, reliable |
| Ollama (local) | `modelconfig-ollama.yaml` | 100% open source, requires GPU for best performance |
| Anthropic, Gemini, Bedrock | — | Supported natively, see [kagent docs](https://kagent.dev/docs/kagent/supported-providers) |

This demo uses **OpenAI** for reliability on stage.

---

## Prerequisites

```bash
# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl

# kagent CLI
curl https://raw.githubusercontent.com/kagent-dev/kagent/refs/heads/main/scripts/get-kagent | bash
```

---

## Setup

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

### 3. Create the OpenAI secret
```bash
kubectl create secret generic kagent-openai \
  --from-literal=OPENAI_API_KEY=<your-key> \
  -n kagent
```

### 4. Apply the ModelConfig and Agent
```bash
kubectl apply -f modelconfig-openai.yaml
kubectl apply -f agent.yaml
kubectl get agent -n kagent   # should show k8s-troubleshooter READY=True
```

### 5. Deploy the broken app
```bash
kubectl apply -f broken-app.yaml
kubectl get pods -n default   # should show ImagePullBackOff after ~30s
```

### 6. Open the dashboard
```bash
kubectl port-forward -n kagent service/kagent-ui 8082:8080
# then open http://localhost:8082
```

> **Note:** On GitHub Codespaces, the port is auto-forwarded. Click the port 8082 URL in the Ports panel.

---

## On-stage demo flow

See [`demo-script.md`](./demo-script.md) for the exact commands and talking points.

---

## References

- kagent docs: https://kagent.dev/docs/kagent
- kagent GitHub: https://github.com/kagent-dev/kagent
- CNCF project page: https://www.cncf.io/projects/kagent/
- Supported LLM providers: https://kagent.dev/docs/kagent/supported-providers