# Wi-Fi Sensing Home Occupancy: Project Plan

## Context
Chris wants to use **RuView** (github.com/ruvnet/RuView, open-source Wi-Fi CSI sensing, MIT) with his
3x **ESP32-S3 DevKitC** boards to detect occupancy on one floor of the house. The end goal is to drive
the **Google Nest thermostat** (and smart lights) from home/away state. First, though, he wants a **simple
dashboard** to see what the sensors detect: motion, person vs. pet, and which of the 3 nodes see it.
That dashboard will be used to pick fusion rules and thresholds before any automation goes live.

Decisions made: ESP32-S3 boards (supported by RuView), this Windows PC as the host for now, Nest
thermostat, new project at `C:\Users\miste\WiFi-Sensing` (its own git repo, public on GitHub, separate from Job Search).

### What research showed about RuView (it shapes the plan)
- **How it works:** each board joins the home Wi-Fi as a normal client. It streams CSI over **UDP 5005**
  to a host running the Rust `sensing-server`, which serves a built-in UI (`http://localhost:3000/ui/index.html`),
  a REST API (`/api/v1/nodes`, `/api/v1/sensing/latest`, `/api/v1/status`), a WebSocket (`/ws/sensing`), and
  **MQTT with Home Assistant auto-discovery**. HA gets presence, motion_level, presence_score and other entities per node.
- **Maturity:** beta software. Firmware v0.8.14 is a prerelease, so pin to **v0.8.8 stable**. Presence is a
  *heuristic threshold, not a trained classifier*. The "100% accuracy" claim was retracted. Person count is
  unreliable, and vitals are experimental.
- **Pets:** RuView's calibration guide says it **cannot tell a pet from a person**. We have to handle this
  ourselves (see Phase 4). It's the biggest open risk for "away" automations.
- **Gotchas:** the ESP32-S3 uses **2.4 GHz only**. The server listens only on localhost unless started with
  `--udp-bind 0.0.0.0 --udp-allow <LAN subnet>`. Docker Desktop on Windows has a source-address bug, so run
  the server natively. ESP-IDF builds don't work from Git Bash, so use prebuilt firmware. Nodes calibrate a
  60-second "empty room" baseline at power-on.
- **Plan B:** if RuView proves too flaky, ESPectre (ESPHome-based CSI motion sensing) runs on the same S3
  boards and plugs straight into Home Assistant.

## Key design idea: absence is the hard part
Wi-Fi sensing detects **movement** well and **still people** (sleeping, reading) poorly. Pets also move.
So the final "away" decision should not rest on CSI alone:
- **Home** = recent motion on >=2 of 3 nodes (corroborated), OR any phone is home.
- **Away** = all phones away (HA Companion app geofence) AND no corroborated CSI motion for N minutes.
- CSI's job is to **catch what phones miss** (a guest, a phone left at home, someone home without a phone),
  and to drive **room-level lights**, where the cost of a false trigger is low.

The dashboard phase exists to measure how well that works in this house before we trust it.

### How phone presence is detected
- **Primary:** Home Assistant Companion app (iOS/Android) on each adult's phone, with location on. A GPS
  geofence around the house gives `person.<name>` = home/away. Arrival is known *before* you walk in, so
  the house can pre-heat or pre-cool. CSI can't do that.
- **Backup:** the Companion app's "connected Wi-Fi SSID" sensor. On the home SSID it counts as home, so it
  still works when GPS is flaky indoors.
- **Skip:** router ping and ARP tracking. Phones sleep their Wi-Fi and randomize MACs, which makes them unreliable.
- HA's `person` entity merges a person's trackers. `zone.home` gives the count of people home.

### Combining phones and CSI (four cases)
| Phones | CSI floor motion | Verdict | Action |
|---|---|---|---|
| Someone home | Any | Home | Normal schedule. Lights follow room motion. |
| All away | None for N min | **Away (confident)** | Nest Eco, lights off |
| All away | Corroborated (>=2 nodes) | **Unexpected** (guest, sitter, phone forgotten, or pet) | Stay comfortable and send a phone notification "motion while everyone's away". Count how often it's a pet. |
| Someone home | None for hours | Home but still (asleep or reading), or a phone left at home | Stay Home (comfort-safe). Night schedule handles sleep. |

- **Arrival:** the phone geofence ends Eco immediately. CSI motion is a backup for a phone-less arrival.
- **Departure:** wait until all phones are away AND CSI has been quiet for N minutes (tuned in Phase 4,
  likely 15-30). This stops a pet or a still person from being the only thing between Eco and comfort.
- **Where it lives:** an HA template `binary_sensor.house_occupied` built from `zone.home` plus our
  `binary_sensor.floor_motion` (published over MQTT by `occupancy/`), and one HA automation for Nest.

### Bonus: phones give free labels
Once HA plus the Companion app is running, "all phones away" periods are automatic ground truth for
"nobody home (except maybe the pet)". The collector pulls `zone.home` from HA's REST API (long-lived
token in `.env`) and logs it next to the CSI readings. That cuts down manual label-tapping, and it directly
measures the pet's false-motion rate. **So bring up HA + Companion app early, right after Phase 2**,
even though Nest control waits until Phase 5.

## Phases

### Phase 0: Prep (≈1 evening)
1. Confirm the router broadcasts **2.4 GHz** on the SSID the boards will use (mesh routers sometimes merge
   or split bands). Give this PC a **DHCP reservation** (fixed LAN IP) so the boards always know where to send data.
2. Install the tools: Python 3.12 + venv, `esptool`, and Rust (`rustup`) unless the release ships a Windows server binary.
3. Plug in one board and run `python -m esptool flash_id` to confirm the chip and **flash size** (8 MB vs 4 MB
   decides which firmware bundle to use).
4. Download the RuView v0.8.8 firmware bundle and check it against `SHA256SUMS.txt`. Clone RuView (pinned tag).
5. Add a Windows Firewall rule allowing inbound UDP 5005 from the LAN.

### Phase 1: One node on the desk (prove the pipeline)
1. Flash board #1 (`esptool ... write_flash` with the bootloader, partition table, ota_data and app at the
   documented offsets), then provision it: `provision.py --port COMx --ssid ... --password-file <outside repo>
   --target-ip <PC IP> --node-id 1`.
2. Build and run `sensing-server` with `--http-port 3000 --source auto --udp-bind 0.0.0.0 --udp-allow 192.168.x.0/24`.
3. **Done when:** the node appears in `/api/v1/nodes`, CSI arrives at about 20 packets/s, and RuView's built-in UI
   visibly reacts when you walk past.
4. Save sample JSON from each endpoint into `samples/`. Our code is written against these real responses,
   not guessed shapes.

### Phase 2: Three nodes on one floor
- **Placement:** spread the boards in a rough triangle across the floor's main living areas, about 1-1.5 m high
  (shelf height), away from the microwave, big metal and fans. Aim for the router-to-node paths to **cross the areas
  people use**. Record each node's room in `config.yaml`.
- Provision node IDs 1-3 with TDM slots (3 total; check `provision.py --help` for the exact flags).
- Calibrate: everyone (and the pet, if possible) out of the floor, HVAC in its normal state, power-cycle all
  3 boards, then wait 60 seconds.
- **Done when:** all 3 show in `/api/v1/nodes`, and a walk-through produces motion on the expected node(s).

### Phase 3: Our occupancy dashboard (the main build)
Simple Python, inside `C:\Users\miste\WiFi-Sensing`:
```
occupancy/
  config.yaml      # node id -> room name, thresholds, fusion rule, poll interval
  collector.py     # polls RuView REST every ~1 s, writes per-node readings to SQLite
  fusion.py        # pure functions: per-node state -> floor verdict (unit-tested)
  dashboard.py     # Streamlit app: live view + history + labeling
  labels.py        # store ground-truth labels
tests/             # pytest for fusion rules, using captured samples/
data/              # occupancy.db (gitignored)
samples/           # captured API JSON from Phase 1
.env               # RUVIEW_API_TOKEN etc. (gitignored); .env.example with names only
```
Dashboard contents:
- **Per-node tiles (3):** room name, online/offline, presence yes/no, motion level, presence score, and
  last-seen time.
- **"Nodes agreeing" indicator:** 0/1/2/3 nodes currently see presence or motion, plus the floor verdict under
  the current rule (for example, "Occupied: 2 of 3 nodes, 40 s").
- **Timeline chart:** motion level per node over the last 1, 6 and 24 hours, so we can see whether
  one node is noisy or always triggers alone.
- **Ground-truth buttons:** "Nobody home", "Only pet home", "People home (moving)", "People home (still or
  asleep)". Each tap writes a timestamped label. This is how we answer the person-vs-pet and threshold
  questions with real data instead of guesses.
- **Rule tuner:** sliders for "min nodes agreeing" and "hold time", replayed against the logged history and
  scored against the labels (how often the rule would have been wrong).

Streamlit is used because it's plain Python, with no HTML/JS to maintain, which suits a beginner.

### Phase 4: Measure and tune (1-2 weeks of normal life)
- Run the collector continuously and tap labels when the household state changes.
- Scripted tests: walk each room, sit still for 10 min, empty house, pet alone (if applicable), fan or
  microwave on.
- Decide from data: (a) is 2-of-3 agreement better than any-1? (b) what hold time avoids flapping? (c) can
  pets be told apart at all? Possible signals are a lower motion level, motion on only one low node, and short bursts.
  If not, pet-only periods count as "away" only when phones agree.
- Output: chosen rule + thresholds written into `config.yaml`, plus a short `docs/findings.md`.

### Phase 5: Home Assistant + Nest + lights (after tuning)
- Run **Home Assistant OS** as a VM on this PC to start (Hyper-V or VirtualBox), or later on a small always-on
  mini PC or Pi. Add the **Mosquitto** MQTT broker add-on.
- Restart RuView's server with `--mqtt` so HA auto-discovers per-node entities. Publish **our fused
  verdict** to MQTT too, as a `binary_sensor.floor_occupied`, so HA uses the tuned rule, not raw single-node flags.
- **Nest:** HA's Google Nest integration through Google Device Access (one-time $5 fee). Automations: away
  → Eco mode, home → resume schedule. Lights: room-level motion → on, no motion for X min → off.
- **Google Home:** not needed for control (HA talks to Nest directly). It can optionally be exposed later through
  Nabu Casa.
- Install the HA Companion app on phones for geofence presence (the AND/OR design above).

## Risks to watch
- RuView is beta and its docs overstate some features, so pin versions and trust measured behavior over the README.
- Pet vs. person may not be separable. Mitigate with phone presence.
- Still occupants look like an empty room. Mitigate with long hold times and phone presence.
- This PC must stay on for detection. Plan to move to an always-on box at Phase 5.
- Privacy: CSI and occupancy logs reveal household routines. The repo is public, so `data/`, `secrets/` and `.env` stay gitignored and no SSID, MACs or address are ever committed.

## Verification
- Phase 1/2: `curl http://localhost:3000/api/v1/nodes` lists the node(s), and walk tests move the motion values.
- Phase 3: `pytest` passes for the fusion rules on captured samples. The dashboard shows live per-node tiles and
  updates within about 2 s of a walk-through. A label tap appears in the DB.
- Phase 4: the rule tuner reports the error rate against labeled history, and the chosen rule beats the alternatives.
- Phase 5: HA shows the RuView entities and `floor_occupied`. A manual "everyone leaves" test puts Nest in
  Eco within the configured time, and a walk-in turns the room lights on.

## First implementation session (after approval)
Phase 0 steps 2-5 plus the Phase 3 project skeleton (folders, venv, config.yaml, README, CLAUDE.md,
.gitignore). Phase 0 step 1 (router and DHCP) and Phase 1 flashing need Chris at the keyboard with a board plugged in.
