
# Craftista on Kubernetes – Deployment Guide

## Overview
`craftista-final.yaml` deploys the complete Craftista stack on Kubernetes:

- Namespace: `craftista`
- PostgreSQL database (PV, PVC, Deployment, Service)
- App services: `catalogue`, `recco`, `voting`, `frontend`
- NGINX `gateway` reverse proxy exposed via NodePort 32000
- All app images pulled from **private Harbor registry**
- Database credentials stored in Kubernetes Secret: `postgres-secret`
- App services configured with `imagePullSecrets: harbor-secret` to access private Harbor images

## Components
- **PostgreSQL**
  - Secret: `postgres-secret` (Opaque) with keys: `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
  - PV: `postgres-pv` (hostPath `/mnt/data/postgres`)
  - PVC: `postgres-pvc`
  - Deployment: `postgres`
  - Service: `postgres-service` (ClusterIP:5432)

- **Microservices**
  - `catalogue` (5000), `recco` (8080), `voting` (8080), `frontend` (3000)
  - Each uses `imagePullSecrets: harbor-secret`

- **NGINX Gateway**
  - Deployment: `gateway`
  - Service: `gateway-service` (NodePort 32000)

## Prerequisites
- Running Kubernetes cluster; `kubectl` configured
- Nodes can reach your Harbor registry (replace `<HARBOR_ADDRESS>` with your Harbor hostname/IP)
- Harbor credentials for image pull
- Ensure `/mnt/data/postgres` exists on the node running PostgreSQL (or update hostPath in YAML)

## 1) Secrets

### 1.1 PostgreSQL Secret
Defined inline in YAML as `postgres-secret`:

```yaml
POSTGRES_DB: craftista_db
POSTGRES_USER: craftista_user
POSTGRES_PASSWORD: craftista_pass
````

To create manually:

```bash
kubectl create namespace craftista || true
kubectl delete secret postgres-secret -n craftista --ignore-not-found
kubectl create secret generic postgres-secret \
  -n craftista \
  --from-literal=POSTGRES_DB=craftista_db \
  --from-literal=POSTGRES_USER=craftista_user \
  --from-literal=POSTGRES_PASSWORD=craftista_pass
```

### 1.2 Harbor Pull Secret

Required for Kubernetes to pull private images:

```bash
kubectl create namespace craftista || true
kubectl delete secret harbor-secret -n craftista --ignore-not-found
kubectl create secret docker-registry harbor-secret \
  --docker-server=<HARBOR_ADDRESS> \
  --docker-username="<HARBOR_USERNAME>" \
  --docker-password="<HARBOR_PASSWORD>" \
  --docker-email="<YOUR_EMAIL>" \
  -n craftista
```

## 2) Deploy

```bash
kubectl apply -f craftista-final.yaml
```

This creates the namespace, secrets (if kept in YAML), storage, Deployments, Services, and the gateway.

## 3) Verify

```bash
kubectl get pods -n craftista -o wide
kubectl get pvc -n craftista
kubectl get svc -n craftista
kubectl get events -n craftista --sort-by=.lastTimestamp | tail -n 50
```

All pods should reach `Running` status.

## 4) Access the Application

Gateway NodePort: `http://<any-node-ip>:32000`

* `/` → frontend
* `/api/catalogue` → catalogue
* `/api/recco` → recco
* `/api/voting` → voting

Example: `http://<NODE_IP>:32000`

## Scaling

```bash
kubectl scale deploy/frontend -n craftista --replicas=2
```

## Cleanup

```bash
kubectl delete -f craftista-final.yaml
kubectl delete namespace craftista --ignore-not-found
```

## Troubleshooting

* **ImagePullBackOff from Harbor**

  * Verify `harbor-secret` exists and credentials are valid
  * For self-signed Harbor CA, install CA on nodes and restart container runtime

* **Postgres not ready**

  * Check logs: `kubectl logs deploy/postgres -n craftista --tail=200`
  * Ensure `/mnt/data/postgres` exists

* **Gateway/Routes issues**

  * Check logs: `kubectl logs deploy/gateway -n craftista --tail=200`

* Ingress-NGINX (`ingress-nginx` namespace) is not required for this deployment

## Author

**Mehdi Esmati**
DevOps / Kubernetes Deployment
Contact: +98 915 387 0132
Email: mehdi.esmaty@gmail.com
