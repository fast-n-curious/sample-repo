# sample-repo

A sample repository containing Kubernetes workload manifests and a Helm chart for demonstrating Kyverno Pod Security Standards (PSS) policy scanning. The resources include a mix of **compliant** and **intentionally non-compliant** configurations to exercise both PSS Baseline and PSS Restricted policy sets.

---

## Repository Structure

```
sample-repo/
├── helm-chart/
│   └── webapp/                  # Sample Helm chart — nginx frontend web app
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── _helpers.tpl
│           ├── deployment.yaml  # Contains PSS violations
│           ├── service.yaml
│           ├── serviceaccount.yaml
│           ├── configmap.yaml
│           └── hpa.yaml
└── k8s-manifests/               # Raw Kubernetes YAML — backend and ops workloads
    ├── namespace.yaml
    ├── debug-pod.yaml           # Contains PSS violations
    ├── monitoring-daemonset.yaml# Contains PSS violations
    ├── backend-deployment.yaml  # Contains PSS violations
    ├── frontend-deployment.yaml # Compliant
    ├── database-statefulset.yaml# Compliant
    └── batch-job.yaml           # Compliant
```

---

## Scanning with nctl

Use `nctl scan` against the Kyverno PSS Baseline and Restricted policy sets to generate violation reports.

```bash
# Scan raw Kubernetes manifests
nctl scan policy --resources k8s-manifests/ --policy-sets pss-baseline
nctl scan policy --resources k8s-manifests/ --policy-sets pss-restricted

# Scan rendered Helm chart templates
helm template webapp helm-chart/webapp/ | nctl scan policy --resources - --policy-sets pss-baseline
helm template webapp helm-chart/webapp/ | nctl scan policy --resources - --policy-sets pss-restricted
```

---

## Helm Chart — `helm-chart/webapp/`

A frontend web application chart that deploys an nginx server. The `values.yaml` contains insecure security defaults that are referenced in `templates/deployment.yaml`, making it straightforward to observe how Helm-rendered manifests surface PSS violations.

### Files

| File | Description |
|------|-------------|
| `Chart.yaml` | Chart metadata: name, version, appVersion |
| `values.yaml` | Default values — includes insecure `security.*` fields used by the deployment template |
| `templates/_helpers.tpl` | Named template helpers for labels, fullname, serviceAccountName |
| `templates/deployment.yaml` | Main workload — **contains PSS violations** (see below) |
| `templates/service.yaml` | ClusterIP Service — compliant |
| `templates/serviceaccount.yaml` | ServiceAccount with `automountServiceAccountToken: false` — compliant |
| `templates/configmap.yaml` | nginx config mounted into the deployment — compliant |
| `templates/hpa.yaml` | HorizontalPodAutoscaler targeting the deployment — compliant |

### PSS Violations in `templates/deployment.yaml`

| Violation | Field | PSS Level | Policy |
|-----------|-------|-----------|--------|
| Host network access | `spec.hostNetwork: true` | Baseline | [Host Namespaces](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) |
| Privileged container | `securityContext.privileged: true` | Baseline | [Privileged Containers](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) |
| Dangerous capability added | `securityContext.capabilities.add: ["NET_ADMIN"]` | Baseline | [Capabilities](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) |
| Privilege escalation allowed | `securityContext.allowPrivilegeEscalation: true` | Restricted | [Privilege Escalation](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) |

---

## Kubernetes Manifests — `k8s-manifests/`

Raw YAML manifests representing backend services and ops tooling deployed into the `backend` namespace. Includes a mix of violating and compliant resources.

### Files

| File | Workload | Compliant |
|------|----------|-----------|
| `namespace.yaml` | Namespace `backend` | Yes |
| `debug-pod.yaml` | Pod: `debug-node` | No — see violations below |
| `monitoring-daemonset.yaml` | DaemonSet: `node-metrics-collector` | No — see violations below |
| `backend-deployment.yaml` | Deployment: `api-server` | No — see violations below |
| `frontend-deployment.yaml` | Deployment: `frontend` + Service + ServiceAccount | Yes |
| `database-statefulset.yaml` | StatefulSet: `postgres` + Service + ServiceAccount | Yes |
| `batch-job.yaml` | Job: `db-migration` + ServiceAccount | Yes |

### PSS Violations

#### `debug-pod.yaml` — Pod: `debug-node`

| Violation | Field | PSS Level | Policy |
|-----------|-------|-----------|--------|
| Host PID namespace shared | `spec.hostPID: true` | Baseline | [Host Namespaces](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) |
| Host filesystem exposed | `volumes[].hostPath.path: /proc` | Baseline | [HostPath Volumes](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) |
| Container runs as root | `securityContext.runAsUser: 0` | Restricted | [Running as Non-root](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) |

#### `monitoring-daemonset.yaml` — DaemonSet: `node-metrics-collector`

| Violation | Field | PSS Level | Policy |
|-----------|-------|-----------|--------|
| Host port bound | `ports[].hostPort: 9100` | Baseline | [Host Ports](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline) |
| Dangerous capability added | `securityContext.capabilities.add: ["SYS_PTRACE"]` | Restricted | [Capabilities](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) |
| Seccomp profile not set | `securityContext.seccompProfile` missing | Restricted | [Seccomp](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) |

#### `backend-deployment.yaml` — Deployment: `api-server`

| Violation | Field | PSS Level | Policy |
|-----------|-------|-----------|--------|
| Container may run as root | `securityContext.runAsNonRoot: false` | Restricted | [Running as Non-root](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) |
| Seccomp profile not set | `securityContext.seccompProfile` missing | Restricted | [Seccomp](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) |

### Compliant Resources

The following resources satisfy all PSS Baseline and Restricted requirements:

- **`frontend-deployment.yaml`**: Sets `runAsNonRoot: true`, `runAsUser: 1000`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, and `seccompProfile.type: RuntimeDefault`.
- **`database-statefulset.yaml`**: Same hardened security context as frontend; uses a PVC for storage instead of hostPath.
- **`batch-job.yaml`**: Short-lived migration job with `readOnlyRootFilesystem: true`, all capabilities dropped, and a RuntimeDefault seccomp profile.

---

## PSS Policy Levels Reference

| Level | Description |
|-------|-------------|
| **Baseline** | Minimally restrictive. Prevents known privilege escalation paths. Targets intentional misconfigurations (privileged containers, host namespaces, hostPath volumes, host ports, dangerous capabilities). |
| **Restricted** | Heavily restricted. Follows current pod hardening best practices. Requires explicit non-root execution, dropped capabilities, and seccomp profiles on top of all Baseline requirements. |

Full specification: [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
