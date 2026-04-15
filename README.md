# Snapee Infrastructure Operations Guide

This repository contains the Infrastructure as Code (IaC) definitions for deploying the Snapee microservices architecture. The setup transitions the project from a manual local setup to a scalable, automated, and GitOps-driven deployment.

## Current Architecture

The infrastructure is defined modularly in the `k8s/` directory. Each of the 7 microservices is defined with:
*   **Deployment:** Configured to pull images from AWS ECR with defined CPU/Memory requests to enable autoscaling.
*   **Service:** Exposed internally via `NodePort`.
*   **Horizontal Pod Autoscaler (HPA):** Configured to automatically scale pods between a minimum of 2 and a maximum of 10 replicas based on a 70% CPU utilization threshold.
*   **Ingress:** Centralized Nginx routing in `k8s/networking/ingress.yaml` handling domain paths (`admin.snapecab.com`, `fleet.snapecab.com`), complete with security headers.

All resources are isolated within the `snapee` namespace.

---

## What Has Been Completed (Minikube Local Setup)

The local environment has been initialized and the infrastructure definition applied:

1.  **Cluster Initialization:** Minikube was started.
2.  **Addons:** The Nginx Ingress controller was enabled (`minikube addons enable ingress`).
3.  **Namespace Creation:** The isolated `snapee` environment was established (`kubectl apply -f k8s/namespace/namespace.yaml`).
4.  **Mass Deployment:** All manifests were applied recursively (`kubectl apply -R -f k8s/`).

*Current State:* All pods are currently in `ImagePullBackOff` state. This is expected until the AWS authentication step (below) is completed.

---

## Remaining Setup Roadmap

Follow these steps to bring the application online and finalize the automated setup.

### Step 1: AWS ECR Authentication (Fixing Image Pulls)
The cluster needs credentials to pull the private images from AWS ECR.

1.  Ensure you have the AWS CLI installed and configured.
2.  Generate a token and create the secret:
    ```powershell
    $AWS_ECR_PASSWORD = aws ecr get-login-password --region ap-south-1
    kubectl create secret docker-registry aws-ecr-cred `
        --docker-server=297436350639.dkr.ecr.ap-south-1.amazonaws.com `
        --docker-username=AWS `
        --docker-password=$AWS_ECR_PASSWORD `
        --namespace=snapee
    ```
3.  Verify the pods recover and transition to the `Running` state:
    ```powershell
    kubectl get pods -n snapee -w
    ```

### Step 2: Local Traffic Routing (Minikube Tunnel)
To access the services locally via the Ingress controller:

1.  In a separate terminal window, start the tunnel (keep this running):
    ```powershell
    minikube tunnel
    ```
2.  Update your local hosts file (`C:\Windows\System32\drivers\etc\hosts`) to map the domains to localhost:
    ```text
    127.0.0.1 admin.snapecab.com
    127.0.0.1 fleet.snapecab.com
    ```

### Step 3: Automate ECR Token Refresh
AWS ECR login tokens expire every 12 hours. A CronJob has been provided to automate this refresh.

1.  Copy the example credentials file:
    ```powershell
    cp k8s/utils/aws-iam-creds.yaml.example k8s/utils/aws-iam-creds.yaml
    ```
2.  Edit `k8s/utils/aws-iam-creds.yaml` and replace the placeholder values with your Base64 encoded AWS Access Key and Secret Key.
3.  Apply the utility manifests:
    ```powershell
    kubectl apply -f k8s/utils/
    ```

### Step 4: Implement GitOps with ArgoCD
Transition to GitOps for automated deployments.

1.  **Push Code to Git:** Ensure this codebase is pushed to your Git repository.
2.  **Install ArgoCD:**
    ```powershell
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
3.  **Update Application Source:** Edit `argocd/application.yaml` and update the `repoURL` line with your Git repository URL.
4.  **Deploy Application Definition:**
    ```powershell
    kubectl apply -f argocd/application.yaml
    ```
5.  **Access Dashboard:**
    ```powershell
    # Get Password
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    
    # Forward Port
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```
    Access UI at `https://localhost:8080` (Username: admin).

## Verification Checklist
- [ ] `kubectl get pods -n snapee` shows all 14 pods as `Running`.
- [ ] `kubectl get hpa -n snapee` successfully shows CPU targets instead of `<unknown>`.
- [ ] Navigating to `http://admin.snapecab.com` in your browser successfully loads the frontend.
- [ ] ArgoCD dashboard shows the `snapee-infra` application as green and `Synced`.
