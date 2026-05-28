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

## Sample output - Agent details

```shell
 $ kubectl get remotemcpservers -n kagent
NAME                 PROTOCOL          URL                                         ACCEPTED
kagent-grafana-mcp   STREAMABLE_HTTP   http://kagent-grafana-mcp.kagent:8000/mcp   True
kagent-tool-server   STREAMABLE_HTTP   http://kagent-tools.kagent:8084/mcp         True
```

```shell
$ kubectl get remotemcpserver kagent-tool-server -n kagent \
  -o jsonpath='{.status.discoveredTools[*].name}' | tr ' ' '\n'
argo_check_plugin_logs
argo_pause_rollout
argo_promote_rollout
argo_rollouts_list
argo_set_rollout_image
argo_verify_argo_rollouts_controller_install
argo_verify_gateway_plugin
argo_verify_kubectl_plugin_install
cilium_connect_to_remote_cluster
cilium_delete_key_from_kv_store
cilium_delete_pcap_recorder
cilium_delete_policy_rules
cilium_delete_service
cilium_delete_xdp_cidr_filters
cilium_disconnect_endpoint
cilium_disconnect_remote_cluster
cilium_display_encryption_state
cilium_display_policy_node_information
cilium_display_selectors
cilium_flush_ipsec_state
cilium_fqdn_cache
cilium_get_bpf_map
cilium_get_daemon_status
cilium_get_endpoint_details
cilium_get_endpoint_health
cilium_get_endpoint_logs
cilium_get_endpoints_list
cilium_get_identity_details
cilium_get_kv_store_key
cilium_get_pcap_recorder
cilium_get_service_information
cilium_install_cilium
cilium_list_bgp_peers
cilium_list_bgp_routes
cilium_list_bpf_map_events
cilium_list_bpf_maps
cilium_list_cluster_nodes
cilium_list_envoy_config
cilium_list_identities
cilium_list_ip_addresses
cilium_list_local_redirect_policies
cilium_list_metrics
cilium_list_node_ids
cilium_list_pcap_recorders
cilium_list_services
cilium_list_xdp_cidr_filters
cilium_manage_endpoint_config
cilium_manage_endpoint_labels
cilium_request_debugging_information
cilium_set_kv_store_key
cilium_show_cluster_mesh_status
cilium_show_configuration_options
cilium_show_dns_names
cilium_show_features_status
cilium_show_ip_cache_information
cilium_show_load_information
cilium_status_and_version
cilium_toggle_cluster_mesh
cilium_toggle_configuration_option
cilium_toggle_hubble
cilium_uninstall_cilium
cilium_update_pcap_recorder
cilium_update_service
cilium_update_xdp_cidr_filters
cilium_upgrade_cilium
cilium_validate_cilium_network_policies
datetime_get_current_time
helm_get_release
helm_list_releases
helm_repo_add
helm_repo_update
helm_uninstall
helm_upgrade
istio_analyze_cluster_configuration
istio_apply_waypoint
istio_delete_waypoint
istio_generate_manifest
istio_generate_waypoint
istio_install_istio
istio_list_waypoints
istio_proxy_config
istio_proxy_status
istio_remote_clusters
istio_version
istio_waypoint_status
istio_ztunnel_config
k8s_annotate_resource
k8s_apply_manifest
k8s_check_service_connectivity
k8s_create_resource
k8s_create_resource_from_url
k8s_delete_resource
k8s_describe_resource
k8s_execute_command
k8s_generate_resource
k8s_get_available_api_resources
k8s_get_cluster_configuration
k8s_get_events
k8s_get_pod_logs
k8s_get_resource_yaml
k8s_get_resources
k8s_label_resource
k8s_patch_resource
k8s_patch_status
k8s_remove_annotation
k8s_remove_label
k8s_rollout
k8s_scale
kubescape_check_health
kubescape_get_application_profile
kubescape_get_configuration_scan
kubescape_get_network_neighborhood
kubescape_get_vulnerability_details
kubescape_list_application_profiles
kubescape_list_configuration_scans
kubescape_list_network_neighborhoods
kubescape_list_vulnerabilities
kubescape_list_vulnerability_manifests
prometheus_label_names_tool
prometheus_promql_tool
prometheus_query_range_tool
prometheus_query_tool
prometheus_targets_tool
shell
```

---


## References

- kagent docs: https://kagent.dev/docs/kagent
- kagent GitHub: https://github.com/kagent-dev/kagent
- CNCF project page: https://www.cncf.io/projects/kagent/
- Supported LLM providers: https://kagent.dev/docs/kagent/supported-providers

https://agentgateway.dev/