# Kubernetes Core Concepts — Production-Focused Mini Project

This README explains **core Kubernetes concepts** in a real-world (production) way and demonstrates them with manifests:
- **Monolithic vs Microservices**
- **Kubernetes Architecture**
- **Setup on Local (minikube/kind) or AWS EC2/EKS**
- **kubectl**
- **Pods, Namespaces, Labels, Selectors, Annotations**

We will deploy a small **two-service microapp** (`api` + `web`) that uses labels/selectors/annotations and expose metrics annotations and environment-specific config. 

---

## 🧠 Core Concepts

### Monolithic vs Microservices
- **Monolith**
  - Single codebase/process, one deployable artifact.
  - ✅ Simple to start, easy local dev, fewer moving parts.
  - ❌ Hard to scale parts independently; releases are “all-or-nothing”; blast radius is large.
  - **Prod use**: Small teams, early stage products, tight coupling acceptable.
- **Microservices**
  - Separate services per business capability; independent deploy/scale/fail.
  - ✅ Scale hotspots (e.g., search) without scaling everything; faster independent releases; clearer ownership.
  - ❌ Operational complexity (networking, observability, data consistency, CI/CD, SLOs).
  - **Prod use**: Larger orgs, differing scaling profiles, clear domain boundaries, mature platform/ops.

### Kubernetes Architecture (Control Plane vs Nodes)
- **Control Plane**: `kube-apiserver` (front door/CRUD), `etcd` (state store), `kube-scheduler` (places Pods), `controller-manager` (reconciles desired state), `cloud-controller-manager` (cloud integrations).
- **Worker Nodes**: `kubelet` (Pod lifecycle), `kube-proxy` (Service VIPs), **container runtime** (e.g., containerd). Pods run here.
- **Prod tips**: Use a **managed control plane** (EKS/GKE/AKS). Enforce **Pod Security Standards**, **NetworkPolicies**, and **resource requests/limits**. Add **observability** (Prometheus/Grafana/OpenTelemetry), **ingress** with TLS, and **autoscaling** (HPA/VPA).

### Setup Options (Local vs AWS EC2/EKS)
- **Local** (fast feedback):
  - `minikube` / `kind` for development; DinD CI for smoke tests.
  - Pros: cheap, quick; Cons: not production-like networking/storage.
- **AWS**
  - **EKS (recommended)**: Managed control plane; node groups or Fargate. Add **aws-load-balancer-controller**, **Cluster Autoscaler**, **EBS CSI driver**.
  - **Self-managed on EC2**: Use `kubeadm` only if you must manage everything (higher ops load).

### kubectl (How we operate clusters)
- Declarative by default: `kubectl apply -f manifests/`.
- Introspection: `kubectl get/describe/logs/top/rollout`.
- Safety: `--namespace`, `--context`, RBAC; use `kubeconfig` per environment.

### Pods, Namespaces, Labels, Selectors, Annotations
- **Pods**: Smallest deployable unit (1+ containers sharing network/storage).
- **Namespaces**: Multi-tenant isolation; policy boundaries (RBAC, quotas, network).
- **Labels**: Key/value tags (e.g., `app: web`, `tier: frontend`, `env: prod`) used by **selectors**.
- **Selectors**: Match sets of objects by labels (Services select Pods; Deployments select Pods).
- **Annotations**: Non-identifying metadata for tooling (Prometheus scrapes, rollouts, links to CI/CD, owners).

---

## 🚀 What You’ll Deploy

A tiny microapp with two services:

- `api` — container echoes JSON on `/health` and `/` (simulated); exposes Prometheus annotations.
- `web` — NGINX serving static content, reverse-proxy to `api` via ClusterIP Service.
- Demonstrates:
  - **Namespaces** (`platform` for shared tools, `demo` for our app)
  - **Labels/Selectors**
  - **Resource requests/limits**
  - **Readiness/Liveness probes**
  - **ConfigMap/Secret** for environment config
  - **Ingress** (optional) and **port-forward** fallback
  - Production-flavored **annotations** (owners, links, metrics)

---

## 📁 Repo Layout

    manifests/
      00-ns-platform.yml
      00-ns-demo.yml
      01-cm-demo.yml
      02-secret-demo.yml
      03-deploy-api.yml
      04-svc-api.yml
      05-deploy-web.yml
      06-svc-web.yml
      07-ingress.yml          # optional (requires an ingress controller)
      90-networkpolicy.yml    # optional baseline deny-all + allow from web->api

---

## 1) Namespaces

Create **manifests/00-ns-platform.yml**:

    apiVersion: v1
    kind: Namespace
    metadata:
      name: platform
      labels:
        purpose: shared-tooling
        env: demo

Create **manifests/00-ns-demo.yml**:

    apiVersion: v1
    kind: Namespace
    metadata:
      name: demo
      labels:
        purpose: app
        env: demo

Apply:

    kubectl apply -f manifests/00-ns-platform.yml
    kubectl apply -f manifests/00-ns-demo.yml
    kubectl get ns --show-labels

**Why**: Isolate app/runtime policies and quotas by environment/team. In prod we also apply **LimitRanges**, **ResourceQuotas**, and **NetworkPolicies** per namespace.

---

## 2) Config & Secrets

Create **manifests/01-cm-demo.yml**:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app-config
      namespace: demo
      labels:
        app: microapp
        component: config
    data:
      APP_NAME: "microapp"
      APP_ENV: "demo"
      API_LOG_LEVEL: "info"
      WEB_BANNER: "Hello from web via ConfigMap"

Create **manifests/02-secret-demo.yml**:

    apiVersion: v1
    kind: Secret
    metadata:
      name: app-secret
      namespace: demo
      labels:
        app: microapp
        component: secret
    type: Opaque
    data:
      API_TOKEN: YXBpLXRva2VuLWRlbW8=   # base64("api-token-demo")

Apply:

    kubectl apply -f manifests/01-cm-demo.yml
    kubectl apply -f manifests/02-secret-demo.yml

**tip**: Back secrets with a KMS (AWS KMS via External Secrets/Sealed Secrets). Restrict via RBAC.

---

## 3) API Deployment (with labels, selectors, probes, annotations)

Create **manifests/03-deploy-api.yml**:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: api
      namespace: demo
      labels:
        app: microapp
        component: api
        tier: backend
      annotations:
        owner.team: "platform-api"
        repo.url: "https://github.com/example/microapp-api"
        runbook.url: "https://runbooks.example.com/microapp/api"
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: microapp
          component: api
      template:
        metadata:
          labels:
            app: microapp
            component: api
            tier: backend
          annotations:
            prometheus.io/scrape: "true"
            prometheus.io/port: "8080"
            prometheus.io/path: "/metrics"
        spec:
          containers:
          - name: api
            image: ghcr.io/example/fake-api:latest
            imagePullPolicy: IfNotPresent
            ports:
            - containerPort: 8080
            env:
            - name: APP_NAME
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_NAME
            - name: API_LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: API_LOG_LEVEL
            - name: API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: API_TOKEN
            resources:
              requests:
                cpu: "100m"
                memory: "128Mi"
              limits:
                cpu: "300m"
                memory: "256Mi"
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 10
              timeoutSeconds: 1
              failureThreshold: 3
            livenessProbe:
              httpGet:
                path: /healthz
                port: 8080
              initialDelaySeconds: 15
              periodSeconds: 20
              timeoutSeconds: 1
              failureThreshold: 3

**Why**:
- **Labels** identify app/component; **selector** binds Deployment to its Pods.
- **Annotations** advertise metrics for Prometheus and link ownership/runbooks.
- **Requests/Limits** give the scheduler & HPA/VPA useful signals.
- **Probes** protect availability during deploys and restarts.

Apply:

    kubectl apply -f manifests/03-deploy-api.yml
    kubectl rollout status deploy/api -n demo

---

## 4) API Service (selector demo)

Create **manifests/04-svc-api.yml**:

    apiVersion: v1
    kind: Service
    metadata:
      name: api
      namespace: demo
      labels:
        app: microapp
        component: api
    spec:
      selector:
        app: microapp
        component: api
      ports:
      - name: http
        port: 80
        targetPort: 8080
      type: ClusterIP

**Why**: The Service **selector** routes traffic to Pods with matching labels. ClusterIP is the internal service VIP used by other services (e.g., `web`).

Apply:

    kubectl apply -f manifests/04-svc-api.yml
    kubectl get svc -n demo

---

## 5) WEB Deployment (labels/selectors + config)

Create **manifests/05-deploy-web.yml**:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web
      namespace: demo
      labels:
        app: microapp
        component: web
        tier: frontend
      annotations:
        owner.team: "frontend-web"
        repo.url: "https://github.com/example/microapp-web"
        runbook.url: "https://runbooks.example.com/microapp/web"
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: microapp
          component: web
      template:
        metadata:
          labels:
            app: microapp
            component: web
            tier: frontend
        spec:
          containers:
          - name: nginx
            image: nginx:1.27-alpine
            ports:
            - containerPort: 80
            env:
            - name: WEB_BANNER
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: WEB_BANNER
            resources:
              requests:
                cpu: "50m"
                memory: "64Mi"
              limits:
                cpu: "200m"
                memory: "128Mi"
            readinessProbe:
              httpGet:
                path: /
                port: 80
              initialDelaySeconds: 5
              periodSeconds: 10
            livenessProbe:
              httpGet:
                path: /
                port: 80
              initialDelaySeconds: 15
              periodSeconds: 20

**note**: In real stacks, we template NGINX to proxy `/api` to `http://api.demo.svc.cluster.local` or we use a SPA and let Ingress/Service mesh route to `api`.

Apply:

    kubectl apply -f manifests/05-deploy-web.yml
    kubectl rollout status deploy/web -n demo

---

## 6) WEB Service

Create **manifests/06-svc-web.yml**:

    apiVersion: v1
    kind: Service
    metadata:
      name: web
      namespace: demo
      labels:
        app: microapp
        component: web
    spec:
      selector:
        app: microapp
        component: web
      ports:
      - name: http
        port: 80
        targetPort: 80
      type: ClusterIP

Apply:

    kubectl apply -f manifests/06-svc-web.yml
    kubectl get svc -n demo

---

## 7) Ingress (optional, needs an Ingress Controller)

Create **manifests/07-ingress.yml**:

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: microapp
      namespace: demo
      annotations:
        kubernetes.io/ingress.class: "nginx"
        cert-manager.io/cluster-issuer: "letsencrypt"
    spec:
      tls:
      - hosts:
        - demo.example.com
        secretName: demo-tls
      rules:
      - host: demo.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80

Apply:

    kubectl apply -f manifests/07-ingress.yml

**note**: On **EKS**, install **AWS Load Balancer Controller**; on local, enable **ingress-nginx**. Use **cert-manager** for TLS automation.

---

## 8) (Optional) NetworkPolicy (baseline security)

Create **manifests/90-networkpolicy.yml**:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: deny-by-default
      namespace: demo
    spec:
      podSelector: {}
      policyTypes: ["Ingress","Egress"]
      ingress: []
      egress: []

    ---
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-web-to-api
      namespace: demo
    spec:
      podSelector:
        matchLabels:
          app: microapp
          component: api
      policyTypes: ["Ingress"]
      ingress:
      - from:
        - podSelector:
            matchLabels:
              app: microapp
              component: web
        ports:
        - protocol: TCP
          port: 8080

Apply:

    kubectl apply -f manifests/90-networkpolicy.yml

**Why**: Default-deny reduces blast radius; then allow only what’s needed.

---

## ▶️ Run & Verify

    # Apply everything
    kubectl apply -f manifests/

    # See what’s running
    kubectl get all -n demo

    # Test internal reachability from a test pod
    kubectl -n demo run tester --image=busybox:1.36 --restart=Never -it -- sh
    # inside:
    wget -qO- http://web.demo.svc.cluster.local
    wget -qO- http://api.demo.svc.cluster.local/healthz
    exit

    # Port-forward web locally if no ingress
    kubectl -n demo port-forward svc/web 8080:80
    # open http://localhost:8080

    # Observe labels/selectors
    kubectl get pods -n demo --show-labels
    kubectl get svc -n demo --show-labels

    # Observe annotations
    kubectl get pod -n demo -l component=api -o jsonpath='{.items[0].metadata.annotations}'

---

## 🔎 Resource Explanations (What & Why)

- **Namespace**: Multi-tenant boundary for RBAC, quotas, policies. Prod: split by env (`prod`, `staging`), team, or app domain.
- **Deployment**: Declarative manager for **stateless** pods; handles rolling updates/rollbacks. Prod: use **readiness/liveness** probes, **resource requests/limits**, **pod disruption budgets**, and **autoscaling**.
- **Service (ClusterIP)**: Stable virtual IP for pods matched by **selector**. Prod: prefer **ClusterIP** internally; use **Ingress/ALB** for external traffic.
- **Labels & Selectors**: Identity and grouping. Prod: adopt a label schema (e.g., `app`, `component`, `tier`, `version`, `env`) to power routing, policies, and dashboards.
- **Annotations**: Tooling metadata (Prometheus, CI/CD links, owners, SLOs). Non-identifying; not used for selection.
- **ConfigMap/Secret**: Twelve-factor config; Secrets for credentials/tokens (encrypt at rest; restrict via RBAC).
- **NetworkPolicy**: Firewall-like controls. Prod: default-deny and allow narrowly.
- **Ingress**: HTTP routing + TLS termination. Prod: automate certs with **cert-manager**, use managed LB (ALB/NLB).
- **Probes**: Health checks for resilient rollouts and self-healing.

---

## 🏭 Production Hardening Checklist (Quick)

- **Security**: Pod Security Standards, NetworkPolicies, image signing/scanning, non-root containers.
- **Resources**: Requests/Limits everywhere; HPA (CPU/memory/custom), VPA for baseline.
- **Reliability**: Readiness+Liveness, PodDisruptionBudgets, multi-AZ nodes, Cluster Autoscaler.
- **Observability**: Prometheus/Grafana + Alertmanager; logs to ELK/CloudWatch; tracing (OTel).
- **Config**: Externalize via ConfigMaps/Secrets; use ExternalSecrets/Parameter Store.
- **Delivery**: GitOps (Argo CD/Flux) or CI/CD pipelines; progressive delivery (Argo Rollouts).
- **Backups**: etcd (if self-managed), app data (snapshots), manifests in Git.

---

## 🧹 Cleanup

    kubectl delete -f manifests/90-networkpolicy.yml --ignore-not-found
    kubectl delete -f manifests/07-ingress.yml --ignore-not-found
    kubectl delete -f manifests/06-svc-web.yml
    kubectl delete -f manifests/05-deploy-web.yml
    kubectl delete -f manifests/04-svc-api.yml
    kubectl delete -f manifests/03-deploy-api.yml
    kubectl delete -f manifests/02-secret-demo.yml
    kubectl delete -f manifests/01-cm-demo.yml
    kubectl delete -f manifests/00-ns-demo.yml
    kubectl delete -f manifests/00-ns-platform.yml

---

## ✅ Summary

 Deployed a small microapp that showcases **Namespaces**, **Pods**, **Deployments**, **Services**, **Ingress**, **Labels/Selectors**, **Annotations**, **ConfigMaps/Secrets**, and optional **NetworkPolicies** .
---
