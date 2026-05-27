# 3 — Volumes & Persistence

ConfigMaps inject config *in*. This module is about **storage**: scratch space shared
between containers in a Pod, and durable storage that outlives the Pod entirely.

Work in your `workshop` namespace.

> The persistence chain, top to bottom:
> **Pod** → **PersistentVolumeClaim (PVC)** "give me 1Gi of storage" → bound to a
> **PersistentVolume (PV)** "an actual piece of storage" → provisioned by a
> **StorageClass** "what *kind* of storage" (e.g. `local-path`, NFS, Longhorn, S3).
> You write the PVC; the StorageClass's provisioner creates the PV for you.

---

## TASK 1: `emptyDir` — scratch space shared inside a Pod

An `emptyDir` volume is created when a Pod starts and deleted when the Pod dies. Its
point is **sharing between containers in the same Pod** (recall: containers in a Pod
share network, but *not* filesystem unless you mount something).

Apply a Pod with a `writer` and a `reader` container sharing one `emptyDir`:

```bash
kubectl apply -f pod-emptydir.yaml
kubectl logs pod/shared -c reader --follow
```

The reader prints what the writer writes — through the shared volume. Now prove it's
ephemeral:

```bash
kubectl delete pod/shared
kubectl apply -f pod-emptydir.yaml
kubectl logs pod/shared -c reader
```

The data is gone — `emptyDir` starts empty every time the Pod is (re)created.

---

## TASK 2: A PersistentVolumeClaim that outlives the Pod

First, confirm you have a default StorageClass to provision from:

```bash
kubectl get storageclass
# local-path  ...  (default)     ← Rancher Desktop ships this
```

> **Seeing more than one `(default)`?** If `get storageclass` lists two defaults
> (e.g. both `local-path` and `longhorn`), the choice is ambiguous and Kubernetes may
> pick the "wrong" one. In that case add an explicit `storageClassName: local-path`
> under `spec:` in `pvc.yaml` so everyone gets the same, fast, local storage.

Open [`pvc.yaml`](./pvc.yaml), fill in the `TODO`s (size and access mode), and create
the claim:

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```

> With `local-path`, the PVC may stay `Pending` until a Pod actually uses it
> (`WaitForFirstConsumer`). That's expected — it binds in the next step.

Now run a Pod that mounts the PVC and writes a file. Open
[`pod-pvc.yaml`](./pod-pvc.yaml), fill in the `claimName` TODO, and apply:

```bash
kubectl apply -f pod-pvc.yaml
kubectl exec -it pod/keeper -- sh -c 'echo "written at $(date)" >> /data/log.txt; cat /data/log.txt'
kubectl get pvc        # now Bound
```

---

## TASK 3: Prove persistence — kill the Pod, keep the data

```bash
kubectl delete pod/keeper
kubectl apply -f pod-pvc.yaml
kubectl exec -it pod/keeper -- cat /data/log.txt
```

Your line from before is **still there** — the data lives on the PV, not in the Pod.
Append again and you'll see history accumulate across Pod lifetimes.

Now delete the claim and watch the data go:

```bash
kubectl delete pod/keeper
kubectl delete pvc/demo-data       # releases the PV; with Delete reclaim policy, storage is freed
```

**Reclaim policy** decides what happens to the PV when its PVC is deleted: `Delete`
(default for dynamic provisioning — storage is wiped) vs `Retain` (PV is kept around,
to be reclaimed manually). Check yours: `kubectl get pv`.

---

## Stuck?

- Field help: `kubectl explain pvc.spec` and `kubectl explain pod.spec.volumes`.
- PVC stuck `Pending`? `kubectl describe pvc/demo-data` (read the Events) and check
  `kubectl get storageclass` — see the multiple-defaults note above.
- Pod stuck `ContainerCreating`? It's usually the volume — `kubectl describe pod/keeper`.
- Docs: <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>

## BONUS

1. A Deployment with `replicas: 3` and one `ReadWriteOnce` PVC — what happens? (RWO =
   mountable by one node at a time. This is exactly the problem **StatefulSets** solve
   by giving each replica its *own* PVC — see [edu-multi-component](../edu-multi-component/README.md).)
2. Inspect where `local-path` actually put your data on the node:
   `kubectl get pv -o jsonpath='{.items[0].spec.hostPath.path}{"\n"}'`.
3. Change the PVC's `accessModes` to `ReadWriteMany` and re-create it. Does
   `local-path` support it? (Most local provisioners don't — RWX needs NFS/Longhorn/etc.)
