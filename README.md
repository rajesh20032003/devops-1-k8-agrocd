# micro-dash Helm Chart

Helm chart for the micro-dash microservices application.

##############
scaling hpa+vpa 
nw policy

done
deployment 
services 
###############
## Quick Start

\`\`\`bash
# create namespace
kubectl create namespace md

# install
helm install micro-dash . -n md

# verify
helm status micro-dash -n md
kubectl get pods -n md
\`\`\`

## What's included

| Resource | Details |
|----------|---------|
| Deployments | frontend, gateway, user-service, order-service |
| StatefulSets | postgres-user, postgres-ord |
| NetworkPolicies | 12 policies — zero trust |
| HPA | All 4 services |
| VPA | Both postgres StatefulSets |
| Ingress | nginx + TLS |
| RBAC | Roles, ClusterRoles, ServiceAccounts |
| Secrets | SealedSecrets for DB credentials |

## Access

\`\`\`bash
# SSH tunnel from laptop
ssh -i ~/.ssh/google_compute_engine \
  -L 7443:192.168.49.2:31471 \
  rajesh@34.133.110.141

# browser
https://micro-dash.local:7443
\`\`\`

## Related Repos

- [devops-1-k8-agrocd](https://github.com/rajesh20032003/devops-1-k8-agrocd) — raw K8s manifests
- [micro-dash-infra](https://github.com/rajesh20032003/micro-dash-infra) — Terraform AWS infra
