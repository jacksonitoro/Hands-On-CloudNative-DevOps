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