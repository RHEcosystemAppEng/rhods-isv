SHELL=bash
export GRAFANA_GLOBAL?=grafana-global
export GRAFANA_GLOBAL_NAMESPACE?=grafana-global

export OIDC_IDP_URL?= dex-dex.$(OBSERVATORIUM_APPS_URL)
export OIDC_CLIENT_ID?=test
export OIDC_CLIENT_SECRET?="ZXhhbXBsZS1hcHAtc2VjcmV0"
export OIDC_ADMIN_USER?="admin@example.com"
export OIDC_ADMIN_PASS?="password"

export OBSERVATORIUM_APPS_URL ?= $(shell oc get --namespace openshift-ingress-operator ingresscontrollers/default --output jsonpath='{.status.domain}{"\n"}')
export OBSERVATORIUM_ROUTE_HOST ?=observatorium-observatorium.$(OBSERVATORIUM_APPS_URL)
export OBSERVATORIUM_TENANT?=test
export OBSERVATORIUM_TOKEN:=$(shell curl --request POST \
              --silent \
              -k \
              --url https://$(OIDC_IDP_URL)/dex/token  --header 'content-type: application/x-www-form-urlencoded' \
			  --data grant_type=password \
              --data username=$(OIDC_ADMIN_USER) --data password=$(OIDC_ADMIN_PASS) \
              --data client_id=$(OIDC_CLIENT_ID) --data client_secret=$(OIDC_CLIENT_SECRET) \
              --data scope="openid email" | sed 's/^{.*"id_token":[^"]*"\([^"]*\)".*}/\1/')

ifndef grafana_env
	export grafana_env :=dev
endif


.PHONY: create/namespace
create/namespace:
	@echo "create namespace: $(GRAFANA_GLOBAL_NAMESPACE)"
	@sed 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g' config/operator/bases/namespace.yaml | cat | oc apply -f -

.PHONY: create/operator
create/operator:
	@echo "Install grafana operator: $(GRAFANA_GLOBAL)"
	@oc kustomize config/operator/bases |sed 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<name>/$(GRAFANA_GLOBAL)/g' | oc apply -f -
	@ echo "Waiting for Grafana CRD to be created"
	@ for i in {1..12}; do oc get pods -l control-plane && break || sleep 5; done
#	Switch to Manual upgrade mode after operator install is done
	@oc patch sub grafana-operator -n $(GRAFANA_GLOBAL_NAMESPACE) --type=json -p='[{"op":"replace", "path":"/spec/installPlanApproval","value": "Manual"}]'

.PHONY: create/grafana
create/grafana:
	@echo "Creating Grafana $(GRAFANA_GLOBAL) in: $(GRAFANA_GLOBAL_NAMESPACE)"
	@oc kustomize config/grafana/bases |sed 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<name>/$(GRAFANA_GLOBAL)/g' | oc apply -f -

.PHONY: delete/grafana
delete/grafana:
	@echo "Delete Grafana $(GRAFANA_GLOBAL) in: $(GRAFANA_GLOBAL_NAMESPACE)"
	@oc kustomize config/grafana/bases |sed 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<name>/$(GRAFANA_GLOBAL)/g' | oc delete -f -

# Deploy datasource based on environment type
.PHONY: create/datasource
create/datasource:
ifeq ($(grafana_env),dev)
create/datasource: create/datasource/dev
endif

.PHONY: create/datasource/dev
create/datasource/dev:
	@echo "Create Datasource observatorium-metrics for  $(GRAFANA_GLOBAL)"
	@cat config/grafana-datasource/dev/external-credentials.yaml | sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<oauth_token>/$(shell echo -n $(OBSERVATORIUM_TOKEN)|base64 -w 0)/g' |oc apply -f -
	@cat config/grafana-datasource/dev/observatorium-metrics-datasource.yaml | sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<tenant_id>/$(OBSERVATORIUM_TENANT)/g' |oc apply -f -

.PHONY: delete/datasource/dev
delete/datasource/dev:
	@echo "Delete Datasource observatorium-metrics for  $(GRAFANA_GLOBAL)"
#	@cat config/grafana-datasource/dev/external-credentials.yaml | sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<oauth_token>/$(shell echo -n $(OBSERVATORIUM_TOKEN)|base64 -w 0)/g' |oc delete -f -
	@cat config/grafana-datasource/dev/observatorium-metrics-datasource.yaml | sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<observatorium_host>/$(OBSERVATORIUM_ROUTE_HOST)/g;s/<tenant_id>/$(OBSERVATORIUM_TENANT)/g' |oc delete -f -


.PHONY: create/dashboards
create/dashboards: ./config/grafana-dashboards/*-dashboard.yaml
	@echo "creating global kafka dashboards"
	@for file in $^ ; do \
		echo "Processing: " $${file} ; \
		cat $${file}| sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<name>/$(GRAFANA_GLOBAL)/g' | cat |oc apply -f -; \
	done
#    @cat ./config/dashboards/starburst-ns-dashboard.yaml | sed -e 's/<namespace>/$(GLOBAL_MONITORING_NAMESPACE)/g' | cat | oc apply -f -

.PHONY: delete/dashboards
delete/dashboards: ./config/grafana-dashboards/*-dashboard.yaml
	@echo "Deleting global kafka dashboards"
	@for file in $^ ; do \
		echo "Processing: " $${file} ; \
		cat $${file}| sed -e 's/<namespace>/$(GRAFANA_GLOBAL_NAMESPACE)/g;s/<name>/$(GRAFANA_GLOBAL)/g' | cat |oc delete -f - ; \
	done

all: create/namespace create/operator create/grafana create/datasource

clean: delete/grafana
	@echo "Uninstalling grafana $(GRAFANA_GLOBAL)"
	@oc delete namespace $(GRAFANA_GLOBAL_NAMESPACE)
	@echo "Done uninstalling $(GRAFANA_GLOBAL)"
