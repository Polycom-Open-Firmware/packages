# Media player

Kodi media center, fullscreen. The panel becomes a touch media device:
video, music, and photos from its own storage or a network share.
Backed by `poly-app-kodi` (per-board profile packages select it); on the
C60 the portrait UI comes from `poly-kodi-skin-c60portrait`.

Kodi runs as the kiosk stack's Wayland client (`KIOSK_ENGINE=kodi` —
same weston compositor as the web kiosk, different client). On the C60
the skin package routes ALSA to the board's codec via `/etc/asound.conf`;
the TC8 uses its ALSA defaults.

## Modes (`MEDIA_MODE`)

- **`full`** (default) — the whole Kodi UI: library, file browsing,
  playback controls, settings.
- **`photoframe`** — light on UI: the panel boots straight into a
  fullscreen **pictures** slideshow, no interaction needed. Implemented
  by `poly-photoframe.service`, which waits for Kodi's JSON-RPC and
  starts the slideshow (repeat-all) from `/persist/media/photos`, or
  `MEDIA_SOURCE` if set. The source is pictures-only by design: a
  slideshow over a mixed tree (photos + video + music) makes Kodi spawn
  a video player and a folder-picker alongside the pictures.

## Media sources

- **Local, always present:** `/persist/media` — on the persistent
  partition, so content survives reboots *and* reinstalls. It's split
  into `photos/`, `music/`, and `video/` subdirs; the photo-frame mode
  slideshows `photos/`. On the TC8 the panel's USB data port exposes
  this storage as an MTP "Portable Device" for drag-and-drop — drop
  pictures in `photos/` to see them in the frame.
- **Network (`MEDIA_SOURCE`):** any Kodi-supported path — `smb://`,
  `nfs://`, `http://`. Added to Kodi's video/music/pictures sources
  alongside the local one.

## Configuration reference

| Key | Values | Effect |
|---|---|---|
| `PROFILE` | `media-player` | selects this application (wizard tile) |
| `MEDIA_MODE` | `full` \| `photoframe` | see Modes above |
| `MEDIA_SOURCE` | Kodi path | network media source (optional) |
| `VOLUME_MASTER` / `VOLUME_SPEAKER` | `0`–`100` | ALSA mixer levels |

Keys arrive via the config blob (the wizard's Configure step); the full
contract is `poly-firmware-build`'s `CONFIG-PARTITION.md`. Kodi's own
per-panel state (library, added sources, settings) lives in the
persistent Kodi home and survives reflashes.

## Behavior notes

- Kodi's web server is enabled (port 8080, no auth) for remote control
  and screenshots; it binds to all device interfaces, so keep the device
  on trusted networks.
- With `poly-app-kodi` not baked into the image, the kiosk falls back to
  the web kiosk (cog) rather than failing to boot.
