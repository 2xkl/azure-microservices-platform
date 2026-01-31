# Azure Microservices Platform

Azure-based microservices platform with AKS, ArgoCD GitOps, and modular Terraform infrastructure.

## Architecture

| Layer | Technology |
|-------|-----------|
| IaC | Terraform (modular) + Azure DevOps Pipelines |
| Compute | Azure Kubernetes Service (AKS) |
| Registry | Azure Container Registry (ACR) |
| Secrets | Azure Key Vault |
| TF State | Azure Storage Account (remote backend) |
| GitOps | ArgoCD (auto-sync, prune, self-heal) |
| Ingress | NGINX Ingress Controller |

## Project Structure

```
infra/
  backend-storage/        # Terraform state backend (Storage Account + KeyVault)
  core-infra/             # AKS cluster + ACR provisioning
    parameters/
      dev.tfvars
      prod.tfvars
  tf-modules/             # Reusable Terraform modules
    acr/
    aks/
    keyvault/
    keyvault-access-policy/
    resource-group/
    storage-account/
    storage-account-container/

k8s/
  manifests/
    argo/                 # ArgoCD Ingress (secms.local)
    users-api/            # Deployment, Service, Ingress + ArgoCD Application

services/
  users-api/              # Spring Boot 3.5 / Java 17 REST API
```

## Services

### users-api

Spring Boot application exposing a `GET /api/users` endpoint. Containerized with OpenJDK 17, built via Azure DevOps pipeline, pushed to ACR, and deployed to AKS through ArgoCD.

## Setup

### 1. Bootstrap Terraform Backend

Run the `backend-storage` pipeline to create:
- Resource Group
- Storage Account + Container (TF state)
- Key Vault (ARM credentials)

### 2. Create Service Principal

```bash
az ad sp create-for-rbac --name "sp-terraform" --role Contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>
```

Store the credentials (`ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`, `ARM_ACCESS_KEY`) in Key Vault.

### 3. Provision Infrastructure

Run the `core-infra` pipeline to create AKS and ACR.

### 4. Deploy ArgoCD + NGINX Ingress

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

helm install ingress-nginx ingress-nginx/ingress-nginx
```

### 5. Deploy Application

ArgoCD watches this repo and auto-syncs the `users-api` manifests to the cluster.

## Environments

| Environment | AKS Nodes | VM Size |
|-------------|-----------|---------|
| dev | 2 | Standard_DS2_v2 |
| prod | TBD | TBD |
