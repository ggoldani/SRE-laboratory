# Runbook: Complete Cluster Reset Procedure

**Severity:** HIGH (Destructive operation)
**Use Case:** When cluster is too corrupted to fix incrementally
**Total Time:** 15-20 minutes
**Data Loss:** YES - All workloads and data will be destroyed

---

## When to Use This Runbook

### ✅ Good Reasons to Reset
- Multiple cascading failures that resist individual fixes
- Corrupted cluster state (etcd issues, broken networking)
- Faster than debugging (time-boxed troubleshooting exceeded)
- Testing infrastructure automation
- Lab environment where data is not critical
- Starting fresh after experiments

### ❌ Bad Reasons to Reset
- Single pod failure (fix the pod instead)
- Production environment (NEVER reset production)
- Haven't tried targeted troubleshooting first
- Data loss is unacceptable

---

## Pre-Reset Considerations

### Before Destroying Everything

**Try fixing individual components first:**
```bash
# Restart a specific deployment
kubectl rollout restart deployment <name> -n <namespace>

# Reinstall a single Helm chart
helm uninstall <component> -n observability
helm install <component> ... -n observability
```

**Document current state (optional):**
```bash
kubectl get all --all-namespaces > cluster-state-before-reset.txt
kubectl get events --all-namespaces --sort-by='.lastTimestamp' > events-before-reset.txt
```

---

## Reset Procedure

### Phase 1: Destroy Cluster

#### Delete Kind Cluster

```bash
kind delete cluster --name sre-lab
```

**Expected output:**
```
Deleting cluster "sre-lab" ...
Deleted nodes: ["sre-lab-control-plane" "sre-lab-worker" "sre-lab-worker2" "sre-lab-worker3"]
```

#### Verify Cleanup

```bash
# Verify no clusters remain
kind get clusters

# Verify containers are gone
docker ps -a | grep kindest

# Verify Kind network is gone
docker network ls | grep kind
```

**Expected:** No output from any of these commands

#### Optional: Deep Cleanup

Only if you want to remove all traces:

```bash
docker container prune -f
docker network prune -f
docker volume prune -f
```

---

### Phase 2: Recreate Infrastructure

#### Create Kind Cluster

```bash
# Navigate to the infra-cluster-kind directory
cd infra-cluster-kind

kind create cluster --config cluster/kind-cluster-3w.yaml
```

**Expected output:**
```
Creating cluster "sre-lab" ...
 ✓ Ensuring node image (kindest/node:v1.31.0)
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-sre-lab"
```

**Wait 30-60 seconds**, then verify:

```bash
kubectl get nodes
```

**Expected:** 4 nodes, all `Ready`

#### Install MetalLB

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

**Wait 60 seconds** for MetalLB pods to start, then:

```bash
# Apply IP pool configuration
kubectl apply -f metallb/
```

**Verify:**

```bash
kubectl get pods -n metallb-system
```

**Expected:** 1 controller + 4 speakers, all `Running`

---

### Phase 3: Deploy Observability Stack

#### Setup

```bash
# Navigate to the observability-stack directory
cd observability-stack

# Add Helm repos
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Create namespace
kubectl create namespace observability
```

#### Install Mimir (Metrics)

```bash
helm upgrade --install mimir grafana/mimir-distributed \
  -f charts/mimir/values-mimir.yaml \
  -n observability
```

**Wait 2-3 minutes**, then verify:

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=mimir
```

**Expected:** ~26 pods, all `Running`

#### Install Loki (Logs)

```bash
helm upgrade --install loki grafana/loki \
  -f charts/loki/values-loki.yaml \
  -n observability
```

**Wait 30-60 seconds**, then verify:

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=loki
```

**Expected:** 2 loki pods + gateway + canaries, all `Running`

#### Install Tempo (Traces)

```bash
helm upgrade --install tempo grafana/tempo \
  -f charts/tempo/values-tempo.yaml \
  -n observability
```

**Wait 30 seconds**, then verify:

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=tempo
```

**Expected:** 1 pod `Running`

#### Install Grafana (Dashboards)

```bash
helm upgrade --install grafana grafana/grafana \
  -f charts/grafana/values-grafana.yaml \
  -n observability
```

**Wait 30 seconds**, then verify:

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=grafana
```

**Expected:** 1 pod `Running`, `1/1 Ready`

#### Install OpenTelemetry Collector

```bash
helm upgrade --install otel-collector open-telemetry/opentelemetry-collector \
  -f charts/otel-collector/values-otel-collector.yaml \
  -n observability
```

**Wait 30 seconds**, then verify:

```bash
kubectl get pods -n observability -l app.kubernetes.io/name=opentelemetry-collector
```

**Expected:** 1 pod `Running`

---

### Phase 4: Verification

#### Check All Pods

```bash
kubectl get pods -n observability
```

**Expected:** 30-35 pods, all `Running`

#### Check LoadBalancer Services

```bash
kubectl get svc -n observability | grep LoadBalancer
```

**Expected:** 5 services, all with EXTERNAL-IP assigned (not `<pending>`)

#### Check Storage

```bash
kubectl get pvc -n observability
```

**Expected:** ~12 PVCs, all `Bound`

#### Test Grafana Access

```bash
kubectl get svc grafana -n observability -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Copy the IP and open in browser: `http://<IP>`

**Expected:** Grafana login page
**Credentials:** admin / admin

---

## Post-Reset Critical Task

### ⚠️ CRITICAL: Set inotify Limits

**This must be done or cluster will fail within 45 hours!**

Check current limits:

```bash
sysctl fs.inotify.max_user_watches
```

If shows `64811` (default), apply fix immediately:

```bash
# Apply immediately
sudo sysctl -w fs.inotify.max_user_watches=524288
sudo sysctl -w fs.inotify.max_user_instances=512

# Make permanent
echo "fs.inotify.max_user_watches = 524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 512" | sudo tee -a /etc/sysctl.conf
```

Verify:

```bash
sysctl fs.inotify.max_user_watches
```

**Expected:** `524288`

See [cluster-dns-failure runbook](./cluster-dns-failure-inotify-exhaustion.md) for details.

---

## Quick Reset Checklist

Use this for fast reference during reset:

```
Phase 1: Destroy
□ kind delete cluster --name sre-lab
□ Verify cleanup (kind get clusters = empty)

Phase 2: Infrastructure
□ kind create cluster --config cluster/kind-cluster-3w.yaml
□ kubectl get nodes (wait for 4 Ready)
□ kubectl apply -f <metallb-manifest-url>
□ Wait 60s
□ kubectl apply -f metallb/
□ Verify MetalLB running

Phase 3: Observability
□ cd observability-stack
□ helm repo add + update
□ kubectl create namespace observability
□ helm install mimir (wait 2-3 min)
□ helm install loki (wait 1 min)
□ helm install tempo (wait 30s)
□ helm install grafana (wait 30s)
□ helm install otel-collector (wait 30s)

Phase 4: Verify
□ kubectl get pods -n observability (30-35 Running)
□ kubectl get svc -n observability (5 LoadBalancers with IPs)
□ kubectl get pvc -n observability (12 Bound)
□ Access Grafana in browser

Post-Reset
□ Set inotify limits (CRITICAL!)
□ Verify limits: sysctl fs.inotify.max_user_watches = 524288
```

---

## Troubleshooting Reset Issues

### Cluster Creation Fails

**Check:**
```bash
sudo systemctl status docker
docker ps
df -h
```

**Common causes:** Docker not running, insufficient disk space

### MetalLB Pods Not Starting

**Fix:**
```bash
kubectl delete -f metallb/
sleep 30
kubectl apply -f metallb/
```

### LoadBalancer Services Stuck `<pending>`

**Check MetalLB:**
```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl logs -n metallb-system -l app=metallb
```

### Helm Install Hangs or Fails

**Check events and pod status:**
```bash
kubectl get events -n observability --sort-by='.lastTimestamp' | tail -20
kubectl get pods -n observability
kubectl describe pod <failing-pod> -n observability
```

**Common causes:** Image pull failures, insufficient resources, storage issues

---

## Alternative: Observability-Only Reset

If cluster infrastructure is healthy but observability stack is broken:

```bash
# Faster: Only reset observability (keeps cluster)
helm uninstall mimir loki tempo grafana otel-collector -n observability
kubectl delete namespace observability

# Wait 30 seconds
sleep 30

# Recreate
kubectl create namespace observability
# Then reinstall all Helm charts (see Phase 3)
```

**Time saved:** ~5 minutes (no cluster recreation)

---

## Time Estimates

| Phase | Duration |
|-------|----------|
| Destroy cluster | 1 minute |
| Recreate infrastructure | 3-4 minutes |
| Deploy observability | 8-10 minutes |
| Verification | 2 minutes |
| **Total** | **15-20 minutes** |

---

## Related Runbooks

- [Cluster Health Check](./cluster-health-check.md) - Verify cluster after reset
- [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md) - inotify limits (critical post-reset)

---

**Last Updated:** 2025-11-05
**Validated:** Yes
**Total Time:** 15-20 minutes
**Data Loss:** Complete (lab environment only)
