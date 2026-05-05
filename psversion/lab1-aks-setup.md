# Lab 1: Prerequisites and AKS Setup

## Overview
This lab covers the prerequisites for working with AKS volumes and persistence, including setting up a resource group, creating an AKS cluster with public endpoints, and configuring cluster access.

## Learning Objectives
- Create Azure resource group
- Deploy AKS cluster with proper configuration
- Configure kubectl access to the cluster
- Verify cluster functionality

## Prerequisites
- Azure CLI installed and logged in
- kubectl installed
- Azure subscription with appropriate permissions

## Lab Steps

### Step 1: Set Environment Variables

Set up environment variables for consistent resource naming:
```powershell
# Set your initials (use 2-4 lowercase letters)
$STUDENT_INITIALS = "abc"  # CHANGE THIS to your initials

# Verify it's set
Write-Host "Your initials: $STUDENT_INITIALS"
```

```powershell
# Set your preferred values
$RESOURCE_GROUP = "rg-aks-volumes-lab-$STUDENT_INITIALS"
$CLUSTER_NAME = "aks-volumes-$STUDENT_INITIALS"
$LOCATION = "swedencentral"
$NODE_COUNT = 3
```

### Step 2: Create Resource Group

Create a resource group to contain all lab resources:

```powershell
# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Verify resource group creation
az group show --name $RESOURCE_GROUP --output table
```

### Step 3: Create AKS Cluster

Create an AKS cluster with the following specifications:
- 3 nodes
- Single node pool
- Public API server endpoint
- System-assigned managed identity

```powershell
# Create AKS cluster
az aks create --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --location $LOCATION --node-count $NODE_COUNT --node-vm-size Standard_D4s_v5 --enable-managed-identity --generate-ssh-keys --enable-addons monitoring --network-plugin azure --network-policy azure --api-server-authorized-ip-ranges "0.0.0.0/0"

# This command takes 5-10 minutes to complete
Write-Host "AKS cluster creation initiated. This may take several minutes..."
```

**Note**: The `--api-server-authorized-ip-ranges "0.0.0.0/0"` parameter allows access from any IP address. In production, restrict this to specific IP ranges for security.

### Step 4: Get AKS Credentials

Configure kubectl to connect to your AKS cluster:

```powershell
# Get cluster credentials
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --overwrite-existing

# Verify kubectl context
kubectl config current-context
```

### Step 5: Verify Cluster Status

Verify that your cluster is running properly:

```powershell
# Check cluster nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check cluster info
kubectl cluster-info

# Verify node pools
az aks nodepool list --resource-group $RESOURCE_GROUP --cluster-name $CLUSTER_NAME --output table
```

Expected output should show:
- 3 nodes in Ready status
- System pods running successfully
- Node pool with 3 nodes

### Step 6: Enable Additional CSI Drivers (Optional)

Enable additional CSI drivers that will be used in subsequent labs:

```powershell
# Enable Azure Blob CSI driver (for Lab 4)
az aks update --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --enable-blob-driver

# Verify CSI drivers
kubectl get storageclass
kubectl get csidriver
```

### Step 7: Verification Commands

Run these commands to ensure everything is set up correctly:

```powershell
# Test cluster connectivity
kubectl get namespaces

# Check available storage classes
kubectl get storageclass

# Verify CSI drivers
kubectl get csidriver

# Check node resources
kubectl describe nodes
```

## Expected Results

After completing this lab, you should have:
- ✅ Resource group created in Norway East region
- ✅ AKS cluster with 3 nodes running
- ✅ kubectl configured to access the cluster
- ✅ Default storage classes available
- ✅ CSI drivers installed and ready

## Troubleshooting

### Common Issues

1. **Authentication Errors**
   ```powershell
   # Re-login to Azure CLI
   az login
   
   # Check current subscription
   az account show
   ```

2. **kubectl Connection Issues**
   ```powershell
   # Re-fetch credentials
   az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --overwrite-existing
   ```

3. **Node Not Ready**
   ```powershell
   # Check node status
   kubectl describe node <node-name>
   
   # Check system pods
   kubectl get pods -n kube-system
   ```

## Cost Considerations

This lab creates:
- AKS cluster (3 Standard_D2s_v3 nodes)
- Public IP for load balancer
- Managed disks for nodes

Estimated cost: ~$200-300/month if left running. Remember to clean up resources after completing labs.

## Next Steps

Once your cluster is ready and verified, proceed to [Lab 2: CSI Driver and Dynamic Provisioning](./lab2-csi-dynamic-volumes.md).

## Cleanup (When Done with All Labs)

**Important**: Only run these commands when you're completely done with all labs:

```powershell
# Delete AKS cluster and all associated resources
az group delete --name $RESOURCE_GROUP --yes --no-wait

# Remove kubectl context
kubectl config delete-context $CLUSTER_NAME
```

## Additional Resources

- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)
- [AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)
- [Azure CLI Reference](https://docs.microsoft.com/en-us/cli/azure/aks)
