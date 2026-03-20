
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
kubectl logs -n argocd deployment/argocd-notifications-controller --tail=20 | grep demo-web
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