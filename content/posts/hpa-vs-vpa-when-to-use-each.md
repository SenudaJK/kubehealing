---
title: "HPA vs VPA: When to Use Each"
date: 2026-08-17
description: "A practical comparison of the Horizontal Pod Autoscaler and Vertical Pod Autoscaler in Kubernetes — how each works, when they conflict, and how to choose."
tags: ["kubernetes", "autoscaling", "hpa", "vpa", "resource-management"]
categories: ["tutorials"]
draft: false
ShowToc: true
TocOpen: false
---

Kubernetes gives you two different autoscalers that solve two different problems, and mixing them up leads to either wasted spend or outages under load. Here's how each actually works and how to pick between them.

## How the Horizontal Pod Autoscaler works

The HPA changes the **number of pod replicas** in response to observed metrics (CPU, memory, or custom/external metrics via the metrics API). It doesn't touch the resource requests or limits on the pod spec at all.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

This is the right tool when your workload is **stateless and horizontally scalable** — a typical HTTP API, a worker pool consuming from a queue, anything where adding more identical replicas linearly increases throughput.

<!--more-->

{{< ad-in-content >}}

## How the Vertical Pod Autoscaler works

The VPA changes the **resource requests and limits on individual pods**, based on historical usage. Depending on its `updateMode`, it either just recommends values (`Off`), or actively evicts and recreates pods with new resource values (`Auto`/`Recreate`).

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: worker-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 4Gi
```

This is the right tool when a workload's **per-instance resource needs are hard to predict up front**, or drift over time — a single stateful process, a batch job with variable input sizes, or anything you can't simply run more copies of to get more throughput (e.g., a single-threaded consumer bound to one partition).

## Why running both on CPU is a bad idea

If HPA scales on CPU utilization and VPA is actively changing CPU requests on the same workload, they fight each other: VPA changes the request, which changes what "70% utilization" means, which triggers HPA to scale replicas, which changes aggregate load, which changes VPA's next recommendation. This is explicitly called out as unsupported in the VPA project's own documentation.

The safe pattern if you want both signals:

- HPA scales on a **custom or external metric** (requests per second, queue depth) instead of CPU/memory.
- VPA manages CPU/memory requests in `Auto` mode, or runs in `Off`/recommendation-only mode and you review suggestions manually.

## Decision guide

| Situation | Use |
|---|---|
| Stateless service, traffic varies through the day | HPA on CPU or custom metric |
| Queue-based workers, backlog grows and shrinks | HPA on queue depth (external metric) |
| Single-instance workload with unpredictable memory needs | VPA, `updateMode: Auto` |
| You don't know good resource requests yet | VPA, `updateMode: Off` (recommendations only), then hardcode values |
| Need both horizontal and vertical signals | HPA on a non-resource metric + VPA on resources, never both on CPU/memory |

In most production clusters I've run, the pattern that works best is: use VPA in `Off` mode continuously as a request/limit sizing tool (review its recommendations monthly), and use HPA for real-time scaling driven by request-rate metrics rather than CPU. That combination avoids the conflict entirely while still getting the benefit of both.
