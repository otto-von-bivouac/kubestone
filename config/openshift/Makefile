VERSION := 0.4.0
#VERSION := $(shell git describe --abbrev=0 --tags --match 'v*' | sed s/^v//)

CONFIG_RBAC      := config/rbac
CONFIG_SAMPLES   := config/samples
CONFIG_OPENSHIFT := config/openshift
CONFIG_CRD_BASES := config/crd/bases
TARGET_OPENSHIFT := target/openshift
TARGET_KUSTOMIZE := target/kustomize
TARGET_OPENSHIFT_VERSIONED := target/openshift/$(VERSION)

# Custom resource definitions
CRDS := $(foreach crd,$(wildcard $(CONFIG_CRD_BASES)/*.yaml),$(TARGET_OPENSHIFT_VERSIONED)/$(notdir $(crd)))

# Sample custom resources
SAMPLES := $(CONFIG_SAMPLES)/perf_v1alpha1_drill.yaml
SAMPLES += $(CONFIG_SAMPLES)/perf_v1alpha1_fio.yaml
SAMPLES += $(CONFIG_SAMPLES)/perf_v1alpha1_ioping.yaml
SAMPLES += $(CONFIG_SAMPLES)/perf_v1alpha1_iperf3.yaml
SAMPLES += $(CONFIG_SAMPLES)/perf_v1alpha1_pgbench.yaml
SAMPLES += $(CONFIG_SAMPLES)/perf_v1alpha1_qperf.yaml
SAMPLES += $(CONFIG_SAMPLES)/perf_v1alpha1_sysbench.yaml

UNAME   := $(shell uname | tr A-Z a-z)

YQ         := $(shell go env GOPATH)/bin/yq
YQ_VERSION := 2.4.0


.PHONY: openshift-csv
openshift-csv: version $(CRDS) $(TARGET_KUSTOMIZE)/alm-examples.patch $(TARGET_KUSTOMIZE)/install-permissions.patch $(TARGET_OPENSHIFT)/kubestone.package.yaml $(TARGET_OPENSHIFT)/kubestone.marketplace.yaml | $(TARGET_KUSTOMIZE)
	cp $(CONFIG_OPENSHIFT)/kustomization.yaml $(TARGET_KUSTOMIZE)/
	cp $(CONFIG_OPENSHIFT)/kubestone.clusterserviceversion.yaml $(TARGET_KUSTOMIZE)/
	kustomize build $(TARGET_KUSTOMIZE) | sed "s#\$$(VERSION)#$(VERSION)#g" >$(TARGET_OPENSHIFT_VERSIONED)/kubestone.$(VERSION).clusterserviceversion.yaml

.PHONY: yq
yq: $(YQ)
$(YQ):
	curl -sLo $(YQ) https://github.com/mikefarah/yq/releases/download/$(YQ_VERSION)/yq_$(UNAME)_amd64
	chmod +x $(YQ)

.PHONY: version
version:
	@echo "### Building CSV v$(VERSION)"

$(TARGET_OPENSHIFT) $(TARGET_KUSTOMIZE) $(TARGET_OPENSHIFT_VERSIONED):
	mkdir -p $@

$(CRDS): $(TARGET_OPENSHIFT_VERSIONED)/%.yaml: $(CONFIG_CRD_BASES)/%.yaml | $(TARGET_OPENSHIFT_VERSIONED)
	cp $< $@

$(TARGET_KUSTOMIZE)/alm-examples.patch: $(CONFIG_OPENSHIFT)/alm-examples.template $(SAMPLES) | $(TARGET_KUSTOMIZE) yq
	set -e && for file in $(SAMPLES); do $(YQ) read $$file --tojson; done | jq -s "$$(cat $(CONFIG_OPENSHIFT)/alm-examples.template)" | yq read - >$@

$(TARGET_KUSTOMIZE)/install-permissions.patch: $(CONFIG_OPENSHIFT)/install-permissions.template $(CONFIG_RBAC)/role.yaml | $(TARGET_KUSTOMIZE) yq
	set -e && yq read $(CONFIG_RBAC)/role.yaml --tojson | jq "$$(cat $(CONFIG_OPENSHIFT)/install-permissions.template)" | yq read - >$@

$(TARGET_OPENSHIFT)/kubestone.marketplace.yaml: $(CONFIG_OPENSHIFT)/kubestone.marketplace.yaml | $(TARGET_OPENSHIFT)
	cp $(CONFIG_OPENSHIFT)/kubestone.marketplace.yaml $@

$(TARGET_OPENSHIFT)/kubestone.package.yaml: version $(CONFIG_OPENSHIFT)/kubestone.package.yaml | $(TARGET_OPENSHIFT)
	sed "s/\$$(VERSION)/$(VERSION)/g" $(CONFIG_OPENSHIFT)/kubestone.package.yaml >$@

