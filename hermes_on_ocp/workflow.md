# Hermes Agent Deployment Workflow

Deploys Hermes Agent on OpenShift and wires it to the RHAIIS inference server running in the `rhaiis` namespace. Traffic between Hermes and RHAIIS stays on the cluster-internal service network. OpenRouter is configured as an automatic fallback when the local model server is unavailable.

**Prerequisite:** RHAIIS must be running in the `rhaiis` namespace (see `../rhaiis/workflow.md`).

## 1. Create Namespace and ServiceAccount

Hermes runs as UID 10000. The `anyuid` SCC is required because OpenShift's default restricted SCC does not allow this.

```bash
oc new-project hermes

oc create serviceaccount hermes -n hermes

oc adm policy add-scc-to-user anyuid \
  -z hermes \
  -n hermes
```

## 2. Create Secrets

```bash
# vLLM bearer token — read from the rhaiis namespace
export RHAIIS_API_KEY=$(oc get secret vllm-api-key-secret -n rhaiis \
  -o jsonpath='{.data.VLLM_API_KEY}' | base64 -d)

oc create secret generic hermes-vllm-secret \
  --from-literal=OPENAI_API_KEY="${RHAIIS_API_KEY}" \
  -n hermes

# OpenRouter fallback key
oc create secret generic hermes-openrouter-secret \
  --from-literal=OPENROUTER_API_KEY=<your_openrouter_key> \
  -n hermes

# API key that clients must present when calling the Hermes endpoint
oc create secret generic hermes-api-secret \
  --from-literal=API_SERVER_KEY=$(openssl rand -hex 32) \
  -n hermes
```

## 3. Apply Manifests

Edit `configmap.yaml` to set `model.default` to the model ID served by RHAIIS and `fallback_model.model` to your preferred OpenRouter model before applying.

```bash
oc apply -f configmap.yaml
oc apply -f pvc.yaml
oc apply -f deployment.yaml
oc apply -f service.yaml
oc apply -f route.yaml
```

## 4. Verify Startup

```bash
oc logs -f deployment/hermes -n hermes
```

The server is ready when the log shows `[api_server] Listening on 0.0.0.0:8642`.

## 5. Test the Endpoint

```bash
export HERMES_HOST=$(oc get route hermes -n hermes -o jsonpath='{.spec.host}')
export HERMES_KEY=$(oc get secret hermes-api-secret -n hermes \
  -o jsonpath='{.data.API_SERVER_KEY}' | base64 -d)

# List available models
curl -sS "https://${HERMES_HOST}/v1/models" \
  -H "Authorization: Bearer ${HERMES_KEY}" | jq -r '.data[].id'

# Send a chat completion request
curl -sS "https://${HERMES_HOST}/v1/chat/completions" \
  -H "Authorization: Bearer ${HERMES_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-Coder-30B-A3B-Instruct",
    "messages": [{"role": "user", "content": "What is OpenShift?"}]
  }' | jq -r '.choices[0].message.content'
```

## 6. Update the Model

Edit `configmap.yaml` and set `model.default` to a different model served by RHAIIS, then apply and restart:

```bash
oc apply -f configmap.yaml
oc rollout restart deployment/hermes -n hermes
```
