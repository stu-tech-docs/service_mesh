---
layout: default
title: Adoption Roadmap
nav_order: 6
description: "Phased rollout plan, verification commands, and pre-flight checklist for service mesh on OpenShift."
permalink: /roadmap/
---

# Phased Implementation Roadmap & Runbook

> A production-proven, 4-phase rollout strategy designed to migrate 40 microservices onto Red Hat OpenShift Service Mesh with zero business disruption.

---

## 1. Roadmap Overview

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                PHASED ADOPTION ROADMAP                                 │
└────────────────────────────────────────────────────────────────────────────────────────┘

 [ PHASE 1: FOUNDATION ] ──► [ PHASE 2: PILOT CANARY ] ──► [ PHASE 3: FLEET SCALE ] ──► [ PHASE 4: ZERO-TRUST ]
   • Install OSSM Operator      • 3-5 Pilot Services       • Onboard all 40 Apps      • Strict mTLS (Enforced)
   • Permissive mTLS            • JWT/Cookie Pinning       • Retire ConfigMap URLs     • AuthorizationPolicies
   • Kiali & Tracing Baseline   • Verify Header Flow       • % Canary Pipelines       • Redis Egress Strategy
```

---

## 2. Phase-by-Phase Execution

### Phase 1: Foundation, Observability & Baseline (Weeks 1 &ndash; 3)

{: .tip }
> **Phase 1 Objective: Zero Traffic Disruption.** Install the control plane and enable sidecar injection in **`PERMISSIVE`** mTLS mode. No routing changes or ConfigMap changes are made yet. This provides instant visibility into 40 services through Kiali without touching application traffic.

1. **Install Operators via OpenShift OperatorHub:**
   * Red Hat OpenShift Service Mesh Operator (Istio)
   * Kiali Operator (Console visualization)
   * Red Hat OpenShift distributed tracing platform (Jaeger / Tempo)
2. **Deploy ServiceMeshControlPlane (SMCP):** Configure control plane with `security.dataPlane.automTLS: true` and `security.controlPlane.mtls: true`.
3. **Join Application Namespaces:** Add application project namespaces to the `ServiceMeshMemberRoll` (SMMR).
4. **Baseline Telemetry:** Observe service communication graphs in Kiali; verify memory and CPU usage on worker nodes.

---

### Phase 2: Pilot Canary on 3&ndash;5 Microservices (Weeks 4 &ndash; 6)

Select a representative vertical slice: 1 ingress frontend service and 2 downstream REST dependencies.

1. **Implement Pilot Context Injection:** Configure the IdP to output `is_pilot: true` in JWTs, or set an `HttpOnly` cookie at login.
2. **Deploy Ingress Gateway & VirtualService:** Match pilot headers at the gateway and route to the `green` subset.
3. **Verify Downstream Propagation:** Validate that when Service A calls Service B and C, the pilot context is preserved (via OpenTelemetry Baggage or shared HTTP interceptor).
4. **Simulate Rollbacks:** Test instant traffic reversal from Green to Blue in seconds without restarting pods.

---

### Phase 3: Fleet Rollout Across 40 Services (Weeks 7 &ndash; 10)

Expand service mesh across all remaining microservices in waves of 8&ndash;10 services per sprint:

* **Retiring ConfigMap URLs:** Update application properties files to reference stable virtual service DNS names (e.g., `http://order-service/api`) rather than brittle, environment-specific URLs.
* **Automated Canary Pipelines:** Integrate OpenShift GitOps (ArgoCD) or CI/CD pipelines with Istio `VirtualService` to automate progressive traffic shifting (5% &rarr; 25% &rarr; 50% &rarr; 100%) during customer rollouts.

---

### Phase 4: Strict Zero-Trust & Redis Strategy (Weeks 11 &ndash; 12)

* **Enforce STRICT mTLS:** Transition from `PERMISSIVE` to `STRICT` mTLS mode using `PeerAuthentication` to reject unencrypted traffic.
* **Lock Down Access with AuthorizationPolicies:** Declare explicit allowed paths (e.g., only `billing-service` can call `payment-service/charge`).
* **Redis Strategy Execution:**
  * Either manage Redis clusters through a dedicated Redis proxy (e.g., HAProxy / Sentinel).
  * Or configure an Istio `ServiceEntry` with explicit TCP drain timeouts on connection pools.

---

## 3. Operational Verification Runbook

Essential OpenShift CLI (`oc`) and `istioctl` diagnostic commands for Day-2 operations:

```bash
# 1. Check Service Mesh Control Plane Health
oc get smcp -n istio-system

# 2. Check Member Roll (Namespaces joined to Mesh)
oc get smmr -n istio-system -o yaml

# 3. Verify sidecar injection status for a specific pod
oc get pod <pod-name> -n production -o jsonpath='{.spec.containers[*].name}'
# Expected output includes: istio-proxy

# 4. Check synchronization status of Envoy proxies
istioctl proxy-status

# 5. Inspect active routes and clusters inside a service's Envoy sidecar
istioctl proxy-config routes <pod-name>.production
istioctl proxy-config endpoints <pod-name>.production

# 6. Check mTLS status across services in a namespace
istioctl authn tls-check <pod-name>.production
```

---

## 4. Pre-Flight Production Readiness Checklist

| Operational Area | Readiness Item | Action Required |
| :--- | :--- | :--- |
| **Capacity** | Cluster worker nodes sized to accommodate +30GB RAM overhead for 120+ Envoy sidecars. | Verify node memory headroom |
| **Security** | Red Hat OpenShift Service Mesh CNI enabled to manage `iptables` without privileged SCCs. | Verify CNI operator status |
| **Application** | HTTP client libraries audited across 40 services to propagate pilot/trace headers. | Deploy interceptor / OTel |
| **Caching** | Redis client connection pool lifetimes configured to prevent indefinite TCP socket pinning. | Set socket max-idle / max-life |
| **Observability** | Kiali dashboard and distributed tracing validated in non-production cluster. | Test visual error flows |
