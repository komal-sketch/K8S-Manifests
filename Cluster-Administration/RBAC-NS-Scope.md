# Kubernetes RBAC — NameSpace level


---

## 🧠 RBAC: What, Why, Use Cases

**What is RBAC?**  
RBAC lets you define **who** (users, groups, service accounts) can **do what** (verbs like get, list, create, update, delete) on **which resources** (Pods, Deployments, Secrets, etc.) in **which scope** (namespace or cluster).

**Why it matters in prod**  
- **Least privilege**: minimize blast radius if credentials are compromised.  
- **Segregation of duties**: devs vs. ops vs. CI/CD robots.  
- **Auditability**: clear, reviewable access policy in Git.

**Common production use cases**  
- **App service accounts** can only read/write their own ConfigMaps/Secrets.  
- **CI/CD** service account with create/patch on Deployments in a single namespace.  
- **Read-only support** role with list/get/watch across namespaces.  
- **Cluster-operators** with cluster-wide rights using ClusterRoles/ClusterRoleBindings.

---

## 🚀 What we Build

A small project that:

1) Creates an **`apache`** namespace and deploys a simple Apache app.  
2) Demonstrates `kubectl auth` checks.  
3) Creates a **Role** (namespace-scoped) that grants limited rights.  
4) Creates a **ServiceAccount** and **RoleBinding** to grant those rights to only that SA.  
5) Verifies access with `--as=system:serviceaccount:apache:apache-user`.


---

## 📁 Repo Layout

    manifests/
      00-namespace.yml
      01-apache-deployment.yml
      02-role.yml
      03-serviceaccount.yml
      04-rolebinding.yml

---

## 1) Quick Cluster Introspection (safe to run as current user)

    kubectl get nodes
    kubectl get ns
    kubectl auth whoami
    kubectl auth can-i get pods

These show current identity and a sample “can I?” query cluster-wide (no namespace specified).

---

## 2) Namespace

Create **manifests/00-namespace.yml**:

    apiVersion: v1
    kind: Namespace
    metadata:
      name: apache
      labels:
        app: apache

Apply & verify:

    kubectl apply -f manifests/00-namespace.yml
    kubectl get ns apache

Re-run an auth check but scoped:

    kubectl auth can-i get pods -n apache

*(This checks your **current** identity. We’ll create a restricted identity next.)*

---

## 3) App: Apache Deployment + Service (in `apache`)

Create **manifests/01-apache-deployment.yml**:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: apache-deployment
      namespace: apache
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: apache
      template:
        metadata:
          labels:
            app: apache
        spec:
          containers:
          - name: apache
            image: httpd:2.4
            ports:
            - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: apache-service
      namespace: apache
    spec:
      selector:
        app: apache
      ports:
      - protocol: TCP
        port: 80
        targetPort: 80
      type: ClusterIP

Apply & verify:

    kubectl apply -f manifests/01-apache-deployment.yml
    kubectl get deploy,svc,pods -n apache

---

## 4) RBAC — Role (namespace-scoped)

> **Important corrections vs. the draft you shared**
> - Kind is **`Role`** (capital R), not `role`.  
> - API group is **`rbac.authorization.k8s.io/v1`**, not `rback...`.  
> - `resources` must be **plural** (e.g., `deployments`, `pods`, `services`).  
> - Valid verbs are: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`. There is **no** `apply` verb.

Create **manifests/02-role.yml**:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: apache-manager
      namespace: apache
    rules:
    - apiGroups: ["apps"]          # for deployments/replicasets
      resources: ["deployments"]
      verbs: ["get","list","watch","create","update","patch","delete"]
    - apiGroups: [""]              # core API group (""), for pods/services
      resources: ["pods","services"]
      verbs: ["get","list","watch"]

**Explanation: `apiGroups`**
- `""` (empty string) is the **core** group (Pods, Services, ConfigMaps, Secrets, etc.).  
- `"apps"` covers Deployments, DaemonSets, StatefulSets, ReplicaSets.  
- Other common groups: `"batch"` (Jobs, CronJobs), `"networking.k8s.io"` (Ingress, NetworkPolicies).

Apply & verify:

    kubectl apply -f manifests/02-role.yml
    kubectl get role -n apache
    kubectl describe role apache-manager -n apache

---

## 5) ServiceAccount (the identity we’ll bind)

> Kind is **`ServiceAccount`** and it lives in a namespace. We’ll grant the Role to this SA.

Create **manifests/03-serviceaccount.yml**:

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: apache-user
      namespace: apache

Apply & verify:

    kubectl apply -f manifests/03-serviceaccount.yml
    kubectl get serviceaccount -n apache

Auth checks:

    # You (current identity), likely yes depending on your cluster role:
    kubectl auth can-i get pods -n apache

    # Impersonate the service account without any binding yet: should be "no"
    kubectl auth can-i get pods -n apache --as=system:serviceaccount:apache:apache-user

---

## 6) RoleBinding (grant the Role to the ServiceAccount)

> **Subjects**: use `kind: ServiceAccount` for SAs.  
> **roleRef**: points to the Role in the same namespace.

Create **manifests/04-rolebinding.yml**:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: apache-manager-rolebinding
      namespace: apache
    subjects:
    - kind: ServiceAccount
      name: apache-user
      namespace: apache
    roleRef:
      kind: Role
      name: apache-manager
      apiGroup: rbac.authorization.k8s.io

Apply & verify:

    kubectl apply -f manifests/04-rolebinding.yml
    kubectl get rolebinding -n apache
    kubectl describe rolebinding apache-manager-rolebinding -n apache

Re-run the impersonated auth checks (now should be **yes** for the verbs allowed by the Role):

    kubectl auth can-i get pods -n apache --as=system:serviceaccount:apache:apache-user
    kubectl auth can-i list deployments -n apache --as=system:serviceaccount:apache:apache-user
    kubectl auth can-i delete services -n apache --as=system:serviceaccount:apache:apache-user   # should be "no" (we granted only get/list/watch on services)



---

## 🧪 Useful Auth Commands

    kubectl auth whoami
    kubectl auth can-i --list -n apache
    kubectl auth can-i create deploy -n apache --as=system:serviceaccount:apache:apache-user
    kubectl get deploy -n apache --as=system:serviceaccount:apache:apache-user
    kubectl describe deploy apache-deployment -n apache --as=system:serviceaccount:apache:apache-user

---

## 🏭 Tips

- **Principle of least privilege**: grant only verbs/resources needed.  
- **Separate SAs per app/namespace**; avoid reusing the default SA.  
- **GitOps** your RBAC—review PRs for rights changes.  
- Pair with **NetworkPolicies**, **PodSecurity**, and **image policies**.  
- Rotate tokens/identities; use **OIDC** with groups for human users.  
- Audit with **`kubectl auth can-i --list`** and cloud audit logs.

---

## 🧹 Cleanup

    kubectl delete -f manifests/04-rolebinding.yml
    kubectl delete -f manifests/03-serviceaccount.yml
    kubectl delete -f manifests/02-role.yml
    kubectl delete -f manifests/01-apache-deployment.yml
    kubectl delete -f manifests/00-namespace.yml

---

## ✅ Summary

Created an isolated namespace, deployed an application, then used **RBAC** to grant a **ServiceAccount** exactly the rights it needs—no more, no less.
Learned how to check access with `kubectl auth`, how `apiGroups` map to resources, and how to bind permissions correctly using **RoleBinding**.

---
