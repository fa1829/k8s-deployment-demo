# Kubernetes Deployment Demo

Deploying and autoscaling a containerized Flask API (`devops-demo-api`) on Kubernetes using Minikube, building on the CI/CD and Docker foundations from earlier projects.

## Why this project

My previous projects covered CI/CD pipelines, Docker, Terraform/AWS, and Prometheus/Grafana monitoring. The missing piece in a complete DevOps skill set was container orchestration — this project closes that gap by taking an existing Dockerized service and deploying it on Kubernetes with horizontal autoscaling.

## Architecture
Client → Service (NodePort) → Deployment (2+ replica Pods) → Flask API container

↑

HorizontalPodAutoscaler (CPU-based, 2-5 replicas)

## What's deployed

- **Deployment**: 2 replicas of `devops-demo-api`, with CPU/memory requests and limits set
- **Service**: NodePort exposing port 80 → container port 5000
- **HorizontalPodAutoscaler**: scales 2→5 replicas based on 60% CPU utilization target

## Tech stack

- Minikube (Docker driver) for local Kubernetes cluster
- kubectl for cluster management
- Reused Docker image from `devops-demo-api` project

## How to reproduce

```bash
minikube start --driver=docker
eval $(minikube docker-env)
docker build -t devops-demo-api:1.0 ../devops-demo-api
kubectl apply -f manifests/
minikube service devops-demo-api-service --url
curl <returned-url>
```

## Verified output
$ kubectl get pods

NAME                              READY   STATUS    RESTARTS   AGE

devops-demo-api-99b848d67-fsprw   1/1     Running   0          8h

devops-demo-api-99b848d67-zrf6f   1/1     Running   0          8h
$ curl http://127.0.0.1:37403

{"service":"devops-demo-api","version":"1.0.0","environment":"production", ...}

## What I'd add next

- Wire pod metrics into the existing Prometheus/Grafana stack from `security-monitoring-dashboard` via `kube-state-metrics`
- Push image to a real registry (Docker Hub/ECR) instead of using `imagePullPolicy: Never`
- Move from Minikube to a managed cluster (EKS/AKS) for a production-equivalent setup

## Notes / lessons learned

- `imagePullPolicy: Never` is required when using locally-built images with Minikube's Docker driver — otherwise Kubernetes tries (and fails) to pull from Docker Hub.
- On the Docker driver (Linux/WSL2), `minikube service --url` opens a tunnel that must stay running in its own terminal session for the URL to remain reachable.
