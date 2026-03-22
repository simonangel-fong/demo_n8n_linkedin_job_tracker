# KubeTriage - Set n8n workflow

[Back](../README.md)

- [KubeTriage - Set n8n workflow](#kubetriage---set-n8n-workflow)
  - [Enable Notification wiht webhook](#enable-notification-wiht-webhook)
    - [Get webhook url](#get-webhook-url)
    - [Update Argocd Notification Webhook](#update-argocd-notification-webhook)
    - [Test Webhook: Break demo-app](#test-webhook-break-demo-app)
  - [Create Workflow](#create-workflow)
    - [Node01 - Webhook: Webhook](#node01---webhook-webhook)
    - [Node02 - HTTP Request: Get\_postlist](#node02---http-request-get_postlist)
    - [Node03 - Code: Get\_podname](#node03---code-get_podname)
    - [Node04 — HTTP Request: Get\_podevent](#node04--http-request-get_podevent)
    - [Node05 — HTTP Request: Get\_podlog](#node05--http-request-get_podlog)
    - [Node06 — Code: gen\_context](#node06--code-gen_context)
    - [Node07 — AI Agent (Gemini)](#node07--ai-agent-gemini)
    - [Node08 — Snd Email](#node08--snd-email)

---

## Enable Notification wiht webhook

### Get webhook url

- Create Webhook Node
  - Urls shown in webhook node:
    - test url: https://homelab-n8n.arguswatcher.net/webhook-test/kube-triage
    - prod url: http://n8n.n8n.svc.cluster.local:5678/webhook/kube-triage
  - By testing, urls should be:
    - test url: https://homelab-n8n.arguswatcher.net/webhook-test/kube-triage
    - prod url: http://n8n.n8n.svc.cluster.local/webhook/kube-triage

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
# {"message":"Workflow was started"}pod "curl-test" deleted from argocd namespace
```

---

### Update Argocd Notification Webhook

```yaml
notifications:
  enabled: true
  cm:
    create: true
  notifiers:
    service.webhook.n8n-incident: |
      url: http://n8n.n8n.svc.cluster.local/webhook/kube-triage
      headers:
        - name: Content-Type
          value: application/json
```

- Confirm

```sh
# confirm notification:

kubectl get cm argocd-notifications-cm -n argocd -o yaml
# apiVersion: v1
# data:
#   context: |
#     argocdUrl: https://argocd.example.com
#   service.webhook.n8n-incident: |
#     url: http://n8n.n8n.svc.cluster.local/webhook/kube-triage
```

- Update Helm

```sh
helm upgrade --install argocd argo/argo-cd -n argocd --version 9.4.15 -f argocd/helm/values.yaml
```

---

### Test Webhook: Break demo-app

```yaml
# nginx configuration — edit this to simulate an incident
nginx:
  config: |
    server {
      listen 80;
      server_name localhost;

      location / {
        root   /usr/share/nginx/html;
        index  index.html;
      
      # comment closing brace — nginx config test will fail
      # }

      location /health {
        return 200 'ok';
        add_header Content-Type text/plain;
      
      }   
    }
```

- monitor and confirm

```sh
kubectl get pods -n argocd | grep notification
# argocd-notifications-controller-8547c686f6-628s6    1/1     Running   0          52m

# force a re-evaluation if needed
kubectl annotate app demo-app -n argocd \
  notifications.argoproj.io/subscribe.on-health-degraded.n8n-incident="" \
  --overwrite

# argocd controller
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-notifications-controller -f | grep -iE "sending|sent"
# {"level":"info","msg":"Sending notification about condition 'on-health-degraded.[0].I9Noxy9j65pHtwZnNz53bcmYtpE' to '{n8n-incident }' using the configuration in namespace argocd","resource":"argocd/demo-app","time":"2026-03-21T18:39:19-04:00"}
# {"level":"info","msg":"Notification about condition 'on-health-degraded.[0].I9Noxy9j65pHtwZnNz53bcmYtpE' already sent to '{n8n-incident }' using the configuration in namespace argocd","resource":"argocd/demo-app","time":"2026-03-21T18:39:19-04:00"}
# {"level":"info","msg":"Notification about condition 'on-health-degraded.[0].I9Noxy9j65pHtwZnNz53bcmYtpE' already sent to '{n8n-incident }' using the configuration in namespace argocd","resource":"argocd/demo-app","time":"2026-03-21T18:39:50-04:00"}

```

![pic](./docs/webhook_testing.png)

---

## Create Workflow

- Diagram

![pic](./docs/n8n_workflow.png)

### Node01 - Webhook: Webhook

- Node Type: Webhook
- Node Name: Webhook
- HTTP Method: POST
- Path: kube-triage

- Sample Output:

```json
[
  {
    "headers": {
      "host": "n8n.n8n.svc.cluster.local",
      "user-agent": "Go-http-client/1.1",
      "content-length": "382",
      "content-type": "application/json",
      "accept-encoding": "gzip"
    },
    "params": {},
    "query": {},
    "body": {
      "app": "demo-app",
      "namespace": "n8n",
      "health": "Degraded",
      "sync": "Synced",
      "revision": "e4aa4408b43282d454fa4c3fa93da20d250cdee8",
      "phase": "Succeeded",
      "message": "successfully synced (all tasks run)",
      "syncStarted": "2026-03-21T22:29:17Z",
      "syncFinished": "2026-03-21T22:29:18Z",
      "destination": "https://kubernetes.default.svc",
      "project": "homelab"
    },
    "webhookUrl": "http://n8n.n8n.svc.cluster.local:5678/webhook/kube-triage",
    "executionMode": "production"
  }
]
```

---

### Node02 - HTTP Request: Get_postlist

- Purpose: get the pod list
- Node Type: http request
- Node Name: Get_postlist
- Method: GET
- URL: {{ $env.K8S_API }}/api/v1/namespaces/{{ $json.body.namespace }}/pods
- Query Parameters:
  labelSelector: app={{ $('Webhook').item.json.body.app }}
- Headers:
  Authorization: Bearer {{ $env.K8S_TOKEN }}

---

### Node03 - Code: Get_podname

- Purpose: extract failed pod name
- Node Type: Code
- Node Name: Get_podname
- JavaScript:

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

---

### Node04 — HTTP Request: Get_podevent

- Purpose: get the pod event
- Node Type: http request
- Node Name: Get_podevent
- Method: GET
- URL: {{ $env.K8S_API }}/api/v1/namespaces/{{ $('Get_podname').item.json.namespace }}/events
- Query Parameters:
  fieldSelector: involvedObject.name={{ $('Get_podname').item.json.podName}}
- Headers:
  Authorization: Bearer {{ $env.K8S_TOKEN }}

---

### Node05 — HTTP Request: Get_podlog

- Purpose: get the pod logs
- Node Type: http request
- Node Name: Get_podlog
- Method: GET
- URL: {{ $env.K8S_API }}/api/v1/namespaces/{{ $('Get_podname').item.json.namespace }}/pods/{{ $('Get_podname').item.json.podName}}/log
- Query Parameters:
  previous: true
- Headers:
  Authorization: Bearer {{ $env.K8S_TOKEN }}

---

### Node06 — Code: gen_context

- Purpose: generate context for prompting
- Node Type: Code
- Node Name: gen_context
- JavaScript:

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
const logs = $("Get_podlog").first().json.data || "no logs found";

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

---

### Node07 — AI Agent (Gemini)

- Node Type: AI Agent
- Node Name: AI Agent
- Model: google gemini
- Configuration:

User message: {{ $json.context }}
System Message:

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

### Node08 — Snd Email

- Subject: [INCIDENT] CrashLoopBackOff — demo-app / demo-app-6b9c97cc79-fqj9x (Namespace: n8n)

- HTML:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>KubeTriage Incident Notification</title>
    <style>
      body {
        font-family:
          -apple-system, BlinkMacSystemFont, "Segoe UI", Arial, sans-serif;
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
        font-family: "Courier New", Courier, monospace;
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
          Dear On-Call Engineer,<br /><br />
          This is an automated incident notification generated by KubeTriage. A
          pod failure has been detected in your Kubernetes cluster. Please
          review the following analysis and take appropriate action.
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
        This is an automated message. Do not reply to this email.<br />
        <strong>KubeTriage — Arguswatcher Homelab</strong>
      </div>
    </div>
  </body>
</html>
```

---
