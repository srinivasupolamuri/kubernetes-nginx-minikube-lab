# Kubernetes Hands-on: Nginx Deployment and Self-Healing

A beginner-friendly Kubernetes lab built with **Ubuntu on WSL, `kubectl`, Minikube, Docker, and Nginx**. The project documents the practical steps used to create a local Kubernetes cluster, deploy Nginx, access the application, and observe Kubernetes replacing Pods managed by a Deployment.

> **Project scope:** This is a local learning lab, not a production deployment. It does not claim cloud hosting, CI/CD automation, monitoring, or production hardening.

## What I practised

- Installed and verified `kubectl`.
- Started a local Kubernetes cluster using Minikube with the Docker driver.
- Created and inspected an Nginx Pod.
- Added a label to a Pod and learned why labels matter for Services.
- Tested Nginx from inside the cluster and from the host using port forwarding.
- Created a Deployment with three replicas using a YAML manifest.
- Watched Pod lifecycle states and deleted a Deployment-managed Pod to observe self-healing.
- Used `kubectl get`, `kubectl describe`, `kubectl logs`, `kubectl get events`, and `curl` for basic troubleshooting.

## Architecture

```text
Windows machine
└── Ubuntu on WSL
    ├── kubectl (Kubernetes CLI)
    └── Minikube cluster (Docker driver)
        ├── Nginx Pod / Deployment
        │   ├── Replica 1
        │   ├── Replica 2
        │   └── Replica 3
        └── Nginx Service (NodePort)
```

For local testing, `kubectl port-forward` creates a temporary connection from `localhost:8080` to port `80` in a Pod. The Pod IP is an internal cluster address and may not be directly reachable from the WSL host.

## Repository structure

```text
kubernetes-nginx-minikube-lab/
├── README.md
├── manifests/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── standalone-pod.yaml
└── docs/
    └── screenshots/
        ├── pod-lifecycle-watch.png
        └── nginx-curl-success.png
```

## Prerequisites

- Windows with Ubuntu/WSL or a Linux machine
- Docker Engine or Docker Desktop configured for the environment
- Minikube
- `kubectl`
- Internet access for initial image downloads

Check the tools:

```bash
kubectl version --client
minikube version
docker --version
docker ps
```

Start the cluster:

```bash
minikube start --driver=docker
minikube status
kubectl get nodes
```

The node should eventually show `Ready`.

> If Minikube warns that the requested memory allocation leaves little room for system overhead, check available WSL memory before changing resource settings. Do not blindly allocate more memory than the machine can provide.

## Exercise 1: Run a standalone Nginx Pod

The initial practice used a standalone Pod. Apply the included manifest:

```bash
kubectl apply -f manifests/standalone-pod.yaml
kubectl get pods -o wide
kubectl describe pod nginx
kubectl exec nginx -- nginx -t
```

The Nginx configuration check should report that the syntax is OK and the test is successful.

### Access Nginx through port forwarding

Run this command in Terminal 1 and leave it running:

```bash
kubectl port-forward pod/nginx 8080:80
```

In Terminal 2:

```bash
curl http://localhost:8080
```

Expected result: HTML containing `Welcome to nginx!`. You can also open `http://localhost:8080` in your browser while port forwarding is active.

Stop port forwarding with `Ctrl+C` when finished. Port forwarding is temporary and must remain active during the test.

### Pod labels and Services

A previous attempt to expose the original Pod failed because it had no labels. In that exercise, a label was added:

```bash
kubectl label pod nginx app=nginx
kubectl get pod nginx --show-labels
```

Then a Service can be created:

```bash
kubectl expose pod nginx --type=NodePort --port=80 --target-port=80
kubectl get services
kubectl get endpoints nginx
minikube service nginx --url
```

If a Service already exists, Kubernetes may report that it already exists. If the Service has no endpoints, check that its selector matches the Pod labels. `10.96.0.1` is the default Kubernetes API Service IP, not the Nginx application address.

## Exercise 2: Deploy Nginx with three replicas

Apply the Deployment and Service manifests:

```bash
kubectl apply -f manifests/deployment.yaml
kubectl apply -f manifests/service.yaml
```

Check the resources:

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
kubectl get services
```

The Deployment requests three replicas. Wait until the three Pods are ready:

```bash
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -w
```

Press `Ctrl+C` to stop watching. This stops only the watch command; it does not stop the Deployment.

## Exercise 3: Observe self-healing

Find a Pod managed by the Deployment:

```bash
kubectl get pods -l app=nginx
```

In Terminal 1, start watching the Pods:

```bash
kubectl get pods -l app=nginx -w
```

In Terminal 2, delete **one current Pod name** from the previous command:

```bash
kubectl delete pod <current-pod-name>
```

Replace `<current-pod-name>` with an actual name. Watch Kubernetes terminate that Pod and create a replacement. The exact intermediate states and timing vary; a fast replacement may make some states difficult to see.

The Deployment controller maintains the desired replica count. Deleting a Pod does not delete its parent Deployment.

Useful additional commands:

```bash
kubectl get deployment nginx-deployment -w
kubectl describe deployment nginx-deployment
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get replicasets
```

## Troubleshooting notes from this lab

### `kubectl` command not found

Downloading a binary with `curl` does not install it automatically. Make it executable and place it in a directory on `PATH`, for example:

```bash
chmod +x kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### `kubectl version` reports a connection error

Use `kubectl version --client` to check the client independently. Commands that contact the cluster require a running cluster and a working kubeconfig/context.

### Pod IP curl fails from the WSL host

Pod IPs are internal cluster addresses and may not be reachable directly from the host. Use:

```bash
kubectl port-forward pod/nginx 8080:80
curl http://localhost:8080
```

### `kubectl expose` says the Pod has no labels

Inspect and add a label before exposing the Pod:

```bash
kubectl get pod nginx --show-labels
kubectl label pod nginx app=nginx
kubectl expose pod nginx --type=NodePort --port=80 --target-port=80
```

### `minikube service nginx --url` says Service not found

Create the Service successfully first, then check it:

```bash
kubectl get services
kubectl get endpoints nginx
```

### `localhost:8080` fails inside `minikube ssh`

`localhost` means the machine or network namespace where the command is executed. If port forwarding is running in the WSL host terminal, test `localhost:8080` from another WSL host terminal—not from inside `minikube ssh`.

### A deleted Pod returns as a new Pod

This is expected when the Pod belongs to a Deployment. Kubernetes creates a replacement to maintain the desired replica count.

## Cleanup

Delete the resources created by this lab:

```bash
kubectl delete -f manifests/service.yaml --ignore-not-found
kubectl delete -f manifests/deployment.yaml --ignore-not-found
kubectl delete -f manifests/standalone-pod.yaml --ignore-not-found
```

If you created the standalone Service separately during practice, remove it if it still exists:

```bash
kubectl delete service nginx --ignore-not-found
```

To stop the local Minikube cluster:

```bash
minikube stop
```

To remove the Minikube cluster entirely (destructive; only do this when you no longer need it):

```bash
minikube delete
```

## Evidence screenshots

Screenshots from the practice session are stored in [`docs/screenshots/`](docs/screenshots/). They show the Nginx HTTP response and the Deployment-managed Pod lifecycle.

## What I learned

- The difference between a standalone Pod and a Deployment-managed Pod.
- How labels and Service selectors connect Kubernetes resources.
- Why a running Pod does not automatically mean it is reachable from the host.
- How port forwarding helps test an application locally.
- How Deployments maintain the requested number of replicas by replacing deleted Pods.
- How to use Kubernetes status, events, and configuration checks to troubleshoot issues.

## Possible next improvements

These are future enhancements, not part of the completed lab:

- Add resource requests and limits.
- Add readiness and liveness probes.
- Use a ConfigMap to customize the Nginx page.
- Add a small CI workflow to validate YAML manifests.
- Deploy a sample application image and document a rolling update and rollback.
