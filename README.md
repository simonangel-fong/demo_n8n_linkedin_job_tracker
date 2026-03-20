# Project KubeTriage

AI Assistant for Kubernetes Incident Triage



```sh
# connect with PVE host
ssh aadmin@10.0.0.50

# connect with VM with ProxyJump
ssh -J aadmin@10.0.0.50 ubuntuadmin@192.168.100.150

# test local connection
curl -vk https://192.168.100.150 -H "Host: homelab-n8n.arguswatcher.net"


kubectl port-forward svc/argocd-server -n argocd 8010:80
kubectl port-forward svc/n8n -n n8n 8020:80

```

**This repository contains sanitized n8n workflow definitions for demonstration purposes. Credentials, secrets, and environment-specific values are managed separately in the runtime environment.**


```sh
# break the app

```