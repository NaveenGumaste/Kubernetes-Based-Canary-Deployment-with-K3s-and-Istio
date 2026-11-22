# Kubernetes-Based Canary Deployment with K3s and Istio

A local canary deployment lab for a Next.js service status dashboard. Two Kubernetes Deployments run side by side while Istio controls the share of incoming HTTP requests each version receives. The lab covers an 80/20 rollout, a simulated canary failure, a traffic rollback, and promotion after the canary recovers.

## How traffic flows

~~~mermaid
flowchart LR
    Client["Browser or PowerShell"] --> Ingress["Istio ingress · HTTP 80"]
    Ingress --> Service["status-dashboard Service · HTTP 3000"]
    Service -->|"80% to subset v1"| Stable["Stable v1.0.0 · 2 pods"]
    Service -->|"20% to subset v2"| Canary["Canary v2.0.0 · 1 pod"]
~~~

The Gateway sets the ingress listener. The DestinationRule maps subsets to the pods' version labels, and the VirtualService sets the traffic weights. The ingress proxy selects the backend. Both image tags use the same dashboard build for the simulation. Each Deployment supplies its own release metadata, visible in the UI and at `/api/version`.

## Run the lab

The lab was tested on Windows with PowerShell, Docker Desktop in Linux container mode, k3d 5.9.0, K3s 1.35.5, and Istio 1.32.0. Install Docker, k3d, kubectl, and istioctl before running the commands. The samples use a local ingress on port 80 and one K3s server node.

### 1. Start K3s and install Istio

~~~powershell
k3d cluster create canary-cluster --image rancher/k3s:v1.35.5-k3s1 --port '127.0.0.1:80:80@loadbalancer' --k3s-arg '--disable=traefik@server:0'
$context = 'k3d-canary-cluster'

kubectl --context $context get nodes
istioctl --context $context install --set profile=demo -y
kubectl --context $context label namespace default istio-injection=enabled --overwrite
~~~

The demo profile supplies the ingress gateway. The namespace label enables automatic proxy injection for the dashboard pods. [k3d setup](https://istio.io/latest/docs/setup/platform-setup/k3d/), [Istioctl installation](https://istio.io/latest/docs/setup/install/istioctl/)

### 2. Build and deploy the dashboard

Run from the repository root.

~~~powershell
docker build -t status-dashboard:v1.0.0 -t status-dashboard:v2.0.0 .\web
k3d image import status-dashboard:v1.0.0 status-dashboard:v2.0.0 --cluster canary-cluster

kubectl --context $context apply -f .\k8s
kubectl --context $context rollout status deployment/status-dashboard-v1 --timeout=120s
kubectl --context $context rollout status deployment/status-dashboard-v2 --timeout=120s
kubectl --context $context get pods -l app=status-dashboard -L version
~~~

### 3. Enable and verify canary routing

~~~powershell
kubectl --context $context apply -f .\istio\gateway.yaml -f .\istio\destination-rule.yaml -f .\istio\virtual-service-canary.yaml
istioctl --context $context analyze -n default

$versions = foreach ($request in 1..100) {
    $response = Invoke-RestMethod -Uri 'http://127.0.0.1/api/version' -TimeoutSec 5 -ErrorAction Stop
    $response.releaseId
}
$versions | Group-Object | Select-Object Count, Name
~~~

Open the [dashboard](http://127.0.0.1/) or check the results in PowerShell. Each batch should approach the configured split as the sample grows.

## Monitor the rollout

Install the sample Prometheus, Kiali, and Grafana add-ons from the matching Istio release.

~~~powershell
$addons = 'https://raw.githubusercontent.com/istio/istio/1.32.0/samples/addons'
kubectl --context $context apply -f "$addons/prometheus.yaml" -f "$addons/kiali.yaml" -f "$addons/grafana.yaml"
kubectl --context $context -n istio-system get deployments prometheus kiali grafana
~~~

Run each port-forward in a separate terminal after the Deployments are ready.

~~~powershell
kubectl --context k3d-canary-cluster -n istio-system port-forward service/kiali 20001:20001
kubectl --context k3d-canary-cluster -n istio-system port-forward service/grafana 13000:3000
~~~

In Kiali, select the default namespace and the Versioned App traffic graph. In Grafana, open the Istio Service Dashboard and select `status-dashboard`. Generate fresh ingress requests while the dashboards are open. [Kiali](http://127.0.0.1:20001/kiali/), [Grafana](http://127.0.0.1:13000/)

## Change the traffic split

Apply one route manifest at a time. All three update the same VirtualService.

| Action | Manifest | Stable | Canary |
| --- | --- | ---: | ---: |
| Start canary | [virtual-service-canary.yaml](istio/virtual-service-canary.yaml) | 80% | 20% |
| Roll back | [virtual-service-rollback.yaml](istio/virtual-service-rollback.yaml) | 100% | 0% |
| Promote | [virtual-service-promote.yaml](istio/virtual-service-promote.yaml) | 0% | 100% |

With the 80/20 route active, set the failure flag on the canary and wait for its rollout.

~~~powershell
kubectl --context $context set env deployment/status-dashboard-v2 --containers=dashboard SIMULATE_UNHEALTHY=true
kubectl --context $context rollout status deployment/status-dashboard-v2 --timeout=90s
~~~

Sample `/api/status` to observe HTTP errors. Apply the rollback to send all ingress traffic to stable and sample the same endpoint again.

~~~powershell
kubectl --context $context apply -f .\istio\virtual-service-rollback.yaml
~~~

Remove the failure flag and verify the canary directly while stable serves ingress traffic.

~~~powershell
kubectl --context $context set env deployment/status-dashboard-v2 --containers=dashboard SIMULATE_UNHEALTHY-
kubectl --context $context rollout status deployment/status-dashboard-v2 --timeout=120s
kubectl --context $context exec deployment/status-dashboard-v2 -c dashboard -- curl -fsS http://127.0.0.1:3000/api/status
~~~

After v2 reports healthy, promote it and recheck `/api/status` through ingress.

~~~powershell
kubectl --context $context apply -f .\istio\virtual-service-promote.yaml
~~~

The [canary strategy](CANARY_STRATEGY.md) defines the proposed release gates for longer observation windows.

## Results

| Check | Observed result |
| --- | --- |
| 80/20 route | 84 stable and 16 canary responses in 100 requests |
| Kiali traffic graph | 72.6% stable and 27.4% canary in a separate five-minute window |
| Simulated failure | 41 HTTP 200 responses and 9 HTTP 500 responses |
| Rollback | 50 successful stable responses |
| Promotion | 50 successful canary responses |

## Repository layout

- [web/](web/) contains the Next.js dashboard, tests, and Dockerfile.
- [k8s/](k8s/) contains the two Deployments and shared Service.
- [istio/](istio/) contains Gateway, subset, and traffic routing manifests.
- [CANARY_STRATEGY.md](CANARY_STRATEGY.md) records the release procedure, rollback criteria, and test results.
