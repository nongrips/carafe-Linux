# Linux build (experimental)

Targets: x86_64 and aarch64 Linux with glibc. Build on the target architecture.
USB installation and log retrieval are not implemented on Linux; install the generated NSP
with DBI separately. Automatic app updates are disabled on Linux because the upstream
update feed and installer flow support Windows only.

On Ubuntu 24.04 / Debian install the desktop build dependencies:

```sh
sudo apt-get update
sudo apt-get install build-essential curl git file libssl-dev pkg-config \
  libwebkit2gtk-4.1-dev libayatana-appindicator3-dev librsvg2-dev patchelf
```

Install Node.js 24+, current stable Rust (at least 1.90), and Docker with the Compose plugin.
Your user must be able to run Docker. For AppImage execution, distributions may additionally
require FUSE 2 (`libfuse2t64` on Ubuntu 24.04).

```sh
git clone https://github.com/nongrips/carafe-Linux.git
cd carafe-Linux/app
npm ci
npm run build:linux
```

The first build compiles the Switch runtime in Docker and can take a long time.
The native hacBrewPack helper is built in a separate Debian container; Wine on the host is
not required. Packages are written under `app/target/release/bundle/` (deb, rpm and AppImage).
For development use `npm run dev:linux`. To rebuild only the desktop app after the runtime
and helper exist, use `npm run build:linux:app`.

The `Linux build` GitHub Actions workflow can also be started manually. Download its
`carafe-linux-x86_64` artifact after a successful run. This is an unsigned experimental build,
not an upstream release. A successful package build still needs a desktop launch test and
an NSP build test with your own game and keys.
