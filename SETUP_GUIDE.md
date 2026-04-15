# Full GKE Setup & Deployment Guide

This guide provides a step-by-step walkthrough to set up your GKE cluster, configure AWS ECR access, and deploy your entire infrastructure using ArgoCD.

---

## Phase 1: GKE Cluster Creation

Run these commands using the `gcloud` CLI. Replace `PROJECT_ID` with your actual Google Cloud Project ID.

### 1. Set your project and region
```bash
gcloud config set project PROJECT_ID
gcloud config set compute/region asia-south1
```

### 2. Create a VPC and Subnet (Optional but Recommended)
```bash
gcloud compute networks create snapee-vpc --subnet-mode=custom

gcloud compute networks subnets create snapee-subnet \
    --network=snapee-vpc \
    --range=10.0.0.0/24 \
    --region=asia-south1
```

### 3. Create the GKE Cluster
We will use a Standard cluster for better control over node pools.
```bash
gcloud container clusters create snapee-cluster \
    --network=snapee-vpc \
    --subnetwork=snapee-subnet \
    --zone=asia-south1-a \
    --num-nodes=3 \
    --enable-autoscaling --min-nodes=1 --max-nodes=10 \
    --workload-pool=PROJECT_ID.svc.id.goog
```

### 4. Get Credentials
```bash
gcloud container clusters get-credentials snapee-cluster --zone asia-south1-a
```

---

## Phase 2: AWS ECR Authentication

Since GKE is on Google Cloud and your images are on AWS ECR, we must create an image pull secret.

### 1. Generate ECR Login Token
```bash
# Get the token from AWS CLI
AWS_ECR_PASSWORD=$(aws ecr get-login-password --region ap-south-1)
```

### 2. Create the Secret in GKE
```bash
kubectl create namespace snapee

kubectl create secret docker-registry aws-ecr-cred \
    --docker-server=297436350639.dkr.ecr.ap-south-1.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$AWS_ECR_PASSWORD \
    --namespace=snapee
```
*Note: This token expires every 12 hours. See the migration plan for automation options.*

---

## Phase 3: Install & Configure ArgoCD

### 1. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Access ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Login at `https://localhost:8080` (User: `admin`).
Password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

### 3. Connect your Git Repo
In the ArgoCD UI:
1. Go to **Settings** -> **Repositories**.
2. Click **Connect Repo** and add your `k8s_practice` repository URL.

---

## Phase 4: Deploy the Infrastructure

Instead of manual `kubectl apply`, we use the ArgoCD Application manifest.

### 1. Update the Repo URL
Open `argocd/application.yaml` and ensure `repoURL` points to your Git repository.

### 2. Apply the Application
```bash
kubectl apply -f argocd/application.yaml
```

### 3. Monitor the Sync
Go to the ArgoCD Dashboard. You will see the `snapee-infra` application. Click it to see a visual map of:
*   **Deployments:** 1 Pod for each of your 7 services.
*   **HPAs:** Ready to scale your pods from 2 to 10 based on CPU load.
*   **Services:** NodePort services routing traffic.
*   **Ingress:** The GCE Load Balancer being provisioned.

---

## Phase 6: Install Helm & Nginx Ingress

Since we are using custom rewrite rules and large body sizes, the Nginx Ingress Controller is the best choice.

### 1. Install Helm
```bash
# Windows (Chocolatey)
choco install kubernetes-helm

# Linux/Mac
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 2. Add Nginx Ingress Repository
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### 3. Install the Controller
```bash
helm install snapee-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

---

## Phase 7: Verification

### 1. Check Pods
```bash
kubectl get pods -n snapee
```
Verify all pods are `Running`. If they show `ImagePullBackOff`, the AWS ECR secret is incorrect or expired.

### 2. Check HPA Status
```bash
kubectl get hpa -n snapee
```
Verify they are targeting the correct deployments and showing `0%/70%` load.

### 3. Get Load Balancer IP
```bash
kubectl get ingress -n snapee
```
Wait 5-10 minutes for the `ADDRESS` field to populate. Once it does, you can access your frontend and APIs.
