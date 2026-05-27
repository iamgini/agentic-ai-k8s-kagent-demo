# Knowledge Base — Agentic AI on Kubernetes
# Speaker reference for Q&A, deep questions, and demo prep
# CNCF Cloud Native Singapore Meetup
# Speaker: Gineesh Madapparambath | Architect, Red Hat

---

## Project Links

| Project | GitHub | Docs |
|---|---|---|
| kagent | https://github.com/kagent-dev/kagent | https://kagent.dev/docs/kagent |
| agentregistry | https://github.com/kagent-dev/agentregistry | https://kagent.dev/docs/kagent |
| agentgateway | https://github.com/agentgateway/agentgateway | https://agentgateway.dev |
| llm-d | https://github.com/llm-d/llm-d | — |
| kagent tools | https://kagent.dev/tools | — |
| kagent agents | https://kagent.dev/agents | — |
| Supported providers | https://kagent.dev/docs/kagent/supported-providers | — |
| API reference | https://kagent.dev/docs/kagent/resources/api-ref | — |
| Release notes | https://kagent.dev/docs/kagent/resources/release-notes | — |

---

## What is in this demo vs what is on the slides

| Component | In slides | In demo |
|---|---|---|
| kagent | ✅ | ✅ Running — k8s-troubleshooter agent |
| agentregistry | ✅ | ❌ Slides only |
| agentgateway | ✅ | ❌ Slides only |
| llm-d | ✅ | ❌ Slides only |
| OpenAI | — | ✅ ModelConfig provider |
| kagent-tool-server | — | ✅ RemoteMCPServer with K8s tools |
| kagent-grafana-mcp | — | ✅ Installed by --profile demo |

---

## Plain-English Explanations

### kagent

Kubernetes lets you declare *what should run* as YAML. kagent extends that idea to AI agents. Instead of writing Python scripts or LangChain code to build an agent, you write a YAML file — an `Agent` CRD — that says: use this LLM, follow these instructions, use these tools. The kagent operator running inside your cluster watches those CRDs, spins up the agent runtime, and manages the lifecycle. The agent lives inside Kubernetes, has access to the Kubernetes API, and can call tools like `k8s_get_pods` or `k8s_patch_resource` natively.

**In one sentence:** kagent makes AI agents first-class Kubernetes citizens.

---

### agentregistry

When developers start building agents, they pop up everywhere — laptops, random namespaces, scripts nobody knows about. You have no idea what agents exist, what tools they have access to, or whether they're governed. agentregistry solves that. It's a centralized catalog where agents, MCP tools, and agent skills are registered, versioned, and discoverable — like Docker Hub but for agents. Platform teams can see every agent in the organization, what it does, who owns it, and whether it was deployed through a governed workflow or snuck in through the back door.

**In one sentence:** agentregistry is the governance and discovery layer for your AI agents.

**On stage if asked:** *"agentregistry is what you'd add when you have multiple teams deploying agents and need governance. Think of it as the Harbor or Artifactory for your agents. Links are in the references slide."*

---

### agentgateway

When an agent calls a tool — querying a database or calling an internal API — that traffic needs to be secured, authenticated, and audited. Without a gateway, agents call tools directly with no visibility or control. agentgateway sits between agents and their tool backends, exactly like Envoy sits between services in a service mesh. It handles identity-based filtering (this agent can only call these tools), OAuth2 token exchange (scoped credentials per backend), and observability (every tool call is logged and traceable). It also supports MCP protocol natively.

**In one sentence:** agentgateway is the secure, observable proxy layer for agent-to-tool traffic.

**On stage if asked:** *"agentgateway is the next layer you'd add in production. Right now the agent calls tools directly — fine for a demo. In a real environment you'd put agentgateway in front of your MCP servers so every tool call is authenticated, filtered, and audited. Same idea as putting Istio or Envoy in front of your services."*

---

### How they fit together

```
Developer writes agent YAML
        ↓
[ agentregistry ]  — is this agent approved and governed?
        ↓
[ kagent ]         — run it inside Kubernetes
        ↓
[ agentgateway ]   — control what tools it can call and how
        ↓
[ Tools / MCP Servers ] — Kubernetes API, databases, internal services
```

Think of it as the same pattern as containers → Kubernetes → service mesh, but for AI agents.

---

## Q&A — Common Questions

### Q: You write a YAML — how does kagent know what to do? Does it need internet? What about air-gapped?

When you `kubectl apply -f agent.yaml`, the kagent operator (a standard Kubernetes controller) detects the new `Agent` CR and reconciles it. It spins up a pod running the kagent agent runtime — a container image already pulled when you ran `kagent install`.

- **Agent runtime image** — pulled once at install time. Air-gapped works if you mirror the image to an internal registry.
- **LLM provider** — needs network access to wherever your model is (OpenAI, Ollama on EC2, internal vLLM). Point `modelConfig` at an internal model server for fully air-gapped setup.
- **Tools** — call the Kubernetes API locally. No internet needed.

Fully air-gappable if you mirror the runtime image and run a local model.

---

### Q: The agent runs inside a common pod image?

Yes. When kagent reconciles your `Agent` CR it creates a standard Kubernetes `Deployment` — one pod per agent. That pod runs the kagent agent runtime container. You can see it:

```bash
kubectl get pods -n kagent
# k8s-troubleshooter-79db45c466-dkvm6   1/1   Running
```

That pod IS your agent. It's a normal Kubernetes workload — resource limits, node selectors, tolerations, all standard K8s primitives apply. The `declarative.deployment` section of the Agent CRD maps directly to pod spec fields.

---

### Q: What level of Kubernetes API access does the agent have? How does RBAC work?

kagent follows standard Kubernetes RBAC — nothing special, nothing bypassed.

When the agent pod starts, the kagent operator creates a **ServiceAccount** automatically (named after the agent). That ServiceAccount gets bound to a **Role** or **ClusterRole** defining what the agent can do.

The `--profile demo` install grants broad permissions for convenience. In production tighten this:

```yaml
# Read-only agent — diagnose only
rules:
  - apiGroups: [""]
    resources: ["pods", "events", "services"]
    verbs: ["get", "list"]

# Fix-capable agent — can also patch
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "patch"]
```

**Two levels of restriction:**
- **Tool list in Agent CRD** — what the agent is allowed to ask for
- **ServiceAccount RBAC** — what the agent is actually allowed to do at the API level

Both need to be configured for proper least-privilege.

**On stage talking point:**
> *"kagent doesn't invent a new security model. It uses the same RBAC your team already understands. The agent pod has a ServiceAccount, the ServiceAccount has a Role, the Role defines what's allowed. Same as any other workload in your cluster."*

---

### Q: Can I see the tools available in the cluster?

Tools are served via `RemoteMCPServer` resources:

```bash
# List all MCP servers
kubectl get remotemcpservers -n kagent

# Output from this demo:
NAME                 PROTOCOL          URL                                         ACCEPTED
kagent-grafana-mcp   STREAMABLE_HTTP   http://kagent-grafana-mcp.kagent:8000/mcp   True
kagent-tool-server   STREAMABLE_HTTP   http://kagent-tools.kagent:8084/mcp         True
```

- `kagent-tool-server` — all built-in K8s, Helm, Argo, Istio, Prometheus, Cilium tools
- `kagent-grafana-mcp` — Grafana dashboard and datasource tools (installed by --profile demo)

```bash
# See all discovered tools
kubectl describe remotemcpserver kagent-tool-server -n kagent

# Quick list of tool names
kubectl get remotemcpserver kagent-tool-server -n kagent \
  -o jsonpath='{.status.discoveredTools[*].name}' | tr ' ' '\n'
```

Or in the kagent dashboard: **View → Tool Servers**

Full tool list: https://kagent.dev/tools

---

### Q: Is agentregistry deployed in this demo?

No. agentregistry is covered in the slides (CNCF Ecosystem slide) but not deployed live. The demo focuses on kagent as the core runtime. agentregistry would be the next layer to add for multi-team governance.

GitHub: https://github.com/kagent-dev/agentregistry

---

### Q: Is agentgateway deployed in this demo?

No. agentgateway is covered in the slides but not deployed live. In the current demo the agent calls `kagent-tool-server` directly. In production you'd put agentgateway in front of those MCP servers for auth, filtering, and observability.

Note: `kagent-tool-server` is kagent's own internal MCP server — it is NOT agentgateway.

GitHub: https://github.com/agentgateway/agentgateway

---

### Q: What is MCP (Model Context Protocol)?

MCP is an open standard for how AI agents communicate with tools and external systems. Instead of each agent framework inventing its own tool interface, MCP defines a common protocol — like HTTP for the web, but for agent-to-tool communication. kagent supports MCP natively, which means any MCP-compatible tool server plugs straight in without custom code.

---

### Q: What is A2A (Agent-to-Agent)?

A2A is Google's open protocol for agents communicating with other agents. In complex workflows, one agent delegates sub-tasks to another specialist agent. A2A defines how those inter-agent calls are structured, authenticated, and traced. kagent supports A2A via the `a2aConfig` field in the Agent CRD.

---

### Q: How is kagent different from LangChain / CrewAI / AutoGen?

Those frameworks run in application code — Python libraries you import and wire up yourself. kagent runs in Kubernetes as a controller.

| | LangChain / CrewAI | kagent |
|---|---|---|
| Where it runs | Your app code | Kubernetes controller |
| Config | Python code | YAML CRD |
| Lifecycle mgmt | Your responsibility | Kubernetes operator |
| Scaling | Manual | HPA / Kubernetes native |
| RBAC | Custom | Standard K8s RBAC |
| GitOps | Hard | Native (Argo, Flux) |

kagent complements those frameworks — you can use LangChain inside a BYO agent and still deploy/govern it via kagent.

---

### Q: What LLM providers does kagent support?

OpenAI, Anthropic, Azure OpenAI, Google Gemini, Google Vertex AI, AWS Bedrock, Ollama, SAP AI Core, xAI (Grok), and any OpenAI-compatible endpoint.

This demo uses OpenAI for reliability on stage. The same agent runs identically with Ollama locally or any GPU-backed model — just change the `modelConfig` field.

Ref: https://kagent.dev/docs/kagent/supported-providers

---

### Q: Is kagent production-ready?

It's a CNCF Sandbox project — early stage but community-vetted and on a path to maturity. Solo.io (company behind Istio and Gloo) is the primary contributor. Red Hat, Google, and NVIDIA are part of the broader ecosystem. For production use today, carefully scope RBAC, consider the enterprise version from Solo.io, and test thoroughly. KubeCon NA 2026 (Atlanta, November) will be a landmark moment for the project.

---

## Demo Scenarios

| Scenario | File | Failure | Agent can fix? |
|---|---|---|---|
| Bad image tag | `broken-app.yaml` | ImagePullBackOff | ✅ Yes — patches image |
| Label mismatch | `myapp-deployment.yaml` | Service has no endpoints | ✅ Yes — patches Service selector |
| Crash loop | `crash-app.yaml` | CrashLoopBackOff | ❌ No — fake DB error, no real fix |

---

## Useful kubectl Commands

```bash
# Check all kagent resources
kubectl get agents,modelconfigs,remotemcpservers -n kagent

# Watch agent status
kubectl get agent -n kagent -w

# Check agent logs
kubectl logs -n kagent -l app.kubernetes.io/name=k8s-troubleshooter -f

# Check tool server logs
kubectl logs -n kagent deploy/kagent-tools -f

# List all available K8s tools
kubectl get remotemcpserver kagent-tool-server -n kagent \
  -o jsonpath='{.status.discoveredTools[*].name}' | tr ' ' '\n'

# Open dashboard
kubectl port-forward -n kagent service/kagent-ui 8082:8080
```