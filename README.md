<p align="center">
  <img src="assets/birdlogo.png" alt="M5 Atom Echo RTSP Mic for BirdNET-Go / BirdNET-Pi" width="260" />
</p>

# birdnet-esp32-rtsp-mic

M5Stack Atom Echo (also sold as Atom Voice) network microphone for **BirdNET-Go** and **BirdNET-Pi**.
It reads the board's onboard PDM MEMS microphone and serves mono **16-bit PCM/L16** audio over
**RTSP**.

This is a personal fork of [esp32-birdnet-mic](https://github.com/Sukecz/esp32-birdnet-mic) by
**@Sukecz**, retargeted from the original's Seeed XIAO ESP32 boards (with external I2S microphones)
to a single M5Stack Atom Echo with its onboard PDM microphone. It does not carry the upstream
project's web flasher or automatic-OTA infrastructure; flash and update it directly with the Arduino
IDE or `arduino-cli`.

- Target sketch: `esp32-birdnet-mic`
- Detailed firmware docs: `esp32-birdnet-mic/README.md`
- Changelog: `esp32-birdnet-mic/CHANGELOG.md` (history predates the Atom Echo port)
- License: MIT (`LICENSE`)

## Quick Start

1. Install the ESP32 Arduino core (pinned version in `build-dependencies.json`) and select board
   **M5Atom** (FQBN `esp32:esp32:m5stack_atom`).
2. Compile and upload `esp32-birdnet-mic/esp32-birdnet-mic.ino` over USB.
3. On first boot, connect to Wi-Fi AP **ESP32-RTSP-Mic-AP**.
4. Open `http://192.168.4.1` if the captive portal does not appear, then enter your Wi-Fi details.
5. After reboot, open the device Web UI at `http://<device-ip>/`.
6. Use these RTSP URLs in BirdNET-Go, BirdNET-Pi, VLC, or ffplay:

```text
rtsp://<device-ip>:8554/audio1    Stream 1
rtsp://<device-ip>:8554/audio2    Stream 2
```

If mDNS is enabled and works on your LAN, the same streams are available through the device hostname:

```text
rtsp://<device-hostname>.local:8554/audio1
rtsp://<device-hostname>.local:8554/audio2
```

The default hostname is unique per device, for example `esp32mic-a1b2c3`.

Only one RTSP client is accepted at a time (on either stream).

Build notes:

- OTA has no password in the default build. Keep the device on your trusted LAN.
- For OTA without USB, open `http://<device-ip>/ota` and upload a `.bin` you built yourself; there
  is no hosted auto-update server in this fork.

## Hardware

The Atom Echo's onboard **SPM1423 PDM microphone** requires no wiring and has no format setting —
unlike the original project's external I2S mic support, this is fixed by the board:

| Signal | GPIO |
|---|:--:|
| PDM clock | 19 |
| PDM data (mic in) | 23 |

The board has a fixed onboard PCB antenna; there is no external antenna connector or
firmware-controlled RF switch.

## Test The Stream

Use `/audio1` for stream 1 unless you specifically need the old `/audio` URL.

```bash
ffplay -rtsp_transport tcp rtsp://<device-ip>:8554/audio1
ffprobe -rtsp_transport tcp rtsp://<device-ip>:8554/audio1
```

For VLC: *Media* -> *Open Network Stream* -> paste `rtsp://<device-ip>:8554/audio1` or
`rtsp://<device-ip>:8554/audio2`.

If VLC/ffplay works, use the same RTSP URL in BirdNET-Go or BirdNET-Pi.

## What The Firmware Provides

- English Web UI on port **80** with live status, settings, logs, and actions.
- JSON API for automation and monitoring.
- Two RTSP streams: `/audio1` and `/audio2`; `/audio` is an alias for `/audio1`.
- One RTSP client at a time, on either stream.
- Per-stream enable/disable and BirdNET-Go / BirdNET-Pi target selection.
- Transport selection based on target: TCP for BirdNET-Go, UDP for BirdNET-Pi.
- Dedicated audio producer task with ring buffer between I2S capture and RTSP packet output.
- Audio/API diagnostics for I2S errors, ring-buffer drops, and RTSP write stalls/timeouts.
- MQTT telemetry and Home Assistant MQTT Discovery.
- Stream schedule by local time, including overnight windows.
- Optional deep sleep outside the stream schedule window.
- Auto-recovery, scheduled reset, CPU frequency control.
- Thermal protection with persistent latch and manual acknowledgement.
- Configurable high-pass filter for low-frequency rumble.
- mDNS hostname support and manual OTA upload.

Default audio settings are 48 kHz, mono 16-bit PCM/L16, gain 1.5, and a 512-sample packet buffer.
Bench-verify the PDM microphone path holds 48 kHz cleanly on your unit; lower the sample rate if not.

## Practical Tips

- Aim for Wi-Fi RSSI better than about **-75 dBm**.
- Use DHCP reservation or fixed IP if BirdNET cannot resolve mDNS reliably.
- Keep the device on a trusted LAN; do not expose HTTP/RTSP to the public internet.

### RF Noise / Wi-Fi TX Power

If the stream is stable but the audio is noisy or distorted, try lowering **Wi-Fi TX Power** in the
Web UI first.

Suggested test:

1. Set Wi-Fi TX Power to about **11 dBm**.
2. Watch RSSI, packet rate, reconnects, and audio quality for a few minutes.
3. If Wi-Fi becomes unstable, raise TX power step by step.

### High-Pass Filter

The firmware includes a configurable high-pass filter to reduce low-frequency rumble.

- Default: ON at 500 Hz.
- Typical range: 300-800 Hz.
- Web UI: Audio -> `High-pass` and `HPF Cutoff`.
- API: `POST /api/set` with `key=hp_enable&value=on|off` or `key=hp_cutoff&value=600`.
- Mutating API calls require header `X-ESP32MIC-CSRF: 1`.

## Compatibility

- Target board: M5Stack Atom Echo / Atom Voice only.
- The PDM microphone path is new to this fork and not yet runtime-validated at all sample rates.

## Arduino IDE Build Size

Firmware includes `esp32-birdnet-mic/build_opt.h`, which the ESP32 Arduino core loads automatically
in Arduino IDE and `arduino-cli`. It disables unused C++ exception/unwind metadata.

## More Documentation

- Firmware details, build instructions, API notes: `esp32-birdnet-mic/README.md`
- Firmware changes: `esp32-birdnet-mic/CHANGELOG.md`

## License

This project is released under the MIT License. See `LICENSE`.
