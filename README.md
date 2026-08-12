# SynkFlowInfrastructure

Local Docker Compose stack for the SynkFlow platform's on-prem services:
Mosquitto (MQTT broker), ESPHome (manages the Sunsynk RS485 bridge firmware),
Home Assistant (automations over the same MQTT data), and — added later,
see Services table below — Postgres and the `synkflow-api` container itself.

> Not cloud infrastructure — no Terraform / Pulumi / Kubernetes here.
> Everything runs on a single LAN host.

## Services

| Service | Image | Port | Description |
|---------|-------|------|-------------|
| Home Assistant | `ghcr.io/home-assistant/home-assistant:stable` | 8123 | Home automation platform |
| ESPHome | `ghcr.io/esphome/esphome` | 6052 | ESP device management |
| Mosquitto | `eclipse-mosquitto:2` | 1883 | MQTT broker |
| Postgres | `postgres:16-alpine` | 5432 | Database for synkflow-api |
| synkflow-api | `registry.stage9software.com/synkflow-api:latest` | 5174 → 8080 | Backend API (runs `depends_on` postgres + mosquitto) |

All services run on the `synk-flow` Docker network, also used by
[SynkFlowAPI](../SynkFlowAPI).

> **Note on image tags:** `esphome` has no tag (defaults to `latest`),
> `home-assistant` uses the floating `:stable` tag, and `synkflow-api` uses
> `:latest` with `pull_policy: always` — none of these are pinned to a
> reproducible version. `mosquitto:2` and `postgres:16-alpine` are pinned to
> a major version only. Expect drift on every `docker compose pull`.

> **Note on Mosquitto:** the broker runs with `allow_anonymous true` and no
> `password_file`/`acl_file`/TLS configured at all — anyone on the LAN who
> can reach port 1883 can publish or subscribe to any topic. There is
> currently no authentication on the broker.

## Prerequisites

- Docker + Docker Compose
- Create the Mosquitto host paths once before the first `up -d`:

  ```bash
  sudo mkdir -p /opt/stage9software/mosquitto/data
  sudo mkdir -p /opt/stage9software/mosquitto/log
  ```

## Getting started

```bash
docker compose up -d        # start all services and create the synk-flow network
```

## Stopping

```bash
docker compose down
```

## Updating

```bash
docker compose pull
docker compose up -d
```

## Directory structure

```
├── docker-compose.yml
├── homeassistant/
│   └── config/             # HA YAML, SQLite DB, .storage
├── esphome/
│   └── config/
│       ├── sunsynk.yaml    # ESP32 + Modbus integration
│       └── secrets.yaml    # wifi / api / ota secrets
└── mosquitto/
    ├── config/             # mosquitto.conf
    ├── data/               # (mounted from /opt/stage9software/mosquitto/data)
    └── log/                # (mounted from /opt/stage9software/mosquitto/log)
```

Not shown above: `docker-compose.yml` also defines `postgres` (data at
`/opt/stage9software/postgres/data`) and `synkflow-api` (data at
`/opt/stage9software/synkflow-api/data`) — these were added after this
directory tree was originally documented.

## Sunsynk Inverter — ESP32 RS485 Wiring

### Hardware
- **ESP32:** NodeMCU ESP32 (USB-C) — `esp32dev`
- **RS485 Module:** MAX485 TTL-to-RS485 converter (blue PCB)
- **Inverter:** Sunsynk 8kW, `modbus_controller` address `0x01` (as set in
  `sunsynk.yaml`)

### Wiring

| MAX485 Pin | ESP32 Pin | Wire Color | Notes |
|------------|-----------|------------|-------|
| DI (Data In) | P16 (GPIO 16) | Purple | UART TX — sends data to inverter |
| RO (Receive Out) | P4 (GPIO 4) | Blue | UART RX — receives data from inverter |
| DE + RE (tied) | P0 (GPIO 0) | Black (via breadboard row 62) | Flow control — **GPIO 0 is a boot pin, consider moving to P5**. Note: `esphome/config/sunsynk.yaml` currently sets `flow_control_pin: 18`, not 0 — this table and the live config disagree, unverified which is actually wired. |
| VCC | 5V | Black | Power |
| GND | GND | Brown | Ground |

### RS485 to inverter
- Connect MAX485 **A** and **B** screw terminals to the Sunsynk
  **Meter-485** port (not the CAN/BMS port).
- If no communication, try **swapping A and B** wires.

### ESPHome config
- Config file: `esphome/config/sunsynk.yaml`
- Secrets file: `esphome/config/secrets.yaml`
- Modbus: 9600 baud, 8N1, `modbus_controller` address `0x01` (corrected —
  earlier versions of this README said `0x00`, which doesn't match the file)
- `esphome/config/archive/sunsynk.yaml` is an older, superseded version of
  this config, kept for reference. It has its own hardcoded API/OTA secrets.
- A fallback WiFi AP is configured for recovery if the configured SSID is
  unreachable. **Its password is hardcoded inline as `fallback123`**
  (`sunsynk.yaml:21`, not via `!secret`) — if that AP ever comes up, anyone in
  radio range can join it.
- Sensor cadence: `fast_update_interval: 10s` drives the `modbus_controller`
  poll, with `medium_skip_updates: 3` (30s) and `slow_skip_updates: 6` (60s) for
  slower registers. The only filters in the file are `multiply`/`offset`/`lambda`
  — there is **no `delta:` or `throttle:`**, so every sensor republishes on every
  poll whether or not its value changed. That is what sets the downstream MQTT
  and SignalR message rate; see `../SynkFlowAPI/CLAUDE.md`.

## Secrets

`!secret` references resolve against `esphome/config/secrets.yaml` and
`homeassistant/config/secrets.yaml`.

> **Warning:** `wifi_password`, `api_key`, and `ota_password` are currently
> committed in the ESPHome `secrets.yaml` — and a second set of hardcoded
> (non-`!secret`) API/OTA credentials plus a fallback-AP password live in
> `esphome/config/archive/sunsynk.yaml`. Rotate all of these before sharing
> this repository, or move them out to an untracked secrets store.
>
> Separately, `.gitignore` targets `homeassistant/.storage/` and
> `homeassistant/*.log`, but the real paths are one level deeper
> (`homeassistant/config/.storage/`, `homeassistant/config/*.log`), so those
> patterns never actually applied. `git ls-files` shows the HA `.storage/`
> directory (including `auth` and `http.auth`), the HA logs, and the SQLite
> DB are all tracked in git history today. Fixing `.gitignore` won't remove
> them from history — that needs a separate `git rm --cached` cleanup.

## Security posture

Reviewed 2026-08-12. Everything below is **open and deliberately deferred** — a
later phase will cover authentication and secret containment together. Nothing
here is an oversight; it is a known state on a trusted LAN.

| # | Finding | Severity | Status |
|---|---------|----------|--------|
| 1 | HA `.storage/` tracked in git: `auth` holds **2 live refresh tokens** each with a 128-char `jwt_key`; `auth_provider.homeassistant` holds the `admin` bcrypt hash. Present since the first two commits. | High | Deferred |
| 2 | `esphome/config/secrets.yaml` — real `wifi_ssid`, `wifi_password`, `api_key`, `ota_password`. Second copy inline in `archive/sunsynk.yaml`. | High | Deferred |
| 3 | Mosquitto `allow_anonymous true`, no `password_file`/`acl_file`/TLS, published `1883:1883`. Any LAN device can publish to `sunsynk/sensor/+/state` and inject fabricated readings into the API's database. | Medium | Deferred |
| 4 | Every service binds `0.0.0.0` — Postgres `5432` (with the default password if `.env` is absent), HA `8123`, ESPHome dashboard `6052`, API `5174`. | Medium | Deferred |
| 5 | Fallback AP password `fallback123` hardcoded in the live `sunsynk.yaml`. | Low | Deferred |

A refresh token plus its `jwt_key` mints working access tokens and **does not
expire on its own**.

### Rotation checklist

Run this when you get to the containment phase. Order matters for the first item.

1. **Home Assistant** — changing the admin password does **not** invalidate
   existing refresh tokens. Revoke them explicitly: stop HA, delete
   `homeassistant/config/.storage/auth`, restart, and re-onboard. Then change the
   admin password.
2. **WiFi** — rotate `wifi_password`; the committed value is the live network key.
3. **ESPHome** — rotate `api_key` and `ota_password` in **both**
   `esphome/config/secrets.yaml` *and* `esphome/config/archive/sunsynk.yaml`.
   Re-flash the ESP32 after changing the API key.
4. **Fallback AP** — replace `fallback123` in `sunsynk.yaml` with a `!secret`.
5. **Postgres** — set a real `POSTGRES_PASSWORD` in `.env`; the compose fallback
   `synkflow_dev_password` is committed in both this repo and `../SynkFlowAPI`.
6. **Sunsynk Connect** — rotate the cloud account password; the account email and
   plant ID are committed in `../SynkFlowAPI/Data/sunsynk.db`.

Then untrack the files. Fixing `.gitignore` alone does nothing for already-tracked
paths — that needs `git rm --cached -r` on the HA `.storage/` tree, both
`secrets.yaml` files, the HA logs and SQLite DB, and `mosquitto/data/`. Git
history still holds the old values afterwards, which is why rotation comes first.

## Networking

The Mosquitto address is hardcoded in `esphome/config/sunsynk.yaml` as
`192.168.0.50:1883` (and duplicated as the `mqtt_broker` secret in
`esphome/config/secrets.yaml`, which still has a "replace with your Docker
host IP" TODO comment next to it). The host running this stack must be
reachable at that IP from the ESP32 on the LAN. The `synkflow-api` container
that now runs *in this same compose file* reaches the broker over the
Docker network instead, via `Mqtt__Host=mosquitto`.

Postgres connection details for `synkflow-api` come from `.env`
(`POSTGRES_DB`/`POSTGRES_USER`/`POSTGRES_PASSWORD`, see `.env.example`) with
insecure defaults (`synkflow`/`synkflow`/`synkflow_dev_password`) baked into
`docker-compose.yml` if `.env` is absent — set a real `.env` before exposing
this stack beyond a trusted LAN.

## Related projects

- [SynkFlow](../SynkFlow) — Flutter client
- [SynkFlowAPI](../SynkFlowAPI) — source for the backend API; this repo runs
  its published image (`registry.stage9software.com/synkflow-api:latest`)
  alongside its own Postgres instance, not the source directly.
