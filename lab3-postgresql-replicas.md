# Lab 3: PostgreSQL with Azure Disk and Read Replicas

## Overview
This lab demonstrates deploying PostgreSQL with persistent storage using Azure Disk, setting up read replicas across multiple nodes, and testing failover scenarios. You'll learn about StatefulSets, headless services, and high-availability patterns in Kubernetes.

## Learning Objectives
- Deploy PostgreSQL with persistent storage
- Configure PostgreSQL primary-replica architecture
- Implement read replicas on different nodes
- Test failover scenarios
- Understand StatefulSets and persistent storage patterns

## Prerequisites
- Completed Lab 1 (AKS cluster setup)
- Completed Lab 2 (Understanding CSI drivers and PVCs)
- kubectl configured and connected to your AKS cluster

## Lab Steps

### Step 1: Create Namespace and Storage Class

Create a dedicated namespace and storage class for PostgreSQL:

```bash
# Create namespace for PostgreSQL
kubectl create namespace postgresql

# Create optimized storage class for database workloads
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: postgresql-ssd
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  cachingmode: ReadWrite
  kind: Managed
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# Verify storage class
kubectl get storageclass postgresql-ssd
```

### Step 2: Create ConfigMap for PostgreSQL Configuration

Create configuration for PostgreSQL with replication settings:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgresql-config
  namespace: postgresql
data:
  postgresql.conf: |
    # Connection settings
    listen_addresses = '*'
    port = 5432
    max_connections = 100
    
    # Memory settings
    shared_buffers = 256MB
    effective_cache_size = 1GB
    work_mem = 4MB
    maintenance_work_mem = 64MB
    
    # Replication settings
    wal_level = replica
    max_wal_senders = 3
    max_replication_slots = 3
    hot_standby = on
    hot_standby_feedback = on
    
    # Logging
    log_destination = 'stderr'
    logging_collector = off
    log_min_messages = warning
    log_line_prefix = '%t [%p-%l] %q%u@%d '
    
    # Checkpoint settings
    checkpoint_timeout = 15min
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    
  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            trust
    host    all             all             ::1/128                 trust
    host    all             all             0.0.0.0/0               md5
    host    replication     replicator      0.0.0.0/0               md5
    
  init-user-db.sh: |
    #!/bin/bash
    set -e
    
    # Create replication user
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
        CREATE USER replicator REPLICATION LOGIN ENCRYPTED PASSWORD 'replicator123';
        CREATE DATABASE testdb;
        \\c testdb;
        CREATE TABLE IF NOT EXISTS sample_data (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        INSERT INTO sample_data (name) VALUES 
            ('Primary Node Data 1'),
            ('Primary Node Data 2'),
            ('Primary Node Data 3');
    EOSQL
EOF
```

### Step 3: Create Secrets for PostgreSQL

Create secrets for database passwords:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-secret
  namespace: postgresql
type: Opaque
data:
  postgres-password: cG9zdGdyZXMxMjM=  # postgres123
  replicator-password: cmVwbGljYXRvcjEyMw==  # replicator123
EOF
```

### Step 4: Create Headless Service

Create a headless service for PostgreSQL StatefulSet:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: postgresql-headless
  namespace: postgresql
  labels:
    app: postgresql
spec:
  clusterIP: None
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
    name: postgresql
  selector:
    app: postgresql
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql-primary
  namespace: postgresql
  labels:
    app: postgresql
    role: primary
spec:
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
    name: postgresql
  selector:
    app: postgresql
    role: primary
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql-replica
  namespace: postgresql
  labels:
    app: postgresql
    role: replica
spec:
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
    name: postgresql
  selector:
    app: postgresql
    role: replica
EOF
```

### Step 5: Deploy PostgreSQL Primary

Create the primary PostgreSQL instance:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-primary
  namespace: postgresql
spec:
  serviceName: postgresql-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
      role: primary
  template:
    metadata:
      labels:
        app: postgresql
        role: primary
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - postgresql
              topologyKey: kubernetes.io/hostname
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        fsGroupChangePolicy: "OnRootMismatch"
      initContainers:
      - name: init-postgresql-data
        image: postgres:14
        securityContext:
          runAsUser: 999
          runAsGroup: 999
        command:
        - /bin/bash
        - -c
        - |
          echo "Initializing PostgreSQL data directory..."
          
          # Always ensure the data directory exists with correct permissions
          mkdir -p /var/lib/postgresql/data/pgdata
          chmod 700 /var/lib/postgresql/data/pgdata
          
          if [ ! -f "/var/lib/postgresql/data/pgdata/PG_VERSION" ]; then
            echo "PostgreSQL not initialized, running initdb..."
            initdb -D /var/lib/postgresql/data/pgdata --auth-local=trust --auth-host=md5
            echo "PostgreSQL database initialized successfully"
            
            # Create replication user and test database
            echo "Starting temporary PostgreSQL to create users and database..."
            pg_ctl -D /var/lib/postgresql/data/pgdata -l /tmp/postgres.log start
            sleep 5
            
            # Set postgres user password
            psql -c "ALTER USER postgres WITH ENCRYPTED PASSWORD 'postgres123';"
            
            createuser -s replicator
            psql -c "ALTER USER replicator WITH ENCRYPTED PASSWORD 'replicator123';"
            psql -c "ALTER USER replicator WITH REPLICATION;"
            createdb testdb
            psql -d testdb -c "CREATE TABLE IF NOT EXISTS sample_data (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );"
            psql -d testdb -c "INSERT INTO sample_data (name) VALUES 
                ('Primary Node Data 1'),
                ('Primary Node Data 2'),
                ('Primary Node Data 3');"
            
            pg_ctl -D /var/lib/postgresql/data/pgdata stop
            echo "Database initialization completed"
          else
            echo "PostgreSQL already initialized, checking permissions..."
            # Ensure permissions are correct on restart
            chmod 700 /var/lib/postgresql/data/pgdata
            echo "Permissions verified"
          fi
        volumeMounts:
        - name: postgresql-data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      containers:
      - name: postgresql
        image: postgres:14
        ports:
        - containerPort: 5432
          name: postgresql
        securityContext:
          runAsUser: 999
          runAsGroup: 999
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        env:
        - name: POSTGRES_DB
          value: postgres
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        - name: PGUSER
          value: postgres
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgresql-data
          mountPath: /var/lib/postgresql/data
        - name: postgresql-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        - name: postgresql-config
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        command:
        - postgres
        - -c
        - config_file=/etc/postgresql/postgresql.conf
        - -c
        - hba_file=/etc/postgresql/pg_hba.conf
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U "postgres" -h 127.0.0.1 -p 5432
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 6
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U "postgres" -h 127.0.0.1 -p 5432
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 6
        resources:
          requests:
            memory: 512Mi
            cpu: 250m
          limits:
            memory: 1Gi
            cpu: 500m
      volumes:
      - name: postgresql-config
        configMap:
          name: postgresql-config
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: postgresql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: postgresql-ssd
      resources:
        requests:
          storage: 20Gi
EOF

# Wait for primary to be ready
kubectl wait --for=condition=Ready pod/postgresql-primary-0 -n postgresql --timeout=300s

# Check primary status
kubectl get pods -n postgresql -l role=primary
```

### Step 6: Initialize Primary Database

Connect to the primary and verify setup:

```bash
# Test connection to primary
kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -c "SELECT version();"

# Check replication user
kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -c "SELECT usename FROM pg_user WHERE userepl = true;"

# Check sample data
kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -d testdb -c "SELECT * FROM sample_data;"
```

### Step 7: Create PostgreSQL Replica

Create read replica instances:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-replica
  namespace: postgresql
spec:
  serviceName: postgresql-headless
  replicas: 2
  selector:
    matchLabels:
      app: postgresql
      role: replica
  template:
    metadata:
      labels:
        app: postgresql
        role: replica
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - postgresql
              topologyKey: kubernetes.io/hostname
      initContainers:
      - name: init-replica
        image: postgres:14
        securityContext:
          runAsUser: 999
          runAsGroup: 999
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        env:
        - name: PGUSER
          value: postgres
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        command:
        - /bin/bash
        - -c
        - |
          echo "Initializing replica from primary..."
          if [ -d "/var/lib/postgresql/data/pgdata" ] && [ "$(ls -A /var/lib/postgresql/data/pgdata)" ]; then
            echo "Data directory already exists, skipping initialization"
          else
            echo "Creating base backup from primary..."
            mkdir -p /var/lib/postgresql/data/pgdata
            chmod 700 /var/lib/postgresql/data/pgdata
            PGPASSWORD=replicator123 pg_basebackup -h postgresql-primary-0.postgresql-headless.postgresql.svc.cluster.local -D /var/lib/postgresql/data/pgdata -U replicator -W -v -P -R
            echo "Base backup completed"
          fi
        volumeMounts:
        - name: postgresql-data
          mountPath: /var/lib/postgresql/data
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
      - name: postgresql
        image: postgres:14
        ports:
        - containerPort: 5432
          name: postgresql
        securityContext:
          runAsUser: 999
          runAsGroup: 999
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        - name: PGUSER
          value: postgres
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgresql-data
          mountPath: /var/lib/postgresql/data
        - name: postgresql-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        - name: postgresql-config
          mountPath: /etc/postgresql/pg_hba.conf
          subPath: pg_hba.conf
        command:
        - postgres
        - -c
        - config_file=/etc/postgresql/postgresql.conf
        - -c
        - hba_file=/etc/postgresql/pg_hba.conf
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U "postgres" -h 127.0.0.1 -p 5432
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 6
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - exec pg_isready -U "postgres" -h 127.0.0.1 -p 5432
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 6
        resources:
          requests:
            memory: 512Mi
            cpu: 250m
          limits:
            memory: 1Gi
            cpu: 500m
      volumes:
      - name: postgresql-config
        configMap:
          name: postgresql-config
  volumeClaimTemplates:
  - metadata:
      name: postgresql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: postgresql-ssd
      resources:
        requests:
          storage: 20Gi
EOF

# Wait for replicas to be ready (this may take a few minutes)
kubectl wait --for=condition=Ready pod/postgresql-replica-0 -n postgresql --timeout=600s
kubectl wait --for=condition=Ready pod/postgresql-replica-1 -n postgresql --timeout=600s

# Check all pods
kubectl get pods -n postgresql -o wide
```

### Step 8: Verify Replication Setup

Test that replication is working properly:

```bash
# Check replication status on primary
kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -c "SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn FROM pg_stat_replication;"

# Understanding the replication status columns:
# - client_addr: IP address of the replica
# - state: streaming (active replication)
# - sent_lsn: Last LSN sent to replica
# - write_lsn: Last LSN written by replica to disk
# - flush_lsn: Last LSN flushed to disk by replica
# - replay_lsn: Last LSN applied by replica
# All LSNs should be close to each other for healthy replication

# Check replica status
kubectl exec -it postgresql-replica-0 -n postgresql -- psql -U postgres -c "SELECT pg_is_in_recovery();"
kubectl exec -it postgresql-replica-1 -n postgresql -- psql -U postgres -c "SELECT pg_is_in_recovery();"

# Verify data is replicated
kubectl exec -it postgresql-replica-0 -n postgresql -- psql -U postgres -d testdb -c "SELECT * FROM sample_data;"

# Add data to primary and verify it appears on replicas
kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -d testdb -c "INSERT INTO sample_data (name) VALUES ('Replication Test $(date)');"

# Check on replicas (may take a moment)
sleep 5
kubectl exec -it postgresql-replica-0 -n postgresql -- psql -U postgres -d testdb -c "SELECT * FROM sample_data ORDER BY id DESC LIMIT 1;"
kubectl exec -it postgresql-replica-1 -n postgresql -- psql -U postgres -d testdb -c "SELECT * FROM sample_data ORDER BY id DESC LIMIT 1;"
```

### Step 9: Test Read/Write Operations

Create a test client to demonstrate read/write splitting:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: postgresql-client
  namespace: postgresql
spec:
  containers:
  - name: client
    image: postgres:14
    command: ["sleep", "3600"]
    env:
    - name: PGPASSWORD
      valueFrom:
        secretKeyRef:
          name: postgresql-secret
          key: postgres-password
    - name: PGUSER
      value: postgres
  restartPolicy: Never
EOF

# Wait for client pod
kubectl wait --for=condition=Ready pod/postgresql-client -n postgresql --timeout=60s

# Test write operations (should only work on primary)
kubectl exec -it postgresql-client -n postgresql -- psql -h postgresql-primary -U postgres -d testdb -c "INSERT INTO sample_data (name) VALUES ('Write test from client');"

# Test read operations on primary
kubectl exec -it postgresql-client -n postgresql -- psql -h postgresql-primary -U postgres -d testdb -c "SELECT COUNT(*) FROM sample_data;"

# Test read operations on replicas
kubectl exec -it postgresql-client -n postgresql -- psql -h postgresql-replica-0.postgresql-headless -U postgres -d testdb -c "SELECT COUNT(*) FROM sample_data;"
kubectl exec -it postgresql-client -n postgresql -- psql -h postgresql-replica-1.postgresql-headless -U postgres -d testdb -c "SELECT COUNT(*) FROM sample_data;"

# Try write on replica (should fail)
kubectl exec -it postgresql-client -n postgresql -- psql -h postgresql-replica-0.postgresql-headless -U postgres -d testdb -c "INSERT INTO sample_data (name) VALUES ('This should fail');" || echo "Expected: Write operation failed on replica"
```

### Step 10: Test Failover Scenario

Simulate primary failure and promote a replica:

```bash
# First, add some data to track
kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -d testdb -c "INSERT INTO sample_data (name) VALUES ('Before failover test');"

# Check current LSN on primary
kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -c "SELECT pg_current_wal_lsn();"

# LSN (Log Sequence Number) explained:
# LSN is a unique identifier that tracks position in PostgreSQL's Write-Ahead Log (WAL)
# It helps monitor replication progress and measure lag between primary and replicas
# Format: segment/offset (e.g., 0/1A2B3C4D)
# Higher LSN = more recent data

# Simulate primary failure by scaling down
kubectl scale statefulset postgresql-primary --replicas=0 -n postgresql

# Wait for primary to be terminated
kubectl wait --for=delete pod/postgresql-primary-0 -n postgresql --timeout=120s

# Check replica status
kubectl get pods -n postgresql

# Promote replica-0 to primary (manual promotion simulation)
kubectl exec -it postgresql-replica-0 -n postgresql -- psql -U postgres -c "SELECT pg_promote();" || echo "Promotion command may vary by PostgreSQL version"

# Alternative promotion method if above doesn't work
kubectl exec -it postgresql-replica-0 -n postgresql -- bash -c "pg_ctl promote -D /var/lib/postgresql/data/pgdata"

# Verify the promoted replica can accept writes
sleep 10
kubectl exec -it postgresql-client -n postgresql -- psql -h postgresql-replica-0.postgresql-headless -U postgres -d testdb -c "INSERT INTO sample_data (name) VALUES ('After failover test');"

# Check data consistency
kubectl exec -it postgresql-replica-0 -n postgresql -- psql -U postgres -d testdb -c "SELECT * FROM sample_data ORDER BY id DESC LIMIT 3;"
```

### Step 11: Restore Original Configuration

Restore the original primary and re-establish replication:

```bash
# Scale primary back up
kubectl scale statefulset postgresql-primary --replicas=1 -n postgresql

# Wait for primary to be ready
kubectl wait --for=condition=Ready pod/postgresql-primary-0 -n postgresql --timeout=300s

# The original primary will need to be reinitialized as it's now behind
# In a production scenario, you would need to rebuild it from the new primary
echo "Note: In production, you would need to rebuild the original primary from the promoted replica"

# For this lab, we'll keep the promoted replica as the active primary
# Check final status
kubectl get pods -n postgresql -o wide
kubectl get pvc -n postgresql
```

## Verification and Validation

### Check All Resources

```bash
# Check all PostgreSQL resources
kubectl get all -n postgresql

# Check storage usage
kubectl exec -it postgresql-replica-0 -n postgresql -- df -h /var/lib/postgresql/data

# Check replication lag
kubectl exec -it postgresql-replica-1 -n postgresql -- psql -U postgres -c "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS replication_lag_seconds;"

# Verify node distribution
kubectl get pods -n postgresql -o wide | grep postgresql
```

### Performance Testing

```bash
# Test database performance
kubectl exec -it postgresql-client -n postgresql -- pgbench -h postgresql-replica-0.postgresql-headless -U postgres -d testdb -i

# Run simple benchmark
kubectl exec -it postgresql-client -n postgresql -- pgbench -h postgresql-replica-0.postgresql-headless -U postgres -d testdb -c 5 -j 2 -t 100
```

## Expected Results

After completing this lab, you should have:
- ✅ PostgreSQL primary instance running with persistent storage
- ✅ Two PostgreSQL read replicas on different nodes
- ✅ Working replication between primary and replicas
- ✅ Read-only access working on replicas
- ✅ Write operations working only on primary
- ✅ Successful failover simulation
- ✅ Data persistence across all scenarios

## Troubleshooting

### Common Issues

1. **PostgreSQL "root execution not permitted" Error**
   ```bash
   # This error occurs when PostgreSQL tries to run as root
   # The fix is to ensure proper securityContext is set
   
   # Check the pod security context
   kubectl describe pod postgresql-primary-0 -n postgresql | grep -A 10 "Security Context"
   
   # Verify the container is running as postgres user (UID 999)
   kubectl exec postgresql-primary-0 -n postgresql -- id
   
   # If you see this error, delete and recreate the StatefulSet with proper security context
   kubectl delete statefulset postgresql-primary -n postgresql
   # Then reapply the corrected StatefulSet configuration
   ```

2. **PostgreSQL "could not access directory" Error**
   ```bash
   # This happens when the data directory doesn't exist or has wrong permissions
   # Best solution for lab: Delete PVC and start fresh
   
   kubectl delete statefulset postgresql-primary -n postgresql
   kubectl delete pvc postgresql-data-postgresql-primary-0 -n postgresql
   
   # Wait for resources to be cleaned up
   kubectl get pvc -n postgresql
   
   # Then reapply the StatefulSet - it will initialize properly
   ```

3. **Replica Initialization Failure**
   ```bash
   # Check init container logs
   kubectl logs postgresql-replica-0 -n postgresql -c init-replica
   
   # Check connectivity to primary
   kubectl exec -it postgresql-replica-0 -n postgresql -- pg_isready -h postgresql-primary-0.postgresql-headless.postgresql.svc.cluster.local
   ```

2. **Replication Lag**
   ```bash
   # Check replication status
   kubectl exec -it postgresql-primary-0 -n postgresql -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"
   
   # Check replica logs
   kubectl logs postgresql-replica-0 -n postgresql
   ```

3. **Connection Issues**
   ```bash
   # Test DNS resolution
   kubectl exec -it postgresql-client -n postgresql -- nslookup postgresql-headless.postgresql.svc.cluster.local
   
   # Check service endpoints
   kubectl get endpoints -n postgresql
   ```

## Clean Up

Clean up resources created in this lab:

```bash
# Delete all PostgreSQL resources
kubectl delete namespace postgresql --force --grace-period=0

# Check that PVs are deleted (or manually delete if retained)
kubectl get pv | grep postgresql

# Delete storage class
kubectl delete storageclass postgresql-ssd
```

## Next Steps

Proceed to [Lab 4: Blob Storage with CSI Driver](./lab4-blob-storage.md) to learn about using Azure Blob Storage for different storage scenarios.

## Additional Resources

- [PostgreSQL on Kubernetes](https://postgres-operator.readthedocs.io/)
- [PostgreSQL Replication Documentation](https://www.postgresql.org/docs/current/high-availability.html)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)