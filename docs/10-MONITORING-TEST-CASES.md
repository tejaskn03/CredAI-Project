# CredPay monitoring - test cases and demo guide

A hands-on guide for class: what to show and explain on each dashboard, how to **break
something on purpose** and follow it through every tool, how to ask **Ask CredAI** good
questions, and how to send a **Slack** notification.

How the monitoring is built is explained in
[08-MONITORING-SETUP.md](08-MONITORING-SETUP.md); what to do when
a real alert fires is in the runbooks in [09-OBSERVABILITY-ALERTS-RUNBOOKS.md](09-OBSERVABILITY-ALERTS-RUNBOOKS.md#alerts-and-runbooks).

- [Before you start](#before-you-start)
- [Part A - Dashboard tour: what to show and explain](#part-a---dashboard-tour-what-to-show-and-explain)
- [Part B - Test cases: break something and watch it](#part-b---test-cases-break-something-and-watch-it)
- [Part C - Ask CredAI: how to prompt](#part-c---ask-credai-how-to-prompt)
- [Part D - Sending a notification to Slack](#part-d---sending-a-notification-to-slack)
- [Part E - After the class: clean up](#part-e---after-the-class-clean-up)

---

## Before you start

**1. Your URLs.** Replace `<domain>` with your ingress IP with dashes, e.g.
`20-104-252-247.sslip.io` (`kubectl get ingress -A` lists them).

| | URL | Login |
| --- | --- | --- |
| Customer portal (prod) | `https://prod.<domain>` | `customer@credpay.local` / `Customer@123` |
| Admin portal (prod) | `https://admin.prod.<domain>` | `admin@credpay.local` / `Admin@123` |
| Grafana | `https://grafana.<domain>` | `admin` / Key Vault `grafana-admin-password` |
| Keycloak console | `https://auth.prod.<domain>/admin` | Key Vault `keycloak-admin-username` / `keycloak-admin-password` |
| Dynatrace | your `https://<id>.apps.dynatrace.com` | your Dynatrace account |
| Slack | your `#credpay-alerts` channel | |

**2. kubectl connected** (Git Bash):

```bash
az aks get-credentials -g rg-credpay -n aks-credpay --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
kubectl get pods -n credpay-prod          # all Running
```

**3. Some traffic.** Graphs are empty without users. Log in to the customer portal, open a
few pages and make 2-3 payments. Then wait 30 seconds and do it again: Prometheus only
counts a brand-new counter once it changes a second time.

**4. Open everything side by side** in browser tabs: Grafana dashboard, admin portal →
Monitoring, admin portal → Ask CredAI, Dynatrace, Slack.

---

## Part A - Dashboard tour: what to show and explain

### A1. Grafana - *CREDPAY - Service Overview*

Grafana → **Dashboards** → folder **CREDPAY** → **CREDPAY - Service Overview**. At the top,
pick **namespace = credpay-prod**.

| Panel | What it shows | What to explain |
| --- | --- | --- |
| **Requests per second** | traffic per service | the "load" signal; health checks are left out - this is real user traffic |
| **Error rate (5xx share)** | share of requests that failed with a server error | the "errors" signal; alert at 5% for 5 minutes |
| **Response time (95th percentile)** | 95 out of 100 requests are faster than this | the "latency" signal; why p95 and not the average (the average hides the slow users); alert above 1 s for 10 min |
| **Payments (last hour)** | successful payments | a **business** metric - monitoring is not only about CPU |
| **Failed payments (last hour)** | payments that failed with 5xx | alert at more than 3 in 10 minutes |
| **Firing alerts** | how many alerts are firing now | should be 0 |
| **Services up** | 1 = Prometheus can scrape the service, 0 = it cannot | the most basic health check |
| **CPU per pod** | CPU cores per pod | compare with the requests in `values.yaml`; HPA reacts to this |
| **Memory per pod** | memory per pod | alert above 90% of the limit (an OOM kill is coming) |
| **Container restarts (last 15 min)** | restarts per pod | more than 3 in 15 min = crash loop alert |

**Talking point - the four golden signals:** traffic, errors, latency, saturation
(CPU/memory). The first row of the dashboard is exactly those, per service.

**More to show in Grafana:**

* **Dashboards → Kubernetes / Compute Resources / Namespace (Pods)** (from the stack) - the
  same CPU/memory for **any** namespace, including dev and qa.
* **Dashboards → Node Exporter / Nodes** - the VMs under the cluster.
* **Explore** (compass icon) → data source Prometheus → try these queries:

  ```promql
  up{namespace="credpay-prod"}                                   # 1 per scraped pod
  credpay:http_error_ratio:rate5m                                # error share per service
  credpay:http_latency_p95:rate5m                                # p95 per service (seconds)
  kube_deployment_status_replicas_available{namespace="credpay-prod"}
  sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="credpay-prod", container!=""}[5m]))
  ALERTS{alertstate="firing"}                                    # what is firing right now
  ```
* **Alerting → Alert rules** - the 7 CredPay rules (group `credpay.*`) and their state:
  *Normal*, *Pending* (condition true, waiting for the "for" time) or *Firing*.

### A2. Admin portal - *Monitoring* page

`https://admin.prod.<domain>` → **Monitoring** (prod only; dev and qa show a note).

| Part of the page | What to explain |
| --- | --- |
| **Status chip** (Healthy / Degraded / Down) | one overall verdict from all sources |
| **Updated … refreshes every 30 seconds** | live, no reload needed |
| **Grafana / Dynatrace buttons** | jump to the detailed tools |
| **AIOps briefing → Generate briefing** | a short summary: main problem, likely cause, impact, next steps. *Generated by Azure OpenAI* (or *rules* without it) |
| **Services** table | per service: state, pods, requests/s, errors, p95 - the same numbers as Grafana |
| **Firing alerts** | what Alertmanager has right now |
| **Dynatrace problems** | open problems from Davis AI, with links |
| **Data source chips** at the bottom | which sources answered (prometheus, alertmanager, kubernetes, dynatrace, azure_openai: `ok` / `not configured` / `unavailable`) |

**Talking point:** this page is built by ai-service from the **same** sources as Grafana,
Alertmanager and Dynatrace - one screen instead of four.

### A3. Dynatrace

Open your Dynatrace environment. The menus change between Dynatrace versions - if you
cannot find one, press **Ctrl+K** and search for its name.

| Where | What to show | What to explain |
| --- | --- | --- |
| **Kubernetes** → cluster `credpay` | nodes, namespaces, workloads, their health and events | found automatically by the ActiveGate - no configuration per service |
| **Kubernetes** → namespace `credpay-prod` → a workload, e.g. `payment-service` | pods, CPU/memory vs requests and limits, events | the same data as Grafana, without writing any query |
| **Services** | `user-service` (Java), `payment-service` (Python), Keycloak, with response time, failure rate and throughput | **code-level** monitoring: the OneAgent was injected into the prod pods - the apps were not changed |
| A service → **Distributed traces** (or *PurePaths*) | one request end to end: portal → user-service → database, time per step, SQL statements | the difference to Prometheus: not only *that* it is slow but *where* |
| **Service flow** / **Smartscape** | who calls whom | the architecture is discovered from real traffic |
| **Problems** | problems Davis AI opened, the root cause and the affected services | AI that correlates many events into one problem; the same problems appear on the admin portal |
| **Hosts** | the AKS nodes (VMs) | the OneAgent pod on every node monitors the VM itself |

Data appears 5-10 minutes after a deployment. Only `credpay-prod` has code-level data;
dev and qa are visible in *Kubernetes* only.

### A4. Prometheus and Alertmanager (optional, for the curious)

```bash
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090      # http://localhost:9090
kubectl port-forward -n monitoring svc/monitoring-alertmanager 9093:9093    # http://localhost:9093
```

Prometheus → **Status → Targets**: every credpay-prod service is *UP*. **Alerts**: the
rules. Alertmanager: the alerts it received and the **silences**.

---

## Part B - Test cases: break something and watch it

Run them in **prod** - only prod has alerts, Slack messages, Dynatrace code modules and
ai-service. Every test has an **undo** step. The next pipeline run also puts everything
back the way the chart describes it.

| ID | Test | Alert / Slack | Time to see it |
| --- | --- | --- | --- |
| [TC-01](#tc-01-a-test-alert-to-slack) | A test alert to Slack | FIRING + RESOLVED message | 1 min |
| [TC-02](#tc-02-the-slack-webhook-on-its-own) | The Slack webhook on its own | a plain message | seconds |
| [TC-03](#tc-03-self-healing-delete-a-pod) | Self-healing: delete a pod | none - and that is the point | 1 min |
| [TC-04](#tc-04-a-crash-loop-out-of-memory) | A crash loop (out of memory) | `CredpayPodCrashLooping` (critical) | 3-6 min |
| [TC-05](#tc-05-a-service-goes-down-network-blocked) | A service goes down (network blocked) | `CredpayServiceDown` (critical) | 3-4 min |
| [TC-06](#tc-06-load-test-and-autoscaling) | Load test and autoscaling | none (HPA handles it) | 2-5 min |
| [TC-07](#tc-07-a-bad-release-image-does-not-exist) | A bad release (image does not exist) | none - users are not affected | 1 min |
| [TC-08](#tc-08-silence-an-alert-during-maintenance) | Silence an alert during maintenance | no Slack message | 2 min |
| [TC-09](#tc-09-brute-force-login-and-account-lock-out) | Brute-force login and account lock-out | none (security control) | 1 min |
| [TC-10](#tc-10-business-metrics-payments) | Business metrics: payments | none | 1 min |

### TC-01. A test alert to Slack

**Goal:** prove the whole alert path Alertmanager → Slack works, without breaking anything.

```bash
kubectl port-forward -n monitoring svc/monitoring-alertmanager 9093:9093 &
sleep 3
curl -s -H "Content-Type: application/json" http://localhost:9093/api/v2/alerts -d '[{
  "labels": {"alertname":"CredpayHighLatency","severity":"warning","team":"credpay",
             "environment":"prod","namespace":"credpay-prod","service":"user-service"},
  "annotations": {"summary":"user-service is slow in prod (CLASS DEMO)",
                  "description":"This is a test alert sent by hand."}}]'
kill %1
```

**Expected:**

| Where | What you see |
| --- | --- |
| Slack | after ~30 s: *🚨 FIRING: CredpayHighLatency (credpay-prod)*, orange; after ~5 min: *✅ RESOLVED* |
| Admin portal → Monitoring | status *Degraded*, the alert under *Firing alerts* |
| Alertmanager UI (`localhost:9093`) | the alert, while it lasts |

**Explain:** `team=credpay` routes it to Slack; `severity` sets the colour (try
`"critical"` for red); a hand-made alert resolves itself after 5 minutes because nobody
keeps sending it.

**Ask CredAI:** *Are any alerts firing? What should I do?*

### TC-02. The Slack webhook on its own

**Goal:** check the Slack side only (useful when no message arrives in TC-01).

```bash
url=$(az keyvault secret show --vault-name kv-credpay-<suffix> --name slack-webhook-url --query value -o tsv)
curl -s -X POST -H "Content-Type: application/json" -d '{"text":":wave: Hello from the CredPay class"}' "$url"; echo
```

**Expected:** `ok` in the terminal and the message in the channel. `invalid_token` or
`no_service` = the webhook was deleted in Slack - create a new one and update Key Vault.

### TC-03. Self-healing: delete a pod

**Goal:** Kubernetes replaces failed pods by itself; two replicas mean no outage.

```bash
pod=$(kubectl get pod -n credpay-prod -l app.kubernetes.io/name=customer-portal -o name | head -1)
kubectl delete -n credpay-prod $pod
kubectl get pods -n credpay-prod -l app.kubernetes.io/name=customer-portal -w     # Ctrl+C to stop
```

**Expected:** a new pod is created within seconds and becomes *Running*. The customer
portal keeps working the whole time (the second replica serves). **No alert** - the
deployment is back to 2/2 long before `CredpayDeploymentUnavailable`'s 10 minutes.

**Explain:** the Deployment wants 2 replicas, so its ReplicaSet creates a new pod at once.
The restart counter does **not** go up - this is a new pod, not a restarted container.

**Undo:** nothing to undo.

### TC-04. A crash loop (out of memory)

**Goal:** a container that keeps crashing, seen in every tool.

```bash
kubectl set resources deploy/payment-service -n credpay-prod --requests=memory=20Mi --limits=memory=20Mi
kubectl get pods -n credpay-prod -l app.kubernetes.io/name=payment-service -w      # Ctrl+C to stop
```

**What happens:** this starts a rolling update. The **new** pod gets only 20 MiB, which is
too little for Python, so it is killed (*OOMKilled*) and restarted again and again
(*CrashLoopBackOff*). The two **old** pods keep running, so payments still work - the
rolling update never removes an old pod before a new one is ready.

| Where | What you see |
| --- | --- |
| `kubectl describe pod <new pod> -n credpay-prod` | *Last State: Terminated, Reason: OOMKilled*, restart count going up |
| Grafana → **Container restarts (last 15 min)** | the new pod climbing |
| Slack, after 3-6 min | *🚨 FIRING: CredpayPodCrashLooping (credpay-prod)*, red |
| Admin portal → Monitoring | *Degraded*, the alert, the pod under recent restarts; **Generate briefing** names payment-service and the restarts |
| Dynatrace → Kubernetes → `payment-service` → events | the out-of-memory kills; Davis may open a problem |

**Ask CredAI:** *Why is payment-service restarting?* · *Which pods use the most memory?* ·
then a follow-up: *What should I change to fix it?*

**Undo:**

```bash
kubectl rollout undo deploy/payment-service -n credpay-prod
kubectl rollout status deploy/payment-service -n credpay-prod
```

The alert resolves within ~15 minutes (the restarts must drop out of the 15-minute window)
and Slack gets a *RESOLVED* message.

### TC-05. A service goes down (network blocked)

**Goal:** a real outage of one service - customers see errors and the critical alert fires.

```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-block-payment
  namespace: credpay-prod
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: payment-service
  policyTypes: [Ingress]
  ingress: []          # allow nothing in
EOF
```

**What happens:** no traffic can reach payment-service any more - not from Traefik (the
customers), not from Prometheus (the monitoring).

| Where | What you see |
| --- | --- |
| Customer portal | pages that use payments fail (gateway errors 502/503/504) |
| Grafana → **Services up** | payment-service drops to 0 |
| Slack, after 3-4 min | *🚨 FIRING: CredpayServiceDown (credpay-prod)* - "payment-service is down in prod" |
| Admin portal → Monitoring | status *Down/Degraded*, payment-service marked down |
| Dynatrace | failing requests from Traefik; Davis may open a problem |

If Kubernetes' own health checks are blocked as well, the pods also become *not ready* and
may restart - you may then see `CredpayPodCrashLooping` too. That is realistic: one fault,
several symptoms. Point out how Alertmanager **groups** them and how the critical alert
mutes the warnings of the same service.

**Ask CredAI:** *Is prod healthy right now?* · *Why can't customers pay?* ·
*Is payment-service up?*

**Undo:**

```bash
kubectl delete networkpolicy demo-block-payment -n credpay-prod
```

The service is back at once; *RESOLVED* in Slack within ~5 minutes.

### TC-06. Load test and autoscaling

**Goal:** the HorizontalPodAutoscaler adds pods under load and removes them afterwards.

```bash
kubectl get hpa -n credpay-prod        # payment-service: 2 replicas, target 70% CPU

# a load generator in the default namespace (not in prod)
kubectl run loadgen -n default --image=busybox:1.36 --restart=Never -- sh -c \
  'for i in 1 2 3 4 5 6 7 8; do (while true; do wget -q -O /dev/null http://payment-service.credpay-prod.svc.cluster.local:8000/api/payment/health; done) & done; wait'

kubectl get hpa -n credpay-prod -w     # Ctrl+C to stop
```

**Expected:** within 1-3 minutes payment-service's CPU goes above 70% of its request and
the HPA scales it from **2 to up to 4** replicas.

| Where | What you see |
| --- | --- |
| `kubectl get hpa` | TARGETS above 70%, REPLICAS 3 → 4 |
| Grafana | **Requests per second** jumps; **CPU per pod** rises, then spreads over more pods |
| Dynatrace → `payment-service` | throughput up |

**Ask CredAI:** *Is the HPA scaling anything right now?* · *Which pods use the most CPU?*

**Undo:**

```bash
kubectl delete pod loadgen -n default
```

The HPA waits 5 minutes (to avoid flapping), then goes back to 2 replicas.

### TC-07. A bad release (image does not exist)

**Goal:** a broken release does not take the site down.

```bash
kubectl set image deploy/user-service -n credpay-prod user-service=creddemoacr.azurecr.io/credpay/user-service:does-not-exist
kubectl get pods -n credpay-prod -l app.kubernetes.io/name=user-service
```

**Expected:** one new pod in *ErrImagePull* / *ImagePullBackOff*; the two old pods keep
running. The site works. **No alert fires** - desired and available replicas are still 2.

**Explain:** a rolling update only removes an old pod once a new one is ready. In the
pipeline, `helm upgrade --atomic` goes one step further: if the new pods are not ready in
15 minutes it rolls the whole release back automatically.

**Ask CredAI:** *Are any pods unhealthy or failing to start?* · *What happened in the
latest rollout?*

**Undo:**

```bash
kubectl rollout undo deploy/user-service -n credpay-prod
```

### TC-08. Silence an alert during maintenance

**Goal:** planned work should not page anyone.

1. Open Alertmanager: `kubectl port-forward -n monitoring svc/monitoring-alertmanager 9093:9093`
   → `http://localhost:9093` → **New Silence**.
2. Matcher `alertname = CredpayHighLatency`, duration `1h`, creator your name, comment
   *maintenance demo* → **Create**.
3. Send the test alert of TC-01 again.

**Expected:** the alert is visible in Alertmanager (marked *suppressed*) but **no Slack
message** arrives. **Undo:** Alertmanager → **Silences** → **Expire**.

### TC-09. Brute-force login and account lock-out

**Goal:** a security control that is part of the platform.

1. Register a test account in the customer portal (so the demo customer is not locked).
2. Log out and type a **wrong password 6 times** for that account.

**Expected:** after 5 failures Keycloak locks the account temporarily - the correct
password is refused too. The lock starts at 1 minute and grows with more failures (up to
15 minutes).

| Where | What you see |
| --- | --- |
| Login page | the login is refused even with the right password |
| `kubectl logs -n credpay-prod deploy/keycloak \| grep LOGIN_ERROR` | one line per failed login (`invalid_user_credentials`, then `user_temporarily_disabled`) |
| Keycloak console → realm **credpay** → **Users** → the user | the account shown as temporarily locked, with an unlock option |

**Ask CredAI:** *How many login attempts failed recently?*

**Undo:** wait, or unlock the user in the Keycloak console.

### TC-10. Business metrics: payments

**Goal:** monitoring also answers business questions.

Make 3-4 payments in the customer portal, wait 30 seconds, make 2 more.

| Where | What you see |
| --- | --- |
| Grafana → **Payments (last hour)** | the count going up |
| Admin portal → **Overview** / **Transactions** | the payments |

**Ask CredAI:** *How many successful payments were made recently?* · *Are any payments
failing?*

---

## Part C - Ask CredAI: how to prompt

### How it works

Admin portal (prod) → **Ask CredAI**. For every question, ai-service collects **live facts**
(the same overview as the Monitoring page, plus extra facts picked by the words in your
question), sends them with your question to Azure OpenAI, and the model answers **only from
those facts**. The line **Based on:** under each answer lists the sources it used.

It remembers the conversation in the browser tab (the last 10 messages), so follow-up
questions work. **New chat** starts over.

### Which words fetch which facts

| Topic | Words that trigger it | Example prompts |
| --- | --- | --- |
| Overall health, firing alerts, Dynatrace problems | *(part of every answer)* | *Is prod healthy right now?* · *Are any alerts firing? What should I do?* · *Are there any problems in Dynatrace?* |
| Root cause | why, root cause, slow | *Why could payments be slow?* · *Why is payment-service restarting?* |
| Capacity | cpu, memory, capacity, bottleneck, hpa, autoscaling, replica | *Which pods use the most CPU and memory?* · *Do we have enough capacity?* · *Is the HPA scaling?* |
| Cluster | cluster, pod, node, unhealthy, restart, scheduling | *Are any pods unhealthy?* · *Which pods restarted recently?* |
| Deployments | deployment, rollout, latest release | *What happened in the latest rollout?* |
| Business | payment, transaction, business metric, login attempt, successful payments | *How many successful payments were made recently?* · *How many login attempts failed?* |
| Daily report | daily/today + summary/report | *Give me today's summary.* · *Daily report please.* |

The suggestion chips under the chat box are good starting points.

### Tips for good questions

| Do | Avoid |
| --- | --- |
| Ask about **now**: *Is payment-service healthy right now?* | Questions about last month - it sees live data and Prometheus' 7 days only |
| Name the **service**: *Why is user-service slow?* | *Why is it broken?* without context in a new chat |
| Use the **words** above to pull in the right facts (*cpu*, *restart*, *payment*) | Asking it to **do** something (*restart the pod*) - it is read-only and only advises |
| Follow up: *What should I check next?* · *What does that alert mean?* | Questions about customers' personal data - it never receives any |

### A good demo conversation (during TC-04)

1. *Is prod healthy right now?* → it reports *Degraded* and the crash-looping pod.
2. *Why is that pod restarting?* → out of memory: usage vs the 20 MiB limit.
3. *What should I change to fix it?* → raise the memory limit / roll back.
4. Undo TC-04, wait a few minutes, then: *Is it fixed now?*

---

## Part D - Sending a notification to Slack

| Way | When to use it | How |
| --- | --- | --- |
| **A test alert** through Alertmanager | show the full alert path; check routing and the message format | [TC-01](#tc-01-a-test-alert-to-slack) |
| **The webhook directly** | check that the Slack webhook itself works | [TC-02](#tc-02-the-slack-webhook-on-its-own) |
| **A real alert** | the realistic demo | break something: [TC-04](#tc-04-a-crash-loop-out-of-memory) or [TC-05](#tc-05-a-service-goes-down-network-blocked) |

**Which alerts go to Slack:** every alert with `team = credpay` - the 7 CredPay rules (prod
only) - when they fire **and** when they resolve. The stack's own Kubernetes alerts and
the *Watchdog* heartbeat are not sent. A message that keeps firing is repeated every 4 hours.

**No message arrived?** Check in this order:

1. The pipeline log, *Read optional integrations*: `SLACK_ENABLED true`? If not, the
   `slack-webhook-url` secret is missing - add it and run the pipeline.
2. TC-02: does the webhook itself work?
3. Did Alertmanager try to send it?

   ```bash
   kubectl exec -n monitoring alertmanager-monitoring-alertmanager-0 -c alertmanager -- \
     wget -qO- http://localhost:9093/metrics | grep 'alertmanager_notifications.*slack'
   kubectl logs -n monitoring alertmanager-monitoring-alertmanager-0 -c alertmanager --tail=30
   ```

   `notifications_total` going up and `notifications_failed_total` 0 = sent.
4. Is the alert silenced (TC-08) or muted by a critical alert of the same service?

---

## Part E - After the class: clean up

```bash
kubectl delete networkpolicy demo-block-payment -n credpay-prod --ignore-not-found   # TC-05
kubectl delete pod loadgen -n default --ignore-not-found                             # TC-06
kubectl get pods -n credpay-prod                          # all Running, nothing restarting
```

If a pod is still crash-looping or stuck in *ImagePullBackOff*, run the **undo** step of
that test (TC-04 or TC-07) **once** - `rollout undo` run a second time switches back to the
broken version. Expire any silences in Alertmanager.

Not sure what is left? Run `credpay-cd` again: Helm puts every deployment back the way the
chart describes it.

When you have finished for the day, **destroy the environment** to stop the costs
([STUDENT-AZURE-SETUP.md, Part 11](03-STUDENT-AZURE-SETUP.md#part-11---save-money-destroy-and-rebuild)).
