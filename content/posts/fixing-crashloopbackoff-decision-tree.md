---
title: "Fixing CrashLoopBackOff: A Decision Tree"
date: 2026-08-10
description: "A practical, step-by-step decision tree for diagnosing and resolving CrashLoopBackOff errors in Kubernetes, from exit codes to liveness probe misconfigurations."
tags: ["kubernetes", "pods", "crashloopbackoff"]
categories: ["troubleshooting"]
draft: false
ShowToc: true
TocOpen: false
---

`CrashLoopBackOff` is one of the most common — and most misdiagnosed — pod states in Kubernetes. It isn't an error itself; it's the kubelet telling you that a container keeps exiting and it's now waiting longer between restarts. The actual cause is almost always somewhere else. Here's the decision tree I run through every time.

## Step 1: Read the exit code before doing anything else

Start with:

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

Or just look at the full status:

```bash
kubectl describe pod <pod-name>
```

Under `Last State: Terminated`, the exit code tells you which branch of the tree to follow:

| Exit Code | Meaning | Likely Cause |
|---|---|---|
| `0` | Clean exit | App finished and exited — not actually crashing, just not a long-running process |
| `1` | Generic application error | Uncaught exception, bad config, missing env var |
| `137` | SIGKILL (128+9) | OOMKilled, or an external kill (often the OOM killer) |
| `139` | SIGSEGV (128+11) | Segfault — usually a native dependency or corrupted binary |
| `143` | SIGTERM (128+15) | Graceful shutdown didn't finish before `terminationGracePeriodSeconds` |

## Step 2: Check if it's actually OOMKilled

Exit code `137` is frequently OOM, but `describe pod` will confirm it explicitly:

```
State:          Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

If you see `OOMKilled`, don't just bump the memory limit and move on — check whether the pod's actual usage is spiking due to a leak or an unbounded cache versus a genuinely underprovisioned limit:

```bash
kubectl top pod <pod-name> --containers
```

Compare against `resources.limits.memory` in the pod spec. If usage climbs steadily over the pod's lifetime rather than spiking under load, that's a leak, not a sizing problem.

<!--more-->

{{< ad-in-content >}}

## Step 3: Pull the previous container's logs, not the current one's

This is the mistake I see most often: running `kubectl logs <pod-name>` on a pod that has already restarted shows you the logs of the *new*, still-starting container — not the one that crashed. You need:

```bash
kubectl logs <pod-name> --previous
```

If the container never produced output before dying, check for:

- A missing `ConfigMap` or `Secret` referenced by `envFrom` or a mounted volume (the container fails before it can log anything)
- An entrypoint script with a syntax error
- A missing binary or wrong `CMD`/`ENTRYPOINT` for the image's actual filesystem layout (common after switching base images, e.g. Alpine to Debian)

## Step 4: Check readiness/liveness probes

If logs look clean and the app appears to start fine, the kubelet itself may be killing the container via a failing liveness probe. Check events:

```bash
kubectl describe pod <pod-name> | grep -A5 Events
```

Look for `Liveness probe failed` entries. Common causes:

- Probe `initialDelaySeconds` too short for a slow-starting app (JVM warm-up, large dataset load, migrations running on boot)
- Probe hitting the wrong port or path after a deployment change
- Probe timeout too aggressive under load, causing false failures

A quick sanity check: temporarily remove the liveness probe (or bump `failureThreshold`/`initialDelaySeconds` generously) and redeploy. If the crash loop stops, the probe configuration — not the app — was the root cause.

## Step 5: Dependency and network checks

If none of the above apply, the container may be crashing while waiting on a dependency (database, message queue, another service) that isn't reachable yet. Check for:

- `initContainers` timing out
- DNS resolution failures inside the pod (`kubectl exec` into a debug pod and run `nslookup` against the dependency's service name)
- NetworkPolicies blocking egress that didn't exist before a recent policy change

## The decision tree, summarized

1. **Exit code `0`** → not a crash, fix the process lifecycle (add a supervisor, or it's a Job not a Deployment).
2. **Exit code `137` + `OOMKilled`** → check actual memory usage trend, then size limits or fix the leak.
3. **Exit code `139`** → native crash, check for architecture mismatches or corrupted images.
4. **`--previous` logs show a clear error** → fix the app/config issue directly.
5. **No useful logs, but events show liveness probe failures** → fix probe timing/config.
6. **Clean start, dies waiting on something** → check init containers, DNS, and NetworkPolicies.

Working through these in order turns "the pod won't stay up" into a specific, fixable root cause almost every time.
