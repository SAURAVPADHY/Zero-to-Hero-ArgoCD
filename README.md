# Zero-to-Hero ArgoCD

This repository provides a beginner-friendly ArgoCD setup and installation guide for Kubernetes environments. The goal is to help you install ArgoCD, access the UI, bootstrap an application, and understand the basic workflow for GitOps deployments.

## Prerequisites

Before installing ArgoCD, make sure you have:

- A running Kubernetes cluster
- `kubectl` installed and configured
- Cluster access with admin privileges
- Optional: `helm` for additional package management

Common local cluster options:

- `kind`
- `minikube`
- `k3d`
- A managed Kubernetes cluster such as AKS, EKS, or GKE

## 1. Create or Start a Kubernetes Cluster

If you do not already have a cluster, create one locally.

### Example with kind

```bash
kind create cluster --name argocd-demo
```

Verify the cluster is ready:

```bash
kubectl cluster-info
kubectl get nodes
```

## 2. Install ArgoCD

Create the ArgoCD namespace and apply the official installation manifest:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check whether the ArgoCD pods are running:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

You should see the `argocd-server`, `argocd-repo-server`, and other ArgoCD components in the running state.

## 3. Access the ArgoCD UI

Expose the ArgoCD API server locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open the following URL in your browser:

```text
https://localhost:8080
```

### Login Credentials

Username:

```text
admin
```

Password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> The initial admin password is only shown once at installation time. If it is lost, recreate the secret or reset the password.

## 4. Change the Admin Password

After logging in, update the default password to a secure value:

```bash
argocd login localhost:8080 --insecure --username admin
argocd account update-password --account admin
```

This will prompt you to enter a new password.

## 5. Connect ArgoCD to Your Cluster

If ArgoCD is installed in the same cluster and you want it to manage that cluster, the in-cluster configuration usually works automatically.

To view the current cluster context:

```bash
kubectl config current-context
```

If you want ArgoCD to manage external clusters, use the CLI:

```bash
argocd cluster add $(kubectl config current-context)
```

## 6. Create Your First Application

ArgoCD deploys applications from Git repositories. A simple example is to deploy the guestbook app from the official ArgoCD examples repository.

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Sync the application:

```bash
argocd app sync guestbook
```

Check the application status:

```bash
argocd app get guestbook
```

## 7. Useful Kubernetes and ArgoCD Commands

Check ArgoCD pods:

```bash
kubectl get pods -n argocd
```

Check ArgoCD services:

```bash
kubectl get svc -n argocd
```

View ArgoCD server logs:

```bash
kubectl logs -n argocd deploy/argocd-server
```

Check application status:

```bash
kubectl get applications -A
```

## 8. Troubleshooting

### ArgoCD pods are not ready

```bash
kubectl get pods -n argocd
kubectl describe pod <pod-name> -n argocd
```

### Cannot access the ArgoCD UI

- Ensure the port-forward command is still active
- Use the correct port (`8080` in the example)
- Check that the `argocd-server` service is running

### Forgot the admin password

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
kubectl -n argocd get pods
```

Then recreate the ArgoCD installation or reset the admin credentials using the ArgoCD commands.

## 9. Uninstall ArgoCD

If you want to remove ArgoCD from the cluster:

```bash
kubectl delete namespace argocd
```

## Summary

This installation gives you a working ArgoCD server, dashboard access, and a basic GitOps workflow. From here, you can connect repositories, define applications, and automate deployments using Git as the single source of truth.

## Next Steps

- Add a Git repository with Kubernetes manifests
- Create a sample app in ArgoCD
- Learn about sync policies, health checks, and project configuration
- Explore ArgoCD application and project manifests for production use
