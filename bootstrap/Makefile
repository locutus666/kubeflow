# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
GCLOUD_PROJECT ?= kubeflow-images-public
GOLANG_VERSION ?= 1.11.5
GOPATH ?= $(HOME)/go
GOOGLE_APPLICATION_CREDENTIALS ?= $(HOME)/auth.json
# To build without the cache set the environment variable
# export DOCKER_BUILD_OPTS=--no-cache
IMG ?= gcr.io/$(GCLOUD_PROJECT)/bootstrapper
TAG ?= $(shell git describe --tags --always --dirty)
PORT ?= 2345
PLUGINS_ENVIRONMENT ?= $(GOPATH)/src/github.com/kubeflow/kubeflow/bootstrap/bin
export GO111MODULE = on
export GO = go

all: build

auth:
	gcloud auth configure-docker

# Run go fmt against code
fmt:
	$(GO) fmt ./pkg/... ./cmd/... ./config/...

# Run go vet against code
vet:
	$(GO) vet ./pkg/... ./cmd/... ./config/...

generate:
	$(GO) generate ./pkg/... ./cmd/... ./config/...
  
deepcopy:
	@deepcopy-gen -i github.com/kubeflow/kubeflow/bootstrap/pkg/apis/... -O zz_generated.deepcopy

build-bootstrap: generate fmt vet
	$(GO) build -gcflags 'all=-N -l' -o bin/bootstrapper cmd/bootstrap/main.go

build-kfctl: generate fmt vet
	$(GO) build -i -gcflags 'all=-N -l' -o bin/kfctl cmd/kfctl/main.go

build-dockerfordesktop-plugin: generate fmt vet
	$(GO) build -i -gcflags 'all=-N -l' -o bin/dockerfordesktop.so -buildmode=plugin cmd/plugins/dockerfordesktop/dockerfordesktop.go

build-local: build-bootstrap build-kfctl

# To edit which registries to add to bootstrapper, edit config (eg. config/default.yaml)
build:
	rm -rf reg_tmp
	mkdir -p reg_tmp/kubeflow
	cp -r ../kubeflow reg_tmp/kubeflow
	cp -r ../deployment reg_tmp/kubeflow
	cp -r ../dependencies reg_tmp/kubeflow
	docker build \
		--build-arg GOLANG_VERSION=$(GOLANG_VERSION) \
		--build-arg registries=reg_tmp \
		--target=bootstrap \
		--tag $(IMG):$(TAG) .
	@echo Built $(IMG):$(TAG)

# Build but don't attach the latest tag. This allows manual testing/inspection of the image
# first.
push: build
	docker push $(IMG):$(TAG)
	@echo Pushed $(IMG):$(TAG)

push-latest: push
	gcloud container images add-tag --quiet $(IMG):$(TAG) $(IMG):latest --verbosity=info
	echo created $(IMG):latest

install: build-kfctl
	@echo copying bin/kfctl to /usr/local/bin
	@cp bin/kfctl /usr/local/bin

run-local-docker:
	docker run -d -it --name bootstrapper \
	  --mount type=bind,source=${HOME}/kf_app,target=/home/kubeflow \
	  --entrypoint /bin/bash $(IMG):$(TAG)

# static and plugins toogle whether we try and load a platform as a .so.
# There is an example plugin 'dockerfordesktop' that can be loaded as a .so.
# By default we disable loading plugins and link dockerfordesktop into the kfctl binary (static)
static:
	@ex pkg/apis/apps/group.go <hack/static
	@ex cmd/kfctl/cmd/root.go <hack/static || echo Plugin functionality disabled

plugins:
	@ex pkg/apis/apps/group.go <hack/plugins
	@ex cmd/kfctl/cmd/root.go <hack/plugins || echo Plugin functionality enabled

clean:
	@rm -rf test
      
test-known-platforms-init: clean install build-dockerfordesktop-plugin
	@mkdir test && \
	echo kfctl init test/dockerfordesktop -V --platform docker-for-desktop && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) PLUGINS_ENVIRONMENT=$(PLUGINS_ENVIRONMENT) kfctl init `pwd`/test/dockerfordesktop -V --platform docker-for-desktop && \
	echo kfctl init test/none -V && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) kfctl init `pwd`/test/none -V && \
	echo kfctl init test/minikube -V --platform minikube && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) kfctl init $(PWD)/test/minikube -V --platform minikube && \
	echo kfctl init test/gcp -V --platform gcp --project $(GCLOUD_PROJECT) && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) kfctl init $(PWD)/test/gcp -V --platform gcp --project $(GCLOUD_PROJECT) && \
	echo SUCCESS

test-known-platforms-generate: test-known-platforms-init
	@cd test/dockerfordesktop && \
	echo 'kfctl generate all (--platform dockerfordesktop)' && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) PLUGINS_ENVIRONMENT=$(PLUGINS_ENVIRONMENT) kfctl generate all && \
	cd ../none && \
	echo 'kfctl generate all (--platform none)' && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) kfctl generate all && \
	cd ../minikube && \
	echo 'kfctl generate all (--platform minikube)' && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) kfctl generate all --mount-local && \
	cd ../gcp && \
	echo 'kfctl generate all (--platform gcp)' && \
	GOOGLE_APPLICATION_CREDENTIALS=$(GOOGLE_APPLICATION_CREDENTIALS) kfctl generate all --email jsmith@acme.com && \
	echo SUCCESS

test-plugins-static: plugins test-known-platforms-generate static test-known-platforms-generate

test-kfctl: test-plugins-static
