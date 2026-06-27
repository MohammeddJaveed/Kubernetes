# Day 05 — Kubernetes Ingress 🚦

## The Problem LoadBalancer Has

If you have 3 apps, and use LoadBalancer for each:
- App 1 → LoadBalancer → gets external IP #1 → costs money 💸
- App 2 → LoadBalancer → gets external IP #2 → costs money 💸
- App 3 → LoadBalancer → gets external IP #3 → costs money 💸

**That's expensive and messy!**

Also — what if you want:
- `myapp.com/api` → goes to the API service
- `myapp.com/shop` → goes to the shop service
- `myapp.com` → goes to the frontend

LoadBalancer can't do path-based routing. **Ingress can!**

---

## What is Ingress?

Ingress is like a **smart traffic cop** sitting at the entrance of your cluster.

```
Internet
    ↓
  Ingress (single entry point)
    ↓
  ┌─────────────────────────────────┐
  │ Rules:                          │
  │  /api      → API Service        │
  │  /shop     → Shop Service       │
  │  default   → Frontend Service   │
  └─────────────────────────────────┘
```

**One IP → many services** — that's the power of Ingress.

---

## Ingress vs Service

| Feature | Service (LoadBalancer) | Ingress |
|---------|----------------------|---------|
| Entry point per service | ✅ Yes (expensive!) | ❌ No (one for all) |
| Path-based routing | ❌ No | ✅ Yes |
| Host-based routing | ❌ No | ✅ Yes |
| SSL/HTTPS termination | ❌ No | ✅ Yes |
| Cost | One IP per service | One IP total |

---

## Ingress Controller

⚠️ **Important:** An Ingress resource (the YAML) is just rules. You also need an **Ingress Controller** — the actual software that reads those rules and routes traffic.

Popular Ingress Controllers:
- **NGINX Ingress Controller** (most common)
- Traefik
- HAProxy

```
Ingress YAML (rules)
       +
Ingress Controller (enforces the rules)
       =
Smart routing to your services ✅
```

---

## Types of Ingress Routing

### 1. Path-Based Routing
```
myapp.com/api   → api-service
myapp.com/shop  → shop-service
myapp.com/      → frontend-service
```

### 2. Host-Based Routing
```
api.myapp.com   → api-service
shop.myapp.com  → shop-service
myapp.com       → frontend-service
```

---

## 📁 YAML Files

### `ingress-path-based.yml`
```yaml
# Path-based routing: different paths go to different services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    # This tells which Ingress Controller to use
    kubernetes.io/ingress.class: "nginx"
    # Strip the path prefix before forwarding to service
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.local           # The domain name
    http:
      paths:
      - path: /api              # myapp.local/api → api-service
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /shop             # myapp.local/shop → shop-service
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 80
      - path: /                 # myapp.local/ → frontend-service
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### `ingress-host-based.yml`
```yaml
# Host-based routing: different subdomains go to different services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: api.myapp.local       # api.myapp.local → api-service
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: shop.myapp.local      # shop.myapp.local → shop-service
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 80
```

---

## 🏃 Hands-On with Minikube

```bash
# Step 1: Enable the NGINX Ingress Controller in Minikube
minikube addons enable ingress

# Verify it's running (wait a minute for it to start)
kubectl get pods -n ingress-nginx

# Step 2: Apply your ingress rules
kubectl apply -f ingress-path-based.yml

# See your ingress
kubectl get ingress

# Step 3: Get Minikube IP
minikube ip
# e.g. 192.168.49.2

# Step 4: Add to /etc/hosts so the domain works
# Add this line to /etc/hosts:
# 192.168.49.2  myapp.local

sudo echo "$(minikube ip) myapp.local" >> /etc/hosts

# Step 5: Test it!
curl http://myapp.local/api
curl http://myapp.local/shop
curl http://myapp.local/
```

---

## 🔍 Debugging Commands

```bash
# See all ingress resources
kubectl get ingress

# Describe ingress (see rules and events)
kubectl describe ingress path-based-ingress

# Check the ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# See if the backend services exist
kubectl get services
```

---

## 📝 Summary

- Ingress = a single entry point that routes to many services
- It supports path-based and host-based routing
- You need an **Ingress Controller** (e.g. NGINX) for it to work
- Saves money: one external IP instead of many
- Supports SSL termination (HTTPS) — great for production

**Next → [Day 06: ConfigMaps & Secrets](../day-06-configmaps-secrets/)**
