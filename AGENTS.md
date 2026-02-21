# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## Project Overview

WLED is LED control firmware for ESP8266/ESP32 microcontrollers. It drives addressable LED strips (WS2812B, SK6812, etc.) with 100+ effects, a web UI, and integrations (MQTT, E1.31/Art-Net, Alexa, Hue, etc.).

## Build Commands

**Web UI assets** (must run first, or PlatformIO pre-build does it automatically):
```
npm install
npm run build          # compresses HTML/CSS/JS into C header arrays
npm run build -- -f    # force rebuild even if cached
```

**Firmware** (default target is esp32dev):
```
uv tool run --python python3.13 platformio run -e esp32dev
```
PlatformIO's espressif32 platform currently rejects Python 3.14, so use Python 3.13.

**Tests** (Node.js only, for the web UI build tooling):
```
npm test               # runs node --test (built-in test runner)
```
Test file: `tools/cdata-test.js`. No embedded C++ unit tests exist.

## Key Build Targets

| Environment | Chip |
|---|---|
| `esp32dev` | ESP32 (4MB flash, default) |
| `esp32dev_V4` | ESP32 with Arduino core v2 (IDF 4.4) |
| `esp32dev_8M` / `esp32dev_16M` | ESP32 with more flash |
| `esp32_eth` | ESP32 with Ethernet (WT32-ETH01) |
| `esp32_wrover` | ESP32-WROVER (PSRAM) |
| `esp32c3dev` | ESP32-C3 |
| `esp32s3dev_8MB_opi` / `esp32s3dev_16MB_opi` | ESP32-S3 |
| `nodemcuv2` | ESP8266 |

Build output goes to `build_output/release/`.

## Architecture

All firmware source is in `wled00/`. Key files:

- **wled.h**: Master header — all includes, global variables (via `WLED_GLOBAL` macro), feature flags (`WLED_DISABLE_*`), default constants
- **wled.cpp**: Main `WLED` class — `setup()`, main `loop()`, WiFi/connection handling
- **fcn_declare.h**: Forward declarations for all subsystem functions
- **FX.h/FX.cpp/FX_fcn.cpp**: Effect engine (`WS2812FX`), segment management, 100+ effects
- **bus_manager.h/cpp, bus_wrapper.h**: Hardware abstraction for LED outputs (digital, PWM, network)
- **json.cpp**: JSON API (state get/set via `/json/state`, `/json/info`)
- **cfg.cpp**: Config serialization — reads/writes `/cfg.json` and `/presets.json` on LittleFS
- **set.cpp**: HTTP parameter-based state mutations (legacy API)
- **wled_server.cpp**: ESPAsyncWebServer route registration

## Web UI Pipeline

`wled00/data/` contains the raw web assets. The build tool (`tools/cdata.js`) inlines external resources, minifies, GZIP-compresses, and hex-dumps them into C arrays:

- `data/index.htm` → `html_ui.h` (main UI, ~46KB compressed)
- `data/settings_*.htm` → `html_settings.h`
- `data/pixart/`, `data/cpal/`, `data/pxmagic/` → individual headers
- Other pages (404, liveview, update, etc.) → `html_other.h`

Version string and repository URL are injected from `package.json` during this process.

## Custom Build Scripts (pio-scripts/)

- **build_ui.py**: Pre-build hook that runs `npm install` + `npm run build`
- **set_metadata.py**: Injects version from package.json and GitHub repo URL (from git remote) into `wled_metadata.cpp` at compile time
- **output_bins.py**: Post-build — copies/renames firmware binaries
- **strip-floats.py**: Removes float printf/scanf to save flash on ESP8266

## Configuration Hierarchy

1. `platformio.ini` — base config (don't modify for personal boards)
2. `platformio_override.ini` — personal overrides (gitignored, see `platformio_override.sample.ini`)
3. `my_config.h` — user compile-time config (auto-copied from sample if missing)

## Feature Flags

Most features can be disabled via build flags to save flash (critical for ESP8266's ~1.5MB limit):

`WLED_DISABLE_OTA`, `WLED_DISABLE_ALEXA`, `WLED_DISABLE_MQTT`, `WLED_DISABLE_INFRARED`, `WLED_DISABLE_ESPNOW`, `WLED_ENABLE_DMX`, `WLED_ENABLE_WEBSOCKETS`, `WLED_ENABLE_ADALIGHT`

## Version

Single source of truth: `version` field in `package.json`. The build scripts propagate it to firmware metadata and web UI.
