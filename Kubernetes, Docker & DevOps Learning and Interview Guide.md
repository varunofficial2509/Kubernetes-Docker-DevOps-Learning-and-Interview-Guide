# Kubernetes, Docker & DevOps: Learning and Interview Guide

Nov 15, 2026 · @Varun

## How to use this guide

Read it top to bottom once, then use it as a reference before interviews. Each topic follows the same pattern: what it is, when to use it, a minimal example, gotchas, and likely interview questions.

The running example is a small realistic service: a Python **worker** that reads jobs from **Redis** and writes results to **Postgres**, plus a nightly batch job. If you run something like this on Docker Compose today, every section shows how the same thing looks in Kubernetes.

**Contents**

1. Cluster architecture
2. Namespaces
3. Workloads: Pod, static pods, Deployment, ReplicaSet, DaemonSet, StatefulSet, Job, CronJob
4. Pod lifecycle and probes
5. Resource requests, limits and QoS
6. Scheduling: nodes, node pools, nodeSelector, affinity, taints and tolerations
7. Networking: Services, Endpoints, EndpointSlice, Ingress, NetworkPolicy
8. Config and storage: ConfigMap, Secret, volumes, PV, PVC, StorageClass, Secrets Store CSI, External Secrets Operator
9. RBAC: ServiceAccount, Role, ClusterRole, RoleBinding, ClusterRoleBinding
10. Agents: node agents and monitoring/logging agents
11. Autoscaling: HPA, VPA, Cluster Autoscaler
12. Rolling updates and rollbacks
13. Helm basics
14. Docker: images, layers, Dockerfile, networks, Compose, Swarm
15. Docker vs Kubernetes
16. kubectl cheat sheet
17. Troubleshooting playbook
18. Rapid-fire interview Q&A

## 1. Cluster architecture

A Kubernetes cluster is a control plane that decides what should run, plus worker nodes that run it. You tell the API server the desired state in YAML; controllers keep nudging the real state until it matches.

&#91;embedded content: cluster architecture · control plane and two worker nodes\]

No component talks to another directly: the scheduler, controllers and kubelets all read and write through the API server, which alone touches etcd.

**Control plane parts**

- **kube-apiserver**: the front door. Every client (kubectl, controllers, kubelets) talks only to it. It validates requests and stores objects in etcd.
- **etcd**: a consistent key-value store holding the whole cluster state. Lose it without a backup and you lose the cluster's memory.
- **kube-scheduler**: picks a node for each new pod, based on resources, affinity, taints and other rules. It only writes the node name onto the pod.
- **kube-controller-manager**: runs the control loops (Deployment, ReplicaSet, Node, Job, EndpointSlice controllers and more). Each loop watches objects and acts to fix drift.
- **cloud-controller-manager**: talks to the cloud provider for load balancers, node lifecycle and routes. Absent on bare-metal clusters.

**Worker node parts**

- **kubelet**: the node agent. It watches for pods assigned to its node and asks the container runtime to run them.
- **container runtime**: containerd or CRI-O, speaking the Container Runtime Interface (CRI).
- **kube-proxy**: programs iptables or IPVS rules so Service IPs reach pods. Some CNIs (Cilium) replace it.
- **CNI plugin**: gives each pod an IP and wires pod-to-pod networking (Calico, Cilium, Flannel).

**What happens on `kubectl apply -f deploy.yaml`**

1. kubectl sends the Deployment to the API server, which stores it in etcd.
2. The Deployment controller creates a ReplicaSet; the ReplicaSet controller creates pods with no node.
3. The scheduler assigns each pod to a node.
4. The kubelet on that node pulls the image and starts containers through the runtime.
5. The kubelet reports status back; the EndpointSlice controller adds ready pods to matching Services.

**Gotchas**

- Everything is declarative and eventually consistent. "Applied" does not mean "running".
- Run etcd with an odd number of members (3 or 5) so it keeps quorum when one fails.
- Managed clusters (EKS, GKE, AKS, OKE) hide the control plane; you still need to know it for interviews.

**Interview questions**

- *What does the scheduler actually do?* It filters nodes that can fit the pod, scores the rest, and binds the pod to the best node. It does not start containers.
- *What happens if the control plane goes down?* Running pods keep running and Services keep routing. Nothing new gets scheduled, healed or scaled until it returns.
- *Why does etcd need an odd number of members?* It uses Raft and needs a majority. Three members tolerate one failure; four still tolerate only one.
- *Where is cluster state stored?* Only in etcd, and only the API server talks to etcd directly.

## 2. Namespaces

A namespace is a named folder inside a cluster. It scopes names, RBAC, quotas and NetworkPolicies, so teams or environments can share one cluster without colliding.

**When to use**: split by team or environment (`dev`, `staging`) or by app (`vedno`, `monitoring`). For hard isolation between untrusted tenants, use separate clusters instead.

```bash
kubectl create namespace pipeline
kubectl config set-context --current --namespace=pipeline
kubectl get pods -A          # all namespaces
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: pipeline-quota, namespace: pipeline }
spec:
  hard: { requests.cpu: "4", requests.memory: 8Gi, pods: "30" }
```

**Gotchas**

- Some objects are cluster-scoped and live outside namespaces: Nodes, PersistentVolumes, StorageClasses, ClusterRoles, Namespaces themselves.
- Namespaces do not isolate network traffic by default. Add NetworkPolicies for that.
- DNS across namespaces needs the full name: `postgres.data.svc.cluster.local` or at least `postgres.data`.
- Deleting a namespace deletes everything in it. A namespace stuck in `Terminating` usually has a finalizer waiting on something.

**Interview questions**

- *Is a namespace a security boundary?* Partly. It scopes RBAC and quotas, but not network or kernel. Pods in different namespaces can talk unless NetworkPolicy blocks it.
- *Which default namespaces exist?* `default`, `kube-system` (cluster components), `kube-public` (readable by all) and `kube-node-lease` (node heartbeats).
- *How does a pod reach a Service in another namespace?* Use `<service>.<namespace>` or the full `<service>.<namespace>.svc.cluster.local` DNS name.

## 3. Workloads

You rarely create bare pods. You create a controller that owns pods and keeps the right number of them alive. Pick the controller by asking: does it run forever or finish, does it need a stable identity, and does it need one copy per node?

| Workload | Runs until | Pod identity | Typical use | Example in our setup |
| --- | --- | --- | --- | --- |
| Deployment | Forever | Random names, interchangeable | Stateless APIs, workers | Python worker, web API |
| ReplicaSet | Forever | Random names | Created by a Deployment; rarely used directly | (owned by the worker Deployment) |
| StatefulSet | Forever | Stable name, ordinal, own disk (`pg-0`, `pg-1`) | Databases, queues, clustered apps | Postgres, Redis with persistence |
| DaemonSet | Forever | One pod per matching node | Node agents: logs, metrics, CNI | Fluent Bit, node-exporter |
| Job | Success | Random names | One-off batch work, migrations | Database migration on release |
| CronJob | Each run succeeds | New Job per schedule | Scheduled batch | Nightly backup or report |
| Static pod | Forever (kubelet restarts it) | Fixed, mirror pod in API | Control plane on kubeadm clusters | (not used by apps) |

### Pod

The smallest unit: one or more containers sharing a network namespace (same IP, talk over `localhost`) and volumes. Extra containers are sidecars (log shipper, proxy) or init containers that run to completion first.

```yaml
apiVersion: v1
kind: Pod
metadata: { name: debug }
spec:
  initContainers:
    - name: wait-db
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z postgres 5432; do sleep 2; done"]
  containers:
    - name: shell
      image: python:3.12-slim
      command: ["sleep", "infinity"]
```

**Gotcha**: a bare pod is not rescheduled if its node dies. Use a controller.

### Deployment and ReplicaSet

A Deployment manages ReplicaSets; a ReplicaSet keeps N identical pods running. Each time you change the pod template (new image, new env), the Deployment creates a new ReplicaSet and shifts pods across. That is how rolling updates and rollbacks work.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: worker, namespace: pipeline }
spec:
  replicas: 3
  selector: { matchLabels: { app: worker } }
  template:
    metadata: { labels: { app: worker } }
    spec:
      containers:
        - name: worker
          image: ghcr.io/acme/worker:1.4.2
          envFrom: [{ configMapRef: { name: worker-config } }]
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits: { memory: 512Mi }
```

**Gotchas**

- `spec.selector` is immutable and must match the template labels.
- Changing only a ConfigMap does not restart pods. Use `kubectl rollout restart` or a checksum annotation (Helm does this).
- Old ReplicaSets are kept (default 10, `revisionHistoryLimit`) so you can roll back.
- Never manage a ReplicaSet owned by a Deployment by hand; the Deployment will fight you.

### StatefulSet

Like a Deployment, but each pod gets a stable name (`pg-0`), a stable DNS name through a headless Service, and its own PersistentVolumeClaim from `volumeClaimTemplates`. Pods start and stop in order by default.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: pg }
spec:
  serviceName: pg          # headless Service gives pg-0.pg.<ns>.svc
  replicas: 1
  selector: { matchLabels: { app: pg } }
  template:
    metadata: { labels: { app: pg } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 20Gi } }
```

**Gotchas**

- Deleting or scaling down a StatefulSet does not delete its PVCs. Data stays until you remove them.
- A StatefulSet does not make an app clustered. Postgres replication still needs an operator (CloudNativePG, Zalando) or manual setup.
- Many teams keep production databases on a managed service and run only stateless pods in Kubernetes.

### DaemonSet

Runs exactly one pod on every node (or every node matching a selector). New nodes get the pod automatically.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: node-exporter, namespace: monitoring }
spec:
  selector: { matchLabels: { app: node-exporter } }
  template:
    metadata: { labels: { app: node-exporter } }
    spec:
      hostNetwork: true
      tolerations: [{ operator: Exists }]   # also run on tainted nodes
      containers:
        - name: exporter
          image: prom/node-exporter:v1.8.2
```

**Gotcha**: tainted nodes (including control plane nodes) skip the DaemonSet unless it has matching tolerations.

### Job

Runs pods until a set number succeed, retrying failures up to `backoffLimit`.

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: migrate-1-4-2 }
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never        # Job pods must use Never or OnFailure
      containers:
        - name: migrate
          image: ghcr.io/acme/worker:1.4.2
          command: ["alembic", "upgrade", "head"]
```

**Gotchas**: pod template is immutable, so a new run needs a new Job name. Make the work idempotent because a retry may repeat it.

### CronJob

Creates a Job on a cron schedule.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly-backup }
spec:
  schedule: "30 2 * * *"
  timeZone: "Asia/Kolkata"
  concurrencyPolicy: Forbid      # skip if last run still going
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:16
              command: ["sh", "-c", "pg_dump $DATABASE_URL | gzip > /backup/db.sql.gz"]
```

**Gotchas**: without `timeZone`, the schedule uses the controller's time zone (usually UTC). A missed schedule can run late or twice, so jobs must be idempotent. Test one run with `kubectl create job --from=cronjob/nightly-backup test-run`.

### Static pods

A pod defined by a YAML file on a node's disk (usually `/etc/kubernetes/manifests`). The kubelet runs it directly, without the scheduler or API server. The API shows a read-only "mirror pod". kubeadm runs the API server, etcd, scheduler and controller manager this way.

**Gotcha**: you cannot delete a static pod with kubectl; it comes back. Remove the file on the node.

### Interview questions

- *Deployment vs StatefulSet?* Deployment pods are interchangeable with random names and shared or no storage. StatefulSet pods have stable names, ordered startup and one PVC each.
- *Why does a Deployment use ReplicaSets instead of managing pods itself?* Each ReplicaSet is one version of the pod template. Keeping old ones makes rolling updates and rollbacks simple.
- *How do you run something on every node?* A DaemonSet, with tolerations if some nodes are tainted.
- *How do static pods bootstrap a cluster?* The kubelet reads manifests from disk and starts the control plane before any API server exists.
- *What does `concurrencyPolicy: Forbid` do?* It skips a new CronJob run if the previous Job is still running. `Replace` kills the old one; `Allow` runs both.

## 4. Pod lifecycle and probes

A pod moves through phases: **Pending** (accepted, not yet running: waiting for scheduling or image pull) → **Running** → **Succeeded** or **Failed**. **Unknown** means the node stopped reporting. Container states inside are Waiting, Running and Terminated, each with a reason such as `CrashLoopBackOff` or `OOMKilled`.

**Probes** let the kubelet check health:

| Probe | Question it answers | On failure |
| --- | --- | --- |
| startupProbe | Has the app finished starting? | Keeps other probes off until it passes; restarts after too many fails |
| readinessProbe | Can it take traffic right now? | Removed from Service endpoints; not restarted |
| livenessProbe | Is it stuck beyond repair? | Container is restarted |

```yaml
containers:
  - name: worker
    image: ghcr.io/acme/worker:1.4.2
    startupProbe:
      httpGet: { path: /healthz, port: 8080 }
      failureThreshold: 30
      periodSeconds: 2          # up to 60s to start
    readinessProbe:
      httpGet: { path: /ready, port: 8080 }   # checks Redis + Postgres reachable
      periodSeconds: 5
    livenessProbe:
      httpGet: { path: /healthz, port: 8080 } # process only, no dependencies
      periodSeconds: 10
      failureThreshold: 3
    lifecycle:
      preStop: { exec: { command: ["sleep", "5"] } }
terminationGracePeriodSeconds: 30
```

**Shutdown sequence**: pod marked Terminating and removed from endpoints → `preStop` hook runs → SIGTERM to PID 1 → wait up to `terminationGracePeriodSeconds` → SIGKILL. A worker should catch SIGTERM, stop taking new jobs, finish or requeue the current one, then exit.

**Gotchas**

- A liveness probe that checks the database will restart every pod when the database blips. Keep liveness about the process; put dependency checks in readiness.
- Without a startup probe, a slow-starting app gets killed by liveness before it is up.
- If your app is not PID 1 (for example a shell script wrapper), it may never receive SIGTERM. Use `exec` in the script or an init like `tini`.
- `restartPolicy` (Always, OnFailure, Never) is per pod. Deployments require Always.

**Interview questions**

- *Readiness vs liveness?* Readiness failing removes the pod from traffic. Liveness failing restarts the container.
- *How do you get zero-downtime shutdowns?* Handle SIGTERM, use a short preStop sleep so endpoints update first, and set a grace period long enough to drain.
- *What does a pod in Pending mean?* It is accepted but not running yet: usually no node fits, a PVC is unbound, or the image is still pulling.

## 5. Resource requests, limits and QoS

**Requests** are what the scheduler reserves for a container; a pod only lands on a node with that much free. **Limits** are the hard ceiling at runtime. CPU over the limit is throttled; memory over the limit gets the container OOMKilled.

Units: CPU in cores or millicores (`250m` = a quarter core). Memory in bytes with binary suffixes (`256Mi`, `1Gi`).

```yaml
resources:
  requests: { cpu: 250m, memory: 256Mi }
  limits:   { memory: 512Mi }     # many teams skip CPU limits to avoid throttling
```

**QoS classes** decide who gets evicted first when a node runs out of memory:

| QoS class | Rule | Eviction order |
| --- | --- | --- |
| Guaranteed | Every container has requests = limits for both CPU and memory | Last |
| Burstable | At least one request or limit set, but not Guaranteed | Middle |
| BestEffort | No requests or limits at all | First |

Use **LimitRange** to give default requests and limits in a namespace, and **ResourceQuota** to cap the namespace total.

**Gotchas**

- No requests means the scheduler thinks the pod costs nothing, so nodes get overpacked.
- Requests far above real use waste money; far below cause noisy neighbours and evictions.
- Python, JVM and Node.js may not respect container memory limits unless configured. Watch heap and worker counts.
- HPA on CPU utilisation is measured against requests, so wrong requests break autoscaling.

**Interview questions**

- *What happens when a container exceeds its CPU limit? Its memory limit?* CPU is throttled and the container slows down. Memory over the limit kills the container with OOMKilled.
- *How do you make a pod Guaranteed?* Set requests equal to limits for both CPU and memory on every container.
- *Should you set CPU limits?* It is debated. Without them pods can burst into idle CPU; with them you get predictable but throttled performance. Always set memory limits and CPU requests.

## 6. Scheduling

Scheduling rules pull pods toward nodes (nodeSelector, node affinity), spread them apart (pod anti-affinity, topology spread), or push them away (taints). Remember the split: **affinity is set on the pod and attracts; taints are set on the node and repel; tolerations on the pod only permit, they never attract.**

### Nodes and node pools

A node is one machine (VM or bare metal) running the kubelet. A node pool (GKE/AKS/OKE) or node group (EKS) is a set of identical nodes managed together: same machine type, labels, taints and autoscaling range. You might have a `general` pool for workers and an `arm-highmem` pool for databases.

```bash
kubectl get nodes -o wide --show-labels
kubectl label node node-3 disktype=ssd
kubectl cordon node-3      # no new pods
kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node-3
```

### nodeSelector

The simplest rule: the pod only runs on nodes with all these labels.

```yaml
spec:
  nodeSelector:
    kubernetes.io/arch: arm64
    pool: general
```

### Node affinity

A richer version of nodeSelector with operators (`In`, `NotIn`, `Exists`, `Gt`) and hard or soft rules.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:   # hard
      nodeSelectorTerms:
        - matchExpressions:
            - { key: pool, operator: In, values: [general, burst] }
    preferredDuringSchedulingIgnoredDuringExecution:  # soft
      - weight: 50
        preference:
          matchExpressions:
            - { key: disktype, operator: In, values: [ssd] }
```

### Pod affinity and anti-affinity

Place pods relative to other pods. Affinity: "put me near the Redis pod". Anti-affinity: "never put two worker replicas on the same node". `topologyKey` sets what "same place" means (node, zone).

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector: { matchLabels: { app: worker } }
```

For even spreading, `topologySpreadConstraints` is often simpler than anti-affinity.

### Taints and tolerations

A taint on a node says "keep off unless you tolerate me". Effects: `NoSchedule` (no new pods), `PreferNoSchedule` (avoid), `NoExecute` (also evict running pods without a toleration).

```bash
kubectl taint nodes db-node-1 dedicated=database:NoSchedule
kubectl taint nodes db-node-1 dedicated=database:NoSchedule-   # remove
```

```yaml
# On the Postgres pod: allowed onto the tainted node...
tolerations:
  - { key: dedicated, operator: Equal, value: database, effect: NoSchedule }
# ...and pulled there, because a toleration alone does not attract
nodeSelector: { dedicated: database }
```

**Gotchas**

- `IgnoredDuringExecution` means relabelling a node does not evict pods already there.
- Hard rules that no node satisfies leave pods Pending forever. Prefer soft rules unless you truly need hard ones.
- Required anti-affinity with more replicas than nodes leaves the extras Pending.
- Control plane nodes carry `node-role.kubernetes.io/control-plane:NoSchedule`, which is why normal pods avoid them.
- Kubernetes adds `NoExecute` taints like `node.kubernetes.io/unreachable` on failed nodes; pods get a default 300s toleration before eviction.

**Interview questions**

- *Taints vs node affinity?* Taints on nodes repel pods that lack a toleration. Node affinity on pods attracts them to nodes. To dedicate nodes, use both.
- *How would you keep replicas on different nodes or zones?* Pod anti-affinity or `topologySpreadConstraints` with `kubernetes.io/hostname` or `topology.kubernetes.io/zone`.
- *nodeSelector vs node affinity?* nodeSelector is exact label match only. Node affinity adds operators and soft preferences.
- *What does `kubectl drain` do?* Cordons the node and evicts its pods (respecting PodDisruptionBudgets) so you can do maintenance.

## 7. Networking

Every pod gets its own IP, and any pod can reach any other pod by default (flat network, no NAT). Pod IPs change on every restart, so you put a **Service** in front: a stable virtual IP and DNS name that load-balances to the pods matching its label selector.

### Service types

| Type | Reachable from | How it works | Use when |
| --- | --- | --- | --- |
| ClusterIP (default) | Inside the cluster only | Virtual IP; kube-proxy routes to pods | Internal traffic: worker → Redis, worker → Postgres |
| NodePort | Outside, via `<any-node-IP>:30000-32767` | Opens the same port on every node, then ClusterIP | Bare metal, quick tests, behind your own load balancer |
| LoadBalancer | Outside, via a cloud LB IP | Asks the cloud for an external LB, then NodePort + ClusterIP | Exposing one service publicly on a cloud |
| ExternalName | Inside the cluster | DNS CNAME to an outside host; no proxying | Point `postgres` at a managed DB hostname |
| Headless (`clusterIP: None`) | Inside the cluster | No virtual IP; DNS returns each pod IP | StatefulSets, client-side discovery (`pg-0.pg`) |

```yaml
apiVersion: v1
kind: Service
metadata: { name: redis, namespace: pipeline }
spec:
  type: ClusterIP
  selector: { app: redis }
  ports: [{ port: 6379, targetPort: 6379 }]
---
apiVersion: v1
kind: Service
metadata: { name: managed-pg }
spec:
  type: ExternalName
  externalName: mydb.abc123.ap-south-1.rds.amazonaws.com
```

Pods find Services through cluster DNS (CoreDNS): `redis` within the namespace, `redis.pipeline.svc.cluster.local` from anywhere.

### Endpoints and EndpointSlice

When a Service has a selector, a controller keeps a list of the ready pod IPs and ports behind it. The old **Endpoints** object holds them all in one object, which gets huge for big Services. **EndpointSlice** (the current default) splits them into slices of up to 100, so updates are cheaper. kube-proxy reads EndpointSlices.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=redis
```

A Service with no selector gets no automatic endpoints; you create them by hand to point at an outside IP.

**Gotcha**: a Service with zero endpoints almost always means a label mismatch or pods failing readiness.

### Ingress

An Ingress is HTTP(S) routing rules: host and path to Service, plus TLS. It does nothing alone; an **Ingress controller** (ingress-nginx, Traefik, a cloud one) reads the rules and runs the proxy. One LoadBalancer in front of the controller can serve many apps.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations: { cert-manager.io/cluster-issuer: letsencrypt }
spec:
  ingressClassName: nginx
  tls: [{ hosts: [api.example.com], secretName: api-tls }]
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
```

The **Gateway API** (Gateway, HTTPRoute) is the newer, more expressive successor to Ingress. Know the name for interviews.

### NetworkPolicy: ingress and egress rules

By default all pod traffic is allowed. A NetworkPolicy selects pods and lists what traffic they may receive (ingress) or send (egress). Once any policy selects a pod for a direction, everything not listed is denied in that direction. Your CNI must support it (Calico, Cilium do; plain Flannel does not).

```yaml
# 1. Default deny everything in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: pipeline }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# 2. Only workers may reach Redis on 6379
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: redis-from-worker, namespace: pipeline }
spec:
  podSelector: { matchLabels: { app: redis } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: worker } } }]
      ports: [{ port: 6379 }]
---
# 3. Workers may call Redis, Postgres, DNS and HTTPS out
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: worker-egress, namespace: pipeline }
spec:
  podSelector: { matchLabels: { app: worker } }
  policyTypes: [Egress]
  egress:
    - to: [{ podSelector: { matchLabels: { app: redis } } }]
      ports: [{ port: 6379 }]
    - to: [{ podSelector: { matchLabels: { app: pg } } }]
      ports: [{ port: 5432 }]
    - to: [{ namespaceSelector: {} }]
      ports: [{ port: 53, protocol: UDP }, { port: 53, protocol: TCP }]
    - to: [{ ipBlock: { cidr: 0.0.0.0/0, except: [10.0.0.0/8] } }]
      ports: [{ port: 443 }]
```

**Gotchas**

- Forgetting to allow DNS (port 53) in an egress policy breaks everything with confusing timeouts.
- Policies are additive allow-lists. There is no "deny" rule in the core API.
- `from: [podSelector, namespaceSelector]` as two list items means OR; as one item with both keys means AND.
- LoadBalancer Services cost money per LB on clouds. Use one Ingress controller for many HTTP apps.
- `externalTrafficPolicy: Local` keeps the client source IP but only routes to nodes that host a pod.

**Interview questions**

- *ClusterIP vs NodePort vs LoadBalancer?* Each builds on the last. ClusterIP is internal; NodePort adds a port on every node; LoadBalancer adds a cloud LB in front of those node ports.
- *What is a headless Service for?* DNS returns pod IPs directly instead of one virtual IP, so StatefulSet pods are addressable by name.
- *Ingress vs LoadBalancer Service?* A LoadBalancer gives one L4 entry per Service. Ingress is L7 routing by host and path, so many services share one entry point.
- *How do you block all traffic except what is needed?* Apply a default-deny NetworkPolicy for ingress and egress, then add narrow allow policies, including DNS egress.
- *Endpoints vs EndpointSlice?* Same purpose; EndpointSlice shards the list so large Services scale and update efficiently.

## 8. Config and storage

Keep config out of the image: plain settings in ConfigMaps, sensitive values in Secrets (ideally synced from a real secret manager), and data that must survive restarts on PersistentVolumes.

### ConfigMap

Non-secret key-value config, used as env vars or mounted as files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: worker-config }
data:
  LOG_LEVEL: info
  REDIS_URL: redis://redis:6379/0
  sources.yaml: |
    feeds: [nse, bse]
```

### Secret

Same shape as a ConfigMap but for passwords, tokens and keys. Values are **base64-encoded, not encrypted**. Protect them with RBAC, etcd encryption at rest, and by never committing them to Git.

```bash
kubectl create secret generic db-creds \
  --from-literal=DATABASE_URL='postgresql://app:***@pg:5432/app'
```

### volumes and volumeMounts

`volumes` (pod level) declare where storage comes from; `volumeMounts` (container level) say where it appears in the filesystem.

```yaml
spec:
  containers:
    - name: worker
      env:
        - name: DATABASE_URL
          valueFrom: { secretKeyRef: { name: db-creds, key: DATABASE_URL } }
      volumeMounts:
        - { name: config, mountPath: /etc/worker, readOnly: true }
        - { name: scratch, mountPath: /tmp/work }
  volumes:
    - name: config
      configMap: { name: worker-config, items: [{ key: sources.yaml, path: sources.yaml }] }
    - name: scratch
      emptyDir: { sizeLimit: 1Gi }     # deleted with the pod
```

Common volume types: `emptyDir` (scratch, lives with the pod), `configMap`, `secret`, `persistentVolumeClaim`, `hostPath` (node disk: avoid for apps), `projected` (several sources in one directory), `csi`.

### PV, PVC and StorageClass

- **PersistentVolume (PV)**: a piece of real storage (cloud disk, NFS share), cluster-scoped.
- **PersistentVolumeClaim (PVC)**: a pod's request for storage ("20Gi, ReadWriteOnce"), namespaced.
- **StorageClass**: a recipe for creating PVs on demand (disk type, reclaim policy, binding mode). With dynamic provisioning you only write PVCs.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: fast }
provisioner: ebs.csi.aws.com          # or blockvolume.csi.oraclecloud.com, pd.csi.storage.gke.io
parameters: { type: gp3 }
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: redis-data }
spec:
  storageClassName: fast
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 5Gi } }
```

Access modes: `ReadWriteOnce` (one node), `ReadOnlyMany`, `ReadWriteMany` (many nodes; needs NFS/EFS-like storage), `ReadWriteOncePod` (one pod).

### SecretProviderClass (Secrets Store CSI Driver)

The Secrets Store CSI Driver mounts secrets straight from an external store (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, HashiCorp Vault) into the pod as files. A **SecretProviderClass** says which secrets to fetch. It can optionally also sync them into a Kubernetes Secret.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata: { name: worker-secrets }
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/worker/db"
        objectType: "secretsmanager"
---
# in the pod spec
volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes: { secretProviderClass: worker-secrets }
```

### External Secrets Operator (ESO)

ESO is a controller that reads from an external store and **creates normal Kubernetes Secrets**, refreshing them on an interval. You define a `SecretStore` (how to connect) and an `ExternalSecret` (what to copy).

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-sm, kind: ClusterSecretStore }
  target: { name: db-creds }          # Secret ESO will create
  data:
    - secretKey: DATABASE_URL
      remoteRef: { key: prod/worker/db, property: url }
```

|  | Secrets Store CSI | External Secrets Operator |
| --- | --- | --- |
| Delivers secrets as | Files in a mounted volume (optional sync to Secret) | Native Kubernetes Secrets |
| Secret exists in etcd | Only if sync is enabled | Yes |
| Works with env vars | Only via sync | Yes |
| Needs pod changes | Yes, a CSI volume | No |
| Good for | Keeping secrets out of etcd | Simple GitOps with existing apps |

**Gotchas**

- Env vars from a ConfigMap or Secret are read once at start. Changes need a pod restart. Mounted files update after a delay (not with `subPath`).
- `reclaimPolicy: Delete` (the usual default) deletes the disk when the PVC is deleted. Use `Retain` for anything precious.
- `WaitForFirstConsumer` creates the disk in the zone where the pod lands; `Immediate` can create it in the wrong zone and leave the pod Pending.
- Block disks are zonal. A pod with an RWO volume cannot move to a node in another zone.
- Secrets in Git: use Sealed Secrets, SOPS, or ESO, never plain YAML.

**Interview questions**

- *Are Kubernetes Secrets secure?* Only base64 by default. Turn on etcd encryption at rest, limit RBAC, and preferably source them from an external manager.
- *PV vs PVC vs StorageClass?* A PV is actual storage, a PVC is a request for it, and a StorageClass tells Kubernetes how to create PVs on demand.
- *How do you rotate a database password without redeploying?* Store it in a secret manager, sync with ESO or the CSI driver, and have the app reread the mounted file or restart via a rollout.
- *CSI driver vs ESO?* CSI mounts secrets as files and can avoid etcd. ESO creates native Secrets, which is simpler for apps that read env vars.

## 9. RBAC

RBAC answers "who can do what to which resources". A **Role** or **ClusterRole** lists allowed verbs on resources; a **RoleBinding** or **ClusterRoleBinding** grants that role to a subject (user, group or ServiceAccount). There are no deny rules; anything not granted is forbidden.

**ServiceAccount**: the identity a pod uses when it calls the Kubernetes API. Each namespace has a `default` one. Give each app its own, with only the rights it needs.

| Object | Scope of the rules | Can grant access to | Bound by |
| --- | --- | --- | --- |
| Role | One namespace | Namespaced resources in that namespace | RoleBinding |
| ClusterRole | Cluster-wide definition | Namespaced resources in any namespace, cluster-scoped resources (nodes, PVs), non-resource URLs (`/healthz`) | RoleBinding (limits it to one namespace) or ClusterRoleBinding (all namespaces) |
| RoleBinding | One namespace | Grants a Role or a ClusterRole, but only inside its namespace | — |
| ClusterRoleBinding | Whole cluster | Grants a ClusterRole everywhere | — |

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: job-launcher, namespace: pipeline }
automountServiceAccountToken: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: manage-jobs, namespace: pipeline }
rules:
  - apiGroups: [batch]
    resources: [jobs]
    verbs: [create, get, list, watch, delete]
  - apiGroups: [""]
    resources: [pods, pods/log]
    verbs: [get, list]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: job-launcher-manage-jobs, namespace: pipeline }
subjects:
  - { kind: ServiceAccount, name: job-launcher, namespace: pipeline }
roleRef: { kind: Role, name: manage-jobs, apiGroup: rbac.authorization.k8s.io }
```

```bash
kubectl auth can-i create jobs -n pipeline \
  --as=system:serviceaccount:pipeline:job-launcher     # yes
kubectl auth can-i delete nodes --as=system:serviceaccount:pipeline:job-launcher  # no
```

**Gotchas**

- `roleRef` is immutable. To change the role, delete and recreate the binding.
- Built-in ClusterRoles `view`, `edit`, `admin`, `cluster-admin` are handy. Bind `edit` with a RoleBinding to give a team rights in one namespace only.
- Pods that never call the API should set `automountServiceAccountToken: false`.
- Wildcards (`*`) and `cluster-admin` bindings are the usual audit findings. Also watch `secrets: get/list`, `pods/exec` and `create pods`, which are privilege escalation paths.
- Cloud IAM for pods (IRSA on EKS, Workload Identity on GKE/AKS) maps a ServiceAccount to a cloud role. That is how pods reach S3 or a secret manager without static keys.

**Interview questions**

- *Role vs ClusterRole?* A Role is namespaced. A ClusterRole is cluster-wide and can also cover nodes, PVs and other cluster-scoped resources.
- *Can a RoleBinding reference a ClusterRole?* Yes. It grants that ClusterRole's rules only within the binding's namespace, which lets you reuse one definition across namespaces.
- *How do you check what a ServiceAccount can do?* `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<name>`.
- *How does a pod authenticate to the API server?* With a short-lived projected ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount`.

## 10. Agents: node agents and monitoring/logging agents

"MA/agent" was not defined, so this section covers the two meanings most likely in an interview: the agents Kubernetes itself runs on every node, and the monitoring or logging agents you add as DaemonSets. If the term meant something else (for example a multi-agent system or a specific vendor's "Monitoring Agent"), note it and I can add that.

### Node agents (built in)

| Agent | Runs as | Job |
| --- | --- | --- |
| kubelet | systemd service on the host | Registers the node, runs assigned pods via the runtime, runs probes, reports status, evicts pods under pressure, runs static pods |
| kube-proxy | DaemonSet (usually) | Turns Services and EndpointSlices into iptables, IPVS or nftables rules on the node |
| Container runtime (containerd, CRI-O) | systemd service | Pulls images and runs containers through CRI |
| CNI plugin agent (Calico node, Cilium agent) | DaemonSet | Pod IPs, routing, NetworkPolicy enforcement |
| CSI node plugin | DaemonSet | Mounts and attaches volumes on the node |

```bash
systemctl status kubelet
journalctl -u kubelet -f          # first stop when a node is NotReady
crictl ps                          # containers as the runtime sees them
```

### Monitoring and logging agents (you add them)

These run as DaemonSets so every node is covered, and usually mount host paths such as `/var/log/pods`.

- **Metrics**: Prometheus node-exporter (node CPU, memory, disk), kube-state-metrics (a Deployment that exposes object state), metrics-server (feeds `kubectl top` and HPA).
- **Logs**: Fluent Bit, Fluentd, Vector or Promtail/Grafana Alloy tail container logs and ship them to Loki, Elasticsearch or a cloud service.
- **All-in-one vendor agents**: Datadog Agent, New Relic, Dynatrace OneAgent, OpenTelemetry Collector in agent mode.

```yaml
# Fluent Bit DaemonSet essentials
spec:
  template:
    spec:
      serviceAccountName: fluent-bit        # needs get/list/watch on pods for metadata
      tolerations: [{ operator: Exists }]
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.1
          resources: { requests: { cpu: 50m, memory: 64Mi }, limits: { memory: 128Mi } }
          volumeMounts: [{ name: varlog, mountPath: /var/log, readOnly: true }]
      volumes:
        - { name: varlog, hostPath: { path: /var/log } }
```

**Gotchas**

- Agents run on every node, so their resource requests multiply by node count. Keep them small.
- Without tolerations, agents skip tainted nodes, which leaves blind spots.
- Apps should log to stdout/stderr; the runtime writes those to files that agents tail. Writing to files inside the container hides logs from the agent.
- A sidecar agent per pod is the alternative to a DaemonSet. It costs more but gives per-app isolation.

**Interview questions**

- *What does the kubelet do?* It is the node agent: it watches the API for pods assigned to its node, runs them through the container runtime, runs probes and reports status.
- *Why run log collectors as a DaemonSet instead of a sidecar?* One agent per node is cheaper and needs no app changes. Sidecars suit per-app routing or multi-tenant isolation.
- *metrics-server vs Prometheus?* metrics-server keeps only current CPU and memory for `kubectl top` and HPA. Prometheus stores history and any custom metric for dashboards and alerts.
- *A node shows NotReady. Where do you look?* `kubectl describe node`, then the kubelet logs on the node, then runtime and CNI agent health, disk and memory pressure.

## 11. Autoscaling: HPA, VPA, Cluster Autoscaler

Three autoscalers work at different levels: HPA changes the number of pods, VPA changes the size of each pod, and the Cluster Autoscaler (or Karpenter) changes the number of nodes.

| Autoscaler | Scales | Based on | Notes |
| --- | --- | --- | --- |
| HPA (Horizontal Pod Autoscaler) | Replica count | CPU/memory vs requests, custom or external metrics | Built in; needs metrics-server |
| VPA (Vertical Pod Autoscaler) | Requests and limits | Historical usage | Add-on; restarts pods to apply; do not combine with HPA on the same CPU/memory metric |
| Cluster Autoscaler | Nodes in a pool | Pending pods that cannot fit; underused nodes | Works per node pool; Karpenter is a faster alternative on AWS |
| KEDA | Replica count, including to zero | Queue length, Redis list length, Kafka lag, cron | Event-driven; great for queue workers |

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: worker }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: worker }
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }
```

For a Redis-backed worker, CPU is often a poor signal. KEDA can scale on the length of the Redis stream or list instead.

**Gotchas**

- HPA utilisation is a percentage of **requests**. No requests, no CPU-based scaling.
- The Cluster Autoscaler only reacts to Pending pods, not high CPU. Pods must have requests so the scheduler sees they do not fit.
- Scale-down can evict pods. Use PodDisruptionBudgets to keep a minimum available.
- Remove `replicas` from your Deployment YAML (or Helm values) when HPA owns it, or every apply resets the count.

**Interview questions**

- *HPA vs VPA vs Cluster Autoscaler?* HPA adds pods, VPA resizes pods, Cluster Autoscaler adds or removes nodes. They often work together: HPA adds pods, some go Pending, the cluster autoscaler adds a node.
- *How would you autoscale a queue consumer?* Use KEDA or an HPA on an external metric such as queue depth, with a sensible max and a scale-down window.
- *Why might HPA not scale?* metrics-server missing, no CPU requests set, target already at max, or metrics unavailable for new pods.

## 12. Rolling updates and rollbacks

A Deployment rolls out a change by creating a new ReplicaSet and shifting pods across a few at a time, waiting for each new pod to be Ready. Two knobs control the pace: `maxSurge` (extra pods allowed above the desired count) and `maxUnavailable` (pods allowed to be missing).

```yaml
spec:
  strategy:
    type: RollingUpdate          # or Recreate: kill all, then start new
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  minReadySeconds: 10
  progressDeadlineSeconds: 600
```

```bash
kubectl set image deploy/worker worker=ghcr.io/acme/worker:1.4.3
kubectl rollout status deploy/worker
kubectl rollout history deploy/worker
kubectl rollout undo deploy/worker                 # back one revision
kubectl rollout undo deploy/worker --to-revision=4
kubectl rollout pause deploy/worker                # batch several edits
kubectl rollout restart deploy/worker              # new pods, same spec
```

StatefulSets roll pods one at a time from the highest ordinal down, and support `partition` for canaries. DaemonSets roll node by node. For blue-green and canary with traffic shifting, teams use Argo Rollouts, Flagger or a service mesh.

**Gotchas**

- Without a readiness probe, Kubernetes marks new pods Ready immediately and rolls out broken versions quickly.
- `kubectl rollout undo` reverts only the pod template. It does not undo ConfigMap changes or database migrations.
- Use immutable image tags (`1.4.3` or a digest), not `latest`, or rollbacks may pull the same broken image.
- Recreate causes downtime but is needed when two versions cannot run side by side (for example an RWO volume or a singleton consumer).
- In GitOps (Argo CD, Flux), roll back by reverting the commit, or the controller re-applies the bad version.

**Interview questions**

- *How does a rolling update avoid downtime?* It starts new pods before removing old ones, and only sends traffic to pods passing readiness.
- *maxSurge 1 / maxUnavailable 0 vs 0 / 1?* The first adds a new pod before removing one (needs spare capacity, no drop). The second removes first (no extra capacity, briefly one fewer pod).
- *How do you roll back?* `kubectl rollout undo`, `helm rollback`, or revert the Git commit in GitOps.
- *What makes a rollout "stuck"?* New pods never become Ready; after `progressDeadlineSeconds` the Deployment reports `ProgressDeadlineExceeded`, while old pods keep serving.

## 13. Helm basics

Helm is a package manager for Kubernetes. A **chart** is a folder of templated YAML plus default `values.yaml`; a **release** is one installed copy of a chart with your values; each upgrade adds a **revision** you can roll back to.

```text
worker-chart/
  Chart.yaml          # name, version, appVersion, dependencies
  values.yaml         # defaults: image tag, replicas, resources
  templates/
    deployment.yaml   # uses {{ .Values.image.tag }}
    service.yaml
    _helpers.tpl
```

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis bitnami/redis -n pipeline --create-namespace -f redis-values.yaml
helm upgrade --install worker ./worker-chart -n pipeline --set image.tag=1.4.3
helm template ./worker-chart -f prod.yaml     # render locally, apply nothing
helm diff upgrade worker ./worker-chart       # needs the helm-diff plugin
helm history worker -n pipeline
helm rollback worker 3 -n pipeline
helm uninstall worker -n pipeline
```

**Kustomize** is the main alternative: no templating, just a base plus overlays that patch it (`kubectl apply -k`). Many teams use Helm for third-party apps and Kustomize or simple charts for their own.

**Gotchas**

- Helm 3 has no Tiller; release state is stored as Secrets in the release namespace.
- Uninstalling a chart usually leaves PVCs and CRDs behind.
- Pin chart versions (`--version`) as well as image tags.
- Templates can hide a lot. Always `helm template` or `helm diff` before upgrading production.
- Do not put secrets in `values.yaml` in Git. Use ESO, SOPS (helm-secrets) or `--set` from a CI secret.

**Interview questions**

- *Chart vs release?* A chart is the package; a release is one installed instance of it with specific values.
- *How does Helm roll back?* It re-applies the manifests from an earlier stored revision with `helm rollback <release> <revision>`.
- *Helm vs Kustomize?* Helm templates with variables and packages apps with versioning. Kustomize patches plain YAML with overlays and is built into kubectl.

## 14. Docker

Docker packages an app and its dependencies into an **image**, and runs that image as an isolated **container** using Linux namespaces (isolation) and cgroups (resource limits). Kubernetes runs the same OCI images, through containerd rather than the Docker daemon.

### Images and layers

An image is a stack of read-only layers, one per filesystem-changing Dockerfile instruction (`RUN`, `COPY`, `ADD`). A running container adds a thin writable layer on top. Layers are cached and shared between images, so order matters: put what changes least at the top of the Dockerfile.

```bash
docker build -t ghcr.io/acme/worker:1.4.3 .
docker history ghcr.io/acme/worker:1.4.3       # layers and their sizes
docker image inspect --format '{{.RepoDigests}}' ghcr.io/acme/worker:1.4.3
docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/acme/worker:1.4.3 --push .
```

**Tag vs digest**: a tag (`1.4.3`, `latest`) is a movable label. A digest (`@sha256:...`) is the exact content and never changes.

### Dockerfile best practices

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS build
WORKDIR /app
# deps first: this layer stays cached until requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# multi-stage: the final image has no build tools
FROM python:3.12-slim
RUN useradd --create-home --uid 10001 app
WORKDIR /app
COPY --from=build /install /usr/local
COPY --chown=app:app src/ ./src/
# never run as root
USER 10001
# logs reach stdout immediately
ENV PYTHONUNBUFFERED=1
HEALTHCHECK CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8080/healthz')"
# exec form: the app is PID 1 and receives SIGTERM
ENTRYPOINT ["python", "-m", "src.worker"]
```

Note: Dockerfile comments must start a line; a `#` after an instruction is passed as an argument.

- Use small, pinned base images (`-slim`, distroless, or a digest).
- Order instructions from least to most frequently changed to reuse cache.
- Combine related `RUN` steps and clean package caches in the same step.
- Add a `.dockerignore` (`.git`, `.venv`, `__pycache__`, `.env`) to keep context small and secrets out.
- Never bake secrets into layers, even if you delete them later; use BuildKit `--mount=type=secret`.
- Scan images (Trivy, Grype, Docker Scout) and build for the target CPU (ARM vs x86).

### Docker networks

| Driver | What it does | Use when |
| --- | --- | --- |
| bridge (default) | Private virtual network on one host; NAT out; `-p` publishes ports | Single-host containers. User-defined bridges add DNS by container name |
| host | Container shares the host's network stack; no isolation, no port mapping | Max network performance, or tools that need host interfaces (Linux only) |
| none | Only loopback | Fully isolated batch jobs |
| overlay | Virtual network spanning several Docker hosts (VXLAN) | Docker Swarm services across nodes |
| macvlan | Container gets its own MAC and IP on the physical LAN | Legacy apps that must look like a physical device on the network |

```bash
docker network create pipeline-net
docker run -d --name redis --network pipeline-net redis:7
docker run -d --name worker --network pipeline-net -e REDIS_URL=redis://redis:6379 ghcr.io/acme/worker:1.4.3
```

**Gotcha**: the default `bridge` network has no name-based DNS. Create your own network (Compose does this for you).

### Docker Compose

Compose describes a multi-container app on one host in a single YAML file. It is ideal for local development and small single-VM deployments.

```yaml
services:
  worker:
    image: ghcr.io/acme/worker:1.4.3
    env_file: .env
    depends_on:
      redis: { condition: service_healthy }
      postgres: { condition: service_healthy }
    restart: unless-stopped
    deploy: { resources: { limits: { memory: 512M } } }
  redis:
    image: redis:7
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s }
    volumes: [redis-data:/data]
  postgres:
    image: postgres:16
    environment: { POSTGRES_PASSWORD_FILE: /run/secrets/pg_password }
    secrets: [pg_password]
    healthcheck: { test: ["CMD", "pg_isready"], interval: 5s }
    volumes: [pg-data:/var/lib/postgresql/data]
volumes: { redis-data: {}, pg-data: {} }
secrets: { pg_password: { file: ./secrets/pg_password.txt } }
```

```bash
docker compose up -d
docker compose ps
docker compose logs -f worker
docker compose pull && docker compose up -d     # update images
docker compose down            # keeps named volumes; add -v to delete them
```

### Docker Swarm

Swarm is Docker's built-in orchestrator. `docker swarm init` turns hosts into a cluster of managers (Raft) and workers. You deploy **services** with a replica count, and **stacks** from Compose files, over overlay networks with a routing mesh.

```bash
docker swarm init
docker stack deploy -c compose.yaml pipeline
docker service scale pipeline_worker=3
docker service update --image ghcr.io/acme/worker:1.4.4 pipeline_worker
```

Swarm is simpler than Kubernetes but has a much smaller ecosystem. Most new projects choose Kubernetes (or k3s for small setups).

**Gotchas**

- `depends_on` waits for start, not readiness, unless you add `condition: service_healthy`.
- `docker compose down -v` deletes named volumes, including your database.
- Containers are ephemeral. Anything not in a volume is lost when the container is removed.
- An image built on an ARM Mac or ARM VM will not run on x86 nodes unless built multi-arch.
- Published ports (`-p 5432:5432`) bypass some host firewalls (Docker edits iptables). Bind to `127.0.0.1` when only local access is needed.

**Interview questions**

- *Image vs container?* An image is a read-only template made of layers. A container is a running instance with a writable layer on top.
- *How do you make images smaller and faster to build?* Multi-stage builds, slim base images, `.dockerignore`, and ordering layers so dependency installs stay cached.
- *CMD vs ENTRYPOINT?* ENTRYPOINT is the fixed executable; CMD is its default arguments, easy to override. Use exec form so signals reach the app.
- *bridge vs host network?* Bridge isolates the container on a private network with NAT and port mapping. Host shares the host's network directly, with no isolation.
- *Compose vs Swarm?* Compose runs a multi-container app on one host. Swarm schedules services across several hosts and can deploy Compose files as stacks.

## 15. Docker vs Kubernetes

They are not rivals: Docker builds and runs containers on one machine; Kubernetes schedules and heals containers across many machines. You build with Docker and run on Kubernetes.

| Concern | Docker / Compose | Kubernetes |
| --- | --- | --- |
| Scope | One host (Swarm for a few) | Many nodes as one cluster |
| Unit | Container / Compose service | Pod, managed by a controller |
| Self-healing | `restart:` policy on the same host | Reschedules to healthy nodes, replaces failed pods |
| Scaling | `--scale` by hand | HPA, VPA, cluster autoscaler |
| Service discovery | Container names on a user network | Services + CoreDNS |
| Rolling updates | Recreate on `up -d` (Swarm can roll) | Built in, with rollback |
| Config and secrets | `.env`, Compose secrets (files) | ConfigMap, Secret, ESO, CSI |
| Storage | Named volumes on the host | PV/PVC with dynamic provisioning |
| Networking policy | Host firewall | NetworkPolicy |
| Access control | Whoever can reach the Docker socket is root | RBAC, ServiceAccounts |
| Learning curve and overhead | Low | High; a control plane to run or pay for |

**When Compose is enough**: one VM, a handful of services, downtime of a few seconds on deploy is fine. A worker plus Redis plus Postgres on one ARM VM is a good Compose fit.

**When Kubernetes earns its cost**: several nodes, need for zero-downtime deploys, autoscaling, many services or teams, or you want the skills. A light middle path is **k3s**, a small single-binary Kubernetes that runs well on one ARM VM and can join a second machine over a VPN such as Tailscale.

**Interview questions**

- *Does Kubernetes use Docker?* It runs OCI images built by Docker, but through containerd or CRI-O. The dockershim was removed in v1.24.
- *When would you not use Kubernetes?* Small apps on one host, small teams without ops time, or when a managed PaaS covers the need.
- *Compose to Kubernetes: what maps to what?* service → Deployment + Service, volume → PVC, env\_file → ConfigMap/Secret, healthcheck → probes, depends\_on → init containers or readiness logic.

## 16. kubectl cheat sheet

```bash
# Context and namespace
kubectl config get-contexts
kubectl config use-context prod
kubectl config set-context --current --namespace=pipeline

# Look around
kubectl get pods -o wide                 # node and IP
kubectl get all -n pipeline
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get events --sort-by=.lastTimestamp
kubectl describe pod worker-7d9f-abcde   # events at the bottom
kubectl explain deployment.spec.strategy # built-in field docs
kubectl api-resources                    # every kind and its short name

# Logs and shells
kubectl logs deploy/worker -f
kubectl logs worker-7d9f-abcde -c worker --previous   # crashed container
kubectl exec -it worker-7d9f-abcde -- sh
kubectl debug -it worker-7d9f-abcde --image=busybox --target=worker
kubectl port-forward svc/postgres 5432:5432

# Apply and change
kubectl apply -f k8s/                   # declarative
kubectl diff -f k8s/                    # what would change
kubectl delete -f k8s/worker.yaml
kubectl scale deploy/worker --replicas=5
kubectl set image deploy/worker worker=ghcr.io/acme/worker:1.4.4
kubectl rollout status|history|undo|restart deploy/worker

# Generate YAML instead of writing it
kubectl create deployment worker --image=ghcr.io/acme/worker:1.4.4 --dry-run=client -o yaml
kubectl create job test --from=cronjob/nightly-backup

# Resources and access
kubectl top pods --sort-by=memory
kubectl top nodes
kubectl auth can-i list secrets --as=system:serviceaccount:pipeline:worker

# Nodes
kubectl cordon|uncordon node-3
kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data
kubectl taint nodes node-3 key=value:NoSchedule

# Output tricks
kubectl get pod worker-7d9f-abcde -o yaml
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
kubectl get pods -l app=worker -w        # watch
```

Tip: set `alias k=kubectl` and enable shell completion; most exams and interviews assume it.

## 17. Troubleshooting playbook

Start every investigation the same way: `kubectl get pods` for the status, `kubectl describe pod <name>` for the events at the bottom, and `kubectl logs <name> --previous` for the last crash. The status word tells you which row below to use.

| Symptom | What it means | Usual causes | How to check and fix |
| --- | --- | --- | --- |
| Pending | Not scheduled yet | Not enough CPU/memory for the requests; nodeSelector, affinity or taints match no node; PVC unbound or volume in another zone; quota exceeded | `describe pod` events ("0/3 nodes are available: ..."); lower requests, add nodes, fix labels/tolerations, check `kubectl get pvc` |
| ImagePullBackOff / ErrImagePull | Kubelet cannot pull the image | Typo in name or tag; tag does not exist; private registry without `imagePullSecrets`; wrong CPU architecture; rate limits | `describe pod` shows the registry error; `docker pull` the exact name; add a pull secret; build multi-arch |
| CrashLoopBackOff | Container starts, exits, and is restarted with growing delay | App error on start; missing env var or Secret; can't reach DB; wrong command; liveness probe killing it | `logs --previous`; check exit code in `describe` (1 = app error, 137 = killed, 139 = segfault); `exec` in or `kubectl debug`; relax probes |
| OOMKilled (exit 137) | Container exceeded its memory limit | Limit too low; memory leak; too many worker processes or threads; large batch loaded at once | `describe pod` → Last State: OOMKilled; `kubectl top pod`; raise the limit or cut memory use; check the node is not under memory pressure (eviction) |
| CreateContainerConfigError | Pod spec references something missing | ConfigMap or Secret (or a key in it) does not exist | `describe pod`; create the object or fix the key name |
| Running but not Ready | Readiness probe failing | Dependency down, wrong probe path/port, slow start | `describe` probe failures; test the endpoint with `exec`/`port-forward`; add a startup probe |
| Service not reachable | Traffic does not reach pods | Selector does not match labels; wrong `targetPort`; pods not Ready; NetworkPolicy blocking; DNS | `kubectl get endpointslices -l kubernetes.io/service-name=<svc>`; test from a debug pod with `nslookup` and `curl` |
| Terminating forever | Pod or namespace will not go away | Finalizers; node gone; app ignores SIGTERM | Check `metadata.finalizers`; `kubectl delete pod --grace-period=0 --force` as a last resort |
| Evicted | Kubelet removed the pod under node pressure | Node low on memory or disk; BestEffort pods go first | `describe node` conditions; set requests/limits; clean up disk |

**A debug pod for network problems**

```bash
kubectl run netcheck --rm -it --image=nicolaka/netshoot -n pipeline -- bash
# inside: nslookup redis; nc -zv postgres 5432; curl -v http://api/healthz
```

**Gotchas**

- `CrashLoopBackOff` is a symptom, not a cause. The cause is always in the previous container's logs or exit code.
- Exit code 137 can be OOMKilled or a liveness-probe kill. `describe` tells you which.
- Events expire after about an hour, so look early.

**Interview questions**

- *A pod is in CrashLoopBackOff. Walk me through it.* Read `logs --previous`, check the exit code and Last State in `describe`, confirm env vars and Secrets exist, check probes, then reproduce with `kubectl debug` or by overriding the command to `sleep`.
- *A pod is stuck Pending. Why?* The scheduler found no node: insufficient resources, unmatched selectors or affinity, untolerated taints, or an unbound PVC. The event message says which.
- *Exit code 137 vs 1?* 137 means killed by SIGKILL (OOM or forced kill); 1 is the app exiting with an error.
- *Service returns connection refused. What do you check?* Endpoints exist, labels match, `targetPort` matches the container port, pods are Ready, and no NetworkPolicy blocks the path.

## 18. Rapid-fire interview Q&A

One-line answers to drill before an interview. Say the short answer first, then add one detail if asked.

**Architecture and basics**

1. *What is a pod?* The smallest deployable unit: one or more containers sharing an IP and volumes.
2. *What stores cluster state?* etcd, and only the API server talks to it.
3. *What does the kubelet do?* Runs the pods assigned to its node and reports their status.
4. *What does kube-proxy do?* Programs node networking rules so Service IPs reach pods.
5. *What is a controller?* A loop that watches desired state and acts to make actual state match.
6. *What is a CRD?* A Custom Resource Definition: it adds your own object kind to the API; an operator is a controller for it.
7. *What is a label vs an annotation?* Labels are for selecting objects; annotations hold non-identifying metadata for tools.

**Workloads**

8. *Deployment vs StatefulSet?* Interchangeable pods vs stable identity and per-pod storage.
9. *DaemonSet use case?* One agent per node: logging, metrics, CNI.
10. *Job vs CronJob?* A Job runs to completion once; a CronJob creates Jobs on a schedule.
11. *What owns the pods of a Deployment?* A ReplicaSet, which the Deployment owns.
12. *What is a static pod?* A pod the kubelet runs from a file on disk, without the API server.
13. *What is an init container?* A container that must finish successfully before the main containers start.
14. *What is a PodDisruptionBudget?* A minimum number of pods that must stay up during voluntary disruptions like drains.

**Scheduling and resources**

15. *Taint vs toleration?* A taint on a node repels pods; a toleration on a pod allows it onto that node.
16. *How do you pin pods to certain nodes?* nodeSelector or node affinity; add taints to keep others off.
17. *Requests vs limits?* Requests are reserved for scheduling; limits are the runtime ceiling.
18. *The three QoS classes?* Guaranteed, Burstable, BestEffort; BestEffort is evicted first.
19. *What kills a container over its memory limit?* The kernel OOM killer; status shows OOMKilled, exit code 137.

**Networking**

20. *Default Service type?* ClusterIP.
21. *Headless Service?* `clusterIP: None`; DNS returns pod IPs directly.
22. *ExternalName Service?* A DNS CNAME to an outside hostname, with no proxying.
23. *What does an Ingress need to work?* An Ingress controller.
24. *Are pods isolated from each other by default?* No; all traffic is allowed until a NetworkPolicy selects them.
25. *Most common egress-policy mistake?* Forgetting to allow DNS on port 53.

**Config, storage, security**

26. *Are Secrets encrypted?* Base64 only by default; enable encryption at rest and use an external manager.
27. *ConfigMap change not picked up?* Env vars are read at start; restart the pods or mount as a file.
28. *PVC stuck Pending?* No matching StorageClass or PV, or the disk is waiting for the first pod (WaitForFirstConsumer).
29. *Role vs ClusterRole?* Namespaced vs cluster-wide permissions.
30. *How does a pod call the API securely?* With its ServiceAccount token, limited by RBAC.
31. *Least privilege for pods?* Own ServiceAccount, minimal Role, non-root user, read-only root filesystem, no token mount if unused.

**Operations**

32. *How do you roll back a bad deploy?* `kubectl rollout undo`, `helm rollback`, or revert the Git commit.
33. *HPA scales on what by default?* CPU utilisation as a percentage of requests.
34. *What adds nodes when pods are Pending?* The Cluster Autoscaler or Karpenter.
35. *What is Helm?* A package manager: charts are templated manifests; releases are installed copies with revisions.
36. *What is GitOps?* Git holds the desired state and a controller (Argo CD, Flux) applies it to the cluster continuously.
37. *How do you drain a node safely?* `kubectl drain` with PDBs in place, after which pods reschedule elsewhere.

**Docker**

38. *Image vs container?* A read-only template vs a running instance with a writable layer.
39. *Why multi-stage builds?* To leave build tools out of the final image so it is smaller and safer.
40. *Why order Dockerfile instructions carefully?* Each instruction is a cached layer; changing one rebuilds everything after it.
41. *CMD vs ENTRYPOINT?* ENTRYPOINT is the fixed executable; CMD supplies default arguments.
42. *Why exec form for ENTRYPOINT?* So the app is PID 1 and receives SIGTERM.
43. *Default Docker network driver?* bridge.
44. *Which Docker network spans hosts?* overlay.
45. *Compose vs Kubernetes?* Compose runs an app on one host; Kubernetes schedules and heals across many.

**Troubleshooting**

46. *CrashLoopBackOff first step?* `kubectl logs <pod> --previous`.
47. *ImagePullBackOff causes?* Wrong name or tag, missing pull secret, wrong architecture, rate limits.
48. *Pending first step?* `kubectl describe pod` and read the scheduler event.
49. *Service has no endpoints?* Label selector mismatch or pods not Ready.
50. *Node NotReady?* Check kubelet logs, runtime, CNI agent, and disk or memory pressure.
