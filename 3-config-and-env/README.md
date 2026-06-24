# 3 — Configuration & Environment

In module 1, custom HTML you wrote inside a Pod vanished on the next rollout. The fix:
keep config and content **outside** the image and inject it at runtime. That's what
ConfigMaps, Secrets, and environment variables are for.

> **ConfigMap** — non-secret key/value config (files or strings).
> **Secret** — same idea, base64-encoded, for credentials (base64 is *encoding*, not
> *encryption* — see the note in TASK 3).
> **Volume mount** — a file or directory injected into a container at runtime. A
> ConfigMap/Secret can be *mounted as files* or *read as env vars*.

---

## TASK 0: Have a working Deployment + Service + Ingress

**If have a working service + ingress from previous exercise, skip this step**:

```bash
# Apply a premade deployment + service + ingress (skip if you completed exercise #2)
kubectl apply -f app.yaml
```

## TASK 1: Serve custom HTML from a ConfigMap (the right way)

Now create a ConfigMap holding an `index.html`:

```bash
kubectl apply -f configmap-html.yaml
kubectl describe configmap/html
```

Now let's mount these into the `web-app`.

Open [`app.yaml`](./app.yaml) and fill in the `TODO`s under `volumes` and `volumeMounts`, then apply it:

```bash
kubectl apply -f app.yaml
# Open http://localhost in your browser
# or from terminal: curl -si localhost

# ALTERNATIVE: if Ingress is not working for you, use port-forward
kubectl port-forward deployment/web-app 8080:80
# Open http://localhost:8080 in your browser
# or from terminal: curl -si localhost:8080
```

You should see your custom page. **Now the real test:**

```bash
kubectl rollout restart deployment/web-app
```

**WARNING:** You may have to type Ctrl/Cmd + Shift + R to get fresh un-cached results.

The content **survives the rollout** — because it lives in the ConfigMap, not the
Pod's ephemeral filesystem. Edit `configmap-html.yaml`, re-`apply`, and restart to
publish a change.

---

## TASK 2: Mount a single file with `subPath`

Mounting a ConfigMap at a directory **replaces the whole directory**.

Often you want to drop in just *one* file (e.g. nginx's `default.conf`) and leave the rest alone. That's what `subPath` does.

```bash
kubectl apply -f configmap-nginx-config.yaml
```

In [`app.yaml`](./app.yaml) there's a second, commented-out
`volumeMount` for the nginx config that uses `subPath`. Un-comment it, re-`apply`, and confirm
that nginx now returns the custom response:

```bash
curl -si http://localhost/anything
# Should give the "Dead-end" 404 result, from the config.
```

**WARNING:** You may have to type Ctrl/Cmd + Shift + R to get fresh un-cached results.

**Experiment:** remove the `subPath:` line, re-apply, and restart. nginx breaks — why?
(Hint: without `subPath`, the mount replaces all of `/etc/nginx/conf.d/`, including
files nginx expects to be there. `subPath` mounts only the single file.)

---

## TASK 3: Environment variables, set directly

The simplest config: key/value `env` entries on the container. Add this to the nginx-container in [`app.yaml`](./app.yaml):

```yaml
          env:
            - name: GREETING
              value: "hello from env"
```

Apply, then read it back from inside the Pod:

```bash
kubectl exec -it deployment/web-app -- printenv GREETING
```

---

## TASK 4: Environment variables from a ConfigMap

Hard-coding values on the Deployment doesn't scale. Pull them from the ConfigMap
instead. The `configmap-nginx-config.yaml` you applied has some plain key/value data;
reference one as an env var:

```yaml
          env:
            - name: GREETING
              valueFrom:
                configMapKeyRef:
                  name: nginx-config
                  key: greeting
```

Apply and verify with `printenv` again. To import *all* keys at once, look up
`envFrom:` + `configMapRef:` in the docs and try it.

---

## Stuck?

- Field help: `kubectl explain deployment.spec.template.spec.volumes` and
  `kubectl explain deployment.spec.template.spec.containers.volumeMounts`.
- See what actually got mounted: `kubectl exec deployment/web-app -- ls /etc/nginx/conf.d`
  and `kubectl exec deployment/web-app -- cat /usr/share/nginx/html/index.html`.
- Check your keys match: `kubectl get configmap/nginx-config -o yaml`.
- nginx won't start after removing `subPath`? `kubectl describe pod -l app=web-pod` and
  read the Events — that's the lesson.
- Docs: <https://kubernetes.io/docs/concepts/configuration/configmap/> and
  <https://kubernetes.io/docs/concepts/configuration/secret/>

## BONUS

1. Add a secret

```bash
kubectl create secret generic web-secret --from-literal=API_KEY=shhhh-its-a-secret
kubectl get secret/web-secret -o yaml
```

Look at the value — it's base64. Decode it:

```bash
kubectl get secret/web-secret -o jsonpath='{.data.API_KEY}' | base64 --decode; echo
```

> ⚠️ **base64 is encoding, not encryption.** Anyone who can read the Secret can read
> the value. Secrets exist to keep credentials out of ConfigMaps/images and to gate
> access via RBAC — not to hide them from someone with read access.

2. Mount the **Secret** as a file (not an env var) under `/etc/secrets` and `cat` it
   from inside the Pod.
3. Change a value in a mounted ConfigMap and `kubectl apply`. Mounted ConfigMap files
   update in place after a short delay (no restart needed) — but **env vars do not**.
   Confirm both behaviors. Which would you use for a value that changes often?
