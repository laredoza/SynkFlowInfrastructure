# SynkFlowInfrastructure

Local Docker Compose stack for the SynkFlow platform's on-prem services:
Mosquitto (MQTT broker), ESPHome (manages the Sunsynk RS485 bridge firmware),
and Home Assistant (automations over the same MQTT data).

> Not cloud infrastructure — no Terraform / Pulumi / Kubernetes here.
> Everything runs on a single LAN host.

## Services

| Service | Image | Port | Description |
|---------|-------|------|-------------|
| Home Assistant | `ghcr.io/home-assistant/home-assistant:stable` | 8123 | Home automation platform |
| ESPHome | `ghcr.io/esphome/esphome` | 6052 | ESP device management |
| Mosquitto | `eclipse-mosquitto:2` | 1883 | MQTT broker |

All services run on the `synk-flow` Docker network, also used by
[SynkFlowAPI](../SynkFlowAPI).

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

## Sunsynk Inverter — ESP32 RS485 Wiring

### Hardware
- **ESP32:** NodeMCU ESP32 (USB-C) — `esp32dev`
- **RS485 Module:** MAX485 TTL-to-RS485 converter (blue PCB)
- **Inverter:** Sunsynk 8kW, Modbus address `0x00`

### Wiring

| MAX485 Pin | ESP32 Pin | Wire Color | Notes |
|------------|-----------|------------|-------|
| DI (Data In) | P16 (GPIO 16) | Purple | UART TX — sends data to inverter |
| RO (Receive Out) | P4 (GPIO 4) | Blue | UART RX — receives data from inverter |
| DE + RE (tied) | P0 (GPIO 0) | Black (via breadboard row 62) | Flow control — **GPIO 0 is a boot pin, consider moving to P5** |
| VCC | 5V | Black | Power |
| GND | GND | Brown | Ground |

### RS485 to inverter
- Connect MAX485 **A** and **B** screw terminals to the Sunsynk
  **Meter-485** port (not the CAN/BMS port).
- If no communication, try **swapping A and B** wires.

### ESPHome config
- Config file: `esphome/config/sunsynk.yaml`
- Secrets file: `esphome/config/secrets.yaml`
- Modbus: 9600 baud, 8N1, address `0x00`
- A fallback WiFi AP is configured for recovery if the configured SSID is
  unreachable.

## Secrets

`!secret` references resolve against `esphome/config/secrets.yaml` and
`homeassistant/config/secrets.yaml`.

> **Warning:** `wifi_password`, `api_key`, and `ota_password` are currently
> committed in the ESPHome `secrets.yaml`. Rotate them before sharing this
> repository, or move them out to an untracked secrets store.

## Networking

The Mosquitto address is hardcoded in `esphome/config/sunsynk.yaml` as
`192.168.0.50:1883`. The host running this stack must be reachable at that IP
from the ESP32 on the LAN, and from [SynkFlowAPI](../SynkFlowAPI).

## Related projects

- [SynkFlow](../SynkFlow) — Flutter client
- [SynkFlowAPI](../SynkFlowAPI) — backend API
