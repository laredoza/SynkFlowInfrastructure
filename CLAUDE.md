# SynkFlowInfrastructure

Local Docker Compose stack that provides the on-prem services the SynkFlow
platform depends on: **Mosquitto** (MQTT broker), **ESPHome** (ESP32 firmware
management for the Sunsynk Modbus bridge), and **Home Assistant** (used for
automations / observability on top of the same MQTT data).

This is **not** cloud infrastructure — there's no Terraform / Pulumi / Bicep /
Kubernetes here. Everything runs locally and is reachable on the LAN.

## Layout

```
docker-compose.yml         Service orchestration (HA, ESPHome, Mosquitto)
README.md                  Setup + Sunsynk RS485 wiring diagram
homeassistant/config/      HA YAML + SQLite DB + .storage
esphome/config/
  sunsynk.yaml             ESP32 + MAX485 Modbus integration with the inverter
  secrets.yaml             wifi / OTA / API secrets (see warning below)
mosquitto/                 Broker config; data/logs mount to /opt/stage9software/mosquitto
```

## Running

```bash
docker compose up -d
docker compose down
docker compose pull && docker compose up -d   # update images
```

The Mosquitto data and log directories under
`/opt/stage9software/mosquitto/` must exist on the host before the first
`up -d` (see README). They are mounted from outside the repo so the volumes
survive a checkout wipe.

## Networking & dependencies

- Docker network: `synk-flow`.
- Mosquitto address is hardcoded in `esphome/config/sunsynk.yaml` as
  `192.168.0.50:1883`. The host running this stack must be reachable at that
  IP from the ESP32 on the LAN.
- `SynkFlowAPI` connects to the same broker — start this stack first.

## Hardware notes

- Sunsynk 8kW inverter ↔ ESP32 over RS485 (MAX485 module).
- Modbus serial: 9600 baud, 8N1, inverter address `0x00`.
- ESPHome includes a fallback WiFi AP for recovery if the configured SSID is
  unreachable.

## State

- Home Assistant: SQLite (`home-assistant_v2.db`) + `.storage/` inside
  `homeassistant/config/`.
- ESPHome: build artifacts in `.esphome/` (gitignored).
- Mosquitto: persistent data outside the repo at
  `/opt/stage9software/mosquitto/`.

## Secrets

YAML secrets files (`esphome/config/secrets.yaml`,
`homeassistant/config/secrets.yaml`) are referenced via `!secret <name>` in
the configs. `.env`-style files are gitignored.

**Warning:** at the time of writing, `wifi_password`, `api_key`, and
`ota_password` appear in-repo. Treat the repo as sensitive and rotate any
secrets before publishing it, or move them out to an untracked secrets file.

## Environments

Single local environment only — no dev/staging/prod split lives here.

## CI/CD

None in this repo.

## Sibling projects

- `../SynkFlowAPI` — backend that consumes MQTT from Mosquitto.
- `../SynkFlow` — Flutter client that talks to the API.
