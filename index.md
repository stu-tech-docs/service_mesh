---
layout: default
title: Overview & Baseline
nav_order: 1
description: "Architecture baseline, current pain points, and executive summary for service mesh evaluation on OpenShift."
permalink: /
---

# Service Mesh Architecture & Evaluation Guide

> An architectural evaluation for modernizing an enterprise microservices platform on **Red Hat OpenShift (OCP)**: moving from static `ConfigMap` properties and pod restarts to dynamic L7 traffic shifting, zero-trust security, and end-to-end distributed observability.

---

## 1. Application Baseline Snapshot

| Architectural Dimension | Current Production State |
| :--- | :--- |
| **Microservice Fleet** | **~40 Microservices** hosted on Red Hat OpenShift Container Platform (OCP). |
| **Inter-Service Protocol** | Synchronous **REST APIs** over HTTP/1.1 across internal services and external backends. |
| **Configuration & Discovery** | Downstream URLs hardcoded in application properties files mounted via **OCP `ConfigMap`s**. |
| **Caching Layer** | **Two Redis clusters** with bi-directional active sync. App connects to one at a time. |
| **Failover Mechanism** | Switching Redis clusters requires **updating ConfigMaps and restarting all microservice pods**. |
| **Deployment Model** | **Blue/Green**: Services are deployed to **Green** first for internal pilot users, then promoted to **Blue** for customers. |

---

## 2. Current Architecture & Operational Workflow

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                CURRENT ARCHITECTURE FLOW                               │
└────────────────────────────────────────────────────────────────────────────────────────┘

  [Pilot / Customer Traffic]
              │
              ▼
    [OCP Route / Router]
              │
              ├──► [Green Services (Pilot)]  ──► Static URLs in ConfigMaps ──► [Backend REST APIs]
              └──► [Blue Services (Prod)]   ──► Static URLs in ConfigMaps ──► [Redis Cluster A]
                                                                                ▲
                                     (Switching to Cluster B requires:           │
                                      1. Update ConfigMap Properties             │
                                      2. Restart 40 Microservice Pods) ──────────┘
```

---

## 3. Core Operational Pain Points

### 1. Static URL Coupling in ConfigMaps
* Microservices look up target hostnames from static configuration properties files.
* Changing an endpoint, shifting namespaces, or rerouting traffic requires modifying `ConfigMap` manifests and performing rolling restarts of dependent pods.

### 2. High-Risk Blue/Green & Pilot Verification
* Internal pilot users currently test changes in the Green environment before customer promotion.
* Managing this separation at the OCP Route/DNS layer often requires duplicating large sections of the application footprint or executing high-stakes all-or-nothing cutovers.

### 3. Manual Redis Failover Bottleneck
* Although dual Redis clusters have bi-directional synchronization, failover remains a manual, disruptive event.
* To switch from Redis Cluster A to Cluster B, engineers must edit ConfigMaps and bounce 40 services, creating temporary downtime or connection churn.

### 4. Observability & Reliability Gaps
* With ~40 microservices communicating synchronously over REST, tracing request latency and diagnosing cascading HTTP 5xx errors requires aggregating fragmented application logs.
* Resilience policies (retries, timeouts, circuit breakers) are implemented inconsistently across codebases.

---

## 4. Documentation Structure

This documentation is organized into focused, modular chapters:

1. **[Benefits Analysis](benefits.md):** How a service mesh solves ConfigMap URL coupling, streamlines Blue/Green pilot routing, automates zero-trust mTLS, and introduces Kiali observability.
2. **[Risks & Complexities](challenges.md):** Sizing data plane resource overhead (120–160+ Envoy sidecars), network hop latency, Redis TCP connection pool draining, and day-2 OpenShift operations.
3. **[Envoy vs. Consul vs. OSSM](mesh-comparison.md):** Deconstructing data plane vs. control plane concepts and explaining why Red Hat OpenShift Service Mesh (OSSM / Istio + Envoy) is the natural fit for an OCP-only estate.
4. **[Pilot User & Canary Session Pinning](canary-routing.md):** In-depth technical guide on pinning pilot users to Green post-login: why `X-Pilot-User: true` fails, how to use JWT claims / cookies, and solving downstream header propagation across 40 services.
5. **[Adoption Roadmap & Runbook](roadmap.md):** A 4-phase migration strategy, operational verification commands (`oc`, `istioctl`), and pre-flight readiness checklists.
