# Kubernetes: From Beginner to Expert

A comprehensive guide to mastering Kubernetes, structured as progressive lessons from basic concepts to advanced cluster management.

## Table of Contents

1. [Beginner Level](#beginner-level)
   - [What is Kubernetes?](#what-is-kubernetes)
   - [Core Concepts](#core-concepts)
   - [Getting Started](#getting-started)
   - [Basic Commands](#basic-commands)

2. [Intermediate Level](#intermediate-level)
   - [Workloads](#workloads)
   - [Services and Networking](#services-and-networking)
   - [Configuration Management](#configuration-management)
   - [Storage](#storage)

3. [Advanced Level](#advanced-level)
   - [Security](#security)
   - [Resource Management](#resource-management)
   - [Monitoring and Logging](#monitoring-and-logging)
   - [Custom Resources](#custom-resources)

4. [Expert Level](#expert-level)
   - [Cluster Administration](#cluster-administration)
   - [Troubleshooting](#troubleshooting)
   - [Performance Optimization](#performance-optimization)
   - [Production Best Practices](#production-best-practices)

---

## Beginner Level

### What is Kubernetes?

**Kubernetes (k8s)** is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

#### Key Benefits:
- **Automated deployment and scaling**
- **Self-healing capabilities**
- **Service discovery and load balancing**
- **Storage orchestration**
- **Configuration management**

#### Architecture Overview:
```
┌─────────────────┐    ┌─────────────────┐
│   Master Node   │    │   Worker Node   │
│                 │    │                 │
│  ┌───────────┐  │    │  ┌───────────┐  │
│  │ API Server│  │    │  │  Kubelet  │  │
│  └───────────┘  │    │  └───────────┘  │
│  ┌───────────┐  │    │  ┌───────────┐  │
│  │  etcd     │  │    │  │ Container │  │
│  └───────────┘  │    │  │ Runtime   │  │
│  ┌───────────┐  │    │  └───────────┘  │
│  │Scheduler  │  │    │  ┌───────────┐  │
│  └───────────┘  │    │  │kube-proxy │  │
│  ┌───────────┐  │    │  └───────────┘  │
│  │Controller │  │    │                 │
│  │ Manager   │  │    │                 │
│  └───────────┘  │    │                 │
└─────────────────┘    └─────────────────┘
```

### Core Concepts

#### 1. Pod
The smallest deployable unit in Kubernetes. Contains one or more containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.20
    ports:
    - containerPort: 80
```

#### 2. Node
A worker machine (VM or physical) that runs Pods.

#### 3. Cluster
A set of nodes that run containerized applications managed by Kubernetes.

#### 4. Namespace
Virtual clusters within a physical cluster for resource isolation.

```bash
# Create namespace
kubectl create namespace development

# List namespaces
kubectl get namespaces
```

### Getting Started

#### Prerequisites:
- Docker installed
- kubectl CLI tool
- Access to a Kubernetes cluster (minikube, kind, or cloud provider)

#### Local Development Setup:

**Option 1: Minikube**
```bash
# Install minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin

# Start cluster
minikube start

# Check status
minikube status
```

**Option 2: Kind (Kubernetes in Docker)**
```bash
# Install kind
go install sigs.k8s.io/kind@v0.20.0

# Create cluster
kind create cluster --name my-cluster

# Get clusters
kind get clusters
```

### Basic Commands

#### Cluster Information:
```bash
# Cluster info
kubectl cluster-info

# Node information
kubectl get nodes

# Detailed node info
kubectl describe node <node-name>
```

#### Working with Pods:
```bash
# Create a pod
kubectl run nginx --image=nginx

# List pods
kubectl get pods

# Describe pod
kubectl describe pod nginx

# Get pod logs
kubectl logs nginx

# Execute commands in pod
kubectl exec -it nginx -- /bin/bash

# Delete pod
kubectl delete pod nginx
```

#### Working with Namespaces:
```bash
# Create namespace
kubectl create namespace my-namespace

# List all resources in namespace
kubectl get all -n my-namespace

# Set default namespace
kubectl config set-context --current --namespace=my-namespace
```

---

## Intermediate Level

### Workloads

#### 1. Deployments
Manages a set of identical Pods, providing declarative updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

```bash
# Apply deployment
kubectl apply -f nginx-deployment.yaml

# Scale deployment
kubectl scale deployment nginx-deployment --replicas=5

# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.21

# Rollback
kubectl rollout undo deployment/nginx-deployment

# Check rollout status
kubectl rollout status deployment/nginx-deployment
```

#### 2. ReplicaSets
Ensures a specified number of Pod replicas are running.

#### 3. DaemonSets
Ensures a Pod runs on all (or some) nodes.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logging-daemon
spec:
  selector:
    matchLabels:
      name: logging-daemon
  template:
    metadata:
      labels:
        name: logging-daemon
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.14
```

#### 4. Jobs and CronJobs
For running batch workloads.

```yaml
# Job
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never

---
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["/backup-script.sh"]
          restartPolicy: OnFailure
```

### Services and Networking

#### 1. Services
Exposes Pods to network traffic.

**ClusterIP Service (Internal):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

**NodePort Service (External):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

**LoadBalancer Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

#### 2. Ingress
HTTP and HTTPS routing to services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

#### 3. Network Policies
Control traffic flow between Pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Configuration Management

#### 1. ConfigMaps
Store non-confidential configuration data.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgresql://localhost:5432/mydb"
  debug: "true"
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    logging:
      level: info
```

**Using ConfigMap:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

#### 2. Secrets
Store sensitive information.

```bash
# Create secret from command line
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secretpassword

# Create secret from file
kubectl create secret generic ssl-certs \
  --from-file=tls.crt=path/to/tls.crt \
  --from-file=tls.key=path/to/tls.key
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded 'admin'
  password: c2VjcmV0cGFzc3dvcmQ=  # base64 encoded 'secretpassword'
```

### Storage

#### 1. Volumes
Provide persistent storage for Pods.

**EmptyDir Volume:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: shared-data
    emptyDir: {}
```

#### 2. Persistent Volumes (PV) and Persistent Volume Claims (PVC)

**Persistent Volume:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/my-pv
```

**Persistent Volume Claim:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

**Using PVC in Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

---

## Advanced Level

### Security

#### 1. RBAC (Role-Based Access Control)

**Role:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**ClusterRole:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

**RoleBinding:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### 2. Service Accounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default

---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  containers:
  - name: app
    image: myapp:latest
```

#### 3. Pod Security Standards

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

### Resource Management

#### 1. Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    persistentvolumeclaims: "4"
```

#### 2. Limit Ranges

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory-limit-range
  namespace: development
spec:
  limits:
  - default:
      cpu: 1
      memory: "512Mi"
    defaultRequest:
      cpu: 0.5
      memory: "256Mi"
    type: Container
```

#### 3. Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Monitoring and Logging

#### 1. Health Checks

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: healthy-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 30
```

#### 2. Monitoring with Prometheus

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-metrics
  labels:
    app: my-app
spec:
  ports:
  - port: 8080
    name: metrics
  selector:
    app: my-app

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
```

### Custom Resources

#### 1. Custom Resource Definition (CRD)

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.stable.example.com
spec:
  group: stable.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
              version:
                type: string
              storage:
                type: string
          status:
            type: object
            properties:
              phase:
                type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
```

#### 2. Custom Resource

```yaml
apiVersion: stable.example.com/v1
kind: Database
metadata:
  name: my-database
spec:
  engine: postgres
  version: "13"
  storage: "10Gi"
```

---

## Expert Level

### Cluster Administration

#### 1. Cluster Setup and Management

**kubeadm Cluster Initialization:**
```bash
# Initialize master node
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Install network plugin (Flannel)
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Join worker nodes
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

#### 2. Backup and Restore

**etcd Backup:**
```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db
```

#### 3. Certificate Management

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Renew certificates
kubeadm certs renew all

# Generate new certificates
kubeadm init phase certs all
```

### Troubleshooting

#### 1. Common Debugging Commands

```bash
# Node debugging
kubectl describe node <node-name>
kubectl get events --sort-by=.metadata.creationTimestamp

# Pod debugging
kubectl logs <pod-name> -c <container-name> --previous
kubectl exec -it <pod-name> -- /bin/sh
kubectl port-forward <pod-name> 8080:80

# Network debugging
kubectl get endpoints
kubectl describe service <service-name>
kubectl get networkpolicies

# Resource debugging
kubectl top nodes
kubectl top pods
kubectl describe quota
```

#### 2. Troubleshooting Scenarios

**Pod Stuck in Pending:**
```bash
# Check events
kubectl describe pod <pod-name>

# Check node resources
kubectl top nodes
kubectl describe nodes

# Check PVCs
kubectl get pvc
```

**Service Not Accessible:**
```bash
# Check service endpoints
kubectl get endpoints <service-name>

# Check if pods are running and ready
kubectl get pods -l <label-selector>

# Test service connectivity
kubectl run test-pod --rm -it --image=busybox -- nslookup <service-name>
```

### Performance Optimization

#### 1. Resource Optimization

```yaml
# Resource requests and limits optimization
apiVersion: apps/v1
kind: Deployment
metadata:
  name: optimized-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        # Quality of Service: Guaranteed
```

#### 2. Node Affinity and Anti-Affinity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values:
                - web-server
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-app
              topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx:latest
```

#### 3. Cluster Autoscaler

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
        name: cluster-autoscaler
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/kubernetes
```

### Production Best Practices

#### 1. Security Hardening

```yaml
# Pod Security Policy (deprecated, use Pod Security Standards)
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

#### 2. Multi-Environment Configuration

```yaml
# Production overlay with kustomize
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

patchesStrategicMerge:
- production-config.yaml

replicas:
- name: web-app
  count: 5

images:
- name: myapp
  newTag: v1.2.3
```

#### 3. Disaster Recovery Strategy

```bash
# Regular backup script
#!/bin/bash

# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save \
  /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Backup application data
kubectl get all --all-namespaces -o yaml > /backup/k8s-resources-$(date +%Y%m%d).yaml

# Backup secrets and configmaps
kubectl get secrets,configmaps --all-namespaces -o yaml > /backup/k8s-configs-$(date +%Y%m%d).yaml
```

#### 4. GitOps with ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-config
    targetRevision: HEAD
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## Additional Resources

### Useful Commands Cheat Sheet

```bash
# Cluster and Node Information
kubectl cluster-info
kubectl get nodes -o wide
kubectl describe node <node-name>

# Workload Management
kubectl get pods,svc,deploy --all-namespaces
kubectl scale deployment <deployment-name> --replicas=3
kubectl rollout restart deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>

# Debugging and Troubleshooting
kubectl logs -f <pod-name> -c <container-name>
kubectl exec -it <pod-name> -- /bin/bash
kubectl port-forward <pod-name> 8080:80
kubectl get events --sort-by=.metadata.creationTimestamp

# Resource Management
kubectl top nodes
kubectl top pods --all-namespaces
kubectl describe quota --all-namespaces

# Configuration and Secrets
kubectl create configmap <name> --from-file=<file>
kubectl create secret generic <name> --from-literal=key=value
kubectl get configmaps,secrets --all-namespaces
```

### Recommended Tools

1. **kubectl** - Official CLI tool
2. **helm** - Package manager for Kubernetes
3. **kustomize** - Configuration management
4. **k9s** - Terminal-based UI
5. **kubectx/kubens** - Context and namespace switching
6. **stern** - Multi-pod log tailing
7. **dive** - Container image analysis
8. **kube-bench** - Security benchmark
9. **falco** - Runtime security monitoring
10. **prometheus + grafana** - Monitoring and alerting

### Learning Path Summary

1. **Beginner (1-2 months)**
   - Understand containers and Docker
   - Learn basic Kubernetes concepts
   - Practice with local clusters (minikube/kind)
   - Master basic kubectl commands

2. **Intermediate (2-3 months)**
   - Deploy applications with Deployments and Services
   - Understand networking and storage
   - Learn configuration management
   - Practice troubleshooting

3. **Advanced (3-4 months)**
   - Implement security best practices
   - Set up monitoring and logging
   - Create custom resources and operators
   - Learn advanced networking concepts

4. **Expert (6+ months)**
   - Master cluster administration
   - Implement production-grade setups
   - Design disaster recovery strategies
   - Contribute to the Kubernetes ecosystem

---

*This documentation serves as a comprehensive guide to mastering Kubernetes. Practice regularly, build projects, and stay updated with the rapidly evolving Kubernetes ecosystem.*
