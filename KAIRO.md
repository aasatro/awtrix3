# Kairo fork of AWTRIX3

This fork exists for exactly one reason: **the timer must count on the clock itself.**

Kairo (a time tracker) used to push one frame per second over MQTT. The frames were sent
perfectly — 241 frames in 240 s, no gap over 1.4 s — but the Wi-Fi path to the clock loses
5–16 % of packets at 1 ping/s and 42 % at 10/s. MQTT rides on TCP, so lost packets are
*retransmitted, not dropped*: the frames pile up and arrive in a burst. The counter stalls,
then catches up several seconds in milliseconds. Kairo/#53 has the measurements.

Counting locally drops the traffic from ~60 messages/minute to ~2, which makes the jitter
irrelevant instead of visible.

## Changes against upstream (CC BY-NC-SA 4.0 requires these be marked)

| Where | What | Issue |
|---|---|---|
| `README.md`, `KAIRO.md` | fork notice and this file | aasatro/Kairo#54 |
| `src/Apps.cpp` | `formatTimer()` + two cases in `replacePlaceholders()`: `{{UP:<epoch>}}` counts up from an instant, `{{DOWN:<epoch>}}` counts down to one | aasatro/Kairo#55 |

### The timer placeholder

`replacePlaceholders()` already ran on every render pass, resolving `{{topic}}` to its last
MQTT value. Two extra cases make it count without anyone speaking to it:

```
mosquitto_pub -h 192.168.178.70 -t 'awtrix/custom/test' \
  -m '{"text":"{{UP:1752791234}}","noScroll":true}'
```

`UP` counts up from that Unix epoch, `DOWN` counts down to it and stops at `00:00` rather
than going negative. The format mirrors `hms()` in Kairo's
[kairo/adapters/awtrix.py](https://github.com/aasatro/Kairo/blob/main/kairo/adapters/awtrix.py)
— `MM:SS`, and `H:MM:SS` from an hour — because the web UI and the clock must not show the
same second in two different shapes.

Any other placeholder still goes to `MQTTManager.getValueForTopic()` untouched, so no
existing setup notices this firmware. The time source is `time(nullptr)`: `ServerManager`
calls `configTzTime()`, so the system clock is real UTC epoch, and the difference of two
epochs carries no timezone — nothing to convert.

Upstream: [Blueforcer/awtrix3](https://github.com/Blueforcer/awtrix3), forked at `723e8c7`,
`VERSION = "0.98"` — the same version the clock ships with.

## Build

PlatformIO, Arduino framework, `default_envs = ulanzi`. PlatformIO is not a project
dependency; install it wherever you build:

```
python -m pip install platformio
python -m platformio run -e ulanzi     # → .pio/build/ulanzi/firmware.bin
```

The first run downloads ~1.2 GB of toolchain (xtensa-esp32 + arduino-espressif32) and takes
about 10 minutes. Later runs are seconds.

**Flash is at 96.8 % (1 269 229 of 1 310 720 bytes).** About 41 kB of headroom — enough for
a placeholder, not for a feature. Watch this number on every build; when it hits 100 % the
link fails, and the fix is a partition change, not a smaller `if`.

## Flash over OTA

The clock serves the Arduino core's `HTTPUpdateServer` at `/update` (wired up in
[lib/webserver/esp-fs-webserver.cpp](lib/webserver/esp-fs-webserver.cpp), `m_httpUpdater`).
No authentication, no USB cable needed — which matters, because it is unclear whether the
cable attached to this clock carries data at all or only power.

```
curl -F "firmware=@.pio/build/ulanzi/firmware.bin" http://192.168.178.143/update
```

The clock reboots itself when the upload verifies. Give it ~20 s, then check:

```
curl -s http://192.168.178.143/api/stats     # version, uptime back near 0
```

**Never flash or reboot while a session is running** — the clock is in daily use. Ask Kairo
first; `null` means the coast is clear:

```
curl -s http://192.168.178.70:8138/api/v1/entries/current
```

## The way back to official 0.98

The ESP32 keeps two OTA banks, so a *failed* upload leaves the running firmware untouched.
The real risk is an upload that succeeds and then misbehaves. That case is covered by
reflashing the official build over the same endpoint — it is an ordinary OTA, no different
from flashing our own:

```
curl -sLO https://github.com/Blueforcer/awtrix3/releases/download/0.98/ulanzi_TC001_0.98.bin
curl -F "firmware=@ulanzi_TC001_0.98.bin" http://192.168.178.143/update
```

`ulanzi_TC001_0.98.bin`, 1 271 856 bytes,
sha256 `d1ae4dabafba7894c7b32cc29f3af24a2b11e039b85bd82b13e0bd01218878d5`.

A local copy is kept outside this repo (`../awtrix3-rollback/`) so the rollback does not
depend on GitHub being reachable at the moment it is needed.

If the clock is bricked badly enough that `/update` no longer answers, OTA cannot help and
it needs USB — see upstream's flasher. That is the risk the whole exercise is built to
avoid, which is why the first flash was a byte-for-byte equivalent build of the firmware
already running.

## Licence

Upstream is [CC BY-NC-SA 4.0](LICENSE.md) — not an OSI licence. Forking and modifying is
allowed; use here is private and non-commercial. Derivatives stay under the same licence,
and **changes must be marked** — that is what the table above is for.
