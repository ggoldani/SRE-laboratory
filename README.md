# SRE Laboratory

A comprehensive documentation and runbook repository for the SRE Lab environment, providing operational guides and troubleshooting procedures for a complete Kubernetes-based observability stack.

---

## Overview

This repository serves as the **central documentation hub** for the SRE Lab project, which consists of three main component repositories:

1. **[infra-cluster-kind](https://github.com/ggoldani/infra-cluster-kind)** - Kubernetes infrastructure
2. **[observability-stack](https://github.com/ggoldani/observability-stack)** - Grafana observability platform
3. **[app-flask-otel](https://github.com/ggoldani/app-flask-otel)** - Sample instrumented application

This repository contains **runbooks**, **operational procedures**, and **troubleshooting guides** that span across all three components.

---

## What's Inside

### 📚 Documentation

- **[Runbooks](./docs/runbooks/)** - Operational procedures for incident response and maintenance
  - [Cluster DNS Failure (Inotify Exhaustion)](./docs/runbooks/cluster-dns-failure-inotify-exhaustion.md) - Critical cluster-wide failure recovery
  - [Daily Cluster Health Check](./docs/runbooks/cluster-health-check.md) - Preventive verification procedures
  - [Complete Cluster Reset](./docs/runbooks/cluster-reset-procedure.md) - Full environment rebuild guide

---

## Component Repositories

### 1. Infrastructure Layer - [infra-cluster-kind](https://github.com/ggoldani/infra-cluster-kind)

**Purpose:** Kubernetes cluster foundation

**What it provides:**
- Kind cluster configuration (1 control-plane + 3 workers)
- MetalLB LoadBalancer implementation
- Network configuration (172.18.0.0/16 with MetalLB pool 172.18.255.200-250)
- Automated installation scripts

**Key technologies:** Kind, MetalLB, Docker

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

---

### 3. Sample Application - [app-flask-otel](https://github.com/ggoldani/app-flask-otel)

**Purpose:** Instrumented application demonstrating full observability

**What it provides:**
- Flask web application with multiple endpoints
- OpenTelemetry SDK instrumentation (traces, metrics, logs)
- OTLP export to OpenTelemetry Collector
- Kubernetes deployment manifests

**Key technologies:** Python, Flask, OpenTelemetry SDK, Kubernetes

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

### Installation Order

1. **Set up infrastructure**
   ```bash
   git clone https://github.com/ggoldani/infra-cluster-kind.git
   cd infra-cluster-kind
   ./install.sh
   ```

2. **Deploy observability stack**
   ```bash
   git clone https://github.com/ggoldani/observability-stack.git
   cd observability-stack
   ./install.sh
   ```

3. **Deploy sample application**
   ```bash
   git clone https://github.com/ggoldani/app-flask-otel.git
   cd app-flask-otel
   kubectl apply -f k8s/
   ```

4. **Access Grafana**
   ```bash
   kubectl get svc grafana -n observability -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
   # Open http://<IP> in browser (admin/admin)
   ```

---

## Operational Documentation

### Runbooks

Operational runbooks are located in [`docs/runbooks/`](./docs/runbooks/):

| Runbook | Severity | Use Case | MTTR |
|---------|----------|----------|------|
| [Cluster DNS Failure](./docs/runbooks/cluster-dns-failure-inotify-exhaustion.md) | CRITICAL | Total cluster failure due to inotify exhaustion | 10-15 min |
| [Health Check](./docs/runbooks/cluster-health-check.md) | INFO | Daily preventive verification | 2-3 min |
| [Cluster Reset](./docs/runbooks/cluster-reset-procedure.md) | HIGH | Complete environment rebuild | 15-20 min |

### When to Use Runbooks

- **During incidents:** Quick reference for diagnosis and resolution
- **Preventive maintenance:** Regular health checks to catch issues early
- **Learning:** Understand troubleshooting methodology and root cause analysis

---

## Key Learnings & Known Issues

### Critical Configuration: inotify Limits

**Issue:** Default Debian inotify limits (64k watches) are insufficient for Docker + Kind workloads, causing cluster-wide failures after 45+ hours of uptime.

**Solution:** Set before cluster creation:
```bash
sudo sysctl -w fs.inotify.max_user_watches=524288
sudo sysctl -w fs.inotify.max_user_instances=512
echo "fs.inotify.max_user_watches = 524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances = 512" | sudo tee -a /etc/sysctl.conf
```

See [Cluster DNS Failure runbook](./docs/runbooks/cluster-dns-failure-inotify-exhaustion.md) for details.

---

## Technology Stack

| Component | Technology | Version |
|-----------|------------|---------|
| Container Orchestration | Kubernetes (Kind) | v1.31.0 |
| LoadBalancer | MetalLB | v0.14.5 |
| Metrics Storage | Grafana Mimir | Latest (distributed) |
| Log Aggregation | Grafana Loki | Latest (single binary) |
| Trace Storage | Grafana Tempo | Latest (monolithic) |
| Dashboards | Grafana | Latest |
| Telemetry Pipeline | OpenTelemetry Collector | Latest (contrib) |
| Object Storage | MinIO | Latest (standalone) |
| Application Runtime | Python + Flask | 3.13.5 |
| Instrumentation | OpenTelemetry SDK | Latest |
| Package Manager | Helm | 3.19.0 |

---

## Contributing

This is a lab/learning environment. Contributions are welcome:

1. **New runbooks:** Document issues you encounter and resolve
2. **Improvements:** Enhance existing procedures based on experience
3. **Corrections:** Fix errors or outdated information

Follow the [runbook template](./docs/runbooks/README.md#runbook-template) when creating new operational documentation.

---

## Resources

### Official Documentation
- **Kubernetes:** https://kubernetes.io/docs/
- **Kind:** https://kind.sigs.k8s.io/
- **MetalLB:** https://metallb.universe.tf/
- **Grafana Mimir:** https://grafana.com/docs/mimir/latest/
- **Grafana Loki:** https://grafana.com/docs/loki/latest/
- **Grafana Tempo:** https://grafana.com/docs/tempo/latest/
- **OpenTelemetry:** https://opentelemetry.io/docs/

### Related Projects
- **SRE Book (Google):** https://sre.google/books/
- **OpenTelemetry Demo:** https://opentelemetry.io/docs/demo/

---

## License

This is a learning/lab project. Use freely for educational purposes.

---

**Maintained by:** SRE Lab Project
**Last Updated:** 2025-11-05
**Status:** Active Development
