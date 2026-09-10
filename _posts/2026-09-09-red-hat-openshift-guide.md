---
title: Red Hat Openshift Guide
date:  2026-09-09 22:00:00 +0800
categories: [Knowledge, Red Hat Openshift]
tags: [kubernetes, docker, documentation, knowledge]
description: A guide on using Red Hat Openshift for building and deploying applications in the cloud enviroment.
---


# What is Red Hat OpenShift?
red Hat OpenShift is a **cloud platform** that runs your applications reliably, at scale, and automatically.

| Environments          | Description                                          |
| --------------------- | ---------------------------------------------------- |
| **Your laptop**       | Runs 1 app, for 1 person                             |
| **Red Hat Openshift** | Runs 100 apps, for millions of people, automatically |
---

# Key concepts in Red Hat Openshift
| Term                  | Description                                      |
| --------------------- | ------------------------------------------------ |
| **Container**         | A packaged app with everything it needs to run   |
| **Pod**               | One running instance of your container           |
| **Deployment**        | Instructions for how to RUN your app             |
| **BuildConfig**       | Instructions for how to BUILD your app           |
| **ImageStream**       | Where your built container image is stored       |
| **Service**           | Makes your app reachable internally              |
| **Route**             | Makes your app reachable from the internet       |
| **Namespace/Project** | A folder that groups related apps                |
| **Secret**            | Stores sensitive data (passwords, API keys)      |
| **ConfigMap**         | Stores non-sensitive settings                    |
| **PVC**               | Persistent storage that survives pod restarts    |
| **CronJob**           | A scheduled task that runs on a defined interval |
---

# Overall flow from Gitlab to deployment
```
YOUR CODE IN GITLAB
(app.py, requirements.txt)
         │
         ▼
┌───────────────────────┐
│      DOCKERFILE       │  "The Recipe"
│  - Use Python 3.11    │
│  - Copy code in       │
│  - Install libraries  │
│  - Run app.py         │
└───────────────────────┘
         │
         ▼
┌───────────────────────┐
│     BUILDCONFIG       │  "The Chef"
│  - Grab code (GitLab) │
│  - Follow Dockerfile  │
│  - Package everything │
│  - Save the result    │
└───────────────────────┘
         │
         ▼
┌───────────────────────┐
│   CONTAINER IMAGE     │  "The Packaged App"
│  - Your code          |
│  - Python runtime     │
│  - All libraries      │
└───────────────────────┘
         │
         ▼
┌───────────────────────┐
│      DEPLOYMENT       │  "The Waiter"
│  - Run N copies       │
│  - Restart if crash   │
│  - Update smoothly    │
└───────────────────────┘
         │
         ▼
    🟢 LIVE RUNNING APP
```
---

# The three environments within Red Hat Openshift
```
┌──────────────────────────────────────────────────────┐
│                  BUILD NAMESPACE                     |
|               e.g. <namespace>-build                 |
|  Contains:                                           |
|  - Dockerfile                                        |
|  - BuildConfig                                       |
|  - ImageStream                                       |
|                                                      |
│  Purpose: Build and store the container image ONCE   │
└──────────────────────────────────────────────────────┘
                            │
            ┌───────────────┴───────────────────┐
            │      SAME IMAGE used by both!     │
            ▼                                   ▼
┌──────────────────────────┐       ┌────────────────────────┐
│    STAGING NAMESPACE     │       │     PROD NAMESPACE     │
│ e.g. <namespace>-staging │       │  e.g. <namespace>-prod │
│                          │       │                        │
│   - deployment.yaml      │       │   - deployment.yaml    │
│   - Test secrets         │       │   - Real secrets       │
│   - 1 replica            │       │   - 3 replicas         │
│   - Test database        │       │   - Real database      │
│   - Test URL             │       │   - Real public URL    │
│                          │       │                        │
│ Purpose: Test safely     │       │ Purpose: Real users    │
└──────────────────────────┘       └────────────────────────┘
```
---

# The minimum files you need for deployment
```
your-project/
├── app.py                    ← Your Python code
├── requirements.txt          ← Python dependencies
├── Dockerfile                ← How to package app
├── bc.yaml                   ← ONE file, build namespace only
├── deployment-staging.yaml   ← Staging environment
└── deployment-prod.yaml      ← Production environment
```
---

# Differences: Staging vs Production enviroment
```
| Setting      | Staging             | Production        |
| ------------ | ------------------- | ----------------- |
| namespace    | <namespace>-staging | <namespace>-prod  |
| replicas     | 1                   | 3                 |
| Image source | <namespace>-build   | <namespace>-build |
| Database     | Test DB             | Real DB           |
| Secrets      | Test values         | Real values       |
```
---

# Differences: Routes vs Services
```
|                 | Service                              | Route                                        |
| --------------- | ------------------------------------ | -------------------------------------------- |
| Purpose         | Internal pod-to-pod communication    | External public access                       |
| Accessible from | Inside the cluster only              | Browser / internet                           |
| Example URL     | http://<namespace>-backend-svc:8080  | https://<namespace>-backend.apps.cluster.com |
| Created by      | Your deployment YAML                 | Manually or via YAML                         |
| Use for         | Pod-to-pod, CronJobs, DB connections | Frontend, external APIs                      |

```
>**Rule:** 
- If traffic stays inside OpenShift (e.g. your CronJob pinging the backend), use the Service name. 
- If a browser or external tool needs access, create a Route.
---

# OC Command Line Tool (CLI)

### Installation

1. In OpenShift UI: click `?` → **Command Line Tools**
2. Download `oc` for your OS (Windows/Mac/Linux)
3. Add `oc.exe` to your system PATH
4. Verify: `oc version`

### Essential Commands

```bash
# Authentication
oc login --token=sha256~xxx --server=https://api.cluster.com:6443
oc whoami                                                           # Check if still logged in

# Namespace
oc project <namespace>-staging                                      # Switch namespace
oc project                                                          # Show current namespace

# Resources
oc get pods                                                         # List pods
oc get pods -w                                                      # Watch pods (live)
oc get deployments
oc get cronjobs
oc get jobs
oc get secrets
oc get buildconfigs
oc get imagestreams

# Logs & Debugging
oc logs <pod-name>
oc logs -f bc/<buildconfig-name>                                    # Follow build logs
oc describe pod <pod-name>
oc get events --sort-by='.lastTimestamp'

# Apply / Update
oc apply -f deployment.yaml                                         # Create or update deployment
oc delete job <job-name>                                            # Delete a job
oc start-build <buildconfig-name>                                   # Build the image

# Rollback
oc rollout undo deployment/<name>
```

> **Token Expiry:** Tokens expire (usually within 24 hours). Run `oc whoami` to check. If expired, get a fresh token from the OpenShift UI → your username → **Copy login command**.
---

# Building images and deploying via OC CLI

### First-Time Setup

```bash
# 1. Switch to build namespace
oc project <namespace>-build

# 2. Import BuildConfig
oc apply -f bc.yaml

# 3. Trigger the first build
oc start-build <buildconfig-name>

# 4. Watch build progress
oc logs -f bc/<buildconfig-name>

# 5. Switch to staging
oc project <namespace>-staging

# 6. Deploy (ONCE ONLY)
oc apply -f deployment-staging.yaml

# 7. Verify pods
oc get pods -w
```

### Future Code Updates

```bash
# 1. Push code to GitLab
git push origin main

# 2. Rebuild image
oc project <namespace>-build
oc start-build <buildconfig-name>

# 3. Wait for build — pod auto-updates via imagePullPolicy: Always
oc logs -f bc/<buildconfig-name>
```

### Future YAML-Only Updates

```bash
# No rebuild needed
oc project <namespace>-staging
oc apply -f deployment-staging.yaml
```
---

# Persistent Volumne Claim (PVC)

A **PVC** is cloud storage for your database inside OpenShift.

- **Without PVC:** All data is lost when a pod restarts
- **With PVC:** Data persists even after pod crashes or restarts
- Think of it as: Container = short-term memory. PVC = the hard drive.

```yaml
kind: PersistentVolumeClaim
metadata:
  name: <namespace>-postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
---

# Containerised PostgresSQL in Red Hat Openshift

`postgres:16-alpine` is a **containerised PostgreSQL database**.

| Part       | Meaning                                              |
| ---------- | ---------------------------------------------------- |
| `postgres` | PostgreSQL — works like MySQL, stores data in tables |
| `16`       | Version 16                                           |
| `alpine`   | Lightweight Linux (~80MB vs ~400MB for standard)     |

The PVC mounts to `/var/lib/postgresql/data` inside the container so all database records survive pod restarts.
---

# When to rebuild vs "oc apply"

### Rebuild image — when code changes

Run `oc start-build <name>` when you change:

- Any `.py` file
- `requirements.txt`
- `Dockerfile`

### `oc apply -f` only — when config file changes

Run `oc apply -f yourfile.yaml` when you change:

- `cronjob.yaml`
- `deployment.yaml`
- `bc.yaml`
---

# Secrets inside Red Hat Openshift

Update and store all secrets directly in OpenShift UI. Whenever there is a change in secret value, there is no need to rebuild and no `oc apply` needed. The next pod startup automatically reads and take in the values from the secrets.
---

# Port forwarding for local development

### What it does
Port forwarding creates a **temporary tunnel** from your local computer to a pod running inside OpenShift. This lets you test services locally without needing a public Route.

```
YOUR LAPTOP          OPENSHIFT
────────────         ──────────
localhost:8080  ←──► <namespace>-backend-svc:8080
     ↑
Your Python script
or browser calls this
```

### Command

```bash
# Forward local port 8080 to the service port 8080
oc port-forward svc/<namespace>-backend-svc 8080:8080

# Forward to a specific pod directly
oc port-forward pod/<pod-name> 8080:8080

# Use a different local port if 8080 is taken
oc port-forward svc/<namespace>-backend-svc 9090:8080
```

> **Important:** Keep the terminal window open. Closing it breaks the tunnel. Open a second terminal to run your scripts.
---

# Cronjobs in Red Hat penShift

A **CronJob** is a scheduled task that runs automatically at defined intervals. It creates a **Job** (one-time run) each time it triggers, which in turn creates a **Pod**.

```
CronJob (schedule) → Job (one-time) → Pod (runs script) → Exits
```

### Key cronjob settings

| Setting                      | Purpose                                                       |
| ---------------------------- | ------------------------------------------------------------- |
| `schedule`                   | Cron expression e.g. `"*/15 * * * *"` = every 15 min          |
| `concurrencyPolicy: Forbid`  | Skip new run if previous still running                        |
| `concurrencyPolicy: Replace` | Kill old run, start new one                                   |
| `startingDeadlineSeconds`    | Grace period to start a missed run                            |
| `activeDeadlineSeconds`      | Kill job if it exceeds this duration                          |
| `successfulJobsHistoryLimit` | How many completed pods to keep                               |
| `failedJobsHistoryLimit`     | How many failed pods to keep                                  |
| `restartPolicy: Never`       | Don't restart a failed pod (CronJob retries on next schedule) |
| `backoffLimit: 0`            | No retries on failure                                         |

### Cron schedule reference

```
"*/15 * * * *"  = every 15 minutes
"0 * * * *"     = every hour
"0 9 * * *"     = every day at 9am
"0 9 * * 1-5"   = weekdays at 9am

┌── minute (0-59)
│ ┌── hour (0-23)
│ │ ┌── day of month (1-31)
│ │ │ ┌── month (1-12)
│ │ │ │ ┌── day of week (0-6)
* * * * *
```

### Manually triggering a cronjob

```bash
# Create a one-time test job from your CronJob template
oc create job --from=cronjob/<namespace>-liveness-check <namespace>-liveness-test-1

# Watch it run
oc get pods -w

# Check logs
oc logs <pod-name>

# Clean up after testing
oc delete job <namespace>-liveness-test-1
```

> Jobs cannot be overwritten. Use a new name (e.g. `<namespace>-liveness-test-2`) or delete the old one first.
---

# Health Checks and Liveness Probes

### Liveness vs Readiness

| Probe              | Purpose                       | On Failure                             |
| ------------------ | ----------------------------- | -------------------------------------- |
| **livenessProbe**  | Is the app still alive?       | OpenShift **restarts** the pod         |
| **readinessProbe** | Is the app ready for traffic? | OpenShift **stops routing** to the pod |

### How It Works

OpenShift does **not** read your Python code. It simply sends an HTTP request to the pod's IP and port at the configured path:

```
OpenShift → GET http://[pod-ip]:8080/health → expects HTTP 200
```

Your FastAPI app must have this endpoint:

```python
@app.get("/health")
def health_check():
    return {"status": "ok"}
```

Returning `HTTP 200` = healthy. Returning `HTTP 500` = OpenShift restarts the pod.

### YAML Configuration

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 20   # wait before first check
  periodSeconds: 30          # check every 30 seconds
  timeoutSeconds: 10         # fail if no response in 10s
  failureThreshold: 3        # restart after 3 failures
```

### Using logger library for Health Checks

Use Python's `logging` library (not `print`) in your health check scripts because:

- **Structured output** — includes timestamps, log levels (`INFO`, `ERROR`)
- **Visible in OpenShift logs** — appears in the Pod **Logs tab** in the UI
- **`sys.exit(1)`** signals failure to OpenShift, marking the Job as **Failed**
- **`sys.exit(0)`** signals success, marking the Job as **Completed**

```python
import logging
import sys

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)
logger = logging.getLogger(__name__)

logger.info("✅ Service is healthy")
logger.error("❌ Service failed")
sys.exit(1)  # OpenShift marks Job as Failed
```
---

# Git Branching Strategy

### When to Branch Instead of Using Main

- **Always branch** when developing new features, fixes, or testing deployments
- **Never test directly on `main`** — it affects the whole team

```
main (stable, protected)
 └── feature/liveness-check   ← your work
 └── feature/new-endpoint
 └── fix/postgres-timeout
```

### Creating and Pushing a Branch

```bash
# Create and switch to new branch
git checkout -b feature/liveness-check

# Stage changes
git add .

# Commit
git commit -m "feat: add liveness checker cronjob"

# Push to GitLab (first time)
git push origin feature/liveness-check
```

> If VS Code shows **"The branch has no remote branch. Would you like to publish this branch?"** — click **OK**. This is normal for a new branch being pushed to GitLab for the first time.

### Switching bc.yaml to Use Your Branch

Update the `ref` field in `bc.yaml` to point to your branch instead of `main`:

```yaml
source:
  git:
    uri: git@gitlab.../<namespace>.git
    ref: feature/liveness-check   # ← change this
```

Then apply the update:

```bash
oc project <namespace>-build
oc apply -f bc.yaml
oc start-build <buildconfig-name>
```

### Flow After Development Is Complete

1. **Test** your feature on `<namespace>-staging` using your branch
2. Go to GitLab → **Create Merge Request** (`feature/liveness-check` → `main`)
3. Team reviews and approves
4. **Merge** to `main`
5. Update `bc.yaml` `ref` back to `main`
6. Rebuild and deploy to production

```
feature/liveness-check
         │
         │ Merge Request (code review)
         ▼
        main
         │
         │ oc start-build (production)
         ▼
    Production Deploy
```
---

# Debugging Guide

### Pod not starting?

1. Go to **Workloads → Deployments → Details** → check **Conditions**
2. Go to **Workloads → Pods** → click pod → **Events tab**
3. Click pod → **Logs tab**

### Common errors

| Error                     | Cause                                  | Fix                                     |
| ------------------------- | -------------------------------------- | --------------------------------------- |
| `exceeded quota: cpu`     | CPU limit hit                          | Reduce CPU requests or ask admin        |
| `secret not found`        | Secret missing in namespace            | Create secret in correct namespace      |
| `ImagePullBackOff`        | Wrong image path or build not complete | Fix image path or wait for build        |
| `CrashLoopBackOff`        | App crashing on startup                | Check Logs tab                          |
| `must specify limits.cpu` | Missing resources on initContainer     | Add `resources` block to all containers |

### Pod status reference

| Status               | Meaning                   |
| -------------------- | ------------------------- |
| 🔵 `Running`          | Healthy and working       |
| 🟢 `Completed`        | Job finished successfully |
| 🔴 `CrashLoopBackOff` | App keeps crashing        |
| ⚪ `Pending`          | Waiting to start          |
| 🟡 `Init`             | initContainer running     |
| 🔴 `ImagePullBackOff` | Cannot find or pull image |
---

# Important considerations

### Secrets

- Secrets are **namespace-scoped** — they do not cross namespaces automatically
- Create secrets with the **same name** in each namespace but with **different values** (test vs real)
- Updating a secret in the UI takes effect on the **next pod startup** — no rebuild needed

### Resources

Always specify `requests` and `limits` for **every container**, including `initContainers`:

```yaml
resources:
  requests:
    cpu: '100m'
    memory: '128Mi'
  limits:
    cpu: '200m'
    memory: '256Mi'
```

| Container       | CPU Request | Memory Request |
| --------------- | ----------- | -------------- |
| `initContainer` | `50m`       | `64Mi`         |
| App container   | `200m`      | `512Mi`        |
| PostgreSQL      | `200m`      | `256Mi`        |

### YAML Files in GitLab

- YAML files in GitLab are **just stored there** — OpenShift does NOT auto-read them
- You must **import them manually** into OpenShift (once)
- After that, use `oc apply -f` or edit via the YAML tab in the UI

### Cleaning Exported YAML
When copying a YAML from one namespace to another, remove these auto-generated fields:

```
❌ resourceVersion
❌ uid
❌ creationTimestamp
❌ generation
❌ managedFields
❌ entire status section
```
