# kube-prometheus-stack

This project is the integration of kube-prometheus-stack, including deployment, installation, setup, and usage.

## Deploy

1. Add the repository of project  
```console
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts  
helm repo update  
helm search repo prometheus-community --versions
```
2. Select a version and export the file 'values.yaml' to the current directory  
```console
helm show values prometheus-community/kube-prometheus-stack --version 78.3.2 > ./values.yaml
```
3. Install the project  
```console
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 78.3.2 -n monitoring -f ./values.yaml
```
4. Check the running status of the project, which is installed by default in the namespace 'monitoring' 
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
1. You can set the type of prometheus service to NodePort and expose a port, and then access the web page through this node and port.
```yaml
# values.yaml
prometheus:
  service:
    type: NodePort
    nodePort: 32006
```

### Grafana
1. You can set the type of grafana service to NodePort and expose a port, and then access the web page through this node and port.
```yaml
# values.yaml
grafana:
  service:
    type: NodePort
    nodePort: 32007
```

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