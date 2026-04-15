# GKE Migration Plan (Hybrid approach with AWS ECR)

## Objective
Migrate existing Kubernetes workloads from their current environment to Google Kubernetes Engine (GKE), while continuing to use AWS Elastic Container Registry (ECR) for hosting container images.

## Key Files & Context
- **Deployments:** Located in `k8s/<service>/deployment.yaml` (e.g., `admin-frontend`, `dev-backend`, etc.). Currently configured to pull from AWS ECR (`297436350639.dkr.ecr.ap-south-1.amazonaws.com`) using `imagePullSecrets: [name: aws-ecr-cred]`.
- **Services:** Located in `k8s/<service>/service.yaml`. Services are exposed as `NodePort`.
- **Ingress:** `k8s/networking/ingress.yaml` is already configured to use the GCE Ingress controller (`kubernetes.io/ingress.class: "gce"`).
- **Secrets:** Environment variables are loaded from `secret.yaml` files.

## Implementation Steps

### 1. Prerequisite: AWS IAM Setup for ECR Access
Since GKE needs to pull images from AWS ECR, we need a mechanism to authenticate.
- **Action:** Create an AWS IAM User with programmatic access and an IAM Policy granting `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, and `ecr:BatchGetImage`.
- **Note:** AWS ECR login tokens expire every 12 hours. We will either need to manually refresh the `aws-ecr-cred` secret, or set up a CronJob within the GKE cluster to automatically fetch a new token from AWS and update the Kubernetes Secret.

### 2. GKE Cluster Setup
- **Action:** Provision a standard or Autopilot GKE cluster in the desired Google Cloud region.
- **Action:** Ensure the cluster has internet access (Cloud NAT if private nodes) to reach AWS ECR endpoints.

### 3. Kubernetes Secrets Migration
- **Action:** Apply the namespace `k8s/namespace/namespace.yaml`.
- **Action:** Migrate all application secrets (`secret.yaml` for each service) to the GKE cluster.
- **Action:** Create the `aws-ecr-cred` docker-registry secret in the `snapee` namespace on GKE using the credentials obtained in Step 1.

### 4. Deploy Workloads and Services
- **Action:** Apply all `deployment.yaml` and `service.yaml` manifests. No changes are strictly necessary to these files as they already reference the correct image paths and `NodePort` service types which are compatible with GCE Ingress.

### 5. Ingress and Networking
- **Action:** Apply `k8s/networking/ingress.yaml`. The GCE Ingress controller will automatically provision an External HTTP(S) Load Balancer in Google Cloud.
- **Action:** Retrieve the external IP address assigned to the Ingress resource.
- **Action:** Update DNS records to point the application domain(s) to the new GCE Load Balancer IP.

## Future Enhancements

### 1. Horizontal Scaling (HPA)
To enable automatic horizontal scaling for your services on GKE, you will need to implement the following:

- **Resource Requests & Limits:** Currently, your `deployment.yaml` files do not define resource constraints. Horizontal Pod Autoscaler (HPA) requires at least `cpu` requests to be defined to make scaling decisions.
    - **Action:** Update all `deployment.yaml` files to include:
      ```yaml
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      ```
- **HPA Resource:** Create a `HorizontalPodAutoscaler` manifest for each service.
    - **Action:** Add `hpa.yaml` files:
      ```yaml
      apiVersion: autoscaling/v2
      kind: HorizontalPodAutoscaler
      metadata:
        name: admin-frontend-hpa
        namespace: snapee
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: admin-frontend
        minReplicas: 2
        maxReplicas: 10
        metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 70
      ```
- **GKE Cluster Autoscaler:** Enable the Cluster Autoscaler in the GKE cluster settings so that Google Cloud can automatically provision new nodes if the HPA creates more pods than the current nodes can handle.

### 2. GitOps with ArgoCD
To visualize your infrastructure and automate deployments, ArgoCD is the recommended tool for GKE.

- **ArgoCD Installation:**
    - **Action:** Install ArgoCD into its own namespace (`argocd`) on the GKE cluster.
- **Application Definition:**
    - **Action:** Create an ArgoCD `Application` manifest that points to your Git repository (where these manifests are stored) and the destination cluster/namespace (`snapee`).
- **Benefits:**
    - **Visualization:** ArgoCD provides a real-time web UI showing the health of your Pods, Services, and Ingress.
    - **Syncing:** Any change pushed to your Git repository will be automatically applied to GKE.
    - **Drift Detection:** ArgoCD will alert you if someone manually changes the cluster configuration outside of Git.

## Verification & Testing
1. **Pod Status:** Ensure all pods in the `snapee` namespace reach the `Running` state, confirming successful image pulls from AWS ECR.
2. **HPA Testing:** Simulate load on a service and verify that the HPA triggers the creation of additional replicas.
3. **ArgoCD Sync:** Push a small change (like a label) to Git and verify ArgoCD detects and syncs it automatically.
4. **Ingress Allocation:** Verify the Ingress resource gets a public IP address allocated.
