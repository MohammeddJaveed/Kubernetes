# Day 04 — Kubernetes Services 🌐

## The Problem Services Solve

Pods are **temporary**. They get created, die, and get replaced with new ones that have **different IP addresses** every time.

So how do other apps find your pods? How does traffic reach them?

**Answer: Services!**

A Service gives your pods a **stable address** that never changes — even as pods come and go.

---

## What a Service Does

```
Without Service:
  User → Pod IP (10.0.0.5) → Pod dies → IP changes → User is lost ❌

With Service:
  User → Service IP (stable!) → Service finds healthy pods → Routes traffic ✅
```

A Service does 3 things:
1. **Load Balancing** — spreads traffic across all healthy pods
2. **Service Discovery** — gives pods a stable DNS name to find each other
3. **Expose to world** — lets external traffic reach your pods

---

## 3 Types of Services

### Type 1: ClusterIP (Default)
- **Only accessible INSIDE the cluster**
- Pods can talk to each other using this
- External users cannot reach it
- Use case: internal microservice communication

```
Internet ❌ → ClusterIP Service → Pod
Pod A ✅ → ClusterIP Service → Pod B
```

### Type 2: NodePort
- **Accessible from OUTSIDE the cluster**
- Opens a port on every Node (30000–32767 range)
- You access it via: `<node-ip>:<node-port>`
- Use case: development and testing

```
Internet ✅ → Node IP : 30007 → NodePort Service → Pod
```

### Type 3: LoadBalancer
- **Accessible from outside with a real IP address**
- Creates a cloud load balancer (AWS ELB, GCP LB, etc.)
- Gives you a single external IP
- Use case: production on cloud

```
Internet ✅ → External IP (e.g. 34.56.78.90) → LoadBalancer Service → Pods
```

---

## Visual Comparison

```
┌─────────────────────────────────────────────┐
│                  CLUSTER                     │
│                                              │
│  ClusterIP  ←→  Pods  (internal only)        │
│                                              │
│  NodePort   ←→  Pods  (node:port from outside)│
│                                              │
│  LoadBalancer ←→ Pods (real IP from outside) │
└─────────────────────────────────────────────┘
```

---

## How Services Find Pods — Labels & Selectors

Services use **labels** to find which pods to route to. This is the key concept!

```yaml
# Pod has a label:
labels:
  app: nginx

# Service selects pods with that label:
selector:
  app: nginx
```

The Service doesn't care about Pod names or IPs — just labels. So even when pods are replaced, the Service keeps finding the new ones (as long as they have the right label).

---

## 📁 YAML Files

### `clusterip-service.yml`
```yaml
# ClusterIP — only accessible inside the cluster
# Other pods can reach nginx using: http://nginx-clusterip-service:80
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip-service
spec:
  type: ClusterIP           # Default type
  selector:
    app: nginx              # Route to pods with label app=nginx
  ports:
  - protocol: TCP
    port: 80                # Port the Service listens on
    targetPort: 80          # Port on the Pod to forward to
```

### `nodeport-service.yml`
```yaml
# NodePort — accessible from outside via <NodeIP>:30007
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80                # Port inside the cluster
    targetPort: 80          # Port on the Pod
    nodePort: 30007         # External port (must be 30000-32767)
```

### `loadbalancer-service.yml`
```yaml
# LoadBalancer — gets an external IP (works with cloud providers or minikube tunnel)
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

---

## 🏃 Hands-On Practice

First, make sure your nginx deployment from Day 03 is running:
```bash
kubectl apply -f ../day-03-pods-deployments-replicasets/deployment.yml
```

### ClusterIP
```bash
kubectl apply -f clusterip-service.yml
kubectl get services

# Test it from inside the cluster (from another pod)
kubectl run test-pod --image=busybox --rm -it -- /bin/sh
# Inside the pod:
wget -O- http://nginx-clusterip-service:80
exit
```

### NodePort
```bash
kubectl apply -f nodeport-service.yml
kubectl get services

# With Minikube — get the URL:
minikube service nginx-nodeport-service --url

# Open it in browser or curl it:
curl <the-url-from-above>
```

### LoadBalancer
```bash
kubectl apply -f loadbalancer-service.yml
kubectl get services

# With Minikube — run this in a SEPARATE terminal:
minikube tunnel

# Now check for external IP:
kubectl get services nginx-loadbalancer-service
# You'll see an EXTERNAL-IP assigned!

curl http://<EXTERNAL-IP>
```

---

## 🔍 Debugging Commands

```bash
# See all services
kubectl get services
kubectl get svc          # Short form

# Describe a service (see which pods it's selecting)
kubectl describe service nginx-nodeport-service

# See endpoints (actual pod IPs the service is routing to)
kubectl get endpoints

# If service isn't working — check if selector matches pod labels
kubectl get pods --show-labels
```

---

## 📝 Summary

| Type | Who Can Access | Use Case |
|------|---------------|----------|
| ClusterIP | Only pods inside cluster | Microservice communication |
| NodePort | Anyone with node IP + port | Dev/testing |
| LoadBalancer | Anyone on internet (with cloud) | Production |

**Key insight:** Services use **label selectors** to find pods. Always make sure your pod labels match your service selector!

**Next → [Day 05: Ingress](../day-05-ingress/)**
