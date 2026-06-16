# Guide for Kubernetes workshop namespace

**SKIP THIS GUIDE, WE DECIDED TO USE DEFAULT NAMESPACE**

The following is a guide for creating a separate `workshop` namespace, to contain all resources created during this workshop.
We've decided to abandon this, and just let participants use `default` namespace.

This avoids some complexity and confusion when already having to learn many other Kubernetes concepts.

This guide is left here for the curious participant.

## 1. Create your namespace

A **namespace** is a labelled fence around a group of resources. Working in your own
namespace keeps your nginx Pod from colliding with the person next to you, and lets
you delete everything in one shot at the end.

```bash
kubectl create namespace workshop
kubectl get namespaces
```

## 2. Set the namespace on your context (do this — it saves pain later)

By default `kubectl` talks to the `default` namespace, so every command would need
`-n workshop`. Instead, pin your namespace onto the **current context** once:

```bash
kubectl config set-context --current --namespace=workshop
```

Now `kubectl get pods` automatically means `kubectl get pods -n workshop`. Verify:

```bash
kubectl config view --minify | grep namespace:
# namespace: workshop
```

> **Why make a point of this:** later in the workshop commands sometimes show an
> explicit `-n demo` / `-n pods`. Once you understand that `-n` just overrides the
> namespace for one command — and that your context already has a default — those
> flags stop being confusing. `-A` (or `--all-namespaces`) shows resources across
> *every* namespace, which is handy when you "lose" something.

## Teardown (after workshop)

Deleting your namespace removes everything you created inside it (be careful what you delete):

```bash
kubectl delete namespace workshop
```
