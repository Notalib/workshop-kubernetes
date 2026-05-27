# 4 — Exposing Workloads

`port-forward` was fine for debugging, but it's a one-off tunnel from your laptop.
Real traffic reaches workloads through two stable objects: a **Service** (stable
identity *inside* the cluster) and an **Ingress** (the door *into* the cluster from
outside).

Work in your `workshop` namespace. You need the `demo` nginx Deployment running — if
you tore it down, recreate it: `kubectl create deployment demo --image=nginx`.

> **Service** — one stable DNS name + IP for a moving set of Pods. Pods are
> short-lived and their IPs churn; the Service name doesn't. It finds its Pods by
> label **selector**. This is the cluster's internal DNS.
> **Ingress** — HTTP(S) routing from outside the cluster to a Service, by hostname
> and path. Handles external DNS and TLS termination. Needs an **ingress controller**
> running (Rancher Desktop ships **Traefik**).

---

## TASK 1: Put a Service in front of the Deployment

```bash
kubectl expose deployment/demo --port=80
kubectl get service demo
```

That created a `ClusterIP` Service named `demo`. Test it **from inside the cluster** —
the Service is reachable by its DNS name, not from your laptop directly:

```bash
kubectl run tester --rm -it --image=busybox:1.36 -- \
  wget -qO- http://demo
```

`http://demo` resolved because the tester Pod is in the same namespace. The fully
qualified name is `demo.workshop.svc.cluster.local` — try that form too. Notice you
never had to know a Pod IP.

> **How does the Service find its Pods?** By label selector. Check:
> `kubectl describe service/demo` and look at `Selector:` and `Endpoints:`. The
> endpoints are the current Pod IPs — scale the Deployment and watch them change.
> (On newer clusters `kubectl get endpoints` prints a deprecation note in favour of
> `kubectl get endpointslices` — both show the same thing.)

---

## TASK 2: Service types (read + try one)

- **ClusterIP** (default) — reachable only inside the cluster.
- **NodePort** — also opens a port on every node.
- **LoadBalancer** — asks the platform for an external IP (cloud, or Rancher's
  built-in on localhost).

For HTTP you usually keep ClusterIP and put an **Ingress** in front (next task) rather
than exposing each Service directly. But try a NodePort once to see the difference:

```bash
kubectl expose deployment/demo --name=demo-np --type=NodePort --port=80
kubectl get service demo-np      # note the high 3xxxx node port
```

---

## TASK 3: Reach it from your browser with an Ingress

Confirm the ingress controller is running:

```bash
kubectl get pods -n kube-system | grep traefik     # Rancher Desktop: Traefik
```

Create an Ingress routing the host `demo.localhost` to your Service:

```bash
kubectl create ingress demo \
  --rule="demo.localhost/=demo:80" \
  --class=traefik
kubectl get ingress demo
```

Now hit it from your host — no port-forward needed:

```bash
curl http://demo.localhost
# open http://demo.localhost in a browser
```

`*.localhost` resolves to `127.0.0.1` automatically, and Traefik listens on port 80,
so this Just Works. Inspect the routing:

```bash
kubectl describe ingress/demo
```

> **Not on Rancher Desktop?** On **kind** there's no ingress controller by default —
> install one (e.g. `ingress-nginx`) and use `--class=nginx`. On **k3d** it's Traefik
> like Rancher. The only thing that changes is the `--class` value and how you reach
> port 80.

---

## TASK 4: Write the Service and Ingress as manifests

Delete the imperative objects and recreate them from YAML. Fill in the `TODO`s in
[`service.yaml`](./service.yaml) and [`ingress.yaml`](./ingress.yaml):

```bash
kubectl delete service/demo service/demo-np ingress/demo
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
curl http://demo.localhost
```

---

## Stuck?

- Field help: `kubectl explain service.spec` and `kubectl explain ingress.spec.rules`.
- Service returns nothing / has no endpoints? Its `selector` doesn't match your Pod
  labels: compare `kubectl describe service/demo` with `kubectl get pods --show-labels`.
- Ingress 404s? `kubectl describe ingress/demo` — check the host and `ingressClassName`,
  and that Traefik is running (`kubectl get pods -n kube-system | grep traefik`).
- Docs: <https://kubernetes.io/docs/concepts/services-networking/>

## BONUS

1. Scale `demo` to 3 replicas and `curl http://demo.localhost` repeatedly while
   running `kubectl get endpoints demo -w`. The Service load-balances across Pods.
2. Add a second path/host to the Ingress pointing at a *different* Service (deploy
   `traefik/whoami` and route `whoami.localhost` to it). One ingress controller, many
   apps.
3. The full internal DNS name is `<service>.<namespace>.svc.cluster.local`. From a
   Pod in *another* namespace, can you reach `demo.workshop`? Why is the namespace
   part needed there but not in TASK 1?
