# OpenPolycom packages

Source monorepo for `op-*` Debian packages: device-profile metapackages,
the archive keyring, and (as content migrates out of the rootfs repos)
`op-{tc8,c60}-base`. One directory per package, standard `debian/` trees,
arch:all unless stated. CI builds everything on push and dispatches the
`apt` repo to publish. **Developer guide** — architecture, publishing
flow, and the add-a-profile/app cookbook: [DEVELOPING.md](DEVELOPING.md).
Roadmap: `polycom_dev/PROFILES-PLAN.md`.

Packages with their own upstream or build gravity live in their own repos
(`c60-kodi-portrait`, `chromium-a53`) — this repo is for the small stuff.
