# Linux build (experimental)

Targets: x86_64 and aarch64 Linux with glibc. Build on the target architecture.
USB installation and log retrieval are not implemented on Linux; install the generated NSP
with DBI separately. Automatic app updates are disabled on Linux because the upstream
update feed and installer flow support Windows only.

## CachyOS / Arch Linux (x86_64)

Use a native pacman package. Building on CachyOS links against its current system libraries;
no Debian packages, FUSE or Wine on the host are needed. Keep the system fully updated:

```sh
sudo pacman -Syu --needed base-devel git rust nodejs npm docker docker-compose \
  webkit2gtk-4.1 gtk3 libayatana-appindicator openssl xdg-utils
sudo systemctl enable --now docker
```

The build user needs access to Docker. If needed, add your user to the docker group
(`sudo usermod -aG docker "$USER"`) and log out and in again. Docker group membership
grants root-equivalent access. Do not run npm or makepkg with sudo.
Node.js must be version 24 or newer; Rust must meet the workspace's minimum version.

```sh
git clone --branch feat/linux-packaging https://github.com/nongrips/carafe-Linux.git
cd carafe-Linux/packaging/cachyos
makepkg -si
carafe
```

`makepkg` fetches the port branch, builds the Switch runtime in Docker, compiles the
Linux desktop app on your host, and creates `carafe-git-*.pkg.tar.zst`. It installs a
launcher in the application menu and registers dependencies with pacman. Install an
already-built package using `sudo pacman -U ./carafe-git-*.pkg.tar.zst`.
To uninstall use `sudo pacman -Rns carafe-git`.

For development, run `npm ci` followed by `npm run dev:linux` from `app/`.
To compile without creating deb/rpm/AppImage bundles use `npm run build:cachyos`;
`npm run build:cachyos:app` reuses the runtime and helper already built.
Wayland and X11 are handled by GTK/WebKit. If a particular Wayland setup fails to
open a window, try `GDK_BACKEND=x11 carafe` with XWayland installed.

This package definition has not yet been built or launch-tested on CachyOS.
The Ubuntu CI artifacts do not validate the native Arch package.

## Other distributions

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
