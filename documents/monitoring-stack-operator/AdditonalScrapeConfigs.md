# Additional Scrape Configs

**IMPORTANT - PLEASE READ FIRST BEFORE STARTING**  
_This is an experimental flow, it is **not supported**, in fact, thought to be impossible as far as MSO team is concerned. Use this flow at your own risk in usecases where a work-around is required to deploy ServiceMonitors and PrometheusRules without using the actual CRDs._

Work has been done in the `rhods-isv/install/isv-monitoring-incluster/install` directory. The idea is to configure MSO without `PrometheusRules` or `ServiceMonitors`.


- [Install Operator](#install-operator)
- [Deploy the MSO](#deploy-the-mso)
- [Deploy Blue](#deploy-blue)
- [Check the console](#check-the-console)
- [Create ScrapeJobs](#create-scrapejobs)
- [Create the Rules](#create-the-rules)
- [Clean Up](#clean-up)
- [Misc Debugging](#debugging)


## Install Operator

```bash
make create/operator
```

## Deploy the MSO

Deploy the MSO and delete the `PrometheusRules` and `ServiceMonitors`

```bash
make create/mso/dev

oc delete servicemonitor,prometheusrules --all -n monitoring-stack-operator
```

## Deploy Blue

We will be using this application for metrics

```yaml
oc apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: null
  name: blue
spec: {}
status: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: blue
    version: v1
  name: blue
  namespace: blue
spec:
  ports:
    - port: 9000
      name: http
  selector:
    app: blue
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: blue
    version: v1
  name: blue
  namespace: blue
spec:
  selector:
    matchLabels:
      app: blue
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: blue
        version: v1
    spec:
      serviceAccountName: blue
      containers:
        - image: docker.io/cmwylie19/metrics-demo
          name: blue
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: 9000
              name: http
          imagePullPolicy: Always
      restartPolicy: Always
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: blue
  namespace: blue
EOF
```

## Check the console
```bash
oc port-forward svc/prometheus-operated -n monitoring-stack-operator 9090
```

Check the [rules](http://localhost:9090/rules)

Check the [targets](http://localhost:9090/targets)

Leave the service port-forwarding

## Create ScrapeJobs

Create the scrape-config file to go into the secret
```yaml
cat <<EOF> self-scrape-config
- job_name: serviceMonitor/monitoring-stack-operator/starburst-sb12-federation/0
  honor_labels: true
  honor_timestamps: true
  params:
    match[]:
    - node_namespace_pod_container:container_memory_working_set_bytes{namespace="blue"}
    - node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace="blue"}
    - namespace_workload_pod:kube_pod_owner:relabel{namespace="blue"}
    - kube_pod_container_info{namespace="blue"}
    - kube_pod_status_ready{namespace="blue"}
    - kube_pod_container_status_last_terminated_reason{namespace="blue"}
    - kube_pod_container_status_waiting{namespace="blue"}
    - kube_namespace_status_phase{namespace="blue"}
    - node_namespace_pod:kube_pod_info:{namespace="blue"}
    - kube_service_info{namespace="blue"}
    - cluster:namespace:pod_memory:active:kube_pod_container_resource_limits{namespace="blue"}
    - container_cpu_cfs_throttled_seconds_total{namespace="blue"}
    - container_fs_usage_bytes{namespace="blue"}
    - container_network_receive_bytes_total{namespace="blue"}
    - container_network_transmit_bytes_total{namespace="blue"}
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /federate
  scheme: https
  authorization:
    type: Bearer
    credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
    server_name: prometheus-k8s.openshift-monitoring.svc.cluster.local
    insecure_skip_verify: false
  follow_redirects: true
  enable_http2: true
  relabel_configs:
  - source_labels: [job]
    separator: ;
    regex: (.*)
    target_label: __tmp_prometheus_job_name
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_instance, __meta_kubernetes_service_labelpresent_app_kubernetes_io_instance]
    separator: ;
    regex: (k8s);true
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_port_name]
    separator: ;
    regex: web
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Node;(.*)
    target_label: node
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Pod;(.*)
    target_label: pod
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: service
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_name]
    separator: ;
    regex: (.*)
    target_label: pod
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_container_name]
    separator: ;
    regex: (.*)
    target_label: container
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: job
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_service_label_openshift_monitoring_federation]
    separator: ;
    regex: (.+)
    target_label: job
    replacement: ${1}
    action: replace
  - separator: ;
    regex: (.*)
    target_label: endpoint
    replacement: web
    action: replace
  - source_labels: [__address__]
    separator: ;
    regex: (.*)
    modulus: 1
    target_label: __tmp_hash
    replacement: $1
    action: hashmod
  - source_labels: [__tmp_hash]
    separator: ;
    regex: "0"
    replacement: $1
    action: keep
  kubernetes_sd_configs:
  - role: endpoints
    kubeconfig_file: ""
    follow_redirects: true
    enable_http2: true
    namespaces:
      own_namespace: false
      names:
      - openshift-monitoring
- job_name: serviceMonitor/monitoring-stack-operator/starburst-sb12-monitor/0
  honor_timestamps: true
  scrape_interval: 2s
  scrape_timeout: 2s
  metrics_path: /metrics
  scheme: http
  follow_redirects: true
  enable_http2: true
  relabel_configs:
  - source_labels: [job]
    separator: ;
    regex: (.*)
    target_label: __tmp_prometheus_job_name
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_label_app, __meta_kubernetes_service_labelpresent_app]
    separator: ;
    regex: (blue);true
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_port_name]
    separator: ;
    regex: http
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Node;(.*)
    target_label: node
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_endpoint_address_target_kind, __meta_kubernetes_endpoint_address_target_name]
    separator: ;
    regex: Pod;(.*)
    target_label: pod
    replacement: ${1}
    action: replace
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: service
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_name]
    separator: ;
    regex: (.*)
    target_label: pod
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_container_name]
    separator: ;
    regex: (.*)
    target_label: container
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: job
    replacement: ${1}
    action: replace
  - separator: ;
    regex: (.*)
    target_label: endpoint
    replacement: http
    action: replace
  - source_labels: [__address__]
    separator: ;
    regex: (.*)
    modulus: 1
    target_label: __tmp_hash
    replacement: $1
    action: hashmod
  - source_labels: [__tmp_hash]
    separator: ;
    regex: "0"
    replacement: $1
    action: keep
  kubernetes_sd_configs:
  - role: endpoints
    kubeconfig_file: ""
    follow_redirects: true
    enable_http2: true
- job_name: prometheus-self
  honor_labels: true
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_app_kubernetes_io_name
    regex: starburst-sb12-prometheus
  - action: keep
    source_labels:
    - __meta_kubernetes_endpoint_port_name
    regex: web
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace
  - source_labels:
    - __meta_kubernetes_service_name
    target_label: service
  - source_labels:
    - __meta_kubernetes_pod_name
    target_label: pod
  - source_labels:
    - __meta_kubernetes_pod_container_name
    target_label: container
  - target_label: endpoint
    replacement: web
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - monitoring-stack-operator
- job_name: alertmanager-self
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  follow_redirects: true
  relabel_configs:
  - source_labels:
    - __meta_kubernetes_service_label_app_kubernetes_io_name
    separator: ;
    regex: starburst-sb12-alertmanager
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_port_name]
    separator: ;
    regex: web
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_namespace]
    separator: ;
    regex: (.*)
    target_label: namespace
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_service_name]
    separator: ;
    regex: (.*)
    target_label: service
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_name]
    separator: ;
    regex: (.*)
    target_label: pod
    replacement: $1
    action: replace
  - source_labels: [__meta_kubernetes_pod_container_name]
    separator: ;
    regex: (.*)
    target_label: container
    replacement: $1
    action: replace
  - separator: ;
    regex: (.*)
    target_label: endpoint
    replacement: web
    action: replace
  kubernetes_sd_configs:
  - role: endpoints
    kubeconfig_file: ""
    follow_redirects: true
    namespaces:
      names:
      - monitoring-stack-operator
EOF
```

Create Secret pre-populated with scrape jobs

```
oc delete secret starburst-sb12-prometheus-additional-scrape-configs -n monitoring-stack-operator

oc create secret generic starburst-sb12-prometheus-additional-scrape-configs -n monitoring-stack-operator --from-file=self-scrape-config 
```

Check the [targets](http://localhost:9090/targets), wait until you see them.

## Create the Rules
Create the rules file to go into the configmap
```yaml
cat <<EOF> monitoring-stack-operator-starburst-sb12-rules-40c23e63-92a3-4f5c-bab0-8b628a0b240e.yaml
groups:
- interval: 1s
  name: starburst_custom_rules
  rules:
  - expr: avg_over_time(jvm_memory_bytes_used{endpoint="metrics"}[5m])
    record: starburst_query_mem
  - expr: jvm_memory_bytes_max{endpoint="metrics", area="heap"}
    record: starburst_max_query_mem
  - expr: jvm_memory_bytes_used{endpoint="metrics",area="heap"}
    record: starburst_heap_mem
  - expr: jvm_memory_bytes_max{endpoint="metrics",area="heap"}
    record: starburst_max_heap_mem
- name: starburst_alert_rules
  rules:
  - alert: high_starburst_query_mem
    annotations:
      description: High average memory used by all queries over a given time period
      summary: High Query Memory
    expr: starburst_query_mem >= 45158388108
    labels:
      severity: page
  - alert: high_starburst_max_query_mem
    annotations:
      description: High amount of heap memory used by the JVMs across all cluster
        nodes
      summary: High Heap Memory
    expr: starburst_max_query_mem >= 94489280512
    labels:
      severity: warn
  - alert: high_starburst_heap_mem
    annotations:
      description: The max amount of heap memory configured in the JVM aggregated
        across the entire cluster
      summary: High Max Heap Memory
    expr: starburst_heap_mem >= 45631505600
    labels:
      severity: page
  - alert: high_starburst_max_heap_mem
    annotations:
      description: The max amount of heap memory configured in the JVM aggregated
        across the entire cluster
      summary: High Max Heap Memory Alert
    expr: starburst_max_heap_mem >= 94489280512
    labels:
      severity: acknowledged
  - alert: starburst_instance_down
    annotations:
      description: The pods churned
      summary: Starburst instance down
    expr: count(up{endpoint="metrics"}) != 3
    labels:
      severity: page
  - alert: high_thread_count
    annotations:
      description: High Thread Count
      summary: High Thread Count
    expr: sum(thread_count) > 400
    labels:
      severity: page
#   - alert: JvmMemoryFillingUp
#     annotations:
#       description: |-
#         JVM memory is filling up (> 80%)
#           VALUE = {{ $value }}
#           LABELS = {{ $labels }}
#       summary: JVM memory filling up (instance {{ $labels.instance }})
#     expr: (sum by (instance)(jvm_memory_bytes_used{area="heap"}) / sum by (instance)(jvm_memory_bytes_max{area="heap"}))
#       * 100 > 80
#     for: 2m
#     labels:
#       severity: page
  - alert: starburst_failed_queries
    annotations:
      description: In the last 5 mins the failed queries have risen
      summary: Queries are failing
    expr: failed_queries >= 4
    labels:
      severity: page
  - alert: trino_node_failure
    annotations:
      description: An active trino node went down
      summary: Trino node failure
    expr: trino_active_nodes <= 1
    labels:
      severity: page
EOF
```

Unlabel the configmap so that it is not overwritten by the operator

```bash
oc label cm prometheus-starburst-sb12-rulefiles-0 -n monitoring-stack-operator prometheus-name- managed-by-    
```

Wait a few seconds, then delete and recreate the configmap (the few seconds are needed for the operator to ignore the configmap)
```bash
oc delete cm prometheus-starburst-sb12-rulefiles-0 -n monitoring-stack-operator 

oc create cm prometheus-starburst-sb12-rulefiles-0 -n monitoring-stack-operator --from-file=monitoring-stack-operator-starburst-sb12-rules-40c23e63-92a3-4f5c-bab0-8b628a0b240e.yaml
```

Check the [rules](http://localhost:9090/rules), wait until you see them.

## Clean Up

Delete the ScrapeJobs

```bash
oc delete secret starburst-sb12-prometheus-additional-scrape-configs -n monitoring-stack-operator
```

Delete the rules

```bash
oc delete cm prometheus-starburst-sb12-rulefiles-0 -n monitoring-stack-operator 
```

Clean up MSO

```bash
make clean/mso
```

Clean Operator

```bash
make clean/operator
```

Clean Blue

```bash
oc delete deploy,svc,sa,po --all -n blue --force
oc delete ns blue
```

## Debugging

Get Scrape Configs

```bash
k get secret -n monitoring-stack-operator -oyaml starburst-sb12-prometheus-additional-scrape-configs -ojsonpath='{.data.self-scrape-config}' | base64 -d
```

Alert Manager Rules:

```
k get cm -n monitoring-stack-operator prometheus-starburst-sb12-rulefiles-0 -oyaml
```

Checking that the configmap was loaded into the prom pod

```bash
k exec -it -n monitoring-stack-operator prometheus-starburst-sb12-0 -- ls -l /etc/prometheus/rules/prometheus-starburst-sb12-rulefiles-0/
```

Read the logs from the prom pod regarding rules

```bash
k logs sts/prometheus-starburst-sb12  -n monitoring-stack-operator --since=1m| grep rules
```


There is a problem with one of our rules `__alert_JvmMemoryFillingUp`

```
ts=2022-07-06T15:26:42.543Z caller=manager.go:974 level=error component="rule manager" msg="loading groups failed" err="/etc/prometheus/rules/prometheus-starburst-sb12-rulefiles-0/monitoring-stack-operator-starburst-sb12-rules-40c23e63-92a3-4f5c-bab0-8b628a0b240e.yaml: group \"starburst_alert_rules\", rule 7, \"JvmMemoryFillingUp\": annotation \"description\": template: __alert_JvmMemoryFillingUp:2: missing value for command
```