# micro-dash-gitops

Kubernetes manifests for the micro-dash microservices application.
Implements zero trust networking with Calico CNI on Minikube, migrating to EKS.

## Architecture
browser → frontend (nginx:80) → gateway (:3000)
├── user-service (:3001) → postgres-user (:5432)
└── order-service (:3002) → postgres-ord (:5432)

## Stack

| Tool | Purpose |
|------|---------|
| Kubernetes | Container orchestration |
| Calico CNI | Network policy enforcement |
| Minikube | Local cluster |
| Harbor | Private image registry |
| Helm | Package manager (WIP) |
| ArgoCD | GitOps CD (WIP) |
| EKS | Production cluster (WIP) |

## Structure
manifests/
├── deployments/    # frontend, gateway, user, order service
├── services/       # ClusterIP, NodePort services
├── storage/        # StatefulSets, PVCs for postgres
├── network/        # NetworkPolicies (zero trust)
└── rbac/           # Roles, ClusterRoles, ServiceAccounts
charts/             # Helm charts (WIP)
argocd/             # ArgoCD app definitions (WIP)

## Zero Trust Network Policy

Default deny-all ingress with explicit allow rules per service pair.

| From | To | Port | Status |
|------|----|------|--------|
| browser | frontend | 80 | ALLOWED |
| frontend | gateway | 3000 | ALLOWED |
| gateway | user-service | 3001 | ALLOWED |
| gateway | order-service | 3002 | ALLOWED |
| user-service | postgres-user | 5432 | ALLOWED |
| order-service | postgres-ord | 5432 | ALLOWED |
| any other | any | any | BLOCKED |

See [network policy documentation](docs/network-policy.md) for full test results.

## Quick Start
```bash
# start minikube with calico
minikube start --cni=calico --insecure-registry="YOUR_HARBOR_IP"

# create namespace and secrets
kubectl apply -f manifests/namespace.yaml
kubectl create secret docker-registry harbor-cred \
  --docker-server=YOUR_HARBOR_IP \
  --docker-username=admin \
  --docker-password=YOUR_PASSWORD \
  --namespace=md

# deploy everything
kubectl apply -f manifests/storage/
kubectl apply -f manifests/deployments/
kubectl apply -f manifests/services/
kubectl apply -f manifests/network/
kubectl apply -f manifests/rbac/
```

## Related Repos

- [micro-dash-app](https://github.com/rajesh20032003/micro-dash-app) — source code + Jenkins CI
- [micro-dash-infra](https://github.com/rajesh20032003/micro-dash-infra) — Terraform AWS infra
