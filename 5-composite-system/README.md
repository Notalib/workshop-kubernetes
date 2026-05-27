# 5 — Composite system (capstone)

Time to wire a **real two-tier app** together yourself: the Spring Boot + Postgres
"greetings" app from [Workshop #1](https://github.com/notalib/workshop-containerisation/tree/main/edu-spring-postgres),
now running on Kubernetes. This pulls together everything from modules 1–4:

- two **Deployments** that must cooperate (backend → database),
- a **Service** so the backend finds the database by name (service discovery),
- a **ConfigMap** for non-secret config and a **Secret** for the DB password,
- a **PVC** so the database survives restarts,
- an **Ingress** so you can reach the app from your browser.

Work in your `workshop` namespace.

```
  Request → Ingress (greetings.localhost) → Service "backend" → Spring Boot Pod
                                                                     │ JDBC: jdbc:postgresql://${POSTGRES_HOST}:5432/${POSTGRES_DB}
                                                                     ▼
                                                Service "postgres" → Postgres Pod → PVC
```

> **The key idea — three things must agree.** The backend is a 12-factor app: it reads
> its database host from the **`POSTGRES_HOST`** environment variable (defaulting to
> `db`). So service discovery here is a contract between three places that must all
> match:
> 1. the **ConfigMap** value `POSTGRES_HOST` (we use `postgres`),
> 2. the **name of the Postgres Service** you create, and
> 3. the **`POSTGRES_HOST` env** the backend imports from the ConfigMap.
>
> Name the Service `postgres`, set `POSTGRES_HOST=postgres`, wire it into the backend —
> and they find each other. Change one without the others and the backend can't connect.
> That's service discovery + config, exactly as the 12-factor "backing services"
> principle describes.

## Prerequisite: the backend image

This uses the image you built in Workshop #1, published to GHCR:
`ghcr.io/notalib/workshop-containerisation/spring-postgres`. It must be **public** so
your cluster can pull it without credentials. (Postgres is the stock public
`postgres` image.)

> The image must be the build that reads `POSTGRES_HOST` from the environment
> (`spring.datasource.url=jdbc:postgresql://${POSTGRES_HOST:db}:...`). If you're using an
> older build that hard-codes the host to `db`, either rebuild it or just name your
> Postgres Service `db` and set `POSTGRES_HOST=db` to match.

> **Private registry instead (e.g. internal Harbor)?** You'd create an image-pull
> Secret and reference it on the Deployment:
> ```bash
> kubectl create secret docker-registry harbor-creds \
>   --docker-server=harbor.internal.kb.dk --docker-username=... --docker-password=...
> ```
> then add `spec.template.spec.imagePullSecrets: [{name: harbor-creds}]`. Public GHCR
> avoids all of this — recommended for the workshop.

---

## TASK 1: Create the config and the secret

Non-secret config goes in a ConfigMap; the DB password in a Secret. Both are provided —
apply them and look at what's inside:

```bash
kubectl apply -f configmap.yaml -f secret.yaml
kubectl get configmap/app-config -o yaml
kubectl get secret/db-credentials -o jsonpath='{.data.password}' | base64 --decode; echo
```

---

## TASK 2: Deploy Postgres (the database tier)

Open [`postgres.yaml`](./postgres.yaml) and fill in the `TODO`s:

- the **Service name** — must match `POSTGRES_HOST` in the ConfigMap (`postgres`),
- the database password, pulled from the **Secret** (`secretKeyRef`),
- the **PVC** so data persists.

```bash
kubectl apply -f postgres.yaml
kubectl rollout status deployment/postgres
kubectl get pvc,svc -l app=postgres
```

Confirm Postgres is alive:

```bash
kubectl exec deployment/postgres -- pg_isready -U postgres
```

---

## TASK 3: Deploy the backend (the app tier)

Open [`backend.yaml`](./backend.yaml) and fill in the `TODO`s:

- the database **name** and **host** from the **ConfigMap** (`configMapKeyRef`) —
  `POSTGRES_HOST` is what tells the backend where to find the DB Service,
- the password from the **Secret** (`secretKeyRef`) — the same one Postgres uses,
- the **Ingress** class and backend service.

```bash
kubectl apply -f backend.yaml
kubectl rollout status deployment/backend
```

Watch the backend connect to the database on startup:

```bash
kubectl logs deployment/backend -f
# look for Hikari/JPA connecting to jdbc:postgresql://postgres:5432/example — Ctrl+C when up
```

If the backend can't reach the DB, the logs will say so — check that `POSTGRES_HOST`,
the Service name, and the ConfigMap all say `postgres` (TASK 2 + 3).

---

## TASK 4: Use the app, end to end

```bash
curl http://greetings.localhost/greetings        # list greetings (served from Postgres)
```

Add one through the form at <http://greetings.localhost/new>, then prove it really
landed in the database by querying Postgres directly:

```bash
kubectl exec deployment/postgres -- psql -U postgres -d example -c "SELECT * FROM greetings;"
```

---

## TASK 5: Prove the tiers are independent

```bash
# Kill the backend — the database (and your data) is untouched:
kubectl delete pod -l app=backend
kubectl rollout status deployment/backend
curl http://greetings.localhost/greetings         # still there

# Kill the database Pod — it restarts and reattaches the SAME PVC:
kubectl delete pod -l app=postgres
kubectl rollout status deployment/postgres
kubectl exec deployment/postgres -- psql -U postgres -d example -c "SELECT count(*) FROM greetings;"
```

Your greetings survive a database Pod restart — that's the PVC from module 3 doing its
job. Each tier heals on its own; the Service names keep them wired together.

## Stuck?

- Backend `CrashLoopBackOff` or can't connect? `kubectl logs deployment/backend --previous`.
  99% of the time it's a mismatch between `POSTGRES_HOST`, the Service name, and the
  ConfigMap — or the password Secret not matching.
- Image won't pull (`ImagePullBackOff`)? `kubectl describe pod -l app=backend` — the
  GHCR package probably isn't public yet.
- Field help: `kubectl explain deployment.spec.template.spec.containers.env`.
- Docs: <https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/>

## BONUS

1. Move Postgres to a **StatefulSet** with a `volumeClaimTemplate` instead of a
   Deployment + PVC. Why is that the more correct choice for a database? (See
   [edu-multi-component](../edu-multi-component/README.md).)
2. Scale the **backend** to 3 replicas. It works because the backend is stateless — all
   state lives in Postgres. Could you scale Postgres the same way? Why not?
3. Rotate the DB password: change the Secret, then restart both Deployments. What has
   to happen for both tiers to pick up the new value?
