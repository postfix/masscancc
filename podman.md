podman run --rm -it -v "$PWD":/w -w /w alpine:3.20 sh -lc '
  set -eux

  # toolchain + build deps
  apk add --no-cache \
    build-base linux-headers git pkgconf curl \
    autoconf automake libtool \
    flex bison upx

  # build a static libpcap (1.10.5 includes security fixes)
  curl -L https://www.tcpdump.org/release/libpcap-1.10.5.tar.gz | tar xz
  cd libpcap-1.10.5
  CC=gcc \
  CFLAGS="-Os -pipe -ffunction-sections -fdata-sections -fno-unwind-tables -fno-asynchronous-unwind-tables" \
  LDFLAGS="-static -Wl,--gc-sections" \
  ./configure --disable-shared           # build only the static archive
  make -j"$(nproc)"
  make install                           # installs /usr/local/lib/libpcap.a and headers
  cd ..

  # fetch masscan and build it against the static libpcap
  git clone --depth=1 https://github.com/robertdavidgraham/masscan
  cd masscan
  make clean || true
  CC=gcc \
  CFLAGS="-Os -pipe -ffunction-sections -fdata-sections -fno-unwind-tables -fno-asynchronous-unwind-tables -DSTATICPCAP" \
  LDFLAGS="-static -Wl,--gc-sections" \
  LIBS="/usr/local/lib/libpcap.a" \
  make -j"$(nproc)"

  # slim it further
  strip -s bin/masscan
  upx --best --lzma bin/masscan || true

  # show result
  ls -lh bin/masscan
  file bin/masscan
  # ldd should say "not a dynamic executable" if fully static
  (ldd bin/masscan || true)
'
