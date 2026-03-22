# Project KubeTriage

- [Project KubeTriage](#project-kubetriage)
  - [`n8n` Workflow](#n8n-workflow)
  - [How it works?](#how-it-works)
    - [T0: Healthy Deployment](#t0-healthy-deployment)
    - [T1: Sync for update](#t1-sync-for-update)
    - [T2: Degraded Deployment and Triage Notification](#t2-degraded-deployment-and-triage-notification)
    - [T3: Bugfix](#t3-bugfix)

---

## `n8n` Workflow

![pic](./docs/diagram_n8n_workflow.png)

[Creating kubetriage n8n workflow](./n8n/README.md)

---

## How it works?

### T0: Healthy Deployment

- Application runs healthly

![pic](./docs/demo_t0_argocd.png)

---

### T1: Sync for update

- New version(with bug) commit and push to repo
- Sync app

![pic](./docs/demo_t1_argocd.png)

---

### T2: Degraded Deployment and Triage Notification

- Application get degraded

![pic](./docs/demo_t2_argocd.png)

- n8n workflow get triggered to create a summary report.

![pic](./docs/demo_t2_n8n.png)

- The summary report is sent.

![pic](./docs/demo_t2_email01.png)

![pic](./docs/demo_t2_email02.png)

---

### T3: Bugfix

- Confirm

![pic](./docs/demo_t3_argocd.png)

- [Rollback and Sync Action](./docs/cli.md)
