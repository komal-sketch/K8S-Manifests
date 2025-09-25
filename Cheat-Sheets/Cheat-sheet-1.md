# Kubernetes Cheat-Sheet


---

## Core Concepts

**Monolith vs Microservices**
- *Monolith*: one deployable. ✅ simpler; ❌ scale & blast radius.
- *Microservices*: many small services. ✅ independent scale/releases; ❌ ops complexity (networking, tracing, consistency).

**Kubernetes Architecture**
- *Control Plane*: API server (front door), etcd (state), controller-manager (reconcile), scheduler (place pods).
- *Nodes*: kubelet (pod lifecycle), kube-proxy (Service VIPs), container runtime (containerd).
- Managed control plane on cloud (EKS/GKE/AKS) is typical.

**Setup (Local vs Cloud)**
- *Local dev*: kind / minikube for fast feedback.
- *AWS Canada*: EKS in `ca-central-1` (Montreal). Data residency can matter (PIPEDA); ask about region, encryption, backups.

**kubectl essentials**
- Read: 
    kubectl get pods -A
    kubectl describe pod <name> -n <ns>
- Change:
    kubectl apply -f <file.yml>
    kubectl rollout restart deploy/<name> -n <ns>
- Context/Namespace:
    kubectl config get-contexts
    kubectl config set-context --current --namespace=<ns>

**Pods & Multi-container patterns**
- *Single container* is most common.
- *Init container*: one-time setup before app.
- *Sidecar*: runs with app (logs, proxy, metrics).

**Namespaces, Labels, Selectors, Annotations**
- NS = isolation & policy boundary.
- Labels = selection (Services, Deployments).
- Annotations = metadata for tools (Prometheus, owners, links).

---

## Workloads

**Deployment** (stateless, rolling updates)
- *Use when:* web/API, workers that can be replaced.
- Health checks:
    readinessProbe  → gate traffic
    livenessProbe   → restart on failure
- Commands:
    kubectl get deploy -n <ns>
    kubectl rollout status deploy/<name> -n <ns>

**ReplicaSet** (keep N pods; usually owned by Deployment)
- Rarely used directly.

**StatefulSet** (stable identity + storage)
- Per-pod DNS & PVC via `volumeClaimTemplates`.
- Use for DBs, Kafka, ZK, RabbitMQ.
- Needs headless Service:
    spec.clusterIP: None

**DaemonSet** (one pod per node)
- Node agents (CNIs, logs, metrics).
    kubectl get ds -n <ns>

**Job/CronJob** (run to completion / on a schedule)
- Job:
    kubectl create job <name> --image=busybox -- echo hi
- CronJob:
    apiVersion: batch/v1
    kind: CronJob
    spec.schedule: "*/5 * * * *"

---

## Networking

**Cluster Networking**
- CNI provides pod IPs & routing (Calico, Cilium, AWS VPC CNI).
- CoreDNS for service discovery.
- kube-proxy (iptables/ipvs) implements Service VIP L4 LB.

**Service types**
| Type | Purpose | Typical use |
|---|---|---|
| ClusterIP | Internal VIP | east-west traffic |
| NodePort | Port on each node | labs/bare metal quick access |
| LoadBalancer | Cloud LB | north-south exposure |
| Headless (`None`) | No VIP, DNS to pods | StatefulSet peer discovery |

**Ingress**
- L7 HTTP routing + TLS termination. Needs a controller (nginx, AWS ALB).
- Example:
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    spec:
      rules:
      - host: app.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80

**NetworkPolicies**
- Default-deny, then allow narrowly.
- Example default-deny:
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    spec:
      podSelector: {}
      policyTypes: ["Ingress","Egress"]

---

## Storage

**PersistentVolume (PV) & PersistentVolumeClaim (PVC)**
- PV = actual storage piece (cluster-scoped).
- PVC = request for storage (namespaced).
- AccessModes: RWO (common), ROX, RWX.

**StorageClass**
- Tells Kubernetes **how** to provision storage (CSI).
- Cloud examples: EBS CSI (EKS), PD CSI (GKE), Azure Disk CSI (AKS).
- `volumeBindingMode: WaitForFirstConsumer` helps zone-aware placement.

**ConfigMaps & Secrets**
- ConfigMap for non-sensitive config.
- Secret for credentials (base64; encrypt at rest; RBAC).
- Mount or `env`:
    env:
    - name: DB_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url

---

## Scaling & Scheduling

**HPA (Horizontal)**
- Scales replicas based on metrics (CPU/memory/custom).
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    spec.metrics[0].resource.target.averageUtilization: 60
- Needs **metrics-server**.

**VPA (Vertical)**
- Recommends/sets container requests/limits.
- Great for baseline right-sizing; combine with HPA carefully.

**Node Affinity / Pod Affinity**
- Steer pods to nodes (labels like `topology.kubernetes.io/zone`):
    spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution…

**Taints & Tolerations**
- Taint nodes (prod=true:NoSchedule); pods with toleration can land there.

**ResourceQuotas & LimitRanges**
- Namespace guardrails: total CPU/mem/PVCs, default/request/limit per pod.

**Probes**
- `readinessProbe` keeps unready pods out of Service endpoints.
- `livenessProbe` restarts stuck containers.
- `startupProbe` for slow starters.

---

## Advanced Features

**Operators / CRDs**
- CRD defines a new API kind; Operator (controller) reconciles it (e.g., `PostgresCluster`, `KafkaTopic`, `Certificate`).
- Use when you need a first-class platform API with automation.

**Helm**
- Package manager for K8s (charts). Reuse templates; values per env.
    helm install <release> <chart> -n <ns> --create-namespace
    helm upgrade <release> <chart> -n <ns>
    helm rollback <release> <rev> -n <ns>

**Service Mesh (Istio)**
- Sidecar proxies (Envoy) + istiod.
- Traffic mgmt (retries, timeouts, canary), mTLS, telemetry.
- Canada tip: Many EKS shops use **AWS App Mesh** or **Istio**; expect questions on mTLS and canary routing.

**Kubernetes API (extensibility)**
- Admission webhooks (validation/mutation), CRDs, controllers.
- RBAC governs who can call which verbs on which resources.

---

## Security (day-1 answers)

- **Pod Security Standards (baseline/restricted)**; avoid privileged; drop capabilities; run as non-root.
- **RBAC** (least privilege). Prefer ServiceAccounts per workload.
- **NetworkPolicies** default-deny.
- **Secrets** encryption at rest + KMS (AWS KMS / GCP CMEK / Azure Key Vault).
- **Image security**: signed images, scanners (Trivy), pin tags by digest.
- **mTLS** via service mesh for east-west encryption.

---

## Cloud-Native Kubernetes

- **EKS (AWS `ca-central-1`)** is common; also AKS & GKE in Canadian regions (e.g., GCP `northamerica-northeast1`). Ask about **data residency** and **private clusters**.
- **Cluster Autoscaler** + **HPA** for cost/elasticity; spot/preemptible nodes for savings with PDBs and priority classes.
- **Ingress**: AWS Load Balancer Controller (ALB/NLB) on EKS is frequent.

---

## Debugging & Troubleshooting

**First steps**
    kubectl get events -n <ns> --sort-by=.lastTimestamp | tail
    kubectl describe pod <name> -n <ns>
    kubectl logs <pod> -n <ns> --all-containers --since=10m

**Into the pod**
    kubectl exec -it <pod> -n <ns> -- sh
    kubectl cp <ns>/<pod>:/path ./local

**Rollouts**
    kubectl rollout status deploy/<name> -n <ns>
    kubectl rollout undo deploy/<name> -n <ns>

**Node issues**
    kubectl get nodes
    kubectl describe node <node>
    kubectl top nodes/pods   (needs metrics-server)

---

## Cluster Administration

**RBAC**
- *Namespace-level*: `Role` + `RoleBinding`
- *Cluster-level*: `ClusterRole` + `ClusterRoleBinding` (nodes, PVs, CRDs)
- Impersonation to test:
    kubectl auth can-i list pods --as=system:serviceaccount:<ns>:<sa> -n <ns>

**Upgrades**
- Managed: upgrade control plane via provider; then upgrade node groups.
- Respect *version skew policies* (kubelet <= API server minor).

**CRDs**
- Treat like public APIs. Version (`v1alpha1` → `v1`), validation schema, status conditions.

---

## Monitoring & Logging

**Metrics Server**
- Resource metrics for `kubectl top` and HPA.
    kubectl top nodes
    kubectl top pods -A

**Prometheus/Grafana**
- Time-series metrics & dashboards; Alertmanager for paging.
- ServiceMonitor/PodMonitor if using Prometheus Operator.

**Logging**
- Fluent Bit/Vector/CloudWatch/Stackdriver; structure logs (JSON).

**Tracing**
- Jaeger/Tempo/Zipkin; mesh or SDK propagation of trace headers.

---

## Common Comparisons

**Services vs Ingress**
- Service exposes L4 in-cluster; Ingress is L7 HTTP routing + TLS from outside.

**Deployment vs StatefulSet**
- Deployment = stateless; replaceable pods.
- StatefulSet = stable ids/storage; ordered rollouts.

**HPA vs VPA**
| Item | HPA | VPA |
|---|---|---|
| Scales | replicas | CPU/mem requests/limits |
| Inputs | resource/custom metrics | historical/recent usage |
| Use | bursty web/API | batch/steady workloads |
| Together | Yes (with care) | Yes (recommendation or auto) |

**RBAC (Namespace vs Cluster)**
| Aspect | Namespace RBAC | Cluster RBAC |
|---|---|---|
| Objects | Role + RoleBinding | ClusterRole + ClusterRoleBinding |
| Scope | One namespace | All namespaces & cluster resources |
| Use cases | App SAs, team-limited access | Operators, SREs, dashboards |
| Risk | Lower | Higher—grant sparingly |

---

## Handy One-liners (show you can operate)

    # Scale
    kubectl -n <ns> scale deploy/<name> --replicas=3
    # Port-forward
    kubectl -n <ns> port-forward svc/<name> 8080:80
    # Live edit values in a ConfigMap and restart pods
    kubectl -n <ns> edit cm/<name>
    kubectl -n <ns> rollout restart deploy/<name>
    # Find pods not ready
    kubectl get pods -A --field-selector=status.phase!=Running -o wide
    # Get endpoints behind a Service
    kubectl -n <ns> get endpoints <svc>

---

## Talking Points (nice to mention)

- **Region & Residency**: “We deploy to `ca-central-1` (AWS) / `northamerica-northeast1` (GCP) for data residency. We encrypt at rest with KMS and enforce private endpoints.”
- **Cost Controls**: “We combine HPA + Cluster Autoscaler and spot nodes for stateless pools; PDBs & pod priority ensure graceful preemptions.”
- **Security Baseline**: “PSS restricted, image signing, NetworkPolicies default-deny, mTLS via mesh, RBAC least privilege, secrets backed by KMS.”



