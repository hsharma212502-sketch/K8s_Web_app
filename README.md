# K8s Web App

A multi-tier web application deployed on Kubernetes — a Node.js webapp connected to a MongoDB backend, wired together using Kubernetes-native primitives: **Deployments**, **Services**, **ConfigMaps**, and **Secrets**. Built as a hands-on project to understand how Kubernetes manages configuration, service discovery, and inter-pod communication inside a cluster.

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-6.0-47A248?style=flat&logo=mongodb&logoColor=white)
![Minikube](https://img.shields.io/badge/Tested%20on-Minikube-blue)

---

## Architecture

```
                       ┌────────────────────────────────┐
   Browser  ─────────► │  webapp-service (NodePort)     │
   (localhost:30300)   │  external entry point          │
                       └──────────────┬─────────────────┘
                                      │
                                      ▼
                       ┌────────────────────────────────┐
                       │  webapp Pod                    │
                       │  (nanajanashia/k8s-demo-app)   │
                       │                                │
                       │  reads at startup:             │
                       │   • USER_NAME ← Secret         │
                       │   • USER_PWD  ← Secret         │
                       │   • DB_URL    ← ConfigMap      │
                       └──────────────┬─────────────────┘
                                      │
                                      ▼
                       ┌────────────────────────────────┐
                       │  mongo-service (ClusterIP)     │
                       │  internal only                 │
                       └──────────────┬─────────────────┘
                                      │
                                      ▼
                       ┌────────────────────────────────┐
                       │  mongo Pod (mongo:6.0)         │
                       │  root user initialized from    │
                       │  the same Secret               │
                       └────────────────────────────────┘
```

## Stack

- **Kubernetes** (developed and tested on minikube)
- **MongoDB 6.0**
- **Node.js demo app** — `nanajanashia/k8s-demo-app:v1.0`

## Repository layout

| File | Resources | Purpose |
| --- | --- | --- |
| `Mongo-Secret.yaml` | Secret | MongoDB root username + password (base64-encoded) |
| `Mongo-config.yaml` | ConfigMap | The in-cluster Mongo URL the webapp uses to find the database |
| `Mongo.yaml` | Deployment + Service | Runs MongoDB and exposes it internally at `mongo-service:27017` |
| `webapp.yaml` | Deployment + Service | Runs the webapp and exposes it externally on NodePort `30300` |

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) or another container runtime
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/)

## Quick start

```bash
# 1. Start the cluster
minikube start

# 2. Apply configuration first (workloads depend on these)
kubectl apply -f Mongo-Secret.yaml
kubectl apply -f Mongo-config.yaml

# 3. Apply workloads
kubectl apply -f Mongo.yaml
kubectl apply -f webapp.yaml

# 4. Verify everything is running
kubectl get all

# 5. Open the app in your browser
minikube service webapp-service
```

The apply order matters: referenced Secrets and ConfigMaps must exist before the pods that consume them, otherwise the pods will fail to start with a `CreateContainerConfigError`.

## How the pieces talk to each other

The webapp pod reads three environment variables at startup, each sourced from a Kubernetes object rather than baked into the image:

| Variable | Source | Key |
| --- | --- | --- |
| `USER_NAME` | Secret `mongo-secret` | `mongo-user` |
| `USER_PWD` | Secret `mongo-secret` | `mongo-password` |
| `DB_URL` | ConfigMap `mongo-config` | `mongo-url` *(value: `mongo-service`)* |

The webapp uses those values to connect to MongoDB. Inside the cluster, Kubernetes' internal DNS resolves the Service name `mongo-service` to the MongoDB pod's cluster IP automatically — no hardcoded IPs, no manual wiring.

MongoDB itself is bootstrapped with `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` (pulled from the same Secret), so when the webapp sends those credentials, Mongo accepts them.

## Configuration

**To change the MongoDB credentials**, edit `Mongo-Secret.yaml`. Values must be base64-encoded:

```bash
echo -n 'your-username' | base64
echo -n 'your-password' | base64
```

**To point the webapp at a different database service**, edit `mongo-url` in `Mongo-config.yaml`.

## Useful debugging commands

```bash
kubectl get pods                              # which pods exist, are they Running?
kubectl describe pod <pod-name>               # why is this pod failing?
kubectl logs <pod-name>                       # what is the container saying?
kubectl exec -it <pod-name> -- sh             # shell into a running pod
kubectl get svc                               # what services exist, on what ports?
minikube service list                         # NodePort URLs for everything
```

## Teardown

```bash
kubectl delete -f webapp.yaml
kubectl delete -f Mongo.yaml
kubectl delete -f Mongo-config.yaml
kubectl delete -f Mongo-Secret.yaml
minikube stop
```

## What I learned building this

- How **ConfigMaps** and **Secrets** decouple configuration from container images, so the same image can run in dev, staging, and prod without rebuilding.
- How **Service DNS** lets pods discover each other by name (`mongo-service`) without hardcoded IPs.
- Why **apply order** matters — referenced resources must exist before the workloads that consume them.
- The strictness of **RFC 1123 naming** in Kubernetes (lowercase only, no underscores, no capitals) and how YAML's list-vs-object distinction can silently break manifests.
- How to read Kubernetes error messages — most "weird" failures are actually `kubectl describe` away from being obvious.

## Roadmap

Planned improvements to turn this from a learning project into a production-shaped one:

- [ ] Replace NodePort with an **Ingress** for cleaner external routing
- [ ] Add **resource requests and limits** + **readiness/liveness probes**
- [ ] Add a **HorizontalPodAutoscaler** for the webapp
- [ ] Set up **GitHub Actions** to lint manifests with `kubeconform`
- [ ] Replace the demo image with a **self-built application** and a Dockerfile
- [ ] Add **persistent storage** (PVC) for MongoDB so data survives pod restarts

## License

MIT
