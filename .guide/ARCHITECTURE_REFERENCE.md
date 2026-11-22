# Architectural Reference: Canary Deployments with Istio & K3s

## System Overview
In a traditional Kubernetes rolling update, Kubernetes replaces old pods with new pods incrementally. However, during a rolling update, you have **zero control** over what percentage of external user requests hit the new version vs. the old version, and if the new version has subtle bugs, 100% of traffic will eventually hit it.

**Canary Deployment with Istio** decouples deployment from release:
- Both `v1.0.0` (Stable) and `v2.0.0` (Canary) run simultaneously in the same cluster.
- A standard Kubernetes `Service` acts as a common DNS name and logical endpoint.
- Istio's **Envoy Proxy sidecars** intercept all incoming packets.
- The **Istio Ingress Gateway** receives external traffic and evaluates the **VirtualService** routing rules.
- The **DestinationRule** maps abstract subsets (`v1`, `v2`) to specific pod labels (`version: v1.0.0`, `version: v2.0.0`).
- Traffic is divided strictly by assigned weights (e.g. 80% to `v1`, 20% to `v2`).

---

## Architecture Diagram (Traffic Flow)

```
                            [ External Client / Browser / curl ]
                                            │
                                            ▼
                        ┌───────────────────────────────────────┐
                        │     Istio Ingress Gateway (Port 80)    │
                        │           (LoadBalancer / NodePort)   │
                        └───────────────────┬───────────────────┘
                                            │
                                            ▼
                             ┌─────────────────────────────┐
                             │       VirtualService        │
                             │  (Weight: 80% v1 / 20% v2)  │
                             └──────────────┬──────────────┘
                                            │
                                            ▼
                             ┌─────────────────────────────┐
                             │       DestinationRule       │
                             │   - subset: v1 (v1.0.0)     │
                             │   - subset: v2 (v2.0.0)     │
                             └──────┬───────────────┬──────┘
                                    │               │
                  80% of Traffic    │               │    20% of Traffic
             ┌──────────────────────┘               └──────────────────────┐
             ▼                                                             ▼
┌───────────────────────────┐                               ┌───────────────────────────┐
│     Subset v1 Pods        │                               │     Subset v2 Pods        │
│   (version: v1.0.0)       │                               │   (version: v2.0.0)       │
│                           │                               │                           │
│ ┌───────────────────────┐ │                               │ ┌───────────────────────┐ │
│ │  Envoy Proxy Sidecar  │ │                               │ │  Envoy Proxy Sidecar  │ │
│ └───────────┬───────────┘ │                               │ └───────────┬───────────┘ │
│             ▼             │                               │             ▼             │
│ ┌───────────────────────┐ │                               │ ┌───────────────────────┐ │
│ │ Next.js App Container │ │                               │ │ Next.js App Container │ │
│ │     (Port 3000)       │ │                               │ │     (Port 3000)       │ │
│ └───────────────────────┘ │                               │ └───────────────────────┘ │
└───────────────────────────┘                               └───────────────────────────┘
```

---

## Core Istio Resources Explained

| Resource | Purpose | DevOps Analogy |
| :--- | :--- | :--- |
| **Gateway** | Configures L4-L6 load balancing at the mesh edge (ports, hosts, TLS). | The front reception door of a building. |
| **VirtualService** | Defines traffic routing rules, path matching, header matching, and weight splits. | The traffic controller directing which department visitors go to. |
| **DestinationRule** | Configures policies that apply to traffic after routing has taken place (subsets/pools, load balancing algorithms, circuit breaking). | The specific rooms or desks designated for each team. |
| **Envoy Sidecar** | High-performance C++ proxy injected alongside every application container in pods with `istio-injection=enabled`. | The personal security escort inspecting every message to and from the container. |

---

## Next.js Application Integration

The Next.js application in `web` is purpose-built to visually and programmatically confirm Canary routing:
- **Badge & Version Display**: Displays the current `APP_VERSION` and `APP_RELEASE_ID` (e.g. `v1.0.0 (stable)` vs `v2.0.0 (canary)`).
- **Endpoint `/api/version`**:
  Returns JSON:
  ```json
  {
    "version": "1.0.0",
    "releaseId": "stable",
    "instanceId": "status-dashboard-v1-xyz",
    "env": "production"
  }
  ```
- **Failure Injection**: Accepts `SIMULATE_UNHEALTHY: "true"` to test health check failures and canary rollback scenarios.
