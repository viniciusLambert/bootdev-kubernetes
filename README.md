# Kubernetes: SynergyChat

Kubernetes manifests written while working through the [Boot.dev "Learn Kubernetes"](https://www.boot.dev/courses/learn-kubernetes) course. They deploy **SynergyChat**, a small chat application made of several microservices, onto a local cluster (minikube) and exercise the core Kubernetes building blocks along the way.

## Architecture

```
                         ┌──────────────────────────┐
  synchat.internal  ───▶ │                          │ ──▶ web-service ──▶ synergychat-web (HPA 1-4)
                         │  app-gateway (Envoy)     │
  synchatapi.internal ─▶ │  Gateway API             │ ──▶ api-service ──▶ synergychat-api ──▶ PVC (1Gi)
                         └──────────────────────────┘                          │
                                                                               ▼
                                                     crawler-service ──▶ synergychat-crawler
                                                                         (3 containers, shared emptyDir)
```

| Service | Image | Purpose |
|---------|-------|---------|
| `synergychat-web` | `bootdotdev/synergychat-web` | Frontend UI, autoscaled by CPU |
| `synergychat-api` | `bootdotdev/synergychat-api` | Backend API, persists data to a PVC |
| `synergychat-crawler` | `bootdotdev/synergychat-crawler` | Crawls keywords; three containers in one pod sharing a cache volume |
| `synergychat-testcpu` | `bootdotdev/synergychat-testcpu` | CPU stress test for resource limits and HPA |
| `synergychat-testram` | `bootdotdev/synergychat-testram` | Memory stress test for requests and limits |

## Concepts covered

- **Deployments** and ReplicaSets, rolling update strategy
- **ConfigMaps** injected via `envFrom` and per-key `configMapKeyRef`
- **Services** (`ClusterIP`) and in-cluster DNS (`<service>.<namespace>.svc.cluster.local`)
- **Gateway API**: `GatewayClass`, `Gateway` and host-based `HTTPRoute`s (Envoy Gateway controller)
- **Storage**: `emptyDir` shared between containers and a `PersistentVolumeClaim`
- **Multi-container pods**
- **Resources**: CPU and memory requests/limits
- **Autoscaling**: `HorizontalPodAutoscaler` driven by CPU utilization

## Files

| Component | Manifests |
|-----------|-----------|
| Gateway | `app-gatewayclass.yaml`, `app-gateway.yaml` |
| Web | `web-configmap.yaml`, `web-deployment.yaml`, `web-service.yaml`, `web-httproute.yaml`, `web-hpa.yaml` |
| API | `api-configmap.yaml`, `api-deployment.yaml`, `api-service.yaml`, `api-httprout.yaml`, `api-pvc.yaml` |
| Crawler | `crawler-configmap.yaml`, `crawler-deployment.yaml`, `crawler-service.yaml` |
| Test CPU | `testcpu-deployment.yaml`, `testcpu-hpa.yaml` |
| Test RAM | `testram-configmap.yaml`, `testram-deployment.yaml` |

## Running locally

Prerequisites: [minikube](https://minikube.sigs.k8s.io/), [kubectl](https://kubernetes.io/docs/tasks/tools/) and [Envoy Gateway](https://gateway.envoyproxy.io/).

```bash
minikube start
minikube addons enable metrics-server

helm install eg oci://docker.io/envoyproxy/gateway-helm -n envoy-gateway-system --create-namespace

kubectl apply -f app-gatewayclass.yaml -f app-gateway.yaml

kubectl apply -f api-configmap.yaml -f api-pvc.yaml -f api-deployment.yaml -f api-service.yaml -f api-httprout.yaml
kubectl apply -f web-configmap.yaml -f web-deployment.yaml -f web-service.yaml -f web-httproute.yaml -f web-hpa.yaml

kubectl create namespace crawler
kubectl apply -n crawler -f crawler-configmap.yaml -f crawler-deployment.yaml -f crawler-service.yaml
```

The API reaches the crawler at `crawler-service.crawler.svc.cluster.local`, so the crawler resources live in the `crawler` namespace.

Expose the gateway and map the hostnames:

```bash
minikube tunnel
```

```
# /etc/hosts
127.0.0.1 synchat.internal
127.0.0.1 synchatapi.internal
```

Then open http://synchat.internal.

## Useful commands

```bash
kubectl get pods -A
kubectl get hpa
kubectl top pods
kubectl describe pod <name>
kubectl logs <pod> -c <container>
kubectl port-forward svc/web-service 8080:80
```

## Notes

Some manifests are intentionally left in the state required by a specific lesson:

- `testram-deployment.yaml` requests `40000Mi` of memory so the pod stays `Pending` and demonstrates unschedulable workloads.
- `testcpu-deployment.yaml` limits CPU to `10m` to show throttling.
