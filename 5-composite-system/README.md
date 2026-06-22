# 5 — Composite system (capstone)

Time to wire a **real two-tier app** together yourself: the Spring Boot + Postgres
"greetings" app from [Workshop #1](https://github.com/notalib/workshop-containerisation/tree/main/edu-spring-postgres),
now running on Kubernetes. This pulls together everything from modules 1–4:

- two **Deployments** that must cooperate (backend → database),
- a **Service** so the backend finds the database by name (service discovery),
- a **ConfigMap** for non-secret config and a **Secret** for the DB password,
- a **PVC** so the database survives restarts,
- an **Ingress** so you can reach the app from your browser.

Continue work in your `default` namespace. Or create a new one and switch to it, it's up to you.

```bash
# Create a new namespace
kubectl create namespace complex-app
# Set it as default for commands without -n flag
kubectl config set-context --current --namespace=complex-app
```

```
  Request → Ingress (localhost) → Service (backend) → Spring Boot Pod
                                                               │
                                                               │ jdbc:postgresql://${POSTGRES_HOST}:5432/${POSTGRES_DB}
                                                               │
                                                               ▼
                                                   Service "database" → Postgres Pod → PVC
```

> **The key idea — three things must agree.** The backend is a "12-factor app": it reads
> its database host from the **`POSTGRES_HOST`** environment variable (defaulting to
> `db`). So service discovery here is a contract between three places that must all
> match:
> 1. the **ConfigMap** value `POSTGRES_HOST` (we use `database`),
> 2. the **name of the database Service** you create, and
> 3. the **`POSTGRES_HOST` env** the backend imports from the ConfigMap.
>
> Name the Service `database`, set `POSTGRES_HOST=database`, wire it into the backend —
> and they find each other. Change one without the others and the backend can't connect.
> That's service discovery + config, exactly as the 12-factor "backing services"
> principle describes.

## Prerequisite: the backend image

This uses the image you built in Workshop #1, published to GHCR:
`ghcr.io/notalib/workshop-containerisation/edu-spring-boot:1.2`. It must be **public** so
your cluster can pull it without credentials. (Postgres is the stock public
`postgres` image.)

> The image must be the build that (a) reads `POSTGRES_HOST` from the environment
> (`spring.datasource.url=jdbc:postgresql://${POSTGRES_HOST:db}:...`) and (b) exposes the
> `/healthz` and `/readyz` health endpoints used by the probes in TASK 3.

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

- the **Service name** — must match `POSTGRES_HOST` in the ConfigMap (`database`),
- the database password, pulled from the **Secret** (`secretKeyRef`),
- the **PVC** so data persists.

```bash
kubectl apply -f postgres.yaml
kubectl rollout status deployment/database
kubectl get pvc,svc -l app=database
```

Confirm Postgres is alive:

```bash
kubectl exec deployment/database -- pg_isready -U postgres
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
# look for Hikari/JPA connecting to jdbc:postgresql://database:5432/example — Ctrl+C when up
```

> **Health probes (in the manifest for you).** The backend declares two probes:
> - **readiness** → `GET /readyz` (checks the DB). The Pod only joins the Service's
>   endpoints once this passes — so traffic is never sent to a Pod that's still booting
>   or whose DB is down. This is what removes the brief 502 you'd otherwise see right
>   after a rollout.
> - **liveness** → `GET /healthz` (process only, no DB). If the app wedges, Kubernetes
>   restarts the container. It deliberately does *not* check the DB — otherwise a DB blip
>   would restart every backend Pod at once.
>
> Watch readiness in action: `kubectl get pods -l app=backend -w` — the Pod stays
> `0/1` (Running but not Ready) until `/readyz` succeeds, then flips to `1/1`.

If the backend can't reach the DB, the logs will say so — check that `POSTGRES_HOST`,
the Service name, and the ConfigMap all say `database` (TASK 2 + 3).

---

## TASK 4: Use the app, end to end

```bash
curl http://localhost/greetings        # list greetings (served from Postgres)
```

Add one through the form at <http://localhost/new>, then prove it really
landed in the database by querying Postgres directly:

```bash
kubectl exec deployment/database -- psql -U postgres -d example -c "SELECT * FROM greetings;"
```

---

## TASK 5: Prove the tiers are independent

```bash
# Kill the backend — the database (and your data) is untouched:
kubectl delete pod -l app=backend
kubectl rollout status deployment/backend
curl http://localhost/greetings         # still there

# Kill the database Pod — it restarts and reattaches the SAME PVC:
kubectl delete pod -l app=database
kubectl rollout status deployment/database
kubectl exec deployment/database -- psql -U postgres -d example -c "SELECT count(*) FROM greetings;"
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

1. **Init container — replace the seed button with a migration step.**
   The "Seed default data" button works, but it's a manual step and it's in the wrong layer
   (the app shouldn't own schema management). The cloud-native pattern is an **init container**:
   a short-lived container that runs to completion *before* the main container starts.

   Uncomment the `initContainers:` block in `backend.yaml` and re-apply:

   ```bash
   kubectl apply -f backend.yaml
   kubectl get pods -l app=backend -w   # watch: Init:0/1 → PodInitializing → Running
   kubectl logs -l app=backend -c migrate  # init container logs: pg_isready + psql output
   ```

   The Pod stays in `Init:0/1` until Postgres is reachable and the SQL succeeds. Only then
   does the backend container start — the opposite of the readiness-probe dance you saw
   earlier. This also means you can delete the postgres Pod during the init phase and watch
   the init container retry automatically.

   > **Why this beats startup code:** the init container and the app run in separate
   > containers with separate concerns. The app has `ddl-auto=none` and no SQL init — it
   > simply assumes the schema exists. Schema management is now an explicit, observable,
   > retriable step in the Pod lifecycle rather than hidden inside app startup.

2. Move Postgres to a **StatefulSet** with a `volumeClaimTemplate` instead of a
   Deployment + PVC. Why is that the more correct choice for a database? (See
   [edu-multi-component](../edu-multi-component/README.md).)

2. Scale the **backend** to 3 replicas. It works because the backend is stateless — all
   state lives in Postgres. Could you scale Postgres the same way? Why not?

3. Rotate the DB password: change the Secret, then restart both Deployments. What has
   to happen for both tiers to pick up the new value?

4. **Init container** — replace the seed button with a migration step.
   The "Seed default data" button works, but it's a manual step and it's in the wrong layer
   (the app shouldn't own schema management). The cloud-native pattern is an **init container**:
   a short-lived container that runs to completion *before* the main container starts.

   Insert this `initContainers:` block in `backend.yaml` and re-apply:

   ```yaml
      # BONUS 4: uncomment initContainers to replace the manual "Seed default data" button.
      # The init container runs to completion BEFORE the backend starts:
      #   1. waits until Postgres accepts connections (pg_isready loop),
      #   2. creates the schema (idempotent — CREATE TABLE IF NOT EXISTS),
      #   3. inserts the default rows (ON CONFLICT DO NOTHING).
      # The Pod stays in Init:0/1 until this succeeds, then the backend container starts.
      #
      initContainers:
        - name: migrate
          image: postgres:18.3-alpine3.23
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: POSTGRES_DB
            - name: POSTGRES_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: POSTGRES_HOST
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          command:
            - sh
            - -c
            - |
              until pg_isready -h $POSTGRES_HOST -p 5432; do
                echo "waiting for postgres..."; sleep 2
              done
              psql postgresql://postgres:$POSTGRES_PASSWORD@$POSTGRES_HOST/$POSTGRES_DB -c "
                CREATE TABLE IF NOT EXISTS GREETINGS (
                    id UUID PRIMARY KEY,
                    name varchar(50) NOT NULL UNIQUE
                );
                INSERT INTO GREETINGS(id, name)
                VALUES (gen_random_uuid(), 'Docker container'),
                       (gen_random_uuid(), 'An awesome workshop'),
                       (gen_random_uuid(), 'The Future')
                ON CONFLICT (name) DO NOTHING;
              "
   ```

   ```bash
   kubectl apply -f backend.yaml
   kubectl get pods -l app=backend -w   # watch: Init:0/1 → PodInitializing → Running
   kubectl logs -l app=backend -c migrate  # init container logs: pg_isready + psql output
   ```

   The Pod stays in `Init:0/1` until Postgres is reachable and the SQL succeeds. Only then
   does the backend container start — the opposite of the readiness-probe dance you saw
   earlier. This also means you can delete the postgres Pod during the init phase and watch
   the init container retry automatically.

   > **Why this beats startup code:** the init container and the app run in separate
   > containers with separate concerns. The app has `ddl-auto=none` and no SQL init — it
   > simply assumes the schema exists. Schema management is now an explicit, observable,
   > retriable step in the Pod lifecycle rather than hidden inside app startup.
