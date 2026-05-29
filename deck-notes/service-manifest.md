# Service manifest

Mirrors the style of the Pod manifest (slide 25) and Deployment manifest (slide 27):
show the YAML on one side, "this manifest says…" bullets on the other.

## Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx          # picks the Pods to send traffic to
  ports:
    - port: 80          # port the Service exposes
      targetPort: 80    # port on the Pod
```

## This manifest says

- There should exist a **Service** named `nginx`.
- It selects every Pod labelled `app=nginx` (the same label the Deployment stamps on
  its Pods) and load-balances across them.
- It listens on **port 80** and forwards to **targetPort 80** on each Pod.
- Kubernetes gives it a stable **ClusterIP** and a DNS name (`nginx`, or
  `nginx.<namespace>.svc.cluster.local`) — these don't change even as Pods come and go.

## The key idea (callback to slide 30)

The **selector** is the binding: Service (long-lived) ⟶ Pods (short-lived). The
Service is a query over labels, continuously re-evaluated, so its set of backing Pods
(its *endpoints*) tracks reality automatically. No Pod IPs anywhere.

> Optional contrast bullet: a Pod/Deployment manifest describes *workloads to run*; a
> Service manifest describes *how to reach them*. Different controllers own each.
