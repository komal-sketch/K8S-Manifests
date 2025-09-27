# Chat Application on Kubernetes – 3‑Tier Architecture

![alt text](image.png)

This repository documents how to deploy a simple full‑stack chat application on Kubernetes using a **three‑tier architecture**.  The tiers are:

1. **Database** – MongoDB stores chat messages and user data.
2. **Backend** – a Node.js/Express API that connects to MongoDB and exposes REST endpoints used by the frontend.
3. **Frontend** – a static site (built with React) served by NGINX that consumes the backend API.

The project walks through cloning the source code, building and pushing Docker images, and then deploying the application to a Kubernetes cluster (e.g. a local kind cluster).  All manifests use namespace‑scoped resources and a PersistentVolume/PersistentVolumeClaim to persist MongoDB data.  An ingress exposes the application via HTTP.

---

## Prerequisites

* A running Kubernetes cluster (kind, minikube, or any managed cluster).
* kubectl installed and configured to talk to your cluster.
* Docker installed and authenticated to Docker Hub (for pushing images).
* A TLS termination ingress controller (e.g. NGINX Ingress) running in the cluster.

---

## Clone the Repository and Build Docker Images

Clone the source code for the chat application and build/push Docker images for the backend and frontend.  Replace `<docker‑username>` with your Docker Hub username.

    # Clone the repository
    git clone https://github.com/komal-sketch/full-stack_chatApp.git
    cd full-stack_chatApp

    # Log in to Docker Hub
    docker login -u <docker‑username>

    # Build & push backend image
    cd backend
    docker build -t prajjwal18/chatapp-backend:latest .
    docker push prajjwal18/chatapp-backend:latest

    # Build & push frontend image
    cd ../frontend
    docker build -t prajjwal18/chatapp-frontend:latest .
    docker push prajjwal18/chatapp-frontend:latest

---

## Kubernetes Manifests

All Kubernetes resources live in a dedicated namespace `chat-app`.  The manifests below can be stored in individual YAML files (as named) and applied with `kubectl apply -f <file>`.  

### 1. Namespace

Create a namespace for isolating the chat application:

    # namespace.yml
    kind: Namespace
    apiVersion: v1
    metadata:
      name: chat-app

Apply the namespace:

    kubectl create -f namespace.yml

Verify the namespace:

    kubectl get ns

---

### 2. PersistentVolume and PersistentVolumeClaim

MongoDB requires persistent storage.  In a local cluster (kind/minikube) the `local-path` provisioner dynamically creates volumes.  Alternatively, you can statically provision a PersistentVolume and a matching PVC.  This example uses a static hostPath volume called `mongodb-pv` with a 5 GiB capacity.  **Ensure the directory exists on the node.**

    # mongodb-pv.yml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: mongodb-pv
      namespace: chat-app
    spec:
      capacity:
        storage: 5Gi
      accessModes:
        - ReadWriteOnce
      storageClassName: standard
      persistentVolumeReclaimPolicy: Retain
      hostPath:
        path: /mnt/data/mongodb  # directory on the node/kind container
        type: DirectoryOrCreate

Create a PersistentVolumeClaim that binds to the above PV:

    # mongodb-pvc.yml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mongodb-pvc
      namespace: chat-app
    spec:
      storageClassName: standard
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi

---

### 3. MongoDB Deployment

Deploy MongoDB as a single replica.  The deployment mounts the PVC at `/data/db` and sets the root credentials via environment variables.  A service named `mongodb` exposes MongoDB on port 27017 within the `chat-app` namespace.

    # mongodb-deployment.yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mongodb-deployment
      namespace: chat-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mongodb
      template:
        metadata:
          name: mongodb-pod
          namespace: chat-app
          labels:
            app: mongodb
        spec:
          containers:
            - name: chatapp-mongodb
              image: mongo:latest
              ports:
                - containerPort: 27017
              env:
                - name: MONGO_INITDB_ROOT_USERNAME
                  value: mongoadmin
                - name: MONGO_INITDB_ROOT_PASSWORD
                  value: secret
              volumeMounts:
                - name: mongodb-storage
                  mountPath: /data/db
          volumes:
            - name: mongodb-storage
              persistentVolumeClaim:
                claimName: mongodb-pvc

    # mongodb-service.yml
    apiVersion: v1
    kind: Service
    metadata:
      name: mongodb
      namespace: chat-app
    spec:
      selector:
        app: mongodb
      ports:
        - protocol: TCP
          port: 27017
          targetPort: 27017

---

### 4. Backend Deployment and Service

The backend is a Node.js API.  It connects to MongoDB using the `MONGODB_URI` environment variable, listens on port 5001, and reads a JWT secret from a Kubernetes secret named `chatapp-secrets`.  A service named `backend` exposes the API to other services within the namespace.

    # backend-deployment.yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend-deployment
      namespace: chat-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: backend
      template:
        metadata:
          name: backend-pod
          namespace: chat-app
          labels:
            app: backend
        spec:
          containers:
            - name: chatapp-backend
              image: prajjwal18/chatapp-backend:latest
              ports:
                - containerPort: 5001
              env:
                - name: MONGODB_URI
                  value: "mongodb://mongoadmin:secret@mongodb:27017/dbname?authSource=admin"
                - name: PORT
                  value: "5001"
                - name: NODE_ENV
                  value: "production"
                - name: JWT_SECRET
                  valueFrom:
                    secretKeyRef:
                      name: chatapp-secrets
                      key: jwt

    # backend-service.yml
    apiVersion: v1
    kind: Service
    metadata:
      name: backend
      namespace: chat-app
    spec:
      selector:
        app: backend
      ports:
        - protocol: TCP
          port: 5001
          targetPort: 5001

---

### 5. Frontend Deployment and Service

The frontend is a React application served by NGINX.  It listens on port 80 and is exposed via a `ClusterIP` service named `frontend`.

    # frontend-deployment.yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: frontend-deployment
      namespace: chat-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: frontend
      template:
        metadata:
          name: frontend-pod
          namespace: chat-app
          labels:
            app: frontend
        spec:
          containers:
            - name: chatapp-frontend
              image: prajjwal18/chatapp-frontend:latest
              ports:
                - containerPort: 80
              env:
                - name: NODE_ENV
                  value: production

    # frontend-service.yml
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend
      namespace: chat-app
    spec:
      selector:
        app: frontend
      ports:
        - port: 80
          targetPort: 80

---

### 6. Kubernetes Secret

Store the JWT secret as a base64‑encoded Kubernetes Secret.  The key `jwt` contains a base64 representation of your JWT secret string.  To generate a base64 string, run `echo -n <secret> | base64`.

    # secrets.yml
    apiVersion: v1
    kind: Secret
    metadata:
      name: chatapp-secrets
      namespace: chat-app
    type: Opaque
    data:
      jwt: Put your base64 encoded key here

Apply the secret before deploying the backend:

    kubectl apply -f secrets.yml

---

### 7. Ingress

If your cluster has an ingress controller, you can expose both the frontend and backend through a single ingress resource.  The example below uses a simple path prefix rule for `/` to route to both services.  For production deployments, refine these rules or use different subpaths/subdomains.

    # ingress.yml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: chatapp-ingress
      namespace: chat-app
      labels:
        name: chatapp-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: "/"
        nginx.ingress.kubernetes.io/ssl-redirect: "false"
    spec:
      rules:
        - host: localhost
          http:
            paths:
              - pathType: Prefix
                path: "/"
                backend:
                  service:
                    name: frontend
                    port:
                      number: 80
              - pathType: Prefix
                path: "/"
                backend:
                  service:
                    name: backend
                    port:
                      number: 5001

Apply the ingress after deploying all services:

    kubectl apply -f ingress.yml

---

## Deployment Order

When deploying resources, create them in the following order to avoid dependency issues:

1. **Namespace:** `kubectl apply -f namespace.yml`
2. **MongoDB Service:** `kubectl apply -f mongodb-service.yml`
3. **Persistent Volume / Claim:** `kubectl apply -f mongodb-pv.yml` then `kubectl apply -f mongodb-pvc.yml` (or create the PVC alone if using a dynamic storage class).
4. **MongoDB Deployment:** `kubectl apply -f mongodb-deployment.yml`
5. **Backend & Frontend Services:** `kubectl apply -f backend-service.yml` and `kubectl apply -f frontend-service.yml`
6. **Kubernetes Secret:** `kubectl apply -f secrets.yml`
7. **Backend & Frontend Deployments:** `kubectl apply -f backend-deployment.yml` and `kubectl apply -f frontend-deployment.yml`
8. **Ingress (optional):** `kubectl apply -f ingress.yml`

Check the status of pods:

    kubectl get pods -n chat-app

Check services and ingress:

    kubectl get svc -n chat-app
    kubectl get ingress -n chat-app

If deploying locally without an ingress controller, you can port‑forward the services to access them from your host:

    # Forward backend API to localhost:5001
    kubectl port-forward service/backend -n chat-app 5001:5001

    # Forward frontend to localhost:80 (use sudo if binding to port 80)
    kubectl port-forward service/frontend -n chat-app 80:80

---

## Accessing the Application

Once all pods are running and services are ready, you can access the chat application via the forwarded ports or via the ingress host.  For example:

* **Frontend:** http://localhost
* **Backend API:** http://localhost:5001

![alt text](image-1.png)

![alt text](image-2.png)

![alt text](image-3.png)

If using the ingress controller, navigate to `http://localhost/` (or whichever host you configured) to see the frontend.  The backend API will be proxied through the same ingress.

---

## Troubleshooting Tips

* Use `kubectl describe pod <pod> -n chat-app` to see detailed information and events when a pod is stuck in `CrashLoopBackOff` or `Pending` status.
* Ensure the PersistentVolumeClaim is **Bound** before the MongoDB pod starts.  You can check with `kubectl get pvc -n chat-app`.
* Confirm that services have endpoints using `kubectl get endpoints -n chat-app`.
* In a kind cluster, ensure the `local-path` provisioner is running so that PVCs are provisioned automatically.
* Verify environment variables (MongoDB URI, JWT secret, ports) match your deployment.

---

## Summary

This guide demonstrated how to build Docker images for a Node.js/React chat application, push them to Docker Hub, and deploy the application on Kubernetes using a dedicated namespace, persistent storage for MongoDB, deployments, services, secrets, and ingress.  By following the steps above you can stand up a basic three‑tier chat application on any Kubernetes cluster.

