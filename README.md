# kube-prometheus-stack

This project is the integration of kube-prometheus-stack, including deployment, installation, setup, and usage.

## Reference
https://prometheus.io/docs/prometheus/latest/getting_started/


## Deploy
### 1. Add the repository of project  
```console
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  
helm repo update  
helm search repo prometheus-community/kube-prometheus-stack --versions
```
### 2. export the values.yaml file
Select a version and export the file 'values.yaml' to the current directory  
```console
helm show values prometheus-community/kube-prometheus-stack --version 87.16.1 > ./values.yaml
```

### 3. Modify the values.yaml file
#### 3.1. Set the storage configuration field for prometheus
```console
prometheus.prometheusSpec.storageSpec
prometheus.prometheusSpec.retention
prometheus.prometheusSpec.retentionSize
```

#### 3.2. Disable scraping for the components
setup the value to false
```console
kubeEtcd.enabled
kubeScheduler.enabled
kubeControllerManager.enabled
```

#### 3.3. Grafana
```console
grafana.ingress.enabled: false  # use envoy-gateway
grafana.adminPassword
grafana.persistence
```

### 4. Pull images
Some images may not be able to be pulled down, you can pull other repositories and use tags to replace the original image address.

#### 4.1. Based on Docker
```console
docker pull k8s.nju.edu.cn/kube-state-metrics/kube-state-metrics:v2.19.1
docker tag k8s.nju.edu.cn/kube-state-metrics/kube-state-metrics:v2.19.1 registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.19.1
```

#### 4.2. Based on Containerd
```console
alias rke2-ctr='sudo /var/lib/rancher/rke2/bin/ctr --address /run/k3s/containerd/containerd.sock --namespace k8s.io'

rke2-ctr images pull k8s.nju.edu.cn/kube-state-metrics/kube-state-metrics:v2.19.1
rke2-ctr images tag k8s.nju.edu.cn/kube-state-metrics/kube-state-metrics:v2.19.1 registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.19.1
```

### 5. Install the project  
```console
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 87.16.1 -n monitoring -f ./values.yaml
```
### 6. Check the status
Check the running status of the project, which is installed by default in the namespace 'monitoring' 
```console
helm list -n monitoring
```

## Undeploy
Directly deleting the helm chart project will delete all related resources  
```console
helm uninstall kube-prometheus-stack -n monitoring
```

## Upgrade
Use the following command to apply your values.yaml file  
```console
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring -f ./values.yaml
```

## Web page access method

### Prometheus
#### 1. NodePort 
You can set the type of prometheus service to NodePort and expose a port, and then access the web page through this node and port.
```yaml
# values.yaml
prometheus:
  service:
    type: NodePort
    nodePort: 32006
```
http://node-ip:32006/query

#### 2. envoy-gateway
```console
kubectl label namespace monitoring gateway=eg-public
kubectl apply -f ./http-route-prometheus.yaml -n monitoring
```

http://prometheus.agf.com:30180/query

### Grafana
You can set the type of grafana service to NodePort and expose a port, and then access the web page through this node and port.
#### 1. NodePort 
```yaml
# values.yaml
grafana:
  service:
    type: NodePort
    nodePort: 32007
```

#### 2. envoy-gateway
```console
kubectl apply -f ./http-route-grafana.yaml -n monitoring
```
http://grafana.agf.com:30180/

## Monitoring Metrics
Prometheus listens to the created ServiceMonitors, which indicate how Prometheus accesses the metrics server.
1. Create a ServiceMonitor for the specified metrics server in this project.  
If your environment does not have a ServiceMonitor CRD and you do not want to create it, you can use this method to create the ServiceMonitor in the namespace.
```yaml
# Monitor the specified serviceMonitor for the namespace 'mongodb-k8s-operator-system'
# values.yaml
prometheus:
  additionalServiceMonitors:
    - name: "mongodb-k8s-operator-controller-manager-metrics-monitor"
      selector:
        matchLabels:
          app.kubernetes.io/name: "mongodb-k8s-operator"
      namespaceSelector:
        matchNames:
          - "mongodb-k8s-operator-system"
      endpoints:
        - port: "https"
          bearerTokenFile: "/var/run/secrets/kubernetes.io/serviceaccount/token"
          path: /metrics
          scheme: https
          tlsConfig:
            insecureSkipVerify: true
```

2. Monitor an existing ServiceMonitor
```yaml
# Monitor the specified serviceMonitor for the namespace 'mongodb-k8s-operator-system'
# values.yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelector:
      matchLabels:
        app.kubernetes.io/name: "mongodb-k8s-operator"
    serviceMonitorNamespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: mongodb-k8s-operator-system
```

## Import Grafana Dashboards