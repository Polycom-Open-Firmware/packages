# Developing packages & device profiles

Developer guide for the OpenPolycom package/profile system: architecture,
implementation, and the cookbook for adding a new profile or app.
Project-level plan and milestone status: `polycom_dev/PROFILES-PLAN.md`
(in the workspace, not this repo).

## Architecture in five sentences

Devices run a **sealed OS** (read-only rootfs behind a tmpfs overlay;
Debian bookworm arm64) whose *role* — web kiosk, media player, … — is a
**profile**: a metapackage `poly-<device>-profile-<name>` whose `Depends:`
pulls the apps and services for that role. Profiles are resolved at
**flash time**: the image builders produce one rootfs variant per profile
(`rootfs-<name>.simg`), and the wizard flashes the one the user picks.
All software is plain **Debian packages** from the OpenPolycom apt archive
(GPG-signed, hosted on Cloudflare R2, `$0` egress), which the base image
trusts out of the box — so the same archive also serves live installs in
`tc8-rw` maintenance mode. Per-device settings (kiosk URL, media source…)
are **not** baked into images: they ride the config blob in the `cache`
partition and are re-applied on every boot (which the ephemeral `/etc`
makes perfectly deterministic). `/root` persists on its own partition
across reboots *and* reflashes.

## Who lives where

| Repo | Role |
|---|---|
| `packages` (this repo) | `debian/` tree per package: all profile metapackages, `poly-archive-keyring`, `poly-*-base` (as rootfs content migrates), small tools |
| `apt` | **The only writer** to the published archive: stateless `publish.sh` (apt-ftparchive + GPG) + CI; pool/indexes live only in R2 |
| `tc8-rootfs` / `c60-rootfs` | debootstrap base + pre-apt essentials (sources.list, keyring, users, fstab) + `--profile=` variant machinery |
| `tc8-firmware-build` / `c60-…` | image composers: kernel, initramfs, `boot.img`, `--os-profile=` packing into flashable variants |
| `provisioner` | the wizard: profile picker (M4) + per-profile settings pages + flashing |
| `c60-kodi-portrait`, `chromium-a53` | packages with their own upstream/build gravity publish from their own repos |

Naming: `poly-` prefix everywhere; `poly-<device>-profile-<name>` for
profiles, `poly-app-<name>` for apps, `arch: all` unless it ships binaries.

## How publishing works

```
packages repo push ──► CI builds changed debs ──► repository_dispatch
                                                        │
                                     apt repo CI ◄──────┘
                              (single writer, queued)
                    pull pool from R2 → add debs → apt-ftparchive
                    → sign Release/InRelease (APT_GPG_KEY) → sync R2
```

Archive URL: `https://pub-1d222577af244182a265fc4d6a35b994.r2.dev`
(`stable`/`main`, archs `arm64 all`). Signing key: ed25519
`7A27D57B0045457E4C51A11EFAABA6E245033620` (secret `APT_GPG_KEY` on the
apt repo; offline copy with Alex). Until the cross-repo `DISPATCH_TOKEN`
is configured, publish manually: build the debs, then run the apt repo's
`publish.sh` + wrangler sync (see its README), or trigger its
`workflow_dispatch`.

## How images consume profiles

`tc8-rootfs/build.sh --profile=a,b` debootstraps the base **once**, then
per profile makes an isolated `rsync` clone, chroot-installs
`poly-tc8-profile-<name>` from the archive, stamps `TC8_PROFILE` into
`/etc/tc8-version`, and tars `rootfs-<name>.tar.gz`. The composer
(`tc8-firmware-build/build.sh --profile=emmc --os-profile=a,b`) packs each
tarball into `rootfs-<name>.{img,simg}`; plain `rootfs.simg` always
aliases the default (`kiosk`) so old tooling and the wizard keep working.
(`--os-profile` because `--profile` was already the emmc/nfs build target.)
Special profile `bare` = the untouched base. C60 hard limit: the image
must fit **1.6 GiB** — the builder refuses, never truncates.

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
   if updating), boot, confirm the role comes up with zero manual steps —
   that's the acceptance bar.
6. **Config keys** (if the role has settings): add them to
   `tc8-firmware-build/CONFIG-PARTITION.md` + implement in
   `rootfs/etc/tc8-config/apply-config.sh`. Settings must be re-appliable
   every boot (idempotent) — `/etc` is ephemeral by design.
7. **Wizard** (M4): add the profile to the device's list in
   `provisioner/packages/core/src/profiles/<device>.ts` and give it a
   `SettingsSection` for its keys. The wizard resolves the release asset
   by name: `rootfs-<id>.simg`, falling back to `rootfs.simg`.

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
  in RO-ROOT.md; don't gate boot on time sync.)
- **Sealed rootfs**: anything you `apt install` on a *sealed* device
  evaporates on reboot — great for testing, useless for keeps. Use
  `tc8-rw`/`tc8-ro` (see `tc8-firmware-build/USING.md`).
- **C60 budget**: 1.6 GiB image ceiling, non-negotiable. Check with the
  builder before publishing a C60 profile.
- **Key hygiene**: never commit the private key; the public keyring is the
  `poly-archive-keyring` package + baked into the bases.
- **Version stamps**: `TC8_OS_PROFILES` (built variants) in `version.env`;
  `TC8_PROFILE` (this image's role) in `/etc/tc8-version` on the device.
