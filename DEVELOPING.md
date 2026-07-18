# Developing packages & device profiles

Developer guide for the OpenPolycom package/profile system: architecture,
implementation, and the cookbook for adding a new profile or app.

## Architecture in five sentences

Devices run a **sealed OS** (read-only rootfs behind a tmpfs overlay;
Debian bookworm arm64) whose *role* — web kiosk, media player, … — is a
**application**: a metapackage `poly-app-<id>` whose `Depends:`
pulls the apps and services for that role. Profiles are resolved at
**flash time**: the image builders produce one rootfs variant per profile
(`rootfs-<name>.simg`), and the wizard flashes the one the user picks.
All software is plain **Debian packages** from the OpenPolycom apt archive
(GPG-signed, hosted on Cloudflare R2), which the base image
trusts out of the box — so the same archive also serves live installs in
`tc8-rw` maintenance mode. Per-device settings (kiosk URL, media source…)
are **not** baked into images: they arrive as a config blob in the `cache`
partition, which the device consumes at the next boot — applied to the
real filesystem, then invalidated in place. The blob is a message, not a
store; on sealed boots the initramfs applies it before the overlay seals.
`/root` persists on its own partition across reboots *and* reflashes.

## Who lives where

| Repo | Role |
|---|---|
| `packages` (this repo) | `debian/` tree per package: all profile metapackages, `poly-archive-keyring`, `poly-*-base`, small tools |
| `apt` | **The only writer** to the published archive: stateless `publish.sh` (apt-ftparchive + GPG) + CI; pool/indexes live only in R2 |
| `poly-rootfs` | debootstrap base + pre-apt essentials (sources.list, keyring, users, fstab) + `--profile=` variant machinery + `--device=<tc8\|c60>` for the per-board profile package (one repo, both boards) |
| `poly-firmware-build` | image composer for both targets: `--target=<tc8\|c60>` builds the kernel, initramfs, `boot.img`, and packs `--os-profile=` variants (boota sparse for tc8, booti zstd for c60) |
| `provisioner` | the wizard: profile picker + per-profile settings pages + flashing |
| `chromium-a53` (and similar) | applications with their own upstream or build gravity live in their own repos; their debs enter the archive via the apt repo's publish paths |

Naming: `poly-` prefix everywhere. Applications are `poly-app-<id>` —
**board-agnostic** metapackages (`arch: all` unless they ship binaries);
an application that needs board hardware is restricted in the wizard's
catalog (`boards:` metadata on the Application entry), not by per-board
packages. The per-board `poly-<device>-profile-<name>` metapackages are
the legacy naming — the rootfs builder
installs `poly-app-<id>` when it exists and falls back to the per-board
name.

## How publishing works

```
packages repo push ──► CI builds all packages ──► repository_dispatch
                                                        │
                                     apt repo CI ◄──────┘
                              (single writer, queued)
                    pull pool from R2 → add debs → apt-ftparchive
                    → sign Release/InRelease (APT_GPG_KEY) → sync R2
```

Archive URL: `https://pub-1d222577af244182a265fc4d6a35b994.r2.dev`
(`stable`/`main`, archs `arm64 all`). Signing key: ed25519
`7A27D57B0045457E4C51A11EFAABA6E245033620` (secret `APT_GPG_KEY` on the
apt repo; an offline copy is held by the maintainers). Two publish
paths: **automatic** — CI sends the `repository_dispatch` above, which
requires the cross-repo `DISPATCH_TOKEN` secret; **manual** — build the
debs, then run the apt repo's `publish.sh` + wrangler sync (see its
README), trigger its `workflow_dispatch`, or drop the debs into its
`incoming/` inbox for a credential-free publish.

## How images consume profiles

`poly-rootfs/build.sh --profile=a,b` debootstraps the base **once**, then
per profile makes an isolated `rsync` clone, chroot-installs
`poly-tc8-profile-<name>` from the archive, stamps `TC8_PROFILE` into
`/etc/tc8-version`, and tars `rootfs-<name>.tar.gz`. The composer
(`poly-firmware-build/build.sh --profile=emmc --os-profile=a,b`) packs each
tarball into `rootfs-<name>.{img,simg}`; plain `rootfs.simg` always
aliases the default (`kiosk`) so existing tooling keeps working.
(`--profile` selects the emmc/nfs build target.)
Special profile `bare` = the untouched base. C60 size: the hard ceiling
is the 1.75 GiB `system_a` partition; the builder's default budget is
**1.6 GiB**, and it refuses — never truncates — oversized images.

## Cookbook: add a new profile

Say `poly-tc8-profile-signage`:

1. **Metapackage** — in this repo:
   ```
   mkdir -p poly-tc8-profile-signage/debian/source
   ```
   `debian/control`:
   ```
   Source: poly-tc8-profile-signage
   Section: metapackages
   Priority: optional
   Maintainer: OpenPolycom <apt@openpolycom.cc>
   Build-Depends: debhelper-compat (= 13)
   Standards-Version: 4.6.2

   Package: poly-tc8-profile-signage
   Architecture: all
   Depends: ${misc:Depends}, poly-app-yourthing, some-debian-pkg
   Description: TC8 device profile: digital signage
    One-line role description. What it boots into, what it depends on.
   ```
   Plus `debian/changelog` (start at `0.1.0`, suite `stable`),
   `debian/rules` (the 3-line dh boilerplate), `debian/source/format`
   (`3.0 (native)`). Copy an existing profile dir as the template.
2. **Services/config the profile needs** live in its *app* packages, not
   the metapackage — systemd units via `debian/<pkg>.install` +
   `dh_installsystemd`. Remember: package postinst runs at image build
   (in the chroot) or in maintenance mode — never against a sealed root,
   so normal Debian packaging just works.
3. **Test on a device without any publishing** (fastest loop):
   ```
   cd poly-tc8-profile-signage && dpkg-buildpackage -us -uc -b
   scp ../poly-tc8-profile-signage_*.deb root@<panel>:/tmp/
   # on the panel: tc8-rw && reboot; apt install /tmp/poly-…deb; test; tc8-ro && reboot
   ```
4. **Publish**: push to this repo (CI builds + dispatches), or the manual
   path above. Verify: `apt update && apt-cache policy poly-tc8-profile-signage`
   on any device.
5. **Image variant**:
   ```
   sudo ./build.sh --profile=emmc --os-profile=kiosk,signage
   # → out/emmc/rootfs-signage.simg
   ```
   Flash `rootfs-signage.simg` to `userdata` (+ the usual boot/dtbo/vbmeta
   if updating), boot, confirm the role comes up with zero manual steps.
6. **Config keys** (if the role has settings): add them to
   `poly-firmware-build/CONFIG-PARTITION.md` + implement in
   `rootfs/etc/tc8-config/apply-config.sh`. Handlers must be
   **idempotent** — every new blob re-runs every handler against the
   already-configured system.
7. **Wizard**: add the profile to the device's list in
   `provisioner/packages/core/src/profiles/<device>.ts` and give it a
   `SettingsSection` for its keys. Release assets are named
   `rootfs-<id>.simg`; plain `rootfs.simg` is the default-profile alias.

## Cookbook: add an app package

Same as a profile but `Architecture: arm64` if it ships binaries, real
content instead of bare `Depends:`. Cross-build tips live in
`chromium-a53` (the worked example of a heavy one — chroot cross-profile
builds, host-arch tool pitfalls). If your app has its own upstream or a
long build, give it its own repo and publish from there; this repo is for
the small stuff.

## Gotchas

- **NO NETWORK AT FIRST BOOT — bake everything in.** A flashed device
  routinely comes up with no internet (no DHCP, no uplink, air-gapped
  install). **Everything a profile needs must be in the baked image.** The
  archive + baked `sources.list` are for *build-time* install (in the
  chroot, on the build host's network) and for *maintenance-mode* installs
  (explicit `tc8-rw` + user-provided network) — NEVER for first boot. Hard
  rule: **no boot-path service or first-boot script may `apt`/`curl`/`wget`
  anything.** A profile's `Depends:` is satisfied at image build; if a role
  needs it at runtime, it's already on disk. (Corollary: no network means no
  NTP either — the clock starts from baked/persisted state, see fake-hwclock
  in `poly-firmware-build/docs/RO-ROOT.md`; don't gate boot on time sync.)
- **Sealed rootfs**: anything you `apt install` on a *sealed* device
  evaporates on reboot — great for testing, useless for keeps. Use
  `tc8-rw`/`tc8-ro` (see `poly-firmware-build/USING.md`).
- **C60 budget**: 1.6 GiB default image budget (hard ceiling: the
  1.75 GiB `system_a` partition). Check with the builder before
  publishing a C60 profile.
- **Key hygiene**: never commit the private key; the public keyring is the
  `poly-archive-keyring` package + baked into the bases.
- **Version stamps**: `TC8_OS_PROFILES` (built variants) in `version.env`;
  `TC8_PROFILE` (this image's role) in `/etc/tc8-version` on the device.
