# 2 — Configuration & Environment

In module 1, custom HTML you wrote inside a Pod vanished on the next rollout. The fix:
keep config and content **outside** the image and inject it at runtime. That's what
ConfigMaps, Secrets, and environment variables are for.

Work in your `workshop` namespace. Start from the Deployment you built in module 1
(`demo`, running nginx).

> **ConfigMap** — non-secret key/value config (files or strings).
> **Secret** — same idea, base64-encoded, for credentials (base64 is *encoding*, not
> *encryption* — see the note in TASK 3).
> **Volume mount** — a file or directory injected into a container at runtime. A
> ConfigMap/Secret can be *mounted as files* or *read as env vars*.

---

## TASK 1: Serve custom HTML from a ConfigMap (the right way)

Create a ConfigMap holding an `index.html`:

```bash
kubectl apply -f html-configmap.yaml
kubectl describe configmap/demo-html
```

Now mount it into nginx's web root. Open [`deployment.yaml`](./deployment.yaml), fill
in the `TODO`s under `volumes` and `volumeMounts`, then apply:

```bash
kubectl apply -f deployment.yaml
kubectl port-forward deployment/demo 8080:80    # http://localhost:8080
```

You should see your custom page. **Now the real test:**

```bash
kubectl rollout restart deployment/demo
kubectl port-forward deployment/demo 8080:80
```

The content **survives the rollout** — because it lives in the ConfigMap, not the
Pod's ephemeral filesystem. Edit `html-configmap.yaml`, re-`apply`, and restart to
publish a change.

---

## TASK 2: Mount a single file with `subPath`

Mounting a ConfigMap at a directory **replaces the whole directory**. Often you want
to drop in just *one* file (e.g. nginx's `default.conf`) and leave the rest alone.
That's what `subPath` does.

```bash
kubectl apply -f nginx-conf-configmap.yaml
```

In [`deployment.yaml`](./deployment.yaml) there's a second, commented-out
`volumeMount` for the nginx config that uses `subPath`. Enable it, apply, and confirm
nginx now returns the custom response:

```bash
kubectl port-forward deployment/demo 8080:80
curl http://localhost:8080/anything   # → the "Dead-end" 404 from the config
```

**Experiment:** remove the `subPath:` line, re-apply, and restart. nginx breaks — why?
(Hint: without `subPath`, the mount replaces all of `/etc/nginx/conf.d/`, including
files nginx expects to be there. `subPath` mounts only the single file.)

---

## TASK 3: A Secret

```bash
kubectl create secret generic demo-secret --from-literal=api-key=sh-its-a-secret
kubectl get secret/demo-secret -o yaml
```

Look at the value — it's base64. Decode it:

```bash
kubectl get secret/demo-secret -o jsonpath='{.data.api-key}' | base64 --decode; echo
```

> ⚠️ **base64 is encoding, not encryption.** Anyone who can read the Secret can read
> the value. Secrets exist to keep credentials out of ConfigMaps/images and to gate
> access via RBAC — not to hide them from someone with read access.

---

## TASK 4: Environment variables, set directly

The simplest config: key/value `env` entries on the container. Add this to the nginx
container in [`deployment.yaml`](./deployment.yaml):

```yaml
          env:
            - name: GREETING
              value: "hello from env"
```

Apply, then read it back from inside the Pod:

```bash
kubectl exec -it deployment/demo -- printenv GREETING
```

---

## TASK 5: Environment variables from a ConfigMap

Hard-coding values on the Deployment doesn't scale. Pull them from the ConfigMap
instead. The `nginx-conf-configmap.yaml` you applied has some plain key/value data;
reference one as an env var:

```yaml
          env:
            - name: GREETING
              valueFrom:
                configMapKeyRef:
                  name: demo-config
                  key: greeting
```

Apply and verify with `printenv` again. To import *all* keys at once, look up
`envFrom:` + `configMapRef:` in the docs and try it.

---

## BONUS

1. Mount the **Secret** as a file (not an env var) under `/etc/secrets` and `cat` it
   from inside the Pod.
2. Change a value in a mounted ConfigMap and `kubectl apply`. Mounted ConfigMap files
   update in place after a short delay (no restart needed) — but **env vars do not**.
   Confirm both behaviours. Which would you use for a value that changes often?
