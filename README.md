# Deploy Nginx on GKE using Argo CD

This project demonstrates how to deploy an Nginx application on Google Kubernetes Engine (GKE) using Argo CD and GitOps principles.

## Architecture

```text
GitHub Repository
        │
        ▼
     Argo CD
        │
        ▼
   GKE Cluster
        │
        ▼
   Nginx Application
```

## Prerequisites

Ensure the following tools and services are available:

* Google Cloud SDK (gcloud)
* kubectl
* Git
* GitHub Account
* Google Cloud Project with Billing Enabled

---

## Step 1: Login to Google Cloud

Authenticate with Google Cloud:

```bash
gcloud auth login
```

Set your project:

```bash
gcloud config set project PROJECT_ID
```

Verify configuration:

```bash
gcloud config list
```

---

## Step 2: Enable Required APIs

Enable the Kubernetes Engine API:

```bash
gcloud services enable container.googleapis.com
```

Verify enabled services:

```bash
gcloud services list --enabled
```

---

## Step 3: Create GKE Cluster

Create a Kubernetes cluster:

```bash
gcloud container clusters create argocd-cluster \
--zone asia-south1-a \
--num-nodes 2
```

This creates:

* Managed Kubernetes Control Plane
* Two Worker Nodes
* GKE Cluster

Verify cluster creation:

```bash
gcloud container clusters list
```

---

## Step 4: Connect kubectl to GKE

Configure kubectl:

```bash
gcloud container clusters get-credentials argocd-cluster \
--zone asia-south1-a
```

Verify connectivity:

```bash
kubectl cluster-info
```

Check cluster nodes:

```bash
kubectl get nodes
```

Expected output:

```text
NAME                 STATUS
gke-node-1           Ready
gke-node-2           Ready
```

---

## Step 5: Verify Kubernetes Cluster

List namespaces:

```bash
kubectl get ns
```

Check system pods:

```bash
kubectl get pods -A
```

At this stage, the Kubernetes cluster is ready.

---

## Step 6: Create Argo CD Namespace

Create a namespace for Argo CD:

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get ns
```

---

## Step 7: Install Argo CD

Install Argo CD using the official manifests:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs:

* Argo CD Server
* Application Controller
* Repo Server
* Redis
* Dex

---

## Step 8: Verify Argo CD Installation

Check pod status:

```bash
kubectl get pods -n argocd
```

Wait until all pods are in the Running state.

Example:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-redis
argocd-dex-server
```

---

## Step 9: Expose Argo CD

Expose Argo CD using a LoadBalancer service:

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec":{"type":"LoadBalancer"}}'
```

Verify:

```bash
kubectl get svc -n argocd
```

Example:

```text
NAME            TYPE           EXTERNAL-IP
argocd-server   LoadBalancer   34.xx.xx.xx
```

Access Argo CD:

```text
https://<EXTERNAL-IP>
```

---

## Step 10: Get Argo CD Admin Password

Retrieve the initial password:

```bash
kubectl get secret argocd-initial-admin-secret \
-n argocd \
-o jsonpath="{.data.password}" | base64 -d
```

Login credentials:

```text
Username: admin
Password: <generated-password>
```

---

## Step 11: Create GitHub Repository

Repository structure:

```text
argocd-nginx-demo/
├── namespace.yaml
├── deployment.yaml
└── service.yaml
```

---

## Step 12: Create Namespace Manifest

### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-app
```

---

## Step 13: Create Deployment Manifest

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: nginx-app

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
      - name: nginx
        image: nginx:latest

        ports:
        - containerPort: 80
```

---

## Step 14: Create Service Manifest

### service.yaml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service
  namespace: nginx-app

spec:
  selector:
    app: nginx

  ports:
  - port: 80
    targetPort: 80

  type: LoadBalancer
```

---

## Step 15: Push Code to GitHub

Initialize and push the repository:

```bash
git init

git add .

git commit -m "Initial manifests"

git branch -M main

git remote add origin https://github.com/arvindgupta-cloud/argocd-nginx-demo.git

git push -u origin main
```

---

## Step 16: Create Application in Argo CD

Open the Argo CD UI and create a new application.

### General

```text
Application Name: nginx-demo
Project: default
```

### Source

```text
Repository URL:
https://github.com/arvindgupta-cloud/argocd-nginx-demo.git

Revision:
HEAD

Path:
.
```

### Destination

```text
Cluster:
https://kubernetes.default.svc

Namespace:
nginx-app
```

Click **Create**.

---

## Step 17: Sync Application

After creation, the application status will be:

```text
OutOfSync
```

Click:

```text
SYNC
```

Then:

```text
SYNCHRONIZE
```

Argo CD will deploy the manifests to the GKE cluster.

---

## Step 18: Verify Deployment

Check deployed resources:

```bash
kubectl get all -n nginx-app
```

Expected:

```text
deployment/nginx
service/nginx-service
pod/nginx
```

---

## Step 19: Access the Application

Get the external IP:

```bash
kubectl get svc -n nginx-app
```

Example:

```text
EXTERNAL-IP
34.xx.xx.xx
```

Open in browser:

```text
http://34.xx.xx.xx
```

You should see:

```text
Welcome to nginx!
```

---

## Step 20: Test GitOps Workflow

Update the deployment:

```yaml
replicas: 5
```

Commit and push:

```bash
git add .
git commit -m "Scale nginx"
git push
```

Argo CD detects the change and marks the application as:

```text
OutOfSync
```

Sync the application.

Verify scaling:

```bash
kubectl get pods -n nginx-app
```

Expected:

```text
5 Running Pods
```


## Step 21: Enable Auto Sync, Self-Healing, and Prune, Then Test GitOps Workflow

Update the Argo CD application and enable:

- Auto Sync = true
- Self-Heal = true
- Prune = true

This ensures Argo CD continuously compares the Kubernetes cluster state with the desired state stored in Git.

Test the GitOps workflow by manually changing the deployment replicas:

```text
kubectl scale deployment nginx --replicas=2 -n nginx-app
```

If the Git repository contains:

```text
replicas: 5
```

Argo CD will detect the drift and automatically revert the deployment back to 5 replicas. Similarly, if replicas are increased or decreased manually, Argo CD will restore the configuration defined in Git, demonstrating self-healing and Git as the single source of truth.

Update the deployment:

```text
replicas: 5
```

---

## Key Concepts Demonstrated

* Google Kubernetes Engine (GKE)
* Kubernetes Deployments and Services
* Argo CD Installation and Configuration
* GitOps Workflow
* Continuous Deployment
* Infrastructure as Code
* Kubernetes Namespace Management

---

## Future Enhancements

* Enable Argo CD Auto Sync
* Deploy a custom application instead of Nginx
* Integrate GitHub Actions
* Use Google Artifact Registry
* Add Ingress and TLS
* Implement Multi-Environment Deployments (Dev, Stage, Prod)
* Monitor with Prometheus and Grafana

---

## Author

Arvind Gupta

Portfolio: https://arvindgupta.cloud

GitHub: https://github.com/arvindgupta-cloud
