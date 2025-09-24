# Kubernetes Horizontal Pod Autoscaler (HPA) - Apache Example

This project demonstrates how to deploy an Apache HTTP server on Kubernetes, expose it via a service, manually scale the deployment, and then configure **Horizontal Pod Autoscaling (HPA)** based on CPU utilization.

---

## 🚀 Steps Overview
1. Create a namespace  
2. Deploy Apache HTTP server  
3. Expose it via a Service  
4. Test and manually scale the Deployment  
5. Configure and test HPA  

---

## 1. Create Namespace

    kubectl create namespace apache

---

## 2. Deployment (Apache Server)

Create a file named **deployment-hpa.yml**:

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
              image: httpd:latest
              ports:
                - containerPort: 80
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
                limits:
                  cpu: 200m
                  memory: 256Mi

Apply it:

    kubectl apply -f deployment-hpa.yml

---

## 3. Service (Expose Apache)

Create a file named **service.yml**:

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
          port: 80       # Exposed port inside cluster
          targetPort: 80 # Container port
      type: ClusterIP

Apply it:

    kubectl apply -f service.yml

Check resources:

    kubectl get all -n apache

---

## 4. Test Access

Inside the cluster:

    curl http://apache-service.apache.svc.cluster.local

Port-forward for external access:

    sudo -E kubectl port-forward service/apache-service -n apache 82:80 --address=0.0.0.0

Now access via:  
👉 http://localhost:82

---

## 5. Manual Scaling

Get current pods:

    kubectl get pods -n apache

Scale to 3 replicas:

    kubectl scale deployment apache-deployment -n apache --replicas=3

Verify:

    kubectl get pods -n apache

---

## 6. Configure Horizontal Pod Autoscaler (HPA)

Create a file named **hpa.yml**:

    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: apache-hpa
      namespace: apache
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: apache-deployment
      minReplicas: 1
      maxReplicas: 5
      metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 50

Apply it:

    kubectl apply -f hpa.yml

Check HPA status:

    kubectl get hpa -n apache

---

## 7. Load Testing

Run a busybox pod to generate load:

    kubectl run -i --tty load-generator --image=busybox -n apache -- /bin/sh

Inside pod, run infinite load:

    while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done

---

## 8. Observe Autoscaling

Monitor HPA:

    kubectl get hpa -n apache

Check pods:

    kubectl get pods -n apache

You should see the number of pods increase automatically as CPU usage grows.

---

## 📝 Notes
- Manual scaling (`kubectl scale`) is overridden by HPA once load triggers scaling.  
- Resource requests/limits must be defined for HPA to work.  
- This demo uses CPU utilization, but HPA can also scale based on custom metrics.  

---

## 📌 References
- Kubernetes HPA Documentation: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/  
- Busybox Image: https://hub.docker.com/_/busybox  

---
