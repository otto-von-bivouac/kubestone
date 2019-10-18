BASEPATH   := config/openshift
CRDPATH    := config/crd/bases
DEPLOYPATH := deploy/openshift
KUSTPATH   := deploy/kustomize
RBACPATH   := config/rbac
SAMPLEPATH := config/samples

# Custom resource definitions
CRDS := $(foreach crd,$(wildcard $(CRDPATH)/*.yaml),$(DEPLOYPATH)/$(notdir $(crd)))

# Sample custom resources
SAMPLES := $(SAMPLEPATH)/perf_v1alpha1_drill.yaml
SAMPLES += $(SAMPLEPATH)/perf_v1alpha1_iperf3.yaml
SAMPLES += $(SAMPLEPATH)/perf_v1alpha1_pgbench.yaml

UNAME   := $(shell uname | tr A-Z a-z)
VERSION := $(shell git describe --tags --long --match 'v[0-9]*0' | awk -F'[.-]' '{print substr($$1,2) "." $$2 "." $$3 + $$4}')

YQ         := $(shell go env GOPATH)/bin/yq
YQ_VERSION := 2.4.0


.PHONY: openshift-csv
openshift-csv: version $(CRDS) $(KUSTPATH)/examples.patch $(KUSTPATH)/rules.patch $(DEPLOYPATH)/kubestone.package.yaml $(DEPLOYPATH)/kubestone.marketplace.yaml | $(KUSTPATH)
	cp $(BASEPATH)/kustomization.yaml $(KUSTPATH)/
	cp $(BASEPATH)/kubestone.clusterserviceversion.yaml $(KUSTPATH)/
	kustomize build $(KUSTPATH) | sed "s#\$$(VERSION)#$(VERSION)#g" >$(DEPLOYPATH)/kubestone.v$(VERSION).clusterserviceversion.yaml

.PHONY: yq
yq: $(YQ)
$(YQ):
	curl -sLo $(YQ) https://github.com/mikefarah/yq/releases/download/$(YQ_VERSION)/yq_$(UNAME)_amd64
	chmod +x $(YQ)

.PHONY: version
version:
	@echo "### Building CSV v$(VERSION)"

$(DEPLOYPATH) $(KUSTPATH):
	mkdir -p $@

$(CRDS): $(DEPLOYPATH)/%.yaml: $(CRDPATH)/%.yaml | $(DEPLOYPATH)
	cp $< $@

$(KUSTPATH)/examples.patch: $(BASEPATH)/examples.template $(SAMPLES) | $(KUSTPATH) yq
	set -e && for file in $(SAMPLES); do $(YQ) read $$file --tojson; done | jq -s "$$(cat $(BASEPATH)/examples.template)" | yq read - >$@

$(KUSTPATH)/rules.patch: $(BASEPATH)/rules.template $(RBACPATH)/role.yaml | $(KUSTPATH) yq
	set -e && yq read $(RBACPATH)/role.yaml --tojson | jq "$$(cat $(BASEPATH)/rules.template)" | yq read - >$@

$(DEPLOYPATH)/kubestone.marketplace.yaml: $(BASEPATH)/kubestone.marketplace.yaml | $(DEPLOYPATH)
	cp $(BASEPATH)/kubestone.marketplace.yaml $@

$(DEPLOYPATH)/kubestone.package.yaml: version $(BASEPATH)/kubestone.package.yaml | $(DEPLOYPATH)
	sed "s/\$$(VERSION)/$(VERSION)/g" $(BASEPATH)/kubestone.package.yaml >$@

