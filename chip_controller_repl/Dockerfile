ARG BUILD_FROM
FROM $BUILD_FROM AS builder-base

# hadolint ignore=DL3009
RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       git \
       patch \
       gcc g++ \
       make \
       pkg-config

FROM builder-base AS builder-ttyd

ARG LIBWEBSOCKETS_VERSION
ARG TTYD_VERSION

COPY cmake-remove-duplicate-LwsCheckRequirements.patch /usr/src

WORKDIR /usr/src

RUN \
    set -x \
    && apt-get install -y --no-install-recommends \
       cmake \
       libuv1-dev \
       libc-dev \
       libssl-dev \
       zlib1g-dev \
       libjson-c-dev \
    && git clone --branch "v${LIBWEBSOCKETS_VERSION}" --depth=1 \
        https://github.com/warmcat/libwebsockets \
    && cd libwebsockets \
    && patch -p 1 < /usr/src/cmake-remove-duplicate-LwsCheckRequirements.patch \
    && mkdir -p build \
    && cd build \
    && cmake .. \
        -DCMAKE_BUILD_TYPE=MinSizeRel \
        -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DCMAKE_VERBOSE_MAKEFILE=TRUE \
        -DLWS_IPV6=ON \
        -DLWS_STATIC_PIC=ON \
        -DLWS_UNIX_SOCK=OFF \
        -DLWS_WITH_LIBUV=ON \
        -DLWS_WITH_SHARED=ON \
        -DLWS_WITHOUT_TESTAPPS=ON \
    && make \
    && make install \
    && make DESTDIR=./install install

RUN \
    cd /usr/src \
    && git clone --branch main --single-branch \
        https://github.com/tsl0922/ttyd \
    && cd ttyd \
    && git checkout "${TTYD_VERSION}" \
    && mkdir -p build \
    && cd build \
    && cmake .. \
        -DCMAKE_BUILD_TYPE=MinSizeRel \
        -DCMAKE_INSTALL_PREFIX=/usr/local \
        -DCMAKE_VERBOSE_MAKEFILE=TRUE \
    && make \
    && make DESTDIR=./install install

FROM builder-base AS builder-chip

ARG BUILD_ARCH=amd64
ARG CHIP_VERSION=700d92eeb695be291bbd2088abbff2ce044dd6ee

# Set shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

COPY 0001-Use-data-as-platform-storage-location.patch /usr/src
COPY 0002-Only-add-avahi-as-build-dependency-when-platform-mDN.patch /usr/src
COPY 0001-Avoid-python_runner.py-deadlock.patch /usr/src

WORKDIR /usr/src

# Install and build CHIP Python Controller
# hadolint ignore=SC1091
RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       protobuf-compiler \
       libprotobuf-dev \
       gn \
       libssl-dev \
       libdbus-1-dev \
       libglib2.0-dev \
       ninja-build \
       python3-venv \
       python3-dev \
       python3-pip \
       python3-dbus \
       python3-cryptography \
       unzip \
       libgirepository1.0-dev \
       libcairo2-dev \
    && git clone --depth 1 -b master \
       https://github.com/project-chip/connectedhomeip \
    && cd connectedhomeip \
    && git fetch origin ${CHIP_VERSION} \
    && git checkout ${CHIP_VERSION} \
    && git submodule update --init \
    && patch -p1 < /usr/src/0001-Use-data-as-platform-storage-location.patch \
    && patch -p1 < /usr/src/0002-Only-add-avahi-as-build-dependency-when-platform-mDN.patch \
    && ( \
           cd third_party/pigweed/repo/ \
           && patch -p1 < /usr/src/0001-Avoid-python_runner.py-deadlock.patch \
       ) \
    && source scripts/no_cipd_bootstrap.sh \
    && scripts/build_python.sh -m minimal

FROM $BUILD_FROM

RUN \
    set -x \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       libuv1 \
       openssl \
       zlib1g \
       libjson-c5 \
       python3-venv \
       python3-pip \
       python3-gi \
       python3-gi-cairo \
       python3-dbus \
       python3-psutil \
       unzip \
       libcairo2 \
       gdb \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /usr/src/*

COPY --from=builder-ttyd /usr/src/libwebsockets/build/install /
COPY --from=builder-ttyd /usr/src/ttyd/build/install /

RUN ldconfig

WORKDIR /root

COPY --from=builder-chip /usr/src/connectedhomeip/credentials /root/credentials
COPY --from=builder-chip \
     /usr/src/connectedhomeip/out/python_lib/controller/python/chip-0.0-cp37-abi3-linux_*.whl \
     /tmp

# hadolint ignore=DL3013
RUN \
    pip3 install --no-cache-dir /tmp/chip-0.0-cp37-abi3-linux_*.whl \
    && pip3 install --no-cache-dir jupyter-console \
    && rm /tmp/chip-0.0-cp37-abi3-linux_*.whl

COPY rootfs /
