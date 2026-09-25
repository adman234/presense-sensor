# CLAUDE.md

ESPHome remote package for Akamatis LD2410 presence sensors (ESP32-C3). Read
[README.md](README.md) before changing anything.

- `presence-sensor.yaml` is the shared body. Sensors pull it via `packages:`
  (`url` / `ref: main` / `files`). `example-device.yaml` is the per-sensor
  template.
- **Never commit real device files, room names, MAC suffixes or entity IDs.**
  The repo is public. The user's device files live in their ESPHome config
  directory.
- Anything pushed to `main` reaches every sensor on its next flash. Validate
  first:
  1. Point a scratch device file at the local body with `!include C:/...`.
  2. Put a dummy `secrets.yaml` next to it.
  3. Run `esphome config`, and `esphome compile` when a lambda changes.

  On Windows, compile from PowerShell (not Git Bash), from a short path like
  `C:\ev`.
- Required substitutions: `name`, `friendly_name`. Optional ones have defaults
  in the body. Keep `example-device.yaml` and the README table in sync with it.
- `name_add_mac_suffix` is `false`. Changing `name` or that flag breaks HA
  entity IDs: flag it loudly and never do it incidentally.
- Keep behaviour changes separate from structural ones, and describe them in
  the commit message. The user flashes real hardware.
