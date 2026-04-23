# syntax=docker/dockerfile:1

# Stage 1: Build BambuStudio from source for aarch64
FROM ubuntu:24.04 AS builder

ARG BAMBUSTUDIO_VERSION=v02.06.00.51
ENV DEBIAN_FRONTEND=noninteractive

RUN \
  echo "**** install build dependencies ****" && \
  apt-get update && \
  apt-get install -y --no-install-recommends \
    autoconf \
    automake \
    build-essential \
    cmake \
    curl \
    git \
    libblosc-dev \
    libbz2-dev \
    libcurl4-openssl-dev \
    libdbus-1-dev \
    libegl-dev \
    libexpat1-dev \
    libfreetype-dev \
    libgl-dev \
    libglfw3-dev \
    libglu1-mesa-dev \
    libgstreamer-plugins-base1.0-dev \
    libgstreamer1.0-dev \
    libgtk-3-dev \
    libjpeg-dev \
    liblzma-dev \
    libnotify-dev \
    libosmesa6-dev \
    libpng-dev \
    libsecret-1-dev \
    libssl-dev \
    libtiff-dev \
    libtool \
    libudev-dev \
    libwayland-dev \
    libwebkit2gtk-4.1-dev \
    libx11-dev \
    libxcursor-dev \
    libxext-dev \
    libxi-dev \
    libxinerama-dev \
    libxkbcommon-dev \
    libxrandr-dev \
    libxxf86vm-dev \
    libwxgtk3.2-dev \
    libwxgtk-webview3.2-dev \
    libzstd-dev \
    m4 \
    ninja-build \
    patchelf \
    perl \
    pkg-config \
    python3 \
    python3-pip \
    wget \
    zlib1g-dev \
    libboost-all-dev \
    gstreamer1.0-plugins-base && \
  rm -rf /var/lib/apt/lists/*

# Copy patches into the builder image
COPY patches/ /patches/

# Clone BambuStudio at the pinned release tag
RUN git clone --depth 1 --branch ${BAMBUSTUDIO_VERSION} \
      https://github.com/bambulab/BambuStudio.git /build/BambuStudio

# Apply patches for aarch64 compatibility
WORKDIR /build/BambuStudio
RUN \
  cp /patches/TBB/GNU.cmake deps/TBB/GNU.cmake && \
  git apply /patches/0001-ENH-Allow-using-system-Blosc.patch && \
  git apply /patches/gmp-fixes.patch && \
  git apply /patches/0001-Apply-TBB-LTO-patch.patch

# Build BambuStudio bundled dependencies
# Disable deps that are already installed from apt to avoid patch/build failures
RUN \
  cmake -S deps -B /build/deps-build \
    -GNinja \
    -DCMAKE_BUILD_TYPE=Release \
    -DDESTDIR=/build/deps-install \
    -DDEP_BUILD_BLOSC=OFF \
    -DDEP_BUILD_BOOST=OFF \
    -DDEP_BUILD_FREETYPE=OFF \
    -DDEP_BUILD_GLFW=OFF \
    -DDEP_BUILD_JPEG=OFF \
    -DDEP_BUILD_OPENSSL=OFF \
    -DDEP_BUILD_PNG=OFF \
    -DDEP_BUILD_TIFF=OFF \
    -DDEP_BUILD_WXWIDGETS=OFF && \
  cmake --build /build/deps-build

# Build and install BambuStudio
RUN \
  CXXFLAGS="-std=gnu++20" \
  cmake -S . -B /build/bambu-build \
    -GNinja \
    -DCMAKE_INSTALL_PREFIX=/opt/bambustudio \
    -DCMAKE_INSTALL_LIBDIR=/opt/bambustudio/lib \
    -DCMAKE_PREFIX_PATH=/build/deps-install/usr/local \
    -DFLATPAK=ON \
    -DSLIC3R_PCH=OFF \
    -DSLIC3R_FHS=ON \
    -DSLIC3R_GTK=3 \
    -DSLIC3R_STATIC=ON \
    -DSLIC3R_BUILD_TESTS=OFF \
    -DSLIC3R_DESKTOP_INTEGRATION=OFF \
    -DCMAKE_BUILD_TYPE=Release \
    -DBBL_RELEASE_TO_PUBLIC=1 \
    -DBBL_INTERNAL_TESTING=0 \
    -DCMAKE_POLICY_VERSION_MINIMUM=3.5 && \
  cmake --build /build/bambu-build --target install

# Stage 2: Runtime image based on LSIO selkies desktop streaming base
FROM ghcr.io/linuxserver/baseimage-selkies:ubuntunoble

ARG BUILD_DATE
ARG VERSION
ARG BAMBUSTUDIO_VERSION

LABEL build_version="arm64 version:- ${VERSION} Build-date:- ${BUILD_DATE}"
LABEL maintainer="wheetazlab"

ENV TITLE=BambuStudio \
    SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt \
    NO_GAMEPAD=true \
    PIXELFLUX_WAYLAND=true

RUN \
  echo "**** add icon ****" && \
  curl -o \
    /usr/share/selkies/www/icon.png \
    https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/bambustudio-logo.png && \
  echo "**** install runtime packages ****" && \
  add-apt-repository ppa:xtradeb/apps && \
  apt-get update && \
  DEBIAN_FRONTEND=noninteractive \
  apt-get install --no-install-recommends -y \
    firefox \
    fonts-dejavu \
    fonts-dejavu-extra \
    gir1.2-gst-plugins-bad-1.0 \
    gir1.2-gstreamer-1.0 \
    gstreamer1.0-nice \
    gstreamer1.0-plugins-* \
    gstreamer1.0-pulseaudio \
    libblosc1 \
    libcurl4 \
    libglfw3 \
    libglu1-mesa \
    libnotify4 \
    libosmesa6 \
    libsecret-1-0 \
    libwebkit2gtk-4.1-0 \
    libwx-perl \
    libwxgtk-webview3.2-1t64 \
    libwxgtk3.2-1t64 && \
  printf "arm64 version: ${VERSION}\nBuild-date: ${BUILD_DATE}" > /build_version && \
  echo "**** cleanup ****" && \
  apt-get autoclean && \
  rm -rf \
    /config/.cache \
    /config/.launchpadlib \
    /var/lib/apt/lists/* \
    /var/tmp/* \
    /tmp/*

COPY --from=builder /opt/bambustudio /opt/bambustudio

# add local files
COPY root/ /

# ports and volumes
EXPOSE 3001
VOLUME /config
