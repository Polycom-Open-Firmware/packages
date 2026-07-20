# Smart speaker (C60)

Voice assistant on the C60's mic array and speaker. Backed by
`poly-c60-profile-smart-speaker` (audio baseline). No voice application
package exists: selecting this role boots the device to a serviceable
console (ssh enabled) with the audio stack configured, and enables
`poly-smart-speaker.service` when that package is installed.

C60-only — it needs the mic array and speaker hardware.

## Configuration reference

| Key | Values | Effect |
|---|---|---|
| `PROFILE` | `smart-speaker` | selects this role (wizard tile) |
| `VOLUME_MASTER` / `VOLUME_SPEAKER` | `0`–`100` | ALSA mixer levels |

Keys arrive via the config blob; full contract in `poly-firmware-build`'s
`CONFIG-PARTITION.md`.
