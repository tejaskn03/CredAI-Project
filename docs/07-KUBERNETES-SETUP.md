# CredPay on Kubernetes

How the AKS cluster is set up, where every YAML file lives, how it is packaged and
installed, how Traefik routes the traffic, how the TLS certificates are issued, and what
the `sslip.io` domain is.

Everything here is installed by the CD pipeline (`pipelines/credpay-cd.yml`) - nothing is
applied by hand. See [06-CD-PIPELINE.md](06-CD-PIPELINE.md) for the pipeline itself.

- [1. The big picture](#1-the-big-picture)
- [2. The AKS cluster](#2-the-aks-cluster)
- [3. Namespaces - what runs where](#3-namespaces---what-runs-where)
- [4. Where the YAML files are](#4-where-the-yaml-files-are)
- [5. How it is packaged and installed](#5-how-it-is-packaged-and-installed)
- [6. Routing with Traefik](#6-routing-with-traefik)
- [7. TLS certificates (cert-manager + Let's Encrypt)](#7-tls-certificates-cert-manager--lets-encrypt)
- [8. The domain: sslip.io](#8-the-domain-sslipio)
- [9. Configuration and secrets](#9-configuration-and-secrets)
- [10. Security settings](#10-security-settings)
- [11. Useful commands](#11-useful-commands)

---

## 1. The big picture

```
 Browser
   │  https://prod.20-104-252-247.sslip.io
   ▼
 DNS (sslip.io) ── answers with the IP written in the name: 20.104.252.247
   ▼
 Azure static public IP  pip-credpay-ingress   (made by Terraform)
   ▼
 Azure Load Balancer      (created by AKS for Traefik's Service)
   ▼
┌──────────────────────────── AKS cluster aks-credpay ────────────────────────────┐
│                                                                                  │
│  namespace traefik      Traefik  ── reads every Ingress, terminates TLS          │
│         │                  │  certificates from Secrets made by cert-manager     │
│         ▼                  ▼                                                     │
│  credpay-dev / credpay-qa / credpay-prod        (same Helm chart, 3 times)       │
│     customer-portal  admin-portal  user-service  payment-service  keycloak       │
│     ai-service (prod)                                                            │
│                                                                                  │
│  monitoring   Prometheus, Alertmanager (→ Slack), Grafana                        │
│  dynatrace    Dynatrace Operator, OneAgent, ActiveGate        (optional)         │
│  cert-manager issues and renews the certificates                                 │
└──────────────────────────────────────────────────────────────────────────────────┘
         │                                         │
         ▼                                         ▼
 Azure Database for PostgreSQL             Azure Container Registry
 (one server, databases per env)           (images, pulled by the nodes)
```

## 2. The AKS cluster

The cluster is created by Terraform ([`terraform/modules/aks/main.tf`](../terraform/modules/aks/main.tf))
in the pipeline's **Infra** stage.

| Setting | Value | Why |
| --- | --- | --- |
| Name / resource group | `aks-credpay` in `rg-credpay` | |
| Node resource group | `rg-credpay-aks-nodes` | AKS keeps the VMs, disks, load balancer and the ingress IP here |
| System node pool | `system`, Standard_D2alds_v6, autoscaling 2-5 nodes | runs everything by default |
| User node pool | `user`, Standard_D2s_v6, autoscaling **0**-4 nodes | extra room; costs nothing until it is needed |
| Max pods per node | 60 | |
| Networking | Azure CNI, Azure network policy, Standard load balancer | pods get VNet IPs (`snet-aks`, 10.0.0.0/20) |
| Service IP range | `10.240.0.0/16`, DNS `10.240.0.10` | kept apart from the VNet so they never overlap |
| Login | **Entra ID only**, Azure RBAC, local accounts **off** | nobody has a shared admin certificate |
| Admins | the pipeline identity + `adminObjectIds` in `credpay-cd.yml` | role *Azure Kubernetes Service RBAC Cluster Admin* |
| Registry access | `AcrPull` role for the nodes' identity | no image pull secrets needed |
| Upgrades | `stable` channel, node images `NodeImage`, surge 33% | patched automatically, one node in three at a time |
| Logs | Container Insights → Log Analytics `log-credpay` | |
| OIDC issuer + Workload Identity | enabled | ready for the Key Vault CSI driver later |
| Ingress IP | static public IP `pip-credpay-ingress` | survives reinstalling Traefik; DNS keeps pointing at it |

Kubernetes access in the pipeline goes through [`scripts/cd/aks-login.sh`](../scripts/cd/aks-login.sh):
`az aks get-credentials` + `kubelogin convert-kubeconfig -l azurecli` turns the Azure CLI
sign-in into a Kubernetes token.

## 3. Namespaces - what runs where

| Namespace | What | Installed by |
| --- | --- | --- |
| `traefik` | Traefik ingress controller (1 replica) | Platform stage, Helm |
| `cert-manager` | cert-manager, its webhook and CA injector | Platform stage, Helm |
| `monitoring` | Prometheus, Alertmanager, Grafana, node exporter, kube-state-metrics | Platform stage, Helm |
| `dynatrace` | Dynatrace Operator, OneAgent (one per node), ActiveGate | Platform stage, Helm - **only when the Dynatrace secrets are in Key Vault** |
| `credpay-dev` | the app: 1 replica of each | Deploy dev stage, Helm |
| `credpay-qa` | the app: 1 replica of each, a bit less memory | Deploy qa stage, Helm (after approval) |
| `credpay-prod` | the app: 2 replicas, autoscaling, PodDisruptionBudgets, monitoring, ai-service | Deploy prod stage, Helm (after approval) |
| `kube-system` | AKS's own components (CoreDNS, CSI drivers, Azure agents) | AKS |

dev, qa and prod share one cluster and one PostgreSQL server, separated by namespace and
by database (`credpay_dev`, `keycloak_dev`, ...).

## 4. Where the YAML files are

```
k8s/platform/                         CLUSTER-WIDE: installed once per cluster (Platform stage)
  traefik-values.yaml                   values for the Traefik Helm chart
  cert-manager-values.yaml              values for the cert-manager Helm chart
  cluster-issuers.yaml                  3 ClusterIssuers: letsencrypt-prod / -staging / selfsigned
  monitoring/
    kube-prometheus-stack-values.yaml   values for Prometheus + Alertmanager + Grafana
  dynatrace/
    dynakube.yaml                       DynaKube: OneAgent + ActiveGate (optional)

charts/credpay/                       THE APPLICATION: one Helm chart, used for dev, qa and prod
  Chart.yaml                            chart name and version
  values.yaml                           defaults for every environment
  values-dev.yaml                       dev: smallest
  values-qa.yaml                        qa: same, less memory
  values-prod.yaml                      prod: replicas, autoscaling, PDBs, monitoring, AIOps
  templates/
    _helpers.tpl                        host names, labels, DB names (shared snippets)
    configmap.yaml                      ConfigMap credpay-config: URLs, issuer, DB names
    keycloak.yaml                       Deployment + Service
    user-service.yaml                   Deployment + Service
    payment-service.yaml                Deployment + Service
    portals.yaml                        customer-portal + admin-portal: Deployments + Services
    ai-service.yaml                     Deployment, Service, ServiceAccount, Role, RoleBinding (prod)
    ingress.yaml                        the Ingress: 3 hosts, 1 certificate
    scaling.yaml                        HorizontalPodAutoscalers + PodDisruptionBudgets (prod)
    monitoring.yaml                     ServiceMonitor, PrometheusRule (alerts), Grafana dashboard
  dashboards/credpay-overview.json      the Grafana dashboard

scripts/cd/                           THE GLUE: what the pipeline runs
  aks-login.sh                          kubectl access (Entra ID via kubelogin)
  install-observability.sh              monitoring stack + Dynatrace
  monitoring-values.sh                  generates Grafana host + Alertmanager → Slack values
  read-optional-secrets.sh              Slack / Dynatrace / OpenAI from Key Vault (each optional)
  resolve-images.sh                     which image tag to deploy for each service
  deploy-app.sh                         namespace, secrets, helm upgrade, smoke tests

pipelines/
  credpay-cd.yml                        Infra → Platform → dev → qa → prod (chart versions pinned here)
  templates/deploy-env.yml              one environment's deploy stage (used 3 times)
```

## 5. How it is packaged and installed

There are **two layers**, both installed with Helm.

### 5.1 Platform layer - third-party Helm charts with our values files

We do not write YAML for Traefik, cert-manager or Prometheus ourselves. We install their
official Helm charts at a **pinned version** and only keep **our settings** in
`k8s/platform/*-values.yaml`.

| Component | Chart (repository) | Version (`credpay-cd.yml`) | Our values / manifests |
| --- | --- | --- | --- |
| Traefik | `traefik/traefik` (traefik.github.io/charts) | 41.7.1 | `traefik-values.yaml` |
| cert-manager | `jetstack/cert-manager` (charts.jetstack.io) | v1.21.2 | `cert-manager-values.yaml` (+ CRDs) |
| ClusterIssuers | - (plain `kubectl apply`) | - | `cluster-issuers.yaml` |
| Prometheus, Alertmanager, Grafana | `prometheus-community/kube-prometheus-stack` | 92.13.1 | `monitoring/kube-prometheus-stack-values.yaml` + a generated file |
| Dynatrace Operator | `oci://public.ecr.aws/dynatrace/dynatrace-operator` | 1.11.1 | `dynatrace/dynakube.yaml` (`kubectl apply`) |

What the Platform stage runs, in order:

```bash
helm upgrade --install traefik traefik/traefik -n traefik \
  -f k8s/platform/traefik-values.yaml \
  --set service.annotations."service\.beta\.kubernetes\.io/azure-pip-name"=pip-credpay-ingress
helm upgrade --install cert-manager jetstack/cert-manager -n cert-manager \
  -f k8s/platform/cert-manager-values.yaml
kubectl apply -f k8s/platform/cluster-issuers.yaml
# then scripts/cd/install-observability.sh:
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack -n monitoring \
  -f k8s/platform/monitoring/kube-prometheus-stack-values.yaml -f <generated values>
helm upgrade --install dynatrace-operator oci://.../dynatrace-operator -n dynatrace   # if configured
kubectl apply -f <dynakube.yaml with the Dynatrace URL filled in>                       # if configured
```

`helm upgrade --install` is **idempotent**: on a fresh cluster it installs, on an existing
one it upgrades, and when nothing changed it does nothing. That is why every pipeline run
can safely run the Platform stage again.

The `azure-pip-name` annotation tells AKS to give Traefik's LoadBalancer Service the static
IP that Terraform created - the stage fails if Traefik ends up with any other IP.

### 5.2 Application layer - our own chart `charts/credpay`

One chart, deployed three times into three namespaces. The difference between the
environments is **only values**, layered like this (later wins):

```
values.yaml            defaults (1 replica, no autoscaling, monitoring off)
  + values-<env>.yaml  per environment (prod: 2 replicas, HPA 2-4, PDBs, monitoring, AIOps)
  + --set ...          per deployment, from the pipeline:
                         domain        20-104-252-247.sslip.io
                         tls.issuer    letsencrypt-prod
                         <service>.image  creddemoacr.azurecr.io/credpay/<service>:1.0.<build>
```

What [`scripts/cd/deploy-app.sh`](../scripts/cd/deploy-app.sh) does for one environment:

1. **Images** - [`resolve-images.sh`](../scripts/cd/resolve-images.sh) (dev only) picks
   each service's image: the one the triggering CI run pushed, otherwise the newest tag in
   ACR. The list is published as the artifact `images`; **qa and prod deploy exactly that
   list** ("build once, promote") - never a newer, untested tag.
2. **Render** - `helm template` writes the final YAML to `rendered-<env>.yaml` (published
   as a pipeline artifact, so you can see exactly what was applied).
3. **Namespace** - creates `credpay-<env>` and labels it with the **restricted** Pod
   Security Standard.
4. **Secrets** - creates `credpay-db`, `keycloak-admin` and (prod) `credpay-aiops` from Key
   Vault with `kubectl`. Secrets never go through Helm, so they are never stored in a Helm
   release record.
5. **Install** - `helm upgrade --install credpay charts/credpay -n credpay-<env> --atomic
   --wait --timeout 15m --history-max 10`. `--atomic` rolls back automatically if the new
   pods do not become ready; the last 10 releases are kept for `helm rollback`.
6. **Certificate** - waits until cert-manager has issued `credpay-tls`.
7. **Smoke tests** - 10 HTTP checks (11 in prod, with the AIOps API) through the real public URLs (HTTP→HTTPS redirect,
   portals, Keycloak issuer, APIs answer 401 without a login, ...). A failure fails the stage.

### 5.3 What else is imported

| What | Where it comes from |
| --- | --- |
| Keycloak realm `credpay` (clients, roles, password policy, lock-out, demo users) | [`infra/keycloak/realm/credpay-realm.json`](../infra/keycloak/realm/credpay-realm.json), built into the Keycloak image and imported on Keycloak's **first** start in each environment |
| Grafana dashboard | `charts/credpay/dashboards/credpay-overview.json` → ConfigMap with label `grafana_dashboard: "1"` → loaded by Grafana's sidecar |
| Alert rules | `PrometheusRule` in `templates/monitoring.yaml` → loaded by Prometheus |
| Prometheus scraping | `ServiceMonitor` in `templates/monitoring.yaml` (prod only) |

## 6. Routing with Traefik

### 6.1 Why Traefik

The usual choice, ingress-nginx, was retired in March 2026
([announcement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)). Traefik
reads the same **standard Kubernetes `Ingress`** objects, so the app's YAML is not tied to
Traefik - only `ingressClassName: traefik` names it.

### 6.2 How Traefik is configured ([`traefik-values.yaml`](../k8s/platform/traefik-values.yaml))

| Setting | Effect |
| --- | --- |
| `ingressClass: traefik`, default class | every Ingress in the cluster is served by Traefik |
| `providers.kubernetesIngress` on, `kubernetesCRD` **off** | only standard Ingress objects - no Traefik-specific `IngressRoute` |
| `publishedService` | `kubectl get ingress` shows the public IP |
| port `web` (80) → redirect to `websecure` (443), permanent | all HTTP is redirected to HTTPS |
| `service.type: LoadBalancer`, `externalTrafficPolicy: Local` | Azure LB in front; the client's real IP is kept (X-Forwarded-For) |
| access log on | each request is logged: `kubectl logs -n traefik deploy/traefik` |

### 6.3 The routes ([`charts/credpay/templates/ingress.yaml`](../charts/credpay/templates/ingress.yaml))

**One Ingress per environment, three hosts, one TLS certificate.** For prod with the IP
`20.104.252.247`:

| Host | Path | → Service (port `http`) |
| --- | --- | --- |
| `prod.20-104-252-247.sslip.io` | `/api/payment` | payment-service |
| (customer portal) | `/api` | user-service |
| | `/` | customer-portal |
| `admin.prod.20-104-252-247.sslip.io` | `/api/ai` | ai-service (prod only) |
| (admin portal) | `/api/admin/payments` | payment-service |
| | `/api/payment` | payment-service |
| | `/api` | user-service |
| | `/` | admin-portal |
| `auth.prod.20-104-252-247.sslip.io` | `/` | keycloak |
| `grafana.20-104-252-247.sslip.io` | `/` | Grafana (namespace `monitoring`, from the kube-prometheus-stack values) |

**The longest matching path wins.** `/api/payment/history` matches both `/api/payment` and
`/api`; the longer one wins, so it goes to payment-service. `/api/cards` only matches `/api`,
so it goes to user-service. Everything else falls through to `/` - the portal.

A request, step by step:

```
GET https://prod.20-104-252-247.sslip.io/api/payment/health
 1. DNS: sslip.io returns 20.104.252.247
 2. Azure LB → Traefik pod (port 443)
 3. Traefik decrypts TLS with the certificate in Secret credpay-tls
 4. Host prod.… + longest path /api/payment → Service payment-service (credpay-prod)
 5. Service → one of the payment-service pods
```

The portals call the APIs on **their own host** (`/api/...`), so the browser never makes a
cross-origin request - no CORS configuration is needed. The services check Keycloak tokens:
the token's issuer is the public URL (`https://auth.<env>.<domain>/realms/credpay`), and the
signing keys are fetched **inside** the cluster (`http://keycloak:8080/...`).

## 7. TLS certificates (cert-manager + Let's Encrypt)

### 7.1 Nobody creates certificates by hand

The Ingress only carries an annotation:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - secretName: credpay-tls
      hosts: [prod.<domain>, admin.prod.<domain>, auth.prod.<domain>]
```

cert-manager sees it and does the rest:

```
 1. Ingress with the annotation        → cert-manager creates a Certificate "credpay-tls"
 2. Certificate                        → an Order at Let's Encrypt (ACME protocol)
 3. Let's Encrypt: "prove you own prod.20-104-252-247.sslip.io"
                                       → a Challenge (HTTP-01)
 4. cert-manager starts a tiny solver pod and a temporary Ingress for
    http://<host>/.well-known/acme-challenge/<token>
 5. Let's Encrypt fetches that URL over the internet → it reaches Traefik → the solver
    → ownership proven
 6. Let's Encrypt signs the certificate → cert-manager stores it in Secret credpay-tls
 7. Traefik picks the Secret up and serves HTTPS with it - no restart needed
 8. Renewal: certificates are valid 90 days; cert-manager renews automatically
    30 days before they expire
```

One certificate covers the three hosts of an environment, so there are four certificates
in the cluster: `credpay-tls` in each app namespace and `grafana-tls` in `monitoring`.

### 7.2 The three issuers ([`cluster-issuers.yaml`](../k8s/platform/cluster-issuers.yaml))

| Issuer | Certificate | When to use |
| --- | --- | --- |
| `letsencrypt-prod` (default) | trusted by every browser | normal use |
| `letsencrypt-staging` | same flow, **not** trusted (browser warning), very high limits | testing the flow without using up the real limits |
| `selfsigned` | made inside the cluster, no internet check | no public access or Let's Encrypt is blocked |

Switch with the variable `tlsIssuer` in [`pipelines/credpay-cd.yml`](../pipelines/credpay-cd.yml);
the pipeline passes it to the chart and to Grafana.

The HTTP→HTTPS redirect does not break step 5: Let's Encrypt follows the redirect to
HTTPS and does not check the certificate while it validates, and Traefik serves the
solver's temporary route on HTTPS as well.

## 8. The domain: sslip.io

### 8.1 What it is

[sslip.io](https://sslip.io) is a free, public DNS service that answers **any name that
contains an IP address with that IP address**:

```
20-104-252-247.sslip.io               → 20.104.252.247
prod.20-104-252-247.sslip.io          → 20.104.252.247
admin.prod.20-104-252-247.sslip.io    → 20.104.252.247
anything.20-104-252-247.sslip.io      → 20.104.252.247
```

Try it: `nslookup admin.prod.20-104-252-247.sslip.io`.

### 8.2 Why we use it

- **No domain to buy and no DNS records to manage** - real host names from day one.
- Every host points at the same IP (Traefik); Traefik tells them apart by host name.
- Let's Encrypt can issue **real, trusted certificates** for these names, because they
  resolve on the public internet.

### 8.3 How the pipeline builds it

```bash
domain="$(echo "$ingressIp" | tr . -).sslip.io"     # 20.104.252.247 → 20-104-252-247.sslip.io
```

Dots are replaced with dashes so the IP stays one DNS label. The chart then builds
`<env>.<domain>`, `admin.<env>.<domain>` and `auth.<env>.<domain>`
(`charts/credpay/templates/_helpers.tpl`); Grafana is `grafana.<domain>`.

### 8.4 Limits, and moving to your own domain

| Limit | What it means |
| --- | --- |
| The name contains the IP | after a destroy and rebuild the IP - and every URL - changes |
| sslip.io is shared by everyone | Let's Encrypt limits certificates per registered domain per week; when that limit is hit, the pipeline warns, the smoke tests skip the certificate check, and you can use `letsencrypt-staging` or `selfsigned` until it resets |
| Third-party service | if sslip.io is down, the names do not resolve |

To use a real domain: set `baseDomain` in `pipelines/credpay-cd.yml` (for example
`credpay.example.com`) and create DNS **A records** pointing at the ingress IP - a wildcard
`*.credpay.example.com` covers every host. Nothing else changes: the chart, Traefik and
cert-manager work the same way.

## 9. Configuration and secrets

| What | Where |
| --- | --- |
| Hosts, public URLs, database names, token audience | chart → ConfigMap `credpay-config` (from `environment` + `domain`) |
| Replicas, autoscaling, monitoring, AIOps | `charts/credpay/values-<env>.yaml` |
| Image tags | set by the pipeline: `--set-string <component>.image=<registry>/credpay/<service>:<tag>` |
| DB host/user/password, Keycloak admin | Secrets `credpay-db`, `keycloak-admin` - created by the pipeline from Key Vault |
| Azure OpenAI / Dynatrace for AIOps | Secret `credpay-aiops` (prod, optional) - from Key Vault |
| Grafana admin, Slack webhook | Secrets `grafana-admin`, `alertmanager-slack` in `monitoring` - from Key Vault |
| Dynatrace tokens | Secret `dynakube` in `dynatrace` - from Key Vault |

A change to `credpay-config` or `credpay-aiops` restarts the pods that use it (a checksum
of the settings is part of the pod template). Pods only read Secrets when they start, so a
changed Key Vault value reaches the cluster only through a **pipeline run**, not through a
pod restart.

## 10. Security settings

The app namespaces enforce the **restricted** Pod Security Standard. Every container runs
as non-root with a read-only root filesystem (a writable `emptyDir` only where the app
needs one, e.g. `/tmp`), no privilege escalation, all Linux capabilities dropped, the
default seccomp profile, and CPU/memory requests and limits. Only ai-service mounts a
service-account token: read-only access to pods, deployments and events of its own
namespace.

## 11. Useful commands

```bash
# Connect (Entra ID login)
az aks get-credentials -g rg-credpay -n aks-credpay --overwrite-existing
kubelogin convert-kubeconfig -l azurecli

# What is installed
helm list -A                                        # every Helm release in the cluster
helm history credpay -n credpay-prod                # releases of one environment
helm get values credpay -n credpay-prod             # what the pipeline set
helm rollback credpay <revision> -n credpay-prod    # go back to a previous release
kubectl get pods,svc,ingress,hpa -n credpay-prod

# Routing
kubectl get ingress -A                              # every host and the public IP
kubectl get svc traefik -n traefik                  # EXTERNAL-IP = the static IP
kubectl logs -n traefik deploy/traefik --tail=50    # access log

# Certificates
kubectl get certificate -A                          # READY should be True
kubectl describe certificate credpay-tls -n credpay-prod
kubectl get certificaterequests,orders,challenges -A   # a stuck certificate shows up here
kubectl get clusterissuer

# Render the chart locally (no cluster needed)
helm template credpay charts/credpay -f charts/credpay/values-prod.yaml \
  --set domain=example.com \
  --set keycloak.image=r/keycloak:1 --set userService.image=r/user-service:1 \
  --set paymentService.image=r/payment-service:1 --set customerPortal.image=r/customer-portal:1 \
  --set adminPortal.image=r/admin-portal:1 --set aiops.image=r/ai-service:1
```
