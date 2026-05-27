# Live cluster demo (facilitator cheat-sheet)

Talking points and commands for showing the **real KB cluster** (dev/staging) during
the workshop. The value here is everything a single-node local cluster *can't* show:
multiple nodes, real ingress with DNS + TLS, real storage backends, RBAC, and
versioned releases actually running.

> **Safety:** demo against **dev/staging only**, and keep it read-only — `get`,
> `describe`, `logs`, `kubectl get -o yaml`. Don't `edit`/`delete` live workloads in
> front of the room. Pre-pick a namespace/app you're comfortable showing. Double-check
> your context before you start: `kubectl config current-context`.

Each section below is anchored to **when** in the workshop it lands best.

---

## A. During "Kubernetes Cluster" (deck slides 15–19) — the part local can't show

Their local cluster is a single node. The real one isn't — make the control-plane /
worker-node split concrete:

```bash
kubectl get nodes -o wide                 # multiple nodes, roles, versions, IPs
kubectl get pods -n kube-system -o wide    # control-plane components spread across nodes
kubectl top nodes                          # real resource usage per node
```

**Talking points:** point out control-plane vs worker roles (slide 17), that workloads
land on different nodes, and that losing one node doesn't take the app down — the
scheduler reschedules elsewhere.

---

## B. During "Namespaces" (deck slide 29)

```bash
kubectl get namespaces
kubectl get resourcequota -A                       # group-wide limits
kubectl describe namespace <a-team-namespace>      # labels, quotas
kubectl auth can-i create deployments -n <ns>      # RBAC in action
```

**Talking points:** namespaces as the per-team fence — quotas, RBAC, network
separation — the multi-tenant story behind "autonomous teams" from the strategy slides.

---

## C. After module 1 (Pods & Deployments) — self-healing & rollout at real scale

```bash
kubectl get deploy -n <app-ns>                     # real replica counts
kubectl rollout history deployment/<app> -n <app-ns>
kubectl get events -n <app-ns> --sort-by=.lastTimestamp | tail
```

**Talking points:** real apps run many replicas across nodes; rollouts and rollbacks
happen here exactly like their local `rollout restart`, just at scale.

---

## D. During module 3 (Volumes & Persistence)

Locally they only have `local-path`. Show real storage:

```bash
kubectl get storageclass                           # Longhorn / NFS / SMB, not just local-path
kubectl get pvc -A | head                          # real claims bound to real storage
kubectl get pv | head                              # the backing volumes + reclaim policies
```

**Talking points:** different StorageClasses for different needs (RWX via NFS/Longhorn,
reclaim policies), and that stateful data lives independently of Pods — same PVC concept
they just learned, backed by real infrastructure.

---

## E. After module 4 (Exposing Workloads) — the strongest "real" moment

Locally they reached `demo.localhost` via Traefik. Show a real app on a real domain
with TLS:

```bash
kubectl get ingress -A                             # real hosts: *-devel.kb.dk
kubectl describe ingress <app> -n <app-ns>         # rules, TLS secret, backend service
```

Then **open the real URL in a browser** — `https://<app>-devel.kb.dk` — and show the
padlock / certificate.

**Talking points:** external DNS + TLS termination (deck slide 52), the
Request → Ingress → Service → Pod path running for real.

---

## F. At the Helm teaser (see [../helm-teaser](../helm-teaser/README.md))

```bash
helm list -A                                       # real releases running in dev/staging
helm history <release> -n <ns>                     # real upgrade/rollback history
```

Then show the same releases in **Rancher → Apps**. Ties the teaser to "this is how we
actually run things here."

---

## G. (Optional) Monitoring & logging

If the cluster has it, this maps to the strategy slides' "ensartet monitorering og
log-opsamling":

- Open the **Grafana** dashboards / **Rancher** monitoring view.
- Show aggregated logs for a workload (Loki / your logging stack).

**Talking points:** uniform monitoring and log aggregation across all teams = the
"datadrevet drift" goal — something you get for free by standardizing on the platform.

---

## Suggested flow

A and B during Basics → C/D/E alongside the matching exercise modules → F/G as the
closing, leading into the Helm workshop tease.
