# Runbook: `checkout-api` OOMKilled Incident

## Purpose

Use this runbook when `checkout-api` pods are repeatedly restarting and there is evidence that containers are being terminated due to exceeding their memory limits.

**Primary objective:** Confirm whether the restarts are caused by an OOM kill, determine whether the configured memory limit is realistic, and restore the service to a known-good configuration.

---

## 1. Diagnosis

### Step 1: Check pod status and restart counts

Start by checking the `checkout-api` pods:

```bash
kubectl get pods -l app=checkout-api
```

Look for:

- Increasing `RESTARTS` counts
- Pods repeatedly transitioning between `Running` and `CrashLoopBackOff`
- Pods that have recently restarted

For more detail:

```bash
kubectl get pods -l app=checkout-api -o wide
```

If you don't know the label used by the deployment:

```bash
kubectl get pods
```

---

### Step 2: Confirm `OOMKilled`

Select an affected pod:

```bash
kubectl describe pod <checkout-api-pod>
```

Under the container's **Last State**, look for:

```text
Last State:
  Terminated:
    Reason: OOMKilled
```

You can also query the termination reason directly:

```bash
kubectl get pod <checkout-api-pod> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

Expected result:

```text
OOMKilled
```

If the result is `OOMKilled`, continue with this runbook.

---

### Step 3: Check the current memory limit

Inspect the deployment:

```bash
kubectl get deployment checkout-api -o yaml
```

Find:

```yaml
resources:
  limits:
    memory: ...
```

A more targeted command:

```bash
kubectl get deployment checkout-api \
  -o jsonpath='{.spec.template.spec.containers[*].resources.limits.memory}'
```

For example:

```text
8Mi
```

If the limit is unexpectedly low compared with the application's normal requirements, this is a strong indication that the resource configuration is responsible.

---

### Step 4: Check current memory usage

If the metrics server is available:

```bash
kubectl top pods -l app=checkout-api
```

For a specific pod:

```bash
kubectl top pod <checkout-api-pod>
```

Example:

```text
NAME                            CPU(cores)   MEMORY(bytes)
checkout-api-7d8f9c6d4b-abc12   15m          72Mi
```

Compare the observed usage against the configured limit.

For example:

```text
Observed usage: 72Mi
Memory limit:    8Mi
```

This indicates the pod requires substantially more memory than its configured limit.

> **Important:** `kubectl top` shows current usage, not necessarily the peak usage that triggered the OOM kill. Use it as supporting evidence rather than proof that the container never exceeded its limit.

---

### Step 5: Check recent configuration changes

Inspect the deployment's rollout history:

```bash
kubectl rollout history deployment/checkout-api
```

Check the current deployment:

```bash
kubectl describe deployment checkout-api
```

Look for recent changes to:

```yaml
resources:
  requests:
    memory: ...
  limits:
    memory: ...
```

If the incident followed a resource configuration change, compare the current value with the previous known-good configuration.

You can also inspect a specific rollout revision:

```bash
kubectl rollout history deployment/checkout-api --revision=<revision>
```

---

### Step 6: Check application logs

Inspect the previous container instance:

```bash
kubectl logs <checkout-api-pod> --previous
```

Also check the current container:

```bash
kubectl logs <checkout-api-pod>
```

An OOMKilled container may not produce useful application-level errors because the process can be terminated by the kernel before the application has an opportunity to log anything.

---

## 2. Diagnosis Decision

### Confirmed OOM due to memory limit

Evidence:

```text
Last State: Terminated
Reason: OOMKilled
```

and:

```bash
kubectl get deployment checkout-api ...
```

shows an unexpectedly low memory limit.

**Proceed to Resolution.**

### OOMKilled but limit appears reasonable

Do **not** immediately increase the limit without investigation.

Check:

```bash
kubectl top pods -l app=checkout-api
kubectl logs <checkout-api-pod> --previous
kubectl describe pod <checkout-api-pod>
```

Investigate whether there was:

- An unusual traffic spike
- A workload change
- A memory leak
- A large request or response
- Another resource-related issue

### Restarts are not `OOMKilled`

Do not use this runbook as the primary remediation path. Check the actual termination reason:

```bash
kubectl get pod <checkout-api-pod> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

Other termination reasons may require a different investigation.

---

# 3. Resolution

## Step 1: Identify the known-good memory limit

For this incident, the known-good value is:

```text
128Mi
```

Before changing anything, verify the current configuration:

```bash
kubectl get deployment checkout-api \
  -o jsonpath='{.spec.template.spec.containers[*].resources.limits.memory}'
```

If it shows an unrealistic value such as:

```text
8Mi
```

restore the known-good limit.

---

## Step 2: Revert the memory limit

If the deployment is managed directly with `kubectl`, edit it:

```bash
kubectl edit deployment checkout-api
```

Change:

```yaml
resources:
  limits:
    memory: 8Mi
```

to:

```yaml
resources:
  limits:
    memory: 128Mi
```

Save and exit.

Alternatively, if the deployment configuration is managed through a manifest, update the manifest and apply it:

```bash
kubectl apply -f checkout-api.yaml
```

If the application is managed by Helm, make the change in the Helm values instead of modifying the live Deployment directly.

---

## Step 3: Monitor the rollout

Check rollout status:

```bash
kubectl rollout status deployment/checkout-api
```

Then inspect the pods:

```bash
kubectl get pods -l app=checkout-api
```

Watch for the restart count to stabilize:

```bash
kubectl get pods -l app=checkout-api -w
```

---

## Step 4: Verify memory usage and stability

Once the new pods are running:

```bash
kubectl top pods -l app=checkout-api
```

Confirm that normal memory usage is comfortably below:

```text
128Mi
```

Also check the pod's recent events:

```bash
kubectl describe pod <new-checkout-api-pod>
```

And verify that containers are healthy:

```bash
kubectl get pods -l app=checkout-api
```

Expected state:

```text
READY   STATUS    RESTARTS
1/1     Running   0
```

A nonzero restart count on a newly recreated pod isn't necessarily a problem, but **the count should stop increasing**.

---

## Step 5: Verify application functionality

Check the application's health endpoint if one exists:

```bash
kubectl exec <checkout-api-pod> -- curl -f http://localhost:<port>/health
```

Then verify application-level request success through the normal service/ingress path.

The key indicators of resolution are:

- Pods remain `Running`
- `RESTARTS` stops increasing
- No new `OOMKilled` events
- Memory usage remains within the configured limit
- Checkout requests succeed consistently

---

# 4. If the Problem Persists

If pods continue being OOMKilled after restoring `128Mi`:

```bash
kubectl get pods -l app=checkout-api
kubectl describe pod <checkout-api-pod>
kubectl top pods -l app=checkout-api
kubectl logs <checkout-api-pod> --previous
```

Check whether the application is actually using more than `128Mi`:

```text
Observed memory > 128Mi
```

If so, **do not repeatedly increase the limit without understanding why**. Investigate workload changes, traffic levels, and application memory behavior.

Also check whether the pod is subject to namespace-level constraints:

```bash
kubectl get resourcequota
kubectl get limitrange
```

---

# 5. Post-Incident Checks

After service is stable:

### Record the incident evidence

Capture:

```bash
kubectl get pods -l app=checkout-api
kubectl describe pod <affected-pod>
kubectl get deployment checkout-api -o yaml
kubectl top pods -l app=checkout-api
```

### Identify how the bad configuration was introduced

Determine whether the `8Mi` limit came from:

- A Git/manifest change
- Helm values
- A CI/CD deployment
- A direct `kubectl` modification

The resource limit should be corrected at its **source of truth**, not just in the live cluster.

### Validate future resource changes

Before lowering the memory limit, compare the proposed value against observed workload usage.

For example:

```text
Current observed usage:  ~70Mi
Proposed limit:            8Mi
```

This should be treated as an obviously unsafe configuration change.

---

# 6. Quick Reference

For an immediate `checkout-api` OOM incident:

```bash
# 1. Find affected pods
kubectl get pods -l app=checkout-api

# 2. Confirm OOMKilled
kubectl describe pod <pod>

# 3. Check current memory limit
kubectl get deployment checkout-api \
  -o jsonpath='{.spec.template.spec.containers[*].resources.limits.memory}'

# 4. Check current memory usage
kubectl top pods -l app=checkout-api

# 5. Check previous container logs
kubectl logs <pod> --previous

# 6. Restore known-good limit
kubectl edit deployment checkout-api
# Set memory limit to 128Mi

# 7. Wait for rollout
kubectl rollout status deployment/checkout-api

# 8. Verify stability
kubectl get pods -l app=checkout-api -w

# 9. Verify memory usage
kubectl top pods -l app=checkout-api
```

**Incident-specific known-good configuration:** `checkout-api` memory limit = **128Mi**. The postmortem indicates that reverting from `8Mi` to `128Mi` resolved the incident.
