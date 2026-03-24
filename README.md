# SynkFlowInfrastructure

Docker Compose stack for the SynkFlow platform's infrastructure services: Home Assistant, ESPHome, and Mosquitto MQTT broker.

## Services

| Service | Image | Port | Description |
|---------|-------|------|-------------|
| Home Assistant | `ghcr.io/home-assistant/home-assistant:stable` | 8123 | Home automation platform |
| ESPHome | `ghcr.io/esphome/esphome` | 6052 | ESP device management |
| Mosquitto | `eclipse-mosquitto:2` | 1883 | MQTT broker |

All services run on the `synk-flow` Docker network, which is also used by [SynkFlowAPI](../SynkFlowAPI).

## Prerequisites

- Docker and Docker Compose

## Getting Started

```bash
# Start infrastructure (creates the synk-flow network)
docker compose up -d
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

## Directory Structure

```
├── docker-compose.yml
├── homeassistant/
│   └── config/              # Home Assistant configuration
├── esphome/
│   └── config/              # ESPHome device configurations
└── mosquitto/
    ├── config/              # mosquitto.conf
    ├── data/
    └── log/
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

### RS485 to Inverter
- Connect MAX485 **A** and **B** screw terminals to the Sunsynk **Meter-485** port (not the CAN/BMS port)
- If no communication, try **swapping A and B** wires

### ESPHome Config
- Config file: `esphome/config/sunsynk.yaml`
- Secrets file: `esphome/config/secrets.yaml`
- Modbus: 9600 baud, 8N1, address `0x00`
