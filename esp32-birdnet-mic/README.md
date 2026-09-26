<p align="center">
  <img src="../assets/birdlogo.png" alt="M5 Atom Echo RTSP Mic for BirdNET-Go / BirdNET-Pi" width="240" />
</p>

# birdnet-esp32-rtsp-mic Firmware

Arduino firmware for the **M5Stack Atom Echo** (also sold as **Atom Voice**) that reads its onboard
**SPM1423 PDM microphone** and serves **mono 16-bit PCM/L16** audio over **RTSP** for **BirdNET-Go**
and **BirdNET-Pi**. It also provides a Web UI, JSON API, MQTT telemetry, and Home Assistant MQTT
Discovery.

This is a personal fork retargeted to a single board. It is based on the multi-board
[esp32-birdnet-mic](https://github.com/Sukecz/esp32-birdnet-mic) project by **@Sukecz**, which
targets Seeed XIAO ESP32 boards with external I2S microphones; see that project for the original
multi-board web flasher and OTA release process, which this fork does not carry.

- Build target: **M5Stack Atom Echo / Atom Voice** (ESP32-PICO-D4)
- Microphone: onboard **SPM1423 PDM** mic (no wiring, no format selection)
- Changelog: `CHANGELOG.md` (history predates the Atom Echo port; see git log for this fork's changes)
- License: MIT (`../LICENSE`)

## Important URLs

Web UI:

```text
http://<device-ip>/
```

RTSP streams:

```text
rtsp://<device-ip>:8554/audio1    Stream 1
rtsp://<device-ip>:8554/audio2    Stream 2
```

mDNS variants, if enabled and supported by your network:

```text
rtsp://<device-hostname>.local:8554/audio1
rtsp://<device-hostname>.local:8554/audio2
```

The API and Web UI publish `/audio1` and `/audio2`. Use `/audio1` in new configurations. `/audio`
remains available only as a compatibility alias for stream 1. Only one RTSP client is accepted at a
time (on either stream).

## First Boot

1. Build and flash the sketch (see **Build And Flash Manually** below).
2. The device starts WiFiManager AP **ESP32-RTSP-Mic-AP**.
3. Connect to the AP and open `192.168.4.1` if the captive portal does not open automatically.
4. Save Wi-Fi credentials.
5. After reboot, open `http://<device-ip>/`.

Default hostname is unique per device, for example `esp32mic-a1b2c3`.

## Hardware

The Atom Echo's SPM1423 microphone is a PDM part wired internally to fixed GPIOs; there is nothing
to wire up and no microphone-format setting:

| Signal | GPIO | Notes |
|---|:--:|---|
| PDM clock | 19 | Shared with the board's internal speaker bus |
| PDM data (mic in) | 23 | Dedicated microphone data line |

GPIO22 (speaker output) and GPIO33 (shared bus LRCLK) are used by the board's onboard speaker path
and are not touched by this firmware. The board has a fixed onboard PCB antenna; there is no
external antenna connector or firmware-controlled RF switch.

## Runtime Defaults

- Sample rate: 48 kHz (bench-verify this holds cleanly over PDM on your unit; drop it if not)
- Audio format: mono 16-bit PCM/L16
- Gain: 1.5
- Buffer: 512 samples
- I2S shift: 0 bits (PDM RX already delivers full-scale 16-bit PCM)
- High-pass filter: ON, 500 Hz
- Wi-Fi TX power: about 19.5 dBm
- CPU: 160 MHz
- Thermal shutdown: 80 C, protection ON
- Max RTSP clients: 1
- mDNS: ON
- Time sync: ON
- OTA password: none by default

The sample-rate setting accepts 8,000-192,000 Hz in the UI/API, but PDM RX on the ESP32-PICO-D4 may
not reliably hold the higher end of that range; verify on real hardware before relying on it.

## Web UI

The Web UI runs on port **80** and includes:

- Status: IP, RSSI, uptime, heap, server state, stream states, packet rates.
- Streams: URLs for `/audio1` and `/audio2`, enable/disable, BirdNET target.
- Audio: sample rate, gain, buffer size, I2S shift, high-pass filter, signal level.
- Audio API diagnostics: producer state, ring-buffer capacity/chunks/drops/flushes, I2S errors,
  and RTSP write stalls/timeouts.
- Time & Network: NTP state, time offset, mDNS, stream schedule, optional deep sleep, Wi-Fi actions.
- Reliability: auto-recovery, threshold mode, check interval, scheduled reset.
- Thermal: current/peak temperature, shutdown limit, protection latch, acknowledgement.
- MQTT & Home Assistant: broker settings, publish interval, discovery republish.
- Firmware update: open `/ota` or click **Firmware update** in the Web UI.

## Firmware Update Without USB

Open:

```text
http://<device-ip>/ota
```

This personal build has no hosted update server, so the automatic "check for update" path is
disabled. Use **Upload compiled file** to push a `.bin` you built yourself with `arduino-cli compile
--output-dir` or the Arduino IDE's "Export compiled binary."

- Logs: ring buffer view and download.
- Actions: RTSP server ON/OFF, reset I2S, reconnect Wi-Fi, reboot, restore defaults.

## Stream Behavior

- Stream 1: `/audio1`; `/audio` is a compatibility alias for the same stream.
- Stream 2: `/audio2`.
- Each stream can be enabled or disabled.
- Each stream can target BirdNET-Go or BirdNET-Pi.
- BirdNET-Go target uses RTP over RTSP/TCP.
- BirdNET-Pi target uses RTP/UDP when the client provides UDP ports.
- RTSP keep-alive via `GET_PARAMETER` is supported.
- Inactive non-streaming sessions time out after about 30 seconds.
- Only one client slot exists; a second connection attempt is refused until the first disconnects.

### Verify With ffmpeg Tools

```bash
ffplay -rtsp_transport tcp rtsp://<device-ip>:8554/audio1
ffprobe -rtsp_transport tcp rtsp://<device-ip>:8554/audio1
```

For stream 2, replace `/audio1` with `/audio2`. If VLC/ffplay works, use the same URL in
BirdNET-Go or BirdNET-Pi.

## Network And Time

- Wi-Fi power save is disabled with `WiFi.setSleep(false)` for stable streaming.
- WiFiManager AP: `ESP32-RTSP-Mic-AP`.
- WiFiManager connect timeout: 60 s.
- WiFiManager portal timeout: 180 s.
- mDNS can be enabled/disabled in the Web UI.
- Hostname can be changed via API: `key=mdns_hostname&value=esp32mic-garden`.
- mDNS often fails on isolated/guest Wi-Fi or inside Docker containers; use device IP in those cases.
- NTP sync runs on boot, retries every hour until synced, then refreshes every 6 hours.
- If time is unavailable, logs fall back to uptime timestamps.

### Stream Schedule And Deep Sleep

Stream schedule is configured in Time & Network.

- Cross-midnight windows are supported, for example `22:00-06:00`.
- If time is invalid, schedule policy is fail-open: streaming stays allowed.
- If start and stop are equal, the window is explicitly empty and streaming is blocked.
- Optional deep sleep can run outside the stream window only when time is valid.
- Deep sleep is blocked during startup grace, with active clients, or without valid time.

API keys:

```text
stream_sched=on|off
stream_start_min=<0..1439>
stream_stop_min=<0..1439>
deep_sleep_sched=on|off
```

## JSON API

The API mirrors the Web UI. Inspect browser DevTools -> Network for exact calls.

Important read endpoints:

```text
GET /api/status
GET /api/audio_status
GET /api/perf_status
GET /api/thermal
GET /api/logs
```

Mutating calls use `POST` and require header `X-ESP32MIC-CSRF: 1`.

Common settings endpoint:

```text
POST /api/set
```

Examples of request bodies:

```text
key=stream1_enabled&value=on
key=stream2_enabled&value=off
key=stream1_target&value=0
key=stream2_target&value=1
key=hp_enable&value=on
key=hp_cutoff&value=600
```

Target values:

```text
0 = BirdNET-Go
1 = BirdNET-Pi
```

`/api/status` includes stream URLs and state, including:

```text
stream1_url_ip
stream2_url_ip
stream1_url_mdns
stream2_url_mdns
stream1_enabled
stream2_enabled
stream1_target
stream2_target
s1_clients
s2_clients
s1_streaming
s2_streaming
s1_pkt_rate
s2_pkt_rate
```

`/api/audio_status` includes audio pipeline diagnostics:

```text
producer_running
i2s_error_count
rb_capacity_bytes
rb_chunks
rb_drops
rb_flushes
rtsp_write_stalls
rtsp_write_timeouts
```

## MQTT And Home Assistant

MQTT settings are available in the Web UI and API.

- Telemetry topic: `<topic_prefix>/state`
- Availability topic: `<topic_prefix>/availability`
- RTSP server command: `<topic_prefix>/cmd/rtsp_server` with `ON` or `OFF`
- Reboot command: `<topic_prefix>/cmd/reboot` with `PRESS` or `REBOOT`
- Publish interval: default 60 s, range 10-3600 s
- Immediate publishes happen on important events such as stream start/stop and connection changes.

Home Assistant MQTT Discovery creates entities for runtime values such as RSSI, uptime, heap,
temperature, stream state, packet rate, client count, reboot reason, restart counter, firmware
version/build, and Wi-Fi reconnect count.

MQTT password is stored in device flash in plain text NVS.

## Build And Flash Manually

### Arduino IDE

1. Open `esp32-birdnet-mic/esp32-birdnet-mic.ino`.
2. Install the ESP32 Arduino core (this fork is pinned to core `3.3.8`; see `../build-dependencies.json`).
3. Select board **M5Atom** under ESP32 Arduino boards (FQBN `esp32:esp32:m5stack_atom`; there is a
   separate **M5AtomS3** entry for the S3 variant — Atom Echo is the original ESP32-PICO-D4 board).
4. Compile and upload over USB.

The sketch folder contains `build_opt.h`, which Arduino IDE uses automatically to trim unused
exception/unwind metadata.

### arduino-cli

```bash
arduino-cli compile --fqbn esp32:esp32:m5stack_atom esp32-birdnet-mic
arduino-cli upload -p <PORT> --fqbn esp32:esp32:m5stack_atom esp32-birdnet-mic
```

### PlatformIO

```ini
[env:m5stack-atom]
platform = espressif32
board = m5stack-atom
framework = arduino
```

```bash
pio run -t upload
```

## Web UI Development

The Web UI source is `webui/index.html`. Firmware serves the compressed generated header
`WebUI_gz.h` from PROGMEM.

After editing the UI, regenerate the header:

```bash
./tools/gen_webui_gzip_header.sh
```

## Persisted Configuration

Most runtime settings are stored in NVS namespace `audio` through ESP32 Preferences.

Main keys:

```text
sampleRate       Audio sample rate
gainFactor      Audio gain
bufferSize       Samples per packet/buffer profile
shiftBits        Pre-gain right shift
hpEnable         High-pass enable
hpCutoff         High-pass cutoff Hz
wifiTxDbm        Wi-Fi TX power
mdnsEn           mDNS enable
timeSyncEn       NTP enable
timeOffset       Local offset in minutes
strSchedEn       Stream schedule enable
strSchStart      Stream window start minute
strSchStop       Stream window stop minute
deepSchSlp       Deep sleep outside schedule window
autoRecovery     Packet-rate recovery enable
thrAuto          Automatic threshold mode
minRate          Manual minimum packet rate
checkInterval    Recovery check interval in minutes
schedReset       Scheduled reset enable
resetHours       Scheduled reset interval
ohEnable         Thermal protection enable
ohThresh         Thermal shutdown threshold C
ohLatched        Persisted thermal latch
```

Apply changes through Web UI or API. Audio-related updates call `restartI2S()` when needed.
Current validation ranges include `sampleRate=8000..192000` and `bufferSize=256..8192`.

## RTSP Implementation Notes

- I2S capture and audio processing run in a producer task.
- RTSP output consumes processed PCM blocks from a FreeRTOS ring buffer and packetizes them as RTP.
- `DESCRIBE` returns SDP with `a=rtpmap:96 L16/<sample-rate>/1` and `a=control:track1`.
- Stream selection is path-based: `/audio1` and `/audio` select stream 1; `/audio2` selects stream 2.
- `SETUP` returns either `RTP/AVP/TCP;unicast;interleaved=0-1` or UDP ports, depending on target.
- `PLAY` starts RTP packet output for that session.
- `TEARDOWN` stops the session.
- RTP timestamp increments by the number of audio samples per packet.

## Stability Notes

- Aim for Wi-Fi RSSI better than about **-75 dBm**.
- Increase buffer size in RF-noisy environments; this adds latency but improves stability.
- If audio is noisy while Wi-Fi is otherwise stable, try lowering Wi-Fi TX power before changing audio settings.
- Thermal protection disables RTSP when the chip reaches the configured limit; the latch survives reboot until acknowledged in the Web UI.

## Security

- Keep the device on a trusted LAN.
- Do not expose HTTP, RTSP, or OTA to the public internet.
- The default build has no OTA password.
- If you build your own firmware for untrusted networks, set an OTA password before publishing it.
- Mutating API endpoints require `POST` and `X-ESP32MIC-CSRF: 1`, but read endpoints are not globally authenticated by default.

## Known Limitations

- No TLS or built-in user authentication for the Web UI/API.
- mDNS depends on multicast support in your LAN and often does not work across VLANs, guest networks, or Docker bridge networks.
- The PDM microphone path is new to this fork and has not yet been runtime-validated at all sample
  rates; bench-test before relying on it.

## Credits

- Original multi-board project: **@Sukecz** ([esp32-birdnet-mic](https://github.com/Sukecz/esp32-birdnet-mic))
- Atom Echo port: this fork

## License

This firmware is released under the MIT License. See `../LICENSE`.
