---
name: k8s-debug
description: Systematically debug Kubernetes workloads — pods stuck Pending, CrashLoopBackOff, OOMKilled, ImagePullBackOff, failing readiness probes, Services with no endpoints, DNS and network problems. Use when the user mentions kubectl, pods, deployments, a cluster, or a workload that won't start or serve traffic.
---

# Kubernetes debugging

Work from the outside in: **workload → pod → container → node → network**.
At each layer, read the status and events before forming a theory.

## Ground rules

- Use read-only commands (`get`, `describe`, `logs`, `top`, `events`,
  `auth can-i`) freely. Before anything that changes the cluster (`delete`,
  `rollout restart`, `scale`, `edit`, `apply`, `cordon`/`drain`), show the
  command and its effect, and get the user's go-ahead.
- Confirm the context and namespace first. Debugging the wrong cluster is a
  classic way to lose an hour or to break something else:
  ```bash
  kubectl config current-context
  kubectl get ns
  ```
- Always pass `-n <ns>` explicitly instead of relying on the default namespace.

## 1. Workload overview

```bash
kubectl get deploy,sts,ds,rs,pods -n <ns> -o wide
kubectl describe deploy/<name> -n <ns>      # conditions, replica counts, events
kubectl rollout status deploy/<name> -n <ns> --timeout=10s
kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -30
```

If `READY` is lower than desired, take one failing pod and go to step 2.
If the rollout is stuck, compare the new and old ReplicaSets: the new one is
usually failing for one of the reasons below.

## 2. Pod status → cause

Run `kubectl describe pod/<pod> -n <ns>` and read **Events** at the bottom,
plus `State` / `Last State` / `Reason` / `Exit Code` per container.

| Symptom | Likely causes | Check |
|---|---|---|
| `Pending` | Insufficient CPU/memory, unsatisfiable nodeSelector/affinity/taints, unbound PVC, ResourceQuota | Events (`FailedScheduling` text says which), `kubectl describe nodes \| grep -A5 Allocated`, `kubectl get pvc -n <ns>`, `kubectl describe quota -n <ns>` |
| `ImagePullBackOff` / `ErrImagePull` | Typo in image/tag, tag doesn't exist, private registry without `imagePullSecrets`, registry rate limit | Event message; `kubectl get pod <pod> -o jsonpath='{.spec.containers[*].image}'`; check the secret exists in *this* namespace |
| `CrashLoopBackOff` | App exits on start: bad config/env, missing secret, can't reach dependency, wrong command, failing migration | `kubectl logs <pod> -c <c> --previous` (the crashed instance, not the new one), exit code below |
| `OOMKilled` (exit 137) | Memory limit too low, or a leak | `kubectl top pod`, limit vs actual; JVM/Node heap flags vs container limit |
| `CreateContainerConfigError` | Referenced ConfigMap/Secret or key missing | Event names the missing object |
| `Running` but not `Ready` | Readiness probe failing: wrong path/port, app slow to start, dependency down | Events (`Unhealthy`), probe spec, curl the probe from inside the pod |
| `Terminating` forever | Finalizers, stuck volume detach, node gone | `kubectl get pod -o jsonpath='{.metadata.finalizers}'`, node status |
| Restarting with no OOM | Liveness probe killing a slow or busy app | Events show `Liveness probe failed`; consider a `startupProbe` or a looser threshold |

Exit codes: `0` the process finished (wrong command for a long-running
container?), `1` app error (read the logs), `126/127` command not
executable / not found, `137` SIGKILL (OOM or liveness), `139` segfault,
`143` SIGTERM (normal on shutdown, suspicious otherwise).

## 3. Inside the container

```bash
kubectl logs <pod> -n <ns> -c <container> --since=15m
kubectl logs <pod> -n <ns> -c <container> --previous
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.containers[*].env[*].name}'   # env var names only, no values
kubectl exec -n <ns> <pod> -c <container> -- sh -c 'wget -qO- localhost:<port>/<probe-path>'
```

For distroless images without a shell, use an ephemeral debug container
(this modifies the pod spec, so ask first):

```bash
kubectl debug -n <ns> <pod> -it --image=busybox:1.36 --target=<container>
```

**Don't print secrets.** If you need to verify that a secret is present, check
the key names and sizes (`kubectl describe secret <s> -n <ns>`), not the values.

## 4. Node layer

When several pods on the same node fail, or the cause is eviction:

```bash
kubectl get nodes -o wide
kubectl describe node <node>     # Conditions: MemoryPressure, DiskPressure, PIDPressure, Ready
kubectl top nodes
kubectl get pods -A --field-selector spec.nodeName=<node>
```

`Evicted` pods mean node pressure, so check the node's conditions. A
`NotReady` node usually means kubelet or runtime trouble, or the node is gone.
That is a cloud/infra question.

## 5. Service and network

A pod is Ready but traffic doesn't reach it:

```bash
kubectl get svc <svc> -n <ns> -o wide
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc>
```

- **No endpoints** → the Service `selector` doesn't match the pod labels, or
  no pods are Ready. Compare `kubectl get svc <svc> -o jsonpath='{.spec.selector}'`
  with `kubectl get pods --show-labels`.
- **`targetPort` mismatch** → the Service points at a port the container isn't
  listening on.
- **DNS** → from a debug pod: `nslookup <svc>.<ns>.svc.cluster.local`; check
  that CoreDNS pods are healthy in `kube-system`.
- **NetworkPolicy** → `kubectl get networkpolicy -n <ns>`. A default-deny
  policy plus a missing allow rule shows up as silent timeouts.
- **Ingress** → `kubectl describe ingress <ing> -n <ns>`, check the backend
  service/port names, then the ingress controller's logs.

## 6. Report

Finish with:
- **Cause** — the specific failing layer and evidence (quote the event/log line)
- **Fix** — the exact manifest change or command, as a diff where possible
- **Verify** — the command that will show it worked
- **Prevent** — e.g. resource requests/limits, probe tuning, a CI check that
  validates image tags or Secret references before deploy
