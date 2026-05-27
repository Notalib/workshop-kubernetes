# Exercises for Kubernetes Workshop

This repo contains hands-on exercises for the Kubernetes workshop — the sequel to the
[Containerisation workshop](https://github.com/notalib/workshop-containerisation).
Workshop #1 was about building and running *one* container. This one is about
**deploying, scaling, exposing, and continuously running** containers on a cluster.

## Repo layout

- [Kubernetes_workshop.pptx](#) — theory parts presented during the workshop.
- [setup/](./setup/README.md) — **do this first.** Get a local cluster and your own
  namespace.
- [kubectl-demo/](./kubectl-demo/README.md) — live-demo showing the power of Kubernetes
  via `kubectl`. Run by the facilitator during the workshop.
- `<num>-<name>/` — exercise modules for **you** to work through, in order. Each has a
  `README.md` with `TASK` blocks and starter manifests with `TODO`s to fill in.
- `edu-<name>/` — finished examples to read and run, not exercises.
- [helm-teaser/](./helm-teaser/README.md) — facilitator demo teasing the Helm workshop.
- [live-demo/](./live-demo/README.md) — facilitator cheat-sheet for demoing the real cluster.
- [deck-notes/](./deck-notes/README.md) — supporting notes for the presentation.

## Prerequisites

- Install **Rancher Desktop (Recommended!)** and enable Kubernetes.
  - Make sure `~/.rd/bin` is on your `PATH` so `kubectl` resolves to Rancher's.
  - It ships a container runtime, `kubectl`, a **Traefik** ingress controller, and a
    `local-path` storage class — everything these exercises need.
  - If you prefer **kind** or **k3d**, that works too; see notes in
    [setup/](./setup/README.md) and module 4.
- Confirm `kubectl get nodes` shows a `Ready` node.
- Then follow [setup/](./setup/README.md) to create and select your namespace.

## Docs

Keep these open while you work:
- <https://kubernetes.io/docs>
- Pods & Deployments: <https://kubernetes.io/docs/concepts/workloads/>
- Services & Ingress: <https://kubernetes.io/docs/concepts/services-networking/>
- `kubectl` reference: <https://kubernetes.io/docs/reference/kubectl/>
- An LLM (ChatGPT/Claude) knows a lot about Kubernetes — use it as a tutor.

## Modules

Work through them in order — each builds on the last.

### [1 — Pods & Deployments](./1-pods-deployments/README.md)
Create an nginx Deployment, scale it, exec in, kill a Pod and watch it heal,
port-forward, roll out a change. Then write it as a YAML manifest.

### [2 — Configuration & Environment](./2-config-and-env/README.md)
Move content and config out of the image: ConfigMaps (incl. `subPath`), Secrets, and
environment variables — set directly and imported from a ConfigMap.

### [3 — Volumes & Persistence](./3-volumes-and-persistence/README.md)
`emptyDir` shared inside a Pod, then a PersistentVolumeClaim that survives Pod
deletion. StorageClass → PVC → PV explained.

### [4 — Exposing Workloads](./4-exposing-workloads/README.md)
Put a Service in front of your Pods, then an Ingress so it's reachable from your
browser at `*.localhost`.

### [5 — Composite system (capstone)](./5-composite-system/README.md)
Wire the Spring Boot + Postgres app from Workshop #1 together yourself: two cooperating
Deployments, a Service for discovery, a ConfigMap + Secret, a PVC, and an Ingress —
everything from modules 1–4 in one real system.

### [edu — Multi-component system](./edu-multi-component/README.md)
A finished two-tier app (nginx → http-echo) wired by Services and an Ingress —
service discovery in action. Plus short notes on StatefulSets, DaemonSets, and
`kubectl debug`.

## How the exercises work

Each module teaches the same idea twice, mirroring Workshop #1's "single-stage then
multi-stage" progression:

1. **Imperative first** — `kubectl create/scale/expose/...` for instant feedback.
2. **Declarative second** — write the same object as a YAML manifest and
   `kubectl apply -f` it. This is how real systems are managed: versioned,
   reviewable, idempotent. Fill in the `TODO`s in each module's starter manifests.

### Solutions

If you're stuck or want to compare, completed manifests will be on the
[`solutions`](#) branch:

```bash
git checkout -t origin/solutions
```

## Essential kubectl

```bash
# What's in my namespace?
kubectl get all
kubectl get pods -o wide
kubectl get <type>/<name> -o yaml        # full object; -o wide for a table

# Inspect & troubleshoot
kubectl describe <type>/<name>           # events + current state
kubectl logs <pod> -f                    # stream stdout/stderr (-p for previous crash)
kubectl exec -it deployment/<name> -- sh # shell into a running Pod

# CRUD
kubectl create deployment <name> --image=<img>
kubectl edit <type>/<name>
kubectl delete <type>/<name>
kubectl apply -f <file-or-url>.yaml      # declarative; idempotent

# Namespaces
kubectl get pods -A                      # across ALL namespaces
kubectl config set-context --current --namespace=<ns>   # set your default

# Tip: shell completion
source <(kubectl completion bash)        # or zsh
```

## How to get unstuck

Before reaching for the solutions branch, these five tools resolve most problems:

- **`kubectl explain <type>.<field>`** — built-in schema docs, works offline. Drill in:
  `kubectl explain deployment.spec.template.spec.containers`. Every field is documented.
- **`kubectl create <thing> ... --dry-run=client -o yaml`** — scaffold a valid manifest
  to start from, then edit it. e.g. `kubectl create deployment x --image=nginx --dry-run=client -o yaml`.
- **`kubectl describe <type>/<name>`** — the **Events** at the bottom usually say exactly
  what's wrong (bad image, unbound PVC, failed mount, no endpoints).
- **`kubectl logs <pod>`** (add `--previous` for a crashed container) — the app's own errors.
- **`kubectl get <type>/<name> -o yaml`** — the full object including its live `status`.

Plus the [docs](https://kubernetes.io/docs), and an LLM makes a strong Kubernetes tutor.

## Best practices

- **Declare desired state; let Kubernetes reconcile it.** Don't script *how* — describe
  *what* you want and apply it.
- **Keep config and content out of images** — use ConfigMaps, Secrets, and env vars so
  changes don't need a rebuild and survive rollouts.
- **Pods are disposable.** Never store anything you care about in a Pod's filesystem;
  use a PVC for data that must persist.
- **Reach workloads by Service name, not Pod IP.** Pod IPs churn; Service DNS is stable.
- **Pin image tags** (e.g. `nginx:1.27`), don't use `:latest` — same rule as Workshop #1.
- **Namespace your work** so you can clean up in one command and avoid collisions.

## Tools worth knowing

- **k9s** — terminal dashboard for navigating cluster objects fast.
- **kubectl krew** — plugin manager for `kubectl`.
- **Rancher** (web + RBAC), **Headlamp** (desktop), **Kubernetes Dashboard** (web) — UIs.

## Bonus

1. **Deploy your own image from Workshop #1.** Once the Workshop #1 images are published
   to a registry, deploy one to your cluster, expose it with a Service, and route to it
   with an Ingress — the same artifact you built, now running on Kubernetes. (See
   [deck-notes/reuse-workshop1-images.md](./deck-notes/reuse-workshop1-images.md).)
2. Bring the whole `edu-multi-component` system up and add a second backend behind the
   same Ingress on a different host.
3. Explore the cluster with **k9s** instead of raw `kubectl` for a module.
4. Tear everything down with a single `kubectl delete namespace workshop` and confirm
   it's all gone with `kubectl get all -A | grep workshop`.
