# Module 2: Pods

## Deep Dive Theory

A **Pod** is the smallest deployable unit in Kubernetes. For senior engineers, understanding Pods goes far beyond "a wrapper around a container." Pods are the fundamental execution boundary where Linux kernel primitives, networking, and storage converge.

### What a Pod Really Is: Linux Primitives

A Pod is NOT a container. A Pod is a group of **Linux namespaces** shared by one or more containers:

```
┌─────────────────── Pod ───────────────────────────────┐
│                                                       │
│  Shared Linux Namespaces:                             │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Network Namespace (single IP, shared localhost)  │  │
│  │ UTS Namespace (shared hostname)                  │  │
│  │ IPC Namespace (shared semaphores, shared memory) │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  Per-Container Namespaces:                            │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │ Container A  │  │ Container B  │                   │
│  │ PID ns       │  │ PID ns       │                   │
│  │ Mount ns     │  │ Mount ns     │                   │
│  │ User ns (opt)│  │ User ns (opt)│                   │
│  │ cgroup       │  │ cgroup       │                   │
│  └──────────────┘  └──────────────┘                   │
│                                                       │
│  Shared Volumes:                                      │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Volume mounts accessible by all containers      │  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

Before any user container starts, Kubernetes creates a **pause container** (also called the "infra container"). This container:
- Creates and holds the network namespace for the Pod
- Has PID 1 and reaps zombie processes
- Does nothing else (runs `/pause`)
- Ensures the network namespace survives container restarts

```bash
# You can see the pause container on the node:
# minikube ssh
# crictl ps | grep pause
```

### Pod Networking In Depth

Every Pod gets its own IP address from the Node's Pod CIDR. Here's what happens at the kernel level:

```
┌─────────────────── Node ──────────────────────┐
│                                               │
│  ┌──── Pod (net ns) ────┐                     │
│  │                      │                     │
│  │  eth0 (10.244.1.5)   │  ◄── veth pair ──► │ ── cni0/cbr0 bridge
│  │                      │                     │        │
│  │  lo (127.0.0.1)      │                     │        │
│  │  Container A: :8080  │                     │     eth0 (node)
│  │  Container B: :9090  │                     │        │
│  │  (share same eth0)   │                     │     Internet
│  └──────────────────────┘                     │
└───────────────────────────────────────────────┘
```

Key networking facts:
- All containers in a Pod share `eth0` - they communicate via `localhost`
- Containers must use different ports (port conflict = crash)
- Pod-to-Pod communication is always routable (flat network model)
- No NAT between Pods, even across Nodes (CNI plugin handles routing)

### Pod Lifecycle: The Full State Machine

```
                          ┌──────────┐
                          │ Pending  │ API accepted, not yet scheduled
                          └────┬─────┘
                               │ scheduler assigns Node
                          ┌────▼─────┐
                          │ Init     │ Init containers run sequentially
                          │ Phase    │ (if any fail, Pod restarts them)
                          └────┬─────┘
                               │ all init containers complete
                          ┌────▼─────┐
                          │ Running  │ At least one container running
                          └────┬─────┘
                               │
                    ┌──────────┼──────────┐
                    │          │          │
              ┌─────▼────┐ ┌──▼───────┐ ┌▼──────────┐
              │Succeeded │ │  Failed  │ │  Unknown   │
              │(exit 0)  │ │(exit !=0)│ │(node lost) │
              └──────────┘ └──────────┘ └────────────┘
```

### Container Restart Policy and CrashLoopBackOff

The `restartPolicy` determines what happens when a container exits:

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `Always` (default) | Restart on any exit | Long-running services |
| `OnFailure` | Restart only on non-zero exit | Jobs, batch workloads |
| `Never` | Never restart | Debug Pods, one-shot tasks |

**CrashLoopBackOff** is the exponential backoff applied to restarting containers:

```
Restart 1: wait 10s
Restart 2: wait 20s
Restart 3: wait 40s
Restart 4: wait 80s
Restart 5: wait 160s
...caps at 300s (5 minutes)
```

This is one of the most common Pods issues SREs encounter. The kubelet backs off to prevent resource exhaustion from a continuously crashing container.

### Quality of Service (QoS) Classes

Kubernetes assigns a QoS class to every Pod based on its resource configuration. This determines eviction priority when the Node is under pressure:

| QoS Class | Condition | Eviction Priority |
|-----------|-----------|-------------------|
| **Guaranteed** | Every container has `requests == limits` for both CPU and memory | Last to be evicted |
| **Burstable** | At least one container has a request or limit, but doesn't qualify as Guaranteed | Middle priority |
| **BestEffort** | No requests or limits on any container | First to be evicted |

```yaml
# Guaranteed (safest for production)
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"       # same as request
    memory: "256Mi"   # same as request

# Burstable (can use more than requested)
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "500m"       # different from request
    memory: "256Mi"   # different from request

# BestEffort (no resources specified) - AVOID IN PRODUCTION
# resources: {}
```

**SRE Best Practice**: Always set resource requests. Set limits equal to requests for predictable behavior (Guaranteed QoS). Burstable is acceptable when you understand the overcommit ratio.

### Probes: Keeping Pods Honest

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| **startupProbe** | Is the app started? | Kill + restart container (blocks other probes until success) |
| **livenessProbe** | Is the app alive? | Kill + restart container |
| **readinessProbe** | Can the app serve traffic? | Remove from Service endpoints (no restart) |

```
    Container starts
         │
    ┌────▼────────────────────┐
    │ startupProbe running    │ ◄── Blocks liveness/readiness
    │ (failureThreshold × pe  │     Use for slow-starting apps
    │  periodSeconds = budget)│
    └────┬────────────────────┘
         │ startup succeeds
    ┌────▼────────────────────┐    ┌─────────────────────────┐
    │ livenessProbe           │    │ readinessProbe           │
    │ running periodically    │    │ running periodically     │
    │                         │    │                         │
    │ Failure → kill + restart│    │ Failure → remove from   │
    │                         │    │ Service endpoints       │
    └─────────────────────────┘    └─────────────────────────┘
```

Probe types:
- `httpGet`: HTTP GET to a path/port (200-399 = success)
- `tcpSocket`: TCP connection to a port (connection = success)
- `exec`: Run a command in the container (exit 0 = success)
- `grpc`: gRPC health check (for gRPC services)

**Common SRE Mistake**: Setting a livenessProbe that depends on external services (database, API). If the database goes down, the livenessProbe fails, kubelet restarts all your Pods, and you've turned a database outage into a cascading failure.

### Multi-Container Pod Patterns

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Sidecar** | Extend main container functionality | Log shipper, service mesh proxy (Envoy) |
| **Ambassador** | Proxy connections to external services | Local Redis proxy, API gateway |
| **Adapter** | Transform output of main container | Log format normalizer, metrics adapter |
| **Init Container** | Setup before main containers start | DB migration, config download, wait for dependency |

### Pod Security: SecurityContext

```yaml
spec:
  securityContext:           # Pod-level
    runAsNonRoot: true       # Prevent running as root
    runAsUser: 1000          # Run as specific UID
    fsGroup: 2000            # Supplementary group for volumes
    seccompProfile:
      type: RuntimeDefault   # Apply seccomp profile
  containers:
    - name: app
      securityContext:       # Container-level (overrides Pod-level)
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]      # Drop all Linux capabilities
```

### Pod Overhead and Resource Accounting

Beyond your container resources, Pods consume overhead:
- **Pause container**: ~1MB memory, negligible CPU
- **Container runtime overhead**: CRI, shim processes
- **Volumes**: inode and disk usage
- **Network**: conntrack entries, iptables rules per Service endpoint

In large clusters, this overhead adds up. A Node with 110 Pods has 110+ pause containers.

## Practical Exercises

### Exercise 1: Create a Pod Imperatively

```bash
# Run a simple nginx Pod
kubectl run my-nginx --image=nginx:1.27

# Check the Pod status
kubectl get pods

# Watch Pod status in real-time
kubectl get pods -w

# Get detailed Pod info
kubectl describe pod my-nginx

# View Pod logs
kubectl logs my-nginx

# Execute a command inside the Pod
kubectl exec -it my-nginx -- /bin/bash

# Inside the Pod, verify nginx is running
curl localhost:80
exit

# Clean up
kubectl delete pod my-nginx
```

### Exercise 2: Create a Pod Declaratively

Apply the manifest:
```bash
kubectl apply -f pod-basic.yaml

# Verify
kubectl get pod nginx-pod
kubectl describe pod nginx-pod

# Clean up
kubectl delete -f pod-basic.yaml
```

### Exercise 3: Pod with Resource Limits

```bash
kubectl apply -f pod-resources.yaml

# Check resource allocation
kubectl describe pod nginx-resources

# Check QoS class
kubectl get pod nginx-resources -o jsonpath='{.status.qosClass}'

# Clean up
kubectl delete -f pod-resources.yaml
```

### Exercise 4: Multi-Container Pod (Sidecar Pattern)

```bash
kubectl apply -f pod-sidecar.yaml

# Check both containers are running
kubectl get pod sidecar-pod

# View logs from each container
kubectl logs sidecar-pod -c app-container
kubectl logs sidecar-pod -c sidecar-container

# Exec into a specific container
kubectl exec -it sidecar-pod -c app-container -- /bin/sh

# Inside: check shared data written by sidecar
cat /usr/share/nginx/html/index.html
exit

# Clean up
kubectl delete -f pod-sidecar.yaml
```

### Exercise 5: Pod with Init Container

```bash
kubectl apply -f pod-init.yaml

# Watch the init container complete before main container starts
kubectl get pod init-pod -w

# Check init container logs
kubectl logs init-pod -c init-myservice

# Clean up
kubectl delete -f pod-init.yaml
```

### Exercise 6: Pod Lifecycle and Debugging

```bash
# Create a Pod that will crash
kubectl apply -f pod-crash.yaml

# Observe CrashLoopBackOff
kubectl get pods -w

# Debug: check logs
kubectl logs crash-pod

# Debug: check events
kubectl describe pod crash-pod

# Debug: check previous container logs
kubectl logs crash-pod --previous

# Clean up
kubectl delete -f pod-crash.yaml
```

### Exercise 7: Health Probes in Action

```bash
kubectl apply -f pod-probes.yaml

# Watch the Pod become Ready
kubectl get pod probed-app -w

# Check the probe configuration
kubectl describe pod probed-app | grep -A 5 "Liveness\|Readiness\|Startup"

# Simulate a liveness failure (the app becomes unresponsive)
kubectl exec probed-app -- rm /usr/share/nginx/html/index.html

# Watch: readiness probe fails first (removed from Service)
# Then liveness probe fails (container restarted)
kubectl get pod probed-app -w
kubectl describe pod probed-app | tail -20

# Clean up
kubectl delete -f pod-probes.yaml
```

### Exercise 8: QoS Classes Comparison

```bash
kubectl apply -f pod-qos.yaml

# Check QoS class of each Pod
kubectl get pods -l experiment=qos -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass,STATUS:.status.phase

# Describe each to see resource allocation
kubectl describe pod qos-guaranteed | grep -A 6 "Requests\|Limits\|QoS"
kubectl describe pod qos-burstable | grep -A 6 "Requests\|Limits\|QoS"
kubectl describe pod qos-besteffort | grep -A 6 "Requests\|Limits\|QoS"

# Clean up
kubectl delete -f pod-qos.yaml
```

### Exercise 9: Security Context

```bash
kubectl apply -f pod-security.yaml

# Verify the container is running as non-root
kubectl exec secure-pod -- id
kubectl exec secure-pod -- whoami

# Try to write to the root filesystem (should fail - readOnlyRootFilesystem)
kubectl exec secure-pod -- touch /test-file

# Writing to the mounted writable volume should work
kubectl exec secure-pod -- touch /tmp/test-file
kubectl exec secure-pod -- ls -la /tmp/test-file

# Check that capabilities were dropped
kubectl exec secure-pod -- cat /proc/1/status | grep Cap

# Clean up
kubectl delete -f pod-security.yaml
```

---

## Challenges

These challenges simulate real-world SRE/DevOps scenarios. Try to solve them before looking at the solutions.

### Challenge 1: Debugging a Failing Pod

**Scenario**: A developer deployed a Pod but it keeps crashing. They hand it off to you. Diagnose and fix the issue.

**Tasks**:
1. Apply the broken Pod manifest
2. Identify why the Pod is failing (check events, logs, describe)
3. Determine the root cause
4. Fix the manifest and redeploy

```bash
kubectl apply -f challenges/challenge-debug-pod.yaml
```

<details>
<summary>Hints</summary>

- Check `kubectl describe pod` for events
- Check `kubectl logs` for application output
- Look at the container command, image, and resource configuration
- Are the resource limits realistic for the workload?

</details>

<details>
<summary>Solution</summary>

```bash
# 1. Apply
kubectl apply -f challenges/challenge-debug-pod.yaml

# 2. Check status
kubectl get pods debug-challenge
# Shows: CrashLoopBackOff or OOMKilled

# 3. Diagnose
kubectl describe pod debug-challenge
# Look at "Last State" - shows OOMKilled
# The container's memory limit is 4Mi - too small for Python

kubectl logs debug-challenge
# May show nothing (OOMKilled happens before output)

# Root cause: memory limit of 4Mi is impossibly low.
# Python itself needs ~30MB to start.

# 4. Fix: edit the manifest to increase memory limits
# Change limits.memory from "4Mi" to "128Mi"
# Change requests.memory from "2Mi" to "64Mi"

kubectl delete pod debug-challenge
# Edit challenges/challenge-debug-pod.yaml:
#   memory limits: "4Mi" -> "128Mi"
#   memory requests: "2Mi" -> "64Mi"
kubectl apply -f challenges/challenge-debug-pod.yaml

# Key SRE insight:
# - OOMKilled is one of the most common Pod failures
# - Always check "Last State" in describe for the termination reason
# - Memory limits should account for the runtime, not just the app
# - Use 'kubectl top pod' to understand actual memory usage
```

</details>

### Challenge 2: Graceful Shutdown

**Scenario**: Your app takes 30 seconds to drain connections on SIGTERM. Users report 502 errors during deployments. Design a Pod spec that ensures zero-downtime shutdown.

**Tasks**:
1. Understand the default termination flow (SIGTERM -> 30s grace -> SIGKILL)
2. Deploy the challenge Pod with a preStop hook and proper grace period
3. Verify that the Pod drains gracefully when deleted
4. Explain why a readiness probe is critical during shutdown

```bash
kubectl apply -f challenges/challenge-graceful-shutdown.yaml
```

<details>
<summary>Hints</summary>

- `terminationGracePeriodSeconds` controls SIGTERM-to-SIGKILL timeout
- `preStop` hooks run BEFORE SIGTERM is sent
- Readiness probe failure removes the Pod from Service endpoints
- There's a race condition between endpoint removal and SIGTERM

</details>

<details>
<summary>Solution</summary>

```bash
# 1. The default termination flow:
#    a. Pod enters Terminating state
#    b. Endpoints controller removes Pod from Service (async!)
#    c. preStop hook runs (if defined)
#    d. SIGTERM sent to PID 1
#    e. Grace period timer starts (default 30s)
#    f. After grace period, SIGKILL
#
#    THE BUG: steps (b) and (c+d) happen IN PARALLEL.
#    Traffic can arrive AFTER SIGTERM if endpoint removal is slow.

# 2. Deploy
kubectl apply -f challenges/challenge-graceful-shutdown.yaml

# 3. Delete and observe graceful drain
kubectl delete pod graceful-pod &
kubectl logs graceful-pod -f
# You should see:
# - preStop hook fires (sleep 5 to allow endpoint removal)
# - SIGTERM received
# - Application drains for up to 30s
# - Pod terminates cleanly

# 4. Why readiness probe matters during shutdown:
# When the Pod starts terminating:
# - The readiness probe should fail (app stops accepting new connections)
# - This causes endpoint removal, stopping NEW traffic
# - The preStop hook adds a delay so in-flight requests complete
# - Without readiness probe: traffic keeps arriving during shutdown = 502s

# Key SRE insight:
# terminationGracePeriodSeconds MUST be > preStop delay + drain time
# Our config: grace=60s, preStop=5s, drain=30s → 35s < 60s ✓
# If grace < preStop + drain, kubelet sends SIGKILL mid-drain

# Clean up
kubectl delete -f challenges/challenge-graceful-shutdown.yaml
```

</details>

### Challenge 3: Resource Right-Sizing

**Scenario**: The following Pods have been running in production. Analyze their resource usage patterns and recommend better resource configurations.

**Tasks**:
1. Deploy the over-provisioned and under-provisioned Pods
2. Check their QoS classes
3. Use `kubectl top` to see actual usage vs configured requests/limits
4. Recommend corrected resource values and explain why

```bash
# Requires metrics-server: minikube addons enable metrics-server
kubectl apply -f challenges/challenge-resource-sizing.yaml
```

<details>
<summary>Hints</summary>

- Use `kubectl top pod` to see actual CPU/memory usage
- Compare actual usage against requests and limits
- Consider: should you use Guaranteed or Burstable QoS?
- What's the blast radius of each misconfiguration?

</details>

<details>
<summary>Solution</summary>

```bash
# 1. Deploy
kubectl apply -f challenges/challenge-resource-sizing.yaml

# Wait for Pods to stabilize
sleep 30

# 2. Check QoS
kubectl get pods -l experiment=sizing -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

# 3. Actual usage (requires metrics-server)
kubectl top pod -l experiment=sizing

# 4. Analysis:
#
# over-provisioned-pod:
#   Configured: 2 CPU / 2Gi memory
#   Actual:     ~5m CPU / ~5Mi memory (nginx idle)
#   Problem:    Wasting 99.7% of requested resources
#   Fix:        requests: 50m CPU, 64Mi memory
#               limits: 100m CPU, 128Mi memory
#   Impact:     Cluster can fit 20x more Pods
#
# under-provisioned-pod:
#   Configured: 1m CPU / 4Mi memory
#   Actual:     Will OOMKill or CPU throttle
#   Problem:    Will be OOMKilled or extremely slow
#   Fix:        requests: 50m CPU, 64Mi memory
#               limits: 100m CPU, 128Mi memory
#
# no-resources-pod:
#   Configured: nothing (BestEffort QoS)
#   Problem:    First to be evicted under node pressure
#   Fix:        Add requests = limits for Guaranteed QoS

# Key SRE insight:
# - Over-provisioning wastes money and prevents bin packing
# - Under-provisioning causes OOMKills and CPU throttling
# - BestEffort = production incident waiting to happen
# - Use VPA (Vertical Pod Autoscaler) recommendations in production
# - Rule of thumb: set requests to P99 usage, limits to 2x requests

# Clean up
kubectl delete -f challenges/challenge-resource-sizing.yaml
```

</details>

### Challenge 4: Pod Troubleshooting Gauntlet

**Scenario**: Four Pods are deployed, each with a different problem. Diagnose and explain each issue without looking at the manifest first.

**Tasks**:
1. Apply all four broken Pods
2. For each Pod, identify: the symptom, the root cause, and the fix
3. Do NOT read the YAML first - use only `kubectl` commands to diagnose

```bash
kubectl apply -f challenges/challenge-troubleshoot.yaml
```

<details>
<summary>Solution</summary>

```bash
kubectl apply -f challenges/challenge-troubleshoot.yaml

# Pod 1: broken-image
kubectl describe pod broken-image | grep -A 5 Events
# Symptom: ImagePullBackOff
# Cause: Image "nginx:nonexistent-version" doesn't exist
# Fix: Use a valid image tag (nginx:1.27)

# Pod 2: broken-command
kubectl logs broken-command
kubectl describe pod broken-command
# Symptom: CrashLoopBackOff
# Cause: Invalid command "/bin/start-server" doesn't exist in the image
# Fix: Use a valid entrypoint or command

# Pod 3: broken-config
kubectl describe pod broken-config | grep -A 5 Events
# Symptom: Pending → CreateContainerConfigError
# Cause: References a ConfigMap "nonexistent-config" that doesn't exist
# Fix: Create the ConfigMap or remove the reference

# Pod 4: broken-resources
kubectl describe pod broken-resources
# Symptom: Pending (Unschedulable)
# Cause: Requests 100 CPU cores - exceeds any node's capacity
# Fix: Reduce CPU request to a realistic value

# Key SRE insight:
# Common diagnostic flow:
# 1. kubectl get pod → what phase/status?
# 2. kubectl describe pod → events, conditions, state
# 3. kubectl logs [--previous] → application output
# 4. kubectl get events --sort-by=.lastTimestamp → cluster context
# 5. kubectl exec -it → inspect filesystem, network, processes

# Clean up
kubectl delete -f challenges/challenge-troubleshoot.yaml
```

</details>

### Challenge 5: Design a Production-Ready Pod Spec

**Scenario**: Write a complete production-grade Pod spec for a Java Spring Boot application that:
- Starts slowly (up to 90 seconds)
- Serves HTTP on port 8080 with a `/actuator/health` endpoint
- Needs 512MB-1GB of memory (JVM heap)
- Requires a database password from a Secret
- Must run as non-root with a read-only filesystem
- Needs a writable `/tmp` directory
- Must drain connections gracefully on shutdown (takes 15 seconds)

**Tasks**:
1. Write the complete Pod spec from scratch
2. Include all appropriate probes, security settings, and resource configuration
3. Justify each configuration choice

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spring-boot-app
  labels:
    app: spring-boot
spec:
  terminationGracePeriodSeconds: 45  # 5s preStop + 15s drain + buffer
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: my-spring-app:1.0
      ports:
        - containerPort: 8080
          name: http
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      resources:
        requests:
          cpu: "250m"        # Typical idle Spring Boot
          memory: "768Mi"    # JVM heap + metaspace + overhead
        limits:
          cpu: "1000m"       # Allow burst during startup
          memory: "1Gi"      # Hard cap, prevents OOM cascade
      # Startup probe: allow 90s for JVM warmup
      # failureThreshold(30) * periodSeconds(3) = 90s budget
      startupProbe:
        httpGet:
          path: /actuator/health
          port: 8080
        failureThreshold: 30
        periodSeconds: 3
      # Liveness: is the app alive? (don't depend on DB!)
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        periodSeconds: 10
        failureThreshold: 3
      # Readiness: can it serve traffic?
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        periodSeconds: 5
        failureThreshold: 2
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]  # Allow endpoint removal
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

Configuration rationale:
- **startupProbe**: Blocks liveness/readiness for up to 90s (30 * 3s)
- **livenessProbe**: Uses `/liveness` NOT `/health` (avoids DB dependency cascade)
- **readinessProbe**: More frequent (5s) to quickly remove/add from endpoints
- **preStop sleep 5**: Race condition fix between SIGTERM and endpoint removal
- **terminationGracePeriodSeconds 45**: 5s preStop + 15s drain + 25s safety buffer
- **Burstable QoS**: Intentional - CPU burst during startup, memory limit set
- **readOnlyRootFilesystem + emptyDir /tmp**: Security + JVM temp files need writable dir
- **drop ALL capabilities**: Minimum privilege

</details>

## Key Takeaways

- Pods are groups of Linux namespaces, not just "containers"
- The pause container holds the network namespace and reaps zombies
- **QoS classes** (Guaranteed > Burstable > BestEffort) determine eviction order
- **Always set resource requests/limits** - BestEffort in production is a liability
- Use **startupProbe** for slow-starting apps, **liveness** for deadlock detection
- Never make livenessProbe depend on external services (cascading failure risk)
- **preStop hooks** solve the SIGTERM/endpoint-removal race condition
- `terminationGracePeriodSeconds` must exceed preStop + drain time
- **CrashLoopBackOff** backs off exponentially up to 5 minutes
- Run containers as non-root with read-only filesystem and dropped capabilities

## Next Module

Proceed to [03-replicasets](../03-replicasets/) to learn about maintaining multiple Pod replicas.
