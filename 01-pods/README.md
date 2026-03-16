# Module 1: Pods

## Concept

A **Pod** is the smallest deployable unit in Kubernetes. It represents a single instance of a running process and can contain one or more containers that share:

- Network namespace (same IP address, can communicate via localhost)
- Storage volumes
- Pod-level settings (e.g., security context)

Most commonly, a Pod runs a single container. Multi-container Pods are used for tightly coupled helper processes (sidecars, adapters, ambassadors).

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

## Key Takeaways

- Pods are ephemeral - they are not rescheduled, they are replaced
- Never create standalone Pods in production - use Deployments instead
- Each Pod gets a unique IP within the cluster
- Containers within a Pod share localhost
- Use `kubectl describe` and `kubectl logs` for debugging

## Next Module

Proceed to [02-replicasets](../02-replicasets/) to learn about maintaining multiple Pod replicas.
