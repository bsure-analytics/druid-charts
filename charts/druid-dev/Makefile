SHELL := /bin/bash

CHART_VERSION ?= $(shell yq '.version' ../druid/Chart.yaml)
HELM_NAMESPACE ?= $(NAMESPACE)
HELM_OPTS ?= $(OPTS)
HELM_RELEASE ?= $(RELEASE)
NAMESPACE ?= druid
RELEASE ?= $(NAMESPACE)

.DEFAULT_GOAL := upgrade

.PHONY: dependency-update init
dependency-update init:
	CHART_VERSION="$(CHART_VERSION)" \
		yq -i \
		'.version = strenv(CHART_VERSION) | (.dependencies[] | select(.name == "druid")).version = "^" + strenv(CHART_VERSION)' \
		Chart.yaml
	helm dependency update

.PHONY: diff
diff:
	helm diff upgrade $(HELM_RELEASE) . \
		--context 3 \
		--namespace $(HELM_NAMESPACE) \
		$(HELM_OPTS)

.PHONY: template
template:
	helm template $(HELM_RELEASE) . \
		--namespace $(HELM_NAMESPACE) \
		$(HELM_OPTS)

.PHONY: test
test:
	helm test $(HELM_RELEASE) \
		--namespace $(HELM_NAMESPACE) \
		$(HELM_OPTS)

.PHONY: uninstall down
uninstall down:
	helm uninstall $(HELM_RELEASE) \
		--namespace $(HELM_NAMESPACE) \
		$(HELM_OPTS)

.PHONY: upgrade up
upgrade up:
	helm upgrade $(HELM_RELEASE) . \
		--create-namespace \
		--install \
		--namespace $(HELM_NAMESPACE) \
		$(HELM_OPTS)
