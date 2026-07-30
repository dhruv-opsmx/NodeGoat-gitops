# NodeGoat Enterprise GitOps Platform

A production-style multi-environment GitOps platform built on Amazon EKS using Argo CD, Helm and GitHub.

This project demonstrates how a single Git repository can declaratively manage Kubernetes applications across Development, Staging and Production environments using the App-of-Apps pattern.

> **Current Status:** ✅ Operational

---

# Features

- Multi-environment deployments
- App-of-Apps architecture
- Helm-based deployments
- GitOps workflow
- Amazon EKS
- AWS Load Balancer Controller
- Environment-specific configuration
- MongoDB StatefulSets
- Automatic reconciliation with Argo CD

---

# Architecture

```
                    Git Commit
                         │
                         ▼
                 GitHub Repository
                         │
                         ▼
                  Argo CD Root App
                         │
               App-of-Apps Pattern
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
      Development      Staging       Production
           │               │               │
      ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
      ▼         ▼     ▼         ▼     ▼         ▼
  MongoDB   NodeGoat MongoDB NodeGoat MongoDB NodeGoat
```

---

# Repository Structure

```
.
├── applications/
│   ├── mongodb/
│   └── nodegoat/
│
├── bootstrap/
│   └── argocd/
│
├── charts/
│   ├── mongodb/
│   └── nodegoat/
│
├── environments/
│   ├── dev/
│   ├── staging/
│   └── production/
│
└── README.md
```

---

# Environment Overview

| Environment | Namespace | Exposure |
|------------|-----------|----------|
| Development | `dev` | Internal ALB |
| Staging | `staging` | Internal ALB |
| Production | `production` | Internet-facing ALB |

---

# Deployment Flow

```
Developer

↓

git commit

↓

git push

↓

GitHub

↓

Argo CD detects changes

↓

Helm renders manifests

↓

Kubernetes updates cluster
```

No manual

```
kubectl apply
```

or

```
helm upgrade
```

should be necessary after bootstrap.

---

# Database Bootstrap

NodeGoat requires MongoDB seed data before the application can accept signups. The seed script creates required collections and counters, including the `userId` sequence used by the application.

Old workflow:

```
MongoDB

↓

NodeGoat

↓

Manual kubectl exec

↓

db:seed
```

That workflow depended on an operator running `npm run db:seed` inside a NodeGoat pod after deployment. It was not fully declarative because database initialization lived outside Git and outside Argo CD reconciliation.

New workflow:

```
Git Push

↓

Argo CD

↓

MongoDB

↓

Database Seed Job

↓

NodeGoat
```

The NodeGoat Helm chart now renders a first-class Kubernetes `Job` when `dbSeed.enabled` is `true`. The Job uses the same NodeGoat image and `MONGODB_URI` value as the Deployment, waits until MongoDB is reachable, then runs:

```bash
npm run db:seed
```

The Job is intentionally modeled as a bootstrap/reset mechanism because NodeGoat's seed script resets and repopulates the database. Enabling or disabling that behavior is controlled through Git-managed values:

```yaml
dbSeed:
  enabled: true
  backoffLimit: 2
  ttlSecondsAfterFinished: 300
  activeDeadlineSeconds: 600
  wait:
    maxAttempts: 60
    intervalSeconds: 5
    connectionTimeoutSeconds: 5
```

Argo CD sync waves order the seed Job before the NodeGoat Deployment when seeding is enabled. MongoDB is deployed by its own Application, so the Job also performs an active reachability check before running the seed command.

This is more GitOps-compliant because bootstrap intent, retry policy, image version, MongoDB connection settings, and environment-specific behavior are all declared in Git and reconciled by Kubernetes and Argo CD.

---

# Environments

| Environment | Namespace |
|------------|-----------|
| Development | dev |
| Staging | staging |
| Production | production |

---

# Bootstrap

## Clone

```bash
git clone https://github.com/dhruv-opsmx/NodeGoat-gitops.git

cd NodeGoat-gitops
```

---

## Verify Kubernetes Context

```bash
kubectl config current-context
```

---

## Install Argo CD

```bash
kubectl create namespace argocd

kubectl apply \
--server-side \
-n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## Bootstrap GitOps

```bash
kubectl apply -f bootstrap/argocd/project.yaml

kubectl apply -f bootstrap/argocd/root.yaml
```

---

# Verify

```bash
kubectl get applications -n argocd
```

Expected

```
root                  Healthy
dev-nodegoat          Healthy
dev-mongodb           Healthy
staging-nodegoat      Healthy
staging-mongodb       Healthy
production-nodegoat   Healthy
production-mongodb    Healthy
```

---

# Access

## Production

Public

```
http://<production-alb>
```

---

## Development

Internal AWS ALB

Accessible only from within the VPC or connected network.

---

## Staging

Internal AWS ALB

Accessible only from within the VPC or connected network.

---

## Argo CD

Currently exposed via Kubernetes port-forward.

```bash
kubectl port-forward svc/argocd-server \
-n argocd \
8080:443
```

Browse to

```
https://localhost:8080
```

Retrieve the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 --decode && echo
```

---

# Discover Endpoints

## Applications

```bash
kubectl get ingress -A
```

## Services

```bash
kubectl get svc -A
```

---

# GitOps Workflow

To deploy a change

1. Update Helm values or manifests.
2. Commit.
3. Push to GitHub.

Example

```bash
git add .

git commit -m "Increase dev replicas"

git push origin main
```

Argo CD will automatically reconcile the cluster.

No manual deployment commands are required.

---

# Useful Commands

## Applications

```bash
kubectl get applications -n argocd
```

---

## Pods

```bash
kubectl get pods -A
```

---

## Deployments

```bash
kubectl get deployments -A
```

---

## Ingress

```bash
kubectl get ingress -A
```

---

## Force Refresh

```bash
kubectl annotate application root \
-n argocd \
argocd.argoproj.io/refresh=hard \
--overwrite
```

---

# Troubleshooting

## Verify Current Context

```bash
kubectl config current-context
```

---

## Verify Cluster

```bash
kubectl get nodes
```

---

## Verify Applications

```bash
kubectl get applications -n argocd
```

---

## Verify Pods

```bash
kubectl get pods -A
```

---

## Verify Ingresses

```bash
kubectl get ingress -A
```

---

# Current Infrastructure

- Amazon EKS
- Amazon ECR
- GitHub
- GitHub Actions
- Argo CD
- Helm
- MongoDB
- AWS Load Balancer Controller

---

# Roadmap

- [x] Multi-environment GitOps
- [x] Helm
- [x] App-of-Apps
- [x] AWS ALB Ingress
- [x] Internal Dev & Staging
- [x] Public Production
- [ ] GitOps deployment validation
- [ ] OpsMx SSD integration
- [ ] Progressive Delivery
- [ ] Canary deployments
- [ ] Blue/Green deployments
- [ ] Approval Gates
- [ ] Automated Rollbacks
- [ ] Monitoring & Dashboards

---

# Lessons Learned

Some key operational challenges encountered while building this platform:

- Kubernetes context mismatches
- Argo CD bootstrap configuration
- AppProject repository restrictions
- Child Application repository references
- App of Apps bootstrap troubleshooting

Keeping these documented should make future debugging much faster.

---

Built as a hands-on GitOps learning project and a foundation for enterprise deployment workflows with Argo CD and OpsMx SSD.
