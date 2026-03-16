# Module 1: Nodes

## Concept

A **Node** is a physical or virtual machine that runs your workloads. Every Pod runs on a Node. A Kubernetes cluster consists of a set of Nodes managed by the control plane.

Each Node runs:
- **kubelet**: Agent that ensures containers are running in Pods
- **kube-proxy**: Maintains network rules for Service communication
- **Container runtime**: Software that runs containers (containerd, CRI-O)

### Node Types

| Type | Role |
|------|------|
| **Control Plane Node** | Runs API server, etcd, scheduler, controller manager |
| **Worker Node** | Runs application workloads (Pods) |

### Node Lifecycle

```
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │   Pending    │────►│    Ready     │────►│  Not Ready   │
  │ (joining)    │     │ (healthy)    │     │ (unhealthy)  │
  └─────────────┘     └──────┬───────┘     └──────────────┘
                              │
                              ▼
                       ┌─────────────┐
                       │ Scheduling  │
                       │ Pods here   │
                       └─────────────┘
```

Nodes self-register with the API server. The kubelet sends heartbeats so the control plane knows the Node is healthy. If heartbeats stop, the Node is marked `NotReady` and Pods are rescheduled.

## Practical Exercises

### Exercise 1: List and Inspect Nodes

```bash
# List all nodes
kubectl get nodes

# Wide output with OS, kernel, container runtime details
kubectl get nodes -o wide

# Get detailed info about a node
kubectl describe node <node-name>

# With minikube, there's typically one node
kubectl describe node minikube
```

Key sections in `describe` output:
- **Conditions**: Ready, MemoryPressure, DiskPressure, PIDPressure
- **Capacity vs Allocatable**: Total resources vs available for Pods
- **Allocated resources**: What's currently used
- **Events**: Recent node-level events

### Exercise 2: Node Resource Capacity

```bash
# View capacity and allocatable resources
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}  CPU capacity: {.status.capacity.cpu}{"\n"}  Memory capacity: {.status.capacity.memory}{"\n"}  Pods capacity: {.status.capacity.pods}{"\n\n"}{end}'

# Check how much is already allocated
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Top command - see real-time resource usage (requires metrics-server)
# minikube addons enable metrics-server
kubectl top nodes
```

### Exercise 3: Node Labels

```bash
# View existing labels
kubectl get nodes --show-labels

# Add a custom label
kubectl label node <node-name> environment=production

# Add more labels
kubectl label node <node-name> disk-type=ssd
kubectl label node <node-name> region=us-east

# Verify labels
kubectl get nodes --show-labels

# Filter nodes by label
kubectl get nodes -l environment=production
kubectl get nodes -l disk-type=ssd

# Remove a label (trailing dash)
kubectl label node <node-name> disk-type-

# Overwrite an existing label
kubectl label node <node-name> environment=staging --overwrite
```

### Exercise 4: Schedule a Pod on a Specific Node

```bash
# Apply the Pod with nodeSelector
kubectl apply -f pod-nodeselector.yaml

# Check which node the Pod landed on
kubectl get pod node-specific-pod -o wide

# Apply the Pod with nodeName (direct assignment)
kubectl apply -f pod-nodename.yaml

kubectl get pod node-pinned-pod -o wide

# Clean up
kubectl delete -f pod-nodeselector.yaml
kubectl delete -f pod-nodename.yaml
```

### Exercise 5: Taints and Tolerations

Taints repel Pods from Nodes. Tolerations allow Pods to be scheduled on tainted Nodes.

```bash
# Taint a node - no new Pods will be scheduled here
kubectl taint nodes <node-name> dedicated=special:NoSchedule

# Try scheduling a regular Pod - it will be Pending (on single-node clusters)
kubectl run test-pod --image=nginx:1.27
kubectl get pods test-pod
kubectl describe pod test-pod | grep -A 3 Events

# Schedule a Pod that tolerates the taint
kubectl apply -f pod-toleration.yaml
kubectl get pod tolerant-pod -o wide

# Remove the taint (trailing dash)
kubectl taint nodes <node-name> dedicated=special:NoSchedule-

# Clean up
kubectl delete pod test-pod
kubectl delete -f pod-toleration.yaml
```

### Exercise 6: Cordon and Drain a Node

Cordoning prevents new Pods from being scheduled. Draining evicts existing Pods.

```bash
# Cordon a node (mark as unschedulable)
kubectl cordon <node-name>

# Verify - status shows SchedulingDisabled
kubectl get nodes

# Try to schedule a Pod - it will be Pending
kubectl run cordon-test --image=nginx:1.27
kubectl get pods cordon-test

# Uncordon the node
kubectl uncordon <node-name>
kubectl get nodes

# Drain a node (evict all Pods) - typically for maintenance
# Use --ignore-daemonsets since DaemonSet Pods can't be evicted
# Use --delete-emptydir-data if Pods use emptyDir volumes
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# After maintenance, uncordon
kubectl uncordon <node-name>

# Clean up
kubectl delete pod cordon-test
```

### Exercise 7: Node Conditions and Debugging

```bash
# Check node conditions
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .status.conditions[*]}  {.type}: {.status}{"\n"}{end}{"\n"}{end}'

# Check system Pods running on a node
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name>

# Check node events
kubectl get events --field-selector involvedObject.kind=Node

# View kubelet logs (on the node itself, or via minikube)
minikube ssh -- journalctl -u kubelet --no-pager -n 50
```

### Exercise 8: Node Affinity

```bash
# Label your node
kubectl label node <node-name> zone=us-east-1a --overwrite

# Deploy with node affinity
kubectl apply -f pod-affinity.yaml

# Verify placement
kubectl get pod affinity-pod -o wide

# Clean up
kubectl delete -f pod-affinity.yaml
kubectl label node <node-name> zone-
```

## Key Takeaways

- Nodes are the machines that run your workloads
- Use `kubectl describe node` to see capacity, allocatable resources, and conditions
- **Labels** classify nodes; use `nodeSelector` or `nodeAffinity` to target them
- **Taints** repel Pods; **tolerations** allow exceptions
- **Cordon** prevents scheduling; **drain** evicts Pods for maintenance
- Node conditions (Ready, MemoryPressure, DiskPressure) indicate health
- The kubelet on each Node reports status back to the control plane
- Always check `Allocated resources` to understand cluster utilization

## Next Module

Proceed to [02-pods](../02-pods/) to learn about Pods, the smallest deployable unit in Kubernetes.
