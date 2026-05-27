# Claude Usage Monitor — CYD

A standalone desk display that shows your Claude Code API usage in real time.
Built for the **ESP32-2432S028R** ("Cheap Yellow Display") — a ~$18 all-in-one
ESP32 board with a 2.8" touchscreen TFT. No laptop server, no extra hardware,
no ongoing maintenance. Just power it on and it sits on your desk showing you
how much of your Claude quota you've used.

---

## What it shows

![Claude Usage Monitor running on the CYD desk display](preview.webp)

## How it works

The ESP32 makes a minimal HTTPS request to `api.anthropic.com/v1/messages`
every 60 seconds (configurable), sending a 1-token probe message:

```json
{ "model": "claude-haiku-4-5-20251001", "max_tokens": 1, "messages": [{"role": "user", "content": "."}] }
```

Anthropic's API returns rate-limit headers with every response:

```
anthropic-ratelimit-unified-5h-utilization: 0.24
anthropic-ratelimit-unified-5h-reset: 1779792000
anthropic-ratelimit-unified-7d-utilization: 0.04
anthropic-ratelimit-unified-7d-reset: 1780322400
```

The device reads these headers, converts utilisation (0.0–1.0) to a percentage,
and updates the display. The probe costs roughly 30 input tokens and 1 output
token — about $0.003/month at 60-second intervals, well under 1% of your quota.

Your OAuth token is stored in NVS flash, XOR-obfuscated using the device's
unique MAC address as a key. It is never transmitted anywhere except directly
to `api.anthropic.com` over HTTPS with certificate verification.

---

## Hardware

| Part | Price | Link |
|---|---|---|
| ESP32-2432S028R (CYD) | ~$17 | [Amazon — DIYmall](https://www.amazon.com/DIYmall-ESP32-2432S028R-Dual-core-240X320-Display/dp/B0BVFXR313) |

---

## 3D Printed Case

The CYD fits neatly into a two-piece printed case. You need two separate
models — one for the front shell and one for the back/stand:

### Front shell
**ESP32 2.8" CYD Screen Case (USB-C mod)**
Printables: https://www.printables.com/model/691234

Clean front bezel that frames the 2.8" screen. This version includes a
modification for a USB-C port adapter so the cable exits more neatly than
the stock micro-USB orientation.

### Back and stand
**ESP32 2.8" CYD Screen Case (back + stand)**
Printables: https://www.printables.com/model/645166

Snaps onto the front shell and includes an angled desk stand so the display
sits at a comfortable viewing angle on your desk.


---

## Software setup

### 1. Install Arduino IDE

Download from [arduino.cc](https://www.arduino.cc/en/software).

### 2. Add ESP32 board support

Open Arduino IDE → **File → Preferences**, and add this URL to
*Additional boards manager URLs*:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Then go to **Tools → Board → Boards Manager**, search **esp32**, and install
the package by Espressif Systems.

### 3. Install libraries

Go to **Sketch → Include Library → Manage Libraries** and install:

- **TFT_eSPI** by Bodmer
- **ArduinoJson** by Benoit Blanchon

### 4. Configure TFT_eSPI for the CYD

Copy `User_Setup.h` from this folder into your TFT_eSPI library directory,
replacing the existing file:

```bash
# Linux / Mac
cp User_Setup.h ~/Arduino/libraries/TFT_eSPI/User_Setup.h

# Windows
# Copy to: Documents\Arduino\libraries\TFT_eSPI\User_Setup.h
```

This tells TFT_eSPI which pins the CYD uses. Without this step the display
will not work.

### 5. Open the sketch

In Arduino IDE: **File → Open** → select the `CYD_ClaudeUsage_ino` folder.
All `.ino` tabs load automatically.

### 6. Select board and port

- **Tools → Board → ESP32 Arduino → ESP32 Dev Module**
- **Tools → Partition Scheme → Huge APP (3MB No OTA)**
  *(required — the sketch is too large for the default partition)*
- **Tools → Port → /dev/ttyUSB0** (Linux) or **COMx** (Windows)

### 7. Upload

Click the **Upload** button (→). The IDE compiles and flashes the firmware.
While uploading, the CYD's blue LED flashes rapidly — this is normal.

If the upload fails with "chip stopped responding":
- Try a different USB cable (many micro-USB cables are charge-only)
- Plug directly into your PC, not a USB hub
- Lower upload speed: **Tools → Upload Speed → 115200**
- Hold the BOOT button on the CYD during the "Connecting..." phase

---

## First boot — provisioning

On first boot (or after a factory reset), the CYD creates a WiFi hotspot:

```
SSID:     ClaudeMonitor-XXXX   (unique to your device)
Password: shown on screen      (random 8 characters)
```

**Steps:**

1. Connect your phone or laptop to the `ClaudeMonitor-XXXX` hotspot
2. A setup page opens automatically — if not, navigate to **192.168.4.1**
3. The page scans nearby WiFi networks and shows them as a dropdown
4. Select your network (2.4 GHz only — the ESP32 does not support 5 GHz)
5. Enter your WiFi password
6. Get your OAuth token by running this in a terminal on your laptop:
   ```bash
   claude setup-token
   ```
   This opens a browser, asks you to log in to Claude, and prints a token
   starting with `sk-ant-oat01-`. Copy the full token.
7. Paste the token into the setup page
8. Choose refresh interval and brightness
9. Click **Save & Reboot**

The device saves everything and reboots. From now on it connects automatically
on every boot — no setup page again unless you factory reset.

---

## Normal use

After provisioning, every boot goes:

```
Initializing → Config loaded → Connecting WiFi → Syncing time → Fetching usage → Dashboard
```

Takes about 5–10 seconds. Then the display refreshes automatically on the
interval you chose (default 60 seconds).

---

## Buttons

| Action | Result |
|---|---|
| Tap BOOT (GPIO 0) | Cycle screen brightness (4 levels: off → dim → normal → bright) |
| Hold BOOT 5 seconds | Factory reset — wipes credentials, reboots into setup mode |
| Tap GPIO35 button | Force refresh now |

**Hold countdown:** when you hold BOOT, the footer turns red and shows
"Keep holding Xs to factory reset..." — release any time before 5 seconds
to cancel with no effect.

---

## Changing WiFi or token

Hold the BOOT button for 5 seconds on the dashboard. The device wipes all
saved credentials and reboots into setup mode, showing the hotspot screen.
Go through first-boot provisioning again with your new details.

---

## File structure

```
CYD_ClaudeUsage_ino/
├── CYD_ClaudeUsage.ino   Main sketch — setup(), loop(), HAL, config, shared structs
├── api.ino               Fetches usage from Anthropic rate-limit headers
├── crypto.ino            XOR obfuscation of token using device MAC as key
├── ui.ino                All display drawing, flicker-free partial redraws
├── provision.ino         WiFi AP + captive portal web server for first-time setup
└── User_Setup.h          TFT_eSPI pin configuration for the CYD
```

### Tab responsibilities

**`CYD_ClaudeUsage.ino`** — the entry point. Defines all shared structs
(`UsageData`, `EncryptedBlob`), `#include`s all libraries, initialises the
display and buttons, runs the boot sequence, and handles the button logic in
`loop()`.

**`api.ino`** — makes the HTTPS probe request and parses the four rate-limit
headers from the response. Uses a pinned GlobalSign root CA certificate for
TLS verification rather than skipping it entirely.

**`crypto.ino`** — `encryptToken()` XOR-obfuscates the OAuth token with the
device's WiFi MAC address before writing to NVS. `decryptToken()` reverses it
on boot. Not military-grade, but prevents the token sitting in plain text in
flash memory.

**`ui.ino`** — draws the dashboard using a flicker-free technique: the static
chrome (header, labels, bar tracks, divider, footer) is drawn once and cached.
Subsequent updates only repaint the regions that changed — bar fills, percentage
text, countdown timers, and the header timestamp — with no `fillScreen()` call,
eliminating the black flash between refreshes.

**`provision.ino`** — when no credentials are saved, starts a WiFi access point
and hosts a captive portal web server. The setup page scans nearby networks
and presents them as a dropdown. Credentials are validated in the browser before
submission. After saving, the token is encrypted and stored in NVS, then the
device reboots.

