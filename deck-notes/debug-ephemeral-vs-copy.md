# `kubectl debug`: ephemeral container vs. copy

This shows two `kubectl debug` examples. They solve **different problems**.

## Example 1 — Ephemeral container (debug a *running* Pod, in place)

```bash
kubectl run ephemeral-demo --image=registry.k8s.io/pause:3.1 --restart=Never
kubectl debug -it ephemeral-demo --image=busybox:1.28 --target=ephemeral-demo
```

- Injects a **new container into the already-running Pod**.
- Shares the target container's process & network namespace (`--target`), so you can
  inspect its processes, sockets, and `/proc` live.
- The original Pod keeps running, untouched. The debug container vanishes when you're
  done.
- **Use when:** the Pod is up and you want to look inside without disturbing it —
  especially useful for **distroless** images that have no shell of their own.

## Example 2 — Copy (debug a Pod that *won't stay up* / needs changes)

```bash
kubectl run --image=busybox:1.28 myapp -- false      # crashes immediately (runs `false`)
kubectl debug myapp -it --copy-to=myapp-debug --container=myapp -- sh
```

- Makes a **copy of the Pod** (`myapp-debug`) and lets you change it — here, overriding
  the command to `sh` instead of the crashing `false`.
- The original Pod is left as-is; you experiment on the copy.
- **Use when:** the Pod crashes on startup (so there's nothing live to inject into), or
  you need to alter its command/image/config to investigate.

## The one-liner difference

| | Ephemeral (`--target`) | Copy (`--copy-to`) |
|---|---|---|
| Touches | the live Pod | a new copy |
| Pod must be running? | yes | no — ideal for crash-loops |
| Can change command/image? | no | yes |
| Best for | inspecting a healthy/distroless Pod | debugging startup failures |

Docs: <https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/>
