# Kubernetes Mini Project: Namespace, Deployments (Resource Requests/Limits), Taints & Tolerations, and Probes

This small project packages four foundational Kubernetes features into a copy-paste friendly guide:

- Create a **Namespace**
- Deploy **nginx** with **CPU/Memory requests & limits**
- Run a **Pod with a toleration** (for tainted nodes)
- Deploy a sample **notes-app** with **liveness & readiness probes**



---

## 🚀 Steps Overview
1. Apply the `nginx` namespace  
2. Deploy `nginx` (with resource requests/limits)  
3. (Optional) Run a Pod with tolerations (to land on tainted nodes)  
4. Deploy `notes-app` with liveness/readiness probes  
5. Verify resources and behavior

---

## 📁 Suggested Repo Layout

    manifests/
      01-namespace-nginx.yml
      02-deploy-nginx.yml
      03-pod-nginx-toleration.yml
      04-deploy-notes-app-with-probes.yml

You can keep the same filenames or choose your own; commands below assume this layout.

---

## 1) Namespace (nginx)

Create **manifests/01-namespace-nginx.yml**:

    kind: Namespace
    apiVersion: v1
    metadata:
      name: nginx

Apply:

    kubectl apply -f manifests/01-namespace-nginx.yml

Verify:

    kubectl get ns
    kubectl get all -n nginx

---

## 2) Deployment: nginx (with CPU/Memory Requests & Limits)

Create **manifests/02-deploy-nginx.yml**:

    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: nginx-deployment
      namespace: nginx
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          name: nginx-dep-pod
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
            resources:
              requests:
                cpu: 100m
                memory: 128Mi
              limits:
                cpu: 200m
                memory: 256Mi

Apply:

    kubectl apply -f manifests/02-deploy-nginx.yml

Check rollout and pods:

    kubectl rollout status deployment/nginx-deployment -n nginx
    kubectl get pods -n nginx -o wide
    kubectl describe deploy nginx-deployment -n nginx

> 💡 **Why requests/limits?**  
> - **Requests**: scheduler guarantee (minimum resources the Pod needs).  
> - **Limits**: hard cap—container can’t exceed these.  
> Setting these enables effective scheduling and is required for autoscalers like HPA/VPA to make informed decisions.

---

## 3) Taints & Tolerations (Optional)

Create **manifests/03-pod-nginx-toleration.yml**:

    kind: Pod
    apiVersion: v1
    metadata:
      name: nginx
      namespace: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      tolerations:
        - key: "prod"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"

Apply:

    kubectl apply -f manifests/03-pod-nginx-toleration.yml

Verify:

    kubectl get pod nginx -n nginx -o wide
    kubectl describe pod nginx -n nginx

> 💡 **How taints & tolerations work**  
> - **Taints** are set **on nodes** to repel certain pods:  
>     `kubectl taint nodes <node-name> prod=true:NoSchedule`  
> - **Tolerations** on a **pod** allow it to be scheduled on nodes that carry matching taints.  
> - A **toleration alone does not force** a pod onto a tainted node; it only **permits** scheduling there if other constraints match.

If you want this pod to land on the tainted node, ensure:
- The node is actually tainted with `prod=true:NoSchedule`.
- Other scheduling constraints (resources, selectors, topology, etc.) permit placement.

---

## 4) Deployment with Probes: notes-app

Create **manifests/04-deploy-notes-app-with-probes.yml**:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: notes-app-deployment
      labels:
        app: notes-app
      namespace: notes-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: notes-app
      template:
        metadata:
          labels:
            app: notes-app
        spec:
          containers:
          - name: notes-app
            image: notes-app-k8s:latest
            ports:
            - containerPort: 8000
            livenessProbe:
              httpGet:
                path: /
                port: 8000
            readinessProbe:
              httpGet:
                path: /
                port: 8000

> This manifest uses a different namespace (`notes-app`). Create it first:

    kubectl create namespace notes-app

Apply the deployment:

    kubectl apply -f manifests/04-deploy-notes-app-with-probes.yml

Check:

    kubectl get all -n notes-app
    kubectl describe deploy notes-app-deployment -n notes-app
    kubectl describe pod -l app=notes-app -n notes-app

> 💡 **Probes explained**  
> - **Liveness Probe**: signals if the container is still alive. If it fails repeatedly, Kubernetes restarts the container.  
> - **Readiness Probe**: signals if the app is ready to serve traffic. If it fails, the pod is removed from Service endpoints (no traffic routed) until it passes.  
> - Common probe types: `httpGet`, `tcpSocket`, `exec`. You can also tune `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `failureThreshold`, etc.

---

## ✅ Verification & Useful Commands

List everything in the `nginx` namespace:

    kubectl get all -n nginx

Describe a specific pod:

    kubectl describe pod <pod-name> -n nginx

Tail logs:

    kubectl logs -f <pod-name> -n nginx

If you later add a Service for nginx (ClusterIP example):

    kubectl expose deployment/nginx-deployment -n nginx --port=80 --target-port=80 --name=nginx-svc --type=ClusterIP
    kubectl get svc -n nginx
    kubectl run -i --tty tester -n nginx --image=busybox --restart=Never -- /bin/sh
    # inside tester:
    wget -qO- http://nginx-svc.nginx.svc.cluster.local

---

## 🧠 Concepts Recap

- **Namespace**: Logical partitioning; helps isolate resources and apply policies per environment/team.  
- **Deployment**: Manages replica sets and provides rolling updates for stateless workloads.  
- **Resources (requests/limits)**:  
  - `requests` guide the **scheduler** (min CPU/memory the pod needs).  
  - `limits` cap container usage; exceeding CPU limit throttles; exceeding memory limit may OOM-kill.  
- **Taints & Tolerations**:  
  - Taint **nodes** to repel pods by default.  
  - Add **tolerations** on pods to allow scheduling on tainted nodes.  
- **Probes**:  
  - **Liveness** restarts unhealthy containers.  
  - **Readiness** gates traffic until the pod is actually ready.

---

## 📌 Tips & Next Steps

- Add a **Service** and **Ingress** (or port-forward) to reach apps externally.  
- Parameterize images/replicas with **Kustomize** or **Helm** for multiple environments.  
- Consider adding **HPA/VPA** later to scale based on utilization or recommendations.

---
