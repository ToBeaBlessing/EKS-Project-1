# EKS Fargate with AWS Load Balancer Controller

![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Helm](https://img.shields.io/badge/helm-%230F1689.svg?style=for-the-badge&logo=helm&logoColor=white)

A production-ready demonstration of deploying a containerized application on Amazon EKS using Fargate compute and AWS Application Load Balancer Controller for intelligent traffic routing.
<img width="725" height="862" alt="image" src="https://github.com/user-attachments/assets/d6166236-eaa1-43e0-a44e-5964fe1f1251" />

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Deployment Guide](#deployment-guide)
- [Verification](#verification)
- [Cleanup](#cleanup)
- [Skills Demonstrated](#skills-demonstrated)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## 🎯 Overview

This project demonstrates the deployment of the 2048 game on a serverless Kubernetes cluster using AWS EKS Fargate. It showcases critical cloud-native engineering skills including:

- Serverless Kubernetes cluster provisioning
- IAM security with OIDC provider integration
- Helm package management
- Application Load Balancer automation via Ingress controllers

### Key Components

| Component | Purpose |
|-----------|---------|
| **EKS Cluster** | Managed Kubernetes control plane |
| **Fargate** | Serverless compute for pods |
| **ALB Controller** | Manages AWS Application Load Balancers |
| **IRSA** | Secure AWS API access for pods |
| **Helm** | Package manager for Kubernetes |

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│           Application Load Balancer         │
│              (Auto-provisioned)             │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│           Kubernetes Ingress                │
│            (game-2048 namespace)            │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│          2048 Game Application              │
│         (Running on Fargate Pods)           │
└─────────────────────────────────────────────┘
```

**Region**: `us-east-1`

## ✅ Prerequisites

Ensure you have the following tools installed and configured:

- [AWS CLI](https://aws.amazon.com/cli/) (configured with appropriate credentials)
- [eksctl](https://eksctl.io/) (v0.150.0+)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) (v1.28+)
- [Helm](https://helm.sh/docs/intro/install/) (v3.x)
- AWS Account with permissions for EKS, EC2, IAM, and VPC

### Verify Prerequisites

```bash
aws --version
eksctl version
kubectl version --client
helm version
```

## 🚀 Deployment Guide

### Step 1: Create EKS Cluster with Fargate

```bash
eksctl create cluster --name demo-cluster1 --region us-east-1 --fargate
```

This creates a fully managed EKS cluster with Fargate as the compute engine.

### Step 2: Configure kubectl

```bash
aws eks update-kubeconfig --name demo-cluster1 --region us-east-1
```

### Step 3: Create Fargate Profile

```bash
eksctl create fargateprofile \
    --cluster demo-cluster1 \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

This profile ensures pods in the `game-2048` namespace run on Fargate.

### Step 4: Deploy the 2048 Application

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

### Step 5: Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster1 --approve
```

This enables IAM roles for service accounts (IRSA).

### Step 6: Create IAM Policy for ALB Controller

```bash
# Download the IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

# Create the policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### Step 7: Create IAM Service Account

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

> ⚠️ **Important**: Replace `<your-aws-account-id>` with your actual AWS account ID.

**Get your Account ID:**
```bash
aws sts get-caller-identity --query Account --output text
```

### Step 8: Install AWS Load Balancer Controller

```bash
# Add EKS Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```

> ⚠️ **Important**: Replace `<your-vpc-id>` with your cluster's VPC ID.

**Get your VPC ID:**
```bash
aws eks describe-cluster --name demo-cluster1 --query "cluster.resourcesVpcConfig.vpcId" --output text
```

## ✔️ Verification

### Check Application Pods

```bash
kubectl get pods -n game-2048
```

Expected output:
```
NAME                              READY   STATUS    RESTARTS   AGE
deployment-2048-xxxxxxxxx-xxxxx   1/1     Running   0          5m
```

### Check ALB Controller Deployment

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Check ALB Controller Pods

```bash
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

### Get Ingress and ALB URL

```bash
kubectl get ingress -n game-2048
```

The `ADDRESS` column will show your Application Load Balancer DNS name. Access the application by opening this URL in your browser.

## 🎮 Accessing the Application

Once the Ingress is provisioned (may take 2-3 minutes), access the 2048 game:

```bash
# Get the ALB URL
kubectl get ingress -n game-2048 -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'
```

Open the URL in your web browser to play the game!

## 🧹 Cleanup

To avoid incurring charges, delete all resources when you're done:

```bash
# Delete the cluster (this removes all associated resources)
eksctl delete cluster --name demo-cluster1 --region us-east-1

# Optionally, delete the IAM policy
aws iam delete-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy
```

## 💼 Skills Demonstrated

This project showcases proficiency in:

- ☁️ **AWS Services**: EKS, Fargate, ALB, IAM, VPC
- 🐳 **Container Orchestration**: Kubernetes workload management
- 🔐 **Cloud Security**: IAM roles, OIDC integration, least-privilege access
- 📦 **Package Management**: Helm chart deployment
- 🛠️ **DevOps Tools**: eksctl, kubectl, helm, AWS CLI
- 🏗️ **Infrastructure as Code**: Declarative resource provisioning
- 🌐 **Networking**: Ingress controllers and traffic routing

## 🔧 Troubleshooting

### Pods Not Starting

**Issue**: Pods stuck in `Pending` state

**Solution**: Verify the Fargate profile namespace matches the application namespace:
```bash
eksctl get fargateprofile --cluster demo-cluster1
```

### Ingress Not Creating ALB

**Issue**: Ingress shows no `ADDRESS`

**Solution**: Check ALB controller logs:
```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

Verify IAM permissions are correctly configured:
```bash
kubectl describe sa aws-load-balancer-controller -n kube-system
```

### Cannot Access Application

**Issue**: ALB URL returns timeout or connection refused

**Solution**: 
1. Verify security groups allow inbound traffic on ports 80/443
2. Check target group health in AWS Console
3. Ensure pods are running: `kubectl get pods -n game-2048`

### ALB Controller Installation Fails

**Issue**: Helm install returns authentication errors

**Solution**: Verify OIDC provider is associated:
```bash
aws eks describe-cluster --name demo-cluster1 --query "cluster.identity.oidc.issuer" --output text
```

## 📚 Additional Resources

- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Amazon EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [eksctl Documentation](https://eksctl.io/)
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## 🚀 Future Enhancements

- [ ] Implement monitoring with Prometheus and Grafana
- [ ] Add GitOps workflow with ArgoCD or Flux
- [ ] Configure Horizontal Pod Autoscaling
- [ ] Add SSL/TLS termination with AWS Certificate Manager
- [ ] Implement CI/CD pipeline with GitHub Actions
- [ ] Add Kubernetes Network Policies for security
- [ ] Configure logging with CloudWatch Container Insights

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

---

**Author**: Blessing  
**Institution**: Morgan State University - Industrial Engineering (MS)  
**GitHub**: [@yourusername](https://github.com/yourusername)  
**LinkedIn**: [Your LinkedIn Profile](https://linkedin.com/in/yourprofile)

⭐ If you found this project helpful, please consider giving it a star!
