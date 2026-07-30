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

Previous non-GitOps workflow:

```
MongoDB

↓

NodeGoat

↓

Out-of-band database seed

↓

db:seed
```

That workflow depended on an operator action after deployment. It was not fully declarative because database initialization lived outside Git and outside Argo CD reconciliation.

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

The NodeGoat Helm chart renders a first-class Kubernetes `Job` when `dbSeed.enabled` is `true`. The Job uses the same NodeGoat image and `MONGODB_URI` value as the Rollout, waits until MongoDB is reachable, then runs:

```bash
npm run db:seed
```

The Job is intentionally modeled as a bootstrap/reset mechanism because NodeGoat's seed script resets and repopulates the database. Enabling or disabling that behavior is controlled through Git-managed values:

```yaml
dbSeed:
  enabled: false
  backoffLimit: 2
  ttlSecondsAfterFinished: 300
  activeDeadlineSeconds: 600
  wait:
    maxAttempts: 60
    intervalSeconds: 5
    connectionTimeoutSeconds: 5
```

Argo CD sync waves order the seed Job before the NodeGoat Rollout when seeding is enabled. MongoDB is deployed by its own Application, so the Job also performs an active reachability check before running the seed command.

This is more GitOps-compliant because bootstrap intent, retry policy, image version, MongoDB connection settings, and environment-specific behavior are all declared in Git and reconciled by Kubernetes and Argo CD.

---

# Enterprise Bootstrap Strategy

The database seed Job is intended only for initial environment bootstrap or an intentional reset. It should not participate in normal reconciliation forever because NodeGoat's `db:seed` command resets and repopulates application data.

Recommended GitOps workflow:

```
Commit:
dbSeed.enabled=true

↓

Argo CD runs bootstrap Job

↓

Commit:
dbSeed.enabled=false
```

Future Argo CD syncs should render no seed Job and therefore should not recreate or rerun database seeding. Development may enable this intentionally during lab setup or reset. Production keeps it disabled by default.

This design keeps bootstrap declarative without Helm hooks, initContainers, lifecycle hooks, imperative scripts, or manual pod execution.

---

# Lab Mode

Lab Mode is a declarative cost-saving switch for non-active lab environments:

```yaml
labMode:
  enabled: true
```

When enabled in the NodeGoat values, the Rollout renders with:

```yaml
replicas: 0
```

When enabled in the MongoDB values, the StatefulSet renders with:

```yaml
replicas: 0
```

PVC definitions remain in the MongoDB StatefulSet, so existing persistent volumes are not removed by Lab Mode. The Argo CD Applications continue to reconcile desired Kubernetes resources, but runtime pods are paused.

Enable Lab Mode by committing `labMode.enabled: true` in the target environment's NodeGoat and MongoDB values files. Disable Lab Mode by committing `labMode.enabled: false`. To resume the lab, disable Lab Mode and let Argo CD reconcile NodeGoat and MongoDB back to their configured replica counts.

Do not combine Lab Mode with `dbSeed.enabled: true`: MongoDB is intentionally scaled to zero in Lab Mode, so bootstrap should be performed only after the environment is resumed.

---

# AWS Cost Optimization

Lab Mode reduces AWS spend by stopping the compute portions of the lab:

```
NodeGoat Rollout replicas: 0

MongoDB StatefulSet replicas: 0

PVCs retained for later resume
```

This can reduce EKS worker-node pressure and application runtime resource consumption while preserving GitOps health and MongoDB data volumes. Storage costs for retained PVCs can still apply, and any shared infrastructure such as the EKS control plane, load balancers, NAT gateways, or persistent volumes may continue to incur cost.

---

# Progressive Delivery

NodeGoat is managed by Argo Rollouts using a Blue/Green strategy:

```
Git Push

↓

Argo CD renders Helm chart

↓

Rollout creates new ReplicaSet

↓

Preview ReplicaSet becomes healthy

↓

Promotion switches stable Service to new ReplicaSet
```

The stable Service name remains unchanged, so the Ingress and AWS Load Balancer Controller continue to route through the same Kubernetes Service. A preview Service can be enabled through values when a separate in-cluster preview endpoint is needed:

```yaml
rollout:
  strategy:
    blueGreen:
      autoPromotionEnabled: false
      previewService:
        enabled: true
        port: 80
```

`autoPromotionEnabled: false` means new versions pause before they become active. This gives platform operators or higher-level delivery automation a controlled promotion point while keeping the desired rollout strategy in Git.

Argo Rollouts CRDs and controller must be installed in the cluster as platform infrastructure before this chart is synced.

---

# Rollback Strategy

Rollback is handled declaratively by reverting the Git change that introduced the bad image tag, configuration, or chart change. Argo CD reconciles the previous desired state and Argo Rollouts shifts the stable Service back to the healthy ReplicaSet.

Blue/Green keeps rollback straightforward because traffic is controlled by Service selector switching rather than in-place pod mutation. The stable Service remains the routing contract for ALB and users.

---

# GitOps Lifecycle

```
Developer commit

↓

Git repository

↓

Argo CD App of Apps

↓

Helm render

↓

MongoDB StatefulSet

↓

Optional database bootstrap Job

↓

NodeGoat Rollout

↓

Stable Service

↓

ALB Ingress
```

The platform remains fully declarative: scaling, lab pause/resume, bootstrap, rollout strategy, promotion readiness, and rollback intent are all represented as Git-managed configuration.

---

# Session Architecture Limitation

NodeGoat is a demo application and currently uses `express-session` with the default `MemoryStore`. That store keeps session data inside the memory of the Node.js process that handled the login request.

This means NodeGoat cannot safely run multiple replicas unless sessions are externalized. With `replicaCount: 2`, each pod has its own private session memory:

```
Browser

↓

AWS ALB

↓

NodeGoat Pod A  ── local MemoryStore

NodeGoat Pod B  ── local MemoryStore
```

A user can log in successfully on Pod A, then refresh the browser and have the next request routed by the ALB to Pod B. Pod B does not have the session created by Pod A, so the user appears unauthenticated and is redirected back to login.

For this GitOps platform, `replicaCount: 1` is intentional for NodeGoat. It keeps session behavior deterministic without runtime patches, manual scaling, ALB stickiness, or changes outside Git. This is unrelated to the MongoDB seed Job; the seed Job initializes application data, while the login refresh issue is caused by process-local session storage.

Production-grade Node.js applications should use a shared session store with `express-session`, such as:

- `connect-mongo`
- `connect-redis`

Once sessions are stored outside the application pod, the Rollout can safely scale horizontally because every replica can read and write the same session data.

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

## Rollouts

```bash
kubectl get rollouts -A
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
- [x] Progressive Delivery
- [ ] Canary deployments
- [x] Blue/Green deployments
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
