# Day 06 — ConfigMaps & Secrets 🔐

## The Problem

Imagine you hardcode your database URL inside your app:
```python
db_url = "postgres://localhost:5432/mydb"
```

Now you want to deploy to production with a different database. You'd have to:
- Change the code
- Rebuild the Docker image
- Redeploy

That's terrible! **ConfigMaps and Secrets solve this.**

---

## ConfigMap vs Secret

| | ConfigMap | Secret |
|--|-----------|--------|
| **Stores** | Non-sensitive config | Sensitive data (passwords, tokens) |
| **Examples** | DB host, app port, feature flags | DB password, API keys, TLS certs |
| **Encoding** | Plain text | Base64 encoded |
| **When to use** | Config that's okay to see | Anything you'd never put in code |

---

## ConfigMap

A ConfigMap lets you store configuration separately from your app.

```
Without ConfigMap:
  Config is baked INTO the Docker image → must rebuild for every environment

With ConfigMap:
  Config lives in Kubernetes → change it without touching the image
```

### How to inject ConfigMap into a Pod

**Method 1: As Environment Variables**
```yaml
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: db_host
```

**Method 2: As a File (Volume)**
```yaml
volumeMounts:
- name: config-volume
  mountPath: /etc/config
```

---

## Secret

Secrets work exactly like ConfigMaps but are for sensitive data.

⚠️ **Important:** Kubernetes Secrets are base64 encoded, NOT encrypted by default. For real security, use:
- Sealed Secrets
- AWS Secrets Manager / GCP Secret Manager
- HashiCorp Vault

For learning/dev, the built-in Secrets work fine.

### How to encode values for secrets:
```bash
echo -n "mypassword" | base64
# Output: bXlwYXNzd29yZA==

# Decode:
echo "bXlwYXNzd29yZA==" | base64 --decode
# Output: mypassword
```

---

## 📁 YAML Files

### `configmap.yml`
```yaml
# ConfigMap — stores non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Key-value pairs — your app can read these
  db_host: "postgres-service"       # Database hostname
  db_port: "5432"                   # Database port
  app_port: "3000"                  # App port
  log_level: "info"                 # Logging level
  feature_dark_mode: "true"         # Feature flag
```

### `secret.yml`
```yaml
# Secret — stores sensitive data (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque                        # Generic secret type
data:
  # Values MUST be base64 encoded
  # echo -n "mydbpassword" | base64  →  bXlkYnBhc3N3b3Jk
  db_password: bXlkYnBhc3N3b3Jk
  # echo -n "mysecretapikey" | base64  →  bXlzZWNyZXRhcGlrZXk=
  api_key: bXlzZWNyZXRhcGlrZXk=
```

### `deployment-with-config.yml`
```yaml
# A deployment that uses both ConfigMap and Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx:latest
        ports:
        - containerPort: 3000

        # Inject ConfigMap values as environment variables
        env:
        - name: DB_HOST               # Env var name in the container
          valueFrom:
            configMapKeyRef:
              name: app-config        # Name of the ConfigMap
              key: db_host            # Key inside the ConfigMap

        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db_port

        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log_level

        # Inject Secret values as environment variables
        - name: DB_PASSWORD           # Env var name in the container
          valueFrom:
            secretKeyRef:
              name: app-secrets       # Name of the Secret
              key: db_password        # Key inside the Secret

        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api_key

        # Alternative: Load ALL ConfigMap keys as env vars at once
        envFrom:
        - configMapRef:
            name: app-config          # All keys become env vars
```

---

## 🏃 Hands-On Practice

```bash
# Create the ConfigMap
kubectl apply -f configmap.yml

# Create the Secret
kubectl apply -f secret.yml

# See them
kubectl get configmaps
kubectl get secrets

# View ConfigMap contents
kubectl describe configmap app-config

# View Secret (values are hidden!)
kubectl describe secret app-secrets

# Decode a secret value
kubectl get secret app-secrets -o jsonpath='{.data.db_password}' | base64 --decode

# Apply the deployment that uses them
kubectl apply -f deployment-with-config.yml

# Verify env vars are set inside the pod
kubectl get pods
kubectl exec -it <pod-name> -- env | grep -E "DB_HOST|DB_PORT|DB_PASSWORD|API_KEY"
```

### Create a ConfigMap directly (without YAML):
```bash
# From literal values
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2

# From a file
kubectl create configmap my-config --from-file=app.properties

# Create a Secret directly
kubectl create secret generic my-secret --from-literal=password=mysecretpassword
```

---

## 🔍 Debugging Commands

```bash
# See all configmaps
kubectl get configmaps

# See configmap contents
kubectl get configmap app-config -o yaml

# See all secrets (values are hidden)
kubectl get secrets

# Decode a specific secret value
kubectl get secret app-secrets -o jsonpath='{.data.db_password}' | base64 --decode

# Verify env vars inside a pod
kubectl exec -it <pod-name> -- env
kubectl exec -it <pod-name> -- printenv DB_HOST
```

---

## 📝 Summary

| | ConfigMap | Secret |
|--|-----------|--------|
| **Purpose** | Non-sensitive config | Passwords, keys, certs |
| **Encoding** | Plain text | Base64 |
| **Inject as** | Env var or volume file | Env var or volume file |
| **Example** | DB_HOST, LOG_LEVEL | DB_PASSWORD, API_KEY |

**Golden Rule:** Never hardcode config or secrets in your Docker image. Always use ConfigMaps and Secrets.

**Next → [Day 07: Real Project](../day-07-real-project/)**
