# Kubernetes Observability Stack with Fluent Bit, Loki & Grafana

This project demonstrates a minimal yet complete observability stack built on Kubernetes using:

- **Fluent Bit** for log collection
- **Grafana Loki** for log aggregation and storage
- **Grafana** for visualization and alerting

It also includes a **random log-generating application** to simulate real-time success/failure events and dashboards to monitor these events visually.

The stack is deployed on a **MicroK8s** cluster with **Istio** enabled for service mesh and routing.

## Project Structure

The repository is organized as follows:

```
.
├── deployments/                  # Kubernetes manifests for all components
│   ├── apps/                     # Log-generating pods
│   │   ├── error-deploy.yaml     # Randomized 'failed' log producer
│   │   └── success-deploy.yaml   # Randomized 'success' log producer
│   ├── fluentbit/                # Fluent Bit setup
│   │   ├── fluent-bit-daemonset.yaml
│   │   └── fluent-bit-rbac.yaml
│   ├── grafana/                  # Grafana deployment & Istio route
│   │   ├── grafana-deployment.yaml
│   │   └── grafana-virtualservice.yaml
│   ├── loki/                     # Loki deployment & Istio route
│   │   ├── loki-deployment.yaml
│   │   └── loki-virtualservice.yaml
│   └── istio/                    # Istio Gateway definition
│       └── gateway.yaml
│
├── manifests/
│   └── namespace.yaml            # Namespace definition for 'obs'
│
├── dashboards/                   # Grafana dashboards (JSON)
│
├── assets/                       # Visuals and media assets for documentation
│   └── images/                     
│       ├── error.png
│       └── success.png
│
├── LICENSE                       # Project license (MIT)
├── CONTRIBUTING.md               # How to contribute to this repo
└── README.md                     # This file
```

## Getting Started

This project sets up a minimal observability stack with:

- A pod generating random `success` and `failed` logs
- **Fluent Bit** collecting logs and forwarding to **Loki**
- **Loki** storing and indexing the logs
- **Grafana** visualizing the logs in time series panels
- **Istio** managing **Ingress** routing

## Installation

- Enable required MicroK8s add-ons

```bash
microk8s enable dns storage istio
```

- Clone this repository

```bash
git clone https://github.com/tuncayyaylali/observability-on-microk8s.git
cd observability-on-microk8s
```

- Apply the namespace

```bash
kubectl apply -f manifests/namespace.yaml
```

- Deploy the Istio Gateway

```bash
kubectl apply -f manifests/istio/gateway.yaml
```

## Configuration

This project uses Kubernetes-native YAML configurations for all components. Here is how each component is configured:

### Fluent Bit
A `DaemonSet` is used to run **Fluent Bit** on each node, tailing logs from `/var/log/containers` and sending them to **Loki**. 
`RBAC` rules and a `ConfigMap` are also applied.

```bash
kubectl apply -f manifests/fluent-bit/fluent-bit-rbac.yaml
kubectl apply -f manifests/fluent-bit/fluent-bit-daemonset.yaml
```

### Loki
A single pod **Loki** deployment configured with:
- Persistent volume using `emptyDir` (for MicroK8s)
- **Istio** `VirtualService` for internal routing

```bash
kubectl apply -f manifests/loki/oki-deployment.yaml
kubectl apply -f manifests/loki/loki-virtualservice.yaml
```

### Grafana
**Grafana** is deployed with `NodePort` access and is configured to use **Loki** as a data source:

```bash
cd ./manifests/grafana
kubectl apply -f manifests/grafana/grafana-deployment.yaml
kubectl apply -f manifests/grafana/grafana-virtualservice.yaml
```

### Deploy Log Generators (Pods with Random Logs)

This project includes two pods that continuously generate logs with different success/failure patterns for observability testing.

### random-error-logger

A simple pod that logs messages to `stderr` at random intervals, with a mix of `failed` and `success` messages.

- Apply it with:

```bash
kubectl apply -f manifests/apps/error-deploy.yaml
```

### random-success-logger

A simple pod that logs messages to `stderr` at random intervals, with a mix of `failed` and `success` messages.

- Apply it with:

```bash
kubectl apply -f manifests/apps/success-deploy.yaml
```

## Dashboards & Visualizations

**Grafana** is used to visualize the logs flowing through **Loki**. The following visualizations are available out of the box:

### Accessing Grafana & Loki via Istio Gateway

To access the web UIs externally using friendly domain names:

- **Istio Gateway** is configured with a `static external IP` using **MetalLB** or similar.
- Add the following entries to your host machine’s `/etc/hosts` file:

```bash
<EXTERNAL-IP> grafana.observability.local loki.observability.local
```

- Replace `<EXTERNAL-IP>` with the external IP assigned to your Istio gateway (e.g., via `kubectl get svc istio-ingressgateway -n istio-system`).
- Access the UIs in your browser:
    - Grafana: [http://grafana.observability.local](http://grafana.observability.local)
    - Loki API: [http://loki.observability.local](http://loki.observability.local)

- Configure **Loki** as a Data Source in **Grafana**:
   - Go to **Grafana** → *Connections* → *Data Sources* → *Add new data source* → *Loki*  
   - Add new **Loki** data source with:

     ```http
     URL: http://loki.observability.local:3100
     ```
   
   - Save and test the connection.

### Pod Selector (Variable)

This variable allows users to dynamically filter logs by specific pod names within the `obs` namespace.

- Go to *Dasboard* → [Select] → *Dashboard settings* → *Variables* → *New variable*
- Set the following:
   - Name: `pod`
   - Type: `Query`
   - Data source: `Loki`
   - Query types: `Label values`
   - Label: `pod`
   - Stream selector: `{namespace="obs"}`

- You can use this variable in your log panel stream selectors to filter logs dynamically:

```logql
{namespace="obs", pod=~"$pod"}
```

### Time Series Graphs

- Failed Log Events (Last 5 Minutes)  
    - Shows the number of `failed` log entries generated by pods labeled with `app=error-logger` in the `obs` namespace.

    ```logql
    {namespace="obs", pod=~"$pod"} |= "failed" 
    ```

- Success Log Events (Last 5 Minutes)
    - Shows the number of `success` log entries from the same pods.

    ```logql
    {namespace="obs", pod=~"$pod"} |= "success" 
    ```

- Combined View
    - Displays both `success` and `failed` log events over time on the same graph for comparison.

    ```logql
    count_over_time({namespace="obs", pod=~"$pod"} |= "failed" [1h])
    count_over_time({namespace="obs", pod=~"$pod"} |= "success" [1h])
    ```

### Sample Grafana Visualizations

#### error.png

![Grafana Overview for \`random-error-log`](assets/images/error.png)

**Figure 1:** Time-series visualization of `failed/success` log messages generated by the `random-error-logger` pod.  
- This panel shows spikes and trends in error frequency over time, useful for detecting instability or failure patterns.

#### success.png

![Grafana Overview for \`random-success-log`](assets/images/success.png)

**Figure 2:** Time-series visualization of `failed/success` log messages generated by the `random-success-logger` pod.   
- Helps monitor the ratio of healthy vs faulty log activity and can be compared against error logs to determine system health.