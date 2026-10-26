---
layout: default
title: Risks & Complexities
nav_order: 3
description: "Data plane overhead, latency, Redis connection pool gotchas, and operational trade-offs of service mesh on OpenShift."
permalink: /challenges/
---

# Risks, Complexities & Drawbacks

> Implementing a service mesh introduces substantial network and operational trade-offs. Before adopting OpenShift Service Mesh across 40 microservices, your engineering team must evaluate these 5 core challenges.

---

## 1. Data Plane Resource Overhead (Envoy Sidecars)

A service mesh injects an Envoy sidecar proxy into **every single pod**. For a deployment of 40 microservices with modest high-availability replica counts, the resource footprint adds up quickly.

### Resource Sizing for Your Fleet:

| Component | Count | RAM / Pod | CPU / Pod | Total Cluster Overhead |
| :--- | :--- | :--- | :--- | :--- |
| **Envoy Sidecars (Baseline)** | 120 pods (40 services &times; 3 replicas) | 128 MB &ndash; 256 MB | 0.05 &ndash; 0.1 cores | **~15 &ndash; 30 GB RAM, 6 &ndash; 12 Cores** |
| **Istio Control Plane (`istiod`)** | 2 &ndash; 3 replicas | 2 GB &ndash; 4 GB | 1.0 &ndash; 2.0 cores | **~6 &ndash; 12 GB RAM, 3 &ndash; 6 Cores** |
| **Telemetry (Jaeger/Kiali/Prometheus)** | Multi-pod stack | 4 GB &ndash; 8 GB | 1.0 &ndash; 2.0 cores | **~4 &ndash; 8 GB RAM, 2 &ndash; 4 Cores** |
| **Total Mesh Footprint** | &mdash; | &mdash; | &mdash; | **~25 &ndash; 50 GB RAM, 11 &ndash; 22 Cores** |

{: .warning }
> **Capacity Planning Impact:** Ensure your OpenShift worker nodes have sufficient headroom. Adding ~30-50GB of RAM across your cluster may require provisioning 1 or 2 additional worker nodes depending on existing node sizes.

---

## 2. Inter-Service Latency Penalty (2 Proxy Hops)

In standard Kubernetes/OpenShift networking, Service A opens a TCP connection directly to Service B. With a sidecar service mesh, every request traverses two proxies:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              LATENCY HOP BREAKDOWN                                     │
└────────────────────────────────────────────────────────────────────────────────────────┘

 [App A Container] ──► [Envoy Sidecar A] ────(mTLS over SDN)────► [Envoy Sidecar B] ──► [App B Container]
   (localhost)          (Outbound Proxy)                           (Inbound Proxy)        (localhost)
     ~0.1ms                  +1.0ms                                    +1.0ms                ~0.1ms
```

* **Additive Latency:** Each REST call incurs approximately **1.5ms to 3ms** of overhead under moderate traffic.
* **Cascading Call Chains:** If a customer action triggers a sequential chain of 5 microservice calls (Service A &rarr; B &rarr; C &rarr; D &rarr; E), the cumulative latency overhead reaches **10ms to 15ms** purely for network proxying.
* **Impact on p99:** Under peak load, garbage collection pauses or CPU throttling on sidecars can cause tail latency (p99) spikes.

---

## 3. The Redis TCP & Connection Pool Gotcha

A critical consideration for your dual-Redis architecture: **HTTP is stateless; Redis is stateful and uses persistent TCP connection pools.**

{: .caution }
> **Why Mesh Cannot Instantly Reroute Open Redis Connection Pools:** Application connection pools (e.g., Jedis, Lettuce, Redisson, or ioredis) open long-lived TCP sockets to Redis at application boot and keep them open indefinitely. If you reconfigure the mesh `ServiceEntry` from Cluster A to Cluster B, **existing TCP sockets remain established with Cluster A** until they are closed, idle out, or encounter a network error.

### Recommended Architectural Approach for Redis:
1. **Do not rely solely on simple mesh TCP routing** to perform instant active/passive cluster cutovers.
2. Use a dedicated Redis-aware proxy (like HAProxy, Twemproxy, or Envoy Redis Proxy filter with forced connection draining).
3. Alternatively, configure the client library with aggressive idle connection lifetimes (e.g., 60 seconds) so old sockets close and reconnect to the newly routed endpoint.

---

## 4. Header Propagation across 40 Microservices

The mesh cannot inspect business context across calls unless your application code passes context headers along downstream REST calls.

```
User (Pilot) ──► Ingress ──► Service A (Green) ──(Calls REST B without header)──► Service B (Blue!)
                                                                                ▲
                                                     B is tricked into falling back to Blue!
```

* If any service in the 40-microservice chain drops the pilot header (or W3C trace context), the request drops off the Green track back into Blue.
* **This requires auditing and updating HTTP client libraries across all 40 code repositories** (see [Pilot & Canary Routing](canary-routing.md) for how to solve this).

---

## 5. Operational Overhead & Day-2 Maintenance on OCP

* **Operator Lifecycle & Upgrades:** Red Hat OpenShift Service Mesh requires maintaining 3 to 4 coordinated operators (OpenShift Service Mesh / Istio, Kiali, and OpenShift Distributed Tracing). Upgrades require synchronizing control plane versions and triggering rolling restarts of all 40 microservices to update sidecars.
* **Debugging Complexity:** When an API call returns a 503 or connection error, diagnosing the root cause becomes significantly more complex. You must verify: Is it the app? Envoy route matching? mTLS handshake rejection? `iptables` packet redirection? Or OpenShift NetworkPolicy?

---

## Actionable Mitigations Table

| Risk / Complexity | Severity | Actionable Engineering Mitigation |
| :--- | :--- | :--- |
| **Resource Footprint (120+ sidecars)** | **Medium** | Define strict resource requests/limits on sidecars. Use Istio `Sidecar` CRDs to limit configuration scoping to only relevant namespaces. |
| **REST Network Latency** | **Medium** | Optimize Envoy concurrency, enable connection reuse (HTTP/2 or persistent keep-alive), and benchmark critical path APIs before mesh rollout. |
| **Redis Connection Pool Latching** | **High** | Keep Redis outside the HTTP mesh initially, or configure Redis client pools with connection max-lifetimes to allow graceful migration. |
| **Context Propagation Loss** | **High** | Deploy a shared corporate HTTP interceptor library or use OpenTelemetry auto-instrumentation agents across all 40 services. |
| **Day-2 OCP Ops Complexity** | **Medium** | Adopt a phased rollout; train SRE team on `istioctl` diagnostics and Kiali troubleshooting workflows prior to production cutover. |

---

**Next:** Compare [Envoy vs. Consul vs. OSSM](mesh-comparison.md) to choose the right control plane.
