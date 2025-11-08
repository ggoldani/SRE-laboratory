# SRE Laboratory

Operational documentation and runbooks for the SRE Lab environment - a centralized knowledge base for deploying and operating a complete Kubernetes-based observability stack.

---

## Purpose

This repository serves as the **documentation hub** for the SRE Lab project, providing:

- **Runbooks:** Step-by-step procedures for diagnosing and resolving incidents
- **Troubleshooting guides:** Systematic approaches to common problems
- **Preventive procedures:** Daily/weekly health checks to catch issues early
- **Architecture overview:** How all components fit together
- **Installation guidance:** Coordinated setup of all infrastructure pieces

**Why a separate documentation repo?** Operational knowledge spans across infrastructure, observability, and applications. This repo provides the "glue" documentation that ties everything together while keeping each component's technical docs in their respective repositories.

---

## SRE Lab Architecture

The SRE Lab consists of **three separate repositories**, each focused on a specific domain:

### 1. Infrastructure Layer - [infra-cluster-kind](https://github.com/ggoldani/infra-cluster-kind)

**Purpose:** Kubernetes cluster foundation

**What it provides:**
- Kind cluster configuration (1 control-plane + 3 workers)
- MetalLB LoadBalancer implementation
- Network configuration (172.18.0.0/16 with MetalLB pool 172.18.255.200-250)
- Automated installation scripts

**Key technologies:** Kind, MetalLB, Docker

**Clone:**
```bash
git clone https://github.com/ggoldani/infra-cluster-kind.git
```

---

### 2. Observability Platform - [observability-stack](https://github.com/ggoldani/observability-stack)

**Purpose:** Complete telemetry collection, storage, and visualization

**What it provides:**
- **Grafana Mimir** - Distributed Prometheus-compatible metrics storage (MinIO backend)
- **Grafana Loki** - Log aggregation and querying
- **Grafana Tempo** - Distributed tracing backend
- **Grafana** - Unified dashboards and visualization
- **OpenTelemetry Collector** - Telemetry pipeline (OTLP receivers → backends)

**Key technologies:** Grafana stack, OpenTelemetry, Helm, MinIO

**Clone:**
```bash
git clone https://github.com/ggoldani/observability-stack.git
```

---

### 3. Sample Application - [app-flask-otel](https://github.com/ggoldani/app-flask-otel)

**Purpose:** Instrumented application demonstrating full observability

**What it provides:**
- Flask web application with multiple endpoints
- OpenTelemetry SDK instrumentation (traces, metrics, logs)
- OTLP export to OpenTelemetry Collector
- Kubernetes deployment manifests

**Key technologies:** Python, Flask, OpenTelemetry SDK, Kubernetes

**Clone:**
```bash
git clone https://github.com/ggoldani/app-flask-otel.git
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│  Infrastructure (infra-cluster-kind)                     │
│  - Kind cluster: 1 control-plane + 3 workers            │
│  - MetalLB: LoadBalancer IPs (172.18.255.200-250)       │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│  Observability Platform (observability-stack)            │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Application (app-flask-otel)                    │  │
│  │  - Flask with OpenTelemetry SDK                  │  │
│  └───────────────────┬──────────────────────────────┘  │
│                      │ OTLP (traces, metrics, logs)     │
│                      ▼                                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  OpenTelemetry Collector                         │  │
│  │  - OTLP receivers (gRPC 4317, HTTP 4318)        │  │
│  └──────────────────┬───────┬───────┬───────────────┘  │
│                     │       │       │                   │
│       ┌─────────────┘       │       └──────────────┐   │
│       ▼                     ▼                      ▼   │
│  ┌────────┐          ┌──────────┐           ┌──────────┐
│  │ Loki   │          │  Mimir   │           │  Tempo   │
│  │ (Logs) │          │(Metrics) │           │ (Traces) │
│  └────────┘          └──────────┘           └──────────┘
│       └───────────────────┬───────────────────────┘   │
│                           ▼                           │
│                    ┌───────────────┐                  │
│                    │    Grafana    │                  │
│                    │  (Dashboards) │                  │
│                    └───────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

---

## Quick Start

### Prerequisites

- Docker 28.5.1+
- kubectl 1.31.0+
- kind 0.23.0+
- Helm 3.19.0+
- jq (for scripts)

### Installation Order

The repositories must be installed in sequence due to dependencies:

**1. Infrastructure first (REQUIRED)**
```bash
git clone https://github.com/ggoldani/infra-cluster-kind.git
cd infra-cluster-kind
./install.sh
```

**2. Observability stack (requires infrastructure)**
```bash
git clone https://github.com/ggoldani/observability-stack.git
cd observability-stack
./install.sh
```

**3. Sample application (optional, requires both above)**
```bash
git clone https://github.com/ggoldani/app-flask-otel.git
cd app-flask-otel
kubectl apply -f k8s/
```

**4. Access Grafana**
```bash
kubectl get svc grafana -n observability -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Open http://<IP> in browser (admin/admin)
```

---

## Critical Pre-Installation Step

⚠️ **REQUIRED:** Set inotify limits BEFORE creating the cluster to prevent DNS failures:

```bash
sudo sysctl -w fs.inotify.max_user_watches=524288
sudo sysctl -w fs.inotify.max_user_instances=512
echo "fs.inotify.max_user_watches = 524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 512" | sudo tee -a /etc/sysctl.conf
```

**Why:** Default Debian limits (64k) cause cluster-wide failures after ~45 hours. See runbook below for details.

---

## Runbook Catalog

### Critical Severity

| Runbook | Impact | MTTR | When to Use |
|---------|--------|------|-------------|
| [Cluster DNS Failure (Inotify Exhaustion)](./docs/runbooks/cluster-dns-failure-inotify-exhaustion.md) | Total cluster failure | 10-15 min | CoreDNS not ready, kube-proxy crashing, DNS resolution failures |

**Symptoms of inotify exhaustion:**
- CoreDNS pods stuck in `0/1 Running`
- kube-proxy in `CrashLoopBackOff` with "too many open files" errors
- Docker restart failures
- All pods unable to communicate

**Prevention:** Set inotify limits BEFORE creating cluster (see above)

### High Severity

| Runbook | Impact | MTTR | When to Use |
|---------|--------|------|-------------|
| [Complete Cluster Reset](./docs/runbooks/cluster-reset-procedure.md) | Environment rebuild | 15-20 min | Unrecoverable cluster state, learning reset, major version upgrades |

**When to reset:**
- Cluster in broken state beyond repair
- Want to start fresh with clean slate
- Testing installation procedures
- Practicing disaster recovery

### Informational

| Runbook | Impact | MTTR | When to Use |
|---------|--------|------|-------------|
| [Daily Health Check](./docs/runbooks/cluster-health-check.md) | Preventive | 2-3 min | Daily verification, before deployments, after incidents |

**Regular health checks catch:**
- Slowly degrading resources
- Pods approaching restart thresholds
- LoadBalancer IP assignment issues
- Inotify usage trends (before exhaustion)

---

## Quick Reference

### Common Diagnostic Commands

```bash
# Cluster-level health
kubectl get nodes -o wide                               # Node status
kubectl get pods -A | grep -v Running                  # Non-running pods
kubectl get events -A --sort-by='.lastTimestamp' | tail -20  # Recent events

# Observability stack
kubectl get pods -n observability -o wide               # Pod status
kubectl get svc -n observability                        # Service endpoints
kubectl get pvc -n observability                        # Storage usage
helm list -n observability                              # Helm releases

# CoreDNS and networking
kubectl get pods -n kube-system -l k8s-app=kube-dns    # DNS pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy  # Proxy pods
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50  # DNS logs

# MetalLB LoadBalancer
kubectl get pods -n metallb-system                      # MetalLB status
kubectl get ipaddresspool -n metallb-system             # IP pool config
kubectl get svc -A | grep LoadBalancer                  # All LoadBalancer services

# System resources (requires metrics-server)
kubectl top nodes                                       # Node CPU/memory
kubectl top pods -n observability                       # Pod resources

# Inotify usage (critical for Kind)
for foo in /proc/*/fd/*; do readlink -f $foo; done | grep inotify | wc -l
sysctl fs.inotify.max_user_watches                     # Check limit
```

### Emergency Response Order

When the cluster is malfunctioning:

1. **Assess scope:** Is it one pod, one component, or cluster-wide?
   ```bash
   kubectl get pods -A | grep -v Running
   kubectl get nodes
   ```

2. **Check CoreDNS first:** Most failures start with DNS
   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   ```

3. **Review recent events:** What changed?
   ```bash
   kubectl get events -A --sort-by='.lastTimestamp' | tail -20
   ```

4. **Check inotify usage:** Is it exhaustion?
   ```bash
   for foo in /proc/*/fd/*; do readlink -f $foo; done | grep inotify | wc -l
   ```

5. **Consult runbooks:** Match symptoms to documented procedures

---

## Using Runbooks Effectively

### Runbook Structure

Each runbook follows a standard format:

1. **Metadata:** Severity, impact, MTTR estimate
2. **Symptoms:** Observable indicators of the problem
3. **Detection commands:** How to confirm the issue
4. **Root cause:** Technical explanation of what's happening
5. **Resolution steps:** Step-by-step fix procedure
6. **Prevention:** How to avoid the issue in the future
7. **Verification:** How to confirm the fix worked
8. **Related resources:** Links to relevant documentation

### When to Use vs. Skip Runbooks

**Use runbooks when:**
- You've never seen the issue before (learn the methodology)
- The system is in a critical state (follow proven steps)
- Multiple people are responding (shared procedure)
- Documenting for post-mortem (reference timeline)

**Skip runbooks when:**
- You're very familiar with the issue (muscle memory)
- Quick one-liner fix (but note it for future runbook)
- Experimenting with different approaches (learning mode)

### Improving Runbooks

As you use these procedures:
- **Note gaps:** Missing steps, unclear instructions
- **Add context:** Why does this command work?
- **Update timings:** Adjust MTTR based on actual experience
- **Add variations:** Document different symptoms of same root cause
- **Create new runbooks:** Document new issues you encounter

This is a learning environment - improving the runbooks IS part of the learning.

---

## Known Issues and Gotchas

### 1. Inotify Exhaustion (CRITICAL)

**Issue:** Default Debian inotify limits (64k watches) insufficient for Docker + Kind + multiple pods.

**Impact:** Total cluster failure after ~45 hours of uptime.

**Prevention:**
```bash
sudo sysctl -w fs.inotify.max_user_watches=524288
sudo sysctl -w fs.inotify.max_user_instances=512
echo "fs.inotify.max_user_watches = 524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 512" | sudo tee -a /etc/sysctl.conf
```

**Runbook:** [cluster-dns-failure-inotify-exhaustion.md](./docs/runbooks/cluster-dns-failure-inotify-exhaustion.md)

### 2. Mimir Upgrade Webhook Conflicts

**Issue:** `helm upgrade mimir` fails with webhook validation errors from `mimir-rollout-operator`.

**Workaround:**
```bash
helm uninstall mimir -n observability
helm install mimir grafana/mimir-distributed \
  -f observability-stack/charts/mimir/values-mimir.yaml \
  -n observability
```

**Why:** Mimir's custom rollout operator doesn't handle certain upgrade paths gracefully.

**Note:** Assumes you're in the `observability-stack` repository directory.

### 3. Pending LoadBalancer IPs

**Issue:** Services stuck with `<pending>` EXTERNAL-IP after creation.

**Common causes:**
- MetalLB pods not running
- IP pool exhausted (only 51 IPs available)
- L2 advertisement misconfigured

**Diagnosis:**
```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl logs -n metallb-system -l app=metallb
```

**Solution:** Review the `infra-cluster-kind` repository README for MetalLB troubleshooting.

### 4. Helm Installation Hangs

**Issue:** `helm install` waits indefinitely with `--wait` flag.

**Cause:** Pods not reaching Ready state (image pull, resource constraints, misconfig).

**What to do:**
- Let it run (first-time image pulls can take 10+ minutes)
- Or Ctrl+C and debug with `kubectl get pods -n observability -w`

---

## Operational Best Practices

### Daily Operations

**Morning routine:**
1. Run [health check runbook](./docs/runbooks/cluster-health-check.md)
2. Review any degraded resources
3. Check inotify usage trends

**Before deployments:**
1. Verify cluster health
2. Note current resource usage
3. Have rollback plan ready

**After incidents:**
1. Verify resolution with health check
2. Document what happened (post-mortem)
3. Update runbooks with learnings

### Preventive Maintenance

**Weekly:**
- Full health check with detailed review
- Check inotify usage trends
- Review disk space on worker nodes
- Verify all LoadBalancer IPs assigned

**Monthly:**
- Consider cluster reset for clean slate
- Update Helm charts to latest versions
- Review and improve runbooks
- Practice incident response procedures

**When to reset:**
- If cluster has been running 45+ days (avoid inotify risks)
- After major configuration changes (test clean install)
- When practicing disaster recovery

---

## Learning Paths

### Beginner: Understanding the Stack

1. Clone and deploy all three repositories in order
2. Run daily health check and understand each command
3. Intentionally trigger the inotify failure (controlled learning)
4. Follow the recovery runbook step-by-step
5. Perform a complete cluster reset

**Goals:**
- Understand Kubernetes cluster architecture
- Learn kubectl diagnostic commands
- Practice incident response methodology
- Build confidence in cluster management

### Intermediate: Observability Deep Dive

1. Deploy applications sending telemetry to OpenTelemetry Collector
2. Create Grafana dashboards querying all three backends
3. Simulate application failures and trace in Grafana
4. Practice correlating metrics → logs → traces
5. Write custom runbooks for application issues

**Goals:**
- Master observability stack components
- Learn PromQL, LogQL, TraceQL
- Understand distributed tracing concepts
- Build operational dashboards

### Advanced: SRE Practices

1. Implement SLOs (Service Level Objectives) in Grafana
2. Set up alerting based on SLO violations
3. Conduct chaos engineering experiments
4. Write post-mortems for simulated incidents
5. Optimize resource allocation based on metrics

**Goals:**
- Apply SRE principles (error budgets, SLIs/SLOs)
- Practice blameless post-mortems
- Learn capacity planning
- Develop on-call procedures

---

## Contributing

### Adding New Runbooks

When you encounter and resolve a new issue:

1. **Document while fresh:** Write immediately after resolution
2. **Follow template:** Use existing runbooks as reference
3. **Include context:** Why does the fix work? What's the root cause?
4. **Test procedure:** Have someone else follow your runbook
5. **Link related resources:** Connect to relevant documentation

### Runbook Template

```markdown
# Runbook: [Title]

**Severity:** [CRITICAL/HIGH/MEDIUM/LOW]
**Impact:** [Brief description]
**MTTR:** [Estimated time]

---

## Symptoms

### Primary Indicators
- [Observable symptom 1]
- [Observable symptom 2]

### Detection Commands
```bash
# Command to verify symptom
kubectl get ...
```

---

## Root Cause

[Technical explanation of what's happening]

---

## Resolution

### Step 1: [Action]
```bash
# Commands
```

**Expected output:** [What success looks like]

### Step 2: [Action]
...

---

## Verification

```bash
# Commands to verify fix
```

---

## Prevention

[How to avoid this issue in the future]

---

## Related Resources

- [Link to relevant docs]
- [infra-cluster-kind repository](https://github.com/ggoldani/infra-cluster-kind)
- [observability-stack repository](https://github.com/ggoldani/observability-stack)
- [app-flask-otel repository](https://github.com/ggoldani/app-flask-otel)
```

---

## Technology Stack

Understanding what each component does aids troubleshooting:

| Component | Repository | Purpose | Failure Impact |
|-----------|------------|---------|----------------|
| **Docker** | (Host system) | Container runtime | Cluster won't start or will crash |
| **Kind** | infra-cluster-kind | Kubernetes cluster | No cluster, can't deploy anything |
| **CoreDNS** | (Kubernetes core) | Service discovery | Pods can't find each other by name |
| **kube-proxy** | (Kubernetes core) | Network routing | Inter-pod communication breaks |
| **MetalLB** | infra-cluster-kind | LoadBalancer IPs | Can't access services externally |
| **Mimir** | observability-stack | Metrics storage | No metrics history, dashboards empty |
| **Loki** | observability-stack | Log aggregation | Can't search logs, no log correlation |
| **Tempo** | observability-stack | Trace storage | Can't see distributed traces |
| **Grafana** | observability-stack | Visualization | Can still query backends directly via API |
| **OTel Collector** | observability-stack | Telemetry pipeline | Applications can send directly to backends |

**Critical path:** Docker → Kind → CoreDNS → Everything else

**Nice to have:** Grafana (can query APIs directly), OTel Collector (apps can send to backends)

---

## Resources

### Project Repositories

- **infra-cluster-kind:** https://github.com/ggoldani/infra-cluster-kind
- **observability-stack:** https://github.com/ggoldani/observability-stack
- **app-flask-otel:** https://github.com/ggoldani/app-flask-otel
- **SRE-laboratory (this repo):** https://github.com/ggoldani/SRE-laboratory

### Official Documentation

- **Kubernetes:** https://kubernetes.io/docs/
- **Kind:** https://kind.sigs.k8s.io/
- **MetalLB:** https://metallb.universe.tf/
- **Grafana Observability:** https://grafana.com/docs/
- **OpenTelemetry:** https://opentelemetry.io/docs/

### SRE Resources

- **Google SRE Book:** https://sre.google/books/
- **Site Reliability Workbook:** https://sre.google/workbook/table-of-contents/
- **SRE Weekly:** https://sreweekly.com/
- **Runbook examples:** https://github.com/topics/runbook

### Learning Platforms

- **Kubernetes tutorials:** https://kubernetes.io/docs/tutorials/
- **Grafana tutorials:** https://grafana.com/tutorials/
- **OpenTelemetry demos:** https://opentelemetry.io/docs/demo/

---

## Incident Response Philosophy

These runbooks follow **blameless post-mortem** principles:

1. **Focus on systems, not people:** Incidents reveal system weaknesses, not human failures
2. **Document everything:** Timeline, actions, outcomes, learnings
3. **Share learnings:** Update runbooks, share with others
4. **Improve systems:** Each incident is an opportunity to make the system more resilient

**Remember:** Breaking things in this lab environment is GOOD. That's how you learn. Document what you learn and improve the runbooks for next time.

---

## Why Separate Repositories?

This project is intentionally split into separate repositories:

- **Modularity:** Users can clone only what they need
- **Role separation:** Infrastructure, observability, and application teams work independently
- **Learning:** Demonstrates real-world multi-repo architecture
- **Reusability:** Each component can be used in other projects

**Example use cases:**
- "I just need a local Kubernetes cluster" → Clone only `infra-cluster-kind`
- "I already have a cluster, need observability" → Clone only `observability-stack`
- "I want to learn OpenTelemetry instrumentation" → Clone only `app-flask-otel`
- "I want the complete lab" → Clone all three + this documentation repo

---

**Repository:** https://github.com/ggoldani/SRE-laboratory
**Status:** Active Development
**Last Updated:** 2025-11-08
