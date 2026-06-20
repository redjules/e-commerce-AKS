# 🤖 Stan's Robot Shop — E-Commerce on AKS

> Deploy a **full e-commerce microservices application** onto **Azure Kubernetes Service (AKS)** using Helm charts, StatefulSets, Persistent Volumes, and an Ingress controller. Based on the [Instana Robot Shop](https://github.com/instana/robot-shop) project by IBM.

---

## 📋 Table of Contents

- [🗺️ Project Overview](#️-project-overview)
- [🧩 Microservices Architecture](#-microservices-architecture)
- [🗄️ Databases & Storage](#️-databases--storage)
- [🛒 Application Workflow](#-application-workflow)
- [✅ Prerequisites](#-prerequisites)
- [☁️ Step 1 — Create an AKS Cluster](#️-step-1--create-an-aks-cluster)
- [🔌 Step 2 — Connect to the Cluster](#-step-2--connect-to-the-cluster)
- [📦 Step 3 — Clone the Repository](#-step-3--clone-the-repository)
- [⛵ Step 4 — Deploy with Helm](#-step-4--deploy-with-helm)
- [🔍 Step 5 — Verify the Deployment](#-step-5--verify-the-deployment)
- [🌐 Step 6 — Configure Ingress](#-step-6--configure-ingress)
- [💾 Understanding Persistent Volumes](#-understanding-persistent-volumes)
- [⛵ Understanding Helm Charts](#-understanding-helm-charts)
- [🐳 Containerisation by Language](#-containerisation-by-language)
- [🛠️ Troubleshooting](#️-troubleshooting)
- [🔗 Resources](#-resources)

---

## 🗺️ Project Overview

This project deploys **Stan's Robot Shop** — a realistic, minimalist e-commerce application — onto an AKS cluster. It is purpose-built as a learning tool, with each microservice written in a **different programming language** so you gain hands-on experience with containerising diverse tech stacks.

### 🌟 Steps

- Deploying **multi-microservice apps** on AKS
- Working with **Deployments** and **StatefulSets**
- Configuring **Persistent Volumes** and **Storage Classes**
- Using **Helm charts** for environment-aware deployments
- Setting up an **Application Gateway Ingress Controller**

---

## 🧩 Microservices Architecture

```
🌍 User (Browser)
      │
      ▼
 🌐 Web (Nginx)          ← Front-end UI
      │
      ├──▶ 👤 User        ← Registration & login        → 🍃 MongoDB
      ├──▶ 📂 Catalogue   ← Product listings & images   → 🐬 MySQL
      ├──▶ 🛒 Cart        ← Shopping cart state         → ⚡ Redis
      ├──▶ ⭐ Ratings     ← Product ratings              → ⚡ Redis
      ├──▶ 🚚 Shipping    ← Shipping cost calculator
      ├──▶ 💳 Payment     ← Payment processing
      └──▶ 📦 Dispatch    ← Order dispatch notifications
```

### 📋 Service Summary

| Service        | Language | Role                          | Storage |
| -------------- | -------- | ----------------------------- | ------- |
| `web` 🌐       | Nginx    | Front-end UI server           | —       |
| `user` 👤      | —        | Registration & authentication | MongoDB |
| `catalogue` 📂 | Node.js  | Product listings & categories | MySQL   |
| `cart` 🛒      | Node.js  | Shopping cart management      | Redis   |
| `ratings` ⭐   | PHP      | Product ratings               | Redis   |
| `shipping` 🚚  | Java     | Shipping cost calculation     | —       |
| `payment` 💳   | Python   | Payment processing            | —       |
| `dispatch` 📦  | Go       | Order dispatch notifications  | —       |

---

## 🗄️ Databases & Storage

| Component      | Type            | Purpose                          |
| -------------- | --------------- | -------------------------------- |
| 🐬 **MySQL**   | Relational DB   | Product images, catalogue data   |
| 🍃 **MongoDB** | NoSQL DB        | User accounts, unstructured data |
| ⚡ **Redis**   | In-memory store | Ratings cache, cart session data |

### 💡 Why Redis for Ratings?

> Ratings are **highly dynamic** — thousands of users can rate a product within minutes. An in-memory store like Redis retrieves this data far faster than a traditional database, avoiding latency under heavy read loads.

Redis is deployed as a **StatefulSet** connected to a **Persistent Volume** so ratings survive pod restarts.

---

## 🛒 Application Workflow

```
1. 📝 Register / Log in
2. 🔍 Browse categories (AI, Robots)
3. 🤖 Select a robot → view details & ratings
4. ⭐ Rate the product
5. 🛒 Add to cart (quantity selectable)
6. 🚚 Calculate shipping by region
7. 💳 Confirm & pay
8. 📦 Receive dispatch confirmation
```

---

## ✅ Prerequisites

- 🔷 **Azure CLI** installed and authenticated
- ☸️ **kubectl** installed
- ⛵ **Helm** installed (`v3+`)
- 📁 This repository cloned locally
- 🆓 Azure subscription (free tier works, watch regional quotas)

---

## ☁️ Step 1 — Create an AKS Cluster

1. In the Azure Portal, search for **Kubernetes services** → **Create**.
2. Create a new **Resource Group** (e.g. `ecommerce-demo`).
3. Recommended settings:

   | Setting            | Value                                      |
   | ------------------ | ------------------------------------------ |
   | Cluster preset     | Dev/Test                                   |
   | Cluster name       | `three-tier`                               |
   | Region             | West US 2 _(change if quota issues arise)_ |
   | Availability zones | Zone 1                                     |
   | AKS pricing tier   | Free                                       |
   | Scale method       | Manual or Autoscale (min 1, max 2)         |

4. Click **Review + Create** → **Create**. ⏳ Allow 10–15 minutes.

> ⚠️ **Quota errors?** Switch to a different region — this is the most common fix on free subscriptions.

---

## 🔌 Step 2 — Connect to the Cluster

```bash
# Pull kubeconfig and merge into local config
az aks get-credentials \
  --resource-group ecommerce-demo \
  --name three-tier

# ✅ Verify you're connected to the right cluster
kubectl config current-context
# Expected output: three-tier

kubectl get pods
# Should return "No resources found" (cluster is empty)
```

---

## 📦 Step 3 — Clone the Repository

```bash
git clone <REPO_URL>
cd three-tier-architecture-demo

# Navigate to the AKS Helm chart directory
cd AKS/helm
```

---

## ⛵ Step 4 — Deploy with Helm

```bash
# 1️⃣ Create a dedicated namespace
kubectl create namespace robot-shop

# 2️⃣ Install the Helm chart into that namespace
helm install robot-shop . --namespace robot-shop
```

> ⏳ Pod creation takes a couple of minutes. Redis and the databases spin up first; application services follow.

---

## 🔍 Step 5 — Verify the Deployment

```bash
# Watch all pods come up in the robot-shop namespace
kubectl get pods -n robot-shop

# Check services (note the web service external IP / NodePort)
kubectl get svc -n robot-shop

# Inspect the Redis StatefulSet's Persistent Volume Claim
kubectl get pvc -n robot-shop
```

**Expected pod states — all should be `Running` ✅:**

```
NAME              READY   STATUS    RESTARTS
cart-xxx          1/1     Running   0
catalogue-xxx     1/1     Running   0
dispatch-xxx      1/1     Running   0
mongodb-0         1/1     Running   0
mysql-0           1/1     Running   0
payment-xxx       1/1     Running   0
rabbitmq-0        1/1     Running   0
ratings-xxx       1/1     Running   0
redis-0           1/1     Running   0
shipping-xxx      1/1     Running   0
user-xxx          1/1     Running   0
web-xxx           1/1     Running   0
```

---

## 🌐 Step 6 — Configure Ingress

### Enable the Application Gateway Ingress Controller

1. In the Azure Portal, go to your **AKS cluster**.
2. Click **Networking** in the left menu.
3. ✅ Check **Enable Ingress controller** (Application Gateway).
4. Click **Apply** — allow 5–10 minutes.

### Apply the Ingress Resource

```bash
kubectl apply -f ingress.yaml -n robot-shop
```

**`ingress.yaml`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: robot-shop-ingress
  namespace: robot-shop
spec:
  ingressClassName: azure-application-gateway # 🏷️ ties this to the AGIC
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web # 🌐 front-end service
                port:
                  number: 8080
```

### Verify the Ingress Address

```bash
kubectl get ingress -n robot-shop
# Wait until ADDRESS column is populated, then open it in your browser 🌍
```

> 💡 **Why use Ingress instead of a LoadBalancer service?**
>
> | LoadBalancer Service      | Ingress Controller                   |
> | ------------------------- | ------------------------------------ |
> | One public IP per service | One public IP for all services       |
> | No routing rules          | Path-based & host-based routing      |
> | No WAF                    | Web Application Firewall support     |
> | Limited features          | TLS termination, rewrites, redirects |

### 🏷️ Ingress Class Names — Why They Matter

If multiple teams share a cluster using **different Ingress controllers** (e.g. one team uses NGINX, another uses Azure Application Gateway), the `ingressClassName` field tells each controller which Ingress resources it should manage — preventing conflicts.

---

## 💾 Understanding Persistent Volumes

Redis and the databases use **StatefulSets** with **Persistent Volume Claims (PVCs)** to survive pod restarts.

```yaml
# Excerpt from redis-statefulset.yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: default # 🗄️ Azure Managed Disk
      resources:
        requests:
          storage: 1Gi
```

### 📦 Azure Storage Classes

```bash
kubectl get storageclass
```

| Storage Class     | Azure Equivalent   | Best For                                                   |
| ----------------- | ------------------ | ---------------------------------------------------------- |
| `default`         | Azure Managed Disk | **Single pod** access (like EBS on AWS)                    |
| `azurefile`       | Azure Files        | **Multi-pod / multi-node** shared access (like EFS on AWS) |
| `managed-premium` | Azure Premium SSD  | High-performance workloads                                 |

> 🔑 **Rule of thumb:**
>
> - One pod reading/writing → use **Azure Disk** (`default`)
> - Multiple pods or nodes sharing a volume → use **Azure Files** (`azurefile`)

---

## ⛵ Understanding Helm Charts

```
helm/
├── Chart.yaml          # 📋 Chart metadata & version
├── values.yaml         # 🎛️ Dynamic configuration values
└── templates/          # 📁 All Kubernetes manifest templates
    ├── web-deployment.yaml
    ├── cart-deployment.yaml
    ├── redis-statefulset.yaml
    ├── mysql-statefulset.yaml
    └── ... (one file per service)
```

### 🎛️ How Templating Works

Inside any template file, dynamic values use **Go/Jinja-style** syntax:

```yaml
# templates/payment-deployment.yaml
image: {{ .Values.image.repo }}/payment:{{ .Values.image.version }}
env:
  - name: PAYMENT_GATEWAY
    value: {{ .Values.paymentGateway }}
```

These are resolved from `values.yaml` at install time:

```yaml
# values.yaml
image:
  repo: robotshop
  version: latest
paymentGateway: "stripe"
```

### 🌍 Why Helm over plain YAML?

| Plain kubectl YAML                   | Helm Charts                          |
| ------------------------------------ | ------------------------------------ |
| 24 separate files to manage          | All templates in one chart           |
| Must duplicate files per environment | Change `values.yaml` per environment |
| Hard to version & rollback           | `helm rollback` in one command       |
| No parameterisation                  | Full Go templating support           |

---

## 🐳 Containerisation by Language

Each microservice has its own `Dockerfile`. Here's the pattern per language:

### 🟢 Node.js (cart, catalogue)

```dockerfile
FROM node:lts-alpine
WORKDIR /usr/src/app
COPY package.json .          # 📦 dependency manifest (like requirements.txt)
RUN npm install              # install dependencies
COPY server.js .             # copy source code
CMD ["node", "server.js"]    # start the app
```

### 🐍 Python (payment)

```dockerfile
FROM python:3-slim
WORKDIR /usr/src/app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

### 🔵 Go (dispatch)

```dockerfile
FROM golang:alpine AS build
COPY go.mod .
RUN go mod download
COPY *.go .
RUN go build -o dispatch        # compile binary

FROM alpine                     # 🪶 tiny runtime image
COPY --from=build /dispatch .
CMD ["/dispatch"]
```

### ☕ Java (shipping) — Multi-stage Build

```dockerfile
# 🔨 Stage 1: Build (heavy image with Maven)
FROM maven:3-jdk-11 AS build
COPY pom.xml .
RUN mvn dependency:resolve
COPY src/ src/
RUN mvn package

# 🚀 Stage 2: Runtime (lightweight JRE only)
FROM openjdk:11-jre
COPY --from=build /target/shipping.jar /opt/shipping/
CMD ["java", "-jar", "/opt/shipping/shipping.jar"]
```

> 💡 **Multi-stage builds** drastically reduce final image size — the Maven build tools stay in Stage 1 and never reach production.

### 📊 Dependency File Cheatsheet

| Language | Dependency File    | Install Command                   |
| -------- | ------------------ | --------------------------------- |
| Node.js  | `package.json`     | `npm install`                     |
| Python   | `requirements.txt` | `pip install -r requirements.txt` |
| Go       | `go.mod`           | `go mod download`                 |
| Java     | `pom.xml`          | `mvn package`                     |
| Rust     | `Cargo.toml`       | `cargo build`                     |

---

## 🛠️ Troubleshooting

| ⚠️ Symptom                          | 🔍 Likely Cause                      | ✅ Fix                                                  |
| ----------------------------------- | ------------------------------------ | ------------------------------------------------------- |
| Pods stuck in `Pending`             | PVC not bound / no storage class     | Check `kubectl get pvc -n robot-shop` and storage class |
| `ImagePullBackOff`                  | Wrong image name or private registry | Verify image tags in `values.yaml`                      |
| Ingress `ADDRESS` stays empty       | AGIC still provisioning              | Wait 5–10 min; delete and recreate the ingress pod      |
| AKS creation quota error            | Regional resource limits             | Switch to a different Azure region                      |
| `failed to list` error in AGIC logs | Intermittent AGIC issue              | Delete the AGIC pod and let it restart                  |
| App loads but ratings missing       | Redis PVC not bound                  | Check `kubectl describe pod redis-0 -n robot-shop`      |
| Cannot connect with `kubectl`       | Wrong context                        | Run `kubectl config current-context` to verify          |

---

## 🔗 Resources

- 🤖 [Instana Robot Shop (original)](https://github.com/instana/robot-shop)
- ☸️ [AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- ⛵ [Helm Documentation](https://helm.sh/docs/)
- 🌐 [Azure Application Gateway Ingress Controller](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview)
- 💾 [AKS Storage Classes](https://learn.microsoft.com/en-us/azure/aks/concepts-storage)
- 🐳 [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)

---

> 💬 _Deployed successfully? Try extending it — add TLS to the Ingress, set resource limits per pod, or wire up a real payment gateway!_
> ⭐ _Found this useful? Star the repo!_
