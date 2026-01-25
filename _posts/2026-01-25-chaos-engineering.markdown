---
layout: post
title: "Kubernetes Chaos Engineering"
date: 2026-01-25 01:00:00 +0000
categories: [infra, networking]
tags: [kubernetes, chaos-engineering, infra]
---

# GitOps Chaos Engineering Without the Heavyweight Stack

*Building a deterministic, request-level HTTP fault injector with Istio, Envoy, and Kubernetes admission control.*

## Why I built this (the honest motive)

Chaos engineering is everywhere… but so is chaos tooling sprawl.

In many organizations, “doing chaos” quickly turns into adopting large platforms (Gremlin, Litmus, Chaos Mesh) that come with their own control planes, agents, permissions, UIs, and operational overhead. Those tools are powerful, but if your goal is **simple, safe, GitOps-first experiments**, they can feel like using a rocket launcher to open a can.

So I built a [*proof of concept*](https://github.com/sghaida/chaos-engineering):
A **minimal GitOps chaos system** that lets teams declare request-level HTTP faults as Kubernetes resources, enforced by policy, applied by a controller, and executed by Envoy — **without agents, privileged access, or a separate chaos platform**.

The core question I wanted to answer was:

> Can we do *safe*, *auditable*, *namespace-scoped* chaos with a straightforward approach that fits naturally into GitOps workflows?

This repo is that answer.

## What I needed to test

I wasn’t trying to simulate “all chaos.” I was focused on the failures I see most often in real systems:

### 1) Dependency failures (downstream)

* What happens if my service’s dependency is **slow**?
* What happens if the dependency becomes **unavailable**?
* Do retries/backoff/circuit breakers behave as expected?
* Does the service degrade gracefully, or cascade failures?

### 2) Service behavior under latency and errors (upstream + downstream)

I wanted to inject:

* **fixed latency delays**
* **fixed HTTP status aborts** (timeouts / 5xx)

…for **specific routes**, and for both:

* **INBOUND** (client → service)
* **OUTBOUND** (service/pod → dependency)

This is crucial because route-level failures are the closest to real incident patterns:

* a single endpoint regresses
* a single downstream dependency slows down
* a specific call path starts timing out

## What I *didn’t* want

The most important constraint: **don’t impact other services**.

This PoC is built for **shared clusters**, where uncontrolled chaos becomes a production incident. So I made blast radius and safety **first-class**, not “best effort.”

The design goal was:

> Experiments must be constrained by *hard guardrails* enforced at admission time, not by “people following a runbook”.

That’s why the system uses **Kubernetes ValidatingAdmissionPolicy (CEL)** to reject unsafe specs *before* they can run.

## What this is: Declarative HTTP chaos (Istio / Envoy)

At its core, this project introduces one custom resource:

```text
FaultInjection (CRD)
```

A `FaultInjection` declares:

* **blast radius** (duration, traffic %)
* **actions** (HTTP latency, HTTP abort)
* **targeting** (routes, headers, direction, destination hosts)
* **scope** (namespace)

If it passes admission policy, a controller reconciles it into deterministic Istio VirtualService rules.

### Why request-level faults?

Because they’re:

* deterministic
* reversible
* narrow in blast radius
* safe in shared clusters
* observable via Envoy metrics

This is *not* packet-level chaos (no tc/netem/iptables). It’s **HTTP-level fault injection**.

## Architecture: policy-first control plane, Envoy runtime

### Control plane (safe-by-default)

* Developer commits a `FaultInjection` YAML
* GitOps applies it (ArgoCD or plain `kubectl apply`)
* Admission policy validates constraints (duration, percentage, semantics)
* Controller reconciles → patches VirtualService
* Controller auto-cleans on expiry

### Runtime execution (deterministic)

Once VirtualService is patched, Envoy sidecars enforce:

* inject delay for matching requests
* or abort with a fixed HTTP status (e.g., 504)

Other routes remain untouched.

## What’s implemented as of the day of publish the post

### Supported faults

* `HTTP_LATENCY` — fixed delay
* `HTTP_ABORT` — deterministic abort (e.g., 504)

### Directions

* **INBOUND**: client → service (VirtualService ref)
* **OUTBOUND**: pod → destination (mesh gateway)

### Targeting

* URI prefix or exact
* optional headers
* source pod labels (outbound)
* destination hosts
* percentage based impact

### Guardrails (enforced at admission)

Examples of what’s already enforced:

* required blast radius + duration
* percent bounds and max traffic cap
* required route match (prefix/exact)
* correct semantics per action type and per direction
* safe routing rules only

If a manifest violates the guardrails, Kubernetes rejects it, meaning:

> the experiment never exists → the experiment never runs.

That’s the right failure mode.

## Why this approach is valuable (benefits)

### 1) GitOps-native by design

Your chaos experiments are just YAML:

* reviewable in PRs
* auditable
* versioned
* reproducible

You can put them in a service repo like:

```text
service-repo/
  chaos/
    latency.yaml
    timeout.yaml
```

And treat experiments like any other deployment artifact.

### 2) Deterministic and route-level

This is not “random network weirdness.” It’s:

* precise
* request-level
* bounded by path + percent + duration

You can answer questions like:

* “Does `/api/vendors` degrade gracefully under +2s latency?”
* “What happens to checkout if payment returns 504 for 10% of calls?”

…without turning the entire cluster into a science experiment.

### 3) Safe in shared clusters

This is the big one.

Instead of trusting teams to do the right thing manually, the platform can enforce:

* max duration
* max traffic percentage
* direction rules
* mandatory selectors and route constraints

All enforced before execution.

That reduces:

* accidental wide blast radius
* cross-namespace impact
* long-running experiments
* “oops I pointed it to the wrong VirtualService”

### 4) Operationally lightweight

No agents, no privileged DaemonSets, no kernel mutation.
Just:

* a controller
* a CRD
* admission policies
* Istio/Envoy (already running in many clusters)

## A concrete example: simulate dependency latency + timeout

You can express a scenario like:

* OUTBOUND latency to a dependency for requests matching `/anything/vendors/`
* OUTBOUND abort when header `x-chaos-mode: timeout` is present
* INBOUND versions of the same (to validate how the service behaves for clients)

And then run the included test script from the `curl-client` pod to validate:

* control path is fast
* abort path returns 504
* delay path takes ~2s
* non-matching paths remain fast

This matters because it proves:

* route isolation works
* guardrails allow controlled chaos
* the mesh does deterministic injection
* cleanup restores baseline state

## looking into the archeticture

### C4 Level 1- System context

[System context](/assets/img/chaos-engineering/c4-1-context.svg)

### C4 Level 2 — Containers

[Containers](/assets/img/chaos-engineering/c4-2-containers.svg)

### Sequence — GitOps apply → policy gate → run → TTL cleanup

[GitOps flow](/assets/img/chaos-engineering/gitops-flow.svg)

### Sequence — Inbound vs Outbound

[Inbound vs Outbound](/assets/img/chaos-engineering/inbound-vs-outbound.svg)

### “Blast radius story” diagram — why other routes don’t break

[Blast radius](/assets/img/chaos-engineering/blast-radius.svg)

## How to take it forward (roadmap)

What exists today covers the “safe HTTP chaos MVP” well. The next step is turning this into a **repeatable resilience workflow**, not just a fault injector.

Here’s the forward plan, aligned with what you already built, and using RFC ideas only for what’s not implemented yet:

### 1) Add k6-loadgen as a first-class workflow

Right now, you can run curl-based tests. The next step is:

* let experiments include a **load generation plan**
* run it during the chaos window
* capture results consistently

**Why it matters:**
Chaos without controlled traffic is hard to interpret. k6 makes experiments measurable and repeatable.

A practical direction:

* a `k6-loadgen` Job triggered alongside the FaultInjection
* scoped to the same namespace
* runs for the same TTL window
* stores a summary (p95/p99, error rate) back into status

### 2) Manual abort as a first-class control

This came up in your earlier discussions too: GitOps deletion is not always fast enough.

Add a spec control field:

```yaml
spec:
  control:
    abort: true
    reason: "Unexpected impact"
    requestedBy: "oncall"
```

Controller behavior:

* detect abort flag
* immediately clean up injected rules
* mark status as `Aborted`
* idempotent cleanup

This gives on-call a “stop button” without touching deployments.

### 3) Stop conditions and auto-abort (optional but powerful)

This is where chaos becomes production-grade:

* define stop conditions (latency, 5xx rate, burn rate)
* evaluate periodically
* abort immediately if breached

You don’t need to implement the full platform stack at once. Even a small set (p99 + 5xx) goes a long way.

### 4) Expand fault types carefully (only if needed)

Your current scope is intentionally narrow and safe. If you expand, do it with the same philosophy:

* bounded pod delete (maxPodsAffected)
* scaling faults with snapshot+restore
* AWS dependency faults as **request-only** objects (no AWS IAM in the controller)

But the key: don’t lose the “shared cluster safe” property.

### how the future will look like

#### adding k6-loadgen (roadmap)

[k6-loadgen](/assets/img/chaos-engineering/k6-loadgen.svg)

## Closing thought: chaos as a capability, not a product

This PoC isn’t trying to compete with Gremlin/Litmus/Chaos Mesh on feature breadth.

It’s taking a different stance:

* **small, deterministic primitives**
* **GitOps first**
* **policy enforced**
* **namespace scoped**
* **safe-by-default**

If you can do 80% of your resilience testing with:

* latency
* aborts
* route targeting
* strict guardrails

…then you can shift chaos from a special event into a normal engineering practice.

**And that’s the real win.**
