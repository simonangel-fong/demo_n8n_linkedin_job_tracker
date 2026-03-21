```sh

```

- Node: HTTP Request node: list pods in demo namespace\*\*

Method: GET
URL: https://kubernetes.default.svc/api/v1/namespaces/demo/pods
Headers: Authorization = Bearer {{ $json.token }}
Content-Type = application/json
SSL: disable certificate verification (self-signed cert in minikube)

---

## Break

```sh
argocd app get argocd/demo-web
# Name:               argocd/demo-web
# Project:            homelab
# Server:             https://kubernetes.default.svc
# Namespace:          n8n
# URL:                https://argocd.example.com/applications/demo-web
# Source:
# - Repo:             https://github.com/simonangel-fong/Project-KubeTriage.git
#   Target:           main
#   Path:             demo-app/helm
#   Helm Values:      values-broken.yaml
# SyncWindow:         Sync Allowed
# Sync Policy:        Automated (Prune)
# Sync Status:        Synced to main (5cc4b63)
# Health Status:      Degraded

# GROUP  KIND        NAMESPACE  NAME                   STATUS  HEALTH    HOOK  MESSAGE
#        ConfigMap   n8n        demo-app-nginx-config  Synced                  configmap/demo-app-nginx-config serverside-applied
#        Service     n8n        demo-app               Synced  Healthy         service/demo-app serverside-applied
# apps   Deployment  n8n        demo-app               Synced  Degraded        deployment.apps/demo-app serverside-applied

# Check if the notification was sent
kubectl logs -n argocd deployment/argocd-notifications-controller --tail=20 | grep demo-app | grep error
# {"level":"info","msg":"Start processing","resource":"argocd/demo-web","time":"2026-03-20T20:24:06Z"}
# {"level":"error","msg":"Failed to evaluate condition of trigger on-health-degraded: trigger 'on-health-degraded' is not configured using the configuration in namespace argocd","resource":"argocd/demo-web","time":"2026-03-20T20:24:06Z"}
# {"level":"info","msg":"Processing completed","resource":"argocd/demo-web","time":"2026-03-20T20:24:06Z"}
# {"level":"info","msg":"Start processing","resource":"argocd/demo-web","time":"2026-03-20T20:24:22Z"}
# {"level":"error","msg":"Failed to evaluate condition of trigger on-health-degraded: trigger 'on-health-degraded' is not configured using the configuration in namespace argocd","resource":"argocd/demo-web","time":"2026-03-20T20:24:22Z"}
# {"level":"info","msg":"Processing completed","resource":"argocd/demo-web","time":"2026-03-20T20:24:22Z"}
# {"level":"info","msg":"Start processing","resource":"argocd/demo-web","time":"2026-03-20T20:25:06Z"}
# {"level":"error","msg":"Failed to evaluate condition of trigger on-health-degraded: trigger 'on-health-degraded' is not configured using the configuration in namespace argocd","resource":"argocd/demo-web","time":"2026-03-20T20:25:06Z"}
# {"level":"info","msg":"Processing completed","resource":"argocd/demo-web","time":"2026-03-20T20:25:06Z"}


```

## set argocd notification

```sh
# confirm notification:

kubectl get cm argocd-notifications-cm -n argocd -o yaml
# apiVersion: v1
# data:
#   context: |
#     argocdUrl: https://argocd.example.com
#   service.webhook.n8n-incident: |
#     url: http://n8n.n8n.svc.cluster.local/webhook/a7560150-a895-4a10-8e22-a27a74fa2b01




```

---

##

```sh
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  --rm -it \
  -n argocd \
  -- curl -X POST \
  http://n8n.n8n.svc.cluster.local/webhook/kube-triage \
  -H "Content-Type: application/json" \
  -d '{
    "app": "demo-app",
    "namespace": "n8n",
    "health": "Degraded",
    "sync": "Synced",
    "revision": "abc123",
    "message": "Back-off restarting failed container nginx in pod demo-app"
  }'

kubectl logs -n argocd -l app.kubernetes.io/name=argocd-notifications-controller --since-time="2026-03-21T10:00:00Z" -f | grep demo-app

# orce a re-evaluation
kubectl annotate app demo-app -n argocd \
  notifications.argoproj.io/subscribe.on-health-degraded.n8n-incident="" \
  --overwrite
```

---

node http request

Method: GET
URL: {{ $env.K8S_API }}/api/v1/namespaces/{{ $('Webhook').item.json.body.namespace }}/pods
Headers:
Authorization: Bearer {{ $env.K8S_TOKEN }}
Query Parameters:
labelSelector: app={{ $('Webhook').item.json.body.app }}

node Code: extract failed pod name

```js
const items = $input.first().json.items;

// find the pod that is not running
const failedPod =
  items.find(
    (pod) =>
      pod.status.phase !== "Running" ||
      pod.status.containerStatuses?.some(
        (c) => c.state?.waiting || c.restartCount > 0,
      ),
  ) || items[0]; // fallback to first pod if all look same

const podName = failedPod.metadata.name;
const namespace = failedPod.metadata.namespace;

return { podName, namespace };
```

**Node 4 — HTTP Request: get pod events**

Events contain the CrashLoopBackOff reason and the nginx config error message — the most useful input for the AI.
Method: GET
URL: {{ $env.K8S_API }}/api/v1/namespaces/{{ $('Webhook').first().json.body.namespace }}/events
Headers:
Authorization: Bearer {{ $env.K8S_TOKEN }}
Query Parameters:
fieldSelector: involvedObject.name={{ $('Get_podname').item.json.podName}}

**Node 5 — HTTP Request: get pod logs**
Method: GET

URL: {{ $env.K8S_API }}/api/v1/namespaces/{{ $('Get_podname').item.json.namespace }}/pods/{{ $('Get_podname').item.json.podName}}/log
Headers:
Authorization: Bearer {{ $env.K8S_TOKEN }}
Query Parameters:
tailLines: 50
previous: true

Before hitting the AI node, clean and combine the data into a structured prompt input:

```js
const podName = $("Get_podname").first().json.podName;
const namespace = $("Get_podname").first().json.namespace;
const app = $("Webhook").first().json.body.app;

// extract event messages
const events =
  $("Get_podevent")
    .first()
    .json.items?.map((e) => `[${e.reason}] ${e.message}`)
    .join("\n") || "no events found";

// logs come back as plain text
const logs = $("Get_podlog").first().json || "no logs found";

const context = `
App: ${app}
Pod: ${podName}
Namespace: ${namespace}

=== EVENTS ===
${events}

=== LOGS ===
${logs}
`.trim();

return { app, podName, context };
```

**Node 7 — AI Agent (Gemini)**

In the AI Agent node:
Node Type: AI Agent
Node Name: AI Agent
Model: google gemini
Previous Node: gen_context
Configuration: 

- User message: {{ $json.context }}
- System Message:

```txt
You are a Kubernetes incident analyst reviewing a pod failure.

Analyze the provided events and logs and produce a structured incident report with exactly these four sections:

WHAT FAILED
One sentence naming the pod and failure mode.

ROOT CAUSE
The specific error message, file, and line number that caused the failure.
Focus on ERROR and EMERG level log entries only. Ignore INFO level messages.

CONFIG CONTEXT
Which Kubernetes resource likely caused the failure.
Choose one: ConfigMap, Secret, Deployment spec, resource limits, or image.

SUGGESTED ACTION
One sentence describing what to investigate.
Then list 2-3 kubectl commands using the exact pod name, namespace, and resource names from the incident data.
Verification commands only — do not suggest fixes.

---
EXAMPLE OUTPUT:

WHAT FAILED
Pod api-7d6f9c-xkp2q is in CrashLoopBackOff because the container exits immediately on startup.

ROOT CAUSE
[emerg] unknown directive "locationaa" in /etc/nginx/conf.d/default.conf:6

CONFIG CONTEXT
ConfigMap — nginx configuration mounted into the container is malformed.

SUGGESTED ACTION
Inspect the nginx ConfigMap and verify the configuration syntax.
kubectl logs api-7d6f9c-xkp2q -n production --previous
kubectl describe pod api-7d6f9c-xkp2q -n production
kubectl get configmap api-nginx-config -n production -o yaml
---

Keep the total response under 200 words.
Return plain text only — no markdown, no asterisks, no bullet points.
```

**Node 8 — Snd Email**

Subject: [INCIDENT] CrashLoopBackOff — demo-app / demo-app-6b9c97cc79-fqj9x (Namespace: n8n)

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>KubeTriage Incident Notification</title>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Arial, sans-serif;
      background-color: #f4f4f4;
      margin: 0;
      padding: 24px;
      color: #333333;
    }
    .container {
      max-width: 640px;
      margin: 0 auto;
      background-color: #ffffff;
      border-radius: 8px;
      overflow: hidden;
      border: 1px solid #e0e0e0;
    }
    .header {
      background-color: #b91c1c;
      padding: 24px 32px;
    }
    .header h1 {
      margin: 0;
      color: #ffffff;
      font-size: 18px;
      font-weight: 600;
      letter-spacing: 0.5px;
    }
    .header p {
      margin: 4px 0 0;
      color: #fecaca;
      font-size: 13px;
    }
    .body {
      padding: 28px 32px;
    }
    .intro {
      font-size: 14px;
      line-height: 1.6;
      color: #444444;
      margin: 0 0 24px;
    }
    .section-label {
      font-size: 11px;
      font-weight: 600;
      letter-spacing: 1px;
      text-transform: uppercase;
      color: #888888;
      margin: 0 0 10px;
    }
    .details-table {
      width: 100%;
      border-collapse: collapse;
      margin-bottom: 24px;
      background-color: #f9f9f9;
      border-radius: 6px;
      overflow: hidden;
      border: 1px solid #e8e8e8;
    }
    .details-table tr:not(:last-child) td {
      border-bottom: 1px solid #e8e8e8;
    }
    .details-table td {
      padding: 10px 16px;
      font-size: 13px;
    }
    .details-table td:first-child {
      color: #888888;
      width: 120px;
      font-weight: 500;
    }
    .details-table td:last-child {
      color: #222222;
      font-family: 'Courier New', Courier, monospace;
      font-size: 12px;
    }
    .analysis-box {
      background-color: #f9f9f9;
      border-left: 4px solid #b91c1c;
      border-radius: 0 6px 6px 0;
      padding: 16px 20px;
      margin-bottom: 24px;
      font-size: 13px;
      line-height: 1.7;
      color: #333333;
      white-space: pre-wrap;
    }
    .footer {
      border-top: 1px solid #e8e8e8;
      padding: 16px 32px;
      background-color: #fafafa;
      font-size: 12px;
      color: #999999;
      text-align: center;
    }
    .footer strong {
      color: #666666;
    }
  </style>
</head>
<body>
  <div class="container">

    <!-- Header -->
    <div class="header">
      <h1>⚠ Incident Detected</h1>
      <p>KubeTriage — Automated Incident Notification</p>
    </div>

    <!-- Body -->
    <div class="body">

      <p class="intro">
        Dear On-Call Engineer,<br><br>
        This is an automated incident notification generated by KubeTriage.
        A pod failure has been detected in your Kubernetes cluster.
        Please review the following analysis and take appropriate action.
      </p>

      <!-- Incident Details -->
      <p class="section-label">Incident Details</p>
      <table class="details-table">
        <tr>
          <td>Application</td>
          <td>{{ $('Webhook').item.json.body.app }}</td>
        </tr>
        <tr>
          <td>Pod</td>
          <td>{{ $('Get_podname').item.json.podName }}</td>
        </tr>
        <tr>
          <td>Namespace</td>
          <td>{{ $('Webhook').item.json.body.namespace }}</td>
        </tr>
        <tr>
          <td>Revision</td>
          <td>{{ $('Webhook').item.json.body.revision }}</td>
        </tr>
        <tr>
          <td>Detected at</td>
          <td>{{ $now.toISO() }}</td>
        </tr>
      </table>

      <!-- AI Analysis -->
      <p class="section-label">AI Analysis</p>
      <div class="analysis-box">{{ $('AI Agent').item.json.output }}</div>

    </div>

    <!-- Footer -->
    <div class="footer">
      This is an automated message. Do not reply to this email.<br>
      <strong>KubeTriage — Arguswatcher Homelab</strong>
    </div>

  </div>
</body>
</html>
