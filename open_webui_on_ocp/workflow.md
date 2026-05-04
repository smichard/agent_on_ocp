# Open WebUI Deployment Workflow

Deploys Open WebUI on OpenShift and connects it to RHAIIS and Hermes Agent over the cluster-internal service network. Neither inference backend needs a public Route for this to work — all model traffic stays inside the cluster.

**Prerequisites:**
- RHAIIS running in the `rhaiis` namespace (see `../rhaiis/workflow.md`)
- Hermes Agent running in the `hermes` namespace (see `../hermes_on_ocp/workflow.md`)

## 1. Create Namespace

```bash
oc new-project open-webui
```

## 2. Create ServiceAccount and SCC

Open WebUI's container image runs as UID 0 by default. The `anyuid` SCC is required because OpenShift's default restricted SCC does not permit this.

```bash
oc apply -f scc.yaml
```

## 3. Create Secrets

The API keys must be ordered to match the backend URLs set in `deployment.yaml` (RHAIIS first, Hermes second).

```bash
export RHAIIS_API_KEY=$(oc get secret vllm-api-key-secret -n rhaiis \
  -o jsonpath='{.data.VLLM_API_KEY}' | base64 -d)
export HERMES_API_KEY=$(oc get secret hermes-api-secret -n hermes \
  -o jsonpath='{.data.API_SERVER_KEY}' | base64 -d)

oc create secret generic open-webui-api-keys \
  --from-literal=API_KEYS="${RHAIIS_API_KEY};${HERMES_API_KEY}" \
  -n open-webui

oc create secret generic open-webui-secret \
  --from-literal=WEBUI_SECRET_KEY=$(openssl rand -hex 32) \
  -n open-webui
```

## 4. Apply Manifests

```bash
oc apply -f pvc.yaml
oc apply -f deployment.yaml
oc apply -f service.yaml
oc apply -f route.yaml
```

## 5. First Login

```bash
oc get route open-webui -n open-webui -o jsonpath='{.spec.host}'
```

Open the hostname in a browser. The first account created on initial launch becomes the administrator.

To verify both backends are connected, go to **Settings > Connections** in the admin panel. Each entry should show a green status indicator.
