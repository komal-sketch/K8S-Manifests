# Kubernetes Cheat‑Sheet (2025)  

This cheat‑sheet summarizes core concepts and common interview topics for **Kubernetes (K8s)**.  

---

## 1. Core Concepts

### Monolithic vs Microservices

| Architecture | Description | Interview Pointers |
| --- | --- | --- |
| **Monolithic** | All components of an application (UI, business logic, database access) are built and deployed together. Scaling requires deploying a full replica. | Easy to develop initially; hard to scale; a failure may bring down the whole app. |
| **Microservices** | Application is broken into small, independently deployable services communicating via APIs (usually REST/gRPC). Each service can scale and be updated independently. | Encourages modularity, resilience and DevOps practices. Kubernetes is designed to manage microservices. |

### Kubernetes Architecture

Kubernetes is a **container orchestration platform** built on the control plane and worker nodes:

| Component | Purpose |
| --- | --- |
| **Control plane** | Manages the cluster state and schedules workloads. Key components include the API server, etcd (persistent store), scheduler, controller manager and cloud controller manager. |
| **Worker nodes** | Run workloads. Each node runs kubelet (agent that talks to the API server), container runtime (e.g., containerd), and kube‑proxy for networking. |

**Setup on local/cloud**: In interviews you may be asked how to bootstrap a cluster. Tools include:

* **Minikube** or **kind** for local development. They run a single‑node cluster using virtual machines or Docker.  
* **Kubeadm** for bootstrapping production clusters.  
* Managed services (**EKS**, **AKS**, **GKE**) simplify control plane management—see [Cloud‑Native Kubernetes](#8-cloud-native-kubernetes).

### `kubectl` basics

`kubectl` communicates with the API server. Useful commands:

    kubectl create -f pod.yaml       # create object from YAML
    kubectl apply -f deployment.yaml  # create/update using declarative config
    kubectl get pods                  # list pods in the current namespace
    kubectl describe pod <pod>        # show details and events
    kubectl delete svc myservice      # delete a service
    kubectl logs -f <pod>             # stream logs for a pod/container
    kubectl exec -it <pod> -- bash    # execute a shell in a container

### Pods

The **pod** is the smallest deployable unit: one or more containers that share storage and network resources. Pods are generally managed by higher‑level controllers such as Deployments or StatefulSets. A simple pod manifest includes metadata (name/labels), a spec (containers/images/resources), and sometimes volumes, tolerations and other settings.

**Example – minimal pod definition**:

    apiVersion: v1
    kind: Pod
    metadata:
      name: my‑nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80

### Namespaces, Labels, Selectors and Annotations

* **Namespaces** logically isolate resources within a cluster. Use `kubectl get ns` to list them. You can create a namespace with `kubectl create ns team‑a`.  
* **Labels** are key‑value pairs attached to objects for grouping and selecting (e.g., `app=web`, `env=prod`).  
* **Selectors** (label selectors) allow controllers or services to match objects. For example, a service selects pods where `app=frontend`.  
* **Annotations** attach non‑identifying metadata (e.g., build info, contact) to objects.

Example pod with labels and annotations:

    apiVersion: v1
    kind: Pod
    metadata:
      name: labeled‑pod
      labels:
        app: web
        tier: frontend
      annotations:
        description: "Demonstrates labels and annotations"
    spec:
      containers:
      - name: alpine
        image: alpine:3.19
        command: ["sh", "-c", "echo Hello"]

---

## 2. Workloads

Kubernetes provides several **controllers** to manage pods and ensure the desired state:

| Workload | Use case | Key features |
| --- | --- | --- |
| **ReplicaSet** | Ensures a specified number of pod replicas are running at any time. Usually managed by Deployments. | Use when you need simple replication but no rollout management. |
| **Deployment** | Declarative update for ReplicaSets; manages rollouts/rollbacks. Ideal for stateless applications. | You can use `kubectl rollout status deployment/myapp` to monitor a rollout. |
| **StatefulSet** | Manages stateful applications with unique identities and stable persistent storage (e.g., databases). | Provides stable hostnames (`pod‑0`, `pod‑1`), ordered deployments and scaling. |
| **DaemonSet** | Ensures one pod per node (or subset of nodes) for system services like logging or monitoring agents. | Use node selectors/taints to target specific nodes. |
| **Job** | Creates one or more pods to run to completion. | Suitable for batch tasks. |
| **CronJob** | Schedules Jobs to run at specified times (cron syntax). | Great for periodic tasks like backups. |

**Example – Deployment**:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web‑deployment
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          containers:
          - name: web
            image: nginx:1.25
            ports:
            - containerPort: 80

**Rolling updates & rollbacks**: Deployments create new ReplicaSets and gradually scale down old ones. You can pause/resume and roll back with `kubectl rollout undo deployment/web‑deployment`.

---

## 3. Networking

### Cluster Networking

Kubernetes requires a **flat, inter‑pod network**: every pod can reach every other pod without NAT. This is implemented by a **Container Network Interface (CNI)** plugin such as Calico, Flannel, or Cilium. Many managed services (EKS, AKS, GKE) provision the CNI for you.

### Services

Services abstract access to a set of pods and provide a stable network identity. Types include:

| Type | Description | Example |
| --- | --- | --- |
| **ClusterIP** | Default. Exposes the service on an internal IP accessible inside the cluster. | Ideal for internal microservices. |
| **NodePort** | Exposes the service on a static port on each node. | Use for quick external access for testing. |
| **LoadBalancer** | Provisions an external load balancer using cloud provider integration. | Common for production on EKS/AKS/GKE. |
| **ExternalName** | Maps the service to an external DNS name. | Useful when migrating to microservices. |

Service manifest example (ClusterIP):

    apiVersion: v1
    kind: Service
    metadata:
      name: web‑service
    spec:
      selector:
        app: web
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: ClusterIP

### Ingress

Ingress exposes HTTP(S) services externally and can provide features like path‑based routing and TLS termination. Ingress resources are implemented by ingress controllers (e.g., Nginx, Traefik, AWS ALB). Example:

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: web‑ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite‑target: /
    spec:
      rules:
        - host: example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: web‑service
                    port:
                      number: 80

### Network Policies

Network policies control how pods communicate with each other and with external endpoints. By default, all traffic is allowed. A pod becomes restricted only when a policy selects it. Policies use selectors and IP blocks to allow or deny traffic. For example, restrict database pods to accept traffic only from application pods:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: db‑allow‑from‑app
    spec:
      podSelector:
        matchLabels:
          app: database
      policyTypes:
        - Ingress
      ingress:
      - from:
        - podSelector:
            matchLabels:
              app: app
        ports:
        - protocol: TCP
          port: 5432

---

## 4. Storage

### Persistent Volumes (PV) & Persistent Volume Claims (PVC)

Persistent storage is decoupled from pods using the **PV/PVC** model:

* A **PersistentVolume** is a cluster resource representing storage (EBS, Azure Disk, NFS, etc.) configured by the admin.  
* A **PersistentVolumeClaim** requests storage of a particular size and access mode. When a claim is bound, it reserves a PV.  
* **StorageClasses** define dynamic provisioning parameters (e.g., EBS gp3 vs io2).  

Example of dynamic provisioning:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: standard
    provisioner: kubernetes.io/aws‑ebs
    parameters:
      type: gp3

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: data‑pvc
    spec:
      storageClassName: standard
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi

### ConfigMaps & Secrets

* **ConfigMaps** store non‑sensitive configuration such as environment variables or config files. They are plain text and can be consumed as environment variables or mounted as volumes.  
* **Secrets** store sensitive data (passwords, tokens) encoded in base64. Access should be restricted (RBAC).  

Example of a Secret and a Pod consuming it:

    apiVersion: v1
    kind: Secret
    metadata:
      name: db‑secret
    type: Opaque
    data:
      username: bXl1c2Vy  # base64("myuser")
      password: bXlwYXNz  # base64("mypass")

    apiVersion: v1
    kind: Pod
    metadata:
      name: db‑client
    spec:
      containers:
      - name: app
        image: alpine
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db‑secret
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db‑secret
              key: password

* To enhance confidentiality, secrets at rest can be encrypted using an `EncryptionConfiguration` on the API server. Example with AES‑CBC provider:

    kind: EncryptionConfiguration
    apiVersion: apiserver.config.k8s.io/v1
    resources:
      - resources: ["secrets"]
        providers:
          - aescbc:
              keys:
                - name: key1
                  secret: <base64‑encoded‑32‑byte‑key>
          - identity: {}

Then start the API server with `--encryption-provider-config=/etc/kubernetes/enc.yaml`. Without this, secrets are stored unencrypted.

---

## 5. Scaling & Scheduling

### Resource Requests and Limits

Containers declare their CPU and memory requirements. **Requests** are guaranteed resources used by the scheduler; **limits** cap usage to prevent noisy neighbours. When a container exceeds a memory limit it is OOM‑killed; exceeding CPU triggers throttling.

Example specifying requests and limits:

    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"

### Horizontal & Vertical Pod Autoscalers (HPA/VPA)

* **HPA** automatically scales the number of pod replicas based on CPU/memory or custom metrics. Use `kubectl autoscale deployment web --cpu-percent=50 --min=2 --max=5`. The metrics‑server must be running:contentReference[oaicite:21]{index=21}.
* **VPA** adjusts resource **requests** up/down for pods. It is useful for right‑sizing workloads but is less common in production as of 2025.
* **Cluster autoscaler** increases or decreases node count based on pod scheduling needs. On managed platforms, enable it via cluster configuration.

### Affinity & Anti‑Affinity

Define rules that influence where pods are scheduled:

* **Node selectors / Node affinity**: `nodeSelector` requires pods to run on nodes with matching labels. Node affinity supports `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution` rules for more flexibility.
* **Pod affinity/anti‑affinity**: Encourage (affinity) or forbid (anti‑affinity) colocating pods on the same node or topology domain (e.g., same zone). Useful for placing components that need proximity or separation.

### Taints & Tolerations

Taints mark nodes as unsuitable for certain workloads; tolerations on pods allow them to schedule onto tainted nodes. For example, taint GPU nodes with `nvidia.com/gpu=true:NoSchedule` and add a toleration to pods that require GPUs.

### Pod Disruption Budget (PDB)

Defines how many pods of a collection can be down during voluntary disruptions (drain, upgrade). PDB ensures high availability when performing maintenance. Example:

    apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: web‑pdb
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          app: web

### Probes (liveness, readiness, startup)

* **Liveness probe** detects if a container is stuck; failing containers are restarted.  
* **Readiness probe** indicates when a container is ready to serve traffic; used by Services.  
* **Startup probe** waits for application startup before failing the container (useful for slow starts).  
Probes can use HTTP, TCP or exec commands. Example HTTP liveness probe:

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10

### Resource Quotas

Namespaces can be limited via **ResourceQuota** objects to avoid resource exhaustion. Example limiting CPU, memory and persistent storage:

    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: team‑quota
      namespace: team‑a
    spec:
      hard:
        requests.cpu: "4"
        requests.memory: "8Gi"
        limits.cpu: "8"
        limits.memory: "16Gi"
        persistentvolumeclaims: "10"

---

## 6. Advanced Features

### Operators

An **Operator** is a software extension that uses Kubernetes APIs and controllers to automate day‑1/day‑2 operations (install, upgrade, backup) for complex applications. Operators define a **Custom Resource Definition (CRD)** to manage application instances (e.g., a Postgres cluster). Benefits include:

* Declarative management of stateful apps (databases, message queues).
* Automatically handles scaling, failover and upgrades.
* Extends Kubernetes beyond stateless workloads.

### Helm

Helm is a package manager for Kubernetes. A **Helm chart** bundles multiple related manifests with templating, enabling parameterization and reuse:contentReference[oaicite:27]{index=27}. Key features:

* Simplifies deployment and upgrade of complex apps (e.g., nginx‑ingress, Prometheus).  
* Supports versioning and rollback.  
* Charts are stored in repositories (`helm repo add`), installed with `helm install`, upgraded with `helm upgrade`, and uninstalled with `helm uninstall`.  
* Encourages standardized release processes and integrates with CI/CD:contentReference[oaicite:28]{index=28}.

### Service Mesh

A **service mesh** provides service‑to‑service communication features like traffic management, observability and security. Popular meshes include **Istio**, **Linkerd** and **Consul**. Service meshes typically run sidecar proxies in pods to handle:

* Secure mutual TLS between services.  
* Traffic routing, retries and circuit breaking.  
* Metrics and tracing integration.

While beneficial, they add complexity; evaluate if built‑in features (Ingress, NetworkPolicy) suffice.

### Kubernetes API & CRDs

Kubernetes exposes a RESTful API. You can create **Custom Resource Definitions (CRDs)** to extend Kubernetes with new resource types. Example CRD snippet:

    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: cronjobs.example.com
    spec:
      group: example.com
      names:
        kind: CronJob
        plural: cronjobs
      scope: Namespaced
      versions:
      - name: v1
        served: true
        storage: true
        schema:
          openAPIV3Schema:
            type: object
            properties:
              spec:
                type: object
                properties:
                  schedule:
                    type: string

After defining the CRD you implement a controller (often with the `operator‑sdk` or Kubebuilder) that reconciles the custom resource.

---

## 7. Security

### Pod Security Standards & Admission

Since Kubernetes 1.25 the deprecated **PodSecurityPolicy** has been replaced by **Pod Security Admission (PSA)**. PSA enforces **Pod Security Standards (PSS)** by applying one of three profiles—*privileged*, *baseline* or *restricted*—at the namespace level.  
* **Privileged**: little/no restrictions (e.g., host access).  
* **Baseline**: prevents known privilege escalations (e.g., prohibits hostPath).  
* **Restricted**: enforces strict security (e.g., non‑root, read‑only root filesystem).  
Configure via the `pod-security.kubernetes.io/<profile>=enforce|audit|warn` namespace labels.

### RBAC

Role‑Based Access Control defines permissions. Use `Roles`/`ClusterRoles` and `RoleBindings`/`ClusterRoleBindings` to grant verbs (get, list, watch, create, delete) on resources. Example: grant read‑only access to pods in `dev` namespace:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: pod‑reader
      namespace: dev
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list", "watch"]

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: read‑pods
      namespace: dev
    subjects:
    - kind: User
      name: alice
    roleRef:
      kind: Role
      name: pod‑reader
      apiGroup: rbac.authorization.k8s.io

### Image Scanning & Supply Chain Security

* Use trusted container registries and avoid running images tagged `latest`.  
* Scan images for vulnerabilities (e.g., Trivy, Clair).  
* Sign images using tools like Cosign and validate signatures with admission controllers (e.g., Kyverno, OPA).  
* Regularly update base images and dependencies.

### Network Policies & Zero Trust

Enforce least privilege by default deny network policies (see [Network Policies](#network-policies)). Combine with service meshes or mTLS for encryption in transit.

### Secrets & Encryption

Follow best practices: use **Kubernetes Secrets** rather than environment variables; enable encryption at rest (see [Secrets](#storage)). Consider external secret managers (e.g., AWS Secrets Manager, HashiCorp Vault) with CSI drivers or external secrets operators.

### Other Security Considerations

* **APIs & Admission Webhooks**: Use validating/admission webhooks to enforce policies and guardrails.  
* **Audit logs**: Enable audit logging on the API server to track changes.  
* **Node hardening**: Use minimal OS images (e.g., Bottlerocket) and restrict SSH access.  
* **Rotate credentials**: Regularly rotate certificates, service account tokens and kubeconfig credentials.

---

## 8. Cloud‑Native Kubernetes

Managed services offer simplified cluster management and integrate with cloud provider features:

| Service | Provider | Notes |
| --- | --- | --- |
| **Amazon EKS** | AWS | Managed control plane; supports IAM integration and Fargate for serverless pods. |
| **Azure AKS** | Microsoft | Integrated with Azure AD; dynamic node pools; supports Windows containers. |
| **Google GKE** | Google Cloud | Autopilot mode handles node management; built‑in Cloud Logging & Monitoring. |

### Cluster Autoscaler

The cluster autoscaler adds/removes nodes based on pending pods. Most managed services have a built‑in autoscaler; you enable it via cluster settings or node group configuration. Understand the trade‑offs: scaling lag vs. capacity waste.

### Spot/Preemptible Nodes

Cloud providers offer discounted **spot** (AWS/GCP) or **preemptible** (GCP) instances. Using them for non‑critical workloads can reduce costs. In EKS/AKS, use node groups or Virtual Node Pools with different priorities. Combine with pod anti‑affinity or taints/tolerations so critical workloads avoid preemptible nodes.

### Hybrid & Multi‑Region Considerations (Canada context)

Canada’s data sovereignty laws (e.g., PIPEDA) may require data residency in Canadian regions. When interviewing for Canadian roles:

* Be aware of the **regional availability zones** in provinces (e.g., *ca‑central‑1* in AWS).  
* Highlight **high availability** across multiple regions to survive regional outages (e.g., using `topology.kubernetes.io/zone`).  
* Discuss **disaster recovery** and backup strategies (e.g., cross‑region replication for etcd or databases).  
* Cost‑optimization using **spot instances** may be beneficial if service level agreements allow brief interruptions.

---

## 9. Debugging & Troubleshooting

When troubleshooting pods or nodes, start with `kubectl` and cluster events:

* **Logs**: `kubectl logs <pod> [-c <container>] [--tail=50]`. Use `-f` to stream logs.  
* **Describe**: `kubectl describe pod <pod>` shows detailed pod status, events, and reasons for failures.  
* **Exec**: `kubectl exec -it <pod> -- /bin/sh` opens a shell.  
* **List failed pods**: `kubectl get pods --field-selector=status.phase=Failed`.

**kubectl debug**: Since v1.18, `kubectl debug` can attach an **ephemeral container** to a running pod for investigation. For example, to debug a busybox container with a shell:

    kubectl debug -it <pod> --image=busybox --target=app

**Node issues**: `kubectl describe node <node>` shows conditions and resource usage. Use `journalctl -u kubelet` on the node for kubelet logs.

**Resource usage**: Install the **metrics‑server** to enable `kubectl top pods` and `kubectl top nodes` for CPU/memory metrics:contentReference[oaicite:37]{index=37}. For advanced debugging, use profiling tools (e.g., `go tool pprof`) and container runtime logs.

**Common errors**:

| Error state | Possible cause |
| --- | --- |
| *ImagePullBackOff/ErrImagePull* | Image does not exist, wrong registry credentials or network issues. |
| *CrashLoopBackOff* | Container process keeps crashing (bad command, missing env vars). Use `kubectl logs` and `kubectl describe`. |
| *Pending* | Insufficient resources or unsatisfied node selectors/taints. Check node capacity and scheduling constraints. |
| *OOMKilled* | Container exceeded memory limit. Adjust limits or investigate memory leaks. |

---

## 10. Cluster Administration

### Cluster Upgrades

Upgrading Kubernetes involves draining nodes, backing up etcd, upgrading control plane components and worker nodes sequentially. Steps:

1. **Drain worker nodes**: `kubectl drain <node> --ignore-daemonsets --delete-local-data`.
2. **Backup etcd** and cluster state (e.g., `etcdctl snapshot save`).  
3. **Upgrade control plane**: Follow distribution instructions (e.g., `apt-get upgrade kube-apiserver kube-controller-manager`).  
4. **Upgrade kubelet and kube-proxy on worker nodes** one by one.  
5. **Re‑enable scheduling**: `kubectl uncordon <node>`.  
6. **Upgrade add‑ons**: Ingress controllers, CSI drivers, network plugins.

### Custom Resource Definitions (CRDs)

CRDs extend the Kubernetes API with new resource types (see [Kubernetes API & CRDs](#kubernetes-api--crds)). For administration, keep track of CRD versions and update them during upgrades. Use `kubectl api-resources` to list all resources and `kubectl get <crd>` to view custom objects.

### RBAC & ClusterRoles

Beyond per‑namespace roles, cluster administrators create **ClusterRoles** for cluster‑wide resources (nodes, persistentvolumes) and bind them with **ClusterRoleBindings**. Example granting cluster‑read to a group:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: cluster‑viewer
    rules:
    - apiGroups: [""]
      resources: ["pods", "services", "nodes"]
      verbs: ["get", "list", "watch"]

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: view‑all
    subjects:
    - kind: Group
      name: developers
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: cluster‑viewer
      apiGroup: rbac.authorization.k8s.io

---

## 11. Monitoring & Logging

### Metrics Server & HPA

Install the **metrics‑server** to collect CPU and memory usage from kubelets. It exposes metrics to the Metrics API used by HPAs and `kubectl top`:contentReference[oaicite:40]{index=40}. Without the metrics‑server, HPAs cannot scale based on resource metrics.

### Prometheus & Grafana

Prometheus scrapes metrics from Kubernetes components (API server, kubelet) and applications. Grafana visualizes these metrics. Many charts are available via Helm (`kube‑prometheus‑stack`). Key metrics:

* API server request rate and latency.  
* etcd disk I/O and DB size.  
* Node CPU/memory usage.  
* Application‑specific metrics via exporters (e.g., Node Exporter, cAdvisor).

### Logging

* **Structured logging**: Use JSON output and include contextual fields (request ID, user).  
* **Log aggregation**: Fluentd/Fluent Bit or Logstash collect logs, send to stores such as **Loki**, **Elasticsearch**, or **CloudWatch**.  
* **Observability**: Combine metrics, logs and traces (e.g., Jaeger or OpenTelemetry) for distributed tracing.

### ELK Stack & Loki

* The **ELK stack** (Elasticsearch, Logstash, Kibana) is widely used for log ingestion and search.  
* **Loki + Promtail + Grafana** is a lightweight alternative optimized for Kubernetes. It stores logs with labels (e.g., pod, namespace) and integrates with Grafana.

### Kubernetes Dashboard

The official web UI provides a graphical overview of resources. Use it carefully (requires RBAC & bearer token). In managed clusters, provider consoles (e.g., AWS EKS console) often offer similar insights.

---

## 12. Additional Tips

* **Emphasize cloud expertise**: Many Canadian companies adopt **EKS** or **GKE** due to multi‑region redundancy (Toronto/Montreal). Highlight familiarity with provider‑specific features (e.g., IAM roles for service accounts in AWS, Workload Identity in GCP).  
* **Regulatory awareness**: Understand data residency requirements (PIPEDA, provincial laws) and how to ensure resources remain in designated regions.  
* **Cost optimization**: Spot instances are attractive for cost savings in Canada’s relatively expensive cloud regions. Show understanding of balancing reliability vs. cost.  
* **French/English**: Some roles require bilingual support; mention any language skills.  
* **Soft skills**: Employers value the ability to explain complex concepts to stakeholders. Use analogies (e.g., pods are like shipping containers; deployments are like blueprints) during interviews.

---

## 13. Frequently Asked Commands

| Task | Command |
| --- | --- |
| Get all resources in a namespace | `kubectl get all -n <ns>` |
| Stream logs from the last failing pod | `kubectl logs --previous <pod>` |
| Check cluster components | `kubectl get componentstatuses` |
| View API resources & versions | `kubectl api-resources` & `kubectl api-versions` |
| Port‑forward service/pod | `kubectl port-forward svc/web 8080:80` |
| Apply a manifest directory recursively | `kubectl apply -k <dir>` (Kustomize) |
| Delete pods stuck in terminating | `kubectl delete pod <pod> --force --grace-period=0` |
| Show resource usage | `kubectl top pod` / `kubectl top node` |

---

## 14. Conclusion

This cheat‑sheet covers core Kubernetes concepts, advanced features, security, cloud‑native considerations, and debugging techniques. In interviews, be prepared to discuss trade‑offs and real‑world scenarios (e.g., how to handle stateful workloads, optimize cost, enforce security policies). Use the commands and examples here as a reference to articulate your understanding of the Kubernetes ecosystem.

