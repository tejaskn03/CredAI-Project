# CREDPAY

A credit-card bill-payment platform built the way enterprises build software: separate
customer and admin web apps, a dedicated identity server (Keycloak), secured Java and Python
services, versioned database migrations and automated tests - on your laptop, and on Azure
Kubernetes Service with a DevSecOps pipeline, Prometheus/Grafana, Slack alerting, Dynatrace
and an AIOps assistant.

## Documentation

Every document is in the [`docs/`](docs) folder, numbered in reading order.

| # | Document | What it covers |
| --- | --- | --- |
| 01 | [Architecture](docs/01-ARCHITECTURE.md) | how the system is designed: services, identity, data, deployment |
| 02 | [Local setup guide](docs/02-LOCAL-SETUP-GUIDE.md) | run everything on your laptop |
| 03 | [Student Azure setup](docs/03-STUDENT-AZURE-SETUP.md) | deploy everything in your own Azure account; destroy and rebuild |
| 04 | [Terraform infrastructure](docs/04-TERRAFORM-INFRASTRUCTURE.md) | the Azure resources Terraform creates |
| 05 | [CI pipeline](docs/05-CI-PIPELINE.md) | build, test, scan and push the images |
| 06 | [CD pipeline](docs/06-CD-PIPELINE.md) | infrastructure, platform and dev → qa → prod |
| 07 | [Kubernetes setup](docs/07-KUBERNETES-SETUP.md) | AKS, namespaces, Helm, Traefik routing, certificates, sslip.io |
| 08 | [Monitoring setup](docs/08-MONITORING-SETUP.md) | how Prometheus, Grafana, Alertmanager, Slack, Dynatrace and AIOps are set up |
| 09 | [Observability, alerts and runbooks](docs/09-OBSERVABILITY-ALERTS-RUNBOOKS.md) | what to do when an alert fires; connecting Slack, Dynatrace and Azure OpenAI |
| 10 | [Monitoring test cases](docs/10-MONITORING-TEST-CASES.md) | dashboard tour, break-it scenarios, Ask CredAI prompts, Slack notifications |
| 11 | [AI service (CredAI)](docs/11-AI-SERVICE.md) | the AIOps backend: endpoints, local development |
| 12 | [Admin portal](docs/12-ADMIN-PORTAL.md) | the administrators' web app |
| | [Infrastructure overview (HTML)](docs/INFRASTRUCTURE.html) | a visual page: reliability, high availability, uptime |

## Quick start (Windows)

```powershell
git config --global core.longpaths true
git clone https://dev.azure.com/bharathreddy3297/Flipkart-Capestone/_git/credwithaiops C:\code\credwithaiops
cd C:\code\credwithaiops

# once, in PowerShell "Run as administrator" - then restart and start Docker Desktop
Set-ExecutionPolicy -Scope Process Bypass -Force
.\scripts\setup\install-prereqs.ps1

# once, in a normal PowerShell window
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
.\scripts\setup\check-prereqs.ps1

# every time
.\scripts\start-all.ps1     # Customer Portal http://localhost:5173 · Admin Portal http://localhost:5174
.\scripts\stop-all.ps1
```

Demo accounts: `customer@credpay.local` / `Customer@123` and `admin@credpay.local` / `Admin@123`.
