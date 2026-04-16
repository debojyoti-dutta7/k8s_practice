# Snapee Infrastructure Operations Guide

This repository contains the Infrastructure as Code (IaC) definitions for deploying the Snapee microservices architecture. The setup transitions the project from a manual local setup to a scalable, automated, and GitOps-driven deployment.

## Current Architecture

The infrastructure is defined modularly in the `k8s/` directory. Each of the 7 microservices is defined with:
*   **Deployment:** Configured to pull images from AWS ECR with defined CPU/Memory requests to enable autoscaling.
*   **Service:** Exposed internally via `ClusterIP` or `NodePort` depending on routing requirements.
*   **Horizontal Pod Autoscaler (HPA):** Configured to automatically scale pods between a minimum of 2 and a maximum of 10 replicas based on a 70% CPU utilization threshold.
*   **Ingress:** Centralized Nginx routing in `k8s/networking/ingress.yaml` handling domain paths (`devv-admin.snapecab.com`, `devv-fleet.snapecab.com`), complete with security headers and Regex URL rewrites.

All resources are isolated within the `snapee` namespace.

---

## What Has Been Completed (Minikube Local Setup)

The local environment has been initialized, core services deployed, and automated deployments configured:

### 1. Cluster & Network Foundation
*   Minikube was started and the Nginx Ingress controller was enabled (`minikube addons enable ingress`).
*   The isolated `snapee` namespace was established.
*   The `hosts` file was updated to route `devv-admin.snapecab.com` and `devv-fleet.snapecab.com` to `127.0.0.1`.
*   Ingress routes were configured with Regex path-rewriting (e.g., stripping `/wallet/` prefixes before passing to the backend) to resolve 404 errors.

### 2. Service Deployment & Configuration
*   All Kubernetes manifests (Deployments, Services, HPAs) were applied recursively.
*   **Secrets Management:** Environment variables for `dev-backend`, `wallet-service`, and `dev-microservice-campaign` were safely encoded into Kubernetes Secrets using the `stringData` approach, keeping them out of version control via `.gitignore`.
*   **Service Health:** 
    *   `dev-backend` is successfully running, connected to external databases, and reachable via Ingress.
    *   `wallet-service` is successfully running, connected to its respective databases, and reachable via Ingress.

### 3. AWS ECR Authentication
*   An IAM access key was used to generate an ECR login token.
*   The `aws-ecr-cred` secret was successfully injected into the cluster, allowing the successful pulling of private Docker images (resolving initial `ImagePullBackOff` errors for the configured services).

### 4. GitOps Implementation (ArgoCD)
*   ArgoCD was installed into the `argocd` namespace.
*   The `snapee-infra` Application was created, pointing to the Git repository.
*   ArgoCD was configured with **Recursive Directory Search** (`recurse: true`), allowing it to successfully discover and map the nested microservice folders, bringing the visual resource tree online.

---

## Remaining Setup Roadmap

Follow these steps to bring the remaining applications online.

### Step 1: Resolve Remaining Image Pull Errors
Some services (like `admin-frontend` and `dev-microservice-campaign`) are failing to pull images.
*   **Action:** Verify the exact ECR repository names and image tags (e.g., ensuring `snape/admin-service:latest` is correct) in the AWS Console, and update the corresponding `deployment.yaml` files.

### Step 2: Configure Remaining Secrets
*   **Action:** Create the `secret.yaml` files for the remaining services (`admin-frontend`, `automation-service`, `extrahourcron`, `payment-service`) containing their respective environment variables. Apply them using `kubectl apply -f <file>`.

### Step 3: Automate ECR Token Refresh
AWS ECR login tokens expire every 12 hours. A CronJob has been provided to automate this refresh so the cluster doesn't break daily.

1.  Copy the example credentials file:
    ```powershell
    cp k8s/utils/aws-iam-creds.yaml.example k8s/utils/aws-iam-creds.yaml
    ```
2.  Edit `k8s/utils/aws-iam-creds.yaml` and replace the placeholder values with your Base64 encoded AWS Access Key and Secret Key.
3.  Apply the utility manifests:
    ```powershell
    kubectl apply -f k8s/utils/
    ```

## Daily Operations Checklist
When resuming work, you must run these commands to "turn on" the local environment:
1.  **Start the Cluster:** `minikube start`
2.  **Open the Network Bridge:** `minikube tunnel` (Leave this terminal running).
3.  **Ensure AWS Auth:** If 12 hours have passed and the automated token refresher isn't running yet, re-run the `aws-ecr-cred` creation command.
