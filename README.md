# lumiatech Java Application - Kubernetes Deployment

## Overview

This project deploys a Java web application (lumiatech) to a Kubernetes cluster using a GitOps approach with two separate repositories:

- **Application Repository**: Contains source code, Dockerfile, and CI/CD pipeline
- **Manifest Repository**: Contains Kubernetes manifests and Helm charts for deployment

## Architecture

![Kubernetes Architecture Diagram](./images/project-4-deploy-to-eks.png)

- **Application**: Java Spring MVC application (WAR file)
- **Database**: MySQL
- **Orchestration**: Kubernetes
- **Deployment Tool**: Helm
- **Ingress**: NGINX Ingress Controller

## Prerequisites

- Kubernetes cluster (v1.19+)
- kubectl configured
- Helm 3.x installed
- Docker registry access
- NGINX Ingress Controller installed in cluster

## Repository Structure

### Application Repository

```
lumiatech-app/
├── src/                    # Java source code
├── pom.xml                 # Maven build configuration
├── Dockerfile              # Container image definition
├── Jenkinsfile             # CI/CD pipeline
└── README.md
```

### Manifest Repository

```
lumiatech-manifests/
├── helm/
│   └── lumia/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── appdeploy.yaml
│           ├── appservice.yaml
│           ├── appingress.yaml
│           ├── dbdeploy.yaml
│           ├── dbservice.yaml
│           ├── dbpvc.yaml
│           └── secret.yaml
└── README.md
```

## Deployment Components

### Application Deployment

- **Image**: `ndzenyuy/lumiatechapp`
- **Port**: 8080
- **Init Containers**: Wait for database and cache services
- **Service Type**: ClusterIP

### Database Deployment

- **Image**: `ndzenyuy/lumiatechdb`
- **Port**: 3306
- **Storage**: PersistentVolumeClaim
- **Credentials**: Stored in Kubernetes Secret

### Ingress

- **Host**: `www.lumiatech.com`
- **Path**: `/`
- **Backend**: lumia-app-service:8080

## AWS EKS Cluster Setup

### Prerequisites

- AWS CLI configured with appropriate credentials
- IAM permissions to create EKS clusters and related resources

### Install Required Tools

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installations
kubectl version --client
eksctl version
helm version
```

### Create EKS Cluster

```bash
# Create EKS cluster
eksctl create cluster \
  --name lumiatech-cluster \
  --region us-east-1 \
  --nodegroup-name lumiatech-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

# Configure kubectl context
aws eks update-kubeconfig --region us-east-1 --name lumiatech-cluster
```

### Install NGINX Ingress Controller

```bash
# Deploy NGINX Ingress Controller for AWS
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/aws/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Verify Cluster

```bash
# Check cluster status
kubectl cluster-info

# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Get ingress load balancer URL
kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

## Deployment Steps

### 1. Clone Repositories

```bash
# Application repository
git clone <app-repo-url>

# Manifest repository
git clone <manifest-repo-url>
```

### 2. Create Namespace

```bash
kubectl create namespace lumiatech
kubectl config set-context --current --namespace=lumiatech
```

### 3. Deploy Using Helm

```bash
cd lumiatech-manifests/helm

# Install
helm install lumiatech ./lumiatech -n lumiatech

# Upgrade
helm upgrade lumiatech ./lumiatech -n lumiatech

# Rollback
helm rollback lumiatech -n lumiatech
```

### 4. Verify Deployment

```bash
kubectl get pods -n lumiatech
kubectl get svc -n lumiatech
kubectl get ingress -n lumiatech
```

### 5. Access Application

```bash
# Update /etc/hosts or DNS
echo "<INGRESS_IP> lumiatech.hkhinfoteck.xyz" >> /etc/hosts

# Access via browser
https://www.lumiatechs.com
```

## CI/CD Pipeline

### Build & Push

1. Maven builds WAR file
2. Docker builds container image
3. Image pushed to registry
4. Trigger manifest repository update

### Deploy

1. Update image tag in values.yaml
2. Commit to manifest repository
3. Helm upgrade deployment
4. Kubernetes rolls out new version

## Configuration

### Secrets

Update `secret.yaml` with base64 encoded values:

```bash
echo -n 'your-password' | base64
```

### Environment Variables

- `MYSQL_ROOT_PASSWORD`: Database root password (from secret)

### Persistent Storage

- Database uses PVC for data persistence
- Default storage class used

## Helm Values

Key values in `values.yaml`:

```yaml
app:
  image: ndzenyuy/lumiatechapp
  tag: latest
  replicas: 1

db:
  image: ndzenyuy/lumiatechdb
  tag: latest
  storage: 5Gi

ingress:
  host: www.lumiatech.com
```

## Monitoring & Troubleshooting

### Check Logs

```bash
kubectl logs -f deployment/lumia-app -n lumiatech
kubectl logs -f deployment/vprodb -n lumiatech
```

### Debug Pods

```bash
kubectl describe pod <pod-name> -n lumiatech
kubectl exec -it <pod-name> -n lumiatech -- /bin/bash
```

### Common Issues

- **Init containers stuck**: Check service DNS resolution
- **ImagePullBackOff**: Verify registry credentials
- **CrashLoopBackOff**: Check application logs and database connectivity

## Cleanup

```bash
helm uninstall lumiatech -n lumiatech
kubectl delete namespace lumiatech
```

## Security Considerations

- Secrets stored in Kubernetes Secret (consider using Sealed Secrets or External Secrets Operator)
- Use private container registry
- Implement RBAC policies
- Enable network policies
- Regular security scanning of images

## Future Enhancements

- Add Horizontal Pod Autoscaler (HPA)
- Implement monitoring with Prometheus/Grafana
- Add centralized logging with ELK/EFK stack
- Implement GitOps with ArgoCD/FluxCD
- Add health checks and readiness probes
- Implement blue-green or canary deployments
