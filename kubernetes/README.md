# Wanderlust Kubernetes Manifests & Architecture

This directory contains the production-ready declarative Kubernetes manifests for Wanderlust, managed via **GitOps with Argo CD** and exposed via **NGINX Ingress Controller + LoadBalancer**.

---

## 1. Kubernetes Architecture & Networking

All hardcoded VM IPs have been replaced with **stable Kubernetes internal DNS service names** and domain-level ingress routing.

```
                                [Client Browser / Internet]
                                             │
                                             ▼
                          [Cloud LoadBalancer (AWS NLB/ELB)]
                                             │
                                             ▼
                         [NGINX Ingress Controller (Port 80/443)]
                         (Reads ingress.yaml via ingressClassName)
                                  │                     │
                    ┌─────────────┘                     └─────────────┐
                    ▼                                                 ▼
             Path: /api                                            Path: /
    [backend-service (ClusterIP:8080)]             [frontend-service (ClusterIP:5173)]
                    │                                                 │
                    ▼                                                 ▼
       [backend-deployment (2 Pods)]                     [frontend-deployment (2 Pods)]
          │                     │
          ▼                     ▼
[mongo-service (ClusterIP:27017)]  [redis-service (ClusterIP:6379)]
          │                                  │
          ▼                                  ▼
[mongo-deployment (1 Pod + PVC)]     [redis-deployment (1 Pod)]
```

---

## 2. Manifest Inventory

| Manifest | Kind | Description | Service Type / Ports |
| :--- | :--- | :--- | :--- |
| `frontend.yaml` | `Deployment`, `Service` | React Vite frontend application (2 replicas, rolling update) | `ClusterIP` (Port 5173) |
| `backend.yaml` | `Deployment`, `Service` | Node.js Express backend API (2 replicas, rolling update) | `ClusterIP` (Port 8080) |
| `mongodb.yaml` | `Deployment`, `Service` | MongoDB database instance | `ClusterIP` (Port 27017) |
| `redis.yaml` | `Deployment`, `Service` | Redis caching instance | `ClusterIP` (Port 6379) |
| `persistentVolume.yaml` | `PersistentVolume` | Local or EBS PV storage definition | 5Gi storage |
| `persistentVolumeClaim.yaml` | `PersistentVolumeClaim` | PVC claimed by MongoDB deployment | 5Gi storage |
| `configmaps.yaml` | `ConfigMap` | Non-sensitive configs (`MONGODB_URI`, `REDIS_URL`, `PORT`, `FRONTEND_URL`) using internal DNS | N/A |
| `secrets.yaml` | `Secret` | Sensitive credentials (`JWT_SECRET`) Base64 encoded | N/A |
| `ingress.yaml` | `Ingress` | Domain routing rules mapping `/api` to backend and `/` to frontend | N/A |

---

## 3. Configuration Management

### ConfigMaps (`configmaps.yaml`)
- `MONGODB_URI`: Points to `mongodb://mongo-service.wanderlust.svc.cluster.local:27017/wanderlust` (internal cluster DNS name).
- `REDIS_URL`: Points to `redis://redis-service.wanderlust.svc.cluster.local:6379`.
- `PORT`: `8080`.
- `FRONTEND_URL`: Ingress domain name (e.g. `http://wanderlust.example.com`).
- `VITE_API_PATH`: Frontend API endpoint (e.g. `http://wanderlust.example.com/api`).

### Secrets (`secrets.yaml`)
- `JWT_SECRET`: Base64 encoded JSON Web Token signing key.
- To encode new secrets:
  ```bash
  echo -n "your-secret-key" | base64
  ```

---

## 4. Manual / Direct Deployment (Optional)

If deploying manually without Argo CD:

```bash
# 1. Create namespace
kubectl create namespace wanderlust

# 2. Apply Storage
kubectl apply -f persistentVolume.yaml -n wanderlust
kubectl apply -f persistentVolumeClaim.yaml -n wanderlust

# 3. Apply ConfigMaps and Secrets
kubectl apply -f configmaps.yaml -n wanderlust
kubectl apply -f secrets.yaml -n wanderlust

# 4. Deploy Databases
kubectl apply -f mongodb.yaml -n wanderlust
kubectl apply -f redis.yaml -n wanderlust

# 5. Deploy Workloads
kubectl apply -f backend.yaml -n wanderlust
kubectl apply -f frontend.yaml -n wanderlust

# 6. Apply Ingress Rules
kubectl apply -f ingress.yaml -n wanderlust
```

---

## 5. Verification Commands

```bash
# Verify pods status across wanderlust namespace
kubectl get pods -n wanderlust -o wide

# Verify ClusterIP services
kubectl get svc -n wanderlust

# Verify Ingress routing
kubectl get ingress -n wanderlust

# Check application logs
kubectl logs -l app=backend -n wanderlust --tail=50
kubectl logs -l app=frontend -n wanderlust --tail=50
```
