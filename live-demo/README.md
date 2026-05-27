# Live cluster demo (facilitator cheat-sheet) — Merkur on Nota-BETA

Talking points for demoing the **real Merkur application** running on the beta cluster
(<https://rancher.beta.dbb.dk>, cluster `local` / Nota-BETA), driven from the **Rancher
UI**. The value here is everything a single-node local cluster *can't* show: multiple
nodes, a real Helm-managed composite system, real Ingress + DNS + TLS, Longhorn storage,
and monitoring/logging.

> **Setup:** in Rancher, set the top namespace filter to **`merkur`** so the views match
> these notes.
>
> **Safety:** this is **beta/staging**, so you *can* make changes live (scale, redeploy)
> — but **revert them afterwards** (scale back, etc.) and never demo destructive actions
> on production. Each section says what's safe.

Merkur is a real composite system in the `merkur` namespace — roughly:
`merkur-nota` (the public app, 3 replicas), `merkur-internal`, `merkur-dodp`,
`merkur-liveupdate`, `merkur-kibana`, an `importer`, and two `redis` caches, with
Elasticsearch/Longhorn-backed storage behind it. It's deployed as the Helm release
**`merkur`** (currently on **revision 266**).

Each section is anchored to **when** in the workshop it lands best.

---

## A. During "Kubernetes Cluster" (deck slides 15–19) — multi-node, which local can't show

1. **Cluster → Nodes**: point out the worker nodes `betaclustint01/02/03`.
2. **Workloads → Deployments → `merkur-nota` → Pods tab**: the 3 replicas run on
   *three different nodes* (`betaclustint01`, `02`, `03`), each with its own Pod IP
   (`10.42.x.x`).

**Talking point:** "This is the same Deployment you wrote locally — but its replicas are
spread across three machines. Lose a node and the scheduler reschedules its Pods onto
the others; the app stays up." (Safe — read-only.)

---

## B. During "Namespaces" (deck slide 29)

The namespace filter you set to `merkur` *is* the concept — Merkur's whole world lives
in one namespace. Drop the filter to show the other teams' namespaces alongside it, and
mention Rancher **Projects**, resource quotas, and RBAC as the per-team fence. (Safe.)

---

## C. After module 1 (Pods & Deployments) — self-healing & scaling, via the UI

On **`merkur-nota`**:

- **Scale:** use the **Pods** card's `− / +` control to go 3 → 4. Watch a new Pod get
  scheduled (Pods tab). Then **scale back to 3.**
- **Self-heal:** in the Pods tab, pick one Pod → `⋮` → **Delete**. Watch the Deployment
  immediately create a replacement to satisfy "3 replicas".
- **Rollout:** the **Redeploy** button is the UI's `kubectl rollout restart` — it rolls
  Pods one at a time with zero downtime.

**Talking point:** "Exactly your local 'kill the Pod' exercise — on a real app, in a
browser." (Safe on beta as long as you scale back; avoid Redeploy if the app is in use.)

---

## D. Config & composite systems (modules 2 & 5)

Merkur is the real version of your module-5 capstone, just bigger:

- **Service Discovery → Services**: components find each other by Service name (e.g. the
  app talking to `redis-cache`, Elasticsearch) — no hardcoded IPs, same as your `db`
  Service.
- **More Resources → ConfigMaps / Secrets** (or Storage): show that Merkur's config and
  credentials live outside the images, injected at runtime — your module-2 pattern.

**Talking point:** "You wired two components together in module 5. Merkur wires ~9 the
same way — Services for discovery, ConfigMaps/Secrets for config." (Safe — read-only.)

---

## E. During module 3 (Volumes & Persistence)

Locally they only had `local-path`. Here:

- **Longhorn** (sidebar) — a real distributed storage backend.
- **Storage → PersistentVolumeClaims** — real claims bound to Longhorn volumes.
- **Workloads → StatefulSets** — the stateful tier (e.g. the cache/search), which gets
  stable identity + its own storage (the module-3 BONUS, for real).

**Talking point:** "Real StorageClasses for real needs — replication, snapshots,
reclaim policies — but the PVC concept is exactly what you just learned." (Safe.)

---

## F. After module 4 (Exposing Workloads) — the strongest "real" moment

On **`merkur-nota` → Ingresses tab** (it has 2):

- show the hosts — e.g. **`merkur.beta.dbb.dk`** (and `merkur-beta.nota.dk`) — and the
  TLS config.
- then **open `https://merkur.beta.dbb.dk` in a browser** and show the padlock /
  certificate.

**Talking point:** "Locally you reached `demo.localhost` through Traefik. Here it's a
real public hostname with real DNS and TLS termination — Request → Ingress → Service →
Pod, in production." (Safe — read-only.) *(Staging cluster hostnames look the same
shape, e.g. `merkur.beta.dbb.dk`.)*

---

## G. At the Helm teaser (see [../helm-teaser](../helm-teaser/README.md)) — the killer example

`merkur-nota`'s labels show **`app.kubernetes.io/managed-by: Helm`**, release
**`merkur`**, **revision 266**.

- In the UI: **Apps → Installed Apps → `merkur`** — the whole system as *one* release,
  with its values and revision history. Or via CLI:
  ```bash
  helm history merkur -n merkur
  ```

**Talking point:** "This entire application — every Deployment, Service, Ingress, cache
— is one Helm release that's been upgraded **266 times**, each a versioned release you
could roll back to. *That's* what the next workshop is about: packaging the system you
built in module 5 into a chart like this." (Read-only — do **not** roll back a real
release in front of the room.)

---

## H. (Optional) Monitoring & logging — "datadrevet drift" (strategy slides)

- Rancher **Monitoring** and **Logging** views for the cluster.
- Your existing stack: **Graylog** (logs), **Zabbix** (metrics) — whatever you point at.

**Talking point:** "Uniform monitoring and log aggregation across every team — the
datadrevet-drift goal you get for free by standardizing on the platform." (Read-only.)

---

## Suggested flow

A + B during Basics → C/D/E/F alongside the matching exercise modules → G + H as the
close, leading into the Helm workshop tease.
