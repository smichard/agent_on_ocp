# RHAIIS Deployment Workflow

Deploys the Red Hat AI Inference Server (RHAIIS) on OpenShift. RHAIIS is the supported, enterprise-packaged distribution of vLLM. It loads a model from Hugging Face Hub and exposes an OpenAI-compatible `/v1/chat/completions` endpoint.

## 1. Create Namespace

```bash
oc new-project rhaiis
```

## 2. Create Secrets

```bash
# Hugging Face access token (required for gated models; skip for public models like Granite)
oc create secret generic hf-secret \
  --from-literal=HF_TOKEN=<your_huggingface_token> \
  -n rhaiis

# API key that clients must present as a bearer token
oc create secret generic vllm-api-key-secret \
  --from-literal=VLLM_API_KEY=$(openssl rand -hex 32) \
  -n rhaiis
```

## 3. Apply Manifests

Edit `configmap.yaml` to set the model ID before applying. The value must match the `--served-model-name` argument in `deployment.yaml`.

```bash
oc apply -f configmap.yaml
oc apply -f pvc.yaml
oc apply -f deployment.yaml
oc apply -f service.yaml
oc apply -f route.yaml
```

## 4. Verify Startup

```bash
oc logs -f deployment/rhaiis-vllm -n rhaiis
```

The server is ready when the log shows `Application startup complete`.

## 5. Test the Endpoint

```bash
export RHAIIS_HOST=$(oc get route rhaiis-vllm -n rhaiis -o jsonpath='{.spec.host}')
export RHAIIS_API_KEY=$(oc get secret vllm-api-key-secret -n rhaiis \
  -o jsonpath='{.data.VLLM_API_KEY}' | base64 -d)
export MODEL=$(oc get configmap vllm-config -n rhaiis \
  -o jsonpath='{.data.MODEL_NAME}')

# List available models
curl -s https://$RHAIIS_HOST/v1/models \
  -H "Authorization: Bearer $RHAIIS_API_KEY" | jq .

# Send a chat completion request
curl -sS "https://${RHAIIS_HOST}/v1/chat/completions" \
  -H "Authorization: Bearer ${RHAIIS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [{"role": "user", "content": "What is OpenShift?"}],
    "temperature": 0.1,
    "max_tokens": 200
  }' | jq -r '.choices[0].message.content'
```
