# Reddit Clone Deployment on AWS EKS with ALB, Prometheus & Grafana

## Project Overview

This project demonstrates a complete DevOps workflow where a containerized Reddit Clone application is deployed on an Amazon EKS Cluster using Kubernetes manifests. The cluster is integrated with AWS Application Load Balancer (ALB) Ingress Controller for external access and monitored using Prometheus and Grafana.

Additionally, a CI/CD pipeline is implemented using GitHub Actions to automate Docker image build, push, and Kubernetes deployment.

---

# Technologies Used

- AWS EC2
- AWS EKS
- Kubernetes
- Docker
- DockerHub
- Helm
- AWS Load Balancer Controller (ALB)
- Prometheus
- Grafana
- GitHub Actions
- kubectl
- eksctl
- AWS CLI

---

# Architecture

```text
GitHub Repository
        ↓
GitHub Actions CI/CD
        ↓
Docker Build & Push to DockerHub
        ↓
Amazon EKS Cluster
        ↓
Kubernetes Deployment + Service + Ingress
        ↓
AWS Application Load Balancer (ALB)
        ↓
Reddit Clone Application
        ↓
Prometheus + Grafana Monitoring
```

---

# Project Structure

```text
.
├── .github/
│   └── workflows/
│       └── cicd.yml
│
├── kubernetes/
│   ├── deployment.yml
│   ├── service.yml
│   └── ingress.yml
│
├── screenshots/
│
├── Dockerfile
└── README.md
```

---

# Step 1 - Launch EC2 Management Instance

- Launch Ubuntu EC2 instance
- Attach IAM Role with AdministratorAccess (For learning purpose only)
- Connect to instance

---

# Step 2 - Install Required Tools

## Update Packages

```bash
sudo apt-get update -y 
```

---

## Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

sudo apt install unzip -y

unzip awscliv2.zip

sudo ./aws/install
```

Verify:

```bash
aws --version
```

---

## Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

## Install eksctl

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```

Verify:

```bash
eksctl version
```

---

## Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

# Step 3 - Create EKS Cluster

```bash
eksctl create cluster \
--name my-eks-cluster \
--region ap-south-1 \
--nodegroup-name my-nodes \
--node-type c7i-flex.large \
--nodes 2 \
--managed
```

---

# Step 4 - Configure kubectl for EKS

```bash
aws eks update-kubeconfig \
--region ap-south-1 \
--name my-eks-cluster
```

Verify:

```bash
kubectl get nodes
```

---

# Step 5 - Configure OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
--region ap-south-1 \
--cluster my-eks-cluster \
--approve
```

---

# Step 6 - Install AWS Load Balancer Controller

## Download IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

---

## Create IAM Policy

```bash
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json
```

---

## Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=my-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

## Add Helm Repo

```bash
helm repo add eks https://aws.github.io/eks-charts
```

```bash
helm repo update
```

---

## Install ALB Controller

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<VPC_ID>
```

Verify:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

# Step 7 - Clone Reddit Clone Repository

```bash
git clone https://github.com/dhruvgoe/Reddit-Clone-EKS-ALB-Monitoring.git
```

```bash
cd Reddit-Clone-EKS-ALB-Monitoring
```

---

# Step 8 - Deploy Application

## Create Namespace

```bash
kubectl create namespace reddit
```

---

## Deploy Resources

```bash
kubectl apply -f deployment.yml -n reddit
```

```bash
kubectl apply -f service.yml -n reddit
```

```bash
kubectl apply -f ingress.yml -n reddit
```

---

## Verify Deployment

```bash
kubectl get pods -n reddit
```

```bash
kubectl get svc -n reddit
```

```bash
kubectl get ingress -n reddit
```

---

# Step 9 - Access Application

Get ALB DNS:

```bash
kubectl get ingress -n reddit
```

Open:

```text
http://<ALB-DNS>
```

---

# Step 10 - Install Prometheus & Grafana

## Add Helm Repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```bash
helm repo update
```

---

## Install kube-prometheus-stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

---

## Verify Monitoring Pods

```bash
kubectl get pods -n monitoring
```

---

# Step 11 - Access Grafana

## Get Grafana Password

```bash
kubectl get secret monitoring-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" | base64 --decode
```

---

## Port Forward Grafana

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open:

```text
http://localhost:3000
```

Credentials:

```text
Username: admin
Password: <output-from-command>
```

---

# Step 12 - Configure CI/CD with GitHub Actions

## GitHub Secrets Used

| Secret Name | Description |
|---|---|
| AWS_ACCESS_KEY_ID | AWS Access Key |
| AWS_SECRET_ACCESS_KEY | AWS Secret Key |
| AWS_REGION | AWS Region |
| EKS_CLUSTER_NAME | EKS Cluster Name |
| DOCKER_USERNAME | DockerHub Username |
| DOCKER_PASSWORD | DockerHub Access Token |

---

# GitHub Actions Workflow

Create:

```text
.github/workflows/deploy.yml
```

---

## Workflow File

```yaml
name: Deploy Reddit Clone to EKS

on: workflow_dispatch

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:

    - name: Checkout Code
      uses: actions/checkout@v4

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Login to DockerHub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build Docker Image
      run: |
        docker build -t <DOCKER_USERNAME>/reddit-clone:latest .

    - name: Push Docker Image
      run: |
        docker push <DOCKER_USERNAME>/reddit-clone:latest

    - name: Install kubectl
      uses: azure/setup-kubectl@v4

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig \
        --region ${{ secrets.AWS_REGION }} \
        --name ${{ secrets.EKS_CLUSTER_NAME }}

    - name: Deploy to EKS
      run: |
        kubectl apply --validate=false -f kubernetes/
        kubectl rollout restart deployment reddit-deployment
```

---

# Skills Demonstrated

- Kubernetes Administration
- Amazon EKS
- AWS ALB Ingress Controller
- Docker Containerization
- CI/CD Automation
- GitHub Actions
- Infrastructure Monitoring
- Prometheus & Grafana
- Helm Package Management
- Kubernetes Networking
- IAM & OIDC Integration
- Cloud Infrastructure Management

---

# Future Improvements

- Terraform for Infrastructure Provisioning
- HTTPS using ACM
- Route53 Custom Domain
- Horizontal Pod Autoscaler (HPA)
- ArgoCD GitOps
- Amazon ECR Integration
- Blue-Green Deployment
- Jenkins Pipeline

---

# Cleanup Commands

## Delete Application

```bash
kubectl delete -f kubernetes/
```

---

## Delete Monitoring Stack

```bash
helm uninstall monitoring -n monitoring
```

```bash
kubectl delete namespace monitoring
```

---

## Delete EKS Cluster

```bash
eksctl delete cluster \
--name my-eks-cluster \
--region ap-south-1
```

---

# Conclusion

This project demonstrates a complete production-style Kubernetes deployment pipeline on AWS using EKS, ALB Ingress Controller, Prometheus, Grafana, and GitHub Actions CI/CD.

The project covers:
- Infrastructure provisioning
- Kubernetes deployment
- Application exposure using ALB
- Monitoring implementation
- CI/CD automation
- Cloud-native DevOps practices

### Dhruv Goel
Cloud | AWS | DevOps
