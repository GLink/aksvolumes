# Lab: AKS Fundamentals - Deployments, Services & Configuration

## Overview

In this hands-on lab you will learn the core Kubernetes concepts on Azure Kubernetes Service (AKS):

1. **Deployments** – Creating, updating, rolling back, and managing deployment strategies
2. **Services** – Exposing workloads via ClusterIP, NodePort, and Internal LoadBalancer
3. **Blue/Green Deployments** – Zero-downtime releases by switching traffic between versions
4. **Canary Deployments** – Gradually rolling out a new version alongside the existing one
5. **ConfigMaps** – Externalising configuration from your containers
6. **Secrets** – Storing sensitive data securely

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Azure Subscription** | With permissions to create/manage AKS clusters |
| **PowerShell 7+** | Windows PowerShell or PowerShell Core |
| **Azure CLI** | Install from https://aka.ms/installazurecli |
| **kubectl** | Installed via `az aks install-cli` |
| **AKS Cluster** | A running cluster with at least 2 Linux nodes |

### Environment Setup (PowerShell)

```powershell
# Login to Azure
az login

# Set your subscription (replace with your subscription id)
az account set --subscription "<your-subscription-id>"

# Get credentials for your AKS cluster
az aks get-credentials --resource-group <your-rg> --name <your-aks-cluster>

# Verify connectivity
kubectl get nodes
```

> **Note:** All commands in this lab use PowerShell and `kubectl`. Ensure your kubeconfig is pointing to the correct cluster before proceeding.

---

## Exercise 1: Basic Deployments

### Concepts

A **Deployment** provides declarative updates for Pods and ReplicaSets. When you change a Deployment's pod template, Kubernetes creates a new ReplicaSet and gradually migrates pods from the old ReplicaSet to the new one (rolling update by default).

Key concepts covered:
- Creating a Deployment
- Triggering a rolling update by changing the pod template
- Using `minReadySeconds` to slow down rollouts
- Rolling back with `kubectl rollout undo`
- Deployment strategies: **RollingUpdate** vs **Recreate**
- Revision history limits

### Step 1.1 – Create Your First Deployment

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-1-dep
spec:
  replicas: 6
  selector:
    matchLabels:
      app: workload-1
  template:
    metadata:
      labels:
        app: workload-1
        color: lime
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.23
          ports:
            - containerPort: 80
"@ | kubectl apply -f -
```

Verify the deployment:

```powershell
kubectl get deployment workload-1-dep
kubectl get replicasets -l app=workload-1
kubectl get pods -l app=workload-1
```

> **Explanation:** This creates a Deployment with 6 replicas running nginx:1.23. The `selector.matchLabels` links the Deployment to its pods. The `nodeSelector` ensures pods land on Linux nodes only.

### Step 1.2 – Trigger a Rolling Update

Modify the label to simulate a template change:

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-1-dep
spec:
  replicas: 6
  selector:
    matchLabels:
      app: workload-1
  template:
    metadata:
      labels:
        app: workload-1
        color: yellow
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.23
          ports:
            - containerPort: 80
"@ | kubectl apply -f -
```

Watch the rollout:

```powershell
kubectl rollout status deployment/workload-1-dep
kubectl get replicasets -l app=workload-1
```

> **Explanation:** Changing any field in `spec.template` (here the `color` label) triggers a new ReplicaSet. The old ReplicaSet scales down while the new one scales up.

### Step 1.3 – Add minReadySeconds

Add `minReadySeconds: 15` and change the color label to `maroon`:

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-1-dep
spec:
  replicas: 6
  minReadySeconds: 15
  selector:
    matchLabels:
      app: workload-1
  template:
    metadata:
      labels:
        app: workload-1
        color: maroon
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.23
          ports:
            - containerPort: 80
"@ | kubectl apply -f -
```

```powershell
kubectl rollout status deployment/workload-1-dep
```

> **Explanation:** `minReadySeconds` specifies how long a new pod must be ready (without crashing) before it's considered available. This slows down the rollout giving you time to observe issues.

### Step 1.4 – Rollback a Deployment

```powershell
# View rollout history
kubectl rollout history deployment/workload-1-dep

# Undo to the previous version
kubectl rollout undo deployment/workload-1-dep

# Verify
kubectl get replicasets -l app=workload-1
```

> **Explanation:** `rollout undo` reverts to the previous ReplicaSet. Kubernetes doesn't delete old ReplicaSets by default, so it can quickly switch back.

### Step 1.5 – Rollback to a Specific Revision

```powershell
kubectl rollout undo deployment/workload-1-dep --to-revision=1
```

> **Explanation:** You can target any revision in history. Use `kubectl rollout history` to see available revisions.

### Step 1.6 – Deploy with an Invalid Image

Deploy with an image that doesn't exist:

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-1-dep
spec:
  replicas: 6
  selector:
    matchLabels:
      app: workload-1
  template:
    metadata:
      labels:
        app: workload-1
        color: aqua
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.12345
          ports:
            - containerPort: 80
"@ | kubectl apply -f -
```

```powershell
kubectl rollout status deployment/workload-1-dep
```

Watch the pods fail:

```powershell
kubectl get pods -l app=workload-1
```

You'll see pods in `ImagePullBackOff` or `ErrImagePull` status. The old ReplicaSet remains because the new pods never become ready.

Recover:

```powershell
kubectl rollout undo deployment/workload-1-dep
```

> **Explanation:** Because the broken pods never become ready, the rolling update stalls. The previous ReplicaSet keeps running, protecting your application. This is why rolling updates are the default strategy.

### Step 1.7 – Recreate Strategy

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-1-dep
spec:
  replicas: 6
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: workload-1
  template:
    metadata:
      labels:
        app: workload-1
        color: blue
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.21
          ports:
            - containerPort: 80
"@ | kubectl apply -f -
```

```powershell
kubectl get pods -l app=workload-1 --watch
```

> **Explanation:** The `Recreate` strategy kills ALL existing pods before creating new ones. This causes downtime but avoids running two versions simultaneously – useful when your app can't handle two versions at once (e.g., database schema migrations).

### Step 1.8 – Revision History Limit

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-1-dep
spec:
  replicas: 6
  revisionHistoryLimit: 2
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: workload-1
  template:
    metadata:
      labels:
        app: workload-1
        color: orange
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.23
          ports:
            - containerPort: 80
"@ | kubectl apply -f -
```

```powershell
kubectl get replicasets -l app=workload-1
```

> **Explanation:** `revisionHistoryLimit: 2` tells Kubernetes to keep only the 2 most recent ReplicaSets. Older ones are garbage-collected. This saves etcd storage in clusters with frequent deployments.

### Cleanup Exercise 1

```powershell
kubectl delete deployment workload-1-dep
```

---

## Exercise 2: Services

### Concepts

A **Service** is an abstraction that defines a logical set of Pods and a policy for accessing them. Services use label selectors to find their target pods. There are three main types:

| Type | Description |
|---|---|
| **ClusterIP** | Internal-only IP, reachable only within the cluster |
| **NodePort** | Exposes the service on each node's IP at a static port |
| **LoadBalancer** | Provisions an Azure load balancer (internal in our case) |

### Step 2.1 – Create the Workload

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-dep
spec:
  replicas: 6
  selector:
    matchLabels:
      app: nginx-1
      release: prod
  template:
    metadata:
      labels:
        app: nginx-1
        release: prod
        color: black
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: servicedemo
        image: scubakiz/servicedemo:1.0
        ports:
        - containerPort: 80
          protocol: TCP
        imagePullPolicy: Always
        env:
        - name: IMAGE_COLOR
          value: black
        - name: NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
"@ | kubectl apply -f -
```

> **Explanation:** This demo app (`scubakiz/servicedemo`) displays pod info (IP, name, node) in a web page, making it easy to see which pod handles each request.

### Step 2.2 – ClusterIP Service

```powershell
@"
apiVersion: v1
kind: Service
metadata:
  name: workload-svc
spec:
  ports:
    - port: 8100
      targetPort: 80
      name: web
  selector:
    app: nginx-1
    release: prod
  type: ClusterIP
"@ | kubectl apply -f -
```

```powershell
kubectl get svc workload-svc
```

Test it from inside the cluster:

```powershell
# Create a temporary pod to curl the service
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://workload-svc:8100
```

> **Explanation:** `ClusterIP` is the default service type. It gives the service an internal cluster IP. It's only reachable from within the cluster – other pods can access it via the service name (DNS) or the cluster IP.

### Step 2.3 – NodePort Service

```powershell
@"
apiVersion: v1
kind: Service
metadata:
  name: workload-svc
spec:
  ports:
    - port: 8100
      targetPort: 80
      name: web
  selector:
    app: nginx-1
    release: prod
  type: NodePort
"@ | kubectl apply -f -
```

```powershell
kubectl get svc workload-svc
```

> **Explanation:** `NodePort` exposes the service on a static port (30000-32767) on every node's IP. You can reach the service at `<any-node-ip>:<nodeport>`. In AKS, node IPs are typically only accessible within the VNet.

### Step 2.4 – Internal LoadBalancer Service

```powershell
@"
apiVersion: v1
kind: Service
metadata:
  name: workload-svc
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  ports:
    - port: 8100
      targetPort: 80
      name: web
  selector:
    app: nginx-1
    release: prod
  type: LoadBalancer
"@ | kubectl apply -f -
```

Wait for the internal IP to be assigned:

```powershell
kubectl get svc workload-svc --watch
```

Once the `EXTERNAL-IP` column shows a private IP (e.g., `10.x.x.x`), the internal load balancer is ready.

> **Explanation:** The annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` tells AKS to create an **internal** Azure Load Balancer instead of a public one. The assigned IP is a private IP within your VNet, making it accessible only from within the virtual network or peered networks. This is the required approach when public internet exposure is restricted.

Test from a pod in the cluster:

```powershell
# Replace <INTERNAL-IP> with the actual IP from kubectl get svc
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://<INTERNAL-IP>:8100
```

### Step 2.5 – Update Deployment and Observe Service Behaviour

Create the updated deployment (changes the color to red and adds `minReadySeconds`):

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-dep
spec:
  replicas: 6
  minReadySeconds: 20
  selector:
    matchLabels:
      app: nginx-1
      release: prod
  template:
    metadata:
      labels:
        app: nginx-1
        release: prod
        color: red
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: servicedemo
        image: scubakiz/servicedemo:1.0
        ports:
        - containerPort: 80
          protocol: TCP
        imagePullPolicy: Always
        env:
        - name: IMAGE_COLOR
          value: red
        - name: NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
"@ | kubectl apply -f -
```

While the rollout progresses, repeatedly curl the service:

```powershell
# From within a pod or jump box on the VNet
kubectl run curl-loop --image=curlimages/curl --rm -it --restart=Never -- sh -c "while true; do curl -s http://workload-svc:8100 | Select-String 'POD_IP'; Start-Sleep -Seconds 1; done"
```

> **Explanation:** As old pods terminate and new pods come online, the Service automatically routes traffic to whichever pods match its selector. You'll see responses alternating between old (black) and new (red) pods during the rollout.

### Cleanup Exercise 2

```powershell
kubectl delete deployment workload-dep
kubectl delete svc workload-svc
```

---

## Exercise 3: Blue/Green Deployments

### Concepts

A **Blue/Green deployment** runs two identical environments (blue = current, green = new). You switch traffic instantly by changing the Service selector. This gives zero-downtime deployments with instant rollback capability.

### Step 3.1 – Deploy the Blue Version

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-dep
spec:
  replicas: 3
  selector:
    matchLabels:
      target: blue-dep
  template:
    metadata:
      labels:
        target: blue-dep
        app: status
        color: blue
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: servicedemo
        image: scubakiz/servicedemo:1.0
        ports:
        - containerPort: 80
          protocol: TCP
        imagePullPolicy: Always
        env:
        - name: IMAGE_COLOR
          value: blue
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
"@ | kubectl apply -f -
```

Create the production service pointing to blue:

```powershell
@"
apiVersion: v1
kind: Service
metadata:
  name: production-svc
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  ports:
    - name: web
      port: 8080
      targetPort: 80
  selector:
    target: blue-dep
  type: LoadBalancer
"@ | kubectl apply -f -
```

```powershell
# Wait for internal IP
kubectl get svc production-svc --watch
```

Test the blue version:

```powershell
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://production-svc:8080
```

> **Explanation:** The service selector `target: blue-dep` routes all traffic to the blue deployment pods.

### Step 3.2 – Deploy the Green Version (Without Traffic)

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-dep
spec:
  replicas: 3
  selector:
    matchLabels:
      target: green-dep
  template:
    metadata:
      labels:
        target: green-dep
        app: status
        color: green
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: servicedemo
        image: scubakiz/servicedemo:1.0
        ports:
        - containerPort: 80
          protocol: TCP
        imagePullPolicy: Always
        env:
        - name: IMAGE_COLOR
          value: green
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
"@ | kubectl apply -f -
```

```powershell
kubectl get pods -l target=green-dep
```

> **Explanation:** The green deployment is now running, but receives NO traffic because the service selector still points to `target: blue-dep`.

### Step 3.3 – Switch Traffic to Green

Update the service selector:

```powershell
@"
apiVersion: v1
kind: Service
metadata:
  name: production-svc
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  ports:
    - name: web
      port: 8080
      targetPort: 80
  selector:
    target: green-dep
  type: LoadBalancer
"@ | kubectl apply -f -
```

Test – you should now see green:

```powershell
kubectl run curl-test2 --image=curlimages/curl --rm -it --restart=Never -- curl http://production-svc:8080
```

> **Explanation:** By changing the selector from `target: blue-dep` to `target: green-dep`, all NEW connections go to green pods. Existing TCP connections to blue pods complete naturally.

### Step 3.4 – Decommission Blue

Once satisfied with green:

```powershell
kubectl delete deployment blue-dep
```

> **Explanation:** To rollback, simply re-apply the service with `target: blue-dep` (assuming you haven't deleted the blue deployment yet). This is instant because the pods are already running.

### Cleanup Exercise 3

```powershell
kubectl delete deployment green-dep
kubectl delete svc production-svc
```

---

## Exercise 4: Canary Deployments

### Concepts

A **Canary deployment** runs a small number of new-version pods alongside the existing version. Both share the same Service selector, so traffic is distributed across both versions proportionally to the number of pods. This lets you test new versions with real traffic before fully rolling out.

### Step 4.1 – Deploy the Stable Version

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-1-dep
spec:
  replicas: 3
  selector:
    matchLabels:
      target: canary-pod
  template:
    metadata:
      labels:
        target: canary-pod
        version: "1.0"
        color: yellow
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: servicedemo
        image: scubakiz/servicedemo:1.0
        ports:
        - containerPort: 80
          protocol: TCP
        imagePullPolicy: Always
        env:
        - name: IMAGE_COLOR
          value: yellow
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
"@ | kubectl apply -f -
```

```powershell
@"
apiVersion: v1
kind: Service
metadata:
  name: canary-svc
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  ports:
    - name: web
      port: 8080
      targetPort: 80
  selector:
    target: canary-pod
  type: LoadBalancer
"@ | kubectl apply -f -
```

```powershell
# Wait for internal IP
kubectl get svc canary-svc --watch
```

> **Explanation:** The service selects ALL pods with `target: canary-pod`. Right now that's only the yellow (v1.0) pods.

### Step 4.2 – Deploy the Canary

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-2-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      target: canary-pod
      version: "2.0"
  template:
    metadata:
      labels:
        target: canary-pod
        version: "2.0"
        color: red
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: servicedemo
        image: scubakiz/servicedemo:1.0
        ports:
        - containerPort: 80
          protocol: TCP
        imagePullPolicy: Always
        env:
        - name: IMAGE_COLOR
          value: red
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
"@ | kubectl apply -f -
```

```powershell
kubectl get pods -l target=canary-pod
```

Test traffic distribution:

```powershell
# Run multiple requests and observe the color changing
kubectl run curl-loop --image=curlimages/curl --rm -it --restart=Never -- sh -c "for i in $(1..10); do curl -s http://canary-svc:8080 | grep IMAGE_COLOR; done"
```

> **Explanation:** With 3 yellow pods and 1 red pod, approximately 25% of traffic goes to the canary (red). If the canary looks healthy, scale it up and scale down the stable version. If it's broken, delete it – only 25% of users were affected.

### Step 4.3 – Promote or Rollback

**To promote** (canary is healthy):

```powershell
kubectl scale deployment canary-2-dep --replicas=3
kubectl delete deployment canary-1-dep
```

**To rollback** (canary is failing):

```powershell
kubectl delete deployment canary-2-dep
```

### Cleanup Exercise 4

```powershell
kubectl delete deployment canary-1-dep canary-2-dep
kubectl delete svc canary-svc
```

---

## Exercise 5: ConfigMaps

### Concepts

A **ConfigMap** stores non-sensitive configuration data as key-value pairs. Pods can consume ConfigMaps as:

1. **Environment variables** – individual keys or all keys at once
2. **Volume-mounted files** – each key becomes a file in the mounted directory

### Step 5.1 – Create ConfigMaps

```powershell
@"
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-configmap
  labels:
    scope: demo
data:
  MD_RABBITMQ_HOST: rabbit-svc
  MD_TOPIC: "notifications"
  MY_VALUE: "555"
"@ | kubectl apply -f -
```

```powershell
@"
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-configmap2
  labels:
    scope: demo
data:
  player_initial_lives: "23"
  game_status: READY
"@ | kubectl apply -f -
```

```powershell
@"
apiVersion: v1
kind: ConfigMap
metadata:
  name: file-configmap
  labels:
    scope: demo
data:
  app.config: |-
    {
      "settings":
      {
        "title": "Example Glossary",
        "option": "123"
      }
    }
  replace.sh: |-
    sed -i 's/123/456/g' app.config
"@ | kubectl apply -f -
```

Inspect:

```powershell
kubectl get configmap simple-configmap -o yaml
kubectl describe configmap file-configmap
```

### Step 5.2 – Consume ConfigMaps in a Pod

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-1-dep
  labels:
    scope: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: workload-1
  template:
    metadata:
      labels:
        app: workload-1
        color: lime
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.22
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: simple-configmap
          env:
            - name: MD_SERVICE_MAP_FILE
              value: /config/service-mappings.json
            - name: MD_ACKNOWLEDGE_HEARTBEAT
              value: "false"
            - name: PLAYER_INITIAL_LIVES
              valueFrom:
                configMapKeyRef:
                  name: simple-configmap2
                  key: player_initial_lives
          volumeMounts:
            - name: configmap-volume
              mountPath: /config-data
      volumes:
        - name: configmap-volume
          configMap:
            defaultMode: 0744
            name: file-configmap
"@ | kubectl apply -f -
```

### Step 5.3 – Verify ConfigMap Consumption

```powershell
# Get the pod name
$podName = kubectl get pods -l app=workload-1 -o jsonpath="{.items[0].metadata.name}"

# Check environment variables from ConfigMap
kubectl exec $podName -- env | Select-String "MD_RABBITMQ_HOST|MY_VALUE|PLAYER_INITIAL_LIVES|MD_TOPIC"

# Check mounted files and debug into the pod
kubectl exec $podName -- ls /config-data
kubectl exec $podName -- cat /config-data/app.config
kubectl exec $podName -it -- bash
```

> **Explanation:**
> - `envFrom.configMapRef` injects ALL keys from `simple-configmap` as environment variables
> - `env.valueFrom.configMapKeyRef` injects a SINGLE key (`player_initial_lives`) from `simple-configmap2`
> - The `volumes/volumeMounts` section mounts the `file-configmap` as files in `/config-data`

### Cleanup Exercise 5

```powershell
kubectl delete deployment -l scope=demo
kubectl delete configmap -l scope=demo
```

---

## Exercise 6: Secrets

### Concepts

A **Secret** is similar to a ConfigMap but designed for sensitive data (passwords, tokens, certificates). Key differences:
- Values are base64-encoded (not encrypted by default!)
- Kubernetes can restrict access to Secrets via RBAC
- Secrets are stored in etcd (ensure encryption at rest is enabled in production)
- Pods can consume them as environment variables or mounted files (just like ConfigMaps)

### Step 6.1 – Create Secrets

```powershell
@"
apiVersion: v1
kind: Secret
metadata:
  name: simple-secret
  labels:
    scope: demo
data:
  cert: RG9uJ3QgbG9vaywgSSdtIGEgc2VjcmV0
  key: dmFsdWU=
type: Opaque
"@ | kubectl apply -f -
```

> **Note:** The values are base64-encoded. You can encode with PowerShell:
> ```powershell
> [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes("my-secret-value"))
> ```

```powershell
@"
apiVersion: v1
kind: Secret
metadata:
  name: simple-secret2
  labels:
    scope: demo
data:
  dbpassword: cGFzc3dvcmQxMjM=
type: Opaque
"@ | kubectl apply -f -
```

```powershell
@"
apiVersion: v1
kind: Secret
metadata:
  name: file-secret
  labels:
    scope: demo
data:
  somevalue: VGhpcyBpcyBhIHNlY3JldCB2YWx1ZSBpbiBhIGZpbGU=
  anothervalue: VGhpcyBpcyBzb21lIG90aGVyIHN0cmluZyB0aGF0IEkgd2FudCB0byBtYWtlIGludG8gYSBzZWNyZXQ=
type: Opaque
"@ | kubectl apply -f -
```

Inspect (values are hidden by default):

```powershell
kubectl get secret simple-secret -o yaml
```

Decode a value:

```powershell
$encoded = kubectl get secret simple-secret -o jsonpath="{.data.cert}"
[System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($encoded))
```

### Step 6.2 – Consume Secrets in a Pod

```powershell
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload-2-dep
  labels:
    scope: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: workload-2
  template:
    metadata:
      labels:
        app: workload-2
        color: yellow
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: workload
          image: nginx:1.22
          ports:
            - containerPort: 80
          envFrom:
            - secretRef:
                name: simple-secret
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: simple-secret2
                  key: dbpassword
          volumeMounts:
            - name: secret-volume
              mountPath: /secret-data
      volumes:
        - name: secret-volume
          secret:
            secretName: file-secret
"@ | kubectl apply -f -
```

### Step 6.3 – Verify Secret Consumption

```powershell
# Get the pod name
$podName = kubectl get pods -l app=workload-2 -o jsonpath="{.items[0].metadata.name}"

# Check environment variables from Secret
kubectl exec $podName -- env | Select-String "cert|key|DB_PASSWORD"

# Check mounted secret files
kubectl exec $podName -- ls /secret-data
kubectl exec $podName -- cat /secret-data/somevalue
kubectl exec $podName -it -- bash
```

> **Explanation:**
> - `envFrom.secretRef` injects ALL keys from `simple-secret` as environment variables (automatically base64-decoded)
> - `env.valueFrom.secretKeyRef` injects the `dbpassword` key from `simple-secret2`
> - The volume mount makes `file-secret` keys available as files in `/secret-data`
>
> **Security tip:** In production, prefer volume-mounted secrets over environment variables. Environment variables can leak in logs, crash dumps, and child processes. Also consider Azure Key Vault with the CSI driver for additional security.

### Cleanup Exercise 6

```powershell
kubectl delete deployment -l scope=demo
kubectl delete secret -l scope=demo
```

---

## Full Lab Cleanup

Run this to remove all resources created during the lab:

```powershell
kubectl delete deployment workload-1-dep workload-dep blue-dep green-dep canary-1-dep canary-2-dep workload-2-dep 2>$null
kubectl delete svc workload-svc production-svc canary-svc 2>$null
kubectl delete configmap simple-configmap simple-configmap2 file-configmap 2>$null
kubectl delete secret simple-secret simple-secret2 file-secret 2>$null
```

---

## Key Takeaways

| Concept | What You Learned |
|---|---|
| **Deployments** | Declarative pod management with rolling updates, rollbacks, and strategies |
| **ReplicaSets** | Automatically created by Deployments; maintain desired pod count |
| **Services** | Stable network endpoint for a set of pods; three types for different access levels |
| **Internal LB** | Azure annotation to keep load balancers private within the VNet |
| **Blue/Green** | Two environments, instant traffic switch via selector change |
| **Canary** | Gradual rollout using shared selectors with proportional traffic |
| **ConfigMaps** | External configuration via env vars or mounted files |
| **Secrets** | Sensitive data with base64 encoding, consumed like ConfigMaps |

---

## Next Steps

- Explore **Ingress Controllers** for path-based routing
- Learn about **Horizontal Pod Autoscaler (HPA)** for auto-scaling
- Implement **Network Policies** to restrict pod-to-pod communication
- Use **Azure Key Vault Provider for Secrets Store CSI Driver** for production secrets management
