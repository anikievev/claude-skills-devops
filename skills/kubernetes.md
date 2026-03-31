---
name: kubernetes
description: Write production-grade Kubernetes manifests following security hardening, resource governance, reliability patterns, and least-privilege RBAC best practices
---

Analyze the current project and write production-grade Kubernetes manifests. If manifests already exist, review and improve them.

## Requirements

**Inspect the project first:**
- Detect application type (stateless web service, stateful database, batch job, daemon, etc.)
- Identify ports, environment variables, config/secret needs, and storage requirements
- Check for existing manifests in `k8s/`, `deploy/`, `helm/`, or similar directories
- Identify target environment (dev/staging/production) — affects replica counts, resource limits, and PDB settings

**File structure — organize manifests by resource type:**

```
k8s/
├── namespace.yaml
├── serviceaccount.yaml
├── rbac.yaml              # Role + RoleBinding (or ClusterRole if needed)
├── configmap.yaml
├── secret.yaml            # Template only — never real values; use sealed-secrets or external-secrets in practice
├── deployment.yaml        # (or statefulset.yaml / daemonset.yaml / cronjob.yaml)
├── service.yaml
├── ingress.yaml           # (if externally exposed)
├── hpa.yaml               # HorizontalPodAutoscaler
├── pdb.yaml               # PodDisruptionBudget
└── networkpolicy.yaml
```

For multi-environment projects, separate by directory: `k8s/base/` + `k8s/overlays/production/` (Kustomize pattern).

---

## Mandatory rules

### YAML hygiene
- Every manifest must have all four top-level fields: `apiVersion`, `kind`, `metadata`, `spec` — missing any causes cryptic `kubectl` errors
- Use only `true`/`false` for booleans — never `yes`, `no`, `on`, `off`; YAML parsers treat these inconsistently across versions
- Keep manifests minimal — do not set fields that Kubernetes already defaults correctly; extra fields create noise and drift
- Add `kubernetes.io/description` annotations to explain *why* a resource exists, not just what it is:
  ```yaml
  annotations:
    kubernetes.io/description: "Serves the public API for the payments platform. Scaled by HPA based on CPU."
  ```

### API versions
- Always use non-deprecated API versions — check the target Kubernetes version
- Common current versions: `apps/v1` (Deployment/StatefulSet/DaemonSet), `batch/v1` (Job/CronJob), `networking.k8s.io/v1` (Ingress/NetworkPolicy), `autoscaling/v2` (HPA)
- Never use `extensions/v1beta1` or `apps/v1beta1` — removed in Kubernetes 1.16+

### Labels — always apply this baseline to every resource
```yaml
labels:
  app.kubernetes.io/name: myapp
  app.kubernetes.io/instance: myapp-production
  app.kubernetes.io/version: "1.0.0"
  app.kubernetes.io/component: backend      # frontend / backend / database / cache
  app.kubernetes.io/part-of: myplatform
  app.kubernetes.io/managed-by: kubectl     # or helm / kustomize / argocd
```
Selectors in Deployment/Service must be a **subset** of these labels. Never include `version` in selector labels — it prevents rolling updates.

### Security context — mandatory on every Pod and container
```yaml
spec:
  securityContext:                        # Pod-level
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault               # or Localhost for custom profiles
  containers:
    - securityContext:                    # Container-level
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true     # mount tmpfs for writable paths explicitly
        capabilities:
          drop: ["ALL"]
          add: []                        # add only if explicitly required (e.g. NET_BIND_SERVICE)
```

### Resource requests and limits — mandatory on every container
- Always set both `requests` and `limits`
- `requests` drives scheduling; `limits` prevents noisy-neighbour resource starvation
- For JVM apps, set `requests` == `limits` (Guaranteed QoS) to prevent OOM-kill during GC pauses
- Never set CPU limits to very low values on bursty workloads — it causes throttling, not OOM
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

### Probes — mandatory for all long-running containers
```yaml
startupProbe:               # prevents liveness killing a slow-starting container
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10

readinessProbe:             # removes pod from Service endpoints when not ready
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:              # restarts the container if it deadlocks
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 30
  failureThreshold: 3
```
Use `tcpSocket` probe when the app has no HTTP health endpoint. Use `exec` probe only as a last resort — it spawns a subprocess on every check.

### Deployment — reliability settings
```yaml
spec:
  replicas: 2                              # minimum 2 for production; 1 acceptable for dev
  revisionHistoryLimit: 3                  # limit rollback history (default 10 wastes etcd space)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0                    # zero-downtime: never take a pod down before a new one is ready
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 30    # must be >= your app's graceful shutdown timeout
      topologySpreadConstraints:           # spread pods across nodes/zones for resilience
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: myapp
```

### Workload type
- **Never use naked Pods in production** — if the node dies, a naked Pod is not rescheduled; always use Deployment, StatefulSet, DaemonSet, Job, or CronJob
- Use `Job` for one-off tasks (database migrations, data imports) and `CronJob` for recurring work
- Add `startingDeadlineSeconds` to every CronJob — without it, if the scheduler misses the window (e.g. during a cluster upgrade), it silently skips the run:
  ```yaml
  spec:
    schedule: "0 2 * * *"
    startingDeadlineSeconds: 300   # fail if not started within 5 minutes of scheduled time
    concurrencyPolicy: Forbid
  ```

### RBAC — least privilege
- Create a dedicated `ServiceAccount` per workload — never use the `default` ServiceAccount
- Disable auto-mounting of the service account token unless the app explicitly needs Kubernetes API access:
  ```yaml
  automountServiceAccountToken: false
  ```
- Scope `Role` to the namespace; only use `ClusterRole` when cross-namespace or cluster-scoped resources are needed
- Grant only the verbs the workload actually needs — never `["*"]` in `apiGroups`, `resources`, or `verbs`
- **Never bind a ServiceAccount to `cluster-admin`** — create a scoped custom Role with only the required permissions; `cluster-admin` grants unrestricted access to every resource in the cluster

### Secrets
- Never put real secret values in manifests — use a reference pattern:
  - **Sealed Secrets** (`bitnami.com/v1alpha1` SealedSecret) for GitOps workflows
  - **External Secrets Operator** (`external-secrets.io/v1beta1` ExternalSecret) for secrets from AWS Secrets Manager, Vault, GCP Secret Manager
- For the manifest itself, provide a `secret.yaml` with `data` values set to `<base64-encoded-placeholder>` and a comment documenting the expected secret keys
- Mount secrets as files (`volumeMounts`) rather than environment variables where possible — env vars are visible in `kubectl describe pod` output

### HorizontalPodAutoscaler
- Always use `autoscaling/v2` (supports multiple metrics)
- Set `minReplicas >= 2` for production to survive a node failure during scale-down
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### PodDisruptionBudget
- Mandatory for any production workload with `replicas >= 2`
- Prevents voluntary disruptions (node drains, cluster upgrades) from taking down all pods simultaneously
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 1      # or maxUnavailable: 1 — never set minAvailable == replicas
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
```

### NetworkPolicy
- Default-deny all ingress and egress; then explicitly allow what is needed
- Always include a default-deny policy in every namespace alongside specific allow policies
```yaml
# Default deny-all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

### Anti-patterns — never do these
- **`image: myapp:latest`** — unpinned tags make deployments non-reproducible; always use a specific digest or version tag
- **`resources: {}`** — no resource limits allows a single pod to consume all node memory
- **`runAsUser: 0` or no `securityContext`** — runs as root; a container breakout becomes a node compromise
- **`hostNetwork: true`, `hostPID: true`, or `hostIPC: true`** — breaks pod isolation; only acceptable for node-level DaemonSets with a documented reason
- **`hostPort`** — ties the pod to a specific node, breaks scheduling, and bypasses Service load balancing; use a Service instead
- **`privileged: true`** — equivalent to root on the host; almost never required
- **Mounting `/var/run/docker.sock`** — gives the container full control over the Docker daemon and thus the node; use Kaniko or Buildah for in-cluster image builds instead
- **Putting secrets in `ConfigMap`** — ConfigMaps are not encrypted at rest in etcd; use `Secret` kind or external secret management
- **Duplicate environment variable names** in the same container `env` array — the last value wins silently; audit for duplicates when merging env lists
- **Hardcoding cluster-internal IPs** — use DNS-based service discovery (`my-service.namespace.svc.cluster.local`); IPs change on Service recreation
- **`selector` includes `version` label** — changing the version label tears down the Deployment instead of rolling it
- **Service selector mismatch with Pod labels** — silently routes to zero pods; always validate that selector keys and values match the Pod template exactly
- **Single replica in production** — one pod means zero tolerance for node failure or eviction
- **`terminationGracePeriodSeconds: 0`** — forces immediate SIGKILL; in-flight requests are dropped; always allow graceful shutdown
- **Binding ServiceAccount to `cluster-admin`** — grants unrestricted cluster access; create a scoped Role instead

---

## Output format

Provide all manifests as separate fenced code blocks labeled with the filename:

```yaml
# k8s/deployment.yaml
...
```

**Apply ordering** — create resources in dependency order so references resolve correctly:
Namespace → ServiceAccount → RBAC → ConfigMap → Secret → **Service** → workload (Deployment/StatefulSet) → Ingress → HPA → PDB → NetworkPolicy

> **Services before workloads** — Kubernetes injects Service host/port as environment variables into Pods, but only if the Service exists when the Pod starts. Creating the Service first ensures env vars are injected correctly.

**Preferred apply command** — use server-side apply to avoid field manager conflicts and get better conflict detection:
```bash
kubectl apply --server-side -f k8s/
```

**Pre-apply validation** — run before `kubectl apply`:
```bash
# Schema validation (no cluster needed)
kubeconform -strict -summary k8s/

# Lint for best-practice violations
kube-linter lint k8s/

# Or: kubeval (offline schema check)
kubeval --strict k8s/*.yaml
```

After the manifests, include:
1. An **Apply** section: `kubectl apply --server-side -f k8s/` with any ordering notes
2. A **Verify** section: `kubectl get`, `kubectl describe`, and `kubectl rollout status` commands to confirm healthy deployment
3. A **Security notes** section flagging any `capabilities.add`, `hostPath` mounts, or other elevated permissions and why they were necessary

---

## Examples

### Stateless web service (Deployment + Service + Ingress + HPA + PDB)

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    app.kubernetes.io/part-of: myplatform
```

```yaml
# k8s/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: myapp
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/component: backend
automountServiceAccountToken: false   # app does not call Kubernetes API
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/component: backend
spec:
  replicas: 2
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp      # no version label in selector
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/version: "1.2.3"
        app.kubernetes.io/component: backend
    spec:
      serviceAccountName: myapp
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: myapp
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:1.2.3   # pinned tag; prefer digest in production
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          envFrom:
            - configMapRef:
                name: myapp-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-password
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp              # writable tmpfs for apps that need a temp dir
          startupProbe:
            httpGet:
              path: /health
              port: http
            failureThreshold: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 30
            failureThreshold: 3
      volumes:
        - name: tmp
          emptyDir: {}
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
  labels:
    app.kubernetes.io/name: myapp
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: myapp
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
```

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp
  namespace: myapp
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
```

---

### StatefulSet (database / message broker)

Key differences from Deployment: stable network identity, ordered pod management, `volumeClaimTemplates` for per-pod persistent storage.

```yaml
# k8s/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres-headless    # must match the headless Service name
  replicas: 1                       # for primary-only; use operator pattern for HA
  selector:
    matchLabels:
      app.kubernetes.io/name: postgres
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: postgres
        app.kubernetes.io/component: database
    spec:
      serviceAccountName: postgres
      securityContext:
        runAsNonRoot: true
        runAsUser: 999              # postgres user UID
        runAsGroup: 999
        fsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      terminationGracePeriodSeconds: 60   # longer for databases — allow checkpoint to complete
      containers:
        - name: postgres
          image: postgres:16.3-alpine3.20  # pinned
          ports:
            - name: postgres
              containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: database-name
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata   # subdirectory avoids lost+found issue on ext4
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false  # postgres writes to PGDATA — cannot use readOnly here
            capabilities:
              drop: ["ALL"]
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 30
            periodSeconds: 30
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3         # use cluster-appropriate StorageClass
        resources:
          requests:
            storage: 10Gi
---
# Headless Service — required for stable DNS names (postgres-0.postgres-headless)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: myapp
spec:
  type: ClusterIP
  clusterIP: None                    # headless
  selector:
    app.kubernetes.io/name: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
```

---

### CronJob (scheduled batch task)

```yaml
# k8s/cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: myapp-cleanup
  namespace: myapp
spec:
  schedule: "0 2 * * *"            # 02:00 UTC daily
  timeZone: "UTC"
  startingDeadlineSeconds: 300      # fail if not started within 5 min of scheduled time (e.g. cluster was down)
  concurrencyPolicy: Forbid         # never run two instances simultaneously
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2               # retry up to 2 times before marking failed
      activeDeadlineSeconds: 3600   # kill job if still running after 1 hour
      template:
        metadata:
          labels:
            app.kubernetes.io/name: myapp-cleanup
            app.kubernetes.io/component: batch
        spec:
          serviceAccountName: myapp
          restartPolicy: OnFailure
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            seccompProfile:
              type: RuntimeDefault
          containers:
            - name: cleanup
              image: ghcr.io/myorg/myapp:1.2.3
              command: ["./bin/cleanup"]
              resources:
                requests:
                  cpu: "100m"
                  memory: "64Mi"
                limits:
                  cpu: "500m"
                  memory: "256Mi"
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities:
                  drop: ["ALL"]
```

---

### RBAC (ServiceAccount + Role + RoleBinding)

```yaml
# k8s/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp
  namespace: myapp
rules:
  # Grant only what the app actually needs — document why each rule is here
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]   # read-only; app watches for config reloads
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]                    # read specific secrets only
    resourceNames: ["myapp-secrets"]  # restrict to named resource — not all secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp
  namespace: myapp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: myapp
subjects:
  - kind: ServiceAccount
    name: myapp
    namespace: myapp
```

---

### NetworkPolicy (default-deny + targeted allow)

```yaml
# k8s/networkpolicy.yaml

# 1. Default deny all traffic in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: myapp
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# 2. Allow ingress from ingress controller only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: myapp
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx   # namespace of ingress controller
      ports:
        - port: 8080
          protocol: TCP
---
# 3. Allow app to reach postgres within the same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-postgres
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: postgres
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: myapp
      ports:
        - port: 5432
          protocol: TCP
---
# 4. Allow DNS egress (required for all pods)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: myapp
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```
