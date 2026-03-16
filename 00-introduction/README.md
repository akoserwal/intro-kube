# Module 0: Introduction to Kubernetes & kubectl

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform that automates deploying, scaling, and managing containerized applications.

### Key Architecture Components

```
                    ┌─────────────────────────────┐
                    │       Control Plane          │
                    │  ┌───────────────────────┐   │
                    │  │ API Server             │   │
                    │  │ etcd                   │   │
                    │  │ Scheduler              │   │
                    │  │ Controller Manager     │   │
                    │  └───────────────────────┘   │
                    └──────────────┬───────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
      ┌───────▼──────┐    ┌───────▼──────┐     ┌───────▼──────┐
      │   Worker      │    │   Worker      │     │   Worker      │
      │   Node 1      │    │   Node 2      │     │   Node 3      │
      │  ┌─────────┐  │    │  ┌─────────┐  │     │  ┌─────────┐  │
      │  │ kubelet  │  │    │  │ kubelet  │  │     │  │ kubelet  │  │
      │  │ kube-    │  │    │  │ kube-    │  │     │  │ kube-    │  │
      │  │ proxy    │  │    │  │ proxy    │  │     │  │ proxy    │  │
      │  │ Pods     │  │    │  │ Pods     │  │     │  │ Pods     │  │
      │  └─────────┘  │    │  └─────────┘  │     │  └─────────┘  │
      └───────────────┘    └───────────────┘     └───────────────┘
```

- **API Server**: Front-end for the control plane; all communication goes through it
- **etcd**: Key-value store holding all cluster state
- **Scheduler**: Assigns Pods to Nodes based on resource requirements
- **Controller Manager**: Runs controllers (ReplicaSet, Deployment, etc.)
- **kubelet**: Agent on each node that manages Pods
- **kube-proxy**: Handles network rules for Service communication

## Practical Exercises

### Exercise 1: Explore Your Cluster

```bash
# Check cluster info
kubectl cluster-info

# View all nodes
kubectl get nodes

# Get detailed info about a node
kubectl describe node <node-name>

# View cluster component status
kubectl get componentstatuses

# Check kubectl version
kubectl version
```

### Exercise 2: Understand kubectl Output Formats

```bash
# Default output
kubectl get nodes

# Wide output with more details
kubectl get nodes -o wide

# YAML output
kubectl get nodes -o yaml

# JSON output
kubectl get nodes -o json

# Custom columns
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type

# JSONPath
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
```

### Exercise 3: Working with Contexts and Namespaces

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Set default namespace for current context
kubectl config set-context --current --namespace=kube-system

# Reset to default namespace
kubectl config set-context --current --namespace=default
```

### Exercise 4: Exploring API Resources

```bash
# List all API resources
kubectl api-resources

# List only namespaced resources
kubectl api-resources --namespaced=true

# Explain a resource (built-in docs)
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers

# View all resources across namespaces
kubectl get all -A
```

## Key kubectl Commands Cheat Sheet

| Command | Description |
|---------|-------------|
| `kubectl get <resource>` | List resources |
| `kubectl describe <resource> <name>` | Show detailed info |
| `kubectl create -f <file>` | Create from file |
| `kubectl apply -f <file>` | Create or update from file |
| `kubectl delete -f <file>` | Delete from file |
| `kubectl logs <pod>` | View Pod logs |
| `kubectl exec -it <pod> -- <cmd>` | Execute command in Pod |
| `kubectl port-forward <pod> <local>:<remote>` | Forward port to Pod |

## Next Module

Once you're comfortable with kubectl, proceed to [01-pods](../01-pods/).
