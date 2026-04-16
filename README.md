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

## What Has Been Completed

The local environment has been initialized, core services deployed, and automated deployments configured:

### 1. Cluster & Network Foundation
*   Minikube was started and the Nginx Ingress controller was enabled.
*   The isolated `snapee` namespace was established.
*   The `hosts` file was updated to route `devv-admin.snapecab.com` and `devv-fleet.snapecab.com` to `127.0.0.1`.
*   Ingress routes were configured with Regex path-rewriting (e.g., stripping `/wallet/` prefixes before passing to the backend) to resolve 404 errors.

### 2. Service Deployment & Configuration
*   **Secrets Management:** Environment variables for `dev-backend`, `wallet-service`, and `dev-microservice-campaign` were safely encoded into Kubernetes Secrets.
*   **Service Health:** `dev-backend` and `wallet-service` are successfully running, connected to databases, and reachable via Ingress.

### 3. AWS ECR Authentication
*   The `aws-ecr-cred` secret was successfully injected into the cluster, allowing the successful pulling of private Docker images.

### 4. GitOps Implementation (ArgoCD)
*   ArgoCD was installed and configured with **Recursive Directory Search**, bringing the visual resource tree online.

---

## Operational Command Reference

### 🛠️ Environment Management
| Command | Use / Explanation |
| :--- | :--- |
| `minikube start` | Powers up the local Kubernetes cluster. |
| `minikube tunnel` | Creates a network bridge to allow your browser to talk to the Ingress. |
| `minikube addons enable ingress` | Installs the Nginx Ingress Controller into the cluster. |

### 🔍 Cluster Observation
| Command | Use / Explanation |
| :--- | :--- |
| `kubectl get pods -n snapee` | Checks the status of all your application pods (e.g., Running, Error). |
| `kubectl logs -l app=dev-backend -n snapee` | Views the real-time application logs (useful for debugging DB connections). |
| `kubectl describe pod <name> -n snapee` | Shows detailed events (useful for finding why an image pull failed). |
| `kubectl get hpa -n snapee` | Checks the status of the Autoscalers and current CPU load. |

### 🚀 Deployment & Lifecycle
| Command | Use / Explanation |
| :--- | :--- |
| `kubectl apply -f <path>` | Applies configuration changes from a YAML file to the cluster. |
| `kubectl apply -R -f k8s/` | Re-deploys every service in the `k8s` folder recursively. |
| `kubectl rollout restart deployment <name> -n snapee` | Restarts pods to force them to pick up new Secret/Config changes. |

### 🔐 Authentication & Secrets
| Command | Use / Explanation |
| :--- | :--- |
| `aws ecr get-login-password` | Generates a temporary login token from your AWS credentials. |
| `kubectl create secret docker-registry aws-ecr-cred ...` | Stores AWS credentials in K8s so it can pull private images. |
| `kubectl get secrets -n snapee` | Lists all active secrets (verify if auth keys exist). |

### 🐙 ArgoCD Operations
| Command | Use / Explanation |
| :--- | :--- |
| `kubectl port-forward svc/argocd-server -n argocd 8080:443` | Maps the ArgoCD dashboard to `https://localhost:8080`. |
| `kubectl -n argocd get secret argocd-initial-admin-secret...` | Retrieves the default admin password for the ArgoCD UI. |

---

## Daily Operations Checklist
When resuming work, you must run these commands to "turn on" the local environment:
1.  **Start the Cluster:** `minikube start`
2.  **Open the Network Bridge:** `minikube tunnel` (Leave this terminal running).
3.  **Ensure AWS Auth:** If 12 hours have passed, re-run the `aws-ecr-cred` creation command.
