---
name: kubernetes-helm
description: Production Helm chart layout — values structure, probes, HPA, NetworkPolicies, PodSecurity, secrets via External Secrets Operator, and the small-team patterns the official tutorial skips. Use when packaging a service for Kubernetes, not when authoring a public chart for a registry.
---

# Production Helm Chart

The official Helm tutorial gets you to "it deploys." This skill gets you to "it survives." Designed for in-house service charts (one chart per service, internal repo), not public charts.

## When to use

- Packaging your own service for Kubernetes deployment.
- Migrating raw `kubectl apply` manifests to Helm.
- Setting probes, autoscaling, or NetworkPolicies for the first time.
- Refactoring a chart that's grown unwieldy.

## When NOT to use

- Pure stateless functions — Knative or KServe may be simpler.
- You're publishing a chart to Bitnami/Artifact Hub — different bar (much higher).
- You only have one service — raw manifests + Kustomize might be lighter.

## Decisions made for you

| Decision | Choice | Why |
|---|---|---|
| Tooling | Helm 3 | Tiller is dead; Helm 3 is what you have |
| Layout | One chart per service | Don't pack multiple services in one chart |
| Values structure | Flat-ish, env-overridable | Deep nested values are unreadable |
| Probes | `startupProbe` + `readinessProbe` + `livenessProbe`, all distinct | Default templates conflate them |
| Secrets | External Secrets Operator → AWS SM/Vault | No secrets in `values.yaml` ever |
| HPA | `metric: requests/sec` or CPU | Memory-based HPA is usually wrong |
| PodSecurity | restricted | Distroless image + nonroot user; no CAP_SYS_ANYTHING |
| NetworkPolicy | default-deny + per-service allows | Air-gap by default |
| Image | distroless or Alpine | Glibc images haul 200MB of attack surface |

## Chart structure

```
charts/myservice/
  Chart.yaml
  values.yaml              # defaults
  values.dev.yaml          # dev overrides
  values.staging.yaml
  values.prod.yaml
  templates/
    _helpers.tpl
    deployment.yaml
    service.yaml
    serviceaccount.yaml
    hpa.yaml
    pdb.yaml               # PodDisruptionBudget
    networkpolicy.yaml
    externalsecret.yaml
    ingress.yaml
    servicemonitor.yaml    # for Prometheus Operator
  README.md
```

## values.yaml (the shape that scales)

```yaml
image:
  repository: ghcr.io/acme/myservice
  tag: ""               # default to .Chart.AppVersion if empty
  pullPolicy: IfNotPresent

replicaCount: 2

resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 1000m, memory: 512Mi }

env:
  LOG_LEVEL: info
  PORT: "8080"

# Secrets are pulled by ExternalSecret, not set here
secrets:
  enabled: true
  store: aws-secretsmanager
  remoteRef: myservice/prod

probes:
  startup:    { path: /healthz, periodSeconds: 5,  failureThreshold: 30 }
  readiness:  { path: /readyz,  periodSeconds: 10, failureThreshold: 3 }
  liveness:   { path: /healthz, periodSeconds: 30, failureThreshold: 3 }

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: myservice.example.com
      paths: [{ path: /, pathType: Prefix }]
  tls:
    - secretName: myservice-tls
      hosts: [myservice.example.com]

networkPolicy:
  enabled: true
  ingressFrom:
    - namespaceSelector: { matchLabels: { name: gateway } }
  egressTo:
    - to: [{ namespaceSelector: { matchLabels: { name: db } } }]
      ports: [{ port: 5432, protocol: TCP }]
    - to: []                           # DNS to kube-system
      ports: [{ port: 53, protocol: UDP }]

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 65532
  fsGroup: 65532
  seccompProfile: { type: RuntimeDefault }

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities: { drop: [ALL] }

serviceMonitor:
  enabled: true
  interval: 30s
  path: /metrics
```

## The three probes (use all three)

```yaml
# templates/deployment.yaml — probe block
startupProbe:
  httpGet: { path: /healthz, port: http }
  periodSeconds: 5
  failureThreshold: 30        # gives slow-start apps 150s before liveness kicks in
readinessProbe:
  httpGet: { path: /readyz, port: http }
  periodSeconds: 10
  failureThreshold: 3         # 30s of failure → yanked from svc
livenessProbe:
  httpGet: { path: /healthz, port: http }
  periodSeconds: 30
  failureThreshold: 3         # 90s of bad health → restarted
```

| Probe | Failing means | Action |
|---|---|---|
| startup | App hasn't booted yet | Suppress liveness/readiness during this phase |
| readiness | Can't serve traffic *right now* | Yank from Service endpoints |
| liveness | Stuck/deadlocked | Restart the pod |

Conflating them = pods that get killed during slow startup, or pods that serve while broken.

## NetworkPolicy: default-deny + allow lists

Most clusters have no NetworkPolicy → every pod can talk to every pod → one compromised pod owns the lateral movement game. Fix it:

```yaml
# Cluster-wide default deny (apply once per namespace)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: prod }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

Then each service chart adds its own allow rules (see `networkPolicy:` in values.yaml above).

## PodDisruptionBudget (don't skip)

```yaml
# templates/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: {{ include "myservice.fullname" . }} }
spec:
  minAvailable: 1
  selector: { matchLabels: { app.kubernetes.io/name: {{ include "myservice.name" . }} } }
```

Without a PDB, a node drain or cluster autoscaler scale-down can take all replicas of your service down at once.

## Anti-patterns

- **Multiple services in one chart** — coupling deploys across services. One chart per service.
- **Hardcoded tags in values.yaml** — re-deploys reset to old version. Use `image.tag: ""` defaulting to `.Chart.AppVersion` and bump the chart per release.
- **Secrets in `values.yaml` or git** — they'll leak. ESO + Secrets Manager.
- **No resource limits** — one bad pod hogs the node, evicts others. Always set requests AND limits.
- **Memory-based HPA on Java/Node services** — JVM/V8 hold heap, never release. HPA scales up infinitely. Use CPU or RPS.
- **Single probe path `/health`** — same problem as in `scaffold-go-microservice`. Split.
- **`runAsRoot`** — fails PodSecurity restricted, audit findings, real escape risk. Run as non-root.
- **`readOnlyRootFilesystem: false`** — typical default. If your app writes to `/tmp`, mount an `emptyDir` for it; lock the rest.
- **No NetworkPolicy** — flat network, one compromised pod = full lateral movement.
- **No PodDisruptionBudget** — node maintenance kills all your pods at once.
- **Privileged containers because it was easier** — security review will fail. Find the actual capability needed (e.g. `NET_ADMIN`) and grant only that.

## Verify it worked

- [ ] `helm template charts/myservice -f values.prod.yaml | kubectl apply --dry-run=server -f -` is clean.
- [ ] `kubectl rollout status deploy/myservice` succeeds within 2 min.
- [ ] Slow-starting app: startup probe gives it ≥150s before liveness kicks in.
- [ ] Killing the DB → readiness goes 503 → service endpoints update → no traffic served.
- [ ] PDB present: `kubectl drain` waits for replacement pods.
- [ ] HPA scales up under load (synthetic test).
- [ ] NetworkPolicy: pod from another namespace cannot reach this service.
- [ ] `kubectl exec -it … -- whoami` returns nonroot UID, not 0.
- [ ] Pod runs with `readOnlyRootFilesystem: true` without errors.
- [ ] Helm rollback works: `helm rollback myservice 1` returns to previous version.
- [ ] Prometheus scrapes `/metrics` (ServiceMonitor present and matching).
