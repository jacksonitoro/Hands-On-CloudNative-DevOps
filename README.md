# Hands-On Cloud-Native DevOps Series: Zero to Production

A comprehensive, evidence-based engineering repository documenting the transition of a full-stack, stateful microservices application into a resilient, observable, and cloud-native architecture.

---

## 🎯 Core Aim of the Series

The primary goal of this series is to bridge the gap between application software engineering and cloud-native platform engineering. Rather than working with superficial demo containers or toy scripts, this series uses **[Budget Buddy](https://github.com/jacksonitoro/budget-buddy)**—a production-grade full-stack personal finance application—as the runtime testbed.

This project deliberately embraces real-world operational challenges:
* **Heterogeneous Tech Stack:** A dynamic Next.js frontend (port `3000`) and an Express/Node.js REST API (port `5000`).
* **Stateful Persistence:** Managing SQLite data across container restarts and Kubernetes pod rescheduling without relying on managed cloud databases.
* **Declarative Orchestration:** Moving from local Compose environments to Kubernetes (`kind`), Helm, GitOps (`ArgoCD`), and enterprise monitoring (`Prometheus/Grafana/Loki`).

---

## 🚀 Key Benefits & What You Learn

Practicing these workflows provides practical platform engineering capabilities:
1. **Container Security & Hardening:** Identifying, triaging, and remediating Common Vulnerabilities and Exposures (CVEs) using Docker Scout before promoting images.
2. **Declarative Kubernetes Architecture:** Translating procedural Docker Compose specifications into native Kubernetes manifests (`Deployments`, `ClusterIP`, `NodePort`, `PersistentVolumeClaims`).
3. **Resilience & Storage Durability:** Establishing decoupled persistent volume claims that maintain application state during node drain, crash loops, and pod replacement.
4. **Diagnostic & Failure Analysis:** Mastering Kubernetes troubleshooting primitives (`kubectl describe`, port-forward namespace diagnostics, and pod liveness/readiness probes).
5. **GitOps & Production Readiness:** Automating deployments with Helm packaging, ArgoCD continuous delivery, and full observability stacks.

---

## 🗺️ Engineering Progression Overview


---

## 📅 Weekly Milestones & Implementation Guide

### Week 1: Multi-Container Orchestration & Security Auditing

#### Objectives
* Containerize the full-stack Budget Buddy application using optimized Dockerfiles.
* Orchestrate frontend, backend, and persistent local volumes using Docker Compose.
* Perform runtime vulnerability auditing using Docker Scout and remediate supply-chain CVEs.

#### Key Procedures
1. **Multi-Stage Container Builds:**
   * Built backend and frontend production images with minimal runtime footprints.
2. **Compose Orchestration:**
   * Configured cross-container service networking with dynamic environment variables.
   * Bound SQLite storage to host volumes to verify persistence across `docker compose down` and `up`.
3. **Vulnerability Assessment:**
   * Audited base images using `docker scout cves`.
   * Remediated high/critical npm dependencies via targeted audit fixes.

#### Security Audit Evidence
![Vulnerability Scan & Analysis](assets/vulnerability%20scan%20and%20analysis.png)
![Frontend Scan Post-Rebuild](assets/frontend%20vulnerability%20scan%20result%20after%20rebuilding.png)

#### Challenges Encountered & Resolutions
* **Challenge:** High-severity vulnerabilities detected in npm dependencies during security scans.
  * **Root Cause:** Outdated transitive dependencies inside the base image.
  * **Resolution:** Executed targeted `npm audit fix` inside container builds, verified zero critical/high CVE flags with Docker Scout, and locked dependency trees in `package-lock.json`.

---

### Week 2: Declarative Orchestration & Persistence on Kubernetes (`kind`)

#### Objectives
* Provision a local Kubernetes control plane using `kind` (Kubernetes in Docker).
* Translate Docker Compose definitions into native Kubernetes manifests (`k8s/`).
* Configure durable state management using `PersistentVolumeClaim` (PVC) for SQLite.
* Establish internal service discovery via `ClusterIP` and external exposure via `NodePort`.
* Validate pod self-healing and SQLite persistence across pod termination.

#### Key Procedures

1. **Cluster Provisioning & Image Ingestion:**

```bash
# Create a local single-node Kubernetes cluster

kind create cluster --name cloudnative-cluster

# Load local Docker images directly into Kind's containerd cache

kind load docker-image budget-buddy-backend:latest --name cloudnative-cluster
kind load docker-image budget-buddy-frontend:latest --name cloudnative-cluster

```

2. **Persistent Storage Manifest (`k8s/backend-pvc.yaml`):**


```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backend-sqlite-pvc
  labels:
    app: budget-buddy
    tier: backend
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

3. **Backend Workload & Service (`k8s/backend-deployment.yaml`, `k8s/backend-service.yaml`):**
* Configured compute boundaries (`requests: 50m CPU / 64Mi RAM`, `limits: 200m CPU / 128Mi RAM`).
* Configured readiness and liveness HTTP probes against `/api/health`.
* Mounted PVC to `/app/data` to persist `budget_buddy.db`.
* Created a `ClusterIP` service exposing internal port `5000`.

4. **Frontend Workload & Service (`k8s/frontend-deployment.yaml`, `k8s/frontend-service.yaml`):**
* Configured `HOSTNAME: "0.0.0.0"` and `NODE_ENV: "production"`.
* Created a `NodePort` service mapping port `3000` to `30080`.

5. **Deploy & Verify:**

```bash
kubectl apply -f k8s/backend-pvc.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/backend-service.yaml
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/frontend-service.yaml

```

6. **Validate State Durability:**
* Inserted budget and expense records through the browser.
* Manually deleted the active backend pod: `kubectl delete pod -l tier=backend`.
* Verified that the replacement pod automatically remounted `backend-sqlite-pvc` and returned the existing records without data loss.

#### Deployment & State Verification Evidence
![Budget Buddy Frontend Page](assets/budget%20buddy%20frontendpage.png)

#### Challenges Encountered & Resolutions

* **Challenge 1: Port Collision during Local Port-Forwarding**
* **Symptom:** `kubectl port-forward svc/budget-buddy-backend 5000:5000` failed with:
 `bind: Only one usage of each socket address (protocol/network address/port) is normally permitted.`
* **Root Cause:** Background Docker Compose containers from Week 1 were still running and occupying port `5000` on the host machine.
* **Resolution:** Executed `docker compose down` in the project root to release the host sockets before establishing the Kubernetes port-forward session.

* **Challenge 2: Frontend Readiness Probe & Port-Forward Connection Refused**
* **Symptom:** `curl -I http://localhost:3000` returned `Empty reply from server`, and `kubectl port-forward` threw:
 `failed to execute portforward in network namespace ... connect: connection refused`.
* **Root Cause:** The Next.js standalone server defaulted to binding only to `127.0.0.1` inside the pod container rather than the container runtime network interface (`0.0.0.0`).
* **Resolution:** Injected `HOSTNAME: "0.0.0.0"` into the container environment spec in `k8s/frontend-deployment.yaml` and redeployed the workload.

* **Challenge 3: Port-Forward Lifetime upon Pod Deletion**
* **Symptom:** During the resilience test, after running `kubectl delete pod -l tier=backend`, the port-forward session abruptly terminated with `error: lost connection to pod`.
* **Root Cause:** `kubectl port-forward` connects to a specific pod endpoint behind the service. Terminating the pod tears down the associated TCP socket.
* **Resolution:** Re-executed `kubectl port-forward svc/budget-buddy-backend 5000:5000` once the Deployment's replacement pod reached `1/1 Running`.

---

## 🛠️ Cluster Lifecycle Management

### Pausing the Cluster (Preserves Data & Zero Resource Usage)

```bash
# Gracefully stop the control plane
docker stop cloudnative-cluster-control-plane

# Resume the cluster when needed
docker start cloudnative-cluster-control-plane

```

### Complete Teardown & Clean Declarative Rebuild

```bash
# Delete cluster
kind delete cluster --name cloudnative-cluster

# Rebuild from declarative manifests
kind create cluster --name cloudnative-cluster
kind load docker-image budget-buddy-backend:latest --name cloudnative-cluster
kind load docker-image budget-buddy-frontend:latest --name cloudnative-cluster
kubectl apply -f k8s/



```

### Week 3: Helm Packaging, Parameterization & Release Lifecycle
#### Overview
**Summary:** This week focuses on transitioning from static Kubernetes manifests to reusable, parameterized Helm charts, enabling scalable, maintainable, and repeatable deployments for the Budget Buddy application across different environments.

#### Objectives
* Scaffold a standardized Helm v3 chart for the full-stack microservices application.
* Decouple configuration from static YAML via Go templates and `values.yaml`.
* Implement multi-tier resource templating (`Deployment`, `Service`, `PersistentVolumeClaim`).
* Validate chart rendering with `helm lint` and `helm template`.
* Execute release lifecycle operations: install, declarative upgrade, and automated rollback.

#### Key Procedures
1. **Chart Structure & Templating:**
   * Modularized templates into `templates/backend/` and `templates/frontend/`.
   * Standardized chart metadata in `Chart.yaml` using SemVer v2 specification.
   * Parameterized compute resources, replica counts, container ports, and probe intervals.
2. **Release Execution:**
   ```bash
   # Install initial release
   helm install budget-buddy ./charts/budget-buddy

   # Dynamic parameter upgrade (scale frontend replicas to 2)
   helm upgrade budget-buddy ./charts/budget-buddy --set frontend.replicaCount=2

   # Revert to base configuration
   helm rollback budget-buddy 1
   ```
3. **Release Lifecycle Execution:**
  ```bash
   # 1. Validate syntax and test template rendering
   helm lint ./charts/budget-buddy
   helm template budget-buddy ./charts/budget-buddy -f ./charts/budget-buddy/values-dev.yaml

   # 2. Deploy initial release (Revision 1)
   helm install budget-buddy ./charts/budget-buddy

   # 3. Dynamic parameter upgrade (scale frontend replicas to 2 via CLI)
   helm upgrade budget-buddy ./charts/budget-buddy --set frontend.replicaCount=2

   # 4. Instant rollback to base state
   helm rollback budget-buddy 1

   # 5. Environment promotion: apply production profile (scales backend to 2, frontend to 3)
   helm upgrade budget-buddy ./charts/budget-buddy -f ./charts/budget-buddy/values-prod.yaml

   # 6. Revert to lightweight development profile
   helm upgrade budget-buddy ./charts/budget-buddy -f ./charts/budget-buddy/values-dev.yaml

#### Challenges Encountered & Resolutions

* ***Challenge 1: Chart Metadata Validation Failure During Linting***
  *  ***Symptom:*** `helm lint budget-buddy` threw `[ERROR] Chart.yaml: name is required, apiVersion is required, and unable to load chart.`
  * ***Root Cause:*** `Chart.yaml` was initialized as an empty file without mandatory Helm metadata fields.
  * ***Resolution:*** Populated `Chart.yaml` with `apiVersion: v2, name: budget-buddy, version: 0.1.0, and appVersion: "1.0.0".`
* ***Challenge 2: Resource Collisions with Existing Static Manifests***

  * ***Symptom:*** Attempting `helm install` while Week 2 static manifests were present caused ownership and port conflicts.

  * ***Resolution:*** Cleaned up static manifests via `kubectl delete -f k8s/` before deploying via Helm, allowing Helm to fully manage resource lifecycles.
* ***Challenge 4: Unanchored .gitignore Rule Blocking Chart Tracking***

  * ***Symptom:*** Running `git add charts/budget-buddy/...` returned `The following paths are ignored by one of your .gitignore files: charts/budget-buddy`.

  * ***Root Cause:*** The rule `budget-buddy/` in `.gitignore` lacked a root anchor, matching any path containing that folder name—including `charts/budget-buddy/`.

  * ***Resolution:*** Anchored the pattern to the repository root as `/budget-buddy/`, ensuring only the top-level application repository clone was ignored while allowing Helm charts to be tracked.


### Week 4: Automated GitOps Continuous Delivery with ArgoCD
#### 1. Overview & Architecture
Week 4 transitioned the Budget Buddy deployment lifecycle from manual, client-side Helm executions (`helm install` / `helm upgrade`) to a declarative, pull-based GitOps engine managed by ArgoCD inside the local Kind cluster.
Under this model, Git acts as the single source of truth. The cluster continuously reconciles its live state against configuration files stored in GitHub, eliminating manual drift and enabling automated rollouts and disaster recovery.

```text

┌──────────────────────────────────────────────────────────────┐
│                 GitHub Repository (main)                     │
│   - charts/budget-buddy (Templates & Default Values)         │
│   - charts/budget-buddy/values-dev.yaml                      │
│   - charts/budget-buddy/values-prod.yaml                     │
│   - argocd/budget-buddy-dev.yaml                             │
│   - argocd/budget-buddy-prod.yaml                            │
└──────────────────────────────┬───────────────────────────────┘
                               │
                       Git Sync / Polling
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                  Kind Kubernetes Cluster                     │
│                                                              │
│  [Namespace: argocd]                                         │
│   └── ArgoCD Controller Stack (Server, Repo Server,          │
│       Application Controller, Redis)                         │
│               │                                              │
│       Reconciliation Loop (Self-Heal & Prune)                │
│               ├──────────────────────────┐                   │
│               ▼                          ▼                   │
│      [Namespace: default]        [Namespace: prod]           │
│       budget-buddy-dev            budget-buddy-prod          │
│       ├── Backend (1 Pod)         ├── Backend (2 Pods)       │
│       ├── Frontend (2 Pods)       ├── Frontend (3 Pods)      │
│       ├── Services                ├── Services               │
│       └── SQLite PVC (1Gi)        └── SQLite PVC (1Gi)       │
└──────────────────────────────────────────────────────────────┘



#### 2. Key Milestones Implemented

#### A. ArgoCD Infrastructure Setup

* Installed ArgoCD into a dedicated `argocd` namespace using server-side apply with conflict overrides:

```bash
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
***(Server-side apply was required to bypass the 262,144-byte limit imposed on `kubectl.kubernetes.io/last-applied-configuration` annotations by the large `applicationsets.argoproj.io` CRD).***

* Verified operational readiness for all seven core ArgoCD controllers and services (`argocd-server, argocd-repo-server`, `argocd-application-controller`, `argocd-redis, argocd-dex-server`, `argocd-notifications-controller`, `argocd-applicationset-controller`).
* Extracted initial administrator credentials from secret `argocd-initial-admin-secret` and established secure local access via port-forwarding to port `8080`.

#### B. Declarative Application CRDs

* Created and version-controlled declarative Custom Resources (`Application` CRD) under `argocd/` targeting the centralized GitHub repository:

| Specification | Development (`budget-buddy-dev.yaml`) | Production (`budget-buddy-prod.yaml`) |
|---|---|---|
| **Target Namespace** | `default` | `prod` (with `CreateNamespace=true`) |
| **Chart Path** | `charts/budget-buddy` | `charts/budget-buddy` |
| **Values Files** | `values.yaml`, `values-dev.yaml` | `values.yaml`, `values-prod.yaml` |
| **Sync Policy** | Automated (`prune: true`, `selfHeal: true`) | Automated (`prune: true`, `selfHeal: true`) |
| **Finalizer** | `resources-finalizer.argocd.argoproj.io` | `resources-finalizer.argocd.argoproj.io` |

#### C. Engineering Fixes & Root Cause Analysis
* Symptom: Backend pod restart loops (`Exit Code: 1, npm error signal SIGTERM`) and port-forwarding connection drops (`connection refused`).

* Root Cause: Overly aggressive probe parameters (`timeoutSeconds: 1s, failureThreshold: 3`) on the `/api/health` endpoint under limited CPU requests (`50m`) caused transient probe timeouts (`context deadline exceeded while awaiting headers`), prompting Kubelet to terminate the container.

* Resolution:

  1. Updated `charts/budget-buddy/templates/backend/deployment.yaml` with relaxed probe timings (`timeoutSeconds: 5`, `failureThreshold: 5`, `initialDelaySeconds: 15`).

  2. Increased backend dev resource limits in `values-dev.yaml` (`cpu: 250m`, `memory: 256Mi`) to provide adequate headroom for SQLite initialization.

  3. Cleaned misplaced probe YAML blocks from within the PVC volume definitions.

#### 3. Verification & Evidence

#### A. Automated GitOps Continuous Delivery (Scale via Git)

* *Action:* Updated `frontend.replicaCount: 2` in `charts/budget-buddy/values-dev.yaml` and pushed commit `841dccc` to `main`.

* *Result:* ArgoCD polled the repository, marked the application `OutOfSync`, initiated automated synchronization, and scaled `budget-buddy-dev-frontend` to `2/2` replicas without executing manual `helm` or `kubectl` commands.


```text
$ kubectl get deployments -l app.kubernetes.io/instance=budget-buddy-dev
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
budget-buddy-dev-backend    1/1     1            1           72m
budget-buddy-dev-frontend   2/2     2            2           60m

```

#### B. Self-Healing & Drift Correction
  * Action: Manually deleted `deployment/budget-buddy-dev-frontend` directly from the cluster using `kubectl delete deployment`.

  * Result: ArgoCD detected out-of-band cluster drift against the Git state and restored the frontend deployment within seconds (`selfHeal: true`).

#### C. Multi-Environment Isolation Verification
  * Verified concurrent deployment across both environments (`default` and `prod`) reflecting their respective values profiles:

```Plaintext
$ kubectl get pods -A -l app.kubernetes.io/name=budget-buddy
NAMESPACE   NAME                                         READY   STATUS    RESTARTS   AGE
default     budget-buddy-dev-backend-855f6b7c8c-6xgx4    1/1     Running   0          170m
default     budget-buddy-dev-frontend-7ffcf96b46-lcnm4   1/1     Running   4          3h34m
default     budget-buddy-dev-frontend-7ffcf96b46-q2pms   1/1     Running   4          3h36m
prod        budget-buddy-prod-backend-6fb5f9b8f5-5zkcs   1/1     Running   0          139m
prod        budget-buddy-prod-backend-6fb5f9b8f5-f5sxb   1/1     Running   0          139m
prod        budget-buddy-prod-frontend-77d885cd5-clf8v   1/1     Running   2          139m
prod        budget-buddy-prod-frontend-77d885cd5-l5bwb   1/1     Running   1          139m
prod        budget-buddy-prod-frontend-77d885cd5-n6m5k   1/1     Running   2          139m


```

#### D. Full-Stack Browser & Persistence Verification

* Port-forwarded frontend (`3000:3000`) and backend (`5000:5000`) services.   
* Verified backend health check returns `HTTP/1.1 200 OK` (`{"status":"OK","message":"Budget Buddy API is running"}`).   
* Verified UI operations: successfully submitted monthly budgets and recorded daily expense transactions to the bound PersistentVolumeClaim (`budget_buddy.db`) with zero client-side fetch errors. 

### Week 4: GitOps Continuous Delivery with ArgoCD

#### Milestones Achieved
1. **ArgoCD In-Cluster Installation:**
   - Deployed the ArgoCD controller stack into the `argocd` namespace on Kind via server-side apply (`--server-side --force-conflicts`).
   - Forwarded port `8080` to access the ArgoCD administrative UI and verified initial admin authentication.

2. **Declarative GitOps Custom Resources:**
   - Authored declarative `Application` CRDs in `argocd/budget-buddy-dev.yaml` and `argocd/budget-buddy-prod.yaml`.
   - Connected both applications to the GitHub repository source (`charts/budget-buddy`) targeting the `main` branch.
   - Configured automated sync policies with `prune: true` and `selfHeal: true`.

3. **Multi-Environment Promotion:**
   - **Dev (`default` namespace):** Uses `values-dev.yaml` (1 backend replica, 2 frontend replicas, lightweight compute footprints).
   - **Prod (`prod` namespace):** Uses `values-prod.yaml` (2 backend replicas with SQLite PVC, 3 frontend replicas, higher compute allocations).

4. **Evidence-Based GitOps Verification:**
   - **Automated Self-Healing:** Simulated manual cluster drift by deleting the frontend deployment via `kubectl`. ArgoCD detected the missing state and restored the deployment within seconds without manual intervention.
   - **Git-Driven Continuous Delivery:** Scaled frontend replicas from 1 to 2 by pushing a commit to `values-dev.yaml`. ArgoCD reconciled the change and scaled up pods automatically.
   - **Probe Tuning & Stability:** Identified and resolved probe timeouts (`timeoutSeconds: 5`, `failureThreshold: 5`) and resource headroom constraints, ensuring 100% stable `Running` pods across both namespaces.

