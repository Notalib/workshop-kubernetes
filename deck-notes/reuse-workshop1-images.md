# Slide 46 — Re-use images from Workshop #1

Goal: let attendees deploy the apps they containerized in Workshop #1 onto Kubernetes,
closing the loop "I built the image → now I run it on a cluster."

## Status / what's needed

The Workshop #1 apps (Java hello-world, .NET, static web app, Python/Jupyter, Go) need
to be **published to a registry** Kubernetes can pull from. Until then, the hands-on
modules use public test images (nginx, traefik/whoami, mendhak/http-https-echo,
busybox) so nothing is blocked.

## Suggested approach: publish to GHCR

The Workshop #1 repo is already on GitHub (`notalib/workshop-containerisation`), so
GitHub Container Registry is the least-friction option:

```bash
# Build and push (example for the static web app)
docker build ./5-static-web-app -t ghcr.io/notalib/workshop-static-web:1.0.0
docker push ghcr.io/notalib/workshop-static-web:1.0.0
```

Better: a small GitHub Actions workflow that builds + pushes each app's image on a tag,
so the images stay reproducible and versioned (ties back to the Workshop #1 best-practice
slide on pinning + metadata labels).

Make the packages **public** so attendees don't need registry credentials during the
workshop. (If kept private, they'd need an `imagePullSecret` — a possible advanced
exercise in its own right.)

## How it slots into the exercises

Once published, attendees swap the image in any module's manifest:

```bash
kubectl create deployment mywebapp --image=ghcr.io/notalib/workshop-static-web:1.0.0
kubectl expose deployment/mywebapp --port=80
kubectl create ingress mywebapp --rule="mywebapp.localhost/=mywebapp:80" --class=traefik
```

This is already wired as the **bonus** in the exercise repo's top-level README — it
just needs the published image name filled in.

## Speaker note

Emphasize the payoff: the image they built in Workshop #1 is the *exact same artifact*
now running, scaling, and self-healing on a cluster — the OCI standard (slide 55) is
what makes that portability work.
