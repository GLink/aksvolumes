# AKS Volumes and Persistence Labs

This repository contains hands-on labs for learning about Azure Kubernetes Service (AKS) volumes and persistent storage. These labs are designed to provide practical experience with various storage solutions in AKS.

## Labs Overview

1. **[Lab 1: Prerequisites and AKS Setup](./lab1-aks-setup.md)**
   - Setting up resource group and AKS cluster
   - Configuring cluster credentials
   - Basic cluster verification

2. **[Lab 2: CSI Driver and Dynamic Provisioning](./lab2-csi-dynamic-volumes.md)**
   - Working with CSI drivers
   - Creating storage classes
   - Dynamic volume provisioning with PVCs

3. **[Lab 3: PostgreSQL with Azure Disk and Read Replicas](./lab3-postgresql-replicas.md)**
   - Deploying PostgreSQL with persistent storage
   - Setting up read replicas
   - Testing failover scenarios

4. **[Lab 4: Blob Storage with CSI Driver](./lab4-blob-storage.md)**
   - Using Azure Blob Storage CSI driver
   - Block storage scenarios
   - Performance considerations

## Prerequisites

- Azure CLI installed and configured
- kubectl installed
- Basic knowledge of Kubernetes concepts
- Valid Azure subscription with appropriate permissions

## Getting Started

Start with Lab 1 and progress through the labs in order, as each lab builds upon concepts from previous labs.

## Clean Up

Each lab includes cleanup instructions. Make sure to follow them to avoid unnecessary Azure charges.