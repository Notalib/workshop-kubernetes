# Kubernetes CLI Demo

A live-demo walkthrough showing the power of Kubernetes using `kubectl`. Run during
the workshop. It mirrors Workshop #1's Docker CLI demo — but where Docker ran one
container on one machine, Kubernetes keeps your workloads **running, scaled, and
reachable** without you babysitting them.

Run everything in your `workshop` namespace (see [../setup](../setup/README.md)).

---

# 1. Run something, the imperative way

```bash
kubectl create deployment web --image=nginx
kubectl get all
```

`get all` shows what one command created: a **Deployment**, a **ReplicaSet**, and a
**Pod**. You asked for "an nginx should exist" — Kubernetes built the scaffolding to
make that true and keep it true.

## Observations

- One command, a whole managed workload.
- You declared *what* you want; you didn't script *how* to start it.

---

# 2. Self-healing — kill it and watch it come back

```bash
kubectl get pods --watch          # leave this running in one terminal
```

In another terminal, delete the Pod:

```bash
kubectl delete pod -l app=web
```

Watch the first terminal: the Pod terminates and a **new one is created within
seconds**. The Deployment's desired state is "1 replica", so the controller
reconciles reality back to that.

## Observations

- You cannot "turn it off" by deleting the Pod — the control loop recreates it.
- This is the core idea: **desired state, continuously reconciled.** (Try the same
  thing with `docker rm` — the container just stays gone.)

---

# 3. Scale to many

```bash
kubectl scale deployment/web --replicas=5
kubectl get pods -o wide
```

Five Pods, possibly spread across nodes (on a multi-node cluster). Scale back down:

```bash
kubectl scale deployment/web --replicas=2
```

## Observations

- Scaling is one number in the desired state.
- Same image, many identical instances — the "more Lego bricks on the board" idea.

---

# 4. Roll out a change with zero downtime

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout history deployment/web
```

Kubernetes brings up new Pods before tearing down old ones. Roll back if it went bad:

```bash
kubectl rollout undo deployment/web
```

### Now break it on purpose — and watch nothing break

Push an image that doesn't exist and try to roll it out:

```bash
kubectl set image deployment/web nginx=nginx:this-tag-does-not-exist
kubectl rollout status deployment/web    # hangs — the rollout is stuck
```

In another terminal, look at what's happening:

```bash
kubectl get deployment/web   # AVAILABLE stays at the old count — still serving!
kubectl get pods -l app=web  # the NEW Pod is stuck ImagePullBackOff
```

The bad version **never receives traffic**, because Kubernetes won't tear down the old
Pods until the new ones are healthy. Zero downtime from a broken deploy. Roll back:

```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web    # healthy again
```

## Observations

- Rolling updates and rollbacks are built in, not something you script.
- A broken rollout is *safe*: the old version keeps serving until the new one is healthy.
- **Health probes** make this protection cover apps that *start but aren't ready yet*
  (e.g. a server still warming up): a **readiness** probe holds traffic until the Pod
  reports ready, and a **liveness** probe restarts a wedged container. You'll wire real
  probes into the [module 5 capstone](../5-composite-system/README.md).

---

# 5. Reach the app

Quick-and-dirty (just your machine, for debugging):

```bash
kubectl port-forward deployment/web 8080:80
# open http://localhost:8080
```

The real way — give the Pods a stable identity with a **Service**:

```bash
kubectl expose deployment/web --port=80
kubectl get service web
```

A Service is one stable DNS name + IP in front of a moving set of Pods. From inside
the cluster, `web` always resolves to whichever Pods are currently healthy.

## Observations

- Pods are short-lived and their IPs change; the Service name does not.
- This is the cluster's internal DNS — the foundation of service discovery.

---

# 6. Peek inside

```bash
kubectl describe deployment/web        # events, replicas, strategy
kubectl logs -l app=web                # aggregated stdout/stderr
kubectl exec -it deployment/web -- /bin/sh   # shell inside a running Pod
```

Prefer a UI? Try **k9s** (terminal dashboard) or Rancher's web UI to navigate the
same objects visually.

---

# 7. Under the hood: kubectl is just an API client

Everything so far went through the Kubernetes **API server** on port 6443. `kubectl`
isn't magic — it's a friendly HTTP client for that REST API. Prove it three ways.

**See the actual HTTP calls** a command makes with `-v=6` (URLs) — bump to `-v=8` for
full request/response bodies:

```bash
kubectl get pods -v=6
# ... "Response" verb="GET" url="https://127.0.0.1:6443/api/v1/namespaces/workshop/pods" status="200 OK"
```

**Hit the API directly** — the raw JSON kubectl received, with no table formatting:

```bash
kubectl get --raw /api/v1/namespaces/workshop/pods | head -c 400; echo
```

**Talk to the API with plain `curl`** via a local proxy that handles auth/TLS for you:

```bash
kubectl proxy --port=8001 &
curl -s http://localhost:8001/api/v1/namespaces/workshop/pods | head -c 400; echo
curl -s http://localhost:8001/apis/apps/v1/namespaces/workshop/deployments | head -c 400; echo
kill %1   # stop the proxy
```

## Observations

- `kubectl create/scale/expose` are just `POST`/`PATCH`/`GET` against a REST API.
- The cluster *is* its API — Rancher, CI pipelines, Terraform and your scripts all
  drive the **same** endpoints. kubectl has no special powers.
- That's why manifests work: `kubectl apply` just `PUT`s an object into the API. Which
  leads straight into…

---

# 8. Declarative — the part that scales to real systems

Everything above was imperative (`kubectl create/scale/expose`). Under the hood each
command just created or edited an **API object**. We can write those objects as YAML
instead and apply them:

```bash
kubectl get deployment/web -o yaml > web.yaml   # this is what we've been building
kubectl apply -f web.yaml                        # idempotent: makes reality match the file
```

This is :star2: Infrastructure as Code :star2: for your whole system — version it,
review it, re-apply it. The exercises build up to writing these manifests yourself.

---

# 9. Cleanup

```bash
kubectl delete deployment/web service/web
# or wipe everything in your namespace:
kubectl delete all --all
```

## Key takeaways

- You declare desired state; Kubernetes continuously reconciles it.
- Self-healing, scaling, rolling updates, and service discovery come built in.
- `kubectl` is just a friendly client for the Kubernetes API.
- The same objects can be imperative commands *or* declarative YAML — real systems
  use YAML.
