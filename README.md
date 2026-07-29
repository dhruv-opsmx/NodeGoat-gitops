# 🚀 NodeGoat Enterprise GitOps Platform

A production-style GitOps deployment of the OWASP NodeGoat application on **Amazon EKS** using **Argo CD**, **Helm**, **GitHub**, and **Amazon ECR**.

This repository follows the **App of Apps** GitOps pattern and manages three independent environments:

- Development
- Staging
- Production

This repository serves as the **GitOps Source of Truth**.

---

# Architecture

```
Developer
     │
     ▼
Git Commit
     │
     ▼
GitHub Repository
     │
     ▼
Argo CD (Root Application)
     │
     ▼
App of Apps
     │
 ┌───┴───────────────┐
 ▼                   ▼
Dev              Staging            Production
 │                   │                   │
 ├── MongoDB         ├── MongoDB         ├── MongoDB
 └── NodeGoat        └── NodeGoat        └── NodeGoat
```

---

# Technology Stack

- Amazon EKS
- Kubernetes
- Argo CD
- Helm
- GitHub
- GitHub Actions
- Amazon ECR
- AWS Load Balancer Controller
- MongoDB
- NodeGoat

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

## applications/

Contains Argo CD Application manifests.

Each file represents one deployable application.

Example:

```
applications/nodegoat/dev.yaml
```

deploys

```
charts/nodegoat
```

using

```
environments/dev/nodegoat-values.yaml
```

---

## charts/

Contains Helm charts.

```
charts/nodegoat
```

deploys NodeGoat.

```
charts/mongodb
```

deploys MongoDB.

---

## environments/

Environment-specific values.

```
dev/
staging/
production/
```

Only values differ.

Charts remain identical.

---

## bootstrap/

Everything required to bootstrap Argo CD.

Contains:

- AppProject
- Root Application
- Installation manifests

---

# GitOps Flow

```
Developer

↓

git push

↓

GitHub

↓

Argo CD detects change

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

# Environments

| Environment | Namespace |
|------------|-----------|
| Development | dev |
| Staging | staging |
| Production | production |

---

# Deploying From Scratch

## 1. Clone

```bash
git clone https://github.com/dhruv-opsmx/NodeGoat-gitops.git

cd NodeGoat-gitops
```

---

## 2. Configure kubectl

Verify you're connected to the EKS cluster.

```bash
kubectl config current-context
```

Expected:

```
arn:aws:eks:...
```

---

## 3. Install Argo CD

```bash
kubectl create namespace argocd

kubectl apply \
--server-side \
-n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## 4. Bootstrap

```bash
kubectl apply -f bootstrap/argocd/project.yaml

kubectl apply -f bootstrap/argocd/root.yaml
```

---

## 5. Verify

```bash
kubectl get applications -n argocd
```

Expected:

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

## Services

```bash
kubectl get svc -A
```

---

## Ingress

```bash
kubectl get ingress -A
```

---

## Describe an Application

```bash
kubectl describe application dev-nodegoat -n argocd
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

# Live Endpoints

## Argo CD UI

```
https://<ARGOCD-LOADBALANCER>
```

or

```
kubectl port-forward svc/argocd-server \
-n argocd \
8080:443
```

Open

```
https://localhost:8080
```

---

## NodeGoat

Development

```
http://<DEV-ALB>
```

Staging

```
http://<STAGING-ALB>
```

Production

```
http://<PRODUCTION-ALB>
```

---

# Updating the Application

Update any environment values.

Example:

```
environments/dev/nodegoat-values.yaml
```

Commit

```bash
git add .

git commit -m "Increase replicas"

git push
```

Argo CD will automatically synchronize the deployment.

---

# Troubleshooting

## Verify Kubernetes

```bash
kubectl get pods -A
```

---

## Verify Argo CD

```bash
kubectl get applications -n argocd
```

---

## Verify Repository

```bash
git remote -v
```

---

## Verify Cluster

```bash
kubectl config current-context
```

---

# Future Enhancements

- OpsMx SSD integration
- Progressive Delivery
- Blue/Green deployments
- Canary deployments
- Approval Gates
- Rollbacks
- Security Scanning
- GitHub Actions CI
- Policy Enforcement
- Monitoring & Dashboards

---

# Lessons Learned

Some notable issues encountered while building this platform:

- Incorrect Kubernetes context (Kind vs EKS)
- Argo CD installed into the wrong namespace
- AWS pod density limits requiring node group scaling
- Repository URL mismatches
- AppProject repository restrictions
- Child Application repository references
- App of Apps bootstrap troubleshooting

Keeping these documented should make future debugging much faster.

---

Built as a hands-on GitOps learning project and a foundation for enterprise deployment workflows with Argo CD and OpsMx SSD.