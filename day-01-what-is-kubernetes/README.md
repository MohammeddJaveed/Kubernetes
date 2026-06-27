# Day 01 — What is Kubernetes? 🤔

## The Problem First

Imagine you built a web app and packaged it as a **Docker container**. You run it on one server. Things are going well.

But then...

- Traffic spikes 10x → your single container crashes 💥
- Your server goes down → your whole app is gone 💀
- You want to update your app → you have to stop it, update, restart (downtime!) ⏳

**This is the problem Kubernetes solves.**

---

## What is Kubernetes?

Kubernetes (also written as **K8s**) is an open-source system that:

| Feature | What it means in plain English |
|---------|-------------------------------|
| **Auto-healing** | If your app crashes, K8s restarts it automatically |
| **Auto-scaling** | If traffic increases, K8s runs more copies of your app |
| **Load balancing** | Spreads traffic evenly across your app copies |
| **Rolling updates** | Updates your app with zero downtime |
| **Self-healing** | Kills unhealthy containers and replaces them |

Think of Kubernetes as the **manager** of your containers. You tell it *what you want*, and it makes sure it stays that way — forever.

---

## The Container → Pod → Cluster mental model

```
Your App Code
     ↓
  Docker Image       (packaged app)
     ↓
  Container          (running instance of the image)
     ↓
  Pod                (Kubernetes wrapper around container)
     ↓
  Node               (a machine/VM that runs Pods)
     ↓
  Cluster            (a group of Nodes managed by Kubernetes)
```

---

## Why not just use Docker?

Docker runs **one container on one machine**. That's it.

Kubernetes runs **many containers across many machines**, and manages them all as one system.

| Docker | Kubernetes |
|--------|-----------|
| Runs containers | Manages containers at scale |
| Manual restart if crash | Auto-restart on crash |
| Manual scaling | Auto-scaling |
| Single machine | Multiple machines (cluster) |

---

## Key Terms for Day 01

| Term | Simple Definition |
|------|------------------|
| **Cluster** | The whole Kubernetes setup (all machines together) |
| **Node** | A single machine (VM or physical) in the cluster |
| **Pod** | The smallest unit in K8s — wraps one or more containers |
| **kubectl** | The command-line tool to talk to Kubernetes |
| **Manifest / YAML file** | A file that describes what you want K8s to create |

---

## 🏃 Your First kubectl Commands

```bash
# Start minikube (your local kubernetes cluster)
minikube start

# Check the cluster is working
kubectl cluster-info

# See the nodes in your cluster
kubectl get nodes

# Check kubectl version
kubectl version --client
```

Expected output for `kubectl get nodes`:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   10m   v1.28.0
```

✅ If you see `Ready` — your cluster is up and running!

---

## 📝 Summary

- Kubernetes solves the problem of managing containers at scale
- It gives you auto-healing, auto-scaling, and load balancing
- A **Cluster** has **Nodes**, Nodes run **Pods**, Pods run **Containers**
- `kubectl` is how you talk to Kubernetes
- YAML files describe what you want Kubernetes to do

**Next → [Day 02: Architecture](../day-02-architecture/)**
