# Kubernetes Mini Project: Workload Controllers (Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob) with Storage Notes

You’ll create namespaces, run `nginx` with different controllers, deploy a `mysql` StatefulSet with per-pod storage, and schedule short-lived/batch work.


---

## 🚀 Steps Overview
1. Create namespaces (`nginx`, `mysql`)  
2. Deploy `nginx` via **Deployment**  
3. Run `nginx` via **ReplicaSet** (for learning; Deployment already creates/manages RS)  
4. Deploy `mysql` via **StatefulSet** (with per-pod PVCs) + headless Service + config/secret  
5. Run `nginx` on every node via **DaemonSet**  
6. Run a one-off **Job**  
7. Schedule repeated work via **CronJob**  
8. Verify and explore behaviors

---

## 📁 Suggested Repo Layout

    manifests/
      00-namespace-nginx.yml
      00-namespace-mysql.yml
      01-deploy-nginx.yml
      02-replicaset-nginx.yml
      03-svc-mysql-headless.yml
      04-cm-mysql.yml
      05-secret-mysql.yml
      06-statefulset-mysql.yml
      07-daemonset-nginx.yml
      08-job-demo.yml
      09-cronjob-minute-backup.yml


---

## 1) Namespaces

Create **manifests/00-namespace-nginx.yml**:

    apiVersion: v1
    kind: Namespace
    metadata:
      name: nginx

Create **manifests/00-namespace-mysql.yml**:

    apiVersion: v1
    kind: Namespace
    metadata:
      name: mysql

Apply:

    kubectl apply -f manifests/00-namespace-nginx.yml
    kubectl apply -f manifests/00-namespace-mysql.yml
    kubectl get ns

---

## 2) Deployment: nginx

Create **manifests/01-deploy-nginx.yml**:

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

Apply and verify:

    kubectl apply -f manifests/01-deploy-nginx.yml
    kubectl rollout status deploy/nginx-deployment -n nginx
    kubectl get deploy,rs,pods -n nginx -o wide

---

## 3) ReplicaSet: nginx (for learning)

> A **Deployment** already creates and manages a **ReplicaSet**. This standalone ReplicaSet is included for educational purposes to see the differences.

Create **manifests/02-replicaset-nginx.yml**:

    kind: ReplicaSet
    apiVersion: apps/v1
    metadata:
      name: nginx-replicasets
      namespace: nginx
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          name: nginx-rep-pod
          labels:
            app: nginx
      # NOTE: template must match the selector and will be managed directly by this RS
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80

Apply and verify:

    kubectl apply -f manifests/02-replicaset-nginx.yml
    kubectl get rs,pods -n nginx -o wide

> ⚠️ Be careful not to have overlapping label selectors that collide with your Deployment’s selector, or two controllers might fight for the same Pods. For learning, you can keep the labels but avoid using both at once in production.

---

## 4) StatefulSet: mysql (with per-pod storage)

A StatefulSet needs:
- A **headless Service** (for stable DNS like `mysql-statefulset-0.mysql-service.mysql.svc.cluster.local`)
- Any required **ConfigMap/Secret** for environment variables
- The **StatefulSet** itself, with a `volumeClaimTemplates` for per-pod PVCs

### 4.1 Headless Service

Create **manifests/03-svc-mysql-headless.yml**:

    apiVersion: v1
    kind: Service
    metadata:
      name: mysql-service
      namespace: mysql
    spec:
      clusterIP: None
      selector:
        app: mysql
      ports:
        - name: mysql
          port: 3306
          targetPort: 3306

Apply:

    kubectl apply -f manifests/03-svc-mysql-headless.yml

### 4.2 ConfigMap and Secret

Create **manifests/04-cm-mysql.yml**:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: mysql-config-map
      namespace: mysql
    data:
      MYSQL_DATABASE: devops

Create **manifests/05-secret-mysql.yml**:

    apiVersion: v1
    kind: Secret
    metadata:
      name: mysql-secret
      namespace: mysql
    type: Opaque
    data:
      # base64("root") => cm9vdA==
      MYSQL_ROOT_PASSWORD: cm9vdA==

Apply:

    kubectl apply -f manifests/04-cm-mysql.yml
    kubectl apply -f manifests/05-secret-mysql.yml

### 4.3 StatefulSet

Create **manifests/06-statefulset-mysql.yml**:

    kind: StatefulSet
    apiVersion: apps/v1
    metadata:
      name: mysql-statefulset
      namespace: mysql
    spec:
      serviceName: mysql-service
      replicas: 3
      selector:
        matchLabels:
          app: mysql
      template:
        metadata:
          labels:
            app: mysql
        spec:
          containers:
          - name: mysql
            image: mysql:8.0
            ports:
            - containerPort: 3306
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config-map
                  key: MYSQL_DATABASE
            volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumeClaimTemplates:
      - metadata:
          name: mysql-data
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi
          # Optionally specify storageClassName for dynamic storage, e.g.:
          # storageClassName: gp2 / standard-rwo / rook-ceph-block / etc.

Apply and verify:

    kubectl apply -f manifests/06-statefulset-mysql.yml
    kubectl rollout status statefulset/mysql-statefulset -n mysql
    kubectl get sts,po,pvc -n mysql -o wide

Notes:
- Each replica (ordinal 0..N) gets its **own PVC** named like `mysql-data-mysql-statefulset-0`, etc.
- You need a default/dynamic **StorageClass** in your cluster for those PVCs to bind automatically, or specify a `storageClassName` explicitly. On local clusters, you may need to install a CSI driver (or use hostPath via a static PV).

---

## 5) DaemonSet: nginx (run on every node)

Create **manifests/07-daemonset-nginx.yml**:

    kind: DaemonSet
    apiVersion: apps/v1
    metadata:
      name: nginx-daemonsets
      namespace: nginx
    spec:
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          name: nginx-dmn-pod
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80

Apply and verify:

    kubectl apply -f manifests/07-daemonset-nginx.yml
    kubectl get ds,pods -n nginx -o wide
    kubectl describe ds nginx-daemonsets -n nginx

---

## 6) Job: one-off batch

Create **manifests/08-job-demo.yml**:

    kind: Job
    apiVersion: batch/v1
    metadata:
      name: demo-job
      namespace: nginx
    spec:
      completions: 1
      parallelism: 1
      template:
        metadata:
          name: demo-job-pod
          labels:
            app: batch-task
        spec:
          containers:
          - name: batch-container
            image: busybox:latest
            command: ["sh", "-c", "echo Hello K8S! && sleep 10"]
          restartPolicy: Never

Apply and check logs:

    kubectl apply -f manifests/08-job-demo.yml
    kubectl get jobs,pods -n nginx
    # find the pod name, then:
    kubectl logs -n nginx <job-pod-name>

---

## 7) CronJob: scheduled batch

Create **manifests/09-cronjob-minute-backup.yml**:

    apiVersion: batch/v1
    kind: CronJob
    metadata:
      name: minute-backup
      namespace: nginx
    spec:
      schedule: "* * * * *"
      jobTemplate:
        spec:
          template:
            metadata:
              labels:
                app: minute-backup
            spec:
              restartPolicy: OnFailure
              containers:
              - name: backup-container
                image: busybox
                command:
                - sh
                - -c
                - >
                  echo "Backup Started" ;
                  mkdir -p /backups &&
                  mkdir -p /demo-data &&
                  cp -r /demo-data /backups &&
                  echo "Backup Completed" ;
                volumeMounts:
                  - name: data-volume
                    mountPath: /demo-data
                  - name: backup-volume
                    mountPath: /backups
              volumes:
                - name: data-volume
                  hostPath:
                    path: /demo-data
                    type: DirectoryOrCreate
                - name: backup-volume
                  hostPath:
                    path: /backups
                    type: DirectoryOrCreate

Apply and watch:

    # ensure the directories exist on nodes (hostPath)
    sudo mkdir -p /demo-data /backups
    kubectl apply -f manifests/09-cronjob-minute-backup.yml
    kubectl get cronjob -n nginx
    # after a minute:
    kubectl get jobs -n nginx
    kubectl get pods -n nginx -l app=minute-backup
    # inspect logs of the latest pod to confirm:
    kubectl logs -n nginx <minute-backup-pod>

---

## ✅ Quick Verification Commands

    kubectl get all -n nginx
    kubectl get all -n mysql
    kubectl get pvc -n mysql
    kubectl describe sts mysql-statefulset -n mysql
    kubectl describe ds nginx-daemonsets -n nginx
    kubectl get jobs,cronjobs -n nginx

---

## 🧠 Concepts & Explanations

### Deployment
- **What**: Declarative manager for stateless Pods with rolling updates and rollbacks.
- **How**: You specify `spec.replicas` and a Pod template; Deployment creates a **ReplicaSet** to satisfy the desired count and manages updates by creating new RS versions.

### ReplicaSet (RS)
- **What**: Ensures a specified number of Pods with matching labels are running.
- **When to use**: Rarely used directly in modern workflows—**Deployment** manages ReplicaSets for you. Useful for educational purposes and special cases requiring direct control.

### StatefulSet
- **What**: Manages **stateful** applications; provides **stable network IDs** and **stable storage** per replica.
- **Key features**:
  - Predictable pod names: `<name>-0`, `<name>-1`, …
  - **Headless Service** for stable DNS records.
  - `volumeClaimTemplates` create a **PVC per Pod** (e.g., `mysql-data-mysql-statefulset-0`).
- **Storage**:
  - Typically uses a **StorageClass** with dynamic provisioning so PVCs bind automatically.
  - Access mode is often **RWO** (one node at a time) for block storage.

### DaemonSet
- **What**: Runs one Pod **per node** (or per selected nodes).
- **Use cases**: Node-level agents (logging, monitoring, CNI, storage daemons, etc.).

### Job
- **What**: Runs Pods to **completion** (succeeds once desired completions reach).
- **Use cases**: Migrations, data processing, batch tasks.

### CronJob
- **What**: Schedules **Jobs** using crontab syntax.
- **Use cases**: Backups, reports, periodic cleanups.
- **Note on `hostPath`**: Ties Pod storage to the node filesystem—OK for demos; prefer CSI/NFS for portable, multi-node persistence.

### Storage notes for StatefulSet
- **volumeClaimTemplates**: Template for creating **one PVC per replica**. Each pod retains its PVC across restarts and rescheduling.
- **StorageClass** (not shown explicitly here): Defines how storage is provisioned (dynamic/CSI vs static). If your cluster has a default StorageClass, the PVCs will bind automatically; otherwise, specify `storageClassName`.

---

## 🧹 Cleanup

    kubectl delete -f manifests/09-cronjob-minute-backup.yml
    kubectl delete -f manifests/08-job-demo.yml
    kubectl delete -f manifests/07-daemonset-nginx.yml
    kubectl delete -f manifests/06-statefulset-mysql.yml
    kubectl delete -f manifests/05-secret-mysql.yml
    kubectl delete -f manifests/04-cm-mysql.yml
    kubectl delete -f manifests/03-svc-mysql-headless.yml
    kubectl delete -f manifests/02-replicaset-nginx.yml
    kubectl delete -f manifests/01-deploy-nginx.yml
    kubectl delete -f manifests/00-namespace-nginx.yml
    kubectl delete -f manifests/00-namespace-mysql.yml

If you created hostPath directories for the CronJob demo:

    sudo rm -rf /demo-data /backups

---

## ✅ Summary

You now have a compact project showing how to:
- Run stateless apps via **Deployment/ReplicaSet**
- Manage stateful storage and stable identities via **StatefulSet**
- Place a pod on **every node** with **DaemonSet**
- Perform **batch** and **scheduled** work with **Job** and **CronJob**

This set is perfect for learning how Kubernetes controllers differ and when to use each.
---
