# Deployment Guide

This document covers the step-by-step deployment order for RideShare Pro on AWS EKS. Follow each phase in sequence, later steps depend on earlier ones being healthy.

---

## Prerequisites

Before running any of the steps below, ensure you have the following tools installed and configured locally:

- `kubectl` configured against your EKS cluster
- `helm` v3+
- `aws` CLI authenticated with sufficient IAM permissions
- `eksctl` (optional, for cluster bootstrap)

Verify your cluster context before proceeding:

```bash
kubectl config current-context
kubectl get nodes
```

---

## Phase 1 - AWS Infrastructure

### 1.1 Apply Node Groups

```bash
kubectl apply -f aws/node-groups.yaml
```

Wait for all nodes to reach `Ready` status:

```bash
kubectl get nodes -w
```

### 1.2 Apply Storage Classes

EBS gp3 storage classes must exist before any StatefulSet is deployed.

```bash
kubectl apply -f aws/storage-classes.yaml
```

### 1.3 Apply IAM Roles (IRSA)

These roles allow pods to authenticate with AWS services (Secrets Manager, EBS CSI, etc.) without static credentials.

```bash
kubectl apply -f aws/iam-roles.yaml
```

---

## Phase 2 - Cluster-Level Platform Components

Install these components in order. They are required by everything that follows.

### 2.1 NGINX Ingress Controller

```bash
kubectl apply -f platform/ingress/nginx-ingress.yaml
```

Wait for the controller pod and its LoadBalancer service to become ready:

```bash
kubectl get pods -n ingress-nginx -w
kubectl get svc -n ingress-nginx
```

Note the external LoadBalancer hostname/IP, you will need this for DNS.

### 2.2 cert-manager

cert-manager handles automatic TLS certificate provisioning via Let's Encrypt.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Wait for cert-manager webhooks to be ready before continuing:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=90s
```

### 2.3 External Secrets Operator

```bash
kubectl apply -f platform/secrets/external-secrets-operator.yaml
```

Wait for the ESO controller to be running:

```bash
kubectl get pods -n external-secrets -w
```

### 2.4 Cluster Autoscaler

```bash
kubectl apply -f platform/autoscaling/cluster-autoscaler.yaml
```

---

## Phase 3 - Stateful Workloads

### 3.1 Deploy PostgreSQL

```bash
kubectl apply -f stateful/postgres/postgres-primary-replica.yaml
```

Wait for the primary pod to be running before proceeding:

```bash
kubectl get pods -l app=postgres -w
```

### 3.2 Create Databases

Once the primary PostgreSQL pod is running, exec into it and create the required databases for each service:

```bash
kubectl exec -it postgres-0 -- psql -U postgres
```

Inside the psql shell:

```sql
CREATE DATABASE riders_db;
CREATE DATABASE drivers_db;
CREATE DATABASE trips_db;
EXIT
```

### 3.3 Store Credentials in AWS Secrets Manager

After databases are created, store all sensitive values (DB passwords, connection strings, API keys) in AWS Secrets Manager. These will be pulled into the cluster by the External Secrets Operator in the next phase.

```bash
aws secretsmanager create-secret \
  --name rideshare/postgres \
  --secret-string '{"username":"postgres","password":"<your-password>"}'

# Repeat for any other secrets required by your services
```

### 3.4 Deploy Redis Cluster

```bash
kubectl apply -f stateful/redis/redis-cluster.yaml
```

Wait for all 6 Redis pods (3 masters, 3 replicas) to be running:

```bash
kubectl get pods -l app=redis -w
```

---

## Phase 4 - Secrets Wiring

With secrets now stored in AWS Secrets Manager and the External Secrets Operator running, apply the SecretStore and ExternalSecret resources to sync them into the cluster.

```bash
kubectl apply -f platform/secrets/secret-store.yaml
kubectl apply -f platform/secrets/external-secrets.yaml
```

Verify secrets were synced successfully:

```bash
kubectl get externalsecrets
kubectl get secrets
```

All ExternalSecrets should show `STATUS: SecretSynced`.

---

## Phase 5 - Application Workloads

Apply each service's manifests. The order within this phase does not matter strictly, but deploy all of them before applying ingress rules.

```bash
# Rider Service
kubectl apply -f applications/rider-service/configmap.yaml
kubectl apply -f applications/rider-service/deployment.yaml
kubectl apply -f applications/rider-service/service.yaml
kubectl apply -f applications/rider-service/hpa.yaml

# Driver Service
kubectl apply -f applications/driver-service/configmap.yaml
kubectl apply -f applications/driver-service/deployment.yaml
kubectl apply -f applications/driver-service/service.yaml
kubectl apply -f applications/driver-service/hpa.yaml

# Trip Service
kubectl apply -f applications/trip-service/configmap.yaml
kubectl apply -f applications/trip-service/deployment.yaml
kubectl apply -f applications/trip-service/service.yaml
kubectl apply -f applications/trip-service/hpa.yaml

# Matching Service
kubectl apply -f applications/matching-service/configmap.yaml
kubectl apply -f applications/matching-service/deployment.yaml
kubectl apply -f applications/matching-service/service.yaml
kubectl apply -f applications/matching-service/hpa.yaml

# Email Service
kubectl apply -f applications/email-service/configmap.yaml
kubectl apply -f applications/email-service/deployment.yaml
kubectl apply -f applications/email-service/service.yaml
kubectl apply -f applications/email-service/hpa.yaml

# Frontend
kubectl apply -f applications/frontend/configmap.yaml
kubectl apply -f applications/frontend/deployment.yaml
kubectl apply -f applications/frontend/service.yaml
kubectl apply -f applications/frontend/hpa.yaml
```

Verify all pods are running:

```bash
kubectl get pods
```

---

## Phase 6 - Security Policies

```bash
kubectl apply -f platform/security/pod-disruption-budgets.yaml
```

---

## Phase 7 - Ingress Rules

Apply ingress routing rules last, after all backend services are healthy. Applying them before the services are ready will result in 503 errors.

```bash
kubectl apply -f platform/ingress/ingress-rules.yaml
```

Verify the ingress was created and has an address assigned:

```bash
kubectl get ingress rideshare
```

Check that the TLS certificate was issued:

```bash
kubectl get certificate
kubectl describe certificate tls-rideshare
```

The certificate status should show `Ready: True`.

---

## Verification & Testing

Once all phases are complete, verify the deployment end-to-end.

### Check All Resources

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl get hpa
kubectl get pvc
```

### Test Endpoints

```bash
# Frontend
curl -I https://rideshare.ijeaweledivine.online/

# Driver Service
curl https://rideshare.ijeaweledivine.online/api/v1/drivers

# Rider Service
curl https://rideshare.ijeaweledivine.online/api/v1/riders

# Trip Service
curl https://rideshare.ijeaweledivine.online/api/trips

# Matching Service
curl https://rideshare.ijeaweledivine.online/api/v1/matching

# Email Service
curl https://rideshare.ijeaweledivine.online/api/v1/email

# WebSocket (Trip Service)
# wss://rideshare.ijeaweledivine.online/ws
```



---

## Deployment Order Summary

| Phase | Step | Manifest |
|---|---|---|
| 1 | Node Groups | `aws/node-groups.yaml` |
| 1 | Storage Classes | `aws/storage-classes.yaml` |
| 1 | IAM Roles | `aws/iam-roles.yaml` |
| 2 | NGINX Ingress Controller | `platform/ingress/nginx-ingress.yaml` |
| 2 | cert-manager | upstream manifest |
| 2 | External Secrets Operator | `platform/secrets/external-secrets-operator.yaml` |
| 2 | Cluster Autoscaler | `platform/autoscaling/cluster-autoscaler.yaml` |
| 3 | PostgreSQL StatefulSet | `stateful/postgres/postgres-primary-replica.yaml` |
| 3 | Create Databases | manual (exec into pod) |
| 3 | Store Secrets | AWS Secrets Manager (CLI) |
| 3 | Redis StatefulSet | `stateful/redis/redis-cluster.yaml` |
| 4 | SecretStore + ExternalSecrets | `platform/secrets/secret-store.yaml` + `external-secrets.yaml` |
| 5 | All Application Services | `applications/*/` |
| 6 | Pod Disruption Budgets | `platform/security/pod-disruption-budgets.yaml` |
| 7 | Ingress Rules | `platform/ingress/ingress-rules.yaml` |
