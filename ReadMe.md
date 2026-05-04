# Agent on OCP

Deployment files for running a self-hosted AI stack on OpenShift. The stack consists of three components, each in its own namespace, connected over the cluster-internal service network.

## Components

### [rhaiis/](rhaiis/) — Red Hat AI Inference Server

Deploys RHAIIS, the supported enterprise distribution of vLLM, into the `rhaiis` namespace. It loads a model from Hugging Face Hub and exposes an OpenAI-compatible `/v1/chat/completions` endpoint on port 8000. A TLS-terminated OpenShift Route provides optional external access.

See [rhaiis/workflow.md](rhaiis/workflow.md) for deployment steps.

### [hermes_on_ocp/](hermes_on_ocp/) — Hermes Agent

Deploys Hermes Agent into the `hermes` namespace. Hermes talks to the RHAIIS inference server over the internal cluster DNS name `rhaiis-vllm.rhaiis.svc.cluster.local:8000` and falls back automatically to OpenRouter when the local server is unavailable. It exposes an OpenAI-compatible API on port 8642, secured with a bearer token.

Requires RHAIIS to be running first. See [hermes_on_ocp/workflow.md](hermes_on_ocp/workflow.md) for deployment steps.

### [open_webui_on_ocp/](open_webui_on_ocp/) — Open WebUI

Deploys Open WebUI into the `open-webui` namespace and connects it to both RHAIIS and Hermes over internal cluster DNS. Neither backend needs a public Route. The only external-facing Route is the Open WebUI browser interface.

Requires RHAIIS and Hermes to be running first. See [open_webui_on_ocp/workflow.md](open_webui_on_ocp/workflow.md) for deployment steps.

## Architecture

```
                        ┌─────────────────────────┐
                        │  open-webui namespace   │
                        │  Open WebUI (:8080)     │
                        └────────┬────────┬────────┘
                   cluster DNS   │        │  cluster DNS
                                 ▼        ▼
          ┌──────────────────────┐      ┌──────────────────────┐
          │  rhaiis namespace    │      │  hermes namespace    │
          │  RHAIIS/vLLM (:8000) │      │  Hermes Agent (:8642)│
          └──────────────────────┘      └──────┬───────────────┘
                                               │  fallback
                                               ▼
                                          OpenRouter
```

## Prerequisites

- OpenShift cluster with at least one GPU-enabled worker node
- Node Feature Discovery (NFD) Operator installed
- NVIDIA GPU Operator installed
- OpenShift CLI (`oc`) installed and logged in
- Hugging Face access token (for gated models)
- OpenRouter API key (for the Hermes fallback)

## References

- [Red Hat AI Inference Server documentation](https://docs.redhat.com/en/documentation/red_hat_ai_inference_server/3.4)
- [Hermes Agent documentation](https://hermes-agent.nousresearch.com/docs/)
- [Open WebUI](https://openwebui.com/)
