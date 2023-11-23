---
title: Integrate Azure Container Registry ACR with AKS
description: Build a Docker Image, Push to Azure Container Registry and  Attach ACR with AKS
---

# Integrate Azure Container Registry ACR with AKS

# Architecture Diagram

![Integrate Azure Container Registry ACR with AKS](./arch-diagram/Azure%20ACR%20and%20AKS%20-%20Integration.drawio.png)

# Commentary on Architectural Diagram

## Overview

The attached architectural diagram shows how to integrate Azure Container Registry (ACR) with Azure Kubernetes Service (AKS). ACR is a container registry service that allows developers to build, store, and manage container images. AKS is a managed Kubernetes service that allows developers to deploy and manage containerized applications.

## Integration

There are two main steps to integrate ACR with AKS:

1. Authorize AKS to pull images from ACR. This can be done using an Azure Active Directory (Azure AD) service principal or a managed identity.
   2.0Configure AKS to use ACR. This can be done using the Azure CLI, Azure PowerShell, or the Azure portal.

Once AKS is authorized to pull images from ACR and configured to use ACR, pods in the AKS cluster can pull images from ACR without any additional configuration.

## Diagram commentary

The diagram shows the following

1. shows a single AKS cluster and a single ACR. However, you can integrate multiple AKS clusters with a single ACR, or a single AKS cluster with multiple ACRs.

2. shows a public IP address for the AKS Load Balancer. However, you can also use a private IP address for the AKS Load Balancer, or you can use a managed Azure service such as Azure Application Gateway.

3. shows an Nginx application as the AKS application. However, you can deploy any type of containerized application to AKS.

## Benefits of integrating ACR with AKS

There are several benefits to integrating ACR with AKS:

1. Simplified image management: ACR provides a central location to store and manage container images. This simplifies image management and makes it easier to share images across teams and projects.
2. Improved security: ACR provides security features such as role-based access control (RBAC) and image signing. This helps to keep your container images secure.
3. Increased performance: ACR is a highly scalable and performant service. This means that your AKS applications can quickly and reliably pull images from ACR.

Overall, integrating ACR with AKS is a simple and effective way to improve the security, performance, and manageability of your containerized applications.

# Steps to follow

## Step 1: Pre-requisites

- Azure AKS Cluster should be running with azure virtual nodes option 'on'

### Step 1-1: Configure Command Line Credentials

az aks get-credentials --name aksdemo2 --resource-group aks-demo-gp2

### Step 1-2: Verify Nodes

kubectl get nodes
kubectl get nodes -o wide

### Step 1-3: Verify aci-connector-linux

kubectl get pods -n kube-system

### Step 1-4: Verify logs of ACI Connector Linux

kubectl logs -f $(kubectl get po -n kube-system | egrep -o 'aci-connector-linux-[A-Za-z0-9-]+') -n kube-system

## Step 2: Create Azure Container Registry

- Go to Services -> Container Registries
- Click on Add
- Resource Group: aks-demo-gp2
- Registry Name: acrforaksdemo2test
- Location: East US
- Click on 'Review + Create'

## Step 3: Build Docker Image Locally

### Step 3-1: Change Directory

cd docker-manifests

### Step 3-2: Docker Build

docker build -t kube-nginx-acr:v1 .

### Step 3-3: List Docker Images

docker images kube-nginx-acr:v1

## Step 4: Run Docker Container locally and test

### Step 4-1: Run locally and Test

docker run --name kube-nginx-acr --rm -p 80:80 -d kube-nginx-acr:v1

### Step 4-2: Access Application locally

http://localhost

### Step 4-3: Stop Docker Image

docker stop kube-nginx-acr

## Step 5: Enable Docker Login for ACR Repository

- Go to Services -> Container Registries -> acrforaksdemo2
- Go to Access Keys
- Click on Enable Admin User
- Make a note of Username and password

## Step 6: Push Docker Image to ACR

### Step 6-1: Build, Test Locally, Tag and Push to ACR

### Step 6-1-1: Login to ACR

docker login acrforaksdemo2test.azurecr.io

### Step 6-1-2: Tag

docker tag kube-nginx-acr:v1 acrforaksdemo2test.azurecr.io/app1/kube-nginx-acr:v1

### Step 6-1-3: List Docker Images to verify

docker images kube-nginx-acr:v1
docker images acrforaksdemo2test.azurecr.io/app1/kube-nginx-acr:v1

### Step 6-1-4: Push Docker Images

docker push acrforaksdemo2test.azurecr.io/app1/kube-nginx-acr:v1

### Step 6-1-5: Verify Docker Image in ACR Repository

- Go to Services -> Container Registries -> acrforaksdemo2test
- Go to Repositories -> app1/kube-nginx-acr

### Step 6-1-6: Configure ACR integration for existing AKS clusters

az aks update -n myAKSCluster -g myResourceGroup --attach-acr acrforaksdemo2test

### Step 6-1-7: Replace Cluster, Resource Group and ACR Repo Name

az aks update -n aksdemo2 -g aks-demo-gp2 --attach-acr acrforaksdemo2test

## Step 7: Update & Deploy to AKS & Test

### Step 7-1: Update Deployment Manifest with Image Name

```yaml
spec:
  containers:
    - name: acrdemo-localdocker
      image: acrforaksdemo2test.azurecr.io/app1/kube-nginx-acr:v1
      imagePullPolicy: Always
      ports:
        - containerPort: 80
```

### Step 7-2: Deploy to AKS and Test

### Step 7-2-1: Deploy

kubectl apply -f kube-manifests/

### Step 7-2-2: List Pods

kubectl get pods

### Step 7-2-3: Describe Pod

kubectl describe pod 'pod-name'

### Step 7-2-4: Get Load Balancer IP

kubectl get svc and copy external IP

## Step 8: Clean-Up

### Step 8-1: Delete Applications

kubectl delete -f kube-manifests/

### Step 8-2: Detach ACR from AKS Cluster

### Step 8-2-1: Detach ACR with AKS Cluster

az aks update -n aksdemo2 -g aks-demo-gp2 --detach-acr acrforaksdemo2test

### Step 8-2-2: Delete ACR Repository

Go To Services -> Container Registries -> acrforaksdemo2 -> Delete it
