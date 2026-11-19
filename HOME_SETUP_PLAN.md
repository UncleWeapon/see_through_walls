# Home WiFi sensing: implementation plan

A hobby plan for running RuView on a home network. It is based on a read of
this repository at upstream commit `13b4788` (November 2026) and on three
things actually run during that review:

- `python archive/v1/data/proof/verify.py` printed `VERDICT: PASS` (needs `numpy` and `scipy`).
- `cargo build --release -p wifi-densepose-sensing-server` succeeded in about 4.5 minutes on Linux.
- The server ran with `--source simulate` and answered `/health` and `/api/v1/status`.

No hardware was involved, so everything about real boards below comes from the
repository's own documentation, not from a test.

## What to expect

The README headline ("see through walls", 17-keypoint pose) is well ahead of
what a home ESP32 setup delivers. The repository's own maturity notes say so.

| Capability | Status on home ESP32 hardware | Source |
|---|---|---|
| Presence and motion along a link | Works; a variance threshold after 60 s of empty-room learning. False positives from fans, microwaves and neighbouring APs | `firmware/esp32-csi-node/README.md`, Tier 2 caveats |
| Breathing rate | Heuristic. Published only after calibration, with exactly one still occupant | `docs/integrations/home-assistant.md` |
| Heart rate | Same pipeline, weaker signal. Treat as a curiosity | same |
| Fall detection | A phase-acceleration threshold. Needs tuning per room (`fall_thresh`) | `docs/user-guide.md` |
| Person count | A subcarrier-diversity heuristic on the node; the server overcounts (6 to 8 reported for 5 bodies) | `docs/TROUBLESHOOTING.md` §2 |
| Through-wall | Presence through one interior wall is plausible ("up to ~5 m, signal-dependent"). No imaging | `README.md` |
| Live 17-keypoint pose | Not working. The on-device model scores PCK@20 = 3.0% and its runtime path is a stub. The skeleton in the UI is synthesized from heuristics | `README.md`, "Model weights: what's real, what's not" |
| Home Assistant | 15 MQTT entities per node. Zones, bed-exit, pose and the Matter bridge are not wired up yet | `docs/integrations/home-assistant.md` |

A realistic goal: per-room presence and motion in Home Assistant, a breathing
rate reading for one person sitting or sleeping, and a live dashboard. That is
a good hobby result, and it is what the plan below aims for.

## Hardware

Your existing router is the signal source. It stays as it is; nothing is
flashed onto it.

| Item | Quantity | Notes |
|---|---|---|
| ESP32-S3-DevKitC-1, 8 MB flash, 8 MB PSRAM (N8R8) | 1 to start, 3 for a room mesh | About $8 to $10 each. Avoid coin-sized clones (S3-Zero, SuperMini): the firmware keeps the radio awake and the docs report boards overheating |
| USB-C data cables and 5 V USB power supplies | one per board | Use the UART port (left port on most DevKitC boards) for flashing |
| A computer that stays on | 1 | Runs the sensing server. Linux or a Raspberry Pi 4/5 is the easy path. Docker Desktop on Windows drops UDP from all but one node and needs a relay script. Docker on macOS needs a different allowlist |
| Optional: 24 GHz or 60 GHz mmWave module (LD2410, MR60BHA2) | 1 | The firmware fuses it with CSI. The most reliable way to improve vitals |

Skip for now: the Cognitum Seed ($140 total kit), ESP32-C6 and C5 boards
(research targets), and Intel 5300 NICs.

Router settings that matter: the nodes join 2.4 GHz, so keep a 2.4 GHz SSID
enabled, fix its channel instead of leaving it on auto, and give the server
machine and each node a DHCP reservation.

## Phases

### Phase 0: software only (no hardware, an evening)

Clone with submodules. A plain clone does not build:

```bash
git clone --recurse-submodules https://github.com/UncleWeapon/see_through_walls
```

Then either run the published Docker image in simulated mode, or build from
source:

```bash
cd v2
cargo build --release -p wifi-densepose-sensing-server
RUVIEW_ALLOW_UNAUTHENTICATED=1 ./target/release/sensing-server --source simulate --http-port 3000 --ws-port 3001
```

Open `http://localhost:3000/ui/index.html` and get familiar with the dashboard
and the REST endpoints (`/api/v1/status`, `/api/v1/nodes`,
`/api/v1/sensing/latest`). Done when you can tell simulated data from live data
in `/api/v1/status` (`source_state: synthetic`).

### Phase 1: one node streaming (a weekend)

1. Download a firmware bundle from the upstream
   [releases page](https://github.com/ruvnet/RuView/releases). Do not use
   `firmware/esp32-csi-node/release_bins/`; the docs call those historical.
   v0.8.8 is the last release with physical-board evidence; later ones are
   prereleases.
2. For a bare DevKitC, build from source with the `sdkconfig.defaults.devkitc`
   overlay. The stock 8 MB bundle enables a display, and on a board without one
   CSI yield drops to zero. The exact Docker build command is in the header of
   that file.
3. Flash with `esptool` at the four documented offsets, then provision:

   ```bash
   python firmware/esp32-csi-node/provision.py --port <PORT> \
     --ssid "<SSID>" --password-file ~/.wifi-pass --target-ip <SERVER_IP> --node-id 1
   ```

4. Start the server so it listens on the LAN. By default it only listens on
   loopback and no node ever appears:

   ```bash
   ./target/release/sensing-server --source esp32 --udp-port 5005 \
     --udp-bind 0.0.0.0 --udp-allow <NODE_IP>/32 --http-port 3000 --ws-port 3001
   ```

Done when `/api/v1/nodes` lists the node and `/api/v1/status` shows
`live_unverified` with a frame rate near 20 Hz. If the rate is low, run
`python scripts/csi-udp-poke.py <NODE_IP> --hz 50 --seconds 60` from a wired
machine; some routers answer pings at rates that carry no CSI.

### Phase 2: make one room meaningful (a weekend)

1. Place the node so the line between it and the router crosses the area you
   care about, at roughly chest height, away from fans and the microwave.
2. Turn on edge processing: re-run `provision.py` with `--edge-tier 2`.
   Power-cycle the node with the room empty so its 60 s baseline is clean.
3. Run the room calibration from `docs/calibration-guide.md`:
   `calibrate` (empty room, 600 frames minimum), `enroll` (8 guided poses,
   about 4 minutes), `train-room`, then `room-status` to see which specialists
   actually trained.
4. Keep a simple log for a few days: when the room was really empty or
   occupied, against what the system reported. This is your own accuracy
   number, and the only one that applies to your house.

Done when presence is right most of the time over several days, and you know
what causes the misses.

### Phase 3: Home Assistant (an evening)

Rebuild the server with `--features mqtt`, point it at a Mosquitto broker with
`--mqtt --mqtt-host <BROKER>`, and keep `--data-dir` on persistent storage so
device IDs survive restarts. Use `presence` and `room_active` for automations
(lights, heating). Leave `fall` and `possible_distress` as notifications only;
they are not reliable enough to act on alone. Consider `--privacy-mode` if
other people live in the house: it removes heart and breathing rate everywhere.

### Phase 4: three-node mesh (when phase 2 is stable)

Add two more boards around the same room so the links cross from different
angles. Give each a unique `--node-id` and `--tdm-slot N --tdm-total 3`. The
docs note that a shared mesh key is described in ADR-032 but not implemented,
so the UDP stream is unauthenticated; rely on the `--udp-allow` list. Compare
your phase 2 log before and after to see whether the extra nodes helped.

### Phase 5: experiments (optional, pick what interests you)

- mmWave fusion for better breathing and heart rate.
- Sleep tracking with `scripts/sleep-monitor.js`, checked against a wearable.
- Camera-supervised pose training (`docs/user-guide.md`, "Camera-Supervised
  Pose Training"): record a webcam and CSI together for 40 minutes and train
  your own model. This is the only route to pose that the repo documents, and
  the stated target is modest (PCK@20 of 35%).
- A second room, to test "through one wall" presence honestly.

## Suggestions

- **Run the server on a Linux box or a Pi with Docker or a native binary.** It
  avoids both documented Docker Desktop UDP problems.
- **Put the nodes on an IoT VLAN or guest network if your router supports it,**
  with only UDP 5005 allowed to the server. The nodes run an HTTP management
  and OTA endpoint on port 8032, and the CSI stream has no authentication. Set
  an OTA key with `--ota-psk-file`.
- **Keep the API token on.** The server refuses to start without
  `RUVIEW_API_TOKEN` unless you opt out; only opt out for local testing, and
  keep ports 3000 and 3001 bound to loopback or the LAN, never forwarded.
- **Treat every vital-sign number as unvalidated.** Nothing here is a medical
  device, and the repo says the same.
- **Sense only your own home.** WiFi sensing reaches a little past walls; aim
  links away from neighbours, and tell the people you live with what it records.
- **Never commit WiFi passwords, tokens or recorded CSI.** The upstream rules
  in `CLAUDE.md` say the same, and recorded CSI of people is personal data.
- **Ignore most of the repository.** Of the 70-odd crates, a home setup uses
  `wifi-densepose-sensing-server`, `wifi-densepose-cli`, the firmware and `ui/`.
  The harness, Homecore, Cognitum, swarm and world-model parts are separate
  projects you can skip.
- **Pin a known-good state.** Upstream moves fast (1,600+ commits) and
  prereleases lack hardware evidence. Tag your repo when a phase works and
  update deliberately.

## Staying in sync with upstream

```bash
git remote add upstream https://github.com/ruvnet/RuView
git fetch upstream
git merge upstream/main
git submodule update --init
```

Keep your own notes and configuration in new files (like this one) so merges
stay clean.
