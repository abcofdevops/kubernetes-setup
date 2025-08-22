# Kubernetes Ingress Controller Installation

A comprehensive guide for installing and configuring ingress controllers in Kubernetes clusters to manage external access to services.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Supported Ingress Controllers](#supported-ingress-controllers)
- [Installation Methods](#installation-methods)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [SSL/TLS Setup](#ssltls-setup)
- [Monitoring and Logging](#monitoring-and-logging)
- [Troubleshooting](#troubleshooting)

## Overview

An ingress controller is a specialized load balancer for Kubernetes environments that manages external access to services within a cluster. This repository provides installation scripts, configurations, and documentation for deploying various ingress controllers.

### Key Features

- Multiple ingress controller options (NGINX, Traefik, HAProxy, etc.)
- Automated SSL/TLS certificate management
- Load balancing and traffic routing
- Health checks and monitoring integration
- Cloud provider integration support

## Prerequisites

Before installing an ingress controller, ensure you have:

- **Kubernetes cluster** (v1.19 or higher recommended)
- **kubectl** configured to access your cluster
- **Helm** (v3.0+) for package management
- **Administrative privileges** on the cluster
- **DNS configuration** for domain routing (optional but recommended)

### Cluster Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 100m | 500m |
| Memory | 128Mi | 512Mi |
| Nodes | 1 | 3+ |

## Supported Ingress Controllers

This repository supports installation of the following ingress controllers:

### 1. NGINX Ingress Controller
- **Use case**: General purpose, most popular
- **Features**: High performance, extensive configuration options
- **Cloud support**: AWS, GCP, Azure

### 2. Traefik
- **Use case**: Cloud-native, microservices
- **Features**: Automatic service discovery, built-in dashboard
- **Cloud support**: Multi-cloud

### 3. HAProxy Ingress
- **Use case**: Enterprise-grade load balancing
- **Features**: Advanced routing, high availability
- **Cloud support**: On-premises, hybrid cloud

### 4. Ambassador/Emissary
- **Use case**: API gateway functionality
- **Features**: gRPC support, rate limiting, authentication
- **Cloud support**: Kubernetes-native

## Installation Methods

Choose your preferred installation method:

### Method 1: Helm (Recommended)

```bash
# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

### Method 2: kubectl + YAML Manifests

```bash
# Apply NGINX Ingress Controller manifests
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

### Method 3: Custom Installation Scripts

```bash
# Clone this repository
git clone https://github.com/your-org/k8s-ingress-controller-installation.git
cd k8s-ingress-controller-installation

# Run installation script
./install.sh --controller=nginx --environment=production
```

## Quick Start

### Step 1: Choose Your Ingress Controller

```bash
# Set environment variables
export INGRESS_CONTROLLER=nginx
export NAMESPACE=ingress-system
```

### Step 2: Install the Controller

```bash
# Using our installation script
./scripts/install-controller.sh \
  --controller=$INGRESS_CONTROLLER \
  --namespace=$NAMESPACE \
  --enable-ssl=true
```

### Step 3: Verify Installation

```bash
# Check pods are running
kubectl get pods -n $NAMESPACE

# Get external IP
kubectl get service -n $NAMESPACE
```

### Step 4: Create Your First Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

Apply the ingress:

```bash
kubectl apply -f example-ingress.yaml
```

## Configuration

### Environment-Specific Configurations

#### Development
```bash
./install.sh --controller=nginx --environment=dev --replicas=1 --enable-metrics=false
```

#### Production
```bash
./install.sh --controller=nginx --environment=prod --replicas=3 --enable-metrics=true --enable-ssl=true
```

### Custom Values

Create a `values.yaml` file for Helm installations:

```yaml
controller:
  replicaCount: 3
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
  
defaultBackend:
  enabled: true
  
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

Install with custom values:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -f values.yaml \
  --namespace ingress-nginx \
  --create-namespace
```

## SSL/TLS Setup

### Option 1: cert-manager (Recommended)

Install cert-manager for automatic certificate management:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

# Create ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

### Option 2: Manual Certificate

```bash
# Create TLS secret
kubectl create secret tls example-tls \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key \
  --namespace=default
```

Add TLS configuration to your ingress:

```yaml
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
```

## Monitoring and Logging

### Enable Metrics

```bash
# Install with metrics enabled
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=true
```

### Prometheus Integration

```yaml
# ServiceMonitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ingress-nginx-metrics
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  endpoints:
  - port: metrics
```

### Log Configuration

```yaml
controller:
  config:
    log-format-json: "true"
    access-log-path: "/var/log/nginx/access.log"
    error-log-path: "/var/log/nginx/error.log"
```

## Troubleshooting

### Common Issues

#### 1. Ingress Controller Not Starting

```bash
# Check pod logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Check events
kubectl get events -n ingress-nginx --sort-by='.metadata.creationTimestamp'
```

#### 2. External IP Pending

```bash
# Check service status
kubectl describe service ingress-nginx-controller -n ingress-nginx

# For cloud providers, ensure LoadBalancer service type is supported
```

#### 3. SSL Certificate Issues

```bash
# Check certificate status
kubectl describe certificate -n default

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager
```

#### 4. 404 Errors

```bash
# Verify ingress configuration
kubectl describe ingress example-ingress

# Check backend service
kubectl get endpoints web-service
```

### Debug Commands

```bash
# Get ingress controller version
kubectl exec -it deployment/ingress-nginx-controller -n ingress-nginx -- /nginx-ingress-controller --version

# Test connectivity
kubectl exec -it deployment/ingress-nginx-controller -n ingress-nginx -- curl -I localhost/healthz

# Validate configuration
kubectl exec -it deployment/ingress-nginx-controller -n ingress-nginx -- nginx -T
```

### Development Setup

```bash
# Clone the repo
git clone https://github.com/abcofdevops/kubernetes-setup/k8s-ingress-controller-installation.git
cd k8s-ingress-controller-installation

# Install development dependencies
make dev-setup

# Run tests
make test

# Lint code
make lint
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

**Note**: This README assumes you have basic knowledge of Kubernetes concepts. For beginners, we recommend reviewing the [Kubernetes documentation](https://kubernetes.io/docs/) first.
