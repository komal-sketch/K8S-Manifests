# Kubernetes Mini Project: Persistent Volumes, PVCs, StorageClass, and Mounted Deployments (nginx) + ConfigMap/Secret Examples

This small project demonstrates **Kubernetes storage fundamentals**:
- Create a **namespace** (`nginx`)
- Define a **StorageClass** for local/static storage
- Create a **PersistentVolume (PV)** and **PersistentVolumeClaim (PVC)**
- Mount the PVC into an **nginx Deployment**
- Expose nginx with a **Service**
- Bonus: **ConfigMap** and **Secret** manifests (in a separate `mysql` namespace) with explanations


---

## 🚀 Steps Overview
1. Create the `nginx` namespace  
2. Create a `StorageClass` (`local-storage`)  
3. Create a `PersistentVolume` (`local-pv`)  
4. Create a `PersistentVolumeClaim` (`local-pvc`)  
5. Deploy `nginx` mounting the PVC  
6. Expose with a `ClusterIP` Service  
7. (Bonus) Apply `mysql` ConfigMap & Secret examples  
8. Verify persistence and behavior

---

## 📁 Suggested Repo Layout

    manifests/
      01-namespace-nginx.yml
      02-storageclass-local.yml
      03-pv-local.yml
      04-pvc-local.yml
      05-deploy-nginx-pvc.yml
      06-svc-nginx.yml
      bonus-mysql/
        01-namespace-mysql.yml
        02-configmap.yml
        03-secret.yml

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

## 2) StorageClass (local static provisioning)

> The PVC in this project references `storageClassName: local-storage`.  
> Define a **StorageClass** to match that name for **static provisioning** on a single node.

Create **manifests/02-storageclass-local.yml**:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-storage
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    reclaimPolicy: Retain

Apply:

    kubectl apply -f manifests/02-storageclass-local.yml

---

## 3) PersistentVolume (PV)

> ⚠️ **PV is cluster-scoped (no namespace field).**  

Create **manifests/03-pv-local.yml**:

    kind: PersistentVolume
    apiVersion: v1
    metadata:
      name: local-pv
      labels:
        app: local
    spec:
      capacity:
        storage: 1Gi
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-storage
      hostPath:
        path: /mnt/data

Apply:

    sudo mkdir -p /mnt/data     # ensure the hostPath exists on the node you're scheduling to
    kubectl apply -f manifests/03-pv-local.yml
    kubectl get pv

> 💡 **hostPath** is best for local dev clusters (minikube/kind/microk8s, single-node).  
> In production, use a cloud/dynamic provisioner (EBS, PD, Ceph/Rook, NFS, etc.).

---

## 4) PersistentVolumeClaim (PVC)

Create **manifests/04-pvc-local.yml**:

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: local-pvc
      namespace: nginx
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: local-storage

Apply:

    kubectl apply -f manifests/04-pvc-local.yml
    kubectl get pvc -n nginx
    kubectl describe pvc local-pvc -n nginx

When bound, `STATUS` becomes `Bound` and it will reference `local-pv`.

---

## 5) Deployment: nginx (mount PVC)

> The original mount used `/var/www/html`. The official `nginx:latest` serves from **/usr/share/nginx/html**.  
> To see content in the default welcome page path, we’ll mount `/usr/share/nginx/html`.

Create **manifests/05-deploy-nginx-pvc.yml**:

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
            volumeMounts:
            - mountPath: /usr/share/nginx/html
              name: my-volume
          volumes:
          - name: my-volume
            persistentVolumeClaim:
              claimName: local-pvc

Apply:

    kubectl apply -f manifests/05-deploy-nginx-pvc.yml

Check rollout and pods:

    kubectl rollout status deployment/nginx-deployment -n nginx
    kubectl get pods -n nginx -o wide

---

## 6) Service: ClusterIP

Create **manifests/06-svc-nginx.yml**:

    kind: Service
    apiVersion: v1
    metadata:
      name: nginx-service
      namespace: nginx
    spec:
      selector:
        app: nginx
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: ClusterIP

Apply:

    kubectl apply -f manifests/06-svc-nginx.yml
    kubectl get svc -n nginx

Test inside cluster:

    kubectl run -i --tty tester -n nginx --image=busybox --restart=Never -- /bin/sh
    # inside tester:
    wget -qO- http://nginx-service.nginx.svc.cluster.local

---

## 7) (Bonus) ConfigMap & Secret for MySQL

Create a separate `mysql` namespace:

Create **bonus-mysql/01-namespace-mysql.yml**:

    kind: Namespace
    apiVersion: v1
    metadata:
      name: mysql

Apply:

    kubectl apply -f bonus-mysql/01-namespace-mysql.yml

ConfigMap:

Create **bonus-mysql/02-configmap.yml**:

    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: mysql-config-map
      namespace: mysql
    data:
      MYSQL_DATABASE: devops

Secret (base64 value is `root`):

Create **bonus-mysql/03-secret.yml**:

    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-secret
      namespace: mysql
    type: Opaque
    data:
      MYSQL_ROOT_PASSWORD: cm9vdAo=

Apply both:

    kubectl apply -f bonus-mysql/02-configmap.yml
    kubectl apply -f bonus-mysql/03-secret.yml

> 💡 To use these in a Deployment (example), you’d reference:
> - `envFrom.configMapRef.name: mysql-config-map`
> - `envFrom.secretRef.name: mysql-secret`
> or individual `env` entries with `valueFrom.configMapKeyRef` / `secretKeyRef`.

---

## ✅ Verify Persistence

1) Write a file via one nginx pod and confirm it appears via the Service:

    # pick one nginx pod:
    POD=$(kubectl get pod -n nginx -l app=nginx -o jsonpath='{.items[0].metadata.name}')
    kubectl exec -n nginx -it "$POD" -- /bin/sh -c 'echo "<h1>Hello from PVC!</h1>" > /usr/share/nginx/html/index.html'

2) Curl from a test pod (or from another shell inside the cluster):

    kubectl run -i --tty tester -n nginx --image=busybox --restart=Never -- /bin/sh
    # inside tester:
    wget -qO- http://nginx-service.nginx.svc.cluster.local

You should see: **Hello from PVC!**  
Because both replicas mount the same PVC (RWO, one node), the content is persisted on the node path `/mnt/data`.

---

## 🧠 Concepts & Explanations

### StorageClass
- **What it is:** A template describing a class of storage (provisioner, parameters, reclaim policy, binding mode).
- **Static vs Dynamic:**
  - **Static provisioning:** You create PVs up-front (as in this project). PVCs bind to matching PVs via `storageClassName`, `capacity`, `accessModes`, and selectors.
  - **Dynamic provisioning:** A controller (provisioner) creates PVs on-demand when a PVC is created (e.g., AWS EBS, GCE PD, CSI drivers).
- **Key fields:**
  - `provisioner`: For static local storage, use `kubernetes.io/no-provisioner`. For cloud/dynamic, use the CSI or cloud provisioner name.
  - `volumeBindingMode`:
    - `Immediate`: Binds PVC to a PV as soon as possible.
    - `WaitForFirstConsumer`: Defers binding until a Pod using the PVC is scheduled, which improves local/zone-aware placement.
  - `reclaimPolicy`: What happens when the claim is released:
    - `Retain`: Keep underlying data (manual cleanup).
    - `Delete`: Delete volume in backend.
    - `Recycle` (deprecated): Basic scrub, not generally used now.

### PersistentVolume (PV)
- **What it is:** Cluster-scoped storage resource representing an actual piece of storage (disk, NFS path, hostPath) available to the cluster.
- **Cluster-scoped:** Do **not** put `namespace` on PVs.
- **Important fields:** `capacity.storage`, `accessModes`, `persistentVolumeReclaimPolicy`, `storageClassName`, and the backing storage (e.g., `hostPath`, `nfs`, `csi`).

### PersistentVolumeClaim (PVC)
- **What it is:** A request for storage by a user/workload.
- **Binding:** The control plane binds a PVC to a compatible PV (matching size, access modes, class).
- **Usage:** Pods reference PVCs via `volumes[].persistentVolumeClaim.claimName`.

### Access Modes
- `ReadWriteOnce (RWO)`: Mounted read-write by a **single node** at a time (most common).
- `ReadOnlyMany (ROX)`: Multiple nodes can read (rare with local storage).
- `ReadWriteMany (RWX)`: Multiple nodes can read-write (requires shared storage like NFS/CSI that supports it).

### hostPath (Dev Only)
- Binds a directory from the **node’s filesystem** into the pod.
- Great for **local testing**; not recommended for **multi-node production** due to tight coupling to a specific node.

### ConfigMap & Secret
- **ConfigMap:** Plain-text configuration (non-sensitive), often mounted as files or injected as env vars.
- **Secret:** Base64-encoded sensitive data; mounted as files or env vars. Prefer Secrets for credentials (and enable encryption-at-rest + RBAC).

---

## 🧪 Useful Commands

    # Namespaced views
    kubectl get all -n nginx
    kubectl describe deploy nginx-deployment -n nginx
    kubectl describe pvc local-pvc -n nginx

    # Cluster-scoped PV/SC
    kubectl get sc
    kubectl describe sc local-storage
    kubectl get pv
    kubectl describe pv local-pv

    # Logs & debugging
    kubectl logs -f -n nginx -l app=nginx
    kubectl exec -n nginx -it deploy/nginx-deployment -- /bin/sh

---

## 🧹 Cleanup

    kubectl delete -f manifests/06-svc-nginx.yml
    kubectl delete -f manifests/05-deploy-nginx-pvc.yml
    kubectl delete -f manifests/04-pvc-local.yml
    kubectl delete -f manifests/03-pv-local.yml
    kubectl delete -f manifests/02-storageclass-local.yml
    kubectl delete -f manifests/01-namespace-nginx.yml

    kubectl delete -f bonus-mysql/03-secret.yml
    kubectl delete -f bonus-mysql/02-configmap.yml
    kubectl delete -f bonus-mysql/01-namespace-mysql.yml

If your StorageClass/PV used `Retain`, manually remove data on the node (e.g., `/mnt/data`) if you want a clean slate.

---

## ✅ Summary

You now have a working example that:
- Defines a **StorageClass**, **PV**, and **PVC**
- Mounts the PVC into an **nginx** Deployment
- Exposes it with a **Service**
- Demonstrates **ConfigMap** and **Secret** usage patterns (bonus)

This pattern is the backbone of persisting data for stateful apps in Kubernetes. For production, switch to a CSI-backed dynamic StorageClass and appropriate access modes.
---
