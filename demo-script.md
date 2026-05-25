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
- [ ] kagent dashboard is open: `kagent dashboard` → http://localhost:8082
- [ ] Browser tab with dashboard is open and showing k8s-troubleshooter agent
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
> **Fix:** Update the deployment to use a valid image tag such as `nginx:latest` or `nginx:1.25`.

Say:
> "Root cause, evidence, fix. In plain English. No kubectl gymnastics.
> And this ran entirely on open source — kagent, Ollama, Llama 3.1,
> all inside a Kubernetes cluster."

---

### BEAT 4 — Close the loop (30 seconds, terminal)

Say:
> "Let me apply the fix the agent suggested."

Type:
```bash
kubectl set image deployment/broken-app web=nginx:latest
kubectl rollout status deployment/broken-app
kubectl get pods -n default
```

Pods go Running.

Say:
> "That's it. A Kubernetes-native AI agent, defined as a CRD,
> using built-in tools, powered by open source.
> No proprietary platform, no vendor lock-in.
> Just Kubernetes doing what Kubernetes does — but now with agents."

---

## FALLBACK PLAN (if something breaks on stage)

**If kagent dashboard won't load:**
Use the CLI instead:
```bash
kagent invoke -t "Why is the broken-app pod failing?" --agent k8s-troubleshooter -n kagent
```

**If the agent hangs or times out:**
Say: "Ollama is thinking — this is the CPU penalty I mentioned.
On a GPU instance this takes 2 seconds."
Then switch to the OpenAI fallback:
```bash
kubectl patch agent k8s-troubleshooter -n kagent \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/modelConfig/name","value":"openai-model-config"}]'
```

**If kind cluster is dead:**
Have a screenshot of a previous successful run saved.
It's a meetup, not a production system — the audience will understand.
Say: "Demo gods were not with me today — but here's what it looks like."

---

## AFTER THE TALK

Point audience to:
- github.com/iamgini/kagent-demo  ← this repo
- kagent.dev/docs/kagent/getting-started/quickstart
- discord.gg/Fu3k65f2k3  ← kagent community Discord
