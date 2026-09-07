# secureblue-niri-dms-os &nbsp; [![bluebuild build badge](https://github.com/raff4el/secureblue-niri-dms-os/actions/workflows/build.yml/badge.svg)](https://github.com/raff4el/secureblue-niri-dms-os/actions/workflows/build.yml)

A [BlueBuild](https://blue-build.org/) image: [secureblue](https://secureblue.dev)'s
hardened Fedora Atomic base, with GNOME's session swapped for
[niri](https://github.com/YaLTeR/niri) + [DankMaterialShell](https://danklinux.com)
and [greetd](https://github.com/kennylevinsen/greetd)/dms-greeter as the login screen.

## Security trade-offs

This image intentionally departs from stock secureblue in ways worth knowing about
before you rebase to it:

- **Screencopy is not restricted.** secureblue only ships images for desktops that
  secure privileged Wayland protocols (GNOME, KDE, Sway, COSMIC). Under niri, any
  application that speaks `wlr-screencopy` can capture the whole desktop, including
  other applications' windows. This is the main security property traded away by
  swapping out GNOME. *Accepted trade-off.*
- **Homebrew is installed and self-updates daily.** That is an additional package
  manager pulling unsigned prebuilt bottles, outside rpm-ostree and outside the
  image's signing chain. Some brew binaries also misbehave under `hardened_malloc`.
  *Accepted trade-off.*
- **Three third-party COPRs are used at build time** (`atim/starship`,
  `avengemedia/dms`, `avengemedia/danklinux`). They are removed from the finished
  image (`cleanup: true`), so they are not left behind as trusted package sources
  for runtime `rpm-ostree install`.
- **Xwayland is not installed.** `xwayland-satellite` is a weak dependency of niri
  and is deliberately excluded, so X11-only applications will not start. Add it to
  `recipes/recipe.yml` if you need them.
- **Weak dependencies are disabled** for the package set, so niri does not silently
  drag in waybar/fuzzel/swaylock (redundant with DMS) or Xwayland. Anything actually
  needed is listed explicitly.

Optional extra hardening, not enabled by default because it costs convenience:
set `greeterRememberLastUser` to `false` in
`files/system/usr/share/dms-greeter-theme/settings.json` so the login screen stops
displaying the last username.

## Installation

> [!WARNING]
> [This is an experimental feature](https://www.fedoraproject.org/wiki/Changes/OstreeNativeContainerStable), try at your own discretion.

To rebase an existing atomic Fedora installation to the latest build:

- First rebase to the unsigned image, to get the proper signing keys and policies installed:
  ```
  rpm-ostree rebase ostree-unverified-registry:ghcr.io/raff4el/secureblue-niri-dms-os:latest
  ```
- Reboot to complete the rebase:
  ```
  systemctl reboot
  ```
- Then rebase to the signed image, like so:
  ```
  rpm-ostree rebase ostree-image-signed:docker://ghcr.io/raff4el/secureblue-niri-dms-os:latest
  ```
- Reboot again to complete the installation
  ```
  systemctl reboot
  ```

The `latest` tag will automatically point to the latest build. That build will still use the Fedora version specified in `recipe.yml`, so you won't get accidentally updated to the next major version.

## Customising

### Your niri config

`/etc/niri/config.kdl` holds the system defaults. Two ways to change them:

- **Add to them:** put your bindings in `~/.config/niri/local.kdl`. The system
  config includes it (optionally) at the end, so it is merged on top.
- **Replace them:** create `~/.config/niri/config.kdl`. niri prefers it and the
  system file is ignored entirely.

### The login screen

dms-greeter reads `settings.json` and `session.json` from `/var/cache/dms-greeter`.
This image populates them from `files/system/usr/share/dms-greeter-theme/` via
`tmpfiles.d`, using `C+` so changes here reach already-installed systems on the next
boot. The flip side is that `dms-greeter sync` results get overwritten every boot —
if you would rather drive the greeter from a running system, change `C+` back to `C`
in `files/system/usr/lib/tmpfiles.d/dms-greeter.conf`.

By default the greeter shows a solid black background. To use a wallpaper, drop a
JPEG at `files/system/usr/share/dms-greeter-theme/wallpaper.jpg`, uncomment the
override line in `files/system/usr/lib/tmpfiles.d/dms-greeter.conf`, and set
`greeterWallpaperPath` in `settings.json` to any non-empty string. Note that
dms-greeter only ever loads the override image from
`/var/cache/dms-greeter/greeter_wallpaper_override.jpg` — `greeterWallpaperPath` acts
as an enable flag, not as the path that is read.

Weather is disabled on the greeter, so it does not make network requests before
anyone has logged in.

### Bumping the Fedora release

`image-version` in `recipes/recipe.yml` and every `chroot: fedora-NN-x86_64` under
the dnf module must move together. The COPRs cannot infer the release from the
secureblue base image, which is why the chroots are pinned by hand.

## Maintenance notes

`renovate.json` tracks the base image tag and the COPR chroots, but only takes
effect if the [Renovate GitHub App](https://github.com/apps/renovate) is installed
on the repository. Dependabot handles the GitHub Actions versions.

## ISO

If built on Fedora Atomic, you can generate an offline ISO with the instructions available [here](https://blue-build.org/how-to/generate-iso/#_top). These ISOs cannot unfortunately be distributed on GitHub for free due to large sizes, so for public projects something else has to be used for hosting.

## Verification

These images are signed with [Sigstore](https://www.sigstore.dev/)'s [cosign](https://github.com/sigstore/cosign). You can verify the signature by downloading the `cosign.pub` file from this repo and running the following command:

```bash
cosign verify --key cosign.pub ghcr.io/raff4el/secureblue-niri-dms-os
```
