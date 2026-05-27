# edu — A multi-component system

**This is not an exercise — it's a worked example to read and run.** It ties the
previous modules together: two Deployments, two Services, a ConfigMap, and an Ingress,
all wired by Kubernetes' internal DNS. The whole system is one `kubectl apply`.

The topology:

```
  Request → Ingress (app.localhost) → Service "web" → nginx Pods
                                                          │  proxy_pass http://echo
                                                          ▼
                                        Service "echo" → http-echo Pods
```

`web` (nginx) doesn't know the backend's Pod IPs — it just talks to the **Service
name** `echo`, and the cluster's DNS resolves it to whichever backend Pods are healthy.
That's service discovery: the thing that makes micro-services possible.

## Run it

```bash
kubectl apply -f app.yaml
kubectl get all
curl http://app.localhost            # nginx proxies your request to the echo backend
```

You'll get back a JSON dump of the request — produced by the **backend**, delivered
through the **frontend**, routed by the **Ingress**. Three independent components,
zero hardcoded IPs.

## Things to try

```bash
# Scale the backend and watch the Service's endpoints follow:
kubectl scale deployment/echo --replicas=3
kubectl get endpoints echo
curl http://app.localhost            # responses now come from different backend Pods

# Kill a backend Pod — the frontend keeps working (Service reroutes to healthy Pods):
kubectl delete pod -l app=echo
curl http://app.localhost
```

## Observations

- Components are wired by **stable Service names**, not Pod IPs.
- Each tier scales and heals independently.
- The entire system is defined as code in one file — like Compose's payoff in
  Workshop #1, but with continuous reconciliation, scaling, and self-healing on top.

## Teardown

```bash
kubectl delete -f app.yaml
```

---

# Appendix: three concepts you'll meet next

These aren't in the hands-on modules, but you'll run into them quickly in production.

## StatefulSet — when Pods need identity

A Deployment's Pods are interchangeable cattle: `demo-7c9f-ab12`, `demo-7c9f-cd34`,
random names, any one is as good as another. A **StatefulSet** instead gives each Pod
a **stable, ordered identity** — `db-0`, `db-1`, `db-2` — and each gets its *own*
PersistentVolumeClaim that follows it across restarts. Use it for databases, queues,
and clustered systems where members must tell each other apart and keep their own
storage. (This is the answer to the BONUS in [module 3](../3-volumes-and-persistence/README.md):
one Deployment can't give each replica its own volume; a StatefulSet can.)

## DaemonSet — one Pod per node

A **DaemonSet** guarantees a copy of a Pod runs on **every node** (or every node
matching a selector). As nodes join the cluster they get the Pod; as they leave it's
cleaned up. Used for node-level agents: log collectors, monitoring daemons, storage
daemons — anything that must run everywhere.

## kubectl debug — getting a shell when there isn't one

Many production images are distroless (no shell, no tools) — you can't
`kubectl exec ... -- sh`. **`kubectl debug`** attaches a temporary debug container
sharing the target's namespaces. Two flavours:

```bash
# Ephemeral container: injected INTO a running Pod, shares its process & network
# namespace. Best for inspecting a live Pod without disturbing it.
kubectl debug -it <pod> --image=busybox:1.28 --target=<container>

# Copy: makes a COPY of the Pod with the debug container added (and optionally a
# changed command). Best when the original crashes on startup or you need to alter it.
kubectl debug <pod> -it --copy-to=<pod>-debug --container=<container> -- sh
```

Rule of thumb: **ephemeral** to look at a Pod that's running fine; **copy** to debug a
Pod that won't stay up or whose command you need to override. (See also
[deck-notes/debug-ephemeral-vs-copy.md](../deck-notes/debug-ephemeral-vs-copy.md).)
