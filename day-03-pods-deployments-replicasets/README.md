# Day 03 — Pods, Deployments & ReplicaSets 📦

## The Difference (Most Important Day!)

This confuses a lot of beginners. Here's the simplest way to think about it:

```
Container  →  just the running app process
Pod        →  a wrapper around container(s) with a Kubernetes address
ReplicaSet →  makes sure N copies of a Pod are always running
Deployment →  manages ReplicaSets + gives you rolling updates & rollbacks
```

**In real life, you almost always create a Deployment — never a bare Pod.**

---

## 1. Pod — The Smallest Unit

A Pod is the smallest thing Kubernetes manages. It wraps one (or more) containers and gives them:
- A shared IP address
- Shared storage (volumes)
- Shared network namespace

### Why not just use containers directly?
Because Kubernetes doesn't manage containers — it manages Pods. A Pod adds the "Kubernetes layer" on top.

### ⚠️ Pod Problem
If you create a Pod manually and it crashes → **it stays dead**. No one restarts it.
This is why we use ReplicaSets and Deployments.

---

## 2. ReplicaSet — Auto-Healing

A ReplicaSet watches your Pods and says:
> "You asked for 3 pods. I see only 2. Let me create one more."

This is **auto-healing**. The moment a Pod dies, the ReplicaSet creates a new one.

### How auto-healing works:
```
You: "I want 3 replicas of nginx"
ReplicaSet: Creates 3 pods ✅

Pod 2 crashes 💥
ReplicaSet: "Only 2 pods! Creating a new one..." ✅
Now back to 3 pods
```

---

## 3. Deployment — The Real Deal

A Deployment does everything a ReplicaSet does, PLUS:
- **Rolling updates** — update your app with zero downtime
- **Rollbacks** — made a bad update? Roll back instantly
- **Pause/Resume** — pause a rollout if something looks wrong

```
Deployment
    └── manages → ReplicaSet
                      └── manages → Pods
                                       └── runs → Container
```

**Always use Deployments in real projects!**

---

## 📁 YAML Files

### `pod.yml` — A basic Nginx Pod
```yaml
# This creates a single Pod running nginx
# If it crashes, it will NOT restart automatically
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod           # Name of the pod
  labels:
    app: nginx              # Label — used to select/filter this pod
spec:
  containers:
  - name: nginx             # Name of the container inside the pod
    image: nginx:latest     # Docker image to use
    ports:
    - containerPort: 80     # Port the container listens on
```

### `replicaset.yml` — 3 copies of Nginx
```yaml
# This ensures 3 nginx pods are ALWAYS running
# If one dies, a new one is created automatically (auto-healing!)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3               # Always keep 3 pods running
  selector:
    matchLabels:
      app: nginx            # Manage pods with this label
  template:
    metadata:
      labels:
        app: nginx          # Label on the pods it creates
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

### `deployment.yml` — Production-ready Nginx
```yaml
# A Deployment is what you use in real projects
# It manages ReplicaSets, gives you rolling updates and rollbacks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3               # Run 3 pods at all times
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate     # Update pods one by one (zero downtime)
    rollingUpdate:
      maxSurge: 1           # Allow 1 extra pod during update
      maxUnavailable: 1     # Allow 1 pod to be down during update
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21   # Specific version (not latest) — good practice!
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"  # Minimum memory needed
            cpu: "250m"     # 250 millicores = 0.25 CPU
          limits:
            memory: "128Mi" # Max memory allowed
            cpu: "500m"     # Max CPU allowed
```

---

## 🏃 Hands-On Practice

```bash
# --- POD ---
# Create the pod
kubectl apply -f pod.yml

# See it running
kubectl get pods

# Get more details
kubectl describe pod nginx-pod

# See logs from the container
kubectl logs nginx-pod

# Delete the pod (it won't come back!)
kubectl delete pod nginx-pod


# --- REPLICASET ---
kubectl apply -f replicaset.yml

# See 3 pods running
kubectl get pods

# Try deleting one pod — watch it come back!
kubectl delete pod <pod-name-from-above>
kubectl get pods    # A new one appears! That's auto-healing.

# Scale up to 5 replicas
kubectl scale replicaset nginx-replicaset --replicas=5
kubectl get pods


# --- DEPLOYMENT ---
kubectl apply -f deployment.yml

# See the deployment
kubectl get deployments
kubectl get replicasets
kubectl get pods

# Update the image (rolling update — zero downtime!)
kubectl set image deployment/nginx-deployment nginx=nginx:1.25

# Watch the rolling update happen live
kubectl rollout status deployment/nginx-deployment

# See rollout history
kubectl rollout history deployment/nginx-deployment

# Oops, bad update? Roll back!
kubectl rollout undo deployment/nginx-deployment

# Scale up/down
kubectl scale deployment nginx-deployment --replicas=5
```

---

## 🔍 Debugging Commands

```bash
# See all pods with more info
kubectl get pods -o wide

# Describe a pod (events section is very useful for debugging!)
kubectl describe pod <pod-name>

# See logs
kubectl logs <pod-name>

# Follow logs live
kubectl logs -f <pod-name>

# Get inside a running pod
kubectl exec -it <pod-name> -- /bin/bash
```

---

## 📝 Summary

| Object | Auto-Restart? | Rolling Update? | Rollback? | Use When? |
|--------|--------------|-----------------|-----------|-----------|
| Pod | ❌ No | ❌ No | ❌ No | Never (learning only) |
| ReplicaSet | ✅ Yes | ❌ No | ❌ No | Rarely (use Deployment) |
| Deployment | ✅ Yes | ✅ Yes | ✅ Yes | **Always in real projects** |

**Next → [Day 04: Services](../day-04-services/)**
