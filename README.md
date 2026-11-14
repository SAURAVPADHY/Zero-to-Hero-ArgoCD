# 🚀 Zero-to-Hero ArgoCD

Welcome to the Argo CD installation guide for Kubernetes beginners and hands-on learners.

This repository walks you through the official Argo CD setup process so you can install Argo CD, access the dashboard, and start managing your workloads with GitOps.

## 🌟 Why Argo CD?

Argo CD is a declarative GitOps continuous delivery tool for Kubernetes. It watches your Git repository and ensures the live cluster state matches the desired state described in your manifests.

In simple terms:

- Push code or YAML to Git
- Argo CD detects the change
- Argo CD syncs the cluster to match the repository
- Your application stays consistent and automated

---

## ✅ Prerequisites

Before installing Argo CD, make sure you have:

- A running Kubernetes cluster
- `kubectl` installed and configured
- Cluster access with enough privileges
- Optional: `argocd` CLI for managing applications

### Local cluster options

You can install Argo CD on any Kubernetes environment, including:

- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [k3d](https://k3d.io/stable/)
- Managed clusters such as EKS, AKS, or GKE

### Check your cluster

```bash
kubectl cluster-info
kubectl get nodes
```

If both commands work, your cluster is ready for Argo CD.

---

## 🛠️ Install Argo CD

Create the namespace and apply the official Argo CD installation manifest:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify the installation:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

You should see Argo CD components such as:

- `argocd-server`
- `argocd-repo-server`
- `argocd-application-controller`
- `argocd-dex-server` (if enabled)

Wait for the server to become available:

```bash
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=600s
```

---

## 🌐 Access the Argo CD UI

Port-forward the Argo CD server to your local machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open this URL in your browser:

```text
https://localhost:8080
```

---

## 🔐 Login Credentials

The default username is:

```text
admin
```

Get the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> This password is generated during installation and should be changed after your first login.

---

## 🔄 Change the Admin Password

Once you log in, update the default password:

```bash
argocd login localhost:8080 --insecure --username admin
argocd account update-password --account admin
```

Follow the CLI prompt to enter a new password.

---

## 📦 Add Cluster Access

If Argo CD is installed in the same Kubernetes cluster it manages, the in-cluster configuration is typically available automatically.

Check the current context:

```bash
kubectl config current-context
```

If you want to add the cluster explicitly:

```bash
argocd cluster add $(kubectl config current-context)
```

---

## 🚀 Deploy Your First Application

The Argo CD example repository includes a simple guestbook application for validation.

Create the application:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Sync the app:

```bash
argocd app sync guestbook
```

Check the status:

```bash
argocd app get guestbook
```

Once synced, you can also view the result in the Argo CD UI.

---

## 🧪 Useful Commands

Check installation resources:

```bash
kubectl get all -n argocd
kubectl get pods -n argocd
kubectl get svc -n argocd
```

View the Argo CD server logs:

```bash
kubectl logs -n argocd deploy/argocd-server
```

List applications:

```bash
kubectl get applications -A
```

---

## ⚠️ Troubleshooting

### Pods are not ready

```bash
kubectl get pods -n argocd
kubectl describe pod <pod-name> -n argocd
```

### Cannot access the UI

- Confirm the port-forward command is still running
- Ensure the `argocd-server` service is healthy
- Check whether port `8080` is blocked by another process

### Lost the admin password

If you forget the initial password, remove the secret and reset the credentials:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 🧹 Uninstall Argo CD

To remove Argo CD from the cluster:

```bash
kubectl delete namespace argocd
```

---

## 🎯 Summary

This installation gives you a working Argo CD deployment with a user-friendly UI and a solid GitOps foundation for Kubernetes automation.

From here, you can connect Git repositories, define applications, and manage deployments declaratively.

## 🔜 Next Steps

- Connect a Git repository with Kubernetes manifests
- Create your own Argo CD application definition
- Learn about sync policies and health checks
- Explore Argo CD projects, applications, and repository management

---

## 🔗 Helpful References

- [Argo CD Official Documentation](https://argo-cd.readthedocs.io/)
- [Argo CD Installation Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [kind Documentation](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [minikube Documentation](https://minikube.sigs.k8s.io/docs/start/)
- [k3d Documentation](https://k3d.io/stable/)
