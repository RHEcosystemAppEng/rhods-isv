# Install and configure in-cluster components for ISV monitoring

Set of scripts to help provisioning all components in dev environment required for collection of metrics from target ISV application running in the Openshift cluster. Script will install and configure following components:

1. Monitoring Stack Operator (renamed as Observability Operator) CatalogSource and Subcription OLM objects -   `make create/operator` target
2. Install Observability operator  with configuration specific to monitoring ISV application in specific namespace  - `make create/mso` target. The same make target will also setup prometheus rules for alerting, servicemonitor specifying what to monitor , servicemonitor object to enable federation of certain metrics from embedded Openshift monitoring facility.  Target also will make configuration changes to enable metrics to be sent to the centralized monitoring facility - Observatorium. 
3. *Optional*: will patch StarBurst operator to enable additional metrics to be collected  from Starburst application - `make update/starburst`

Scripts are based on K8s kustomize tool (https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) and capable of provisioning in-cluster monitoring stack for different environments (dev, staging)

# Usage

Script will not connect to target Openshift cluster , make sure that you have established connection before using script, eg.
`oc login --token=sha256~xxxxxxxxxxxxx --server=https://api.sb12.gtzm.s1.devshift.org:6443`

```
make all - will install MSO operator and all components
make clean - will delete all components but not the MSO operator and will clean all temporary files (due to the way kustomize tool applies patch script will create x.generated.x files that should not be checked-in to GitHub repository)
make update/starburst - Optional target specific to the Starburst ISV , meant to add additional metrics that don't come by default)
make create/mso - will configure objects needed for collection of metrics but not Operator itself (In cases when MSO operator is already installed to the target cluster)
```
While script defines default values for the environment variables , depending on target environment some of them may need to be overridden . Most common customizations: 

`MONITORED_NAMESPACE - namespace with application that needs to be monitored`

Example : To configure in-cluster monitoring stack for the Starburst application installed in starburst-sb12 use following command : 

`make MONITORED_NAMESPACE=starburst-sb12 create/mso`
or to update Starburst application 
`make MONITORED_NAMESPACE=starburst-sb12 update/starburst`

Additional arguments that changing flow of the script 

`print_only=true - applicable only to targets - make create/mso and make create/operator .Will print full set of objects to be created in yaml format without applying them to cluster (useful for  documentation)`
`mso_env=dev - type of environment that mostly affects setup of target Observatorium. By default it will be dev environment`

Example: 
Print all object definitions that will be created for application deployed in starburst-sb12 without applying them to the cluster

`make MONITORED_NAMESPACE=starburst-sb12 print_only=true create/mso`

For more details on environment variables that can be overriden see "Environment Variables" section in `Makefile`

**NOTE:** As script generates temporary files during the execution sometimes it is useful to delete these files before committing to git repository . Use target `make clean/files`  to achieve that . This command won't make any changes to the cluster itself 

To configure Observability operator to use Staging RHOBS environment specify `mso_env=stage` as argument, ex.

`make MONITORED_NAMESPACE=starburst-sb12 OIDC_CLIENT_SECRET=xxxxxxxxx create/mso mso_env=stage`

# Reduce usage of OCP resources for StarBurst 

Default Starburst instance  sets limits on resources to facilitate reasonable production workload. These settings however could be two  to high for dev or testing deployements. To reduce resource usage use following command: 

`oc patch StarburstEnterprise/starburstenterprise-sample -n redhat-starburst-operator --type merge --patch-file config/starburst/starburst_dev_size.yaml`