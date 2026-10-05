# kubevigil

> A Kubernetes operator, written in Go, that turns raw vulnerability scans into **one clear
> security status per workload**, and takes a **safe, auditable action** when a critical CVE
> shows up on something that is actually running.

**Status: in development (started Oct 2026).** This README describes the design the code is
built against. Nothing is claimed as working until its box is ticked in the [Roadmap](#roadmap).

---

## Why this exists

A CI scan tells you an image was clean **at build time**. A week later a new CVE is
published, the image is still running in production, and nothing in your pipeline knows.

The [Trivy Operator](https://github.com/aquasecurity/trivy-operator) closes half of that gap:
it keeps scanning running workloads and publishes `VulnerabilityReport` objects. But those
reports are:

- **per container** and **per ReplicaSet revision**, so one Deployment produces many of them,
- **scattered** across namespaces and mixed with reports of old, already-replaced pods,
- **passive**: nothing reacts to them.

kubevigil is the missing piece. It answers the question an on-call engineer actually asks:

> *"Which of my live workloads are exposed to a critical CVE right now, and did anyone get told?"*

## What it does

1. **Watches** Trivy Operator `VulnerabilityReport` objects and the workloads they belong to.
2. **Correlates** them: walks the owner chain (Pod → ReplicaSet → Deployment) to the top-level
   workload (Deployment, StatefulSet, DaemonSet) and aggregates every container. Reports of old
   ReplicaSet revisions are ignored.
3. **Publishes** the result in its own CRD, `WorkloadSecurityStatus`, kept up to date.
4. **Reacts** when a workload crosses a critical-CVE threshold, **dry-run by default**.
5. **Exposes** Prometheus metrics.

```
 running pods ──► Trivy Operator ──► VulnerabilityReport (per container / ReplicaSet)
                   (not our code)               │
                                                ▼
                                    ┌───────────────────────┐
   Deployment / StatefulSet ───────►│       kubevigil       │
   DaemonSet (owner chain)          │  watch → resolve →    │
                                    │  aggregate → decide   │
                                    └───┬───────┬───────┬───┘
                                        │       │       │
                       WorkloadSecurityStatus   │       └─► Prometheus metrics
                       (status + conditions)    │
                                                └─► Reaction (policy-gated)
                                                     Event · annotation · webhook
```

Trivy Operator is a dependency that is **deployed**, never modified or reimplemented.

### What you see

```console
$ kubectl get wss
NAME               KIND         WORKLOAD   CRITICAL   HIGH   ACTION-REQUIRED   AGE
deployment-nginx   Deployment   nginx      3          12     True              4m
```

*(Illustrative output matching the printer columns of the CRD; a real demo capture will
replace this once `make demo` exists.)*

## Design decisions worth reading

| Decision | Why |
|---|---|
| **Level-triggered reconciliation** | The status is recomputed from the current state, not from a stream of events. A missed event or an operator restart cannot leave it stale |
| **Dry-run is the default** | An operator that can act on a cluster must be safe to install first and trust later |
| **Reaction fingerprint** | A reaction fires only when the *set* of critical CVEs changes, so reconciling 100 times never spams events |
| **One status object per workload** | Matches how people reason about ownership and keeps GC trivial through owner references |
| **Annotate the Deployment, never the pod template** | Touching the template would trigger a rollout: a security tool must not restart production |
| **No hot loop** | An unchanged status is never rewritten, so a quiet cluster costs almost nothing |

The reasoning is written down as ADRs in `docs/adr/` (typed vs dynamic client for Trivy reports,
one WSS per workload, dry-run + fingerprint). *(Planned, see roadmap.)*

## API at a glance

| | |
|---|---|
| CRD | `WorkloadSecurityStatus` (`wss`), `security.nuage.dev/v1alpha1`, namespaced |
| Cardinality | One object per top-level workload, named `<kind>-<name>`, in the workload's namespace |
| Lifecycle | Created by the operator, garbage-collected when the workload is deleted |
| Status | Severity counts, per-image breakdown, `lastScanTime`, conditions `Scanned` / `ActionRequired`, `lastReaction` |

**Reactions**

| Action | Milestone | Behavior |
|---|---|---|
| Warning `Event` (`CriticalVulnerability`) | v0.1 | Always. Prefixed `[dry-run]` in dry-run mode |
| Metric `kubevigil_reactions_total{action,dry_run}` | v0.1 | Always |
| Annotate the workload (`security.nuage.dev/critical-cves`) | v0.2 | `enforce` only. Never touches the pod template |
| Webhook POST (URL read from a Secret) | v0.2 | `enforce` only |
| Node cordon | later | Off by default: needs a flag, `enforce` and an opt-in node label |

**Metrics**: `kubevigil_workload_vulnerabilities{namespace,kind,workload,severity}` (gauge, series
removed when the workload is deleted) and `kubevigil_reactions_total{action,dry_run}`, plus the
standard controller-runtime metrics.

## How it's built

- **Go**, scaffolded with Kubebuilder and controller-runtime (standard layout).
- **RBAC** is generated from kubebuilder markers: no `*` verbs, no cluster-admin.
- **Observability**: Prometheus metrics with series cleanup, structured logs with consistent
  keys (`namespace`, `workload`, `reportName`).
- **Tests**: unit tests on the pure logic, `envtest` + Ginkgo scenarios against a real API
  server, and one end-to-end smoke test on kind.
- **CI**: GitHub Actions running lint, `go vet`, `go test -race` and envtest; images published
  to `ghcr.io` on tag; installable with kustomize or Helm.
- **Decisions** are recorded as ADRs in `docs/adr/`.

## Related tools

kubevigil is deliberately narrow and sits on top of existing tools rather than replacing them.

| Tool | Relationship |
|---|---|
| [Trivy Operator](https://github.com/aquasecurity/trivy-operator) | The scanner kubevigil consumes. Provides raw reports, no per-workload rollup or reaction |
| Prometheus / Alertmanager on Trivy metrics | The common DIY approach. Alerts, but no per-workload status object and no dedup by CVE set |
| [Kubescape](https://github.com/kubescape/kubescape), commercial platforms | Broader posture and runtime products. kubevigil does one thing: rollup and a safe reaction |
| Kyverno / OPA Gatekeeper | Admission-time policy. kubevigil acts on what is already running |

## Quickstart

*Not available yet.* The target is a single command on a local [kind](https://kind.sigs.k8s.io/)
cluster:

```bash
make demo   # kind cluster + Trivy Operator + kubevigil + a vulnerable workload → visible reaction
```

## Roadmap

**v0.1: first public release**
- [ ] Kubebuilder scaffold, standard layout
- [ ] `WorkloadSecurityStatus` CRD: validation, status subresource, printer columns
- [ ] Controller: report → workload resolution, multi-container aggregation, old ReplicaSets ignored, GC
- [ ] Reaction: Event + metric, dry-run default, fingerprint (no event spam)
- [ ] Prometheus gauge with series cleanup
- [ ] Unit tests + envtest/Ginkgo suite (≥ 6 scenarios)
- [ ] `make demo` on kind
- [ ] 3 ADRs

**v0.2**
- [ ] Annotation and webhook actions (`enforce` mode)
- [ ] Per-workload override via `spec.reaction`
- [ ] `Scanned` / `ActionRequired` conditions with reasons and transition times
- [ ] CI + image on `ghcr.io` + kustomize/Helm install

**Later**
- [ ] Guarded node cordon
- [ ] Cosign signature + SBOM verification at runtime
- [ ] Job / CronJob workloads
- [ ] Leader election with a test proving a single actor reacts

## Limitations

Stated up front, on purpose:

- It **depends on the Trivy Operator**. It does not scan images itself.
- It is only as fresh as the Trivy reports: if Trivy Operator is down, kubevigil keeps showing
  the last known state (reflected by `lastScanTime`) rather than inventing data.
- v1alpha1: the API may change.
- Single cluster only; no multi-cluster aggregation.
- Deployments, StatefulSets and DaemonSets only at first. Jobs and CronJobs are a bonus item.
- It reports and nudges; it is **not** an admission controller and does not block deployments.
- `enforce` mode is intentionally limited: annotate and notify, never delete or restart pods.
