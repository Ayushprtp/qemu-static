#!/bin/bash

# Exit on any error
set -e

# Set base directory to the script's directory
BASE_DIR="$(cd "$(dirname "$0")" && pwd)"
QEMU_VERSION="10.0.0"
QEMU_SRC="qemu-$QEMU_VERSION"
BUILD_DIR="$BASE_DIR/build"
INSTALL_DIR="$BASE_DIR/qemu-static"
STATIC_DEPS="$BASE_DIR/static-install"
VENV_DIR="$BASE_DIR/.venv"

# Colors
GREEN="\033[1;32m"
RED="\033[1;31m"
RESET="\033[0m"

# Print step
step() {
  echo -e "${GREEN}▶ $1${RESET}"
}

step "Creating directories..."
mkdir -p "$BUILD_DIR" "$STATIC_DEPS" "$INSTALL_DIR"
cd "$BASE_DIR"

step "Installing minimal system dependencies..."
sudo apt-get update
sudo apt-get install -y build-essential wget curl git pkg-config \
  python3 python3-venv cmake autoconf automake libtool bison flex \
  gettext libxml2-utils

step "Creating Python virtual environment for meson/ninja..."
python3 -m venv "$VENV_DIR"
source "$VENV_DIR/bin/activate"
pip install --upgrade pip setuptools
pip install meson ninja

export PKG_CONFIG_PATH="$STATIC_DEPS/lib/pkgconfig:$STATIC_DEPS/lib/x86_64-linux-gnu/pkgconfig"
export CFLAGS="-I$STATIC_DEPS/include"
export LDFLAGS="-L$STATIC_DEPS/lib"

build_zlib() {
  step "Building static zlib..."
  cd "$BUILD_DIR"
  wget -nc https://zlib.net/zlib-1.3.1.tar.gz
  tar -xf zlib-1.3.1.tar.gz && cd zlib-1.3.1
  ./configure --static --prefix="$STATIC_DEPS"
  make -j$(nproc) && make install
}

build_pixman() {
  step "Building static pixman..."
  cd "$BUILD_DIR"
  wget -nc https://cairographics.org/releases/pixman-0.43.4.tar.gz
  tar -xf pixman-0.43.4.tar.gz
  cd pixman-0.43.4
  meson setup build --prefix="$STATIC_DEPS" -Ddefault_library=static
  ninja -C build && ninja -C build install
}

build_glib() {
  step "Building static glib..."
  cd "$BUILD_DIR"
  wget -nc https://download.gnome.org/sources/glib/2.78/glib-2.78.4.tar.xz
  tar -xf glib-2.78.4.tar.xz && cd glib-2.78.4
  meson setup build --prefix="$STATIC_DEPS" -Ddefault_library=static -Dlibmount=disabled -Dselinux=disabled
  ninja -C build && ninja -C build install
}

build_libfdt() {
  step "Building static libfdt with Meson..."
  cd "$BUILD_DIR"
  rm -rf dtc
  git clone https://github.com/dgibson/dtc.git
  cd dtc
  meson setup builddir --prefix="$STATIC_DEPS" -Ddefault_library=static
  meson compile -C builddir
  meson install -C builddir
}

build_slirp() {
  step "Building static libslirp..."
  cd "$BUILD_DIR"
  rm -rf libslirp
  git clone https://gitlab.freedesktop.org/slirp/libslirp.git
  cd libslirp
  meson setup build --prefix="$STATIC_DEPS" -Ddefault_library=static
  ninja -C build && ninja -C build install
}

build_sdl2() {
  step "Building static SDL2..."
  cd "$BUILD_DIR"
  wget -nc https://github.com/libsdl-org/SDL/releases/download/release-2.30.2/SDL2-2.30.2.tar.gz
  tar -xf SDL2-2.30.2.tar.gz
  cd SDL2-2.30.2
  ./configure --disable-shared --enable-static --prefix="$STATIC_DEPS"
  make -j$(nproc)
  make install
}

build_qemu() {
  step "Downloading QEMU $QEMU_VERSION..."
  cd "$BASE_DIR"
  if [ ! -d "$QEMU_SRC" ]; then
    wget https://download.qemu.org/$QEMU_SRC.tar.xz
    tar -xf $QEMU_SRC.tar.xz
  fi

  step "Configuring QEMU..."
  cd "$QEMU_SRC"
  rm -rf build && mkdir build && cd build
  if ! ../configure \
  --prefix="$INSTALL_DIR" \
  --target-list=x86_64-softmmu,aarch64-softmmu \
  --static --enable-tools --disable-docs \
  --enable-kvm --enable-slirp --enable-fdt \
  --enable-sdl --disable-gtk --disable-opengl \
  --disable-vnc \
  --disable-spice --disable-virglrenderer --disable-curl \
  --disable-gnutls --disable-nettle --disable-gcrypt \
  --disable-auth-pam --disable-brlapi --disable-vhost-net \
  --disable-xen --disable-libnfs --disable-libssh --disable-libiscsi \
  --disable-glusterfs --disable-vte --disable-virtfs \
  --disable-slirp-smbd --disable-capstone --disable-alsa \
  --disable-pa --disable-sndio --disable-mpath \
  --disable-libudev; then
    echo "QEMU configure script missing or failed."
    exit 1
  fi

  step "Building QEMU..."
  make -j$(nproc)
  make install

  step "✅ QEMU $QEMU_VERSION built and installed in $INSTALL_DIR"
}

build_zlib
build_pixman
build_glib
build_libfdt
build_slirp
build_sdl2
build_qemu

