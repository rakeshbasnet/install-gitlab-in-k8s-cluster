# GitLab and GitLab Runner Deployment on Kubernetes

## Overview

This repository contains all the necessary deployment files to set up GitLab and GitLab Runner on a Kubernetes cluster. Additionally, it includes a custom GitLab Runner Dockerfile located in the gitlab-runner/ folder.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Repository Structure](#repository-structure)
3. [Clone Repository](#clone-repository)
4. [Custom Gitlab Runner Setup](#custom-gitlab-runner-setup)
5. [Deployment Instructions](#deployment-instructions)
    - [Create Namespace](#create-namespace)
    - [Deploy Persistent Volume](#pdeploy-persistent-volume)
    - [Deploy Gitlab and Gitlab Runner](#deploy-gitlab-and-gitlab-runner)
    - [Expose gitlab Service](#expose-gitlab-service)
6. [Verify Deployment](#verify-deployment)
7. [Access Gitlab](#access-gitlab)
8. [Create Variables and Runner](#create-variables-and-runner)
9. [Register Gitlab Runner](#register-gitlab-runner)
10. [Cleanup Resources](#cleanup-resources)
11. [Challenges and Solutions](#challenges-and-solutions)
12. [Conclusion](#conclusion)
---

## Prerequisites

- A running Kubernetes (K8s) cluster.
- [kubectl installed and configured](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to interact with the K8s cluster.
- [Docker](https://docs.docker.com/get-docker/) installed to build custom images.
- Sufficient storage on the VM for persistent volumes.

---

## Repository Structure

```bash
    .
    ├── gitlab-namespace.yaml       # Namespace definition for GitLab
    ├── gitlab-pv-pvc.yaml          # Persistent Volume and Persistent Volume Claim
    ├── gitlab-deployment.yaml      # Deployment configuration for GitLab
    ├── gitlab-service.yaml         # Service configuration for exposing GitLab
    ├── gitlab-ingress.yaml         # Ingress configuration for exposing GitLab to domain (Optional)
    ├── gitlab-runner/              # Folder containing custom GitLab Runner Dockerfile
    │   ├── Dockerfile              # Custom GitLab Runner image
```

---

## Clone Repository
```bash
   git clone https://github.com/rakeshbasnet/docker-compose-example.git
   cd docker-compose-example
```
---

## Custom Gitlab Runner Setup
Navigate to the gitlab-runner folder and build the image:
```bash
    cd gitlab-runner
    docker build -t <your-dockerhub-username>/gitlab-runner .
    docker push <your-dockerhub-username>/gitlab-runner
```
---
## Deployment Instructions

### Create namespace
Apply the namespace configuration:
```bash
  kubectl apply -f gitlab-namespace.yaml
```   
### Deploy Persistent Volume
Apply the persistent storage configuration:
```bash
  kubectl apply -f gitlab-pv-pvc.yaml
```   
### Deploy Gitlab and Gitlab Runner
Apply the deployment configuration:
```bash
  kubectl apply -f gitlab-deployment.yaml
``` 

### Expose Gitlab Service
Apply the service configuration:
```bash
  kubectl apply -f gitlab-service.yaml
``` 
---

## Verify Deployment

Check namespace:
```bash
  kubectl get namespaces
```   
Check Deployments:
```bash
  kubectl get deployments -n gitlab
```  
Check pods:
```bash
  kubectl get pods -n gitlab
```  
Check services:
```bash
  kubectl get services -n gitlab
```  
---
## Access Gitlab
Once deployed, access GitLab at:
```bash
http://<Azure_VM_IP>:30085
``` 
Default username: root

Retrieve the default password:
```bash
kubectl exec -it <pod-name> -n gitlab -- grep 'Password:' /etc/gitlab/initial_root_password
```   
---

## Create Variables and Runner
To use GitLab CI/CD pipelines, you need to add the required environment variables and register a GitLab Runner.

1. Go to Settings → CI/CD in your GitLab instance.
2. Under Variables, add necessary variables like `DOCKER_USERNAME`, `DOCKER_PASSWORD`, `GITLAB_ACCESS_TOKEN`, etc.
3. Under Runners, click on New Project Runner.
4. If using a NodePort, replace the pod's name with your VM's IP address.
Example URL to register the runner:

```bash
http://<vm-ip>:<port>/root/gitlabci-and-argocd-example/-/runners/4/register
```  
Select your runner platform (Linux for this example), then proceed with registration.

---

## Register Gitlab Runner
Once you've obtained the runner token, run the following command to register your GitLab Runner inside the pod:
Access the pod with shell execution:
```bash
kubectl exec -it <pod-name> -n gitlab -c <container-name> /bin/bash
```  
Now resgister the runner:
```bash
gitlab-runner register \
  --url http://<vm-ip>:30085 \
  --token <gitlab_runner_token>
``` 
Follow the prompts to complete the registration process. The configuration will be stored in the `/etc/gitlab/config.toml` file inside the persistent volume.
---

## Cleanup Resources
To delete the resources you created for GitLab, run the following commands:
```bash
kubectl delete -f gitlab-service.yaml
kubectl delete -f gitlab-deployment.yaml
kubectl delete -f gitlab-pv-pvc.yaml
kubectl delete -f gitlab-namespace.yaml
``` 
---

## Challenges and Solutions

### 1. ** Insufficient Disk Space and Domain Requirement for GitLab Deployment via Helm**:
- **Problem**: 
    While attempting to deploy GitLab using a Helm chart, two issues were encountered:
    1. Domain Name Requirement: GitLab installation requires a domain name to configure external access. As the VM didn’t have a domain name, the Helm chart deployment process was stuck during configuration.
    2. Insufficient Disk Space: The VM didn't have enough disk space to create multiple Persistent Volumes (PVs) and Persistent Volume Claims (PVCs) for GitLab. GitLab's large storage requirements (for logs, configuration, and application data) led to the deployment being stuck in the in-progress state due to insufficient resources.
- **Solution**:
    - We switched to a different approach to deploy GitLab without relying on Helm and without requiring domain names or excessive disk space.
    - Instead, we deployed GitLab in a Docker container and manually created the required deployment files for GitLab inside the Kubernetes cluster. This removed the dependency on Helm's complex configuration and external domain name requirements.


### 2. **Issues with Azure VM as GitLab Runner**:
- **Problem**
    - After deploying GitLab using the Docker approach, we moved on to configure a GitLab Runner on an Azure VM. However, a significant challenge arose when the GitLab Runner attempted to clone the repository from GitLab. The Git URL generated by GitLab was pod-based (e.g., git://pod-name) because of the NodePort and pod access method used in GitLab’s deployment.
    - The Azure VM running the GitLab Runner could not resolve the pod-based URL due to the VM not being part of the Kubernetes cluster’s internal DNS. This prevented the GitLab Runner from successfully cloning the repository, causing the CI/CD pipeline to fail.

- **Solution**:
    - To solve this, we deployed GitLab Runner within the Kubernetes cluster itself to allow for better network connectivity and DNS resolution.

### 3. **Challenges Faced During GitLab Runner Deployment Inside Kubernetes**:
- **Problem**
    To address the issue from Challenge 2, we deployed GitLab Runner as a separate deployment inside the same Kubernetes cluster where GitLab was running. However, deploying the runner within Kubernetes introduced new challenges:
    - Loss of Configuration: If the GitLab Runner pod was restarted (as part of Kubernetes’ auto-healing process), the configuration files for GitLab Runner (including the Docker configuration and GitLab registration settings) would be lost, as they were not persisted across pod restarts.
    - Pod Name Resolution Issue: The GitLab Runner within the Kubernetes cluster was still facing the issue where the pod-name was being used in the GitLab URL, which couldn’t be resolved because of the DNS resolution limitations between the runner and GitLab pods. The internal DNS only allowed pod-to-pod communication within the Kubernetes cluster, which made it difficult to use the runner with external resources, such as the VM or GitLab instance outside of Kubernetes.

- **Solution**: 
    To solve this issue, we tried to deploy gitlab-runner as a sidecar container with gitlab container.

### 4. **Kubernetes Networking**:
- **Problem**
    After attempting to deploy GitLab Runner as a sidecar container within the same pod as GitLab, several issues arose:
    - Memory Constraints: Running both GitLab and GitLab Runner within the same pod led to high memory consumption. The combination of the GitLab container and the runner container placed a significant load on the available resources, resulting in insufficient memory for proper operation, especially when dealing with resource-heavy workloads.
    - Loss of Docker Configuration: As before, the sidecar approach also faced the issue where Docker configuration and GitLab Runner registration settings were lost if the pod was restarted. Even with the sidecar approach, the lack of persistent storage for configuration files meant that the runner had to be re-registered after each pod restart, disrupting the workflow.

- **Solution**:
    - To address the memory constraint issue, we carefully monitored the pod's resource limits and allocated sufficient CPU and memory for both containers.
    - To overcome the configuration loss, we created Persistent Volumes (PVs) and Persistent Volume Claims (PVCs) for storing Docker configuration files and GitLab Runner settings. This ensured that even after pod restarts, the GitLab Runner maintained its configuration and did not need to be re-registered.

---

## Conclusion

This guide helps you set up a fully functional GitLab and GitLab Runner deployment on Kubernetes Cluster. However the gitlab and gitlab runner are configured succesfully inside a k8s cluster, still we have to register the gitlab runner manually for the first time because we don’t have any gitlab runner token during the first deployment to register runner since the gitlab also gets deployed at the same time.