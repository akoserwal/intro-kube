# Module 1: Nodes

## Deep Dive Theory

A **Node** is a worker machine (physical or virtual) in the Kubernetes cluster. Understanding Nodes at a deep level is essential for SRE/DevOps engineers responsible for cluster reliability, capacity planning, and incident response.

### Node Architecture Internals

```
┌──────────────────────────────────────────────────────────────────┐
│                         Node (Linux Host)                        │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐  │
│  │   kubelet     │  │  kube-proxy  │  │   Container Runtime    │  │
│  │              │  │              │  │   (containerd/CRI-O)   │  │
│  │ - Pod        │  │ - iptables/  │  │                        │  │
│  │   lifecycle  │  │   IPVS rules │  │ - OCI image pull       │  │
│  │ - cAdvisor   │  │ - Service    │  │ - Container create/    │  │
│  │   metrics    │  │   routing    │  │   start/stop           │  │
│  │ - CSI mount  │  │ - Endpoint   │  │ - CRI gRPC interface   │  │
│  │ - Device     │  │   tracking   │  │ - Image GC             │  │
│  │   plugins    │  │              │  │                        │  │
│  └──────┬───────┘  └──────────────┘  └────────────────────────┘  │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │              Linux Kernel (cgroups v2, namespaces)            │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────────────┐   │ │
│  │  │ cgroup  │ │  net ns  │ │  pid ns  │ │  mount ns / UTS  │   │ │
│  │  │ cpu/mem │ │ veth/br  │ │ isolate  │ │  filesystem iso  │   │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └──────────────────┘   │ │
│  └──────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### kubelet: The Node Agent

The kubelet is the primary agent on every Node. Senior engineers must understand its critical responsibilities:

1. **Pod Lifecycle Management**: Watches the API server for PodSpecs assigned to its Node. Ensures the described containers are running and healthy.

2. **Node Registration**: On startup, kubelet self-registers with the API server, reporting:
   - Node name, labels, taints
   - Capacity: total CPU, memory, ephemeral storage, max Pods (default 110)
   - Allocatable: capacity minus resources reserved for system daemons

3. **Node Lease (Heartbeat)**: kubelet creates and periodically renews a `Lease` object in `kube-node-lease` namespace (default every 10s). If the lease expires (default 40s timeout), the node controller marks the Node `NotReady`.

4. **Container Health Probing**: Executes liveness, readiness, and startup probes.

5. **Resource Enforcement**: Enforces resource requests/limits via Linux cgroups. Triggers Pod eviction when the Node is under resource pressure.

6. **Volume Management**: Mounts volumes (CSI drivers, ConfigMaps, Secrets) into Pods.

7. **Static Pods**: Can run Pods defined in a local directory (e.g., `/etc/kubernetes/manifests/`) without the API server. This is how control plane components run on kubeadm clusters.

```bash
# Key kubelet flags to know:
# --node-status-update-frequency=10s     Heartbeat interval
# --eviction-hard                        Hard eviction thresholds
# --max-pods=110                         Max Pods per node
# --kube-reserved                        Resources reserved for K8s daemons
# --system-reserved                      Resources reserved for OS processes
# --enforce-node-allocatable             What to enforce limits on
```

### Node Conditions (What SREs Monitor)

| Condition | Meaning | SRE Impact |
|-----------|---------|------------|
| `Ready` | kubelet is healthy, can accept Pods | If False for >5min, Pods rescheduled |
| `MemoryPressure` | Node memory is low | Triggers Pod eviction (OOM risk) |
| `DiskPressure` | Node disk is low | Triggers image GC, then container/Pod eviction |
| `PIDPressure` | Too many processes | Prevents fork bombs from killing the Node |
| `NetworkUnavailable` | Node network not configured | CNI plugin issue, Pods can't communicate |

### Node Resource Model: Capacity vs Allocatable

```
                    Node Capacity (total)
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ kube-reserved│  │system-reserved│  │ eviction   │  │
│  │             │  │              │  │ threshold  │  │
│  │ kubelet,    │  │ sshd, OS     │  │ memory:    │  │
│  │ kube-proxy, │  │ kernel,      │  │ 100Mi      │  │
│  │ containerd  │  │ journald     │  │ nodefs:    │  │
│  │             │  │              │  │ 10%        │  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐    │
│  │          Allocatable (for Pods)               │    │
│  │                                              │    │
│  │  Allocatable = Capacity - kube-reserved      │    │
│  │                - system-reserved             │    │
│  │                - eviction-threshold          │    │
│  └──────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

Understanding this model is critical for capacity planning. Over-committing `requests` vs `limits` is one of the most common production issues.

### Eviction: The kubelet's Last Resort

When a Node runs low on resources, kubelet evicts Pods in this order:

1. **BestEffort Pods** (no requests/limits) - evicted first
2. **Burstable Pods** (requests < limits) - evicted next, sorted by resource usage relative to request
3. **Guaranteed Pods** (requests = limits) - evicted last

Hard eviction thresholds trigger immediate eviction. Soft thresholds allow a grace period.

```
Default hard eviction thresholds:
  memory.available < 100Mi
  nodefs.available < 10%
  imagefs.available < 15%
  nodefs.inodesFree < 5%
```

### kube-proxy Modes

| Mode | Mechanism | Performance | When to Use |
|------|-----------|-------------|-------------|
| **iptables** | Kernel iptables rules | O(n) rule count, slower at scale | Default, < 1000 Services |
| **IPVS** | Kernel IPVS (L4 LB) | O(1) lookup, faster | Large clusters, > 1000 Services |
| **nftables** | nftables rules | Modern replacement for iptables | Newer kernels (5.13+) |

### Node Networking: How Pods Get IPs

```
┌─────────── Node A ───────────┐     ┌─────────── Node B ───────────┐
│  Pod CIDR: 10.244.1.0/24     │     │  Pod CIDR: 10.244.2.0/24     │
│                               │     │                               │
│  ┌─────────┐  ┌─────────┐    │     │  ┌─────────┐  ┌─────────┐    │
│  │ Pod A1   │  │ Pod A2   │    │     │  │ Pod B1   │  │ Pod B2   │    │
│  │10.244.1.2│  │10.244.1.3│    │     │  │10.244.2.2│  │10.244.2.3│    │
│  └────┬─────┘  └────┬─────┘    │     │  └────┬─────┘  └────┬─────┘    │
│       └──────┬───────┘         │     │       └──────┬───────┘         │
│           cbr0 bridge          │     │           cbr0 bridge          │
│              │                 │     │              │                 │
│         eth0 (node)            │     │         eth0 (node)            │
└──────────────┼─────────────────┘     └──────────────┼─────────────────┘
               │          CNI Overlay/Routing          │
               └───────────────────────────────────────┘
```

The CNI plugin (Calico, Cilium, Flannel) handles:
- Assigning Pod CIDRs to Nodes
- Creating veth pairs and bridges
- Programming routes or overlay tunnels between Nodes

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

# Compare capacity vs allocatable (what's reserved for system?)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}  CPU allocatable: {.status.allocatable.cpu}{"\n"}  Mem allocatable: {.status.allocatable.memory}{"\n"}  CPU capacity:    {.status.capacity.cpu}{"\n"}  Mem capacity:    {.status.capacity.memory}{"\n\n"}{end}'

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

# Check the node lease (heartbeat mechanism)
kubectl get lease -n kube-node-lease
kubectl describe lease <node-name> -n kube-node-lease
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

---

## Challenges

These challenges simulate real-world SRE scenarios. Try to solve them before looking at the solutions.

### Challenge 1: Capacity Planning Audit

**Scenario**: You've been asked to audit the cluster for capacity risks before a traffic surge. Produce a report of resource utilization.

**Tasks**:
1. Calculate the percentage of CPU and memory allocated vs allocatable on each Node
2. Identify how many more Pods can be scheduled on each Node
3. Find all Pods without resource requests (BestEffort QoS) - these are eviction risks

```bash
# Hint: Start here
kubectl describe nodes | grep -A 10 "Allocated resources"
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{" QoS="}{.status.qosClass}{"\n"}{end}'
```

<details>
<summary>Solution</summary>

```bash
# 1. Resource allocation percentage
echo "=== Node Resource Utilization ==="
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  echo "--- $node ---"
  kubectl describe node "$node" | grep -A 6 "Allocated resources"
  echo ""
done

# 2. Remaining Pod capacity
echo "=== Pod Capacity ==="
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  max=$(kubectl get node "$node" -o jsonpath='{.status.allocatable.pods}')
  current=$(kubectl get pods -A --field-selector spec.nodeName="$node" --no-headers | wc -l)
  echo "$node: $current / $max Pods ($(( max - current )) remaining)"
done

# 3. BestEffort Pods (eviction risk)
echo ""
echo "=== BestEffort Pods (no requests/limits - eviction risk) ==="
kubectl get pods -A -o jsonpath='{range .items[?(@.status.qosClass=="BestEffort")]}{.metadata.namespace}{"/"}{.metadata.name}{"\n"}{end}'
```

</details>

### Challenge 2: Node Failure Simulation

**Scenario**: A production node becomes unhealthy. Understand what happens to Pods and how the system recovers.

**Tasks**:
1. Deploy a 3-replica Deployment
2. Cordon the node to simulate it going offline
3. Observe what happens to existing Pods
4. Scale the Deployment to 5 replicas - observe scheduling behavior
5. Uncordon and verify the system rebalances

```bash
kubectl apply -f challenges/challenge-node-failure.yaml
```

<details>
<summary>Solution</summary>

```bash
# 1. Deploy
kubectl apply -f challenges/challenge-node-failure.yaml
kubectl get pods -o wide

# 2. Cordon the node
kubectl cordon <node-name>
kubectl get nodes  # Shows SchedulingDisabled

# 3. Existing Pods KEEP RUNNING (cordon only affects new scheduling)
kubectl get pods -o wide  # Pods still Running

# 4. Try to scale - new Pods will be Pending (on single-node cluster)
kubectl scale deployment challenge-app --replicas=5
kubectl get pods -o wide
# On multi-node: new Pods go to other nodes
# On single-node: new Pods stay Pending

# 5. Uncordon
kubectl uncordon <node-name>
kubectl get pods -o wide  # Pending Pods now get scheduled

# Key SRE insight:
# - Cordon != drain. Running Pods stay running.
# - On multi-node clusters, new Pods route to healthy nodes.
# - Pod Disruption Budgets (PDBs) control drain behavior.

# Clean up
kubectl delete -f challenges/challenge-node-failure.yaml
```

</details>

### Challenge 3: Taint-Based Workload Isolation

**Scenario**: You need to dedicate specific nodes to GPU workloads. Only GPU-aware Pods should be scheduled on GPU nodes, and all other workloads must be kept off.

**Tasks**:
1. Taint a node with `gpu=true:NoSchedule`
2. Deploy a regular workload (should avoid the GPU node)
3. Deploy a GPU-tolerant workload (should be able to use the GPU node)
4. Add a `NoExecute` taint and observe what happens to running Pods

```bash
kubectl apply -f challenges/challenge-gpu-isolation.yaml
```

<details>
<summary>Solution</summary>

```bash
# 1. Taint the node
kubectl taint nodes <node-name> gpu=true:NoSchedule

# 2. Regular workload - will be Pending on single-node cluster
kubectl apply -f challenges/challenge-gpu-isolation.yaml
kubectl get pod regular-workload  # Pending (can't tolerate taint)

# 3. GPU workload - has toleration, will be scheduled
kubectl get pod gpu-workload  # Running

# 4. Add NoExecute taint - EVICTS running Pods that don't tolerate it
kubectl taint nodes <node-name> maintenance=true:NoExecute
# Pods without this toleration are immediately evicted

kubectl get pods  # regular-workload terminated, gpu-workload too
# (gpu-workload only tolerates gpu=true, not maintenance=true)

# Key SRE insight:
# - NoSchedule: prevents new scheduling only
# - PreferNoSchedule: soft version, scheduler tries to avoid
# - NoExecute: evicts existing Pods (most aggressive)
# - tolerationSeconds on NoExecute controls how long before eviction

# Clean up
kubectl taint nodes <node-name> gpu=true:NoSchedule-
kubectl taint nodes <node-name> maintenance=true:NoExecute-
kubectl delete -f challenges/challenge-gpu-isolation.yaml
```

</details>

### Challenge 4: Node Drain with Pod Disruption Budget

**Scenario**: You need to drain a node for kernel patching, but your application requires at least 2 replicas running at all times during the drain.

**Tasks**:
1. Deploy a 4-replica application with a PodDisruptionBudget (minAvailable: 2)
2. Attempt to drain the node
3. Observe how the PDB interacts with the drain process
4. Understand when drains can get stuck and how to troubleshoot

```bash
kubectl apply -f challenges/challenge-pdb-drain.yaml
```

<details>
<summary>Solution</summary>

```bash
# 1. Deploy app with PDB
kubectl apply -f challenges/challenge-pdb-drain.yaml

kubectl get pods -o wide
kubectl get pdb

# 2. Attempt drain
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 3. The drain respects the PDB:
# - It evicts Pods one at a time
# - After each eviction, it waits for the replacement Pod to be Ready
# - It ensures minAvailable=2 is maintained throughout
# - On single-node clusters, drain can get stuck because
#   evicted Pods can't be rescheduled anywhere else

# 4. If drain gets stuck (common SRE scenario):
# Check PDB status
kubectl get pdb -o wide
# Shows: ALLOWED DISRUPTIONS, CURRENT, DESIRED, etc.

# If you MUST force the drain (data loss risk):
# kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force
# ^^^ This ignores PDBs - only use in emergencies

# Key SRE insight:
# - PDBs are only respected by voluntary disruptions (drain, eviction API)
# - Node crashes bypass PDBs entirely
# - Set PDBs based on your application's actual tolerance for disruption
# - maxUnavailable is often better than minAvailable for rolling drains

# Uncordon and clean up
kubectl uncordon <node-name>
kubectl delete -f challenges/challenge-pdb-drain.yaml
```

</details>

### Challenge 5: Multi-Dimension Scheduling

**Scenario**: Design a scheduling strategy for a 3-tier application where:
- Frontend Pods should prefer nodes in `zone=us-east-1a` but can run anywhere
- Backend Pods MUST run on nodes labeled `tier=compute`
- Database Pods must NOT share a node with other database Pods

**Tasks**:
1. Label your node appropriately
2. Deploy all three tiers using the correct affinity/anti-affinity rules
3. Verify the placement meets all requirements

```bash
kubectl apply -f challenges/challenge-scheduling.yaml
```

<details>
<summary>Solution</summary>

```bash
# 1. Label the node
kubectl label node <node-name> zone=us-east-1a tier=compute --overwrite

# 2. Deploy
kubectl apply -f challenges/challenge-scheduling.yaml

# 3. Verify
kubectl get pods -o wide --show-labels

# Check frontend: should be scheduled (prefers zone=us-east-1a)
kubectl get pod -l app=challenge-frontend -o wide

# Check backend: should be scheduled (requires tier=compute)
kubectl get pod -l app=challenge-backend -o wide

# Check database: anti-affinity prevents 2 db pods on same node
# On single-node cluster, only 1 db Pod will be Running, others Pending
kubectl get pod -l app=challenge-db -o wide

# Key SRE insight:
# - requiredDuringScheduling = hard constraint (Pod stays Pending if unsatisfied)
# - preferredDuringScheduling = soft constraint (best effort, scored)
# - Pod anti-affinity at topology "kubernetes.io/hostname" = 1 per node
# - Pod anti-affinity at topology "topology.kubernetes.io/zone" = 1 per AZ

# Clean up
kubectl delete -f challenges/challenge-scheduling.yaml
kubectl label node <node-name> zone- tier-
```

</details>

## Key Takeaways

- Nodes are the machines that run your workloads
- Use `kubectl describe node` to see capacity, allocatable resources, and conditions
- **Labels** classify nodes; use `nodeSelector` or `nodeAffinity` to target them
- **Taints** repel Pods; **tolerations** allow exceptions
- **Cordon** prevents scheduling; **drain** evicts Pods for maintenance
- Node conditions (Ready, MemoryPressure, DiskPressure) indicate health
- The kubelet on each Node reports status back to the control plane via Node Leases
- Always check `Allocated resources` to understand cluster utilization
- **Eviction order**: BestEffort -> Burstable -> Guaranteed (set requests/limits!)
- **PDBs** protect availability during voluntary disruptions (drains)
- Capacity planning requires understanding: capacity - kube-reserved - system-reserved - eviction threshold = allocatable

## Next Module

Proceed to [02-pods](../02-pods/) to learn about Pods, the smallest deployable unit in Kubernetes.
