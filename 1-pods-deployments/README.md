# 1 — Pods & Deployments

Your first workload. You'll create an nginx Deployment, poke it, break it, and watch
Kubernetes heal it. Then you'll write the same thing as a YAML manifest.

Work in your `workshop` namespace (see [../setup](../setup/README.md)). Have the docs
handy: <https://kubernetes.io/docs>.

> **Pod** = the smallest deployable unit: one or more containers that share an IP and
> run as one unit.
> **Deployment** = declarative management of a set of identical Pods (via a
> ReplicaSet). It keeps *N* copies running and handles rollouts.

---

## TASK 1: Create an nginx Deployment (imperative)

```bash
kubectl create deployment demo --image=nginx
kubectl get all
```

Find your Pod and the ReplicaSet the Deployment created for you:

```bash
kubectl get pods
kubectl describe deployment/demo
```

---

## TASK 2: Scale it up and down ⛰

```bash
kubectl scale deployment/demo --replicas=3
kubectl get pods -o wide        # three Pods now
kubectl scale deployment/demo --replicas=1
```

What changed in `kubectl get all` each time? Which object actually tracks the replica
count — the Deployment or the ReplicaSet?

---

## TASK 3: Exec into a running Pod

```bash
kubectl exec -it deployment/demo -- /bin/sh
# inside the container:
ls /usr/share/nginx/html
cat /usr/share/nginx/html/index.html
exit
```

You're now inside the same isolation as the container — just like `docker exec`, but
`kubectl` found the Pod for you via the Deployment.

---

## TASK 4: Kill your Pod 😈

Watch your Pods in one terminal:

```bash
kubectl get pods --watch
```

In another, delete one:

```bash
kubectl delete pod -l app=demo
```

**What happened?** The Deployment's desired state is "1 replica", so the controller
immediately creates a replacement. You can't get rid of the workload by deleting the
Pod — you have to change the desired state (scale to 0, or delete the Deployment).

---

## TASK 5: Port-forward to your machine

```bash
kubectl port-forward deployment/demo 8080:80
# open http://localhost:8080  → the nginx welcome page
```

`8080:80` means **local 8080 → container 80**. Ctrl+C to stop. This is a debugging
shortcut, not how real traffic reaches the cluster (that's module 4).

---

## TASK 6: Re-deploy and observe a rollout

```bash
kubectl rollout restart deployment/demo
kubectl rollout status deployment/demo
kubectl get pods            # note the Pod name changed — it's a fresh Pod
```

**What happened?** A rollout replaced the old Pod with a new one. Any change you made
*inside* the old Pod's filesystem is gone — Pods are disposable.

---

## TASK 7: Change the HTML nginx serves

First, the quick (and doomed) way — edit it live inside the Pod:

```bash
kubectl exec -it deployment/demo -- /bin/sh -c \
  'echo "<h1>Hello from the workshop</h1>" > /usr/share/nginx/html/index.html'
kubectl port-forward deployment/demo 8080:80   # confirm it shows your text
```

Now trigger a rollout and check again:

```bash
kubectl rollout restart deployment/demo
kubectl port-forward deployment/demo 8080:80
```

**Does your change survive?** No — the new Pod started from the unmodified image. This
is the motivation for the next module: configuration and content belong **outside** the
container image, mounted in. (You'll serve custom HTML properly with a ConfigMap in
[module 2](../2-config-and-env/README.md).)

---

## TASK 8: Now write it as a manifest (declarative)

Everything above was imperative. Real systems are described as YAML you can review and
version. Open [`deployment.yaml`](./deployment.yaml) and fill in the `TODO`s, then:

```bash
kubectl delete deployment/demo            # remove the imperative one first
kubectl apply -f deployment.yaml
kubectl get deployment/demo
```

Re-run `kubectl apply -f deployment.yaml` after editing the replica count — notice it
*updates* rather than erroring. That idempotency is the whole point of declarative.

> **Cheat:** `kubectl create deployment demo --image=nginx --dry-run=client -o yaml`
> prints a starting manifest you can compare against.

---

## BONUS

1. `kubectl get pod <name> -o yaml` — read the full Pod object Kubernetes filled in.
   How much did it add beyond what you specified?
2. Scale to `--replicas=0`. Is the Deployment gone? What does `get all` show?
3. Add a second container to the Pod template (e.g. a `busybox` running
   `sleep 3600`). Both share the Pod's network — from one, can you reach the other on
   `localhost`?
