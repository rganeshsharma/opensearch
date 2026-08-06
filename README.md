# OpenSearch & OpenSearch Dashboards — Installation SOP for POC in Testing cluster 

## NOTE: This design is meant for POC only we need more nodes and resources for production ready OpenSearch Implementation 

## Deployment of OpenSearch 3-node cluster and OpenSearch Dashboards on**RKE2 Kubernetes** with **Rook Ceph** block storage.

---

## Table of Contents

- [1. Purpose & Scope](#1-purpose--scope)
- [2. Architecture](#2-architecture)
- [3. Prerequisites](#3-prerequisites)
- [4. Installation Procedure](#4-installation-procedure)
- [5. Post-Deployment Validation](#5-post-deployment-validation)
- [6. Troubleshooting](#6-troubleshooting)
- [7. Upgrading OpenSearch](#7-upgrading-opensearch)
- [8. References](#8-references)
- [9. Revision History](#9-revision-history)

---

## 1. Purpose & Scope

This SOP provides step-by-step instructions for deploying a POC OpenSearch cluster (3-node StatefulSet) and OpenSearch Dashboard on an RKE2 Kubernetes cluster using Rook Ceph block storage for persistent volumes. It covers prerequisites, network requirements, Helm-based installation, security hardening, and post-deployment validation.

### 1.1 Scope

| Item | Detail |
|---|---|
| Target environment | RKE2 Kubernetes cluster (3+ worker nodes) |
| Storage backend | Rook Ceph (CephBlockPool / RBD) |
| OpenSearch | 3-replica StatefulSet (combined master/data/ingest roles) |
| OpenSearch Dashboards | 1–2 replica Deployment |
| Namespace | `opensearch` |

### 1.2 Out of Scope

- RKE2 cluster installation
- Rook Ceph cluster initial setup (assumed pre-configured)
- LDAP/SSO integration
- Index lifecycle management configuration

---

## 2. Architecture

![alt text](image-1.png)

### NOTE: Anti-affinity: HARD — each pod on a separate worker node and PodDisruptionBudget: maxUnavailable=1

### 2.1 Component Summary

| Component | Type | Count | Key Ports |
|---|---|---|---|
| `opensearch-cluster-master-{0,1,2}` | StatefulSet Pod | 3 | 9200 (HTTP), 9300 (Transport), 9600 (Metrics) |
| `opensearch-cluster-master` | ClusterIP Service | 1 | 9200 |
| `opensearch-cluster-master-headless` | Headless Service | 1 | 9300 (Discovery) |
| OpenSearch Dashboards | Deployment Pod | 1–2 | 5601 (UI), 9601 (Metrics) |
| PersistentVolumeClaim | Rook Ceph RBD | 3 (one per pod) | — |
| `opensearch-credential` | Kubernetes Secret | 1 | — |

---

## 3. Prerequisites

### 3.1 Infrastructure Requirements

| Requirement | Minimum | Recommended | Notes |
|---|---|---|---|
| RKE2 Worker Nodes | 3 nodes | 3–5 nodes | One OpenSearch pod per node (hard anti-affinity) |
| CPU per worker | 4 vCPU | 8 vCPU | OpenSearch + Rook OSD on same node |
| RAM per worker | 16 GB | 32 GB | JVM heap = 50% of container memory |
| Disk per worker | 200 GB | 500+ GB | Ceph OSD raw disk + PVC data |
| Rook Ceph | v1.10+ | v1.13+ | CephBlockPool (RBD) must be healthy |
| Kubernetes | v1.25+ | v1.28+ | RKE2 tested |
| Helm | v3.10+ | v3.14+ | For chart deployment |
| kubectl | Matching cluster | Latest stable | With kubeconfig configured |

### 3.2 Required Tools on Deployment Host

- `helm` v3.10+ — installed and in PATH
- `kubectl` — configured with kubeconfig pointing to target RKE2 cluster
- `curl` / `nc` — for connectivity testing
- `git` — to clone/manage Helm values files

Verify tools are available:

```bash
helm version
kubectl version --client
kubectl cluster-info
kubectl get nodes -o wide
```

### 3.3 Rook Ceph Health Check

Before deploying OpenSearch, verify Rook Ceph is healthy and the StorageClass exists:

```bash
# Verify Ceph cluster health
kubectl get cephcluster -n rook-ceph
# Expected: HEALTH_OK

# Verify CephBlockPool exists
kubectl get cephblockpool -n rook-ceph

# Verify StorageClass
kubectl get storageclass | grep rook-ceph-block

# Verify Ceph CSI pods are running
kubectl get pods -n rook-ceph | grep csi
```

> **⚠️ NOTE:** If the CephBlockPool shows `HEALTH_WARN` or `HEALTH_ERR`, do not proceed. Resolve Ceph issues first — PVC provisioning will fail silently, causing pods to hang in `Init` state.

### 3.4 Network / Firewall Requirements

The following ports must be open between all RKE2 worker nodes:

| Port | Protocol | Direction | Service | Purpose |
|---|---|---|---|---|
| 9200 | TCP | Worker ↔ Worker | OpenSearch | REST API / HTTP |
| 9300 | TCP | Worker ↔ Worker | OpenSearch | Inter-node transport & cluster discovery |
| 9600 | TCP | Worker ↔ Worker | OpenSearch | Performance Analyzer / Metrics |
| 5601 | TCP | Client → Worker | Dashboards | Web UI |
| 6789 | TCP | Worker ↔ Worker | Rook Ceph MON | Ceph Monitor v1 |
| 3300 | TCP | Worker ↔ Worker | Rook Ceph MON | Ceph Monitor v2 |
| 6800–7300 | TCP | Worker ↔ Worker | Rook Ceph OSD | OSD peering and replication |
| 9283 | TCP | Prometheus → Worker | Ceph MGR | Prometheus metrics scrape |

Apply firewall rules on each worker node (`firewalld`):

```bash
sudo ufw allow 9200/tcp
sudo ufw allow 9300/tcp
sudo ufw allow 9600/tcp
sudo ufw allow 5601/tcp
sudo ufw allow 6789/tcp
sudo ufw allow 3300/tcp
sudo ufw allow 9283/tcp
sudo ufw allow 6800:7300/tcp
sudo ufw status
```

### 3.5 Kernel Parameter — vm.max_map_count

OpenSearch requires `vm.max_map_count >= 262144` on every worker node. Apply on each worker:

```bash
# Temporary (lost on reboot)
sudo sysctl -w vm.max_map_count=262144

# Permanent (survives reboot)
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.d/99-opensearch.conf
sudo sysctl --system

# Verify
sysctl vm.max_map_count
```

> **🔴 CRITICAL:** If `vm.max_map_count` is not set, OpenSearch pods will crash-loop with a bootstrap check failure. This **must** be set on **all** worker nodes before deployment.

---
## 4. Installation Procedure

### Step 1 — Create Namespace

```bash
kubectl create namespace opensearch

# Verify
kubectl get namespace opensearch
```

### Step 2 — Create Credentials Secret

The password must meet OpenSearch 2.12+ complexity requirements: minimum 8 characters with at least one uppercase, lowercase, digit, and special character.

```bash
kubectl -n opensearch create secret generic opensearch-credential \
  --from-literal=username=admin \
  --from-literal=password='YourStr0ng!Pass'

# Verify secret was created
kubectl get secret opensearch-credential -n opensearch
kubectl describe secret opensearch-credential -n opensearch
```

### Step 3 — Add OpenSearch Helm Repository

```bash
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm repo update

# Verify available chart versions
helm search repo opensearch/opensearch --versions | head -10
helm search repo opensearch/opensearch-dashboards --versions | head -5
```

### Step 4 — Prepare opensearch-values.yaml

#### NOTE: Kindly refer the override-values.yml file for our use case customization

## Create `opensearch-values.yaml` with the following configuration:
### NOTE : For Production Kindly separate master and data roles of OpenSearch on separate worker nodes

```yaml
clusterName: "opensearch-cluster"
nodeGroup: "master"
masterService: "opensearch-cluster-master"
singleNode: false
replicas: 3

roles:
  - master
  - ingest
  - data
  - remote_cluster_client

# JVM heap — must be 50% of container memory
opensearchJavaOpts: "-Xmx8g -Xms8g"

resources:
  requests:
    cpu: "2000m"
    memory: "16Gi"
  limits:
    cpu: "4000m"
    memory: "16Gi"

# Rook Ceph block storage
persistence:
  enabled: true
  enableInitChown: false        # Disabled — use fsGroupChangePolicy instead
  storageClass: "rook-ceph-block"
  accessModes:
    - ReadWriteOnce
  size: 100Gi

podSecurityContext:
  fsGroup: 1000
  runAsUser: 1000
  fsGroupChangePolicy: "OnRootMismatch"

# Admin password from secret
extraEnvs:
  - name: OPENSEARCH_INITIAL_ADMIN_PASSWORD
    valueFrom:
      secretKeyRef:
        name: opensearch-credential
        key: password
  - name: OPENSEARCH_USERNAME
    valueFrom:
      secretKeyRef:
        name: opensearch-credential
        key: username

# Hard anti-affinity — one pod per node
antiAffinity: "hard"
antiAffinityTopologyKey: "kubernetes.io/hostname"

# Disable service links to prevent slow startup
enableServiceLinks: false

# sysctl handled at node level (see prerequisite Step 3.5)
sysctlInit:
  enabled: false

# Pod Disruption Budget
maxUnavailable: 1
podManagementPolicy: "Parallel"
terminationGracePeriod: 300

# Probes
startupProbe:
  tcpSocket:
    port: 9200
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 30

readinessProbe:
  tcpSocket:
    port: 9200
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

livenessProbe:
  tcpSocket:
    port: 9200
  periodSeconds: 20
  timeoutSeconds: 5
  failureThreshold: 10
  initialDelaySeconds: 60

# OpenSearch config
config:
  opensearch.yml: |
    cluster.name: opensearch-cluster
    network.host: 0.0.0.0
    plugins.security.disabled: false
    plugins.security.ssl.http.enabled: false
    plugins.security.allow_unsafe_democertificates: true
    plugins.security.allow_default_init_securityindex: true
    plugins.security.authcz.admin_dn:
      - 'CN=kirk,OU=client,O=client,L=test,C=de'
    plugins.security.audit.type: internal_opensearch
    plugins.security.restapi.roles_enabled:
      - 'all_access'
      - 'security_rest_api_access'

protocol: https
httpPort: 9200
transportPort: 9300
metricsPort: 9600
```

### Step 5 — Install OpenSearch via Helm

```bash
# Dry-run first to validate
helm install opensearch opensearch/opensearch \
  --namespace opensearch \
  --values opensearch-values.yaml \
  --dry-run --debug 2>&1 | head -50

# Actual install
helm install opensearch opensearch/opensearch \
  --namespace opensearch \
  --values opensearch-values.yaml \
  --wait --timeout 10m

# Verify Helm release
helm list -n opensearch
```

> **📝 NOTE:** The `--wait --timeout 10m` flag makes Helm wait until all pods are Ready before returning. If it times out, check pod events with: `kubectl describe pod opensearch-cluster-master-0 -n opensearch`

### Step 6 — Monitor OpenSearch Pod Startup

```bash
# Watch pod status
kubectl get pods -n opensearch -w

# Check initContainer progress if stuck in Init state
kubectl describe pod opensearch-cluster-master-0 -n opensearch | grep -A 20 'Init Containers'

# Check initContainer logs individually
kubectl logs opensearch-cluster-master-0 -n opensearch -c fsgroup-volume
kubectl logs opensearch-cluster-master-0 -n opensearch -c sysctl
kubectl logs opensearch-cluster-master-0 -n opensearch -c configfile

# Check main container logs once pods are Running
kubectl logs -f opensearch-cluster-master-0 -n opensearch

# Verify PVCs are Bound
kubectl get pvc -n opensearch
```

### Step 7 — Prepare opensearch-dashboards-values.yaml

Create `opensearch-dashboards-values.yaml`:

```yaml
opensearchHosts: "https://opensearch-cluster-master:9200"
replicaCount: 1

# Credentials from same secret
opensearchAccount:
  secret: "opensearch-credential"
  keyPassphrase:
    enabled: false

extraEnvs:
  - name: OPENSEARCH_USERNAME
    valueFrom:
      secretKeyRef:
        name: opensearch-credential
        key: username
  - name: OPENSEARCH_PASSWORD
    valueFrom:
      secretKeyRef:
        name: opensearch-credential
        key: password
  - name: NODE_OPTIONS
    value: "--max-old-space-size=900"

config:
  opensearch_dashboards.yml: |
    server.name: opensearch-dashboards
    server.host: "0.0.0.0"
    opensearch.hosts: [https://opensearch-cluster-master:9200]
    opensearch.ssl.verificationMode: none
    opensearch.username: '${OPENSEARCH_USERNAME}'
    opensearch.password: '${OPENSEARCH_PASSWORD}'
    opensearch.requestHeadersAllowlist: [authorization, securitytenant]
    opensearch_security.multitenancy.enabled: true
    opensearch_security.multitenancy.tenants.preferred: [Private, Global]
    opensearch_security.cookie.secure: false

resources:
  requests:
    cpu: "200m"
    memory: "1Gi"
  limits:
    cpu: "1000m"
    memory: "1Gi"

serviceAccount:
  create: true
  automountServiceAccountToken: false

startupProbe:
  httpGet:
    path: /api/status
    port: 5601
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 30
  initialDelaySeconds: 30

readinessProbe:
  httpGet:
    path: /api/status
    port: 5601
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 5
  initialDelaySeconds: 30

livenessProbe:
  tcpSocket:
    port: 5601
  periodSeconds: 20
  timeoutSeconds: 5
  failureThreshold: 10
  initialDelaySeconds: 60

updateStrategy:
  type: "RollingUpdate"

service:
  type: ClusterIP
  port: 5601
```

### Step 8 — Install OpenSearch Dashboards via Helm

```bash
helm install opensearch-dashboards opensearch/opensearch-dashboards \
  --namespace opensearch \
  --values opensearch-dashboards-values.yaml \
  --wait --timeout 5m

# Verify
helm list -n opensearch
kubectl get pods -n opensearch
```

---

## 5. Post-Deployment Validation

### 5.1 Verify All Pods Running

```bash
kubectl get pods -n opensearch -o wide

# Expected output:
# opensearch-cluster-master-0   1/1   Running   0   5m   <ip>   worker-1
# opensearch-cluster-master-1   1/1   Running   0   5m   <ip>   worker-2
# opensearch-cluster-master-2   1/1   Running   0   5m   <ip>   worker-3
# opensearch-dashboards-xxx     1/1   Running   0   3m   <ip>   worker-x
```

### 5.2 Verify Cluster Health
#### NOTE: Exposed the Dashboard service through Ingress based on Ingress Controller (Kong)

```bash
# Port-forward to access API locally
kubectl port-forward svc/opensearch-cluster-master 9200:9200 -n opensearch &

# Check cluster health (status should be 'green')
curl -s -k https://opensearch-cluster-master.opensearch.svc.cluster.local:9200/_cluster/health?pretty \
  -u admin:'YourStr0ng!Pass'

# Check all 3 nodes are in the cluster
curl -s -k https://opensearch-cluster-master.opensearch.svc.cluster.local:9200/_cluster/nodes?v \
  -u admin:'YourStr0ng!Pass'

# Verify cluster has elected a master
curl -s -k https://opensearch-cluster-master.opensearch.svc.cluster.local:9200/_cluster/master?v \
  -u admin:'YourStr0ng!Pass'
```

Expected cluster health response:

```json
{
  "cluster_name" : "opensearch-cluster",
  "status" : "green",
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : X,
  "unassigned_shards" : 0
}
```

### 5.3 Verify PVCs and Storage

```bash
# All PVCs should be Bound
kubectl get pvc -n opensearch

# Check PV details
kubectl get pv | grep opensearch

```

### 5.4 Verify Dashboards Connectivity
#### NOTE: Exposed the Dashboard service through Ingress based on Ingress Controller (Kong)
```bash
# Port-forward Dashboards
kubectl port-forward svc/opensearch-dashboards 5601:5601 -n opensearch &

# Check Dashboards status API
curl -s http://localhost:5601/api/status | python3 -m json.tool | head -20

# Access in browser: http://localhost:5601
# Login with credentials from opensearch-credential secret
```

### 5.5 Validation Checklist

| Check | Command / Action | Expected Result | Status |
|---|---|---|---|
| All pods Running | `kubectl get pods -n opensearch` | 3 OpenSearch + 1 Dashboard pods Ready `1/1` | `[ ]` |
| PVCs Bound | `kubectl get pvc -n opensearch` | All PVCs show `STATUS=Bound` | `[ ]` |
| Cluster health green | `curl /_cluster/health` | `status: green, nodes: 3` | `[ ]` |
| Master elected | `curl /_cat/master` | One node listed as master | `[ ]` |
| No unassigned shards | `curl /_cluster/health` | `unassigned_shards: 0` | `[ ]` |
| Dashboards accessible | `curl :5601/api/status` | HTTP 200, state: green | `[ ]` |
| Dashboard login works | Browser `http://localhost:5601` | Login with admin credentials succeeds | `[ ]` |
| Anti-affinity enforced | `kubectl get pods -n opensearch -o wide` | Each pod on different worker node | `[ ]` |

---

## 6. Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| Pods stuck in `Init:2/3` | Busybox image pull failure (Docker Hub rate limit) | Set `persistence.enableInitChown: false` and use `fsGroupChangePolicy: OnRootMismatch` |
| Pods stuck in `Init:2/3` | PVCs in Pending state | Check `kubectl describe pvc`. Verify `storageClass` name matches Rook Ceph StorageClass |
| Pods stuck in `Init:2/3` | ConfigMap missing or secret not found | `kubectl describe pod` and check Events section for missing resource errors |
| Cluster status yellow/red | `vm.max_map_count` too low | Set `vm.max_map_count=262144` on all worker nodes and restart pods |
| Cluster status yellow/red | Nodes cannot communicate on `:9300` | Open TCP 9300 between all worker nodes in firewall |
| OOMKill on pods | JVM heap exceeds container memory | Ensure `-Xmx` = `-Xms` and both are < 50% of `resources.limits.memory` |
| Dashboard cannot connect | TLS verification failure | Set `opensearch.ssl.verificationMode: none` in dashboards config |
| Dashboard 403 errors | Wrong credentials in `opensearchAccount.secret` | Verify secret name and that it has `username` and `password` keys |
| `ImagePullBackOff` (busybox) | Docker Hub unavailable / rate limited | Configure RKE2 registry mirror in `/etc/rancher/rke2/registries.yaml` |
| Password rejected at startup | Password complexity too low | Use password with uppercase, lowercase, digit **and** special character |

---

## 7. Upgrading OpenSearch

Use `helm upgrade` with the updated values file. Always take a snapshot/backup before upgrading.

```bash
# Update Helm repo
helm repo update

# Check available new versions
helm search repo opensearch/opensearch --versions | head -5

# Upgrade (one minor version at a time)
helm upgrade opensearch opensearch/opensearch \
  --namespace opensearch \
  --values opensearch-values.yaml \
  --wait --timeout 15m

# Verify after upgrade
kubectl get pods -n opensearch
curl -s -k https://localhost:9200/_cluster/health?pretty -u admin:'YourStr0ng!Pass'
```

> **⚠️ WARNING:** Never upgrade more than one minor version at a time (e.g., `2.11 → 2.12`, then `2.12 → 2.13`). Rolling upgrades are handled automatically by the StatefulSet `RollingUpdate` strategy with `maxUnavailable=1`, ensuring the cluster maintains quorum throughout.

---

## 8. References

- [OpenSearch Helm Charts](https://opensearch-project.github.io/helm-charts/)
- [OpenSearch Documentation](https://opensearch.org/docs/latest/)
- [Rook Ceph Documentation](https://rook.io/docs/rook/latest-release/)
- [RKE2 Documentation](https://docs.rke2.io/)
- [OpenSearch Security Plugin](https://opensearch.org/docs/latest/security/)

---

## 9. Revision History

| Version | Date | Author | Description |
|---|---|---|---|
| 1.0 | 13 February 2026 | Ganesh Sharma| Initial release — RKE2 + Rook Ceph deployment |