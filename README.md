# NodeGoat GitOps Repository

This repository is the GitOps source of truth for the NodeGoat demo platform on Amazon EKS.

## Architecture

- Developers change application configuration and Helm values in Git.
- GitHub Actions can update application images or repository content.
- Argo CD watches this repository and deploys the desired state into Amazon EKS.
- Helm charts under charts/ define the application manifests.
- Environment-specific values live under environments/.

## Repository Structure

- bootstrap/argocd/ - Argo CD installation and bootstrap manifests
- charts/mongodb/ - Helm chart for MongoDB
- charts/nodegoat/ - Helm chart for the NodeGoat application
- environments/dev/ - values for the dev environment
- environments/staging/ - values for the staging environment
- environments/production/ - values for the production environment

## Argo CD Workflow

1. Apply the bootstrap manifests from bootstrap/argocd/.
2. Argo CD creates the root application and child applications.
3. Child applications deploy the MongoDB and NodeGoat Helm charts.
4. Helm renders manifests from charts/ using values from environments/.
5. Argo CD keeps the cluster state aligned with Git.

## Deployment Flow

- MongoDB uses the chart in charts/mongodb and values from environments/*/mongodb-values.yaml.
- NodeGoat uses the chart in charts/nodegoat and values from environments/*/nodegoat-values.yaml.
- The chart defaults in charts/*/values.yaml remain the baseline; environment files should only contain overrides that differ by environment.
- Argo CD uses automatic sync, prune, self-heal, and namespace creation.
