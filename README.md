# OpenPolycom packages

Source monorepo for `poly-*` Debian packages: device-profile metapackages,
application metapackages, `poly-archive-keyring`, and small first-party
tools. One directory per package, standard `debian/` trees,
arch:all unless stated. CI builds everything on push and dispatches the
`apt` repo to publish. **Developer guide** — architecture, publishing
flow, and the add-a-profile/app cookbook: [DEVELOPING.md](DEVELOPING.md).

Packages with their own upstream or build gravity live in their own repos
(e.g. `chromium-a53`, `poly-kodi`) — this repo is for the small stuff.

**Application pages** — what each application does and its config options:
[kiosk](apps/kiosk.md) · [media player](apps/media-player.md) ·
[smart speaker](apps/smart-speaker.md).
