# Helm teaser (facilitator demo)

A short closing demo. By now attendees have hand-applied a dozen YAML files and, in
module 5, wired a whole system together manifest by manifest. The natural question is:
*"do I really copy-paste all this YAML for every app, every environment?"*

**No — you package it.** That's Helm, and it's the subject of the next workshop. This
demo plants the seed: a complex system as **one versioned, installable, rollback-able
package**.

> Run by the facilitator. `helm` ships with Rancher Desktop (`~/.rd/bin`). Verify with
> `helm version`.

---

## 1. The contrast: one command, a whole system

In module 5 you applied 4 files to stand up backend + Postgres. A Helm **chart** bundles
all of that — Deployments, Services, ConfigMaps, Ingress — behind a single install:

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update
helm install demo podinfo/podinfo -n helm-demo --create-namespace --set replicaCount=2
```

See what that one command created:

```bash
helm list -n helm-demo
kubectl get deploy,svc,pod -n helm-demo
```

> `podinfo` is just a small, reliable demo app. The point is the *packaging*, not the app.

---

## 2. A chart is configurable and inspectable

The `--set replicaCount=2` above is a **value** — charts expose knobs instead of
forcing you to edit raw YAML:

```bash
helm show values podinfo/podinfo | head -30   # every knob the chart exposes
helm get manifest demo -n helm-demo | head -40 # the actual YAML Helm rendered & applied
```

Same Kubernetes objects underneath — Helm just *templated* them from values.

---

## 3. Versioned releases: upgrade and roll back

This is the payoff that raw `kubectl apply` doesn't give you. Every change is a
numbered **revision** you can roll back to:

```bash
helm upgrade demo podinfo/podinfo -n helm-demo --set replicaCount=3
helm history demo -n helm-demo
#  1  ... superseded  Install complete
#  2  ... deployed    Upgrade complete

helm rollback demo 1 -n helm-demo      # back to 2 replicas, atomically
helm history demo -n helm-demo         # rollback is itself a new revision
```

Talking point: *the whole system's state is versioned, like git for your deployment.*

---

## 4. See it in the Rancher UI

Open Rancher Desktop → **Cluster Dashboard**, then **Apps → Installed Apps** (Helm
releases). The `demo` release shows up as **one managed application** with its chart
version and values — not a loose pile of Pods and Services. You can upgrade or
uninstall it from here too. (Charts can also be browsed and installed visually under
**Apps → Charts**.)

This is how teams actually ship and operate systems on Kubernetes.

---

## 5. The hook for Workshop #3

> "Remember the four files you wrote in module 5? Next workshop we'll turn *that* system
> into a chart — templated, versioned, installable in dev/staging/prod with one command
> and a values file per environment."

Tie back to the deck's closing themes: OCI (charts ship as OCI artifacts too) and the
12-factor app (config per environment, via values).

---

## Cleanup

```bash
helm uninstall demo -n helm-demo
kubectl delete namespace helm-demo
```
