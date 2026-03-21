
```sh

```

- Node: HTTP Request node: list pods in demo namespace**

Method:  GET
URL:     https://kubernetes.default.svc/api/v1/namespaces/demo/pods
Headers: Authorization = Bearer {{ $json.token }}
         Content-Type  = application/json
SSL:     disable certificate verification (self-signed cert in minikube)


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
  -n n8n \
  -- curl -X POST \
  http://n8n.n8n.svc.cluster.local/webhook/a7560150-a895-4a10-8e22-a27a74fa2b01 \
  -H "Content-Type: application/json" \
  -d '{
    "app": "demo-app",
    "namespace": "n8n",
    "health": "Degraded",
    "sync": "Synced",
    "revision": "abc123",
    "message": "Back-off restarting failed container nginx in pod demo-app"
  }'
```

---

node http request

Method: GET
URL:    {{ $env.K8S_API }}/api/v1/namespaces/{{ $('Webhook').item.json.body.namespace }}/pods
Headers:
  Authorization: Bearer {{ $env.K8S_TOKEN }}
Query Parameters:
  labelSelector: app={{ $('Webhook').item.json.body.app }}