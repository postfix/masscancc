# masscancc
Podman script to compile Masscan  as static binary

# Masscan — Static **musl** Build (Tiny Single Binary)

**Goal:** produce a **fully‑static** `musl`‑linked `masscan` binary that’s portable and small, without runtime `dlopen()` of `libpcap`.

* ✅ Links **statically** against `libpcap.a`
* ✅ Uses Masscan’s `-DSTATICPCAP` path (no runtime `dlopen()`)
* ✅ Builds on Alpine (musl by default) inside **Podman**
* ✅ Strips and (optionally) UPX‑packs for minimum size

> **Tip**: On Alpine you **don’t** need `musl-gcc`; `gcc` already targets musl.

---

## Quick Start (Podman on any host)

This will leave `./masscan/bin/masscan` on your host when it finishes.

```bash
podman run --rm -it -v "$PWD":/w -w /w alpine:3.20 sh -lc '
  set -eux

  # toolchain + build deps
  apk add --no-cache \
    build-base linux-headers git pkgconf curl \
    autoconf automake libtool \
    flex bison upx

  # 1) Build a static libpcap (installs /usr/local/lib/libpcap.a)
  curl -L https://www.tcpdump.org/release/libpcap-1.10.5.tar.gz | tar xz
  cd libpcap-1.10.5
  CC=gcc \
  CFLAGS="-Os -pipe -ffunction-sections -fdata-sections -fno-unwind-tables -fno-asynchronous-unwind-tables" \
  LDFLAGS="-static -Wl,--gc-sections" \
  ./configure --disable-shared
  make -j"$(nproc)"
  make install
  cd ..

  # 2) Build masscan fully-static against that libpcap.a
  git clone --depth=1 https://github.com/robertdavidgraham/masscan
  cd masscan
  make clean || true
  CC=gcc \
  CFLAGS="-Os -pipe -ffunction-sections -fdata-sections -fno-unwind-tables -fno-asynchronous-unwind-tables -DSTATICPCAP" \
  LDFLAGS="-static -Wl,--gc-sections" \
  LIBS="/usr/local/lib/libpcap.a" \
  make -j"$(nproc)"

  # 3) Slim it further
  strip -s bin/masscan
  upx --best --lzma bin/masscan || true   # optional

  # 4) Show result
  ls -lh bin/masscan
  file bin/masscan
'
```
### Embedded ARM (musl static) builds

Cross‑compile fully static musl binaries for ARM boards (routers/SBCs) using Zig (recommended for simplicity)
aarch64 (ARM64) , it links libpcap.a and define -DSTATICPCAP.

#### Zig toolchain 

```bash
podman run --rm -it -v "$PWD":/w -w /w alpine:3.20 sh -lc '
  set -eux

  apk add --no-cache build-base linux-headers git pkgconf curl \
                         autoconf automake libtool flex bison zig upx file

  # Sanity: zig + target probe + tiny test link for aarch64-musl
  zig version
  (zig targets | grep -E "aarch64-linux-musl" || true)
  printf "int main(){}" > t.c
  zig cc -target aarch64-linux-musl -static t.c -o /tmp/t
  file /tmp/t   # should say: ELF 64-bit LSB executable, aarch64, statically linked

  # 1) Build libpcap (static) for aarch64-musl into /opt/aarch64
  curl -L https://www.tcpdump.org/release/libpcap-1.10.5.tar.gz | tar xz
  cd libpcap-1.10.5
  CC="zig cc -target aarch64-linux-musl" AR="zig ar" RANLIB="zig ranlib" \
  CFLAGS="-Os -ffunction-sections -fdata-sections" \
  ./configure --host=aarch64-linux-musl --disable-shared --prefix=/opt/aarch64
  make -j"$(nproc)" && make install
  cd ..

  # 2) Build masscan (fully static) linked to that libpcap.a
  git clone --depth=1 https://github.com/robertdavidgraham/masscan
  cd masscan && make clean || true
  CC="zig cc -target aarch64-linux-musl" \
  CFLAGS="-Os -ffunction-sections -fdata-sections -fno-unwind-tables -fno-asynchronous-unwind-tables -DSTATICPCAP" \
  LDFLAGS="-static -Wl,--gc-sections" \
  LIBS="/opt/aarch64/lib/libpcap.a" \
  make -j"$(nproc)"

  strip -s bin/masscan
  upx --best --lzma bin/masscan || true
  file bin/masscan && ls -lh bin/masscan
'
```
#### armv7 (ARM32 hard-float, NEON, musl, fully static)
```bash
podman run --rm -it -v "$PWD":/w -w /w alpine:3.20 sh -lc '
  set -eux
  apk add --no-cache build-base linux-headers git pkgconf curl \
                         autoconf automake libtool flex bison zig upx file

  # Sanity: make sure zig has the target; prove it can link static for armv7
  zig version
  (zig targets | grep -E "arm-linux-musleabihf" || true)
  printf "int main(){}" > t.c
  zig cc -target arm-linux-musleabihf -static t.c -o /tmp/t
  file /tmp/t   # expect: ELF 32-bit LSB executable, ARM, EABI5, statically linked

  # 1) libpcap (static archive) for armv7 hard-float into /opt/armv7
  curl -L https://www.tcpdump.org/release/libpcap-1.10.5.tar.gz | tar xz
  cd libpcap-1.10.5
  CC="zig cc -target arm-linux-musleabihf" AR="zig ar" RANLIB="zig ranlib" \
  CFLAGS="-Os -ffunction-sections -fdata-sections -march=armv7-a -mfpu=neon -mfloat-abi=hard" \
  ./configure --host=arm-linux-musleabihf --disable-shared --prefix=/opt/armv7
  make -j"$(nproc)" && make install
  cd ..

  # 2) masscan (fully static) linked against that libpcap.a
  git clone --depth=1 https://github.com/robertdavidgraham/masscan
  cd masscan && make clean || true
  CC="zig cc -target arm-linux-musleabihf" \
  CFLAGS="-Os -ffunction-sections -fdata-sections -march=armv7-a -mfpu=neon -mfloat-abi=hard -fno-unwind-tables -fno-asynchronous-unwind-tables -DSTATICPCAP" \
  LDFLAGS="-static -Wl,--gc-sections" \
  LIBS="/opt/armv7/lib/libpcap.a" \
  make -j"$(nproc)"

  strip -s bin/masscan
  upx --best --lzma bin/masscan || true
  file bin/masscan && ls -lh bin/masscan
'
```

**Output path:** `./masscan/bin/masscan`

---

## Why a static `musl` build?

* **Portability:** No external shared libraries required at runtime.
* **Reliability:** Masscan’s default uses a runtime loader (`dlopen`) for `libpcap`; a fully static binary won’t load `.so` files at runtime, so we compile with `-DSTATICPCAP` and link `/usr/local/lib/libpcap.a` directly.
* **Small size:** `-Os` + section GC + `strip` (and optional UPX) keep the artifact tight.

---


#### Run as non‑root (optional, Linux):

```bash
sudo install -m755 masscan/bin/masscan /usr/local/bin/masscan
sudo setcap 'CAP_NET_RAW+ep CAP_NET_ADMIN+ep' /usr/local/bin/masscan
```

---

## Makefile knobs explained

* `CFLAGS` — size‑first flags; disables unwind tables to shave bytes.
* `LDFLAGS="-static -Wl,--gc-sections"` — link fully static and garbage‑collect unused sections.
* `-DSTATICPCAP` — tells masscan to **call** `pcap_*` directly (no runtime loader).
* `LIBS="/usr/local/lib/libpcap.a"` — use the static archive we built from source.

> If you prefer, swap in `-flto` (and add `AR="gcc-ar" RANLIB="gcc-ranlib"`) on both libpcap and masscan for a few more KB shaved via whole‑program optimization.

---

## Troubleshooting

**`configure: error: Neither flex nor lex was found.`**
Install `flex` (and also `bison`). They’re required to generate libpcap’s filter parser.

**`C compiler cannot create executables` during libpcap configure**
On Alpine, use `gcc` (musl‑native) instead of `musl-gcc`. Ensure `build-base` is installed.

**`libpcap-static (no such package)` on Alpine 3.20**
Alpine 3.20 doesn’t ship a `libpcap-static` package. Build libpcap from source as shown.

**Masscan prints `failed to load libpcap shared library`**
You either didn’t pass `-DSTATICPCAP` or aren’t linking `libpcap.a`. Rebuild with the flags above.

**PF\_RING/Lua messages**
Those optional backends are `dlopen()`-loaded in dynamic builds. They’re harmless if absent; this guide doesn’t include PF\_RING.

---

## Advanced: even smaller builds

* Add `-flto` to both libpcap and masscan; also set `AR=gcc-ar RANLIB=gcc-ranlib`.
* Disable optional libpcap backends (DBus, libnl, RDMA, OpenSSL capture, etc.) if `configure` finds them on your system. Keeping the environment minimal is usually enough on Alpine.
* UPX can shrink further; some security tools flag packed binaries—use with judgment.

---

---
## Usage (examples)

```bash
# Scan a single host for top common ports
./masscan -p80,443 203.0.113.10 --rate 10000

# Scan a /24 quickly
./masscan 198.51.100.0/24 -p 22,80,443 --rate 50000 --wait 0

# Export to JSON
./masscan 10.0.0.0/8 -p 80 --output-format json --output-filename out.json
```

> **Legal & ethical notice:** Only scan networks you own or have **explicit written authorization** to test. High‑rate scans can disrupt networks and trigger defenses.

---

## References & further reading

* Masscan repository — [https://github.com/robertdavidgraham/masscan](https://github.com/robertdavidgraham/masscan)
* libpcap releases — [https://www.tcpdump.org/#latest-releases](https://www.tcpdump.org/#latest-releases)
* musl libc basics — [https://wiki.musl-libc.org/getting-started.html](https://wiki.musl-libc.org/getting-started.html)
* Alpine and musl background — [https://www.reddit.com/r/AlpineLinux/comments/mfjj5s/explanation\_alpine\_linux\_is\_built\_around\_musl/](https://www.reddit.com/r/AlpineLinux/comments/mfjj5s/explanation_alpine_linux_is_built_around_musl/)
* Runtime loader context (`dlopen` + libpcap) — Masscan source tree (`rawsock-pcap.c`/\`stub-pcap.c>)

---

### Credits

* Robert David Graham and contributors for Masscan
* The tcpdump/libpcap maintainers

