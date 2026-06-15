# Health probes — liveness, readiness, startup

Two slides, mirroring the deck's "concept slide + manifest slide" pattern (cf. Pod →
Deployment → Service). **Placement:** the composite-systems section, right before the
`5-composite-system` exercise — that module's `backend.yaml` already ships these probes,
so this is the theory that explains the YAML they're about to apply. (Works equally well
parked just after the Deployment/rollout material if you'd rather teach it earlier.)

---

## Slide 1 — Health probes (concept)

> **"Running" is not the same as "ready".** A container can be up while the app inside
> is still booting, deadlocked, or cut off from its database. Probes let Kubernetes tell
> the difference — and act on it.

Three probe types, each answering a different question:

| Probe | Question | On failure |
|-------|----------|------------|
| **livenessProbe** | Is the process alive / not wedged? | **Restart** the container |
| **readinessProbe** | Can it serve traffic *right now*? | **Remove** Pod from the Service's endpoints (no restart) |
| **startupProbe** | Has a slow app finished booting? | Hold off liveness/readiness until it passes |

**The trap to teach (this is the whole point):**

> **Liveness must NOT check dependencies like the database.** If liveness pinged the DB
> and the DB blipped, *every* replica would fail liveness and restart **at once** — a
> self-inflicted outage (cascading restarts). Liveness checks only *this process*.
> **Readiness** is where you check the DB: a Pod that can't reach the DB is pulled out of
> rotation (no traffic) but keeps running, and rejoins automatically when the DB returns.

Probe mechanisms (same for all three): **`httpGet`** (most common), `tcpSocket`,
`exec` (run a command, exit 0 = healthy), or `grpc`.

---

## Slide 2 — Probes in our app (manifest + example)

The backend you wire up in **module 5** already carries these — two endpoints, two jobs:

```yaml
# in the backend container spec
readinessProbe:          # gates traffic — checks the DB
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
livenessProbe:           # restarts if wedged — process only, NO DB
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

What the two endpoints actually do (in `HealthController.java`):

| Endpoint | Probe | Checks | Returns |
|----------|-------|--------|---------|
| `/healthz` | liveness | nothing — just "the process answers" | `200 healthy` |
| `/readyz` | readiness | opens a DB connection (`isValid(2)`) | `200 ready` / `503 not-ready` |

**The payoff:** readiness is what removes the brief **502** right after a rollout —
Spring Boot takes a few seconds to start, so the new Pod stays *out* of the Service until
`/readyz` returns 200. Traffic only ever hits a Pod that can actually serve it.

**Better than `initialDelaySeconds: 30`:** that generous liveness delay is a crude way to
"give Spring time to boot." The clean fix is a **startupProbe** — it runs first, and
liveness doesn't even begin until startup succeeds, so you don't have to guess the boot
time:

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 5
  failureThreshold: 30   # up to 30 × 5s = 150s to boot, then liveness takes over
```

> Callback to self-healing (slide ~19 demo): liveness is the per-container version of the
> same reconcile loop — Kubernetes doesn't just keep the *Pod* count right, it keeps each
> container *healthy*, restarting or de-routing on your terms instead of guessing.
