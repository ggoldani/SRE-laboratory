# Runbook: Daily Cluster Health Check

**Severity:** INFORMATIONAL (Preventive)
**Purpose:** Proactive monitoring to detect issues before they escalate
**Time Required:** 2-3 minutes

---

## Overview

This runbook provides a quick daily health check for the SRE lab environment. Use this to verify cluster health before starting work or after system changes.

**When to use:**
- Daily/weekly routine verification
- After deployments or configuration changes
- Before starting lab work sessions
- After system reboots

---

## Health Check Procedure

### 1. Check Nodes

All nodes should be Ready.

```bash
kubectl get nodes
```

**Expected:**
- 4 nodes total (1 control-plane + 3 workers)
- All STATUS = `Ready`

---

### 2. Check Core Kubernetes Components

#### CoreDNS

DNS must be working for cluster networking.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

**Expected:**
- 2 pods
- Both `Running` and `1/1 Ready`

#### kube-proxy

Network proxy must run on every node.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

**Expected:**
- 4 pods (one per node)
- All `Running` and `1/1 Ready`

---

### 3. Check MetalLB

MetalLB provides LoadBalancer IPs.

```bash
kubectl get pods -n metallb-system
```

**Expected:**
- 5 pods total: 1 controller + 4 speakers
- All `Running`

**Verify configuration:**

```bash
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

**Expected:**
- IPAddressPool named `first-pool` with range `172.18.255.200-172.18.255.250`
- L2Advertisement named `example-l2adv`

---

### 4. Check Observability Stack

#### All Pods Status

```bash
kubectl get pods -n observability
```

**Expected:**
- Approximately 30-35 pods total
- All in `Running` state
- No `CrashLoopBackOff` or `Error` states

#### Key Components

**Grafana:**
```bash
kubectl get pods -n observability -l app.kubernetes.io/name=grafana
```
Expected: 1 pod Running

**Mimir:**
```bash
kubectl get pods -n observability -l app.kubernetes.io/name=mimir
```
Expected: ~26 pods Running (distributors, ingesters, queriers, compactor, etc.)

**Loki:**
```bash
kubectl get pods -n observability -l app.kubernetes.io/name=loki
```
Expected: 2-5 pods Running (loki + gateway + canaries)

**Tempo:**
```bash
kubectl get pods -n observability -l app.kubernetes.io/name=tempo
```
Expected: 1 pod Running

**OpenTelemetry Collector:**
```bash
kubectl get pods -n observability -l app.kubernetes.io/name=opentelemetry-collector
```
Expected: 1 pod Running

**MinIO (Mimir storage):**
```bash
kubectl get pods -n observability -l app=minio
```
Expected: 1 pod Running

---

### 5. Check LoadBalancer Services

All services must have assigned IPs (not `<pending>`).

```bash
kubectl get svc -n observability | grep LoadBalancer
```

**Expected:**
- 5 services with EXTERNAL-IP assigned:
  - `grafana` - 172.18.255.203
  - `loki-gateway` - 172.18.255.201
  - `mimir-nginx` - 172.18.255.200
  - `otel-collector-opentelemetry-collector` - 172.18.255.204
  - `tempo` - 172.18.255.202

**None should show `<pending>`**

---

### 6. Check Storage

All PVCs must be bound.

```bash
kubectl get pvc -n observability
```

**Expected:**
- Approximately 12 PVCs
- All STATUS = `Bound`

---

### 7. Check Host System Resources

#### inotify Limits (CRITICAL)

These must be set correctly or cluster will fail.

```bash
sysctl fs.inotify.max_user_watches
sysctl fs.inotify.max_user_instances
```

**Expected:**
```
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

**If showing 64811 (default):** Apply fix immediately from [cluster-dns-failure runbook](./cluster-dns-failure-inotify-exhaustion.md#resolution)

#### Disk Space

```bash
df -h /
```

**Expected:**
- Root filesystem usage < 80%

**Warning:** > 80% = plan cleanup
**Critical:** > 90% = immediate cleanup needed

---

### 8. Connectivity Tests

#### Test Internal DNS

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

**Expected:** Successful resolution to `10.96.0.1`

#### Test Grafana Access

```bash
kubectl get svc grafana -n observability -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Copy the IP and access in browser: `http://<IP>`

**Expected:** Grafana login page (credentials: admin/admin)

---

## Quick Reference: Expected Healthy State

| Component | Expected Count | Status |
|-----------|----------------|--------|
| Nodes | 4 | All Ready |
| CoreDNS | 2 | Running, 1/1 Ready |
| kube-proxy | 4 | Running, 1/1 Ready |
| MetalLB | 5 | Running (1 controller + 4 speakers) |
| Observability pods | 30-35 | All Running |
| LoadBalancer services | 5 | All with EXTERNAL-IP |
| PVCs | ~12 | All Bound |
| inotify watches | 524288 | Must be set |
| Disk usage | < 80% | Root filesystem |

---

## Troubleshooting Failed Checks

| Failed Check | Likely Cause | Next Steps |
|--------------|--------------|------------|
| CoreDNS not ready | Inotify exhaustion or API server down | See [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md) |
| kube-proxy crashing | Inotify exhaustion | See [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md) |
| MetalLB not running | Installation issue or misconfiguration | Check installation: `kubectl logs -n metallb-system -l app=metallb` |
| LoadBalancer `<pending>` | MetalLB down or IP pool exhausted | Verify MetalLB running and IPAddressPool configured |
| Pods CrashLoopBackOff | Various (check logs) | `kubectl describe pod <name> -n observability`<br>`kubectl logs <name> -n observability` |
| PVC Pending | No storage available | Check storage class: `kubectl get storageclass` |
| inotify limits wrong | Not configured | **CRITICAL:** Fix immediately - [see prevention](./cluster-dns-failure-inotify-exhaustion.md#prevention) |
| Disk > 90% | Storage exhaustion | Clean Docker: `docker system prune -a`<br>Check large files: `du -sh /var/lib/docker/*` |

---

## Common Diagnostic Commands

### View Recent Events
```bash
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
```

### Check Pod Details
```bash
kubectl describe pod <pod-name> -n <namespace>
```

### View Pod Logs
```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous  # Previous container instance
```

### Check Resource Usage
```bash
kubectl top nodes  # Requires metrics-server (not installed by default)
```

### Check Docker
```bash
docker ps | grep kindest
docker stats --no-stream
```

---

**Last Updated:** 2025-11-05
**Estimated Time:** 2-3 minutes
**Frequency:** Daily or before each work session
