# 2 — Exposing Workloads

Using `kubectl port-forward` was fine for debugging, but it's a one-off tunnel from your laptop.
Real traffic reaches workloads through two stable objects:
- **Service** (stable identity *inside* the cluster, like a DNS)
- **Ingress** (the door *into* the cluster from outside).

To reduce confusion you may want to delete the deployment from exercise #1:
```bash
kubectl delete deploy/demo
```

You first need the `web-app` Deployment running - Apply the one provided:
```bash
kubectl apply -f app.yaml
```

> **Service** — one stable DNS name + IP for a moving set of Pods. Pods are
> short-lived and their IPs churn; the Service name doesn't. It finds its Pods by
> label **selector**. This is the cluster's internal DNS.
>
> **Ingress** — HTTP(S) routing from outside the cluster to a Service, by hostname
> and path. Handles external DNS and TLS termination. Needs an **ingress controller**
> running (Rancher Desktop / K3D ships **Traefik**).

---

## Pre-req: Ensure Rancher Desktop is exposed.

Confirm that Rancher Desktop has bound port 80 on your machine:

```bash
curl -s http://localhost
# You should see: 404 page not found
```

If you do NOT see a response, then follow these steps:

Temporary fix:
```bash
# Fix (survives until reboot):
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
```

Permanent fix:
```bash
sudo echo "net.ipv4.ip_unprivileged_port_start=80" >> /etc/sysctl.d/99-rancher.conf
sudo sysctl --system
```

Finally restart Rancher Desktop entirely
```bash
rdctl shutdown && rdctl start
```

Traefik should now have bound port 80. Test with the first cURL command.

---

## TASK 1: Put a Service in front of the Deployment

```bash
kubectl expose deployment/web-app --port=80
kubectl get service web-app
```

That created a `ClusterIP` Service named `web-app`. Test it **from inside the cluster** —
the Service is reachable by its DNS name, not from your laptop directly:

```bash
kubectl run test --rm -it --restart=Never --image=busybox -- wget -qO- http://web-app
```

Alternatively start a sleeping debugger:
```bash
kubectl run net-debug --image=nicolaka/netshoot --restart=Never -- sleep infinity
kubectl exec -it pod/net-debug -- sh

# Then inside the container, try this:
nslookup web-app
curl -si web-app
```

`http://web-app` resolved because the `net-debug` Pod is in the same namespace. The fully
qualified name is `web-app.default.svc.cluster.local` — try that form too. Notice you
never had to know a Pod IP.

> **How does the Service find its Pods?** By label selector. Check:
> `kubectl describe service/web-app` and look at `Selector:` and `Endpoints:`. The
> endpoints are the current Pod IPs — scale the Deployment and watch them change.

---

## TASK 2: Service types (read + try one)

- **ClusterIP** (default) — reachable only inside the cluster.
- **NodePort** — also opens a port on every node.
- **LoadBalancer** — asks the platform for an external IP (cloud, or Rancher's
  built-in on localhost).

For HTTP you usually keep ClusterIP and put an **Ingress** in front (next task) rather
than exposing each Service directly. But try a NodePort once to see the difference:

```bash
kubectl expose deployment/web-app --name=web-app-np --type=NodePort --port=80
kubectl get service web-app-np
# note the auto-generated high 3xxxx node port
```

---

## TASK 3: Reach it from your browser with an Ingress

Confirm the ingress controller is running:

```bash
# Rancher Desktop uses Traefik
kubectl get pods -n kube-system | grep traefik
```

Create an Ingress routing the host `localhost` to your Service:

```bash
kubectl create ingress web-app \
  --rule="localhost/=web-app:80" \
  --class=traefik
kubectl get ingress web-app
```

Now hit it from your host — no port-forward needed:

```bash
curl http://localhost
# OR open http://localhost in your browser
```

Inspect the routing:
```bash
kubectl describe ingress/web-app
```

> **Not on Rancher Desktop?** On **kind** there's no ingress controller by default —
> install one (e.g. `ingress-nginx`) and use `--class=nginx`. On **k3d** it's Traefik
> like Rancher. The only thing that changes is that the cluster must have been created
> with the parameter  `--port "127.0.0.1:80:80@loadbalancer"`

---

## TASK 4: Write the Service and Ingress yourself - as manifests

1. Now delete the imperative objects from earlier tasks & recreate them from YAML.

```bash
kubectl delete service/web-app service/web-app-np ingress/web-app
```

2. Fill in the `TODO`s in [`service.yaml`](./service.yaml) and [`ingress.yaml`](./ingress.yaml) and then apply them to the Kubernetes cluster.

```bash
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
curl http://localhost -H 'Host: localhost'
```

---

## Stuck?

- Field help: `kubectl explain service.spec` and `kubectl explain ingress.spec.rules`.
- Service returns nothing / has no endpoints? Its `selector` doesn't match your Pod
  labels: compare `kubectl describe service/web-app` with `kubectl get pods --show-labels`.
- Ingress 404s? `kubectl describe ingress/web-app` — check the host and `ingressClassName`,
  and that Traefik is running (`kubectl get pods -n kube-system | grep traefik`).
- Docs: <https://kubernetes.io/docs/concepts/services-networking/>

## BONUS

1. Scale `web-app` to 3 replicas and `curl http://localhost` repeatedly while
   running `kubectl get endpoints web-app -w`. The Service load-balances across Pods.
2. Add a second path/host to the Ingress pointing at a *different* Service (deploy
   `traefik/whoami` and route `whoami.localhost` to it). One ingress controller, many
   apps.
3. The full internal DNS name is `<service>.<namespace>.svc.cluster.local`. From a
   Pod in *another* namespace, can you reach `web-app.default.svc`? Why is the namespace
   part needed there but not in TASK 1? See `setup/namespaces.md` to understand how to create one and create objects inside it.

   Hint: Check the `/etc/resolv.conf` file inside a container.
