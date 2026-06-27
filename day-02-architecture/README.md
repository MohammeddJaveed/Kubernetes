# Day 02 — Kubernetes Architecture 🏗️

## The Big Picture

A Kubernetes cluster has two types of machines:

```
┌─────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER              │
│                                                  │
│  ┌──────────────────────┐                        │
│  │   CONTROL PLANE      │  ← The "Brain"         │
│  │  (Master Node)       │                        │
│  │                      │                        │
│  │  • API Server        │                        │
│  │  • etcd              │                        │
│  │  • Scheduler         │                        │
│  │  • Controller Mgr    │                        │
│  └──────────┬───────────┘                        │
│             │ manages                            │
│    ┌────────┴────────┐                           │
│    ▼                 ▼                           │
│  ┌──────┐        ┌──────┐  ← Worker Nodes        │
│  │Node 1│        │Node 2│    (where apps run)    │
│  │      │        │      │                        │
│  │ Pods │        │ Pods │                        │
│  └──────┘        └──────┘                        │
└─────────────────────────────────────────────────┘
```

---

## Control Plane Components (The Brain)

### 1. API Server (`kube-apiserver`)
- **The front door of Kubernetes**
- Every command you run with `kubectl` goes through the API Server
- It validates your request and stores it

```bash
# When you run this:
kubectl apply -f pod.yml

# kubectl talks to the API Server, which processes the request
```

### 2. etcd
- **The memory of Kubernetes**
- A key-value database that stores the entire cluster state
- What pods exist? What's their status? All stored here
- If etcd is gone, Kubernetes forgets everything

### 3. Scheduler (`kube-scheduler`)
- **Decides WHERE to run your pod**
- Looks at available nodes and picks the best one
- Considers: CPU available, memory available, and your requirements

```
You: "I want to run a pod"
Scheduler: "Node 2 has the most free memory, putting it there"
```

### 4. Controller Manager (`kube-controller-manager`)
- **Makes sure reality matches what you asked for**
- Runs many "controllers" in the background
- Example: You asked for 3 pods → Controller Manager keeps checking there are always 3

---

## Worker Node Components (Where Your Apps Run)

### 1. kubelet
- **The agent on each node**
- Talks to the API Server and makes sure pods are running
- Like a local manager on each machine

### 2. kube-proxy
- **Handles networking on each node**
- Makes sure pods can talk to each other
- Manages the network rules

### 3. Container Runtime
- **Actually runs the containers**
- Usually `containerd` or `Docker`
- kubelet tells the runtime to start/stop containers

---

## The Flow: What Happens When You Apply a YAML

```
kubectl apply -f pod.yml
       ↓
  API Server        (receives & validates request)
       ↓
   etcd             (stores: "user wants a pod")
       ↓
  Scheduler         (picks: "put it on Node 1")
       ↓
  kubelet           (on Node 1: "starting container now")
       ↓
Container Runtime   (actually runs the container)
       ↓
  Pod is Running ✅
```

---

## 🏃 Hands-On Commands

```bash
# See the nodes in your cluster
kubectl get nodes

# See ALL system components running
kubectl get pods -n kube-system

# Describe a node (see its resources, conditions)
kubectl describe node minikube

# See node resource usage
kubectl top node
```

### What you'll see in kube-system:
```
NAME                               READY   STATUS
coredns-xxx                        1/1     Running    ← DNS for the cluster
etcd-minikube                      1/1     Running    ← The database
kube-apiserver-minikube            1/1     Running    ← API Server
kube-controller-manager-minikube   1/1     Running    ← Controller Manager
kube-proxy-xxx                     1/1     Running    ← Network proxy
kube-scheduler-minikube            1/1     Running    ← Scheduler
```

---

## 📝 Summary

| Component | Lives In | Job |
|-----------|----------|-----|
| API Server | Control Plane | Front door — all commands go through it |
| etcd | Control Plane | Database — stores cluster state |
| Scheduler | Control Plane | Decides which node runs your pod |
| Controller Manager | Control Plane | Ensures desired state = actual state |
| kubelet | Worker Node | Runs and monitors pods on the node |
| kube-proxy | Worker Node | Handles networking |
| Container Runtime | Worker Node | Actually runs containers |

**Next → [Day 03: Pods, Deployments & ReplicaSets](../day-03-pods-deployments-replicasets/)**
