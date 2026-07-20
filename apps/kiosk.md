# Kiosk

A locked fullscreen browser: the panel boots straight into one web page —
a dashboard, calendar, camera feed, Home Assistant, any URL. The default
application. Backed by `poly-app-kiosk` (weston + cog); the chromium
engine ships in `poly-app-chromium`.

The stack is weston (kiosk-shell) with the browser as its only client,
started by `kiosk-launch` from `kiosk.service`.

## Configuration reference

| Key | Values | Effect |
|---|---|---|
| `PROFILE` | `kiosk` | selects this application (wizard tile) |
| `KIOSK_URL` | URL | the page the panel displays |
| `KIOSK_URL_FALLBACK` | URL | stored for a fallback-on-unreachable behavior that is not implemented — the key is accepted and persisted, nothing reads it |
| `KIOSK_ENGINE` | `webkit` \| `chromium` | browser engine: cog (lightweight, default) or Chromium (full Chrome engine; needs `poly-app-chromium` baked) |
| `COG_OPTS` | flags | extra cog options |

Keys arrive via the config blob (the wizard's Configure step); full
contract in `poly-firmware-build`'s `CONFIG-PARTITION.md`. If a selected
engine's package isn't baked, `kiosk-launch` falls back to cog with a
logged warning — the kiosk never fails to start over a missing engine.
