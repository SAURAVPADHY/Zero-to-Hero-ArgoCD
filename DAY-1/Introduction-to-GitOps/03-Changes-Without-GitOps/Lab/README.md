# Additional Practical Lab — Change a Deployment by Hand

> **Purpose:** Create configuration drift in the `demo` namespace, inspect Kubernetes `managedFields`, and save the manifest-vs-live difference.  
> **Important:** This lab intentionally leaves the Deployment drifted. Do **not** re-apply `web.yaml` at the end.

## Task outcome

You will create these files:

| File | Purpose |
|---|---|
| `/root/gitops-intro/web.yaml` | Original desired state: 2 replicas, `nginx:1.27-alpine` |
| `/root/gitops-intro/live.yaml` | Live Deployment exported with `managedFields` |
| `/root/gitops-intro/drift.txt` | Difference reported by `kubectl diff` |

At the end, the Deployment should have **5 replicas** and image **`nginx:1.26-alpine`**, while `web.yaml` still says **2 replicas** and **`nginx:1.27-alpine`**.

## Step 0 — Confirm your Kubernetes context

Use a lab or disposable cluster, not production.

```bash
kubectl config current-context
kubectl get nodes
```

Make sure the context is the cluster where you intend to do this exercise.

## Step 1 — Create the working directory

The task uses `/root`, so run these commands as root or use an environment where you have permission to write there.

```bash
mkdir -p /root/gitops-intro
cd /root/gitops-intro
```

## Step 2 — Create the `demo` namespace

```bash
kubectl create namespace demo
```

If the namespace already exists, that is fine; verify it:

```bash
kubectl get namespace demo
```

## Step 3 — Write the original desired-state file

Create `/root/gitops-intro/web.yaml` with exactly this content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```

Apply the file:

```bash
kubectl apply -f /root/gitops-intro/web.yaml
```

Verify the initial Deployment:

```bash
kubectl get deployment web -n demo
kubectl get pods -n demo -l app=web
kubectl get deployment web -n demo \
  -o jsonpath='{.spec.replicas}{" replicas; image: "}{.spec.template.spec.containers[0].image}{"\n"}'
```

**Expected initial state:** 2 replicas and `nginx:1.27-alpine`. Wait for the two pods to become Ready before continuing:

```bash
kubectl rollout status deployment/web -n demo
```

## Step 4 — Play the other DevOps engineer

**Do not edit `web.yaml`.** Change only the live Deployment using `kubectl`.

Scale it to five replicas:

```bash
kubectl scale deployment web -n demo --replicas=5
```

Change the container image:

```bash
kubectl set image deployment/web -n demo nginx=nginx:1.26-alpine
```

Check the live Deployment:

```bash
kubectl get deployment web -n demo
kubectl get deployment web -n demo \
  -o jsonpath='{.spec.replicas}{" replicas; image: "}{.spec.template.spec.containers[0].image}{"\n"}'
```

Wait for the updated rollout:

```bash
kubectl rollout status deployment/web -n demo
```

**Expected live state:** 5 replicas and `nginx:1.26-alpine`.

Now prove that the original file was not updated:

```bash
grep -nE 'replicas:|image:' /root/gitops-intro/web.yaml
```

It should still show:

```yaml
replicas: 2
image: nginx:1.27-alpine
```

## Step 5 — Save the live Deployment with managed fields

Export the live Deployment to the required file:

```bash
kubectl get deployment web -n demo \
  -o yaml --show-managed-fields \
  > /root/gitops-intro/live.yaml
```

Confirm the file exists and inspect the `managedFields` section:

```bash
test -s /root/gitops-intro/live.yaml && echo "live.yaml exists"
grep -n -A 35 'managedFields:' /root/gitops-intro/live.yaml
```

You may see manager names such as `kubectl-client-side-apply`, `kubectl-scale`, or `kubectl-set`, depending on the Kubernetes and kubectl versions and how the object was changed.

**What this proves:** `managedFields` records field-management information, including manager/tool and operation metadata. It does not reliably identify the human engineer who ran the command. It is not a replacement for an identity-aware audit log.

## Step 6 — Save the drift report

Run `kubectl diff` against the original manifest and save its output:

```bash
kubectl diff -f /root/gitops-intro/web.yaml \
  > /root/gitops-intro/drift.txt
```

**Note about the exit code:** `kubectl diff` returns exit code `1` when differences are found. That is expected for this lab; it does not mean the diff operation failed. If you want to capture and explain the result explicitly, use this Bash block instead (it overwrites `drift.txt` with the same report):

```bash
if kubectl diff -f /root/gitops-intro/web.yaml \
    > /root/gitops-intro/drift.txt; then
  echo "No differences found — check whether the manual changes were made."
else
  status=$?
  if [ "$status" -eq 1 ]; then
    echo "Differences found and saved to drift.txt (expected)."
  else
    echo "kubectl diff failed with exit code $status."
    exit "$status"
  fi
fi
```

Read the report:

```bash
cat /root/gitops-intro/drift.txt
```

Look for the differences in `spec.replicas` and the container image. `kubectl diff` compares the manifest with the live object; it does **not** apply the manifest or repair the Deployment.

## Step 7 — Final validation (do not repair the drift)

Run all these checks:

```bash
# Both required files must exist and be non-empty.
test -s /root/gitops-intro/live.yaml && echo "PASS: live.yaml exists"
test -s /root/gitops-intro/drift.txt && echo "PASS: drift.txt exists"

# The original desired-state file must still contain the old values.
grep -nE 'replicas:|image:' /root/gitops-intro/web.yaml

# The live Deployment must contain the manually changed values.
kubectl get deployment web -n demo \
  -o jsonpath='{.spec.replicas}{" replicas; image: "}{.spec.template.spec.containers[0].image}{"\n"}'

# Confirm the Deployment exists in the intended namespace.
kubectl get deployment web -n demo
```

### Expected final state

| Check | Expected value |
|---|---|
| `web.yaml` replicas | `2` |
| `web.yaml` image | `nginx:1.27-alpine` |
| Live Deployment replicas | `5` |
| Live Deployment image | `nginx:1.26-alpine` |
| `live.yaml` | Exists and contains `managedFields` |
| `drift.txt` | Exists and shows differences |

The two required output files are:

```text
/root/gitops-intro/
├── web.yaml
├── live.yaml
└── drift.txt
```

<details>
<summary><strong>What to submit</strong></summary>

Submit only after all of the following are true:

- [ ] `/root/gitops-intro/web.yaml` exists and contains the original desired state.
- [ ] `/root/gitops-intro/live.yaml` exists and includes `managedFields`.
- [ ] `/root/gitops-intro/drift.txt` exists and contains the diff output.
- [ ] The live Deployment in namespace `demo` has 5 replicas and image `nginx:1.26-alpine`.
- [ ] The original file still has 2 replicas and image `nginx:1.27-alpine`.

**Press Submit when both output files exist and the Deployment differs from the original file.**

</details>

## Step 8 — Leave the cluster drifted

Stop here. **Do not run**:

```bash
kubectl apply -f /root/gitops-intro/web.yaml
```

That command would move the live Deployment back toward the original manifest and undo the drift you were asked to demonstrate.

Leave the Deployment at five replicas with `nginx:1.26-alpine`. A later lesson will introduce Argo CD and show how a GitOps controller detects and handles this difference.

---

## Instructor note — what this lab demonstrates

This exercise reproduces the three problems from the lesson:

1. **Who changed it?** `managedFields` can identify a manager/tool, but not reliably the human.
2. **Nobody is told.** Without a controller or an external monitoring process, the manifest/live mismatch is not continuously reported by Kubernetes.
3. **Nothing fixes it.** No GitOps reconciler is configured in this exercise, so the live Deployment remains changed until an operator or another controller acts.

