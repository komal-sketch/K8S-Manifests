# Kubernetes Vertical Pod Autoscaler (VPA) - Apache Example

This project demonstrates how to configure a **Vertical Pod Autoscaler (VPA)** in Kubernetes.  
Unlike the Horizontal Pod Autoscaler (HPA) that scales the **number of pods**, VPA automatically adjusts the **CPU and memory requests/limits** of containers in a Deployment.

---

## 🚀 Steps Overview
1. Install VPA components  
2. Create Apache Deployment (re-use from HPA example)  
3. Configure VPA manifest  
4. Apply and test scaling behavior  

---

## 1. Install VPA Components

First, clone the Kubernetes Autoscaler repository:

    git clone https://github.com/kubernetes/autoscaler.git
    cd autoscaler/vertical-pod-autoscaler/

Run the installation script:

    ./hack/vpa-up.sh

Then go back to the root folder:

    cd ..

---

## 2. Vertical Pod Autoscaler Manifest

Now create a file named **vpa.yml**:

    apiVersion: autoscaling.k8s.io/v1
    kind: VerticalPodAutoscaler
    metadata:
      name: apache-vpa
      namespace: apache
    spec:
      targetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: apache-deployment
      updatePolicy:
        updateMode: "Auto"

Apply the manifest:

    kubectl apply -f vpa.yml

Check if VPA is created:

    kubectl get vpa -n apache

---

## 3. Port Forward for Access

Forward the Apache service to make it available locally:

    sudo -E kubectl port-forward service/apache-service -n apache 82:80 --address=0.0.0.0

Now the application can be accessed via:  
👉 http://localhost:82

---

## 4. Load Testing

Start a busybox pod to generate traffic:

    kubectl run -i --tty load-generator --image=busybox -n apache -- /bin/sh

Inside the busybox shell, run continuous requests:

    while true; do wget -q -O- http://apache-service.apache.svc.cluster.local; done

---

## 5. Observe VPA in Action

In another terminal, check VPA recommendations:

    watch kubectl get vpa -n apache

Monitor pod resource usage:

    kubectl top pod -n apache

As the load increases, you should see CPU/memory recommendations and resource adjustments applied by the VPA.

---

## 📝 Notes
- VPA adjusts pod resources (CPU/Memory) instead of creating new pods.  
- In **Auto** mode, pods may restart to apply new resource requests/limits.  
- Use VPA with workloads that have variable but predictable resource needs.  
- VPA can be used with or without HPA, but combining both requires careful planning.  

---

## 📌 References
- Kubernetes VPA Documentation: https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler  
- Metrics Server Installation: https://github.com/kubernetes-sigs/metrics-server  
- HPA vs VPA: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/  

---


