# Complete ArgoCD Installation and Practice Guide for Beginners

## What is ArgoCD?

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes. It automatically synchronizes your Kubernetes cluster with configuration stored in Git repositories, making deployments easier and more reliable.

## Prerequisites

Before starting, ensure you have:
- A running Kubernetes cluster (minikube, kind, or any cloud provider)
- kubectl installed and configured
- Basic understanding of Kubernetes concepts
- Git installed on your machine

## Part 1: Installing ArgoCD

### Step 1: Create ArgoCD Namespace

First, create a dedicated namespace for ArgoCD:

```bash
kubectl create namespace argocd
```

### Step 2: Install ArgoCD

Install ArgoCD using the official manifest:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This command installs all ArgoCD components including the API server, repository server, and application controller.

### Step 3: Wait for Pods to be Ready

Check if all ArgoCD pods are running:

```bash
kubectl get pods -n argocd
```

Wait until all pods show STATUS as "Running". This may take 2-5 minutes.

```bash
# Watch pods in real-time
kubectl get pods -n argocd -w
```

Press Ctrl+C to stop watching once all pods are running.

## Part 2: Exposing ArgoCD with NodePort

By default, ArgoCD server uses a ClusterIP service. We'll change it to NodePort to access it from outside the cluster.

### Step 1: Edit the ArgoCD Server Service

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

### Step 2: Get the NodePort

Find the assigned NodePort:

```bash
kubectl get svc argocd-server -n argocd
```

Look for the PORT(S) column. You'll see something like:
```
443:32000/TCP,80:31000/TCP
```

The number after the colon (e.g., 32000) is your NodePort for HTTPS.

### Step 3: Get Your Node IP

**For Minikube:**
```bash
minikube ip
```

**For other clusters:**
```bash
kubectl get nodes -o wide
```

Look at the EXTERNAL-IP or INTERNAL-IP column.

### Step 4: Access ArgoCD UI

Open your browser and navigate to:
```
https://<NODE_IP>:<HTTPS_NODEPORT>
```

For example: `https://192.168.49.2:32000`

**Note:** You'll see a security warning because ArgoCD uses a self-signed certificate. Click "Advanced" and proceed to the site.

## Part 3: Getting the Initial Admin Password

### Method 1: Using kubectl (Recommended)

The initial admin password is stored in a Kubernetes secret. Retrieve it using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

This command will output the password. Copy it.

### Method 2: For Windows PowerShell

```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

### Step 5: Login to ArgoCD

- **Username:** `admin`
- **Password:** (the password you just retrieved)

## Part 4: Installing ArgoCD CLI (Optional but Recommended)

The ArgoCD CLI makes managing applications easier.

### For Linux:

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

### For macOS:

```bash
brew install argocd
```

### For Windows:

Download from: https://github.com/argoproj/argo-cd/releases/latest

### Login via CLI

```bash
argocd login <NODE_IP>:<NODEPORT> --username admin --password <your-password> --insecure
```

## Part 5: Practice Projects for Beginners

### Practice 1: Deploy Your First Application (Guestbook)

This is ArgoCD's "Hello World" application.

#### Step 1: Create Application via UI

1. Click "New App" in the ArgoCD UI
2. Fill in the details:
   - **Application Name:** guestbook
   - **Project:** default
   - **Sync Policy:** Manual
   - **Repository URL:** https://github.com/argoproj/argocd-example-apps.git
   - **Revision:** HEAD
   - **Path:** guestbook
   - **Cluster URL:** https://kubernetes.default.svc
   - **Namespace:** default

3. Click "Create"

#### Step 2: Sync the Application

1. Click on your "guestbook" application
2. Click "Sync" button
3. Click "Synchronize"

Watch as ArgoCD deploys the application!

#### Step 3: Verify Deployment

```bash
kubectl get all -n default | grep guestbook
```

#### Step 4: Access the Guestbook

```bash
kubectl port-forward svc/guestbook-ui 8080:80
```

Open http://localhost:8080 in your browser.

### Practice 2: Deploy Using CLI

Create an application using the ArgoCD CLI:

```bash
argocd app create helm-guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path helm-guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Sync it:

```bash
argocd app sync helm-guestbook
```

### Practice 3: Automated Sync

Create an app with automatic synchronization:

```bash
argocd app create nginx-app \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

This application will automatically sync when changes are detected in Git.

### Practice 4: Create Your Own GitOps Repo

#### Step 1: Create a GitHub Repository

1. Go to GitHub and create a new repository called "my-argocd-apps"
2. Clone it locally:

```bash
git clone https://github.com/<your-username>/my-argocd-apps.git
cd my-argocd-apps
```

#### Step 2: Create a Simple Deployment

Create a file `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
---
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

#### Step 3: Push to GitHub

```bash
git add .
git commit -m "Add nginx deployment"
git push origin main
```

#### Step 4: Create ArgoCD Application

```bash
argocd app create my-nginx \
  --repo https://github.com/<your-username>/my-argocd-apps.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

#### Step 5: Test GitOps Workflow

1. Edit `nginx-deployment.yaml` and change `replicas: 2` to `replicas: 3`
2. Commit and push:

```bash
git add .
git commit -m "Scale to 3 replicas"
git push origin main
```

3. Watch ArgoCD automatically sync the changes:

```bash
argocd app get my-nginx --watch
```

## Part 6: Essential ArgoCD Commands

### Application Management

```bash
# List all applications
argocd app list

# Get application details
argocd app get <app-name>

# Sync an application
argocd app sync <app-name>

# Delete an application
argocd app delete <app-name>

# View application logs
argocd app logs <app-name>

# View application history
argocd app history <app-name>

# Rollback to previous version
argocd app rollback <app-name> <revision-number>
```

### Monitoring

```bash
# Watch application sync status
argocd app wait <app-name>

# Get sync status
argocd app get <app-name> --refresh
```

## Part 7: Troubleshooting Common Issues

### Issue 1: Can't Access ArgoCD UI

**Solution:**
- Check if pods are running: `kubectl get pods -n argocd`
- Verify NodePort: `kubectl get svc argocd-server -n argocd`
- Check firewall rules on your node

### Issue 2: Application Stuck in "Progressing"

**Solution:**
```bash
# Check application details
argocd app get <app-name>

# Look at events
kubectl get events -n <namespace>
```

### Issue 3: Sync Failed

**Solution:**
- Check if the Git repository is accessible
- Verify the path in the application configuration
- Check ArgoCD server logs: `kubectl logs -n argocd deployment/argocd-server`

## Part 8: Best Practices for Learning

### Week 1: Basics
- Install ArgoCD and deploy the guestbook example
- Practice with CLI commands
- Explore the UI thoroughly

### Week 2: GitOps Workflow
- Create your own Git repository
- Deploy applications from your repo
- Practice making changes and syncing

### Week 3: Advanced Features
- Experiment with sync policies (manual vs. automated)
- Try different sync options (prune, self-heal)
- Practice rollbacks

### Week 4: Real Projects
- Deploy a multi-tier application (frontend + backend + database)
- Set up different environments (dev, staging, prod)
- Implement proper Git branching strategy

## Part 9: Useful Resources

- **Official Documentation:** https://argo-cd.readthedocs.io/
- **Example Applications:** https://github.com/argoproj/argocd-example-apps
- **Community:** ArgoCD Slack channel
- **Video Tutorials:** Search for "ArgoCD tutorial" on YouTube

## Part 10: Security Tips

### Change Default Password

After first login, change the admin password:

```bash
argocd account update-password
```

### Delete Initial Secret

After changing the password, delete the initial secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### Create Additional Users

Edit the ConfigMap to add more users:

```bash
kubectl edit configmap argocd-cm -n argocd
```

## Quick Reference Card

```
# Installation
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get Password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Expose with NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get NodePort
kubectl get svc argocd-server -n argocd

# Create App
argocd app create <name> --repo <url> --path <path> --dest-server https://kubernetes.default.svc --dest-namespace <namespace>

# Sync App
argocd app sync <name>

# Delete App
argocd app delete <name>
```

## Conclusion

ArgoCD is a powerful tool that simplifies Kubernetes deployments through GitOps principles. Start with the simple examples, gradually move to more complex scenarios, and practice regularly. The key to mastering ArgoCD is understanding the GitOps workflow: your Git repository is the single source of truth, and ArgoCD ensures your cluster always matches what's in Git.

Happy learning!