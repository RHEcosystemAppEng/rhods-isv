## Installation of Grafana 

### Pre-requisites
1. Starburst operator installed
2. MSO operator installed

### Adding ServiceMonitor to enable scraping

This `servicemonitor` allows scraping from `openshift-monitoring` namespace which exposes metrics like `Pod Information` `Container Information` `CPU` `Memory Usage` etc.
 
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: <monitored-namespace>-federation
  namespace: monitoring-stack-operator
  labels:
    app: <monitored-namespace>
spec:
  jobLabel: openshift-monitoring-federation
  namespaceSelector:
    matchNames:
      - openshift-monitoring
  selector:
    matchLabels:
      app.kubernetes.io/instance: k8s
  endpoints:
    - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      interval: 30s
      params:
        'match[]':
          - console_url
          - node_namespace_pod_container:container_memory_working_set_bytes{namespace=<monitored-namespace>}
          - node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace=<monitored-namespace>}
          - namespace_workload_pod:kube_pod_owner:relabel{namespace=<monitored-namespace>}
          - kube_pod_container_info{namespace=<monitored-namespace>}
          - kube_pod_status_ready{namespace=<monitored-namespace>}
          - kube_pod_container_status_last_terminated_reason{namespace=<monitored-namespace>}
          - kube_pod_container_status_waiting{namespace=<monitored-namespace>}
      path: /federate
      port: web
      scheme: https
      tlsConfig:
        caFile: /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
        serverName: prometheus-k8s.openshift-monitoring.svc.cluster.local
      honorLabels: true
```

This `servicemonitor` scrapes `starburst` metrics -

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: starburst-enterprise
  name: starburst-monitor
  namespace: starburst-sb12
spec:
  endpoints:
  - port: metrics
    scrapeTimeout: 2s
    interval: 2s
  namespaceSelector:
    any: true
    matchNames:
    - starburst-sb12
  selector:
    matchLabels:
      app: starburst-enterprise
```

You might see errors of not able to scrape metrics, so for this add `clusterroles` and `clusterrolebindings` to be able to scrape metrics from 
`ns=starburst-sb12` and `ns=openshift-monitoring`

```
oc create clusterrole starburst-sb12-role --verb=get,list,watch --resource=pods,pods/status,endpoints,services --dry-run=client -oyaml

oc create clusterrolebinding starburst-sb12-binding --clusterrole=starburst-sb12-role --serviceaccount=monitoring-stack-operator:monitoring-stack-operator-prometheus --dry-run=client -oyaml
```

```
oc create clusterrolebinding starburst-sb12-prometheus-crb --clusterrole=prometheus-user-workload --serviceaccount=monitoring-stack-operator:starburst-sb12-prometheus --dry-run=client -oyaml
```

You can port-forward `prometheus` and would be able to see all metrics specified -

```
oc port-forward pod/prometheus-pod -n monitoring-stack-operator 9090
```

### Configuring Grafana

- Create namespace dedicated for grafana on `dev/stage` observatorium
- Install grafana operator from operator hub
- Create `grafana` instance

```
apiVersion: integreatly.org/v1alpha1
kind: Grafana
metadata:
  name: grafana
  namespace: grafana
spec:
  client:
    preferService: true
  config:
    log:
      mode: console
      level: info
    security:
      admin_user: <username>
      admin_password: <password>
    auth.anonymous:
      enabled: True
    users:
      viewers_can_edit: true  
  service:
    name: "grafana-operator-controller-manager-metrics-service"
    labels:
      app: "grafana"
      type: "grafana-operator-controller-manager-metrics-service"    
  deployment:
    envFrom:
      - secretRef:
          name: external-credentials    
  dashboardLabelSelector:
    - matchExpressions:
        - { key: app, operator: In, values: [grafana] }
  resources:
    # Optionally specify container resources
    limits:
      cpu: 200m
      memory: 200Mi
    requests:
      cpu: 100m
      memory: 100Mi  
```

- Create `grafana datasource`
```
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: prometheus-grafanadatasource
  namespace: grafana
spec:
  datasources:
    - access: proxy
      editable: true
      isDefault: true
      jsonData:
        httpHeaderName1: 'Authorization'
        timeInterval: 5s
        tlsSkipVerify: true
      name: Prometheus
      secureJsonData:
        httpHeaderValue1: 'Bearer $OAUTH_TOKEN'
      type: prometheus
      url: 'https://observatorium-xyz-observatorium-api.observatorium.svc.cluster.local:8080/api/metrics/v1/test'
  name: prometheus-grafanadatasource.yaml
```

Add `OAUTH_TOKEN` as a secret. You can get `OAUTH_TOKEN` on curling observatorium url.

- Create `grafana dashboard`

Approach - 
1. Port-forward grafana pod to port 3000 - `oc port-forward pod/grafana -n grafana 3000`
2. Login with username and password you specified while creating `grafana` instance
3. Navigate to `create dashboard` and write some queries
4. Navigate to settings logo on the top of the panel and select `JSON Model`
5. Copy the `json model` and paste it to your manifest file like this -

```
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDashboard
metadata:
  name: memory-dashboard
  namespace: grafana  
spec:
  datasources:
  - datasourceName: prometheus-grafanadatasource
    inputName: middleware.yaml
  json: |
    <json-model-copy-and-paste>
```