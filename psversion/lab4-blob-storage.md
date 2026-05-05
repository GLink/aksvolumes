# Lab 4: Blob Storage with CSI Driver

## Overview
This lab demonstrates using Azure Blob Storage with the Azure Blob CSI driver for block storage scenarios in AKS. You'll learn how to mount Azure Blob containers as volumes, understand different access modes, and explore use cases for blob storage in Kubernetes workloads.

## Learning Objectives
- Configure Azure Blob Storage CSI driver
- Create storage accounts and containers
- Mount blob storage as Kubernetes volumes
- Understand different blob storage access modes (NFS, Fuse)
- Implement shared storage scenarios
- Compare performance characteristics

## Prerequisites
- Completed Lab 1 (AKS cluster setup)
- Completed Lab 2 (Understanding CSI drivers)
- kubectl configured and connected to your AKS cluster
- Azure CLI logged in with appropriate permissions

## Lab Steps

### Step 1: Verify Blob CSI Driver

First, verify that the Azure Blob CSI driver is enabled on your cluster:

```powershell
# Check if blob CSI driver is installed
kubectl get csidriver | Select-String blob

# Check blob CSI driver pods
kubectl get pods -n kube-system | Select-String blob

# If not enabled, enable it
$RESOURCE_GROUP = "rg-aks-volumes-lab"
$CLUSTER_NAME = "aks-volumes-cluster"
az aks update `
    --resource-group $RESOURCE_GROUP `
    --name $CLUSTER_NAME `
    --enable-blob-driver

# Wait for driver to be ready
kubectl wait --for=condition=Ready pods -l app=csi-blob-controller -n kube-system --timeout=300s
```

### Step 2: Create Azure Storage Account

Create a storage account for blob storage:

```powershell
# Set variables
$RESOURCE_GROUP = "rg-aks-volumes-lab"
$LOCATION = "norwayeast"
$STORAGE_ACCOUNT = "aksblobstorage$(Get-Random -Minimum 100000 -Maximum 999999)"
$CONTAINER_NAME = "data-container"
echo "Storage Account: $STORAGE_ACCOUNT"

# Create storage account with hierarchical namespace for NFS support
az storage account create `
    --resource-group $RESOURCE_GROUP `
    --name $STORAGE_ACCOUNT `
    --location $LOCATION `
    --sku Standard_LRS `
    --kind StorageV2 `
    --access-tier Hot `
    --https-only true `
    --min-tls-version TLS1_2 `
    --allow-blob-public-access false `
    --allow-shared-key-access true `
    --enable-hierarchical-namespace true `
    --enable-nfs-v3 true

# Get storage account key
$STORAGE_KEY = az storage account keys list `
    --resource-group $RESOURCE_GROUP `
    --account-name $STORAGE_ACCOUNT `
    --query '[0].value' -o tsv

echo "Storage Key: $STORAGE_KEY"

# Create blob container using Azure AD authentication (more secure)
az storage container create `
    --name $CONTAINER_NAME `
    --account-name $STORAGE_ACCOUNT `
    --auth-mode login `
    --public-access off

# Verify container creation using Azure AD auth
az storage container list `
    --account-name $STORAGE_ACCOUNT `
    --auth-mode login `
    --output table
```

### Step 3: Configure Azure AD Authentication for CSI Driver

Instead of using storage account keys, we'll use Azure AD authentication with managed identity, which is more secure and required in environments that disable storage account keys:

```powershell
# Get the AKS cluster's managed identity
$CLUSTER_IDENTITY = az aks show `
    --resource-group $RESOURCE_GROUP `
    --name aks-volumes-cluster `
    --query identityProfile.kubeletidentity.clientId `
    --output tsv

echo "Cluster Identity: $CLUSTER_IDENTITY"

# Assign Storage Blob Data Contributor role to the AKS managed identity
az role assignment create `
    --role "Storage Blob Data Contributor" `
    --assignee $CLUSTER_IDENTITY `
    --scope "/subscriptions/$(az account show --query id --output tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT"

# Create namespace for blob storage demos
kubectl create namespace blob-storage

# Create secret with storage account name and empty key (required by CSI driver)
kubectl create secret generic azure-storage-secret `
    --from-literal=azurestorageaccountname=$STORAGE_ACCOUNT `
    --from-literal=azurestorageaccountkey="" `
    --namespace blob-storage

# Verify secret creation
kubectl get secret azure-storage-secret -n blob-storage -o yaml
```

### Step 4: Create Storage Classes for Blob Storage

Create different storage classes for various blob storage scenarios:

```powershell
# Create storage class for Fuse-based blob mounting with managed identity
@"
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: blob-fuse
provisioner: blob.csi.azure.com
parameters:
  protocol: fuse
  skuName: Standard_LRS
  containerName: $CONTAINER_NAME
  # Managed identity configuration
  csi.storage.k8s.io/provisioner-secret-name: azure-storage-secret
  csi.storage.k8s.io/provisioner-secret-namespace: blob-storage
  csi.storage.k8s.io/node-stage-secret-name: azure-storage-secret
  csi.storage.k8s.io/node-stage-secret-namespace: blob-storage
  csi.storage.k8s.io/controller-publish-secret-name: azure-storage-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: blob-storage
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
mountOptions:
  - -o allow_other
  - --file-cache-timeout-in-seconds=120
  - --use-attr-cache=true
  - --cancel-list-on-mount-seconds=10
  - -o attr_timeout=120
  - -o entry_timeout=120
  - -o negative_timeout=120
  - --log-level=LOG_WARNING
  - --cache-size-mb=1000
  - --use-adls=false
"@ | kubectl apply -f -

# Verify storage classes
kubectl get storageclass | Select-String blob

# Verify managed identity permissions are working
echo "Verifying managed identity setup..."
$CLUSTER_IDENTITY = az aks show `
    --resource-group $RESOURCE_GROUP `
    --name aks-volumes-cluster `
    --query identityProfile.kubeletidentity.clientId `
    --output tsv

echo "Cluster Identity: $CLUSTER_IDENTITY"

# Check role assignment
az role assignment list `
    --assignee $CLUSTER_IDENTITY `
    --scope "/subscriptions/$(az account show --query id --output tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT" `
    --output table
```

### ⚠️ **Important Note About NFS Protocol**

Azure Blob Storage NFS v3 has specific requirements:
- **Hierarchical Namespace (HNS)** must be enabled on the storage account
- **Premium performance tier** is recommended for NFS
- **Virtual Network integration** may be required for enterprise scenarios
- **Regional availability** - NFS v3 isn't available in all regions

For learning purposes, **we'll focus on the FUSE protocol** which is more universally compatible and easier to set up. The NFS example is provided as an optional advanced exercise.

### Step 5: Create PVC for FUSE-based Blob Storage

Since dynamic provisioning with managed identity can be inconsistent, we'll use static provisioning for more reliable results:

```powershell
# Create static PV and PVC for Fuse-based blob storage
@"
apiVersion: v1
kind: PersistentVolume
metadata:
  name: blob-fuse-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  csi:
    driver: blob.csi.azure.com
    readOnly: false
    volumeHandle: blob-fuse-volume-handle
    volumeAttributes:
      resourceGroup: $RESOURCE_GROUP
      storageAccount: $STORAGE_ACCOUNT
      containerName: $CONTAINER_NAME
      protocol: fuse
    nodeStageSecretRef:
      name: azure-storage-secret
      namespace: blob-storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: blob-fuse-pvc
  namespace: blob-storage
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: blob-fuse-pv
  storageClassName: ""
"@ | kubectl apply -f -

# Check PVC status
kubectl get pvc -n blob-storage
kubectl describe pvc blob-fuse-pvc -n blob-storage

### Step 6: Deploy Applications Using Blob Storage

Create applications that demonstrate different blob storage usage patterns:

```powershell
# Deploy multi-pod application with shared blob storage
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blob-writer
  namespace: blob-storage
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blob-writer
  template:
    metadata:
      labels:
        app: blob-writer
    spec:
      containers:
      - name: writer
        image: busybox:1.35
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting blob writer..."
          mkdir -p /data/logs /data/shared
          while true; do
            echo "`$(date): Writer `$HOSTNAME writing data" >> /data/logs/writer-`$HOSTNAME.log
            echo "`$(date): Shared data from `$HOSTNAME" >> /data/shared/shared-data.txt
            ls -la /data/ >> /data/logs/directory-listing-`$HOSTNAME.log
            sleep 30
          done
        volumeMounts:
        - name: blob-storage
          mountPath: /data
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumes:
      - name: blob-storage
        persistentVolumeClaim:
          claimName: blob-fuse-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blob-reader
  namespace: blob-storage
spec:
  replicas: 3
  selector:
    matchLabels:
      app: blob-reader
  template:
    metadata:
      labels:
        app: blob-reader
    spec:
      containers:
      - name: reader
        image: busybox:1.35
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting blob reader..."
          while true; do
            echo "=== Reader `$HOSTNAME Report `$(date) ==="
            echo "Shared data content:"
            if [ -f /data/shared/shared-data.txt ]; then
              tail -5 /data/shared/shared-data.txt
            else
              echo "Shared data file not found yet"
            fi
            echo ""
            echo "Available log files:"
            ls -la /data/logs/ 2>/dev/null || echo "Logs directory not found yet"
            echo ""
            sleep 45
          done
        volumeMounts:
        - name: blob-storage
          mountPath: /data
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumes:
      - name: blob-storage
        persistentVolumeClaim:
          claimName: blob-fuse-pvc
"@ | kubectl apply -f -

# Wait for deployments to be ready
kubectl wait --for=condition=Available deployment/blob-writer -n blob-storage --timeout=300s
kubectl wait --for=condition=Available deployment/blob-reader -n blob-storage --timeout=300s

# Check pod status
kubectl get pods -n blob-storage -o wide
```

### Step 7: Verify Shared Storage Functionality

Test that data is shared across pods:

```powershell
# Check blob writer logs
kubectl logs -l app=blob-writer -n blob-storage --tail=20

# Check blob reader logs
kubectl logs -l app=blob-reader -n blob-storage --tail=20

# Exec into a writer pod and check shared data
$WRITER_POD = kubectl get pods -l app=blob-writer -n blob-storage -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $WRITER_POD -n blob-storage -- /bin/sh -c "
echo 'Checking shared storage...'
ls -la /data/
echo ''
echo 'Shared data content:'
cat /data/shared/shared-data.txt | tail -10
echo ''
echo 'Log files:'
ls -la /data/logs/
"

# Exec into a reader pod and verify it sees the same data
$READER_POD = kubectl get pods -l app=blob-reader -n blob-storage -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $READER_POD -n blob-storage -- /bin/sh -c "
echo 'Reader view of shared storage...'
ls -la /data/
echo ''
echo 'Latest shared data:'
tail -5 /data/shared/shared-data.txt
"
```

### Step 8: Performance Testing

Test blob storage performance:

```powershell
# Create a performance test pod  
@"
apiVersion: v1
kind: Pod
metadata:
  name: storage-performance-test
  namespace: blob-storage
spec:
  containers:
  - name: perf-test
    image: busybox:1.35
    command: ["sleep", "3600"]
    volumeMounts:
    - name: blob-fuse-storage
      mountPath: /test/fuse
  volumes:
  - name: blob-fuse-storage
    persistentVolumeClaim:
      claimName: blob-fuse-pvc
  restartPolicy: Never
"@ | kubectl apply -f -

# Wait for test pod
kubectl wait --for=condition=Ready pod/storage-performance-test -n blob-storage --timeout=300s

# Test write performance on Fuse
echo "Testing Fuse write performance..."
kubectl exec storage-performance-test -n blob-storage -- /bin/sh -c "
cd /test/fuse
echo 'Testing Fuse write performance...'
time dd if=/dev/zero of=testfile-fuse bs=1M count=50
ls -lh testfile-fuse
"

# Test read performance
echo "Testing read performance..."
kubectl exec storage-performance-test -n blob-storage -- /bin/sh -c "
echo 'Fuse read performance:'
time dd if=/test/fuse/testfile-fuse of=/dev/null bs=1M
"
```

### Step 9: Verify Data in Azure Portal

First, assign yourself the necessary role to access blob data:

```powershell
# Refresh Azure CLI login if needed
az login --use-device-code

# Get your object ID directly 
$CURRENT_USER_OBJECT_ID = az ad signed-in-user show --query id --output tsv
echo "Current user object ID: $CURRENT_USER_OBJECT_ID"

# Assign Storage Blob Data Contributor role using object ID
az role assignment create `
    --role "Storage Blob Data Contributor" `
    --assignee-object-id $CURRENT_USER_OBJECT_ID `
    --assignee-principal-type User `
    --scope "/subscriptions/$(az account show --query id --output tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT"

echo "Role assignment completed. Waiting 30 seconds for propagation..."
Start-Sleep -Seconds 30
```

**Alternative approach using storage account key:**

```powershell
# If RBAC assignment fails, use storage account key
$STORAGE_KEY = az storage account keys list `
    --resource-group $RESOURCE_GROUP `
    --account-name $STORAGE_ACCOUNT `
    --query '[0].value' -o tsv
```

Now check that data is actually stored in Azure Blob Storage:

```powershell
# Option 1: Using Azure AD authentication (if role assignment succeeded)
az storage blob list `
    --container-name $CONTAINER_NAME `
    --account-name $STORAGE_ACCOUNT `
    --auth-mode login `
    --output table

# Option 2: Using storage account key (if RBAC approach failed)
az storage blob list `
    --container-name $CONTAINER_NAME `
    --account-name $STORAGE_ACCOUNT `
    --account-key $STORAGE_KEY `
    --output table

# Download a file to verify content (using appropriate auth method)
# With Azure AD:
az storage blob download `
    --container-name $CONTAINER_NAME `
    --name "shared/shared-data.txt" `
    --file "./downloaded-shared-data.txt" `
    --account-name $STORAGE_ACCOUNT `
    --auth-mode login

# OR with storage key:
# az storage blob download `
#     --container-name $CONTAINER_NAME `
#     --name "shared/shared-data.txt" `
#     --file "./downloaded-shared-data.txt" `
#     --account-name $STORAGE_ACCOUNT `
#     --account-key $STORAGE_KEY

# Display the content
Get-Content ./downloaded-shared-data.txt  | Select-Object -Last 10
# Get container properties using Azure AD auth
az storage container show `
    --name $CONTAINER_NAME `
    --account-name $STORAGE_ACCOUNT `
    --auth-mode login
```

### Step 10: Test Cross-Pod Data Sharing

Demonstrate that data written by one pod is immediately visible to others:

```powershell
# Write unique data from a specific pod
$WRITER_POD = kubectl get pods -l app=blob-writer -n blob-storage -o jsonpath='{.items[0].metadata.name}'
$UNIQUE_ID = Get-Random -Minimum 100000 -Maximum 999999

kubectl exec $WRITER_POD -n blob-storage -- /bin/sh -c "
echo 'Cross-pod test data - ID: $UNIQUE_ID - Written by: $WRITER_POD' > /data/cross-pod-test.txt
echo 'Data written with unique ID: $UNIQUE_ID'
"

# Immediately read from a different pod
$READER_POD = kubectl get pods -l app=blob-reader -n blob-storage -o jsonpath='{.items[0].metadata.name}'
kubectl exec $READER_POD -n blob-storage -- /bin/sh -c "
echo 'Reading from: $READER_POD'
cat /data/cross-pod-test.txt
"
```

## Verification and Validation

### Check All Resources

```powershell
# List all blob storage resources
kubectl get all -n blob-storage

# Check storage classes
kubectl get storageclass | Select-String blob

# Check PVCs and PVs
kubectl get pvc,pv -n blob-storage

# Check CSI driver components
kubectl get pods -n kube-system | Select-String blob
```

### Monitor Storage Usage

```powershell
# Check disk usage in pods
kubectl exec $WRITER_POD -n blob-storage -- df -h /data
kubectl exec storage-performance-test -n blob-storage -- df -h /test/fuse

# Check storage account usage
az storage account show-usage `
    --name $STORAGE_ACCOUNT `
    --query 'currentValue' `
    --output tsv
```

## Expected Results

After completing this lab, you should have:
- ✅ Azure Blob CSI driver enabled and running
- ✅ Azure Storage Account and blob container created
- ✅ Multiple applications sharing blob storage via Fuse
- ✅ NFS-based blob storage working with web application
- ✅ Cross-pod data sharing demonstrated
- ✅ Performance characteristics observed
- ✅ Data persistence verified in Azure Portal

## Troubleshooting

### Common Issues

1. **Blob CSI Driver Not Available**
   ```powershell
   # Check driver installation
   kubectl get csidriver
   kubectl get pods -n kube-system -l app=csi-blob-controller
   
   # Reinstall if needed
   az aks update --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --enable-blob-driver
   ```

2. **Mount Failures with Managed Identity**
   ```powershell
   # Check pod events
   kubectl describe pod <pod-name> -n blob-storage
   
   # Verify managed identity permissions
   $CLUSTER_IDENTITY = az aks show --resource-group $RESOURCE_GROUP --name aks-volumes-cluster --query identityProfile.kubeletidentity.clientId --output tsv
   az role assignment list --assignee $CLUSTER_IDENTITY --scope "/subscriptions/$(az account show --query id --output tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCOUNT"
   
   # Check storage account access
   az storage account show --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --query 'allowSharedKeyAccess'
   
   # Verify secret format
   kubectl get secret azure-storage-secret -n blob-storage -o yaml
   ```

3. **PVC Stuck in Pending**
   ```powershell
   # Check CSI driver logs
   kubectl logs -n kube-system -l app=csi-blob-controller --tail=50
   
   # Check storage class parameters
   kubectl describe storageclass blob-fuse
   
   # Verify container exists
   az storage container show --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT --auth-mode login
   ```

4. **Performance Issues**
   ```powershell
   # Check mount options
   kubectl describe pv <pv-name>
   
   # Check storage account tier
   az storage account show --name $STORAGE_ACCOUNT --query 'accessTier'
   ```

## Use Cases for Blob Storage

### When to Use Blob Storage CSI Driver:

1. **Shared File Storage**: Multiple pods need access to the same files
2. **Large File Processing**: Processing large media files, logs, or datasets
3. **Content Distribution**: Static web content, downloads, etc.
4. **Backup and Archive**: Long-term storage of application data
5. **Data Analytics**: Input/output for big data processing jobs

### Performance Considerations:

- **Fuse**: Better for small files, POSIX compliance
- **NFS**: Better for large files, higher throughput
- **Premium Storage**: Better performance, higher cost
- **Standard Storage**: Lower cost, adequate for most workloads

## Clean Up

Clean up resources created in this lab:

```powershell
# Delete Kubernetes resources
kubectl delete namespace blob-storage --force --grace-period=0

# Delete storage classes
kubectl delete storageclass blob-fuse

# Delete Azure storage account
az storage account delete `
    --name $STORAGE_ACCOUNT `
    --resource-group $RESOURCE_GROUP `
    --yes

# Clean up downloaded files
Remove-Item -Force ./downloaded-shared-data.txt -ErrorAction SilentlyContinue
```

## Next Steps

You have now completed all four labs! You should have a comprehensive understanding of:
- AKS cluster setup and configuration
- Dynamic volume provisioning with CSI drivers
- StatefulSet applications with read replicas
- Blob storage integration for shared scenarios

## Additional Resources

- [Azure Blob CSI Driver Documentation](https://docs.microsoft.com/en-us/azure/aks/azure-blob-csi)
- [Azure Blob Storage Documentation](https://docs.microsoft.com/en-us/azure/storage/blobs/)
- [Kubernetes CSI Driver Development](https://kubernetes-csi.github.io/docs/)
- [Azure Storage Performance Guidelines](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-performance-checklist)