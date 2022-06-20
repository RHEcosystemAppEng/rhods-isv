# Environment variables 
# ---------------------------
SHELL=bash
export OCP_CLUSTER_ID?= $(shell oc get clusterversion -o jsonpath='{.items[].spec.clusterID}{"\n"}')
#export OBSERVATORIUM_APPS_URL:=$(shell oc get --namespace openshift-ingress-operator ingresscontrollers/default --output jsonpath='{.status.domain}{"\n"}')
export OBSERVATORIUM_APPS_URL?=apps.mmikhail-obs1.kni.syseng.devcluster.openshift.com
export OBSERVATORIUM_ROUTE_HOST?= observatorium-observatorium.$(OBSERVATORIUM_APPS_URL)
export MONITORED_NAMESPACE?=starburst-sb12
export MSO_NAMESPACE?=monitoring-stack-operator
export OBSERVATORIUM_TENANT?=test
export MONITORED_ISV_APP?=starburstenterprise-sample

ifndef mso_env
	export mso_env :=dev
endif

export OIDC_IDP_URL?= dex-dex.$(OBSERVATORIUM_APPS_URL)
export OIDC_CLIENT_ID?=test
export OIDC_CLIENT_SECRET?="ZXhhbXBsZS1hcHAtc2VjcmV0"
ifeq ($(mso_env),dev)
export OIDC_ADMIN_USER?="admin@example.com"
export OIDC_ADMIN_PASS?="password"
export OBSERVATORIUM_TOKEN:=$(shell curl --request POST \
              --silent \
              -k \
              --url https://$(OIDC_IDP_URL)/dex/token  --header 'content-type: application/x-www-form-urlencoded' \
			  --data grant_type=password \
              --data username=$(OIDC_ADMIN_USER) --data password=$(OIDC_ADMIN_PASS) \
              --data client_id=$(OIDC_CLIENT_ID) --data client_secret=$(OIDC_CLIENT_SECRET) \
              --data scope="openid email" | sed 's/^{.*"id_token":[^"]*"\([^"]*\)".*}/\1/')
endif
ifeq ($(mso_env),stage)
export OIDC_CLIENT_ID:=observatorium-rhods-isv-staging
#export OIDC_CLIENT_SECRET:=
export OBSERVATORIUM_ROUTE_HOST:=observatorium-mst.api.stage.openshift.com
export OBSERVATORIUM_TENANT:=rhods
endif

#---------------------------

.PHONY: create/mso
create/mso:
ifeq ($(mso_env),dev)
create/mso: create/mso/dev
endif
ifeq ($(mso_env),stage)
create/mso: create/mso/stage
endif


# Deploy MSO stack for ISV monitoring 
.PHONY: create/mso/dev
create/mso/dev: 
	@echo "Create MonitoringStack for  $(MONITORED_NAMESPACE) in Dev environment"
	@cat config/mso/bases/monitoringstack.yaml| sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;;s/<cluster_id>/$(OCP_CLUSTER_ID)/g' > config/mso/bases/monitoringstack.generated.yaml
	@cat config/mso/dev/patch-remotewrite.yaml| sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<bearer_token>/$(OBSERVATORIUM_TOKEN)/g;s/<tenant_id>/$(OBSERVATORIUM_TENANT)/g' > config/mso/dev/patch-remotewrite.generated.yaml
    ifeq (${print_only},true)
	@oc kustomize config/mso/dev |sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<bearer_token>/$(OBSERVATORIUM_TOKEN)/g;s/<cluster_id>/$(OCP_CLUSTER_ID)/g'
    else
	@oc kustomize config/mso/dev |sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<bearer_token>/$(OBSERVATORIUM_TOKEN)/g;s/<cluster_id>/$(OCP_CLUSTER_ID)/g' |oc apply -f -
    endif

.PHONY: create/mso/stage
create/mso/stage: 
	@echo "Create MonitoringStack for  $(MONITORED_NAMESPACE) with Staging Observatorium"
	@cat config/mso/bases/monitoringstack.yaml| sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;;s/<cluster_id>/$(OCP_CLUSTER_ID)/g' > config/mso/bases/monitoringstack.generated.yaml
	@cat config/mso/stage/patch-remotewrite.yaml| sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<tenant_id>/$(OBSERVATORIUM_TENANT)/g' > config/mso/stage/patch-remotewrite.generated.yaml
	@cat config/mso/stage/env.conf |sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<oidc_client_id>/$(OIDC_CLIENT_ID)/g;s/<oidc_client_secret>/$(OIDC_CLIENT_SECRET)/g;s/<cluster_id>/$(OCP_CLUSTER_ID)/g' > config/mso/stage/env.generated.conf
    ifeq (${print_only},true)
	@oc kustomize config/mso/stage |sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<oidc_client_id>/$(OIDC_CLIENT_ID)/g;s/<oidc_client_secret>/$(OIDC_CLIENT_SECRET)/g;s/<cluster_id>/$(OCP_CLUSTER_ID)/g'
    else
	@oc kustomize config/mso/stage |sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<oidc_client_id>/$(OIDC_CLIENT_ID)/g;s/<oidc_client_secret>/$(OIDC_CLIENT_SECRET)/g;s/<cluster_id>/$(OCP_CLUSTER_ID)/g' |oc apply -f -
    endif

# Provision MSO operator
.PHONY: create/operator
create/operator:
#Kustomize doesn't work well in this case due to timming of creation of catalogsource vs subscription
	@echo "Create MSO operator in: $(MSO_NAMESPACE)"
	@oc apply -f config/operator/bases/catalogsource.yaml
	@for i in {1..12}; do oc get pod -l olm.catalogSource=monitoring-operators -n openshift-marketplace && break || sleep 5; done
	@oc apply -f config/operator/bases/subscription.yaml
	@for i in {1..12}; do oc get pod -l app.kubernetes.io/name=monitoring-stack-operator -n openshift-operators && break || sleep 5; done
	@oc apply -f config/operator/bases/namespace.yaml


.PHONY: update/starburst
update/starburst:
	@echo "Enable Starburst metrics in $(MONITORED_NAMESPACE) namespace"
	@oc patch StarburstEnterprise/$(MONITORED_ISV_APP) -n $(MONITORED_NAMESPACE) --type merge --patch-file config/starburst/starburst.yaml


.PHONY: clean/operator
clean/operator: 
	@echo "Delete MSO operator subscription and catalogsource in: $(MSO_NAMESPACE)"
	@oc delete -f config/operator/bases/subscription.yaml
	@oc delete -f config/operator/bases/catalogsource.yaml

.PHONY: clean/mso
clean/mso: 
	@echo "Cleaning  MonitoringStack objects created for monitoring of $(MONITORED_NAMESPACE)"
	@cat config/mso/bases/monitoringstack.yaml| sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;;s/<cluster_id>/$(OCP_CLUSTER_ID)/g' > config/mso/bases/monitoringstack.generated.yaml
	@oc kustomize config/mso/bases |sed 's/<monitored_namespace>/$(MONITORED_NAMESPACE)/g;s/<cluster_id>/$(OCP_CLUSTER_ID)/g' |oc delete -f -

.PHONY: clean/files
clean/files:
	@find ./config -type f -name "*.generated.*" -exec rm {} \;

clean: clean/mso clean/files


all: create/operator create/mso
	@echo "Done installing ISV monitoring stack"