# Runbook: Cluster-Wide DNS Failure Due to Inotify Exhaustion

**Severity:** CRITICAL
**Impact:** Total cluster failure - all pods unable to communicate
**MTTR:** 10-15 minutes (with this runbook)

---

## Symptoms

### Primary Indicators
- CoreDNS pods stuck in `0/1 Running` (not ready)
- kube-proxy pods in `CrashLoopBackOff`
- Error in kube-proxy logs: `too many open files`
- API server unreachable from pods (timeouts to `10.96.0.1:443`)
- Docker daemon restart failures with: `Failed to allocate directory watch: Too many open files`

### Secondary Effects (Cascading Failures)
- MetalLB controller and speakers failing
- nginx-based gateways (Mimir, Loki) unable to resolve DNS
- Mimir distributed components failing with gossip-ring DNS resolution errors
- All new pod creation fails
- Existing applications lose connectivity

### Detection Commands

```bash
# Check CoreDNS status (will show not ready)
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check kube-proxy status (will show CrashLoopBackOff)
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# View kube-proxy logs for "too many open files" error
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50

# Try to restart Docker (will fail if inotify exhausted)
sudo systemctl restart docker
```

---

## Root Cause

### What Happened
**Inotify watch exhaustion** - NOT file descriptor exhaustion (despite misleading error message).

### Technical Explanation

**Inotify watches** are Linux kernel resources used to monitor filesystem events. Docker uses them extensively to monitor:
- Container filesystem changes
- Volume mounts
- Configuration files
- Log files
- Network namespaces

**The Problem Chain:**
1. Debian default limits are too low for Docker + Kind workloads:
   - `fs.inotify.max_user_watches = 64,811` (too low)
   - `fs.inotify.max_user_instances = 128` (too low)

2. Kind cluster (4 nodes) + observability stack (27 pods) + long uptime (45+ hours) gradually exhausts all 64k watches

3. Once exhausted:
   - Docker cannot monitor containers → operations fail
   - kube-proxy cannot function → iptables rules break
   - API server service becomes unreachable
   - CoreDNS cannot initialize (requires API server)
   - **Cascading total cluster failure**

### Why the Misleading Error
The kernel syscall `inotify_add_watch()` returns `ENOSPC` (No space left on device), which applications interpret as "too many open files". The real issue is watch limit exhaustion.

---

## Diagnostic Steps

### Step 1: Check Inotify Limits (Current vs Used)

```bash
# Check current limits
sysctl fs.inotify.max_user_watches
sysctl fs.inotify.max_user_instances

# Check actual usage (requires root)
sudo find /proc/*/fd -lname anon_inode:inotify 2>/dev/null | wc -l
```

**Expected values for Docker + Kind:**
- Current system: 64,811 watches (BAD - too low)
- Recommended: 524,288 watches (GOOD - 8x increase)

### Step 2: Check File Descriptor Limits (Rule out FD exhaustion)

```bash
# Check limits
ulimit -n
cat /proc/sys/fs/file-max

# Check actual usage
cat /proc/sys/fs/file-nr
```

**Note:** If limits show "unlimited" or very high values (9,223,372,036,854,775,807), file descriptors are NOT the problem.

### Step 3: Check Docker Status

```bash
# Try to restart Docker (will fail if inotify exhausted)
sudo systemctl status docker
sudo systemctl restart docker  # Will show the actual error

# Check Docker logs
sudo journalctl -u docker --since "1 hour ago" | grep -i "inotify\|too many"
```

### Step 4: Check Kubernetes Components

```bash
# CoreDNS (should be not ready)
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# kube-proxy (should be crashing)
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=100

# API server connectivity test
kubectl get --raw /healthz
```

---

## Resolution

### Step 1: Increase Inotify Limits (Immediate Fix)

```bash
# Apply immediately (temporary - lost on reboot)
sudo sysctl -w fs.inotify.max_user_watches=524288
sudo sysctl -w fs.inotify.max_user_instances=512

# Verify
sysctl fs.inotify.max_user_watches
sysctl fs.inotify.max_user_instances
```

### Step 2: Make Limits Permanent (Survives Reboot)

```bash
# Add to sysctl.conf
echo "fs.inotify.max_user_watches = 524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 512" | sudo tee -a /etc/sysctl.conf

# Verify persistence
cat /etc/sysctl.conf | grep inotify
```

### Step 3: Restart Docker

```bash
# Restart Docker daemon (should succeed now)
sudo systemctl restart docker

# Verify Docker is running
sudo systemctl status docker
docker ps
```

### Step 4: Verify Kind Cluster Recovery

```bash
# Wait 30-60 seconds for components to restart, then check:

# Check kube-system pods (should all be Running)
kubectl get pods -n kube-system

# Check CoreDNS specifically (should be 2/2 Ready)
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

### Step 5: Verify Observability Stack

```bash
# Check all pods in observability namespace
kubectl get pods -n observability

# Check LoadBalancer services got IPs
kubectl get svc -n observability | grep LoadBalancer

# Test Grafana access
curl -I http://$(kubectl get svc grafana -n observability -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

---

## Verification Checklist

After resolution, verify these indicators:

- [ ] `sysctl fs.inotify.max_user_watches` shows 524288
- [ ] `sysctl fs.inotify.max_user_instances` shows 512
- [ ] `/etc/sysctl.conf` contains inotify settings (permanent)
- [ ] Docker service is active and running
- [ ] All kube-system pods are Running and Ready
- [ ] CoreDNS pods show 2/2 Ready
- [ ] kube-proxy pods are Running (4/4 for 3-worker cluster)
- [ ] API server responds to `kubectl get --raw /healthz`
- [ ] DNS resolution works: `kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default`
- [ ] MetalLB has assigned LoadBalancer IPs
- [ ] All observability pods are Running
- [ ] Grafana UI is accessible

---

## Prevention

### Permanent Configuration

The fix applied in Step 2 (adding to `/etc/sysctl.conf`) prevents recurrence by:
- Increasing watch limit 8x (64k → 524k)
- Surviving system reboots
- Applied automatically at boot time

### Monitoring (Future Enhancement)

Consider implementing:

```bash
# Add to a monitoring script/cronjob
# Alert if inotify usage exceeds 80% of limit

WATCH_LIMIT=$(sysctl -n fs.inotify.max_user_watches)
WATCH_USED=$(sudo find /proc/*/fd -lname anon_inode:inotify 2>/dev/null | wc -l)
USAGE_PCT=$((WATCH_USED * 100 / WATCH_LIMIT))

if [ $USAGE_PCT -gt 80 ]; then
  echo "WARNING: inotify watch usage at ${USAGE_PCT}%"
fi
```

### Best Practices

1. **Set inotify limits BEFORE cluster creation** on new hosts
2. **Document in host setup procedures** for reproducibility
3. **Monitor long-running Kind clusters** (not designed for 45+ hour uptimes)
4. **Consider periodic cluster recreation** for lab environments

---

## Related Issues

### Grafana Init Container Failure (Secondary Issue)

**Symptom:** `Init:CrashLoopBackOff` with permission errors
**Cause:** PVC corruption during cluster failure (bad ownership/permissions)
**Resolution:** Delete and recreate Grafana deployment + PVC

```bash
kubectl delete deployment grafana -n observability
kubectl delete pvc grafana -n observability
helm upgrade --install grafana grafana/grafana \
  -f observability-stack/charts/grafana/values-grafana.yaml \
  -n observability
```

---

## Key Learnings

1. **"Too many open files" is ambiguous** - Could mean FDs OR inotify watches
2. **Check multiple metrics** - Don't assume first error message is accurate
3. **Kind clusters have limits** - Default Debian inotify settings inadequate for container-heavy workloads
4. **Cascading failures** - Single host-level issue can cause total cluster failure
5. **Order matters** - Fix host → fix Docker → fix kube-proxy → DNS recovers → everything else follows

---

## References

- Linux inotify documentation: `man inotify`
- Docker inotify requirements: https://docs.docker.com/engine/install/linux-postinstall/#adjust-inotify-limits
- Kubernetes DNS debugging: https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/

---

**Last Updated:** 2025-11-05
**Validated:** Yes - Successfully recovered sre-lab cluster
**Author:** SRE Lab Troubleshooting Session
