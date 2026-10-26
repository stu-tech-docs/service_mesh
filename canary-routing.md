---
layout: default
title: Pilot & Canary Routing
nav_order: 5
description: "Techniques for session pinning, JWT/Cookie extraction, and downstream header propagation across 40 microservices."
permalink: /canary-routing/
---

# Pilot User Session Pinning in Canary Deployments

> How to ensure internal pilot users stay pinned to the Green environment throughout their authenticated session, why raw client headers fail, and how to reliably propagate context across 40 downstream microservices.

---

## 1. The Flaws of Using `X-Pilot-User: true`

While passing a custom header like `X-Pilot-User: true` is commonly conceived during early design discussions, in production it introduces severe vulnerabilities:

* **Security & Spoofing:** Any user can inspect browser network traffic, add `X-Pilot-User: true` to their headers, and gain unauthorized access to untested internal features or staging databases.
* **Client State Fragility:** Single-page applications (SPAs) or mobile apps must manage header injection. Opening a new tab, loading static assets, or navigating directly via URL can cause the header to be dropped.
* **Downstream Cascade Trap:** Envoy sidecars do not automatically pass arbitrary request headers across outbound calls made by application code. Downstream REST hops lose the header and fall back to Blue.

---

## 2. Pattern 1 (Recommended): JWT Claims + Istio `RequestAuthentication`

The enterprise standard for API and modern web architectures is to cryptographically sign the pilot identity inside the user's JSON Web Token (JWT) at login.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                          JWT-BASED PILOT PINNING WORKFLOW                              │
└────────────────────────────────────────────────────────────────────────────────────────┘

 [Pilot Logs In] ──► [Keycloak / Auth Server]
                             │
                             ▼ (Signs JWT with "is_pilot": true)
 [Client Browser] ──(Authorization: Bearer <JWT>)──► [Istio Ingress Gateway]
                                                              │
                                            (1. Validates JWT Signature)
                                            (2. Extracts claim to 'x-internal-pilot: true')
                                                              │
                                                              ▼
                                            [VirtualService Matches 'x-internal-pilot']
                                                              │
                                                              ▼
                                                    [Service A (GREEN)]
```

### 1. Ingress JWT Extraction Configuration
Deploy a `RequestAuthentication` on the Ingress Gateway to validate the token and extract the claim into a sanitized internal header:

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-pilot-extractor
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "https://auth.company.com/realms/enterprise"
    jwksUri: "https://auth.company.com/realms/enterprise/protocol/openid-connect/certs"
    outputClaimToHeaders:
    - header: "x-internal-pilot"
      claim: "is_pilot"
```

### 2. VirtualService Routing Rule
Define routing logic in `VirtualService` to direct verified pilot headers to the `green` subset:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: account-service-routing
  namespace: production
spec:
  hosts:
    - account-service
  http:
    - match:
        - headers:
            x-internal-pilot:
              exact: "true"
      route:
        - destination:
            host: account-service
            subset: green
    - route:
        - destination:
            host: account-service
            subset: blue
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: account-service-destination
  namespace: production
spec:
  host: account-service
  subsets:
    - name: blue
      labels:
        version: blue
    - name: green
      labels:
        version: green
```

---

## 3. Pattern 2: Signed `HttpOnly` Cookies (Best for Web SPAs)

If your application serves browser-based web applications:

1. **At Login:** The authentication backend sets an encrypted, `HttpOnly`, `SameSite=Lax` cookie upon successful authentication:
   ```http
   Set-Cookie: user_release=pilot_green; Path=/; Secure; HttpOnly; Max-Age=28800
   ```
2. **On Subsequent Requests:** The browser automatically attaches this cookie on **every single outbound HTTP request**, asset download, and background API call.
3. **In `VirtualService`:** Istio inspects the cookie string natively:
   ```yaml
   http:
     - match:
         - headers:
             cookie:
               regex: ".*user_release=pilot_green.*"
       route:
         - destination:
             host: account-service
             subset: green
     - route:
         - destination:
             host: account-service
             subset: blue
   ```

---

## 4. Pattern 3: Dedicated Pilot Ingress / Hostname

If you want zero code changes in frontend and backend auth systems:

* Create a dedicated DNS hostname for pilot testers: `pilot.app.company.com` alongside `app.company.com`.
* Configure Istio Gateway to route all traffic on `pilot.app.company.com` directly to the `green` subsets across services.
* Internal testers simply bookmark and log into the pilot domain.

---

## 5. Solving Downstream Propagation Across 40 Microservices

Once the Ingress Gateway routes the initial request to **Green Microservice A**, how do downstream calls from `A &rarr; B &rarr; C` stay in the Green environment?

{: .warning }
> **The Downstream Cascade Trap:** Envoy proxies inbound requests to your application container on `localhost`. When your code makes an outbound REST call to `http://order-service/api`, your code creates a brand new HTTP request. Unless your code explicitly attaches the pilot context, Envoy will route that call to Blue!

### Solution A: OpenTelemetry W3C Baggage (Recommended)
W3C Baggage (`baggage: pilot=true`) is part of standard distributed tracing. OpenTelemetry auto-instrumentation agents (e.g., OpenTelemetry Java Agent, Node.js SDK) **automatically propagate baggage headers across HTTP client calls without any manual business logic changes**.

### Solution B: Shared HTTP Client Interceptor (Spring Boot Example)
If you maintain a shared base library or internal starter across your 40 microservices, register an outbound HTTP interceptor:

```java
@Component
public class PilotHeaderInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                        ClientHttpRequestExecution execution) throws IOException {
        ServletRequestAttributes attrs = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attrs != null) {
            HttpServletRequest currentRequest = attrs.getRequest();
            String pilotHeader = currentRequest.getHeader("x-internal-pilot");
            if (pilotHeader != null) {
                // Propagate verified pilot status to downstream REST call
                request.getHeaders().add("x-internal-pilot", pilotHeader);
            }
        }
        return execution.execute(request, body);
    }
}
```

---

**Next:** Review the [Adoption Roadmap & Runbook](roadmap.md) for phased execution.
