VERSION 0.6
FROM alpine

ARG KUBEADM_VERSION=latest
ARG BASE_IMAGE=quay.io/kairos/core-opensuse:latest
ARG IMAGE_REPOSITORY=quay.io/kairos

ARG CRICTL_VERSION=1.25.0
ARG RELEASE_VERSION=0.4.0

ARG LUET_VERSION=0.33.0
ARG GOLINT_VERSION=v1.50.1
ARG GOLANG_VERSION=1.19.2

ARG KUBEADM_VERSION=latest
ARG BASE_IMAGE_NAME=$(echo $BASE_IMAGE | grep -o [^/]*: | rev | cut -c2- | rev)
ARG BASE_IMAGE_TAG=$(echo $BASE_IMAGE | grep -o :.* | cut -c2-)
ARG KUBEADM_VERSION_TAG=$(echo $KUBEADM_VERSION | sed s/+/-/)
ARG FIPS_ENABLED=false
ARG PROVIDER_IMAGE_NAME=kubeadm

build-cosign:
    FROM gcr.io/projectsigstore/cosign:v1.13.1
    SAVE ARTIFACT /ko-app/cosign cosign

go-deps:
    FROM golang:$GOLANG_VERSION
    WORKDIR /build
    COPY go.mod go.sum ./
    RUN go mod download
    RUN apt-get update && apt-get install -y upx
    SAVE ARTIFACT go.mod AS LOCAL go.mod
    SAVE ARTIFACT go.sum AS LOCAL go.sum

BUILD_GOLANG:
    COMMAND
    WORKDIR /build
    COPY . ./
    ARG BIN
    ARG SRC

    ENV CGO_ENABLED=0

    IF $FIPS_ENABLED
        ENV GOEXPERIMENT=boringcrypto
    END

    RUN go build -ldflags "-s -w" -o ${BIN} ./${SRC} && upx ${BIN}
    SAVE ARTIFACT ${BIN} ${BIN} AS LOCAL build/${BIN}

VERSION:
    COMMAND
    FROM alpine
    RUN apk add git

    COPY . ./

    RUN echo $(git describe --exact-match --tags || echo "v0.0.0-$(git log --oneline -n 1 | cut -d" " -f1)") > VERSION

    SAVE ARTIFACT VERSION VERSION

build-provider:
    FROM +go-deps
    DO +BUILD_GOLANG --BIN=agent-provider-kubeadm --SRC=main.go

lint:
    FROM golang:$GOLANG_VERSION
    RUN wget -O- -nv https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s v1.50.1
    WORKDIR /build
    COPY . .
    RUN golangci-lint run --timeout=3m

DOWNLOAD_BINARIES:
    COMMAND
    IF $FIPS_ENABLED
        RUN curl -L "https://storage.googleapis.com/spectro-fips/cri-tools-${CRICTL_VERSION}.tar.gz" | sudo tar -C /usr/bin/ -xz
        RUN curl -L --remote-name-all https://storage.googleapis.com/spectro-fips/${KUBEADM_VERSION}/kubeadm
        RUN curl -L --remote-name-all https://storage.googleapis.com/spectro-fips/${KUBEADM_VERSION}/kubelet
        RUN curl -L --remote-name-all https://storage.googleapis.com/spectro-fips/${KUBEADM_VERSION}/kubectl
    ELSE
        RUN curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/v${CRICTL_VERSION}/crictl-v${CRICTL_VERSION}-linux-amd64.tar.gz" | sudo tar -C /usr/bin/ -xz
        RUN curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/v${KUBEADM_VERSION}/bin/linux/amd64/kubeadm
        RUN curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/v${KUBEADM_VERSION}/bin/linux/amd64/kubelet
        RUN curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/v${KUBEADM_VERSION}/bin/linux/amd64/kubectl
    END

SETUP_CONTAINERD:
    COMMAND
    RUN mkdir -p /opt/cni/bin

    IF $FIPS_ENABLED
        RUN curl -sSL https://storage.googleapis.com/spectro-fips/containerd-1.6.4.tar.gz | sudo tar -C /opt/ -xz
        RUN curl -SL -o runc https://storage.googleapis.com/spectro-fips/runc-1.1.4/runc
        RUN curl -sSL https://storage.googleapis.com/spectro-fips/cni-plugins-1.1.1.tar.gz | sudo tar -C /opt/cni/bin/ -xz
    ELSE
        RUN curl -sSL https://github.com/containerd/containerd/releases/download/v1.6.4/containerd-1.6.4-linux-amd64.tar.gz | sudo tar -C /opt/ -xz
        RUN curl -SL -o runc "https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64"
        RUN curl -sSL https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz | sudo tar -C /opt/cni/bin/ -xz
    END

    RUN install -m 755 runc /opt/bin/runc
    RUN curl -sSL "https://raw.githubusercontent.com/containerd/containerd/main/containerd.service" | sed "s?ExecStart=/usr/local/bin/containerd?ExecStart=/opt/bin/containerd?" | sudo tee /etc/systemd/system/containerd.service

SAVE_IMAGE:
    COMMAND
    ARG VERSION
    IF $FIPS_ENABLED
        SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-${PROVIDER_IMAGE_NAME}:${KUBEADM_VERSION_TAG}_fips
        SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-${PROVIDER_IMAGE_NAME}:${KUBEADM_VERSION_TAG}_fips_${VERSION}
    ELSE
        SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-${PROVIDER_IMAGE_NAME}:${KUBEADM_VERSION_TAG}
        SAVE IMAGE --push $IMAGE_REPOSITORY/${BASE_IMAGE_NAME}-${PROVIDER_IMAGE_NAME}:${KUBEADM_VERSION_TAG}_${VERSION}
    END

docker:
    DO +VERSION
    ARG VERSION=$(cat VERSION)

    FROM $BASE_IMAGE

    WORKDIR /usr/bin

    DO +DOWNLOAD_BINARIES

    RUN chmod +x kubeadm
    RUN chmod +x kubelet
    RUN chmod +x kubectl

    RUN curl -sSL "https://raw.githubusercontent.com/kubernetes/release/v${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sudo tee /etc/systemd/system/kubelet.service
    RUN mkdir -p /etc/systemd/system/kubelet.service.d
    RUN curl -sSL "https://raw.githubusercontent.com/kubernetes/release/v${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

    WORKDIR /

    DO +SETUP_CONTAINERD

    COPY containerd/config.toml /etc/containerd/config.toml

    COPY upgrade/upgrade.sh /opt/kubeadm/upgrade.sh

    RUN echo "overlay" >> /etc/modules-load.d/k8s.conf
    RUN echo "br_netfilter" >> /etc/modules-load.d/k8s.conf

    RUN echo net.bridge.bridge-nf-call-iptables=1 >> /etc/sysctl.d/k8s.conf
    RUN echo net.bridge.bridge-nf-call-ip6tables=1 >> /etc/sysctl.d/k8s.conf
    RUN echo net.ipv4.ip_forward=1 >> /etc/sysctl.d/k8s.conf

    COPY +build-provider/agent-provider-kubeadm /system/providers/agent-provider-kubeadm

    DO +SAVE_IMAGE --VERSION=$VERSION

cosign:
    ARG --required ACTIONS_ID_TOKEN_REQUEST_TOKEN
    ARG --required ACTIONS_ID_TOKEN_REQUEST_URL

    ARG --required REGISTRY
    ARG --required REGISTRY_USER
    ARG --required REGISTRY_PASSWORD

    DO +VERSION
    ARG VERSION=$(cat VERSION)

    FROM docker

    ENV ACTIONS_ID_TOKEN_REQUEST_TOKEN=${ACTIONS_ID_TOKEN_REQUEST_TOKEN}
    ENV ACTIONS_ID_TOKEN_REQUEST_URL=${ACTIONS_ID_TOKEN_REQUEST_URL}

    ENV REGISTRY=${REGISTRY}
    ENV REGISTRY_USER=${REGISTRY_USER}
    ENV REGISTRY_PASSWORD=${REGISTRY_PASSWORD}

    ENV COSIGN_EXPERIMENTAL=1
    COPY +build-cosign/cosign /usr/local/bin/

    RUN echo $REGISTRY_PASSWORD | docker login -u $REGISTRY_USER --password-stdin $REGISTRY

    DO +SAVE_IMAGE --VERSION=$VERSION

docker-all-platforms:
     BUILD --platform=linux/amd64 +docker

cosign-all-platforms:
     BUILD --platform=linux/amd64 +cosign
