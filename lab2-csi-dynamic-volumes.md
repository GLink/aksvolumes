# Lab 2: CSI Driver and Dynamic Provisioning

## Overview
This lab demonstrates how to use Container Storage Interface (CSI) drivers for dynamic volume provisioning in AKS. You'll work with storage classes, persistent volume claims (PVCs), and deploy a pod that uses persistent storage.

## Learning Objectives
- Understand CSI drivers and storage classes
- Create custom storage classes
- Use Persistent Volume Claims (PVCs)
- Deploy pods with persistent storage
- Verify data persistence across pod restarts

## Prerequisites
- Completed Lab 1 (AKS cluster setup)
- kubectl configured and connected to your AKS cluster

## Lab Steps

### Step 1: Examine Default Storage Classes

First, let's examine the default storage classes available in AKS:

```bash
# List available storage classes
kubectl get storageclass

# Get detailed information about default storage class
kubectl describe storageclass default

# Check the managed-csi storage class
kubectl describe storageclass managed-csi
```

You should see storage classes like:
- `default` - Standard Azure Disk
- `managed-csi` - CSI-based Azure Disk
- `azurefile-csi` - Azure File CSI driver
- `azureblob-nfs-premium` - Azure Blob NFS (if enabled)

### Step 2: Create a Custom Storage Class

Create a custom storage class with specific performance characteristics:

```yaml
# Create custom-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingmode: ReadOnly
  kind: Managed
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
mountOptions:
  - debug
```

Apply the storage class:

```bash
# Create the storage class
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingmode: ReadOnly
  kind: Managed
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
mountOptions:
  - debug
EOF

# Verify storage class creation
kubectl get storageclass fast-ssd-retain -o yaml
```

### Step 3: Create a Persistent Volume Claim

Create a PVC that uses the custom storage class:

```bash
# Create PVC with custom storage class
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc-premium
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd-retain
  resources:
    requests:
      storage: 10Gi
EOF

# Check PVC status (should be Pending until pod is created)
kubectl get pvc my-pvc-premium
kubectl describe pvc my-pvc-premium
```

### Step 4: Create a Second PVC with Default Storage Class

Create another PVC using the default storage class for comparison:

```bash
# Create PVC with default storage class
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc-standard
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 5Gi
EOF

# Check both PVCs
kubectl get pvc
```

### Step 5: Deploy a Pod with Persistent Storage

Create a pod that uses both PVCs to demonstrate persistent storage:

```bash
# Create pod with persistent volumes
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo-pod
spec:
  containers:
  - name: app-container
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: premium-storage
      mountPath: /data/premium
    - name: standard-storage
      mountPath: /data/standard
    - name: config-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: premium-storage
    persistentVolumeClaim:
      claimName: my-pvc-premium
  - name: standard-storage
    persistentVolumeClaim:
      claimName: my-pvc-standard
  - name: config-volume
    configMap:
      name: nginx-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>AKS Storage Demo</title>
    </head>
    <body>
        <h1>AKS Persistent Storage Demo</h1>
        <p>This page is served from a ConfigMap volume.</p>
        <p>Persistent data is stored in mounted volumes.</p>
    </body>
    </html>
EOF

# Wait for pod to be running
kubectl get pods -w
```

### Step 6: Verify Persistent Volumes

Check that the persistent volumes were created:

```bash
# Check PVCs status (should now be Bound)
kubectl get pvc

# Check persistent volumes
kubectl get pv

# Get detailed information about the volumes
kubectl describe pv
```

### Step 7: Test Data Persistence

Write data to the persistent volumes and verify persistence:

```bash
# Write data to premium storage
kubectl exec storage-demo-pod -- bash -c "echo 'Premium storage data - $(date)' > /data/premium/test.txt"

# Write data to standard storage
kubectl exec storage-demo-pod -- bash -c "echo 'Standard storage data - $(date)' > /data/standard/test.txt"

# Create some additional test files
kubectl exec storage-demo-pod -- bash -c "for i in {1..5}; do echo 'File $i content - $(date)' > /data/premium/file$i.txt; done"

# Verify files were created
kubectl exec storage-demo-pod -- ls -la /data/premium/
kubectl exec storage-demo-pod -- ls -la /data/standard/

# Read the content
kubectl exec storage-demo-pod -- cat /data/premium/test.txt
kubectl exec storage-demo-pod -- cat /data/standard/test.txt
```

### Step 8: Test Pod Restart and Data Persistence

Delete and recreate the pod to verify data persistence:

```bash
# Delete the pod
kubectl delete pod storage-demo-pod

# Recreate the pod with the same configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo-pod-v2
spec:
  containers:
  - name: app-container
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: premium-storage
      mountPath: /data/premium
    - name: standard-storage
      mountPath: /data/standard
    - name: config-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: premium-storage
    persistentVolumeClaim:
      claimName: my-pvc-premium
  - name: standard-storage
    persistentVolumeClaim:
      claimName: my-pvc-standard
  - name: config-volume
    configMap:
      name: nginx-config
EOF

# Wait for new pod to be ready
kubectl wait --for=condition=Ready pod/storage-demo-pod-v2 --timeout=60s

# Verify data still exists
kubectl exec storage-demo-pod-v2 -- cat /data/premium/test.txt
kubectl exec storage-demo-pod-v2 -- cat /data/standard/test.txt
kubectl exec storage-demo-pod-v2 -- ls -la /data/premium/
```

### Step 9: Volume Expansion (Optional)

Test volume expansion capabilities:

```bash
# Expand the premium PVC from 10Gi to 15Gi
kubectl patch pvc my-pvc-premium -p '{"spec":{"resources":{"requests":{"storage":"15Gi"}}}}'

# Check expansion status
kubectl get pvc my-pvc-premium -w

# Verify the expansion in the pod
kubectl exec storage-demo-pod-v2 -- df -h /data/premium
```

### Step 10: Performance Testing

Compare performance between storage classes:

```bash
# Test write performance on premium storage
kubectl exec storage-demo-pod-v2 -- bash -c "time dd if=/dev/zero of=/data/premium/testfile bs=1M count=100"

# Test write performance on standard storage
kubectl exec storage-demo-pod-v2 -- bash -c "time dd if=/dev/zero of=/data/standard/testfile bs=1M count=100"

# Test read performance
kubectl exec storage-demo-pod-v2 -- bash -c "time dd if=/data/premium/testfile of=/dev/null bs=1M"
kubectl exec storage-demo-pod-v2 -- bash -c "time dd if=/data/standard/testfile of=/dev/null bs=1M"

# Check disk usage
kubectl exec storage-demo-pod-v2 -- df -h
```

## Verification and Validation

### Check Storage Resources

```bash
# List all storage-related resources
kubectl get storageclass,pv,pvc,pods

# Get detailed information about storage usage
kubectl exec storage-demo-pod-v2 -- du -sh /data/*

# Check volume mounts in pod
kubectl describe pod storage-demo-pod-v2 | grep -A 20 "Mounts:"
```

### Verify CSI Driver Components

```bash
# Check CSI driver pods
kubectl get pods -n kube-system | grep csi

# Check CSI driver configuration
kubectl get csidriver

# Verify storage class parameters
kubectl get storageclass -o yaml
```

## Expected Results

After completing this lab, you should have:
- ✅ Custom storage class created with Premium SSD
- ✅ Two PVCs created with different storage classes
- ✅ Pod successfully using persistent volumes
- ✅ Data persisting across pod restarts
- ✅ Volume expansion working (if tested)
- ✅ Performance differences observed between storage classes

## Troubleshooting

### Common Issues

1. **PVC Stuck in Pending**
   ```bash
   # Check storage class
   kubectl describe storageclass <storage-class-name>
   
   # Check events
   kubectl get events --sort-by=.metadata.creationTimestamp
   ```

2. **Pod Cannot Mount Volume**
   ```bash
   # Check pod events
   kubectl describe pod <pod-name>
   
   # Check PVC status
   kubectl describe pvc <pvc-name>
   ```

3. **Volume Expansion Failed**
   ```bash
   # Check if storage class supports expansion
   kubectl get storageclass <storage-class-name> -o yaml | grep allowVolumeExpansion
   
   # Check expansion events
   kubectl describe pvc <pvc-name>
   ```

## Understanding Storage Classes

### Key Parameters for Azure Disk CSI Driver:

- **skuName**: Storage type (Standard_LRS, Premium_LRS, etc.)
- **cachingmode**: ReadOnly, ReadWrite, None
- **kind**: Managed, Shared, Dedicated
- **reclaimPolicy**: Delete, Retain
- **allowVolumeExpansion**: true/false
- **volumeBindingMode**: Immediate, WaitForFirstConsumer

## Clean Up

Clean up resources created in this lab:

```bash
# Delete pods
kubectl delete pod storage-demo-pod-v2 --force --grace-period=0

# Delete ConfigMap
kubectl delete configmap nginx-config

# Delete PVCs (this will also delete PVs with Delete reclaim policy)
kubectl delete pvc my-pvc-standard

# For retained volumes, delete PVC first, then manually delete PV
kubectl delete pvc my-pvc-premium
kubectl get pv
# kubectl delete pv <pv-name>  # if retention policy was used

# Delete custom storage class
kubectl delete storageclass fast-ssd-retain
```

## Next Steps

Proceed to [Lab 3: PostgreSQL with Azure Disk and Read Replicas](./lab3-postgresql-replicas.md) to learn about running stateful applications with persistent storage.

## Additional Resources

- [Azure Disk CSI Driver Documentation](https://docs.microsoft.com/en-us/azure/aks/azure-disk-csi)
- [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [CSI Driver Specifications](https://kubernetes-csi.github.io/docs/)