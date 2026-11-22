# Curriculum Roadmap: Kubernetes-Based Canary Deployment with K3s and Istio

## Project Objective
Build and master an enterprise-grade Canary Deployment simulation where incoming traffic is dynamically split between a Stable version (80%) and a Canary version (20%) using **K3s (lightweight Kubernetes)** and **Istio Service Mesh**. The application running on the cluster is the custom Next.js service dashboard located in the `web` directory.

---

## Deliverables Checklist
- [ ] **K3s Local Cluster**: Running lightweight Kubernetes environment.
- [ ] **Istio Service Mesh**: Control plane installed with automatic Envoy sidecar injection enabled on the target namespace.
- [ ] **Docker Images**: Next.js App packaged into `v1.0.0` (Stable) and `v2.0.0` (Canary).
- [ ] **Kubernetes Manifests**: `v1-deployment.yaml`, `v2-deployment.yaml`, and `service.yaml`.
- [ ] **Istio Traffic Routing**: `gateway.yaml`, `destination-rule.yaml` (subsets `v1` and `v2`), and `virtual-service.yaml` (80/20 traffic weight split).
- [ ] **Traffic Simulation & Verification**: Automated curl loop showing 80% / 20% distribution between versions.
- [ ] **Observability & Telemetry**: Kiali mesh topology graph and Prometheus/Grafana metrics.
- [ ] **Canary Rollback & Promotion Testing**: Simulating failure with `SIMULATE_UNHEALTHY=true` to test rollback, followed by 100% promotion.
- [ ] **Canary Deployment Strategy Document**: Comprehensive operational documentation explaining criteria, rollback gates, and SLA/SLO thresholds.

---

## The 9 Guided Steps

### Step 1: Environment Verification & Local K3s Cluster Setup
- **Goal**: Verify prerequisites (Docker, WSL2, kubectl) and spin up a lightweight K3s cluster (via `k3d` or native K3s in WSL2).
- **Key Concepts**: Kubernetes architecture, master/worker node roles, K3s minimal footprint, `kubeconfig` context.
- **Verification**: `kubectl cluster-info` and `kubectl get nodes` showing `Ready`.

### Step 2: Istio Service Mesh Installation & Namespace Configuration
- **Goal**: Install Istio CLI (`istioctl`) or Helm charts, install Istio using the `demo` or `default` profile, and label the `default` namespace for automatic sidecar injection (`istio-injection=enabled`).
- **Key Concepts**: Control Plane (`istiod`), Data Plane (Envoy proxies), sidecar injection mechanism, `iptables` packet interception.
- **Verification**: `kubectl get pods -n istio-system` showing `istiod` and `istio-ingressgateway` running; namespace label verified.

### Step 3: Containerizing Next.js Web App for Stable (v1) and Canary (v2)
- **Goal**: Review `web/Dockerfile` and compile two distinct container versions:
  - `status-dashboard:v1.0.0` (Stable release, green theme / version 1 badge)
  - `status-dashboard:v2.0.0` (Canary release, purple theme / version 2 badge)
  Load the images directly into the K3s cluster.
- **Key Concepts**: Multi-stage Docker builds, image tagging conventions, local image registry importing in K3s.
- **Verification**: `docker images` showing both versions built successfully and present in K3s image cache.

### Step 4: Authoring Kubernetes Core Manifests (Deployments & Service)
- **Goal**: Create the `k8s/` directory and write:
  - `v1-deployment.yaml` (`app: status-dashboard`, `version: v1.0.0`, 2 replicas)
  - `v2-deployment.yaml` (`app: status-dashboard`, `version: v2.0.0`, 1 replica)
  - `service.yaml` (ClusterIP service targeting selector `app: status-dashboard` on port 3000)
- **Key Concepts**: How label selectors work in standard Kubernetes Services vs. how Istio intercepts and routes between subsets.
- **Verification**: `kubectl get pods -l app=status-dashboard` showing all pods running with `2/2` containers ready (app container + `istio-proxy` sidecar).

### Step 5: Authoring Istio Routing Manifests (Gateway, DestinationRule, VirtualService)
- **Goal**: Create the `istio/` directory and author:
  - `gateway.yaml`: Istio Gateway exposing port 80 for HTTP traffic.
  - `destination-rule.yaml`: Defines subsets `v1` and `v2` based on pod labels (`version: v1.0.0` and `version: v2.0.0`).
  - `virtual-service-canary.yaml`: Route rule distributing traffic with weights: `80%` to subset `v1` and `20%` to subset `v2`.
- **Key Concepts**: The trinity of Istio traffic routing (`Gateway` -> `VirtualService` -> `DestinationRule`), weight-based load balancing.
- **Verification**: `kubectl get gateway,virtualservice,destinationrule` showing healthy resources; `istioctl analyze` reports no configuration errors.

### Step 6: Deploying & Validating the 80/20 Canary Traffic Split
- **Goal**: Find the Istio Ingress Gateway host/port and run a high-volume traffic test script (e.g. 100 requests via PowerShell / curl loop to `/api/version`).
- **Key Concepts**: Measuring statistically significant sample sizes, observing Canary ratio in action, response header analysis.
- **Verification**: Curl test results demonstrating ~80 responses from `v1.0.0` and ~20 responses from `v2.0.0`.

### Step 7: Mesh Observability & Telemetry (Kiali & Prometheus)
- **Goal**: Install Istio sample telemetry addons (Prometheus, Kiali) and launch the Kiali dashboard to visualize the real-time live traffic graph.
- **Key Concepts**: Distributed tracing, Golden Signals (traffic rate, error rate, duration), live topology graph.
- **Verification**: Kiali UI displaying active traffic splitting with percentage badges on edges connecting to `v1` and `v2`.

### Step 8: Simulating Failure & Executing Instant Rollback
- **Goal**: Simulate a bug/degraded state in the Canary release by toggling `SIMULATE_UNHEALTHY: "true"` or introducing artificial error responses. Detect the degradation and immediately apply `virtual-service-rollback.yaml` (100% to v1, 0% to canary).
- **Key Concepts**: Canary blast radius containment, automated vs. manual rollback triggers, MTTR (Mean Time to Resolution).
- **Verification**: Zero traffic routed to `v2.0.0`; all user traffic safely maintained on `v1.0.0` without pod downtime.

### Step 9: Promoting Canary to 100% Production & Strategy Documentation
- **Goal**: After fixing the canary issue, promote `v2.0.0` by applying `virtual-service-promote.yaml` (100% to v2). Write the comprehensive **Canary Deployment Strategy Document** (`CANARY_STRATEGY.md`) as the project capstone.
- **Key Concepts**: Zero-downtime cutover, decommissioning legacy versions, SLO validation gates.
- **Verification**: Complete documentation generated and project fully operational.
