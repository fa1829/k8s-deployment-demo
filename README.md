# Kubernetes Deployment Demo

Deploying and autoscaling a containerized Flask API (`devops-demo-api`) on Kubernetes, with an explanation of *why* each piece exists — written for someone who has never touched Kubernetes before.

## Why this project exists

My earlier projects covered CI/CD pipelines, Docker, Terraform/AWS, and Prometheus/Grafana monitoring. The one DevOps skill pillar I hadn't demonstrated hands-on was **container orchestration**. Docker alone runs a single container on a single machine — it doesn't restart it if it crashes, doesn't scale it under load, and doesn't manage multiple machines. Kubernetes is the layer that solves those problems, and it's effectively assumed knowledge for any SecDevOps/Cloud role today (including the Ericsson posting this project was built for).

## What is Kubernetes, in plain terms

Kubernetes (K8s) is a system that runs your containers for you and keeps them running the way you told it to, automatically, without you babysitting them.

You describe the *desired state* in YAML files ("I want 2 copies of this container always running, exposed on this port"), and Kubernetes continuously works to make reality match that description. If a container crashes, Kubernetes restarts it. If a server it's running on dies, Kubernetes reschedules the container elsewhere. If traffic spikes, Kubernetes can spin up more copies automatically.

This is the same model Anthropic, Google, Netflix, and most large-scale production systems use under the hood — `minikube` (used here) is just a single-node version of the same thing, running on one laptop for learning/demo purposes instead of across a real data center.

## What's actually deployed here

| Kubernetes Object | What it is | What it does in this project |
|---|---|---|
| **Deployment** | A blueprint that says "keep N copies of this container running" | Keeps 2 replicas of the Flask API container alive at all times |
| **Pod** | The smallest deployable unit — one running instance of your container | Each replica = 1 pod, each pod runs the Flask app |
| **Service** | A stable network address that load-balances across pods | Lets you reach the API at one address even though pods can be replaced/rescheduled with new internal IPs |
| **HorizontalPodAutoscaler (HPA)** | A controller that watches resource usage and changes replica count | Automatically scales pods 2→5 if CPU usage gets high, and back down when it drops |

## What is HPA, specifically — and why it matters

**The problem it solves:** without it, you manually decide how many copies of your app to run. Too few and the app slows down or crashes under load; too many and you're wasting money on idle compute capacity.

**How it works here:**
```yaml
minReplicas: 2
maxReplicas: 5
metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 60
```
Kubernetes checks the average CPU usage across all running pods. If it climbs above 60%, it adds pods (up to 5) to spread the load. If usage drops, it scales back down (never below 2, so there's always redundancy).

**Why this matters for a business, not just technically:**
- **Small organization / startup**: you don't have a dedicated on-call engineer watching dashboards at 2am to scale up before a traffic spike crashes the site. HPA does that automatically, which is exactly the kind of "do more with a small team" leverage a startup needs.
- **Large organization**: at scale, manually scaling thousands of services is impossible. HPA (and Kubernetes generally) is how companies like Netflix or your bank's mobile app backend handle Black Friday-style traffic spikes without over-provisioning hardware year-round "just in case."
- **DevSecOps angle specifically**: stability and availability are security properties too — a service that falls over under load is an availability failure (the "A" in the CIA security triad). Autoscaling is a concrete mitigation against both legitimate traffic spikes *and* certain denial-of-service conditions, by absorbing load instead of falling over immediately.

## Architecture
                     ┌─────────────────────────────┐
Client request  ───▶  │   Service (NodePort)        │

│   stable address, port 80   │

└──────────────┬───────────────┘

│ load-balances across

┌─────────────┼─────────────┐

▼             ▼             ▼

┌──────────┐ ┌──────────┐  (up to 3 more

│  Pod 1   │ │  Pod 2   │   pods if HPA

│ Flask API│ │ Flask API│   scales out)

└──────────┘ └──────────┘

▲             ▲

└──────┬──────┘

│ watched by

┌─────────────────┐

│       HPA        │

│ scales 2→5 pods  │

│ based on CPU %   │

└─────────────────┘
## Tech stack

- **Minikube** (Docker driver) — runs a single-node Kubernetes cluster locally, inside Docker, no separate VM needed
- **kubectl** — the command-line tool used to talk to the Kubernetes cluster (create resources, check status, view logs)
- **Docker image** reused from the `devops-demo-api` project — no new application code, only the orchestration layer is new here

## How to reproduce this yourself

```bash
# 1. Start a local Kubernetes cluster
minikube start --driver=docker

# 2. Build the image *inside* minikube's Docker context
#    (so Kubernetes can find it locally instead of pulling from the internet)
eval $(minikube docker-env)
docker build -t devops-demo-api:1.0 ../devops-demo-api

# 3. Deploy everything described in the manifests folder
kubectl apply -f manifests/

# 4. Check pods came up healthy
kubectl get pods

# 5. Expose the service through a local tunnel (Docker driver requirement on Linux/WSL2)
minikube service devops-demo-api-service --url

# 6. In a separate terminal, hit the API
curl <url-printed-by-step-5>
```

## Verified output
$ kubectl get pods

NAME                              READY   STATUS    RESTARTS   AGE

devops-demo-api-99b848d67-fsprw   1/1     Running   0          8h

devops-demo-api-99b848d67-zrf6f   1/1     Running   0          8h
$ curl http://127.0.0.1:37403

{

"service": "devops-demo-api",

"version": "1.0.0",

"environment": "production",

"message": "CI/CD pipeline deployed this automatically ✓",

"endpoints": {

"health": "/health",

"info": "/info",

"readiness": "/ready",

"tasks": "/api/tasks"

}

}

## Troubleshooting — issues I actually hit

| Symptom | Cause | Fix |
|---|---|---|
| `kubectl apply -f manifests/` → `error: the path "manifests/" does not exist` | Ran the command from *inside* the `manifests/` folder, so the relative path didn't resolve | `cd` back to the project root first, where `manifests/` is a subfolder |
| Pods stuck in `ImagePullBackOff` | Kubernetes tried to pull the image from Docker Hub instead of using the local one | Set `imagePullPolicy: Never` in the deployment, and make sure the image was built *after* running `eval $(minikube docker-env)` |
| `minikube service --url` prints a URL but the terminal doesn't return to a prompt | With the Docker driver on Linux/WSL2, this command opens a tunnel that must stay alive in its own process | Run it with `&` to background it, or open a second terminal tab for the `curl` test, and don't `Ctrl+C` it until you're done testing |
| `services "devops-demo-api-service" not found` | Tried to check the service before `kubectl apply` actually succeeded | Re-run `kubectl get pods` / `kubectl get svc` only after confirming `apply` output shows `created` for all three resources |
| `git push` → `Invalid username or token. Password authentication is not supported` | GitHub no longer accepts account passwords for Git operations over HTTPS | Generate a Personal Access Token (Settings → Developer settings → Tokens) and use that in place of the password, or authenticate via `gh auth login` |

## What I'd add next

- Wire pod-level metrics into the existing Prometheus/Grafana stack from `security-monitoring-dashboard` via `kube-state-metrics`, so HPA scaling events are visible on a dashboard instead of only in `kubectl` output
- Push the image to a real container registry (Docker Hub/ECR) instead of relying on `imagePullPolicy: Never`, which only works for local clusters
- Move from Minikube to a managed cluster (EKS/AKS/GKE) to demonstrate the same manifests working against production-grade infrastructure
- Add a NetworkPolicy to restrict which pods can talk to this service, as a basic security control