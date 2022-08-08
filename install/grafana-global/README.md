# Install script for Grafana 

Set of scripts to install Grafana operator, setup Grafana, provision datasources for Observatorium and demo versions of dashboards. 

By default it is assumed that Grafana operator will be deployed to the same cluster where test version of the Observatorium is installed. If Grafana is running on separate cluster `OBSERVATORIUM_APPS_URL` or `OBSERVATORIUM_ROUTE_HOST` variables needs to be defined to point to the right Observatorium

# Usage

Script will not connect to target Openshift cluster , make sure that you have established connection before using script, eg.
`oc login --token=sha256~xxxxxxxxxxxxx --server=https://api.sb12.gtzm.s1.devshift.org:6443`

## List of make targets

| Make Target | Description |
|-------------|-------------|
| make all| Will install Grafana operator, create Grafana CR and provision datasource|
|      | connecting to the default (local version) of the observatorium
| make clean| Will delete all objects including namespace where Grafana operator is installed|
| make create/dashboards| Will deploy dashboards (not part of the make all target) from config/grafana-dashboards location |
|  | script will itterate through all of files matching pattern *-dashboard.yaml in the directory and will create corresponding CRs |
| make create/datasource | will create new datasource (included in make all)|
| make OBSERVATORIUM_ROUTE_HOST=obsrvatorium.example.com \ | create datasource that points to the remote Observatorium |
|      OBSERVATORIUM_TOKEN=xxxxx create/datasource ||
|||

## Updating Grafana dashboard 

Please use `config/garafana-dashboards/starburst-ns-dashboard.yaml` as an example of the Dashboard CR format. Note that Grafana CR is using label selector to define which dashboards will be included so dashboard yaml file needs to include appropriate labels

```
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDashboard
metadata:
  labels:
    app: <name>
  name: starburst-ns-dashboard
  namespace: <namespace>
spec:
```
where `<name>` will be updated based on value of `GRAFANA_GLOBAL` variable which is by default `grafana-global`
