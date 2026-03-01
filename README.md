# RideShare Kubernetes Deployment on AWS EKS

A production-grade, microservices-based ride-sharing platform deployed on Amazon EKS. Built to handle real-world scale, traffic spikes, and high availability requirements.

---

## Live Endpoint

**Base URL:** `https://rideshare.ijeaweledivine.online`

| Service | Path |
|---|---|
| Frontend | `https://rideshare.ijeaweledivine.online/` |
| Driver Service | `https://rideshare.ijeaweledivine.online/api/v1/drivers` |
| Rider Service | `https://rideshare.ijeaweledivine.online/api/v1/riders` |
| Trip Service | `https://rideshare.ijeaweledivine.online/api/trips` |
| Matching Service | `https://rideshare.ijeaweledivine.online/api/v1/matching` |
| Email Service | `https://rideshare.ijeaweledivine.online/api/v1/email` |
| WebSocket (Trip) | `wss://rideshare.ijeaweledivine.online/ws` |

> TLS is handled automatically via cert-manager with a Let's Encrypt production certificate.

---

## System Architecture

```
Internet
   │
   ▼
NGINX Ingress Controller  (TLS termination, rate limiting, routing)
   │
   ├──► rider-service        (TypeScript / Node.js)
   ├──► driver-service       (TypeScript / Node.js)
   ├──► trip-service         (Python)  ◄── WebSocket on port 3006
   ├──► matching-service     (Go)
   ├──► email-service        (Python)
   └──► rideshare-frontend   (TypeScript / React)
            │
            ├── PostgreSQL  (StatefulSet: 1 primary + 2 read replicas)
            └── Redis       (StatefulSet: 6-node cluster)
```

All services sit behind the NGINX Ingress Controller. Secrets are pulled from AWS Secrets Manager via the External Secrets Operator. Stateful workloads use Amazon EBS gp3 persistent volumes.

---

## Repository Structure

```
rideshare-k8s/
├── README.md
├── ARCHITECTURE.md
├── DEPLOYMENT.md
├── aws/
│   ├── node-groups.yaml
│   ├── iam-roles.yaml
│   └── storage-classes.yaml
├── platform/
│   ├── ingress/
│   │   ├── nginx-ingress.yaml
│   │   └── ingress-rules.yaml
│   ├── secrets/
│   │   ├── external-secrets-operator.yaml
│   │   ├── secret-store.yaml
│   │   └── external-secrets.yaml
│   ├── autoscaling/
│   │   └── cluster-autoscaler.yaml
│   └── security/
│       └── pod-disruption-budgets.yaml
├── stateful/
│   ├── redis/
│   │   └── redis-cluster.yaml
│   └── postgres/
│       └── postgres-primary-replica.yaml
└── applications/
    ├── rider-service/
    ├── driver-service/
    ├── trip-service/
    ├── matching-service/
    ├── email-service/
    └── frontend/
```

---

## Key Features

**Infrastructure**
- EKS 1.33+ with managed node groups across multiple availability zones
- EBS CSI driver with gp3 StorageClass and volume expansion enabled
- IRSA (IAM Roles for Service Accounts) for secure, scoped AWS access

**Workloads**
- All stateless services run as Deployments with Horizontal Pod Autoscalers (HPA)
- Redis and PostgreSQL run as StatefulSets with persistent EBS volumes and stable network identity
- Pod Disruption Budgets ensure availability during node maintenance

**Secrets Management**
- Sensitive credentials stored in AWS Secrets Manager
- External Secrets Operator syncs secrets into Kubernetes automatically
- No plaintext secrets in any manifest

**Traffic Management**
- NGINX Ingress Controller handles all inbound traffic
- Rate limiting: 10 RPS per client with a burst multiplier of 3
- TLS termination with auto-renewed Let's Encrypt certificates
- Path-based routing to all services from a single domain

**Autoscaling**
- HPA on all stateless services (CPU and memory metrics)
- Cluster Autoscaler adjusts node count based on pending pod demand


> See `DEPLOYMENT.md` for the full step-by-step deployment guide.
