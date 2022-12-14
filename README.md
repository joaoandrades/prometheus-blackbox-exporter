# How to deploy blackbox-exporter on kubernetes

The purpose of this doc is to deploy prometheus **blackbox-exporter** in a kubernetes cluster in a simple and easy way.

# Requirements

 - docker
 - minikube
 - kubectl
 - helm

## Preparing the environment

First you need to create a cluster in minikube:

`minikube start --cpus 8 --memory 12384mb --disk-size='200000mb' --nodes 1`

> Note: The amount of memory and cpus depends on your computer's resource.

Creating the monitoring namespace:

`kubectl create namespace monitoring`

Installing the kube-prometheus-stack with Helm:

 1. Adding helm repository from stack

```console
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

 2. Updating the repository

`helm repo update`

 3. Deploying the stack

For this deployment, the stack values were modified by adding the following block:

```yaml
    additionalScrapeConfigs:
      - job_name: blackbox
        metrics_path: /probe
        params:
          module: [http_2xx]
        static_configs:
          - targets:
            - https://www.google.com
        relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: prometheus-blackbox-exporter.monitoring:9115
```
The yaml file that is in the **kube-stack-prometheus** directory of this repository.
```
helm upgrade --install kube-stack-prometheus prometheus-community/kube-prometheus-stack -f kube-stack-prometheus/values.yaml -n monitoring
```
It is important, after deploying, to verify that the pods are OK (with running status):

`kubectl get pods -n monitoring`

 4. Accessing the prometheus interface
```
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-kube-prometheus 9090:9090
```
Just go to the browser and access: http://localhost:9090
 
 5. Accessing Grafana
```
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-grafana 8080:80
```
Just go to the browser and access: http://localhost:8080

User: admin

Pass: prom-operator

## Prometheus blackbox-exporter

Before deploying blackbox-exporter it is necessary to make some changes to *values.yaml*. Just as the stack's *values.yaml* was modified, the blackbox one also has some modifications:

```yaml
config:
  modules:
    http_2xx:
      prober: http
      timeout: 5s
      http:
        valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
        # A modificao acontece aqui
        valid_status_codes: [200, 403]
        ##############################
        follow_redirects: true
        preferred_ip_protocol: "ip4"
```

> Remembering that it is possible to configure/modify the values according to your needs.

 - Deploy
```console
helm install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter -f blackbox-exporter/values.yaml -n monitoring
```

After the deployment is done, it is interesting to visualize the metrics on a dashboard, for that we will access Grafana again:
```
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-grafana 8080:80
```
Just go to the browser and access: http://localhost:8080

User: admin

Pass: prom-operator

Go to Dashboards > Import and insert the dashboard code which is: **7587**.

Finally, the end result is this:

![Dashboard](./dashboard/dash-blackbox.png)
