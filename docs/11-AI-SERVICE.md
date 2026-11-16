# CredAI Service

The AI Operations Assistant backend for CredPay - a FastAPI
microservice that reasons over telemetry already collected by
Prometheus, Kubernetes, and Azure Monitor to answer operational
questions in plain language.

The code is in the [`ai-service/`](../ai-service) folder. This service **adds** to
CredPay; it does not modify any existing service, Deployment, or pipeline.

## What CredAI is not

- Not a dashboard. Grafana already exists for that (`charts/credpay/dashboards/`).
- Not a replacement for Prometheus/Kubernetes/Azure Monitor - it only
  *reads* from all three; none of them changed.
- Not a chatbot with general knowledge - every answer is grounded in
  CredPay's own telemetry, or the model says so.

## Quick start (local development)

```bash
cd ai-service
python -m venv .venv
.venv/Scripts/activate   # Windows; use `source .venv/bin/activate` on Linux/macOS
pip install -r requirements.txt

# Minimum required environment variables (the full list, with defaults, is in
# app/config/settings.py):
export OPENAI_ENDPOINT="https://your-resource.services.ai.azure.com/api/projects/your-project/openai/v1/responses"
export OPENAI_KEY="your-key"
export OPENAI_DEPLOYMENT="your-deployment-name"
export OPENAI_API_VERSION="2025-08-07"

uvicorn app.main:app --reload --port 8010
```

Then open `http://localhost:8010/docs` for interactive Swagger UI.

Outside a cluster, the Kubernetes, Azure Monitor, and Log Analytics
clients disable themselves automatically - only Prometheus (if
reachable) and Azure OpenAI are required for local development.

## Running against the real cluster

Prometheus is only reachable at `monitoring-prometheus.monitoring.svc.cluster.local:9090`
*from inside* the AKS cluster. For local development against real data,
port-forward it first:

```bash
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090
```

Then set `PROMETHEUS_URL=http://localhost:9090` before starting Uvicorn.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/ai/health` | Service + dependency health |
| POST | `/api/ai/chat` | Free-text question → AI answer |
| POST | `/api/ai/root-cause` | Symptom description → root-cause analysis |
| GET | `/api/ai/prometheus` | Raw PromQL passthrough (debugging) |
| GET | `/api/ai/kubernetes` | Raw cluster state passthrough (debugging) |
| GET | `/api/ai/cluster-summary` | AI-generated cluster health summary |
| GET | `/api/ai/business-summary` | AI-generated business health summary |

Full request/response schemas: the Swagger UI at `/docs` (above).

Deployment: part of the CREDPAY Helm chart (`charts/credpay/templates/ai-service.yaml`),
enabled in prod (`aiops.enabled`). It powers the admin portal's Monitoring page through
`GET /api/ai/overview` and `POST /api/ai/insight` (admin-only Keycloak token). See
[09-OBSERVABILITY-ALERTS-RUNBOOKS.md](09-OBSERVABILITY-ALERTS-RUNBOOKS.md) for the whole monitoring and AIOps setup.
