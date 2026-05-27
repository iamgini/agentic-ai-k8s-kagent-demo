# Demo Script — Agentic AI on Kubernetes
# CNCF Cloud Native Singapore Meetup
# Speaker: Gineesh Madapparambath
#
# Total demo time: ~5 minutes
# Rehearse at least 3 times before the talk. Know the timing cold.

---

## PRE-TALK CHECKLIST (30 min before)

- [ ] kind cluster is running: `kubectl cluster-info --context kind-kagent-demo`
- [ ] kagent pods are Running: `kubectl get pods -n kagent`
- [ ] broken-app is deployed and showing ImagePullBackOff: `kubectl get pods -n default`
- [ ] kagent dashboard is open: `kubectl port-forward -n kagent service/kagent-ui 8082:8080`
- [ ] Browser tab with dashboard is open and showing k8s-troubleshooter agent
- [ ] OpenAI model warmed up — send one test message before going on stage
- [ ] Font size on terminal is large (min 18pt) — audience at the back can't read small text
- [ ] Zoom in on browser (Cmd + to 125%)
- [ ] Disable notifications (Do Not Disturb on)
- [ ] Close Slack, email, everything except terminal + browser

---

## STAGE SETUP (what should be on screen before you start the demo)

Two windows side by side:
- LEFT: Terminal (dark theme, large font)
- RIGHT: Browser with kagent dashboard open on k8s-troubleshooter agent

---

## DEMO FLOW

### BEAT 1 — Set the scene (30 seconds, terminal)

Say:
> "We have a Kubernetes cluster running. Let's check what's happening."

Type:
```bash
kubectl get pods -n default
```

Expected output shows: `broken-app-xxxx   0/1   ImagePullBackOff   0   2m`

Say:
> "There it is — a pod stuck in ImagePullBackOff. In the old world, I'd start
> running kubectl describe, kubectl logs, google the error, patch the YAML.
> Today, I'm going to ask my agent."

---

### BEAT 2 — Show the agent is a Kubernetes object (30 seconds, terminal)

Say:
> "The agent itself is just a Kubernetes resource. Let me show you."

Type:
```bash
kubectl get agent -n kagent
kubectl describe agent k8s-troubleshooter -n kagent | head -40
```

Point out: modelConfig, systemPrompt, tools list.

Say:
> "No code. No scripts. Just a YAML file declaring what the agent can do
> and how it should behave. That's the cloud native way."

---

### BEAT 2.5 — Talk about model flexibility (20 seconds, no typing needed)

Say:
> "One thing worth noting — kagent is model-agnostic. You can plug in any
> LLM provider: local Ollama on your laptop, a GPU instance running Llama or
> Qwen, or an enterprise model like OpenAI or Anthropic. You change one field
> in a YAML file — the ModelConfig — and the agent switches providers.
> For today's demo I'm using OpenAI for reliability, but the same agent
> runs identically on a fully open source stack."

---

### BEAT 3 — Ask the agent (3 minutes, browser dashboard)

Switch to browser. Click on k8s-troubleshooter agent.

Type in the chat box:
> "Why is the broken-app pod failing in the default namespace and how do I fix it?"

While the agent is thinking (narrate what's happening):
> "The agent is now calling k8s_get_resources to list pods...
> now k8s_get_events to check what happened...
> now k8s_describe_resource to look at the pod spec..."

Point out the tool calls expanding in the UI as they complete.

Expected agent response structure:
> **Root cause:** The pod is failing because the container image `nginx:totally-does-not-exist` does not exist in the registry.
> **Evidence:** Events show `Failed to pull image: not found`. The pod spec specifies an invalid image tag.
> **Fix:** Update the deployment to use a valid image tag such as `nginx:latest`.

Say:
> "Root cause, evidence, fix. In plain English. No kubectl gymnastics."

---

### BEAT 4 — Apply the fix (30 seconds, terminal)

Say:
> "The agent diagnosed the problem. Now let me apply the fix it suggested."

Type:
```bash
kubectl set image deployment/broken-app web=nginx:latest
kubectl rollout status deployment/broken-app
kubectl get pods -n default
```

Pods go Running.

Say:
> "That's it. A Kubernetes-native AI agent, defined as a CRD,
> using built-in tools, connected to any LLM you choose.
> No proprietary platform, no vendor lock-in.
> Just Kubernetes doing what Kubernetes does — but now with agents."

---

## FALLBACK PLAN (if something breaks on stage)

**If kagent dashboard won't load (502 error):**
Check the port-forward is running:
```bash
kubectl port-forward -n kagent service/kagent-ui 8082:8080
```
Or use the CLI instead:
```bash
kagent invoke -t "Why is the broken-app pod failing?" --agent k8s-troubleshooter -n kagent
```

**If OpenAI returns model_not_found error:**
Switch model to gpt-4o-mini:
```bash
kubectl patch modelconfig openai-model-config -n kagent \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/model","value":"gpt-4o-mini"}]'
```

**If agent shows Accepted: False:**
Check the modelconfig secret exists:
```bash
kubectl get secret kagent-openai -n kagent
kubectl get modelconfig openai-model-config -n kagent -o yaml
```

**If kind cluster is dead:**
Have a screenshot of a previous successful run saved.
It's a meetup, not a production system — the audience will understand.
Say: "Demo gods were not with me today — but here's what it looks like."

---

## AFTER THE TALK

Point audience to:
- github.com/iamgini/agentic-ai-k8s-kagent-demo  ← this repo
- kagent.dev/docs/kagent/getting-started/quickstart
- discord.gg/Fu3k65f2k3  ← kagent community Discord

---

## BONUS SCENARIO — CrashLoopBackOff (optional, if time allows)

**Setup (do this before the talk, alongside broken-app):**
```bash
kubectl apply -f crash-app.yaml
kubectl get pods -n default   # should show CrashLoopBackOff after ~30s
```

---

### BEAT 1 — Show both failures

Say:
> "Let's look at another common failure — this one's even more frustrating
> because the image pulls fine but the container keeps dying."

Type:
```bash
kubectl get pods -n default
```

Expected output shows both:
```
broken-app-xxxx   0/1   ImagePullBackOff   0   2m
crash-app-xxxx    0/1   CrashLoopBackOff   3   90s
```

Say:
> "CrashLoopBackOff. The image pulled fine, Kubernetes started the container,
> but something inside is failing and it keeps restarting. Let's ask the agent."

---

### BEAT 2 — Ask the agent

Switch to browser. Click on k8s-troubleshooter agent. Start a new chat.

Type in the chat box:
> "Why is crash-app failing in the default namespace?"

This time the agent will use `k8s_get_pod_logs` — point that out:

Say:
> "Notice the agent is now reading the actual container logs — something
> you'd normally do with kubectl logs. It found our error message:
> 'Database connection failed'. Root cause identified."

Expected agent response:
> **Root cause:** The container is exiting immediately with exit code 1 due to an application error.
> **Evidence:** Pod logs show `ERROR: Database connection failed`. The container command exits after printing the error.
> **Fix:** Fix the database connection configuration or environment variables for the application.

---

### BEAT 3 — Close the loop

Say:
> "Two completely different failure modes — ImagePullBackOff and CrashLoopBackOff.
> Same agent. Same interface. Just natural language.
> That's what it means to run AI agents the cloud native way."

**Cleanup after demo:**
```bash
kubectl delete -f crash-app.yaml
kubectl delete -f broken-app.yaml
```