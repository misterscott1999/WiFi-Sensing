# Wi-Fi Sensing Home Occupancy: Rules for Claude

## Who this is for
Chris Scott. He is new to coding: explain decisions briefly, prefer simple, maintainable Python.
Separate project from the Job Search repo; never mix files between them.

## What this is
3x ESP32-S3 (N16R8: 16 MB flash, 8 MB PSRAM) boards running RuView CSI firmware stream to the
RuView `sensing-server` on this Windows PC. Our Python code (occupancy/) reads its API, fuses
the 3 nodes, logs to SQLite, and shows a Streamlit dashboard. Later: Home Assistant + Nest.
Full plan: docs/plan.md.

## Pinned versions (do not upgrade casually; RuView ships CI builds daily)
- Firmware: RuView v0.8.8-esp32, **`s3-4mb` bundle** (firmware/v0.8.8-esp32/s3-4mb/, SHA256-verified).
  Not the 8mb bundle: it compiles display support in, and on display-less DevKitC boards the panel
  probe false-positives and CSI drops to 0 pps. The 4mb build is headless and runs fine on 16 MB flash.
  Flash: `esptool --chip esp32s3 --port COMx --baud 460800 write-flash 0x0 bootloader.bin
  0x8000 partition-table.bin 0xf000 ota_data_initial.bin 0x20000 esp32-csi-node.bin`.
- Provision (vendor/RuView/firmware/esp32-csi-node/provision.py): read the password from
  secrets/wifi-pass.txt into a variable, never inline it. ALWAYS pass --node-id (and later --tdm-slot,
  --tdm-total): the script saves settings per COM port (in plain text, incl. password) and silently
  reuses them, so a new board on the same port would inherit the old node ID. Use
  `--state-dir secrets/provision-state` to keep that file in the gitignored folder.
- Server: built from the same tag, vendor/RuView (git tag v0.8.8-esp32), crate
  `wifi-densepose-sensing-server`, `--features mqtt`. Rust toolchain pinned by v2/rust-toolchain.toml.
- Run the server natively (Docker Desktop on Windows has a UDP source-address problem).

## Safety and privacy
- The GitHub repo (github.com/misterscott1999/WiFi-Sensing) is PUBLIC. Never commit anything that
  identifies or locates the home: Wi-Fi SSID, MAC addresses, the address, occupancy data or logs.
  Those go in secrets/ or data/ (both gitignored). Check `git diff --cached` before every commit.
- Occupancy data reveals when the house is empty: data/ is gitignored and must stay that way.
- Never commit secrets: the Wi-Fi password lives in secrets/ (gitignored), tokens in .env.
- Before flashing, confirm the COM port belongs to an ESP32-S3 (esptool flash-id) and never
  use `erase-flash` unless Chris asks (it wipes the node's stored Wi-Fi settings).
- Never control the thermostat or lights without Chris approving the rule first.

## Environment
- Python venv: `.venv/Scripts/python` (esptool installed there).
- Rust/cargo: `%USERPROFILE%\.cargo\bin`. MSVC Build Tools 2026 installed.
- Windows Smart App Control was turned OFF by Chris on 2026-11-13 because it blocked every
  freshly compiled (unsigned) program, including cargo build scripts ("os error 4551").
- PC LAN IP 192.168.0.41 (subnet 192.168.0.0/24, gateway 192.168.0.1); network profile Private.
- Home Wi-Fi: TP-Link Archer BE550 mesh, 2.4 GHz on channel 3, WPA3-Personal (boards join it fine).
  The SSID, MAC addresses and other identifying details live in secrets/local-notes.md.
