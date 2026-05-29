# Namespaces visualization

A namespace is a **labelled fence** around a group of resources in one cluster. Same
physical cluster, logically separated tenants.

## Suggested visual (ASCII sketch to redraw as a diagram)

```
┌───────────────────────────── Kubernetes Cluster ─────────────────────────────┐
│                                                                               │
│   ┌── namespace: team-a ──────────┐   ┌── namespace: team-b ──────────┐       │
│   │  Deployment  web              │   │  Deployment  api              │       │
│   │  Service     web              │   │  Service     api              │       │
│   │  ConfigMap   web-config       │   │  Secret      api-creds        │       │
│   │  ── resource quota: 4 CPU ──  │   │  ── RBAC: only team-b can edit│       │
│   └───────────────────────────────┘   └───────────────────────────────┘       │
│                                                                               │
│   ┌── namespace: kube-system (cluster components) ───────────────────────┐    │
│   │  traefik   coredns   metrics-server   ...                            │    │
│   └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│   Nodes / storage / networking are SHARED across all namespaces.              │
└───────────────────────────────────────────────────────────────────────────────┘
```

## Talking points

- **What it is:** an organizational + policy boundary, not a security sandbox by
  itself. Two namespaces can have a `web` Deployment each with no collision.
- **What it can carry (optional):**
  - Resource limits — cap CPU/memory for the whole group (ResourceQuota).
  - Role-based access — who can do what, scoped per namespace (RBAC).
  - Network separation — restrict cross-namespace traffic (NetworkPolicy).
- **What it does NOT isolate:** nodes, the physical network, and the cluster's storage
  are shared. Namespaces partition the *API objects*, not the hardware.
- **DNS tie-in:** a Service's full name includes its namespace —
  `web.team-a.svc.cluster.local`. Inside the same namespace you can just say `web`.
