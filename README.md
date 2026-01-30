EKS Fargate with AWS Load Balancer Controller
A production-ready demonstration of deploying a containerized application on Amazon EKS using Fargate compute and AWS Application Load Balancer Controller for intelligent traffic routing.
Project Overview
This project showcases the deployment of the popular 2048 game on a serverless Kubernetes cluster using AWS EKS Fargate. It demonstrates critical cloud-native skills including cluster provisioning, IAM role configuration, Helm package management, and ingress controller setup.
Architecture

Compute: AWS EKS with Fargate (serverless containers)
Networking: Application Load Balancer with Kubernetes Ingress
Identity: OIDC provider integration with IAM for service accounts
Package Management: Helm for controller deployment
Region: us-east-1

Key Components

EKS Cluster: Serverless Kubernetes cluster eliminating node management overhead
Fargate Profile: Dedicated compute profile for the game-2048 namespace
AWS Load Balancer Controller: Manages ALB/NLB creation from Kubernetes Ingress resources
IAM Roles for Service Accounts (IRSA): Secure AWS API access without static credentials

Prerequisites

AWS CLI configured with appropriate credentials
eksctl CLI tool
kubectl CLI tool
helm v3.x
Active AWS account with EKS and EC2 permissions

Deployment Steps
1. Create EKS Cluster with Fargate
basheksctl create cluster --name demo-cluster1 --region us-east-1 --fargate
2. Configure kubectl Context
bashaws eks update-kubeconfig --name demo-cluster1 --region us-east-1
3. Create Fargate Profile for Application
basheksctl create fargateprofile \
    --cluster demo-cluster1 \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
4. Deploy Sample Application
bashkubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
5. Verify Application Deployment
bashkubectl get pods -n game-2048
kubectl get ingress -n game-2048
6. Configure IAM OIDC Provider
basheksctl utils associate-iam-oidc-provider --cluster demo-cluster1 --approve
7. Create IAM Policy for Load Balancer Controller
bash# Download the IAM policy document
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

# Create the IAM policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
8. Create IAM Service Account
basheksctl create iamserviceaccount \
  --cluster=demo-cluster1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
Note: Replace <your-aws-account-id> with your actual AWS account ID.
9. Install AWS Load Balancer Controller via Helm
bash# Add the EKS Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
Note: Replace <your-vpc-id> with your cluster's VPC ID (found in EKS console or via AWS CLI).
10. Verify Controller Deployment
bashkubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system
kubectl get ingress -n game-2048
Accessing the Application
After successful deployment, retrieve the ALB DNS name:
bashkubectl get ingress -n game-2048
The output will display the Application Load Balancer URL. Access the 2048 game by navigating to this URL in your browser.
Technical Skills Demonstrated

Infrastructure as Code: Declarative cluster and resource provisioning
Container Orchestration: Kubernetes workload management
Cloud Security: IAM roles, OIDC integration, least-privilege access
Service Mesh Basics: Ingress controllers and traffic management
Package Management: Helm chart deployment and configuration
AWS Services: EKS, Fargate, ALB, IAM, VPC
DevOps Tools: eksctl, kubectl, helm, AWS CLI

Cost Optimization Notes
This deployment uses Fargate, which bills per-second for vCPU and memory resources. Remember to delete the cluster when testing is complete:
basheksctl delete cluster --name demo-cluster1 --region us-east-1
Troubleshooting
Pods not starting: Verify Fargate profile namespace matches application namespace (game-2048)
Ingress not creating ALB: Check controller logs and ensure IAM permissions are correctly configured
Cannot access application: Verify security groups allow inbound traffic on ports 80/443
Future Enhancements

Add monitoring with Prometheus/Grafana
Implement GitOps with ArgoCD or Flux
Configure autoscaling policies
Add SSL/TLS termination with ACM certificates
Implement CI/CD pipeline for automated deployments

References

AWS Load Balancer Controller Documentation
EKS Fargate Documentation
eksctl Documentation


Author: Blessing
LinkedIn: [Your LinkedIn Profile]
Contact: [Your Email]
