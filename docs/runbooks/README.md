# SRE Lab Runbooks

This directory contains operational runbooks for the SRE Lab environment. Runbooks provide step-by-step procedures for diagnosing and resolving common issues.

## Purpose

Runbooks serve as:
- **Incident response guides** - Quick reference during outages
- **Knowledge base** - Captured operational experience
- **Training material** - Learn troubleshooting methodology
- **Standardization** - Consistent approach to problem resolution

## How to Use

1. **During an incident**: Identify symptoms, find matching runbook, follow diagnostic steps
2. **Preventively**: Review runbooks to understand potential issues
3. **For learning**: Study the diagnostic methodology and root cause analysis

## Available Runbooks

### Critical Severity

| Runbook | Symptoms | MTTR | Status |
|---------|----------|------|--------|
| [Cluster DNS Failure (Inotify Exhaustion)](./cluster-dns-failure-inotify-exhaustion.md) | CoreDNS not ready, kube-proxy crashes, "too many open files" | 10-15 min | ✅ Validated |

### High Severity

| Runbook | Symptoms | MTTR | Status |
|---------|----------|------|--------|
| *More runbooks will be added as issues are encountered* | - | - | - |

### Medium Severity

| Runbook | Symptoms | MTTR | Status |
|---------|----------|------|--------|
| *More runbooks will be added as issues are encountered* | - | - | - |

## Runbook Template

When creating new runbooks, follow this structure:

```markdown
# Runbook: [Issue Title]

**Severity:** [CRITICAL|HIGH|MEDIUM|LOW]
**Impact:** [Description of business/technical impact]
**MTTR:** [Expected time to resolve]

## Symptoms
- Primary indicators (what you see first)
- Secondary effects (cascading failures)
- Detection commands

## Root Cause
- What happened
- Why it happened
- Technical explanation

## Diagnostic Steps
1. Step-by-step investigation procedure
2. Commands to run
3. Expected outputs

## Resolution
1. Step-by-step fix procedure
2. Exact commands
3. Verification steps

## Verification Checklist
- [ ] Checklist items to confirm resolution

## Prevention
- Long-term fixes
- Monitoring recommendations
- Best practices

## Key Learnings
- Insights from troubleshooting
- Gotchas and edge cases

## References
- Related documentation
- External resources
```

## Common Issue Quick Reference

### Symptoms → Runbook Mapping

| Symptom | Likely Cause | Runbook |
|---------|--------------|---------|
| CoreDNS pods not ready | Inotify exhaustion, API server issues | [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md) |
| "too many open files" error | Inotify watches exhausted (NOT FDs) | [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md) |
| kube-proxy CrashLoopBackOff | Inotify exhaustion, network issues | [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md) |
| Docker restart fails | Inotify exhaustion | [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md) |
| Init container permission errors | PVC corruption after cluster failure | [Cluster DNS Failure](./cluster-dns-failure-inotify-exhaustion.md#related-issues) |

## General Troubleshooting Workflow

When encountering issues not covered by existing runbooks:

1. **Gather symptoms**
   ```bash
   kubectl get pods --all-namespaces
   kubectl get events --all-namespaces --sort-by='.lastTimestamp'
   kubectl describe pod <failing-pod> -n <namespace>
   kubectl logs <failing-pod> -n <namespace> --previous
   ```

2. **Check cluster health**
   ```bash
   kubectl get nodes
   kubectl cluster-info
   kubectl get componentstatus  # Deprecated but useful
   kubectl get --raw /healthz
   ```

3. **Check infrastructure**
   ```bash
   docker ps
   docker stats
   sudo systemctl status docker
   df -h  # Disk space
   free -h  # Memory
   ```

4. **Review logs**
   ```bash
   sudo journalctl -u docker --since "1 hour ago"
   sudo journalctl -u kubelet --since "1 hour ago"
   ```

5. **Search for similar issues**
   - Check this runbook directory
   - GitHub issues for Kind, Kubernetes, specific components
   - Stack Overflow with specific error messages

6. **Document the solution**
   - Create a new runbook using the template above
   - Submit PR to add it to this repository

## Contributing

Found a new issue and resolved it? Please:
1. Create a runbook using the template above
2. Add it to the appropriate severity section in this README
3. Update the symptoms mapping table
4. Commit with descriptive message: `docs: add runbook for [issue]`

## Resources

- **Kubernetes Debugging**: https://kubernetes.io/docs/tasks/debug/
- **Kind Troubleshooting**: https://kind.sigs.k8s.io/docs/user/known-issues/
- **Docker Issues**: https://docs.docker.com/config/daemon/troubleshoot/
- **SRE Book (Google)**: https://sre.google/books/

---

**Last Updated:** 2025-11-05
**Maintained By:** SRE Lab Project
**Runbook Count:** 1
