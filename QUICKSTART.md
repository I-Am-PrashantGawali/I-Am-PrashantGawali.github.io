# Quick Start Guide - Portfolio Website Deployment

## Important Note
I cannot directly access your AWS account. You need to run these commands on your local machine after configuring AWS CLI with your credentials.

## Pre-configuration Required

### 1. Install AWS CLI
Download from: https://aws.amazon.com/cli/

### 2. Configure AWS Credentials
```bash
aws configure
```
You'll need:
- AWS Access Key ID
- AWS Secret Access Key
- Default region: `us-east-1`
- Default output format: `json`

### 3. Install Docker
Download from: https://www.docker.com/

### 4. Install kubectl
Download from: https://kubernetes.io/docs/tasks/tools/

### 5. Install eksctl (for EKS)
Download from: https://eksctl.io/

## Quick Deployment Steps

### Step 1: Create ECR Repository
```bash
aws ecr create-repository --repository-name portfolio-website --region us-east-1
```

### Step 2: Login to ECR
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 132551339390.dkr.ecr.us-east-1.amazonaws.com
```

### Step 3: Build and Push Docker Image
```bash
cd I-Am-PrashantGawali.github.io
docker build -t 132551339390.dkr.ecr.us-east-1.amazonaws.com/portfolio-website:latest .
docker push 132551339390.dkr.ecr.us-east-1.amazonaws.com/portfolio-website:latest
```

### Step 4: Create EKS Cluster (if you don't have one)
```bash
eksctl create cluster --name portfolio-cluster --region us-east-1 --nodes 2 --node-type t3.medium
```

### Step 5: Install ArgoCD on EKS
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 6: Install NGINX Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

### Step 7: Install cert-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

### Step 8: Apply ClusterIssuers
```bash
kubectl apply -f gitops/cert-manager/cluster-issuer-staging.yaml
kubectl apply -f gitops/cert-manager/cluster-issuer.yaml
```

### Step 9: Get Ingress Controller IP
```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```
Copy the EXTERNAL-IP and add an A record in your DNS provider:
- Domain: `portfolio.prashantgawali.site`
- Value: `<EXTERNAL-IP>`

### Step 10: Apply ArgoCD Application
```bash
kubectl apply -f gitops/argocd-application.yaml
```

### Step 11: Sync Application
```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Get ArgoCD initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open browser to http://localhost:8080
# Login with username: admin and the password from above

# Sync the application
argocd app sync portfolio-website
```

### Step 12: Verify Deployment
```bash
kubectl get pods -l app=portfolio-website
kubectl get ingress portfolio-website-ingress
```

### Step 13: Access Your Website
Open: https://portfolio.prashantgawali.site

## Configuration Summary

Your configuration is pre-configured with:
- **AWS Account ID**: 132551339390
- **AWS Region**: us-east-1
- **Email**: prashantgawali2807@gmail.com
- **Domain**: portfolio.prashantgawali.site
- **ECR Repository**: portfolio-website
- **Image**: 132551339390.dkr.ecr.us-east-1.amazonaws.com/portfolio-website:latest

## Need Help?
See the detailed deployment guide in `DEPLOYMENT.md` for troubleshooting and advanced configurations.
