# Wi-Fi Sensing Home Occupancy

Detect whether people are home on one floor using Wi-Fi Channel State Information (CSI) from three
ESP32-S3 boards and [RuView](https://github.com/ruvnet/RuView), then (later) drive the Nest thermostat
and lights through Home Assistant. Plan: [docs/plan.md](docs/plan.md).

## Setup (Phase 0, done 2026-11-13)

| Piece | Where | Notes |
|---|---|---|
| Python venv | `.venv/` | `.venv\Scripts\python -m pip install -r requirements.txt` |
| Firmware | `firmware/v0.8.8-esp32/s3-8mb/` | From the [v0.8.8-esp32 release](https://github.com/ruvnet/RuView/releases/tag/v0.8.8-esp32), SHA256 verified |
| RuView source | `vendor/RuView/` | `git clone --depth 1 --branch v0.8.8-esp32 https://github.com/ruvnet/RuView.git vendor/RuView` then `git -C vendor/RuView submodule update --init --recursive --depth 1` |
| Sensing server | `vendor/RuView/v2/target/release/sensing-server.exe` | `cd vendor/RuView/v2; cargo build --release --locked -p wifi-densepose-sensing-server --features mqtt` |

`firmware/` and `vendor/` are not committed (re-download with the commands above).

## Run the server

```powershell
cd vendor\RuView\v2
.\target\release\sensing-server.exe --http-port 3000 --ws-port 3001 --source esp32 --ui-path ..\ui `
  --udp-bind 0.0.0.0 --udp-allow 192.168.0.0/24
```
`--ws-port 3001` matters: the web UI served on port 3000 looks for the live stream on 3001. Without it the
server streams on 8765, and the UI shows "Data Source: OFFLINE" and plays a fake in-browser simulation.
RuView's built-in dashboard: http://localhost:3000/ui/index.html. Node list: http://localhost:3000/api/v1/nodes.
Sample API responses (for writing and testing our code) are in `samples/`.

## Boards

All three: ESP32-S3 N16R8 (16 MB flash, 8 MB PSRAM), RuView firmware 0.8.8 (s3-4mb headless build).
MAC addresses are kept out of this public repo (local copy in gitignored secrets/local-notes.md).

| Node | USB port when flashed | LAN IP | CSI rate | Room |
|---|---|---|---|---|
| 1 | COM3 | 192.168.0.125 | ~30 pps | TBD |
| 2 | COM4 | 192.168.0.20 | ~33 pps | TBD |
| 3 | COM5 | 192.168.0.210 | ~30 pps | TBD |
