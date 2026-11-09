# RemoteLED

QR-to-device payment activation system. A customer scans a QR code shown on a Raspberry Pi kiosk, picks a service (laundry, vending, air compressor, …), pays (Stripe or mock), and the Pi activates a GPIO relay/LED for the authorized duration. The backend signs each authorization with ECDSA (secp256k1); the Android app relays it to the Pi over Bluetooth Low Energy.

The repo contains five parts:
- **backend/** — Python FastAPI API (port 9999): devices, services, orders, payments, ECDSA authorizations, telemetry, admin.
- **frontend/admin-react/** — React 18 + TypeScript + Vite admin console (served at `/admin/`, dev port 5173).
- **database/** — PostgreSQL schema, seed data, migrations.
- **pi/** — Raspberry Pi BLE peripheral (Python) + web/React kiosk.
- **android/** — Kotlin Android app (QR scanner + BLE GATT client).

At the BLE layer the demo still works standalone: the Pi displays a deep link as a QR, the phone scans it, connects to the Pi's BLE peripheral, and toggles a GPIO with a short per-session key.

> Important: Android emulators do not support real Bluetooth LE. Use a physical Android device for end‑to‑end testing.

## Overview
- Architecture: Pi runs a BLE peripheral and displays a deep link QR code on a local web page. Android scans the QR or opens the deep link directly, connects via BLE, and writes commands to toggle a GPIO.
- Deep link format: `remoteled://connect/<MAC>/<svc16>/<char16>/<bleKey>`.
- BLE stack on Pi: **Python** (`pi/python/code.py`). A Node.js variant is referenced by some scripts/docs, but `pi/node/main.js` is not present in this repo, so only the Python implementation is functional.

## Repository Layout
- `backend/`: FastAPI API server (`app/main.py`), PostgreSQL, Stripe, ECDSA
- `frontend/admin-react/`: React + TypeScript + Vite admin console
- `database/`: PostgreSQL `schema.sql`, `seed.sql`, migrations
- `pi/`: Raspberry Pi provisioning and runtime
  - `install.sh`: One‑shot provisioning of Python deps, nginx, kiosk, and the Python systemd service. Note: it copies a `kiosk.sh` from the `pi/` dir that is **not checked into this repo** — supply one (Chromium kiosk launcher pointing at `http://localhost`) before running the installer on a Pi.
  - `python/`: Python BLE peripheral (`code.py`), `led_service.py`, `remoteled-python.service`, starter scripts
  - `node/`: Node BLE starter (`start.sh`) that execs `node main.js` — `main.js` is **not present**, so this path is non-functional as shipped
  - `web/`: Static kiosk (`index.html`) that renders the QR code
  - `kiosk/`: Alternative React (Create React App) kiosk
- `android/RemoteLedBLE/`: Android app (deep link + BLE GATT + QR scanner)
- `tests/`: Shell-script integration tests
- `docs/`: Architecture notes (`ARCHITECTURE.md`, `SYSTEM.md`)

## Python Dependency Management

This project uses [uv](https://github.com/astral-sh/uv) for Python package management with a single `pyproject.toml` at the root.

**Installation:**
- **On macOS (development)**: `uv sync`
- **On Raspberry Pi OS**: `uv sync --extra pi`

The Pi-specific dependencies (RPi.GPIO, bluezero, dbus-python) are only installed when using the `--extra pi` flag.

## Requirements
- Raspberry Pi with Bluetooth (tested on Raspberry Pi OS Bookworm, Wayfire/LXDE)
- Internet on first setup (to install packages)
- Android device (current app sets `minSdk=26`, i.e., Android 8.0+)

## End‑to‑End Quick Start

### A. Raspberry Pi setup
1. Copy the `pi/` directory to the Pi and run the installer:

   ```bash
   bash pi/install.sh
   ```

   What the installer does:
   - Installs Python BLE dependencies and nginx
   - Copies `pi/python/` and `splash.png` to `/usr/local/remoteled/`
   - Sets up the kiosk to show `http://localhost` in Chromium on boot (via a `kiosk.sh` you must provide, see Repository Layout)
   - Installs and enables the `remoteled-python.service` systemd unit

2. Enable the Python BLE service (uses GPIO BCM 17):

   ```bash
   sudo systemctl enable --now remoteled-python.service
   ```

3. Ensure web server is running:

   ```bash
   sudo systemctl enable --now nginx
   sudo systemctl status nginx | cat
   ```

4. Reboot, then verify the kiosk shows a QR:

   ```bash
   sudo reboot
   # After reboot, on the Pi screen you should see "Scan QR" and a code
   ```

5. View logs for the BLE service until you see a deep link being generated:

   ```bash
   journalctl -u remoteled-python.service -f
   ```

   You should see a line like:

   ```
   Generated Deep Link: remoteled://connect/AA:BB:CC:DD:EE:FF/abcd/ef01/1234
   ```

Hardware note: The LED/relay should be connected to ground and GPIO BCM 17 (Python). Add a series resistor for a bare LED.

### B. Android app setup
1. Use a physical device (Android 8.0+ / `minSdk=26`). Connect via USB with USB debugging enabled.
2. Open `android/RemoteLedBLE` in Android Studio.
3. Select your phone in the device dropdown and click Run. Grant Camera, Location, and Bluetooth permissions.
4. The app launches the QR scanner. Scan the kiosk QR.

Deep link alternative (without scanning):

```bash
adb shell am start -a android.intent.action.VIEW -d "remoteled://connect/AA:BB:CC:DD:EE:FF/abcd/ef01/1234"
```

The app targets `minSdk=26` (Android 8.0+). If your phone is older, lower `minSdk` in `android/RemoteLedBLE/app/build.gradle.kts` and rebuild.

## Quick Start (Raspberry Pi)
1) Prepare the Pi
- Update OS packages and ensure Bluetooth is enabled.
- Optional dependency: if `dos2unix` is not installed, run `sudo apt-get install dos2unix`.

2) Run the installer
- Copy this repo (or just the `pi/` folder) onto the Pi.
- Execute: `bash pi/install.sh`
- What it does:
  - Installs Python BLE deps and nginx
  - Copies code to `/usr/local/remoteled/`
  - Sets Chromium to autostart a kiosk at `http://localhost` and displays QR
  - Installs and enables `remoteled-python.service`

3) Reboot and verify
- After reboot, the kiosk should show “Scan QR” with a code.
- Check service: `sudo systemctl status remoteled-python`
- Tail logs: `sudo journalctl -u remoteled-python -f`

### Rapid Pi + Backend test loop
For the exact workflow documented in `~/Downloads/pi_ble_qr_setup.txt`, use the following checklist whenever you need to validate the entire BLE/QR flow quickly:

1. **Start the backend on your Mac**
   - From the repo root: `cd backend && uv run --no-project uvicorn app.main:app --reload --host 0.0.0.0 --port 9999`
   - Keep it running and confirm `http://<MAC_LAN_IP>:9999/health` responds (the IPs below are examples — use your own, e.g. `ipconfig getifaddr en0`).

2. **Launch the Pi BLE peripheral**
   - On the Pi: `cd ~/Documents/remoteled/pi/python && ./start.sh`
   - The script reads `/usr/local/remoteled/device_id`, activates `venv/`, and runs `code.py`.
   - Watch the terminal for `[BLE Debug] Detail URL ready for kiosk: http://192.168.1.158:9999/detail?...` and “Advertisement registered”.

3. **Refresh the kiosk view**
   - On the Pi desktop press `F5` in Chromium (already pointed at `http://localhost`), or relaunch the kiosk launcher (`/usr/local/remoteled/kiosk.sh`, the script you supplied at install time).

4. **Scan the QR from Android**
   - Once the QR is visible, scan it with the Android app. If the kiosk ever sticks on `CONNECTED!`, press `Ctrl+C` in the Pi terminal, rerun `./start.sh`, and refresh Chromium again.

## Quick Start (Android)
- Build from source: open `android/RemoteLedBLE` in Android Studio and run on a device (Android 8.0+ / `minSdk=26`). No prebuilt APK is committed — build your own (see Manual APK Build below).
- Launch the app and scan the QR on the Pi. The app also handles deep links of the form `remoteled://connect/...` if opened directly.
- The API base URL is baked in at build time via `API_BASE_URL` in `android/RemoteLedBLE/app/build.gradle.kts` (default `http://192.168.1.158:9999` — change to your Mac's LAN IP).

Emulator note: The Android emulator does not support Bluetooth LE and the app declares BLE as a required feature in the manifest. Installation and BLE will fail on the emulator—use a real device.

### Manual APK Build & Installation (macOS)

For building and installing the APK manually on a physical Android device using macOS:

**1. Install Java (OpenJDK 17)**
```bash
brew install openjdk@17
```

**2. Add Java to your shell config**

Add to `~/.zshrc` (or `~/.bashrc` if using bash):
```bash
export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"
export JAVA_HOME="/opt/homebrew/opt/openjdk@17"
```

Reload your shell:
```bash
source ~/.zshrc
```

**3. Update API endpoint**

Edit `android/RemoteLedBLE/app/build.gradle.kts` and set your Mac's local IP:
```kotlin
buildConfigField("String", "API_BASE_URL", "\"http://YOUR_MAC_IP:9999\"")
```

Find your Mac IP: `ipconfig getifaddr en0`

**4. Build the APK**
```bash
cd android/RemoteLedBLE
chmod +x gradlew
./gradlew assembleDebug
```

APK location: `app/build/outputs/apk/debug/app-debug.apk`

**5. Install MacDroid**

Download from [macdroid.app](https://www.macdroid.app/) to enable file transfers between Mac and Android.

**6. Transfer APK to phone**

Connect phone via USB, enable USB file transfer mode, then:
```bash
sudo cp app/build/outputs/apk/debug/app-debug.apk /Volumes/MacDroid/sdcard/Download/
```

**7. Install on phone**

On your Android device:
- Open Files app → Downloads
- Tap `app-debug.apk`
- Allow installation from unknown sources if prompted
- Install and launch

**8. Start backend server**
```bash
cd backend
uv run --no-project uvicorn app.main:app --reload --host 0.0.0.0 --port 9999
```

Your phone can now communicate with the backend at `http://YOUR_MAC_IP:9999`.

## Using It
- Commands written to the BLE characteristic are JSON:
  - `{"command":"CONNECT","bleKey":"xxxx"}`: verifies the session
  - `{"command":"ON","bleKey":"xxxx"}`: turns the LED on
  - `{"command":"OFF","bleKey":"xxxx"}`: turns the LED off
- The `bleKey` is a 4‑hex‑digit value generated per session and included in the deep link/QR.

## GPIO Pins
- Python implementation: uses BCM pin `17` (`RPi.GPIO`). Change in `pi/python/code.py` if needed.
- The payment flow uses three LEDs by default (BCM): green `17`, yellow `19`, red `27` (see `backend/.env.example`).

## Run Manually (without systemd)
- Python BLE peripheral:
  - `cd /usr/local/remoteled/python && source bt/bin/activate && python3 code.py`
  - (On a dev machine: `cd pi/python && python3 code.py`)
- Kiosk page (already served by nginx): `http://localhost` on the Pi

## Troubleshooting
- No QR on kiosk:
  - Check nginx: `systemctl status nginx`
  - Verify the Python BLE service is running
- Bluetooth won't power on:
  - `rfkill list`, then `sudo rfkill unblock bluetooth && sudo hciconfig hci0 up`
- Can’t see a deep link in logs:
  - Tail logs with `journalctl -u remoteled-python.service -f`
- Android cannot connect:
  - Ensure Bluetooth and Location permissions are granted
  - The app targets `minSdk=26` (Android 8.0+)
  - Confirm the phone's `API_BASE_URL` points at your running backend's LAN IP
- GPIO not toggling:
  - Confirm the pin numbers match your hardware (BCM 17 by default)

## How it works (one‑minute version)
- Pi generates random 16‑bit service/characteristic UUIDs and a short session key (`bleKey`), then displays a deep link as a QR code on the kiosk.
- The Android app scans the QR, reconstructs full 128‑bit UUIDs, connects over BLE, reads current state, then sends JSON commands including the `bleKey`.

## Security Notes
- Session key: `bleKey` is short (16‑bit) and rotates per session; fine for demos, not strong security. For production, use pairing or stronger authentication.
- Authorizations are ECDSA-signed (secp256k1) by the backend and verified on the device.
- Privileges: the BLE peripheral runs with `sudo` to access HCI. Consider a dedicated service user and Linux capabilities (`CAP_NET_RAW`, `CAP_NET_ADMIN`).

## Reference
- Web kiosk displays the deep link as a QR code.
- Systemd service: `remoteled-python.service`.

## Database Setup

RemoteLED uses PostgreSQL to store device information, services/products, orders, authorizations, and telemetry logs. The database supports the complete customer journey from QR scan to device activation.

### Quick Database Setup

1. **Install PostgreSQL 15+** (macOS with Homebrew):
   ```bash
   brew install postgresql@15
   export PATH="/opt/homebrew/opt/postgresql@15/bin:$PATH"
   brew services start postgresql@15
   ```

2. **Create and initialize the database**:
   ```bash
   createdb remoteled
   psql -d remoteled -f database/schema.sql
   psql -d remoteled -f database/seed.sql  # Optional: load test data
   ```

3. **Verify setup**:
   ```bash
   psql remoteled -c "SELECT * FROM v_devices_summary;"
   ```

### Database Schema Overview

The database consists of 6 main tables:

- **devices**: Raspberry Pi devices with their public keys for signature verification
- **services**: Products offered by each device (TRIGGER, FIXED, VARIABLE types)
- **orders**: Customer orders tracking the lifecycle (CREATED → PAID → RUNNING → DONE)
- **authorizations**: Cryptographically signed payloads sent to devices via BLE
- **logs**: Communication and telemetry logs between Pi and server
- **admins**: Administrative users managing the system

### Service Types

- **TRIGGER**: One-time activation (2 seconds) - e.g., vending machine dispense
- **FIXED**: Fixed duration - e.g., 40-minute laundry cycle ($2.50)
- **VARIABLE**: Pay-per-time - e.g., air compressor ($0.25 per 6 minutes)

### Order Lifecycle

```
CREATED → PAID → RUNNING → DONE
           ↓         ↓
         FAILED   FAILED
```

### Connection String

For application configuration:
```
postgresql://localhost:5432/remoteled
```

**📚 Complete database documentation**: See [`database/README.md`](database/README.md) for detailed schema, queries, and maintenance instructions.

## Testing

### Quick Testing Guide

**1. Install Dependencies**

macOS (development):
```bash
uv sync
```

Raspberry Pi:
```bash
uv sync --extra pi
```

**2. Start Pi BLE Peripheral**

On Raspberry Pi:
```bash
cd pi/python
python3 code.py
```

Note the UUIDs printed in the terminal:
```
Service UUID: 0000XXXX-0000-1000-8000-00805f9b34fb
Characteristic UUID: 0000YYYY-0000-1000-8000-00805f9b34fb
bleKey: ZZZZ
```

**3. Start FastAPI Backend**

On your development machine (the whole stack uses port 9999):
```bash
cd backend
uv run --no-project uvicorn app.main:app --reload --host 0.0.0.0 --port 9999
```

Backend starts on `http://localhost:9999` (interactive Swagger docs at `/docs`).

**4. Test LED Control directly (bypassing payment)**

Use the `/led/control` endpoint. `color` is `red|yellow|green`, `mode` is `on|off|blink`:

```bash
# GREEN LED on
curl -X POST http://localhost:9999/led/control \
  -H "Content-Type: application/json" \
  -d '{"color": "green", "mode": "on"}'

# RED LED blink
curl -X POST http://localhost:9999/led/control \
  -H "Content-Type: application/json" \
  -d '{"color": "red", "mode": "blink"}'

# All LEDs off
curl -X POST http://localhost:9999/led/control \
  -H "Content-Type: application/json" \
  -d '{"color": "green", "mode": "off"}'
```

**Expected Results:**
- Backend logs: `[LED Control] Request: GREEN ON`
- Hardware: Correct LED lights up (Green=GPIO17, Yellow=GPIO19, Red=GPIO27)

**Troubleshooting:**
- "Device not found": Ensure Pi is running `code.py` and Bluetooth is enabled
- "Module not found" errors: Run `uv sync` (or `uv sync --extra pi` on Pi)
- Database connection errors: Ensure PostgreSQL is running and the `remoteled` database exists

### Frontend (admin console)

```bash
cd frontend/admin-react
npm install
npm run dev        # http://localhost:5173/admin/ (proxies /api -> http://localhost:9999)
npm run build      # production build -> dist/
npm run test:unit  # vitest (note: script is `test:unit`, not `test`)
```

📚 **Detailed API documentation**: See [`backend/README.md`](backend/README.md) for the complete API reference, and [`frontend/admin-react/README.md`](frontend/admin-react/README.md) for the admin console.

## Further reading
- `docs/ARCHITECTURE.md` — component and data-flow overview
- `docs/SYSTEM.md` — detailed system design
- `backend/README.md` — API reference
- `database/README.md` — schema, queries, maintenance

## Notes
- No prebuilt Android APK is committed — build from source (see Manual APK Build above).
- More background and diagrams: see `docs/ARCHITECTURE.md` and `docs/SYSTEM.md`.
