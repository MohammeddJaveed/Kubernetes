# ☸️ Kubernetes Learning Journey — 7 Days

> A beginner-friendly Kubernetes repo with real YAML files, simple explanations, and hands-on examples. Learn by doing!

---

## 👋 Who is this for?

This repo is for anyone who wants to learn Kubernetes from scratch — no prior experience needed. Every concept is explained in plain English, with working YAML files you can apply yourself.

---

## 📅 7-Day Roadmap

| Day                                              | Topic                           | What You'll Learn                 |
| ------------------------------------------------ | ------------------------------- | --------------------------------- |
| [Day 01](./day-01-what-is-kubernetes/)           | What is Kubernetes?             | The problem it solves, core ideas |
| [Day 02](./day-02-architecture/)                 | Architecture                    | Control plane, nodes, components  |
| [Day 03](./day-03-pods-deployments-replicasets/) | Pods, Deployments & ReplicaSets | How apps run in Kubernetes        |
| [Day 04](./day-04-services/)                     | Services                        | Load balancing, service discovery |
| [Day 05](./day-05-ingress/)                      | Ingress                         | Exposing apps to the internet     |
| [Day 06](./day-06-configmaps-secrets/)           | ConfigMaps & Secrets            | Managing config and passwords     |

---

## 🛠️ Prerequisites

- Docker basics (know what a container is)
- A terminal you're comfortable with
- **Minikube** installed → [Install Guide](https://minikube.sigs.k8s.io/docs/start/)
- **kubectl** installed → [Install Guide](https://kubernetes.io/docs/tasks/tools/)

### Start Minikube before any day's practice:

```bash
minikube start
```

---

## 🚀 How to use this repo

1. Go through each day **in order**
2. Read the `README.md` inside each day's folder
3. Apply the YAML files with `kubectl apply -f <filename>`
4. Run the debug commands to see what's happening
5. Clean up and move to the next day

---

## 🧹 Useful Commands Cheatsheet

```bash
# See everything running
kubectl get all

# Apply a YAML file
kubectl apply -f filename.yml

# Delete what you applied
kubectl delete -f filename.yml

# Describe a resource (great for debugging)
kubectl describe pod <pod-name>

# See logs from a pod
kubectl logs <pod-name>

# Get inside a running pod
kubectl exec -it <pod-name> -- /bin/bash

# Watch resources live
kubectl get pods -w
```

---

## ⭐ Star this repo if it helped you learn!

Feel free to open issues or PRs if you find anything confusing or want to add examples.
