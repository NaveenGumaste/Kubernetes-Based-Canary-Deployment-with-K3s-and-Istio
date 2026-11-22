# Canary Deployment Strategy

## Scope

The status dashboard runs on the local K3s cluster `canary-cluster`. Kubernetes runs two stable replicas (`v1.0.0`) and one canary replica (`v2.0.0`) behind the shared `status-dashboard` Service. The Istio Gateway receives ingress traffic, the DestinationRule maps version labels to subsets, and the VirtualService assigns traffic weights.

Both releases use the same application build for this lab. Runtime configuration supplies the release metadata and lets us simulate a canary failure.

## Route selection

Apply one route file at a time. Each file updates the same `status-dashboard` VirtualService in the `default` namespace.

| Action           | Manifest                              | Stable | Canary |
| ---------------- | ------------------------------------- | -----: | -----: |
| Start the canary | `istio/virtual-service-canary.yaml`   |    80% |    20% |
| Roll back        | `istio/virtual-service-rollback.yaml` |   100% |     0% |
| Promote          | `istio/virtual-service-promote.yaml`  |     0% |   100% |

## SLOs and release gates

These are proposed targets for this lab runbook. The lab has no customer SLA.

| Measure            | Proposed service SLO over a rolling 30-day window | Canary promotion gate                                      |
| ------------------ | ------------------------------------------------- | ---------------------------------------------------------- |
| HTTP server errors | HTTP 5xx rate at or below 0.1%                    | Canary 5xx rate at or below 1%                             |
| Request latency    | p95 at or below 300 ms                            | Canary p95 at or below 300 ms and within 20% of stable p95 |
| Readiness          |                                                   | All expected canary replicas ready                         |

Evaluate promotion after at least five minutes and 200 canary responses. Use Prometheus and Grafana for the error and latency measurements. Use the Kiali Versioned App graph to confirm traffic reaches both versions. If traffic is too low to evaluate the gates, keep the current weights while collecting more data.

## Rollback and recovery

Roll back if the canary loses all ready replicas, shows a repeatable server error, or breaches a promotion gate after the observation window.

```powershell
kubectl --context k3d-canary-cluster apply -f .\istio\virtual-service-rollback.yaml
```

After the route propagates, send 50 requests to `http://127.0.0.1/api/status`. Confirm every request succeeds and reports `1.0.0 (stable)`. Fix and verify the canary while stable handles ingress traffic. For a future release, repeat the 80/20 observation before promoting again.

After promotion, keep stable replicas ready through the observation window so traffic can be switched back. Scale down the old release once the promoted version has met the release gates throughout that window.

## Lab results

| Check                     | Observed result                                           |
| ------------------------- | --------------------------------------------------------- |
| 80/20 ingress test        | 84 stable and 16 canary responses in 100 requests         |
| Kiali, five-minute window | 72.6% stable and 27.4% canary, with 100% reported success |
| Simulated canary failure  | 41 HTTP 200s and 9 HTTP 500s in 50 requests               |
| Traffic rollback          | 50 stable responses, zero errors                          |
| Canary recovery           | `2.0.0` reported `healthy` from inside the canary pod     |
| Promotion                 | 50 canary responses, zero errors                          |

The final lab route sends 100% of ingress traffic to `v2.0.0`. The checks above verify routing, failure, rollback, and promotion in the lab.
