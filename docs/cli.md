# Rollback & Sync

[Back](../README.md)

- [Rollback \& Sync](#rollback--sync)
  - [Rollback](#rollback)
  - [Sync App](#sync-app)

---

## Rollback

```sh
# ###############################
# confirm the situation
# ###############################
argocd app get demo-app | grep -i "health status"
# Health Status:      Degraded

kubectl logs demo-app-65cf56b86d-vpnx8 -n n8n --previous
# /docker-entrypoint.sh: Configuration complete; ready for start up
# 2026/03/21 23:59:41 [emerg] 1#1: unexpected end of file, expecting "}" in /etc/nginx/conf.d/default.conf:19
# nginx: [emerg] unexpected end of file, expecting "}" in /etc/nginx/conf.d/default.conf:19

kubectl describe pod demo-app-65cf56b86d-vpnx8 -n n8n
# Events:
#   Type     Reason     Age                   From               Message
#   ----     ------     ----                  ----               -------
#   Warning  BackOff    62s (x50 over 31m)    kubelet            spec.containers{nginx}: Back-off restarting failed container nginx in pod demo-app-65cf56b86d-vpnx8_n8n(2a6acb5d-1600-45c2-94f6-276ce8093c92)

kubectl get configmap demo-app-nginx-config -n n8n -o yaml

# ###############################
# Decision: hotfix / rollback
# ###############################

# ###############################
# Mitigation Action
# ###############################
# disable auto-sync
argocd app set demo-app --sync-policy none

# or patch
kubectl patch app demo-app -n argocd \
  --type merge \
  -p '{"spec":{"syncPolicy":null}}'
# application.argoproj.io/demo-app patched

# Confirm
argocd app get demo-app | grep -i "sync policy"
# Sync Policy:        Manual

# Roll back
argocd app history demo-app
argocd app rollback demo-app 1
# mannually restart deploy
kubectl rollout restart deployment/demo-app -n n8n
# minitor
kubectl rollout status deployment/demo-app -n n8n

#  Verify service health
kubectl get deployment/demo-app -n n8n
```

---

## Sync App

```sh
# ###############################
# Bug fixing
# ###############################

# ###############################
# Deploy new version
# ###############################
# Sync to the new revision
argocd app sync demo-app
```
