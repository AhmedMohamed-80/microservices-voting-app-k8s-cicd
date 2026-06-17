# Voting App on Kubernetes

A microservices voting application deployed on Kubernetes, with a GitHub Actions pipeline that builds, tests, and publishes versioned images to Docker Hub. Validated end-to-end on a local **kind** cluster.

`Kubernetes` · `Docker` · `GitHub Actions` · `PostgreSQL` · `Redis` · `kind`

---

## Table of contents

- [Architecture](#architecture)
- [Components](#components)
- [CI/CD pipeline](#cicd-pipeline)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Accessing the app](#accessing-the-app)
- [Testing the data path](#testing-the-data-path)
- [Updating to a new image version](#updating-to-a-new-image-version)
- [Troubleshooting](#troubleshooting)
- [Roadmap](#roadmap)

---

## Architecture

A user casts a vote through the **vote** web UI. The vote is pushed onto a list in **Redis**. The **worker** continuously polls Redis, pulls new votes off the list, and writes them into **PostgreSQL**. The **result** web UI reads from PostgreSQL and pushes live results back to the browser.

![Voting app architecture and data flow](images/architecture.svg)

All resources are deployed into the `voting-app` namespace.

## Components

| Component | Kind | Replicas | Purpose |
|---|---|---|---|
| `vote` | Deployment + Service (LoadBalancer) | 2 | Voting web UI |
| `result` | Deployment + Service (LoadBalancer) | 2 | Live results web UI |
| `worker` | Deployment | 1 | Moves votes from Redis to Postgres |
| `redis` | StatefulSet + Service (headless) | 1 | Vote queue |
| `db` | StatefulSet + Service (headless) | 1 | PostgreSQL — persistent vote storage |

**Supporting resources**

| File | Resource(s) |
|---|---|
| `namespace.yaml` | `voting-app` Namespace |
| `rbac.yaml` | ServiceAccount, Role, RoleBinding (`voting-app-sa`) |
| `secret.yaml` | PostgreSQL credentials |
| `configmap.yaml` | Shared non-secret env vars (`POSTGRES_DB`, `POSTGRES_HOST`, `REDIS_HOST`) |

All workload pods run under the `voting-app-sa` ServiceAccount, scoped by a least-privilege Role granting `get/list/watch` on pods, services, and configmaps within the namespace.

## CI/CD pipeline

Each custom service — `vote`, `result`, `worker` — has its own GitHub Actions workflow that builds, tests, and publishes a semantic-versioned image to Docker Hub.

![CI/CD pipeline from GitHub to Kubernetes](images/cicd-pipeline.svg)

**Flow**

1. **Commit & push** — a developer pushes code to the GitHub repository.
2. **Webhook trigger** — GitHub notifies Actions, which starts the workflow.
3. **Build image** — `docker build` produces the image from the service's Dockerfile.
4. **Run container** — the freshly built image runs as a container inside CI.
5. **Test** — the running container is exercised against the test suite.
6. **Push to Docker Hub** — on a passing test run, the image is tagged with a semantic version (for example `v1.2.0`) and pushed.
7. **Deploy** — the cluster pulls the new tagged image and rolls it out to the relevant Deployment's pods.

> **Status:** step 7 is currently a manual step. After a new version lands on Docker Hub, the corresponding Deployment's image tag is updated and reapplied (see [Updating to a new image version](#updating-to-a-new-image-version)). Automating this — via a deploy workflow or a GitOps tool such as Argo CD or Flux — is tracked in [Roadmap](#roadmap).

**Image tagging convention**

Images are published using semantic versioning, not `:latest`:

```
docker.io/<dockerhub-user>/vote:v1.0.0
docker.io/<dockerhub-user>/result:v1.0.0
docker.io/<dockerhub-user>/worker:v1.0.0
```

> The manifests in this repo currently reference `:latest`. Once the deploy step above is automated, update `vote-deployment.yaml`, `result-deployment.yaml`, and `worker-deployment.yaml` to pin explicit version tags. Pinning avoids the two most common headaches with `:latest`: unpredictable rollouts when the tag silently moves, and unnecessary re-pulls on every pod restart under `imagePullPolicy: Always`.

## Prerequisites

- `kubectl` configured against your target cluster
- `kind`, for local testing, or any conformant Kubernetes cluster (EKS, GKE, AKS, etc.)
- A StorageClass available in-cluster for the two StatefulSets (`db`, `redis`) — confirm with `kubectl get storageclass`
- A Docker Hub account with the three images already pushed, or this repo's CI configured to push them

## Deployment

Apply the namespace first, then the rest, so namespaced resources have somewhere to land:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f rbac.yaml
kubectl apply -f secret.yaml
kubectl apply -f configmap.yaml
kubectl apply -f db-statefulset.yaml
kubectl apply -f redis-statefulset.yaml
kubectl apply -f vote-deployment.yaml
kubectl apply -f result-deployment.yaml
kubectl apply -f worker-deployment.yaml
```

Or, once the namespace exists, apply everything in a single pass — Kubernetes resolves Secret and ConfigMap references at pod-start time rather than at object-creation time, so strict ordering of the remaining files isn't required:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f .
```

### Local testing with kind

If images were built locally rather than pulled from Docker Hub, load them into the kind cluster's node first — kind nodes can't see the host Docker daemon's image cache:

```bash
kind load docker-image <dockerhub-user>/vote:v1.0.0 <dockerhub-user>/result:v1.0.0 <dockerhub-user>/worker:v1.0.0 --name <your-cluster-name>
```

### Verify rollout

```bash
kubectl get pods -n voting-app -o wide
kubectl get svc -n voting-app
```

Every pod should reach `Running` with `READY 1/1`. The `vote` and `result` Services are type `LoadBalancer`; in kind these show `EXTERNAL-IP: <pending>` indefinitely, since kind has no cloud controller manager. That's expected, not a fault.

## Accessing the app

### In kind (no Ingress yet)

`LoadBalancer` doesn't resolve to a real external IP in kind, so use port-forwarding:

```bash
kubectl port-forward -n voting-app svc/vote 8080:80
```

Open `http://localhost:8080` for the voting UI. In a second terminal:

```bash
kubectl port-forward -n voting-app svc/result 8081:80
```

Open `http://localhost:8081` for the live results UI.

### In a managed cluster

On EKS, GKE, or AKS, `EXTERNAL-IP` populates automatically once the cloud provider provisions the load balancer:

```bash
kubectl get svc -n voting-app -w
```

### Planned: Ingress

Not yet implemented. Once added, `vote` and `result` will likely switch from `type: LoadBalancer` to `type: ClusterIP`, fronted by a single Ingress resource (for example `ingress-nginx`) routing by host or path — removing the need for two separate load balancers and for local port-forwarding during testing.

## Testing the data path

Pods reaching `Running` doesn't confirm the vote → Redis → worker → Postgres → result pipeline is wired correctly. To verify end-to-end:

1. **Cast a vote** through the `vote` UI.
2. **Confirm Redis received it:**

   ```bash
   kubectl exec -n voting-app -it redis-0 -- redis-cli LRANGE votes 0 -1
   ```

3. **Confirm the worker wrote it to Postgres:**

   ```bash
   kubectl exec -n voting-app -it db-0 -- psql -U postgres -d postgres -c "SELECT * FROM votes;"
   ```

4. **Confirm the result UI reflects it** by opening the results page and checking that the count updates.

If step 2 returns nothing, check DNS resolution from inside the `vote` pod:

```bash
kubectl exec -n voting-app -it deploy/vote -- nslookup redis
```

If step 3 returns nothing despite step 2 succeeding, check the worker's logs:

```bash
kubectl logs -n voting-app deploy/worker
```

### Resilience check

Confirm the app survives a pod restart without dropping traffic — this validates readiness probes and Service endpoint management:

```bash
kubectl delete pod -n voting-app -l app=vote --field-selector status.phase=Running
kubectl get pods -n voting-app -l app=vote -w
```

Voting should continue uninterrupted while the deleted pod is rescheduled and becomes ready again.

## Updating to a new image version

Once CI pushes a new semantic-versioned image, roll it out with `kubectl set image`:

```bash
kubectl set image deployment/vote vote=<dockerhub-user>/vote:v1.1.0 -n voting-app
kubectl rollout status deployment/vote -n voting-app
```

`vote` and `result` use `RollingUpdate` with `maxSurge: 1` and `maxUnavailable: 1`, plus a `progressDeadlineSeconds` of 600, so a bad rollout is flagged as failed rather than left hanging. `worker` uses `maxUnavailable: 0`, since its single replica shouldn't go fully unavailable mid-update.

To roll back:

```bash
kubectl rollout undo deployment/vote -n voting-app
```

## Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Pod stuck `Pending` | No matching StorageClass for the PVC | `kubectl get storageclass`, `kubectl describe pvc -n voting-app` |
| Pod `CreateContainerConfigError` | Missing or misnamed Secret/ConfigMap key | `kubectl describe pod <name> -n voting-app` |
| Pod `ImagePullBackOff` | Image not pushed yet, wrong tag, or not loaded into kind | `kubectl describe pod <name> -n voting-app`; for kind, confirm `kind load docker-image` was run |
| `vote` / `result` unreachable | `LoadBalancer` has no `EXTERNAL-IP` in kind | Use `kubectl port-forward`, or install MetalLB for real LB behavior |
| Vote not appearing in results | Break somewhere in the vote → Redis → worker → Postgres chain | Walk through [Testing the data path](#testing-the-data-path) to isolate which hop is failing |
| Readiness/liveness probe failing | Container not actually ready, or probe command unavailable in the image | `kubectl logs <pod> -n voting-app`, `kubectl exec -it <pod> -n voting-app -- sh` |

## Roadmap

- **Ingress** — replace `LoadBalancer` Services with a single Ingress resource.
- **Automated deploy step** — wire the CI pipeline directly into the cluster rollout via a deploy workflow or a GitOps tool (Argo CD, Flux).
- **Pinned image tags** — move manifests from `:latest` to the semantic version tags the pipeline already produces.
- **Secrets management** — replace plaintext `stringData` with Sealed Secrets, External Secrets Operator, or a cloud KMS-backed store for anything beyond local development.
- **Autoscaling** — add HPA for `vote` and `result`, currently fixed at static replica counts.
