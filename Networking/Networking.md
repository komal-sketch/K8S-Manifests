# Kubernetes Networking — Mini Project

- **Cluster Networking** (Pod CIDRs, CNI, DNS, kube-proxy)
- **Services** (ClusterIP, NodePort, LoadBalancer)
- **Ingress** (HTTP routing + TLS)
- **Network Policies** (zero-trust east-west)
- **Metrics Server** (resource metrics for `kubectl top`, HPA, dashboards)


---

## 🧠 Concepts (What & Why)

### Cluster Networking
- **CNI Plugin** (Calico, Cilium, AWS VPC CNI, etc.) provides **Pod IPs** and **Pod-to-Pod routing** across nodes. Every Pod gets a **routable IP** in the **Pod CIDR**; no NAT between Pods.
- **kube-proxy** programs Service VIPs (iptables/ipvs) so a **Service** stable virtual IP load-balances to matching Pods.
- **CoreDNS** gives service discovery (`svc.ns.svc.cluster.local`).
- **Prod tips**
  - Choose a CNI that matches your **cloud/VPC** model and **NetworkPolicy** needs (e.g., Cilium for eBPF, Calico for policy scale).
  - Allocate **non-overlapping CIDRs** for Pods/Services that fit future growth.
  - Enforce **NetworkPolicies** at namespace/app level (default deny).
  - Run **Pod Security Standards** and **Node security groups/SGs**.

### Services
- **ClusterIP**: internal VIP for east-west traffic.
- **NodePort**: opens a port on each node; useful for lab/testing or bare metal.
- **LoadBalancer**: managed cloud LB (ELB/ALB/NLB, etc.) fronting a Service.
- **Headless** (`clusterIP: None`): no VIP; returns Pod IPs for stateful sets.

### Ingress
- L7 HTTP(S) routing and TLS termination. You install an **Ingress Controller** (nginx, contour, haproxy, AWS Load Balancer Controller, etc.). In prod, automate TLS with **cert-manager** and watch for WAF/OWASP controls.

### Network Policies
- Like **firewall rules for Pods**. Default-deny + explicit allow drastically reduces blast radius. CNI must support it (Calico/Cilium do; AWS VPC CNI supports via extra components).

### Metrics Server
- Lightweight aggregator of **resource usage** (CPU/memory) from Kubelets.
- Powers `kubectl top`, **HPA** (resource metrics), and dashboards.
- **Prod use cases**: autoscaling signals (HPA), capacity planning, SLO/SLA burn alerts (paired with Prometheus for custom metrics).
- Not a time-series DB. For historical/alerting, deploy **Prometheus**; metrics-server is complementary.

---

## 📁 Suggested Repo Layout

    manifests/
      00-namespace.yml
      01-deploy-api.yml
      02-svc-api.yml
      03-deploy-web.yml
      04-svc-web.yml
      05-ingress.yml              # optional (requires an Ingress Controller)
      06-netpol-default-deny.yml
      07-netpol-allow-web-to-api.yml
      10-metrics-server-notes.md  # notes/how-to install (vendor-specific)

The manifests demonstrate labels/selectors, service discovery, ingress routing, and zero-trust east-west with NetworkPolicies.

---

## 1) Namespace

Create **manifests/00-namespace.yml**:

    apiVersion: v1
    kind: Namespace
    metadata:
      name: net-demo
      labels:
        app: net-demo
        env: demo

Apply:

    kubectl apply -f manifests/00-namespace.yml

---

## 2) Backend API Deployment + Service (ClusterIP)

Create **manifests/01-deploy-api.yml**:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: api
      namespace: net-demo
      labels:
        app: net-demo
        component: api
        tier: backend
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: net-demo
          component: api
      template:
        metadata:
          labels:
            app: net-demo
            component: api
            tier: backend
        spec:
          containers:
          - name: api
            image: ghcr.io/knative/helloworld-go:latest   # simple HTTP container
            env:
            - name: TARGET
              value: "Hello from API"
            ports:
            - containerPort: 8080
            resources:
              requests: { cpu: "100m", memory: "128Mi" }
              limits:   { cpu: "300m", memory: "256Mi" }
            readinessProbe:
              httpGet: { path: "/", port: 8080 }
              initialDelaySeconds: 5
            livenessProbe:
              httpGet: { path: "/", port: 8080 }
              initialDelaySeconds: 15

Create **manifests/02-svc-api.yml**:

    apiVersion: v1
    kind: Service
    metadata:
      name: api
      namespace: net-demo
      labels:
        app: net-demo
        component: api
    spec:
      selector:
        app: net-demo
        component: api
      ports:
      - name: http
        port: 80
        targetPort: 8080
      type: ClusterIP

Apply:

    kubectl apply -f manifests/01-deploy-api.yml
    kubectl apply -f manifests/02-svc-api.yml
    kubectl rollout status deploy/api -n net-demo
    kubectl get svc -n net-demo

**Explanation**
- The **Service** gives a stable VIP `api.net-demo.svc.cluster.local` that load-balances across API Pods chosen by the **selector**.

---

## 3) Frontend Web Deployment + Service

Create **manifests/03-deploy-web.yml**:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web
      namespace: net-demo
      labels:
        app: net-demo
        component: web
        tier: frontend
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: net-demo
          component: web
      template:
        metadata:
          labels:
            app: net-demo
            component: web
            tier: frontend
        spec:
          containers:
          - name: nginx
            image: nginx:1.27-alpine
            ports:
            - containerPort: 80
            volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
            - name: conf
              mountPath: /etc/nginx/conf.d
          volumes:
          - name: html
            projected:
              sources:
              - configMap:
                  name: web-html
                  items:
                  - key: index.html
                    path: index.html
          - name: conf
            projected:
              sources:
              - configMap:
                  name: web-conf
                  items:
                  - key: default.conf
                    path: default.conf

Create the ConfigMaps for NGINX HTML and reverse-proxy:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: web-html
      namespace: net-demo
    data:
      index.html: |
        <html><body>
        <h1>Net Demo Web</h1>
        <p>Try hitting /api</p>
        </body></html>
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: web-conf
      namespace: net-demo
    data:
      default.conf: |
        server {
          listen 80;
          location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
          }
          location /api {
            proxy_pass         http://api.net-demo.svc.cluster.local;
            proxy_set_header   Host $host;
            proxy_http_version 1.1;
          }
        }

Create **manifests/04-svc-web.yml**:

    apiVersion: v1
    kind: Service
    metadata:
      name: web
      namespace: net-demo
      labels:
        app: net-demo
        component: web
    spec:
      selector:
        app: net-demo
        component: web
      ports:
      - name: http
        port: 80
        targetPort: 80
      type: ClusterIP

Apply and verify:

    kubectl apply -f manifests/03-deploy-web.yml
    kubectl apply -f manifests/04-svc-web.yml
    kubectl rollout status deploy/web -n net-demo
    kubectl get svc -n net-demo

**Explanation**
- The **web** Deployment reverse-proxies `/api` to the **api Service DNS**. This is classic **east-west** traffic via **ClusterIP**.

---

## 4) Ingress (HTTP + TLS)

Create **manifests/05-ingress.yml** (requires an Ingress Controller such as ingress-nginx, or AWS Load Balancer Controller):

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: net-demo
      namespace: net-demo
      annotations:
        kubernetes.io/ingress.class: "nginx"
        # TLS via cert-manager (optional)
        cert-manager.io/cluster-issuer: "letsencrypt"
    spec:
      tls:
      - hosts:
        - netdemo.example.com
        secretName: netdemo-tls
      rules:
      - host: netdemo.example.com
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

    kubectl apply -f manifests/05-ingress.yml

**Production**
- Use **ALB (AWS)** or **nginx** with **WAF**, enable **TLS 1.2+**, HSTS, and mTLS if needed.
- Centralize routing rules and canaries (Argo Rollouts/Flagger).

---

## 5) Network Policies (Zero-Trust)

Create **manifests/06-netpol-default-deny.yml**:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: default-deny-all
      namespace: net-demo
    spec:
      podSelector: {}
      policyTypes: ["Ingress","Egress"]
      ingress: []
      egress: []

Create **manifests/07-netpol-allow-web-to-api.yml**:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: allow-web-to-api
      namespace: net-demo
    spec:
      podSelector:
        matchLabels:
          app: net-demo
          component: api
      policyTypes: ["Ingress"]
      ingress:
      - from:
        - podSelector:
            matchLabels:
              app: net-demo
              component: web
        ports:
        - protocol: TCP
          port: 8080

Apply and test:

    kubectl apply -f manifests/06-netpol-default-deny.yml
    kubectl apply -f manifests/07-netpol-allow-web-to-api.yml

**Explanation**
- `default-deny-all` blocks all ingress/egress by default in the namespace.
- `allow-web-to-api` explicitly allows traffic from **web Pods** to **api Pods** on port 8080 only.
- You can add egress policies to restrict web’s outbound calls to specific CIDRs or Services.

---

## 6) (Optional) Service Types Demo

NodePort & LoadBalancer examples (do **not** apply both in prod for the same app):

- **NodePort** (lab/bare-metal quick access):

      kubectl -n net-demo patch svc/web -p '{"spec": {"type": "NodePort"}}'
      kubectl -n net-demo get svc web -o wide
      # access via http://<nodeIP>:<nodePort>

- **LoadBalancer** (cloud managed LB fronting web):

      kubectl -n net-demo patch svc/web -p '{"spec": {"type": "LoadBalancer"}}'
      kubectl -n net-demo get svc web -o wide
      # wait for EXTERNAL-IP; attach DNS/TLS at the LB or use Ingress for L7

---

## 7) Metrics Server — What to Install & Why

**What it provides**
- Real-time **resource usage** (CPU/mem) from each Node/Pod via Kubelet summary API.
- Enables:
  - `kubectl top nodes/pods` for quick visibility.
  - **HPA** based on CPU/Memory utilization (`autoscaling/v2`).
  - Dashboards (Kubernetes Dashboard “Metrics” tab).

**Install notes (not auto-installed on many clusters)**
- **EKS**: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` then patch flags for EKS if needed (e.g., `--kubelet-insecure-tls` in lab; in prod prefer proper TLS/CA).
- **GKE/AKS**: often preinstalled or available as an add-on.
- **Local (kind/minikube)**: enable via addons or apply upstream manifests.

**Validate**
- After install:
  
      kubectl top nodes
      kubectl top pods -A

**Use with HPA**
- Example HPA against our `api` (only if metrics-server is up):

      apiVersion: autoscaling/v2
      kind: HorizontalPodAutoscaler
      metadata:
        name: api-hpa
        namespace: net-demo
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: api
        minReplicas: 2
        maxReplicas: 6
        metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 60

  Apply it if desired:

      kubectl apply -f - <<'EOF'
      apiVersion: autoscaling/v2
      kind: HorizontalPodAutoscaler
      metadata:
        name: api-hpa
        namespace: net-demo
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: api
        minReplicas: 2
        maxReplicas: 6
        metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 60
      EOF

**tip**
- metrics-server is **not** a TSDB; for history, recording rules, and alerting use **Prometheus** (or cloud managed metrics). Pair HPA with **PodDisruptionBudgets** and **requests/limits** for stable autoscaling.

---

## ▶️ Run & Verify

    # Apply all manifests
    kubectl apply -f manifests/

    # Watch resources
    kubectl get deploy,svc,ing,netpol -n net-demo -o wide

    # Test in-cluster from a toolbox pod
    kubectl -n net-demo run toolbox --image=busybox:1.36 --restart=Never -it -- sh
    # inside:
    wget -qO- http://web.net-demo.svc.cluster.local
    wget -qO- http://api.net-demo.svc.cluster.local
    exit

    # If no ingress controller, port-forward web:
    kubectl -n net-demo port-forward svc/web 8080:80
    # open http://localhost:8080

    # If metrics-server installed:
    kubectl top nodes
    kubectl top pods -n net-demo

---

## 🧹 Cleanup

    kubectl delete -f manifests/07-netpol-allow-web-to-api.yml --ignore-not-found
    kubectl delete -f manifests/06-netpol-default-deny.yml --ignore-not-found
    kubectl delete -f manifests/05-ingress.yml --ignore-not-found
    kubectl delete -f manifests/04-svc-web.yml
    kubectl delete -f manifests/03-deploy-web.yml
    kubectl delete -f manifests/02-svc-api.yml
    kubectl delete -f manifests/01-deploy-api.yml
    kubectl delete -f manifests/00-namespace.yml

If you patched Service types (NodePort/LoadBalancer), revert to ClusterIP or delete the Service first.

---

## ✅ Summary

 Networking stack:
- **ClusterIP Services** for reliable east-west
- **Ingress** for HTTP/TLS north-south
- **NetworkPolicies** for default-deny + least privilege
- **Metrics Server** for live resource usage and **HPA** inputs

---
