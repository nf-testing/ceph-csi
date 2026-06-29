ARG SRC_DIR="/go/src/github.com/ceph/ceph-csi/"
ARG BASE_IMAGE
ARG FINAL_BASE_IMAGE="quay.io/centos/centos:stream9-minimal"
ARG TARGETPLATFORM
ARG CSI_IMAGE_NAME
ARG CSI_IMAGE_VERSION

FROM --platform=${TARGETPLATFORM} ${BASE_IMAGE} as updated_base

ARG TARGETPLATFORM

# TODO: remove the following cmd, when issues
# https://github.com/ceph/ceph-container/issues/2034
# https://github.com/ceph/ceph-container/issues/2141 are fixed.
RUN true \
    && dnf -y reinstall https://download.ceph.com/rpm-${CEPH_VERSION}/el9/noarch/ceph-release-1-1.el9.noarch.rpm \
    && ( dnf config-manager --disable tcmu-runner,tcmu-runner-source,tcmu-runner-noarch,ceph-iscsi,ganesha || true ) \
    && true

RUN mkdir -p /etc/selinux && touch /etc/selinux/config

RUN dnf -y update --nobest \
       && dnf -y install nfs-utils \
       && dnf clean all \
       && rm -rf /var/cache/yum

FROM --platform=${TARGETPLATFORM} updated_base as builder

LABEL stage="build"

ARG CSI_IMAGE_NAME=quay.io/cephcsi/cephcsi
ARG CSI_IMAGE_VERSION=canary
ARG SRC_DIR
ARG GIT_COMMIT
ARG GO_BUILD_VERBOSE
ARG GOROOT=/usr/local/go
ARG TARGETPLATFORM

COPY build.env /

RUN source /build.env && \
    ( test -n "${TARGETPLATFORM}" && exit 0; echo -e "\n\nMissing TARGETPLATFORM argument for building image, run: make image-cephcsi PLATFORM=linux/amd64,linux/arm64\n\n"; exit 1 ) && \
    export GO_ARCH="${TARGETPLATFORM#linux/}" && \
    mkdir -p ${GOROOT} && \
    curl -L https://go.dev/dl/go${GOLANG_VERSION}.linux-${GO_ARCH}.tar.gz | tar xzf - -C ${GOROOT} --strip-components=1

# test if the downloaded version of Golang works (different arch?)
RUN ${GOROOT}/bin/go version && ${GOROOT}/bin/go env

# other/conflicting versions of protobuf get installed as dependency
RUN dnf -y remove protobuf

RUN dnf -y install --nodocs \
	librados-devel librbd-devel libcephfs-devel \
	/usr/bin/cc \
	make \
	git \
	&& dnf clean all \
	&& rm -rf /var/cache/yum \
    && true

ENV GOROOT=${GOROOT} \
    GOPATH=/go \
    CGO_ENABLED=1 \
    GIT_COMMIT="${GIT_COMMIT}" \
    ENV_CSI_IMAGE_VERSION="${CSI_IMAGE_VERSION}" \
    ENV_CSI_IMAGE_NAME="${CSI_IMAGE_NAME}" \
    PATH="${GOROOT}/bin:${GOPATH}/bin:${PATH}"


WORKDIR ${SRC_DIR}

# Copy source directories
COPY . ${SRC_DIR}

# Build executable
RUN make cephcsi GO_BUILD_VERBOSE="${GO_BUILD_VERBOSE}"

#-- Final container
FROM --platform=${TARGETPLATFORM} ${FINAL_BASE_IMAGE}

ARG SRC_DIR
ARG CEPH_VERSION
ARG TARGETPLATFORM
ARG CSI_IMAGE_VERSION

LABEL maintainers="Northflank + Ceph-CSI Authors" \
    version=${CSI_IMAGE_VERSION} \
    architecture=${TARGETPLATFORM#linux/} \
    description="Ceph-CSI Plugin"

COPY --from=builder ${SRC_DIR}/_output/cephcsi /usr/local/bin/cephcsi

RUN true \
  && rpm -ivh https://download.ceph.com/rpm-${CEPH_VERSION}/el9/noarch/ceph-release-1-1.el9.noarch.rpm \
  && microdnf -y install kmod nfs-utils nvme-cli epel-release psmisc e2fsprogs xfsprogs cryptsetup attr --setopt=install_weak_deps=0 --setopt=tsflags=nodocs \
  && microdnf -y install thrift librados2 librbd1 libcephfs2 ceph-common ceph-fuse rbd-nbd --setopt=install_weak_deps=0 --setopt=tsflags=nodocs \
  && microdnf clean all \
  && true

# verify that all dynamically linked libraries are available
RUN [ $(ldd /usr/local/bin/cephcsi | grep -c '=> not found') = '0' ]

ENTRYPOINT ["/usr/local/bin/cephcsi"]
