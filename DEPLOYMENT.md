# Deployment Guide - Portfolio Website with AWS ECR and ArgoCD

## Prerequisites
- AWS CLI installed and configured
- Docker installed
- Kubernetes cluster with ArgoCD installed
- Git repository access
- Domain name: portfolio.prashantgawali.site

## Step 1: Configure AWS CLI Credentials

```bash
# Install AWS CLI if not already installed
# Windows: Download from https://aws.amazon.com/cli/

# Configure AWS credentials
aws configure

# Enter your AWS Access Key ID and Secret Access Key
# Default region: us-east-1
# Default output format: json
```

**Note:** You need an AWS IAM user with the following permissions:
- ECR: CreateRepository, GetLoginPassword, BatchGetImage, PutImage
- EKS: DescribeCluster, ListClusters (if using EKS)
- EC2: DescribeInstances (if using EC2 instances)

## Step 2: Create AWS ECR Repository

```bash
# Create ECR repository
aws ecr create-repository --repository-name portfolio-website --region us-east-1

# Verify repository was created
aws ecr describe-repositories --repository-names portfolio-website --region us-east-1
```

## Step 3: Login to ECR

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 132551339390.dkr.ecr.us-east-1.amazonaws.com
```

## Step 4: Build and Push Docker Image

```bash
# Navigate to the project directory
cd I-Am-PrashantGawali.github.io

# Build the Docker image
docker build -t 132551339390.dkr.ecr.us-east-1.amazonaws.com/portfolio-website:latest .

# Push the image to ECR
docker push 132551339390.dkr.ecr.us-east-1.amazonaws.com/portfolio-website:latest
```

## Step 5: Create EKS Cluster (if not exists)

```bash
# Create EKS cluster (if you don't have one)
eksctl create cluster \
  --name portfolio-cluster \
  --region us-east-1 \
  --nodes 2 \
  --node-type t3.medium

# Or use an existing cluster
kubectl config use-context <YOUR_CLUSTER_CONTEXT>
```

## Step 6: Configure ECR IAM Permissions (for EKS)

If using EKS, ensure your node IAM role has the necessary ECR permissions:

```bash
# Attach ECR policy to your node role
aws iam attach-role-policy \
  --role-name <YOUR_EKS_NODE_ROLE> \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

## Step 7: Install NGINX Ingress Controller and Cert-Manager

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer for Let's Encrypt (staging - for testing)
kubectl apply -f gitops/cert-manager/cluster-issuer-staging.yaml

# Create ClusterIssuer for Let's Encrypt (production)
kubectl apply -f gitops/cert-manager/cluster-issuer.yaml
```

**Note:** Replace `<YOUR_EMAIL>` in the ClusterIssuer files with your email address for Let's Encrypt notifications.

**Testing SSL (Optional):**
Before using production, you can test with the staging issuer:
1. Update `k8s/ingress.yaml` to use `letsencrypt-staging` instead of `letsencrypt-prod`
2. Apply the ingress and verify the certificate is issued
3. Once verified, switch back to `letsencrypt-prod`

## Step 8: Update DNS Records

Point your domain `portfolio.prashantgawali.site` to your Ingress Controller's IP:

```bash
# Get Ingress Controller IP
kubectl get svc ingress-nginx-controller -n ingress-nginx

# Add A record in your DNS provider:
# portfolio.prashantgawali.site -> <INGRESS_IP>
```

## Step 9: Create Namespace (if needed)

```bash
kubectl create namespace argocd
```

## Step 10: Apply ArgoCD Application Manifest

```bash
# Apply the ArgoCD Application manifest
kubectl apply -f gitops/argocd-application.yaml

# Verify the application is created
kubectl get application -n argocd
```

## Step 11: Sync the Application

You can sync via ArgoCD UI or CLI:

**Via CLI:**
```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login to ArgoCD
argocd login <ARGOCD_SERVER> --username <username> --password <password>

# Sync the application
argocd app sync portfolio-website
```

**Via UI:**
1. Open ArgoCD UI
2. Navigate to the portfolio-website application
3. Click "Sync"

## Step 12: Verify Deployment

```bash
# Check pods
kubectl get pods -l app=portfolio-website

# Check service
kubectl get svc portfolio-website

# Check ingress
kubectl get ingress portfolio-website-ingress

# Verify SSL certificate
kubectl describe certificate portfolio-website-tls
```

## Step 13: Access Your Website

Open your browser and navigate to: https://portfolio.prashantgawali.site

## Step 14: CI/CD Automation (Optional)

To automate the build and push process, you can use GitHub Actions. Create `.github/workflows/deploy.yml`:

```yaml
name: Build and Push to ECR

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build and push Docker image
      env:
        ECR_REGISTRY: 132551339390.dkr.ecr.us-east-1.amazonaws.com
        ECR_REPOSITORY: portfolio-website
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

## Troubleshooting

**Image Pull Errors:**
```bash
# Create secret for ECR (if not using IAM roles)
kubectl create secret docker-registry ecr-secret \
  --docker-server=132551339390.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  --docker-email=prashantgawali2807@gmail.com

# Add to deployment.yaml under spec.template.spec:
# imagePullSecrets:
# - name: ecr-secret
```

**ArgoCD Sync Issues:**
```bash
# Check application status
argocd app get portfolio-website

# Check application logs
argocd app logs portfolio-website
```

**SSL/TLS Certificate Issues:**
```bash
# Check certificate status
kubectl get certificate portfolio-website-tls

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager

# Describe certificate for more details
kubectl describe certificate portfolio-website-tls
```

**Ingress Issues:**
```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Check ingress events
kubectl describe ingress portfolio-website-ingress
```

## Clean Up

```bash
# Delete ArgoCD application
kubectl delete application portfolio-website -n argocd

# Delete Kubernetes resources
kubectl delete -f k8s/

# Delete ECR repository (optional)
aws ecr delete-repository --repository-name portfolio-website --region us-east-1 --force
```
