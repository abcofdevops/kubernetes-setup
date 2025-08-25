# Kubernetes Cluster Ingress Controller Installation Guide

This guide provides comprehensive instructions for setting up ingress controllers on different types of Kubernetes clusters: Amazon EKS, kind (Kubernetes in Docker), and kubeadm-based clusters.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [EKS Cluster Setup](#eks-cluster-setup)
3. [kind Cluster Setup](#kind-cluster-setup)
4. [kubeadm Cluster Setup](#kubeadm-cluster-setup)
5. [Testing Your Setup](#testing-your-setup)
6. [Troubleshooting](#troubleshooting)
7. [Advanced Configuration](#advanced-configuration)

## Prerequisites

Before starting, ensure you have the following tools installed:

- `kubectl` (v1.24+)
- `helm` (v3.8+)
- Access to your Kubernetes cluster with admin privileges

### Tool Installation

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installations
kubectl version --client
helm version
```

## EKS Cluster Setup

Amazon EKS requires specific considerations for ingress controllers due to its managed nature and AWS Load Balancer integration.

### Option 1: AWS Load Balancer Controller (Recommended for EKS)

The AWS Load Balancer Controller is the recommended ingress solution for EKS as it integrates natively with AWS services.

#### Step 1: Install AWS Load Balancer Controller

```bash
# Add the EKS repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Create IAM service account (replace with your cluster name and region)
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess \
  --approve

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### Step 2: Verify Installation

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

### Option 2: NGINX Ingress Controller on EKS

If you prefer NGINX over the AWS Load Balancer Controller:

```bash
# Add NGINX ingress repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb"
```

### EKS-Specific Configuration

Create an ingress resource for EKS with AWS Load Balancer Controller:

```yaml
# eks-ingress-example.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  ingressClassName: alb
  rules:
  - host: example.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: your-service
            port:
              number: 80
```

## kind Cluster Setup

kind (Kubernetes in Docker) is excellent for local development and testing.

### Step 1: Create kind Cluster with Port Mapping

Create a kind configuration that exposes ports for the ingress controller:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

```bash
# Create the cluster
kind create cluster --config=kind-config.yaml --name=ingress-cluster

# Verify cluster is ready
kubectl cluster-info --context kind-ingress-cluster
```

### Step 2: Install NGINX Ingress Controller

```bash
# Install NGINX ingress controller for kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Step 3: Test with Sample Application

```yaml
# kind-test-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-app
  labels:
    app: test-app
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: test-service
spec:
  selector:
    app: test-app
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-service
            port:
              number: 80
```

```bash
# Apply the test application
kubectl apply -f kind-test-app.yaml

# Test the ingress (should work on localhost)
curl localhost
```

## kubeadm Cluster Setup

For clusters created with kubeadm, you'll typically use NGINX ingress controller with NodePort or MetalLB for LoadBalancer functionality.

### Option 1: NGINX with NodePort (Simplest)

```bash
# Install NGINX ingress controller with NodePort
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

### Option 2: NGINX with MetalLB (Recommended for Production)

First, install MetalLB for LoadBalancer services:

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

# Wait for MetalLB to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

Configure MetalLB with an IP address pool:

```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250  # Adjust to your network
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
```

```bash
# Apply MetalLB configuration
kubectl apply -f metallb-config.yaml

# Install NGINX ingress controller with LoadBalancer
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

### Verify kubeadm Installation

```bash
# Check ingress controller status
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx

# Get the external IP or NodePort
kubectl get service ingress-nginx-controller -n ingress-nginx
```

## Testing Your Setup

### Create a Test Application

```yaml
# test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: hello-world-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-world-html
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Hello World</title></head>
    <body><h1>Hello from Kubernetes Ingress!</h1></body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx  # Use 'alb' for EKS with AWS Load Balancer Controller
  rules:
  - host: hello-world.local  # Add this to /etc/hosts for testing
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world-service
            port:
              number: 80
```

```bash
# Deploy the test application
kubectl apply -f test-deployment.yaml

# Check if everything is running
kubectl get pods,services,ingress

# For testing with hello-world.local, add to /etc/hosts:
# 127.0.0.1 hello-world.local  # for kind
# <EXTERNAL_IP> hello-world.local  # for EKS/kubeadm
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Ingress Controller Not Starting

```bash
# Check pod status and logs
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Check for resource constraints
kubectl describe pods -n ingress-nginx
```

#### 2. External IP Pending (EKS/kubeadm)

For EKS:
```bash
# Check AWS Load Balancer Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verify IAM permissions
aws sts get-caller-identity
```

For kubeadm with MetalLB:
```bash
# Check MetalLB status
kubectl get pods -n metallb-system
kubectl logs -n metallb-system -l app=metallb
```

#### 3. Ingress Not Routing Traffic

```bash
# Check ingress status
kubectl describe ingress hello-world-ingress

# Verify service endpoints
kubectl get endpoints hello-world-service

# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

#### 4. SSL/TLS Issues

```bash
# Check certificate status (if using cert-manager)
kubectl get certificates
kubectl describe certificate <cert-name>

# Force certificate renewal
kubectl delete secret <tls-secret-name>
```

### Debug Commands

```bash
# General cluster health
kubectl get nodes
kubectl get pods --all-namespaces

# Ingress-specific debugging
kubectl get ingressclass
kubectl get ingress --all-namespaces
kubectl describe ingress <ingress-name>

# Network connectivity testing
kubectl run test-pod --image=busybox --rm -it -- /bin/sh
# Then inside the pod: wget -O- http://hello-world-service
```

## Advanced Configuration

### SSL/TLS with cert-manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# Create ClusterIssuer for Let's Encrypt
```

```yaml
# letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

### Rate Limiting and Security

```yaml
# Add these annotations to your ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
```

### Multi-path Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-path-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

## Cleanup

### Remove Ingress Controllers

```bash
# For NGINX ingress
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx

# For AWS Load Balancer Controller
helm uninstall aws-load-balancer-controller -n kube-system
eksctl delete iamserviceaccount --cluster=my-cluster --name=aws-load-balancer-controller --namespace=kube-system

# For kind cluster
kind delete cluster --name=ingress-cluster
```

---

## Additional Resources

- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.aws-load-balancer-controller.io/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [MetalLB Documentation](https://metallb.universe.tf/)

For issues and contributions, please refer to the respective project repositories.
