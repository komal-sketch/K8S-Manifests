# Kubernetes CustomResourceDefinition (CRD) — Learning Guide

This README explains **CRDs** in a clear, production-aware way and provides a tiny, self-contained example you can apply to any cluster for learning.

---

## 🧠 What is a CRD?

A **CustomResourceDefinition (CRD)** extends the Kubernetes API with **new resource types** (Custom Resources, or CRs).  
After you install a CRD, Kubernetes will accept and store objects of your new type (e.g., `DevOpsLearning`) just like built-in resources (`Deployment`, `Service`, etc.).

- **CRD** → Defines the **schema** and API surface for your new kind (group/version/kind, names, scope).
- **CR (Custom Resource)** → An **instance** of that kind (e.g., `DevOpsLearning` named `devops-1`).
- (Optional) **Controller/Operator** → Code that **watches** these CRs and **reconciles** real-world state (create deployments, call APIs, rotate certs, etc.). You can use a CRD without a controller for declarative data storage, but most production value comes when you pair CRDs with a controller.

---

## 💼 Why CRDs Are Important in Production

- **Standardized platform APIs**: Expose “platform as a product” to app teams (e.g., `Database`, `KafkaTopic`, `Canary`).
- **Self-service & guardrails**: App teams declare intent; controllers enforce policy, quotas, and golden paths.
- **GitOps-friendly**: Everything is Kubernetes-native YAML (auditable, versioned, reviewed).
- **Separation of concerns**: Devs declare desired state; operators implement automation in controllers.

---

## ✅ Common Use Cases

- **Data services**: `PostgresCluster`, `Redis`, `KafkaTopic` with lifecycle automation.
- **Traffic mgmt & delivery**: `IngressRoute`, `HTTPRoute`, `Rollout` (canaries, blue/green).
- **Security & PKI**: `Certificate`, `Issuer` (cert-manager).
- **Platform config**: `AlertPolicy`, `Backup`, `CostBudget`, `NetworkPolicyTemplate`.
- **Batch/ML**: `SparkApplication`, `TrainingJob`.

---

## 🔧 How & When to Use CRDs

**Use a CRD when:**
- You need a **first-class API** to model domain concepts (not covered by built-ins).
- Teams will **create/update** these objects often, ideally via GitOps.
- You plan to **automate reconciliation** with a controller (operator pattern).

**Design tips (prod):**
- Choose a **stable API group** (e.g., `platform.example.com`) and version it (`v1`, `v1alpha1`).
- Provide a **structural schema** (`openAPIV3Schema`) with descriptions, types, defaults, enums, and **validation**.
- Support **conversion** and **webhooks** if you’ll evolve the API across versions.
- Ship **RBAC** for your CRDs (who can CRUD them) and for your controller.
- Include **status** fields for observability (`.status.conditions`, `.status.phase`).
- Treat CRDs like code: version, test, backward-compatible changes, and **document**.

---

## 🎯 Benefits Summary

- **API as a contract** between platform and apps.
- **Consistency**: common CRs across environments; one way to request services.
- **Safety & velocity**: guardrails in the controller; changes flow via PRs.
- **Observability**: `kubectl get/describe` works for your domain objects; can expose `.status`.

---

## 🧪 Hands-On: Minimal CRD + Custom Resource


### 1) `crd.yml` — CustomResourceDefinition

> Notes on important points:
> - `metadata.name` **must be** `<plural>.<group>`.
> - `spec.group` should be a DNS-style domain; we’ll use `groupname.com` to match your CR.
> - `names.kind` must be **CamelCase** (e.g., `DevOpsLearning`).
> - Structural schema indentation and keys corrected; `storage: true` is under the version.

    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: devopslearnings.groupname.com
    spec:
      group: groupname.com
      scope: Namespaced
      names:
        plural: devopslearnings
        singular: devopslearning
        kind: DevOpsLearning
        shortNames:
          - devops
          - batch
          - aws
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
                    name:
                      type: string
                      description: "This is the name of batch"
                    duration:
                      type: string
                      description: "This is the duration of batch"
                    mode:
                      type: string
                      description: "This is the mode of batch"
                    platform:
                      type: string
                      description: "This is the platform of batch"

### 2) Apply & Inspect the CRD

    kubectl apply -f crd.yml
    kubectl get crd
    kubectl describe crd devopslearnings.groupname.com

### 3) `devops-cr.yml` — Custom Resource (instance)

> Notes on important points:
> - `apiVersion` must match your CRD group/version.
> - `kind` must match `names.kind` in the CRD.
> - YAML scalars like “3 months” and names with capitals are quoted for safety.

    apiVersion: groupname.com/v1
    kind: DevOpsLearning
    metadata:
      name: devops-1
    spec:
      name: "Devops-Learning"
      duration: "3 months"
      platform: "website name"
      mode: "offline"

### 4) Apply & Query the Custom Resource

    kubectl apply -f devops-cr.yml

List using the **plural** name from the CRD (or a shortName):

    kubectl get devopslearnings
    # or with a short name:
    kubectl get devops

Describe it:

    kubectl describe devopslearning devops-1


---

## 🔍 What You’ll See (Without a Controller)

- The CR will be **stored** by the API server and visible via `kubectl`.
- Since we didn’t build a controller here, nothing else will happen—which is fine for **learning** the API surface.  
- In production, a controller would **watch** `DevOpsLearning` objects and **act** (e.g., spin up a training environment), then **update `.status`** back on the CR.

---

## 🛡️ Production Tips for CRDs

- **Versioning**: Start with `v1alpha1`; graduate to `v1` after field stability. Support conversion webhooks if you must change shapes.
- **Validation**: Use openAPI schema with `required`, `enum`, `pattern`, `min/maxLength`, and defaults (via webhook) to prevent bad configs.
- **RBAC**: Ship Roles/ClusterRoles so only intended users/controllers can CRUD your CRs.
- **Status & Conditions**: Provide clear feedback in `.status.conditions[*]` for UX and automation.
- **Docs & Examples**: Include `kubectl examples`, field docs, and diagrams. Treat CRDs like public APIs.

---

## 🧹 Cleanup

    kubectl delete -f devops-cr.yml
    kubectl delete -f crd.yml

---

## ✅ Summary

- **CRDs** let you add **new Kubernetes-native APIs** for your platform or domain.  
- They’re crucial in production to provide **self-service**, **guardrails**, and **automation** via controllers.  
- You created and queried a custom type (`DevOpsLearning`) to understand the lifecycle and naming rules (group, version, kind, names, scope, schema).
---
