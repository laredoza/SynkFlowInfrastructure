# SynkFlowInfrastructure

Local Docker Compose stack that provides the on-prem services the SynkFlow
platform depends on: **Mosquitto** (MQTT broker), **ESPHome** (ESP32 firmware
management for the Sunsynk Modbus bridge), **Home Assistant** (used for
automations / observability on top of the same MQTT data), plus **Postgres**
and the **synkflow-api** container itself (added after the initial cut of this
repo — see `docker-compose.yml`). So this stack now runs the API and its
database, not just the IoT-facing services.

This is **not** cloud infrastructure — there's no Terraform / Pulumi / Bicep /
Kubernetes here. Everything runs locally and is reachable on the LAN.

## Layout

```
docker-compose.yml         Service orchestration (HA, ESPHome, Mosquitto, Postgres, synkflow-api)
README.md                  Setup + Sunsynk RS485 wiring diagram
homeassistant/config/      HA YAML + SQLite DB + .storage
esphome/config/
  sunsynk.yaml             ESP32 + MAX485 Modbus integration with the inverter
  secrets.yaml             wifi / OTA / API secrets (see warning below)
  archive/sunsynk.yaml     old version of the config — also has hardcoded (non-!secret) API/OTA credentials, see warning below
mosquitto/                 Broker config; data/logs mount to /opt/stage9software/mosquitto
```

## Services (docker-compose.yml)

| Service | Image | Tag pinning |
|---|---|---|
| homeassistant | `ghcr.io/home-assistant/home-assistant` | `:stable` — floating tag, not a fixed version |
| esphome | `ghcr.io/esphome/esphome` | no tag at all → implicit `:latest` |
| mosquitto | `eclipse-mosquitto` | `:2` — major version only |
| postgres | `postgres` | `:16-alpine` — major version only |
| synkflow-api | `registry.stage9software.com/synkflow-api` | `:latest` **and** `pull_policy: always` — always pulls newest on `up` |

Every image here floats to some degree; none is pinned to an exact
version/digest. `synkflow-api` is the least reproducible (explicit `latest` +
forced pull). Installed HA version observed in `.HA_VERSION` at time of
writing: `2026.3.3` — will drift on next pull since the tag is `stable`.

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
  `192.168.0.50:1883` (also duplicated in `esphome/config/secrets.yaml` as
  `mqtt_broker`, with a "replace with your Docker host IP" TODO still in
  place). The host running this stack must be reachable at that IP from the
  ESP32 on the LAN.
- `synkflow-api` (now a service in this repo's own `docker-compose.yml`, see
  Services table) connects to the broker in-network via `Mqtt__Host=mosquitto`
  / `Mqtt__Port=1883` env vars — no hardcoded IP needed there since it's on
  the same `synk-flow` Docker network as `mosquitto`.

**Mosquitto has no authentication or TLS.** `mosquitto/config/mosquitto.conf`
is exactly: `listener 1883`, `allow_anonymous true`, plain persistence — no
`password_file`, no `acl_file`, no `cafile`/`certfile`. Anyone who can reach
port 1883 (the whole LAN, since it's a straight `1883:1883` port mapping) can
publish/subscribe to any topic, including the `sunsynk` topic tree feeding
Home Assistant and, now, `synkflow-api`.

## Hardware notes

- Sunsynk 8kW inverter ↔ ESP32 over RS485 (MAX485 module).
- Modbus serial: 9600 baud, 8N1. **Inverter/modbus_controller address in
  `sunsynk.yaml` is `0x01`**, not `0x00` (corrected — previous version of this
  doc had it wrong).
- `sunsynk.yaml` sets `flow_control_pin: 18` for DE/RE — the README's wiring
  table lists GPIO 0 for DE/RE. These two disagree; unverified which is the
  physically-correct one. Check the live config before trusting the wiring
  table's pin number.
- ESPHome includes a fallback WiFi AP for recovery if the configured SSID is
  unreachable. Its password is hardcoded inline as `fallback123`
  (`sunsynk.yaml:21`, not via `!secret`).

## Sensor cadence (load-bearing for the API and client)

`sunsynk.yaml` substitutions set the whole downstream message rate:

```yaml
fast_update_interval: "10s"   # modbus_controller poll
medium_skip_updates: "3"      # 3 * 10s = 30s
slow_skip_updates:   "6"      # 6 * 10s = 60s
```

97 sensor entries, 32 of them using `skip_updates`. **The only filters in the file
are `multiply` (32), `offset` (3) and `lambda` (1) — there is no `delta:` and no
`throttle:`**, so ESPHome republishes every sensor on every poll regardless of
whether the value moved. Of the 41 topics `../SynkFlowAPI`'s `SensorMapper`
recognises, 22 are "fast" (10s), which works out to roughly **160 MQTT messages
per minute** reaching the API.

That number is why the API coalesces its SignalR broadcast rather than forwarding
per-message — see `../SynkFlowAPI/CLAUDE.md`. If you add `delta:` filters here to
cut traffic, be aware the API's rollup averaging assumes a roughly fixed sample
interval, and gaps become indistinguishable from power failures (which is a real
scenario on this install).

## State

- Home Assistant: SQLite (`home-assistant_v2.db`) + `.storage/` inside
  `homeassistant/config/`.
- ESPHome: build artifacts in `.esphome/` (gitignored).
- Mosquitto: persistent data outside the repo at
  `/opt/stage9software/mosquitto/`.

## Secrets

YAML secrets files (`esphome/config/secrets.yaml`,
`homeassistant/config/secrets.yaml`) are referenced via `!secret <name>` in
the configs. `.env`-style files are gitignored (`.env.example` documents
`POSTGRES_DB`/`POSTGRES_USER`/`POSTGRES_PASSWORD`).

**Warning — real credentials are committed, not placeholders:**
- `esphome/config/secrets.yaml` — real-looking `wifi_ssid`, `wifi_password`,
  `api_key` (ESPHome native API encryption key), `ota_password`.
- `esphome/config/archive/sunsynk.yaml` — an older config with an `api_key`
  and `ota_password` hardcoded **inline** (not even via `!secret`), plus a
  second WiFi fallback-AP password. Same exposure, different file — don't
  miss it when rotating.
- `homeassistant/config/secrets.yaml` only has one placeholder-looking entry
  (`some_password`) — appears to be the stock template, not a real secret,
  but wasn't traced to any usage in `configuration.yaml`/automations.

**Warning — `.gitignore` doesn't match the real paths and is itself
uncommitted.** `.gitignore` excludes `homeassistant/.storage/`,
`homeassistant/*.log`, etc., but the actual directory is
`homeassistant/config/.storage/` (one level deeper) — so the patterns never
matched. `git ls-files` confirms `homeassistant/config/.storage/auth`,
`.storage/auth_provider.homeassistant`, `.storage/http.auth`,
`home-assistant.log*`, and `home-assistant_v2.db` are all **tracked in git
history**, along with both `secrets.yaml` files. Note: adding/fixing
`.gitignore` now will NOT untrack files already committed — that needs an
explicit `git rm --cached` pass (not done here; this is a read-only audit).
Treat this repo's git history as sensitive regardless of working-tree state.

Rotate all of the above before making this repo non-private, or move the
values out to an untracked secrets store.

**Verified contents (2026-08-12 review)** — these are live credentials, not
fixtures. `.storage/auth` holds **2 refresh tokens** (one `system`, one `normal`
for `http://localhost:8123/`), each with a 128-char `token` and a 128-char
`jwt_key`; `.storage/auth_provider.homeassistant` holds the `admin` user's bcrypt
hash. A refresh token plus its `jwt_key` mints working access tokens and **never
expires on its own** — so changing the HA admin password does not revoke them.
Revoking requires stopping HA, deleting `.storage/auth`, and restarting.

**Status: deliberately deferred.** The repo owner has chosen to defer secret
containment and rotation to a later phase, together with API authentication. This
is a known state, not an oversight. The full severity table and an ordered
rotation checklist live in `README.md` under "Security posture" — follow that
rather than improvising, because the HA token-revocation step is easy to get wrong.

Also note every compose service binds `0.0.0.0`: Postgres `5432` (with the
committed `synkflow_dev_password` default when `.env` is absent), Mosquitto
`1883` anonymous, HA `8123`, ESPHome dashboard `6052`, API `5174`.

## Environments

Single local environment only — no dev/staging/prod split lives here.

## CI/CD

None in this repo.

## Sibling projects

- `../SynkFlowAPI` — source for the backend; this repo runs a *built* image of
  it (`registry.stage9software.com/synkflow-api:latest`) plus its Postgres DB
  and consumes MQTT from Mosquitto. If `SynkFlowAPI` source changes, this repo
  doesn't rebuild automatically — it just pulls whatever tag `:latest` points
  to on `docker compose pull`.
- `../SynkFlow` — Flutter client that talks to the API (exposed here on host
  port `5174`, mapped to container port `8080`).

## Unverified / not checked

- Whether `homeassistant/config/automations.yaml` / `scripts.yaml` /
  `scenes.yaml` currently define anything — at time of writing all three are
  effectively empty (`automations.yaml` is `[]`).
- ~~What MQTT topics `synkflow-api` subscribes to~~ — **resolved 2026-08-12**:
  it subscribes to `sunsynk/sensor/+/state` and publishes nothing. The
  `sunsynk_` prefix is stripped during mapping, and only the 41 sensor names in
  `SensorMapper.TopicMap` are kept; everything else is silently dropped.
- Which of the two conflicting DE/RE pin values (GPIO 0 vs GPIO 18) reflects
  the physically wired hardware.
