# InnovateMart – Project Bedrock  
**Retail Store Microservices Deployment on Amazon EKS**

---

## 1. Architecture Overview

Project Bedrock modernizes InnovateMart’s e-commerce platform by deploying a **microservices-based retail-store-sample-app** on **Amazon Elastic Kubernetes Service (EKS)**.  

### Key Components
- **Infrastructure as Code (IaC):**  
  - Terraform provisions the VPC, subnets, IAM roles, and the EKS cluster.  
  - Node groups are attached with the required IAM role for Kubernetes workloads.  

- **Kubernetes Workloads:**  
  - Microservices deployed include **catalog, orders, cart, checkout, and UI**.  
  - All workloads are managed in the `default` namespace.  
  - An **NGINX proxies layer** routes traffic cleanly between services.  

- **IAM & RBAC:**  
  - Admin access mapped via `aws-auth`.  
  - Developer access restricted using `dev-readonly` IAM user and Kubernetes RBAC.  

- **CI/CD Pipeline:**  
  - GitHub Actions runs Terraform in two stages:  
    - **Plan** on pull requests.  
    - **Apply** on merge to `main`.  
  - AWS credentials are managed via GitHub Secrets, no hardcoding.  

- **Bonus (Production Hardening):**  
  - External persistence:  
    - **Amazon RDS for PostgreSQL** → Orders service.  
    - **Amazon RDS for MySQL** → Catalog service.  
    - **Amazon DynamoDB** → Cart service.  
  - Networking & Security:  
    - **AWS Load Balancer Controller** for ingress.  
    - **Application Load Balancer (ALB)** exposes the UI service.  
    - SSL via AWS Certificate Manager (ACM).  
    - Optional domain configuration in Route 53.  

---

## 2. Deployment Guide

### 2.1 Infrastructure Setup
1. Navigate into the Terraform directory:  
   ``sh
   `cd terraform/eks/minimal`
   `terraform init`
   `terraform apply`

2. This provisions:
 - VPC, subnets, security groups.
 - EKS control plane + worker nodes.
 - Required IAM roles and OIDC provider.

### 2.2 Application Deployment
1. Deploy the app microservices:
`helm upgrade --install ui ./src/ui/chart -f ./terraform/eks/default/values/ui.yaml -n default
kubectl apply -f proxies.yaml`

2. Verify the pods are running:
`kubectl get pods -n default`

3. Access services internally:
`kubectl exec -it deploy/ui -n default -- curl http://proxies.default.svc.cluster.local:80/catalog/products`

### 2.3 Developer Access (Read-Only)
1. IAM user dev-readonly created with AmazonEKSClusterPolicy and a custom inline policy.
2. Mapped in aws-auth ConfigMap under mapUsers.
3. Kubernetes RBAC restricts to read-only:
   - ClusterRole: dev-readonly-role.
   - ClusterRoleBinding: dev-readonly-binding
4. Developer kubeconfig setup:
`aws eks update-kubeconfig \
  --name innovatemart-bedrock-eks \
  --region us-east-1 \
  --profile dev-readonly`
5. Allowed commands (examples):
`kubectl get pods
kubectl describe pod <name>
kubectl logs <name>`
Denied commands: Creating or deleting resources.

### 2.4 CI/CD Pipeline
1. GitHub Actions workflows:
   - .github/workflows/terraform-plan.yml → Runs on PR, executes terraform plan.
   - .github/workflows/terraform-apply.yml → Runs on merge to main, executes terraform apply.
2. AWS credentials pulled from GitHub Secrets (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY).


### 3. Bonus Implementation

### 3.1 Managed Databases
1. Orders → Amazon RDS PostgreSQL
2. Catalog → Amazon RDS MySQL
3. Cart → Amazon DynamoDB
4. Updated Kubernetes ConfigMaps to point microservices at these endpoints.
5 Secrets stored securely (no plaintext in Git).

### 3.2 Load Balancing & Ingress
1. Installed AWS Load Balancer Controller.
2. Created Kubernetes Ingress resource for the ui service.
3. Application exposed through an ALB with optional Route 53 DNS mapping.
4. HTTPS enabled via ACM-issued SSL certificate.


### 4. Access Instructions
1. Developer Access:
   - AWS IAM user: dev-readonly
   - Permissions: Read-only (logs, describe, init)
   - Connect via:
     `aws eks update-kubeconfig --name innovatemart-bedrock-eks --region us-east-1 --profile dev-readonly`


### 5. Deliverables
1. GitHub Repository:
https://github.com/Obidike-Chinedu/innovatemart-bedrock

2. Deployment & Architecture Guide:
This README.md (≈ 2 pages).
