# Kubernetes deployment

These manifests deploy the biodiversity app to the NRP Nautilus cluster, in the `biodiversity` namespace, at https://nature.nrp-nautilus.io.

## Files

- `deployment.yaml` runs nginx. An init container clones this repository and copies `index.html`, `layers-input.json` and `system-prompt.md` into the web root.
- `configmap-nginx.yaml` holds two ConfigMaps:
  - `biodiversity-maplibre-config` is the template for `config.json`, which the browser reads. It sets the default LLM (`llm_model`) and the model picker list (`llm_models`). It contains no secrets.
  - `biodiversity-maplibre-nginx` is the nginx server config.
- `service.yaml` is the ClusterIP Service.
- `ingress.yaml` is the HAProxy ingress for `nature.nrp-nautilus.io`.

## How the keys stay out of the browser

The browser talks only to this app's own nginx, at two same-origin paths:

- `/api/llm/` goes to the in-cluster `open-llm-proxy` Service. nginx sets the `Authorization` header from `PROXY_KEY`.
- `/api/mcp` goes to the in-cluster DuckDB MCP server. nginx sets the `Authorization` header from `MCP_AUTH_TOKEN`.

At startup the nginx container runs `envsubst` over both templates. `config.json` gets only `MCP_SERVER_URL`. The nginx config gets `PROXY_KEY` and `MCP_AUTH_TOKEN`, so neither key is ever served to the browser.

Both keys come from Secrets that already exist in the namespace and are shared with the other apps:

| Variable | Secret | Key |
|---|---|---|
| `PROXY_KEY` | `open-llm-proxy-secrets` | `proxy-key` |
| `MCP_AUTH_TOKEN` | `duckdb-mcp-auth` | `MCP_AUTH_TOKEN` |

This app does not create or own either Secret. Never put a key in this repository.

## Deploy

```bash
kubectl apply -f k8s/configmap-nginx.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

## Update

The init container clones the repository on every pod start, so a restart picks up changes to `index.html`, `layers-input.json` and `system-prompt.md`:

```bash
kubectl rollout restart deployment/biodiversity-maplibre -n biodiversity
```

ConfigMap changes, such as a new default model or picker entry, also need a restart, because nginx renders its templates only at startup:

```bash
kubectl apply -f k8s/configmap-nginx.yaml
kubectl rollout restart deployment/biodiversity-maplibre -n biodiversity
kubectl rollout status deployment/biodiversity-maplibre -n biodiversity
```

A change to either shared Secret also needs a restart, because the keys are read when the container starts.

## Check

```bash
kubectl get pods -n biodiversity -l app=biodiversity-maplibre
kubectl logs -n biodiversity -l app=biodiversity-maplibre -c git-clone
curl -s https://nature.nrp-nautilus.io/config.json
```

`/health` returns 200 and is what the liveness and readiness probes use.
