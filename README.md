# Home Assistant & ESPHome

Docker Compose setup for running Home Assistant and ESPHome.

## Prerequisites

- Docker and Docker Compose installed
- Linux host recommended (for `network_mode: host` and `/run/dbus` support)

## Services

| Service | Port | Dashboard |
|---------|------|-----------|
| Home Assistant | 8123 | `http://<your-ip>:8123` |
| ESPHome | 6052 | `http://<your-ip>:6052` |

## Getting Started

```bash
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
.
├── docker-compose.yml
├── homeassistant/
│   └── config/          # Home Assistant configuration
└── esphome/
    └── config/          # ESPHome device configurations
```

Config directories are created automatically on first run.

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

## Notes

- Both services run with `network_mode: host` to enable mDNS discovery and local network access to ESP devices.
- Both services run in privileged mode for USB/hardware access.
- Services restart automatically unless explicitly stopped.
