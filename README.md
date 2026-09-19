# esphome-delta-acrel-mitm

An ESPHome man-in-the-middle between a **Delta E-series hybrid inverter** and its **Acrel
AGF-AE-D** revenue meter, on a single ESP32. It reads the real meter, applies an offset, and
serves the result to the inverter.

That offset is the point. A Delta E runs a self-consumption loop against what the meter tells
it, so biasing the meter reading is a working **control channel** for charge and discharge —
on an inverter that exposes no such control of its own.

> 🔴 **This one transmits, and it changes what your inverter does.**
>
> It sits **in series with the revenue meter**, which means cutting into the meter run, and it
> becomes the only thing the inverter can see of the grid. A bug here is not a missing sensor
> reading — it is an inverter acting on a number you made up. Read [Before you build
> this](#before-you-build-this).
>
> Its passive sibling, [**esphome-delta-lg-monitor**](https://github.com/dalklein/esphome-delta-lg-monitor),
> only listens and can be wired in with no risk of this kind. If you want data rather than
> control, start there.

## How it works

Two RS485 buses, one ESP32, opposite roles on each:

| | **bus 1 — client** | **bus 2 — server** |
|---|---|---|
| pins | GPIO16 TX / GPIO17 RX | GPIO18 TX / GPIO19 RX |
| speed | 9600 8N1 | 9600 8N1 |
| who is master | **this board** | **the Delta** |
| what is on it | the real Acrel meter, address 2 | the Delta, and whatever else is on its RGM bus |
| this board | polls the meter | **answers as the meter**, address 2 |

```
  bus 1   ESP32 (master) ────── real Acrel AGF-AE-D  (addr 2)
  bus 2   Delta (master) ────── ESP32 answering as the meter (addr 2)
                          └──── LG RESU battery, and anything else on the RGM bus
```

The real meter is moved off the inverter's bus and onto a segment of its own. The Delta keeps
polling address 2 and never knows the difference.

⚠️ **RX/TX cross between the ESP32 and each RS485-TTL module** — the pin labelled TX on the
module goes to the pin labelled RX on the devkit silkscreen. Follow the comments at each `uart:`
block, not the silkscreen.

## Control

Three MQTT topics drive it:

| topic | meaning |
|---|---|
| `cmnd/delta-gt-meter/adjust_pwr` | watts to add to the served reading |
| `cmnd/delta-gt-meter/ctrl_mode` | `0` = pass through with offset, `1` = serve zeros |
| `cmnd/delta-gt-meter/ctrl_gain` | scale factor, `1` = normal |

Telemetry — the real meter's readings, the served values, and diagnostics — is published under
`delta/inv/`.

> ⚠️ **`ctrl_mode: 1` does not stop the inverter.** Serving zeros gives its control loop no
> error to act on, so it **freezes at whatever it was doing**. It is a freeze, not a stop. If
> you want the inverter to stop, this is the wrong lever.

### What the loop is actually like

Worth knowing before you tune anything: the Delta's meter loop measures as a **pure
integrator** — proportional gain is approximately zero, and there is no anti-windup. The loop
gain you experience is therefore set by your *battery's* droop characteristic, not by the
inverter. A stiffer battery makes the same offset act far more aggressively.

## Timing, and why the config looks the way it does

Most of the non-obvious settings exist because this sits on the control path, and the numbers
behind them were measured rather than guessed. The config comments carry the detail; the
headlines:

* **`rx_full_threshold: 120` on bus 1.** Put it above the longest expected reply (51 bytes
  here) so each response arrives as one chunk. A split reply is what produces modbus's
  `not enough data for value`. Shortening `rx_timeout` to chase that makes it *worse* —
  measured, `rx_timeout: 1` multiplied the failure rate by 17×. **Never `rx_timeout: 0`.**
* **Three `modbus_controller` tiers at 0.3 s / 5 s / 60 s**, all on the same address and hub.
  Per-sensor `skip_updates` is inert from 2026.9.0, and one controller per cadence is the
  replacement. Cadences are matched to what the Delta actually asks for, which keeps the
  51-byte worst-case read off the control path entirely.
* **0.3 s on the fast tier.** The Delta re-reads the served power block every ~113 ms, so this
  sets how stale the number it acts on can be. One transaction costs 44 ms of bus time, so
  0.3 s is about 15% duty.
* **`send_wait_time: 150ms`.** The 2000 ms default blocks the whole client hub on any dropped
  reply — measured as ~1.3% dead time feeding an integrator.
* **A 200 ms EMA on the measurement**, not for noise (the signal is coherent, not noisy) but to
  replace the zero-order-hold staircase with a ramp, so the integrator stops seeing a
  discontinuity at every sample. The offset is applied *after* it, so commands still act
  immediately.

## Requirements

**ESPHome 2026.9.0 or newer**, stock — no fork and no patched components.

2026.8 is not enough: `reuse_previous_range` does not exist there, and per-sensor `skip_updates`
is still live rather than inert, so the tier split would not poll as intended.

If you are porting an older config of this kind, three things changed upstream and each one
will bite:

* `modbus_controller`'s `server_registers:` is gone. The server side is now the dedicated
  `modbus_server:` component, which clamps each response to the requested register count.
  Configs built on the old path can emit **over-long replies**.
* `allow_partial_read: true` is needed on every `S_DWORD`. The default is `false`, which turns a
  read that clips a 32-bit register into an `ILLEGAL_DATA_ADDRESS` exception — a response the
  Delta has never been shown.
* `modbus_server` does **not** handle NaN for you. Every `read_lambda` here returns
  `optional<T>` and declines on NaN/Inf; without that you serve undefined bytes.

## Hardware

* **Delta E6-TL-US** inverter — E4/E8/E10-TL-US should work
* **Acrel AGF-AE-D** revenue meter
* ESP32-WROOM-32 (esp32dev, arduino framework)
* Two RS485-TTL modules, auto-direction (no DE/RE pin), from the ESP32's 3.3 V rail

## Use

```bash
cp secrets.yaml.example secrets.yaml   # then fill it in
esphome run delta-acrel-mitm.yaml
```

## Before you build this

Things that are easy to find out the hard way:

* **The meter run is the grid connection.** Cutting into it is electrical work on the supply
  side of your system, subject to whatever rules apply where you are. This repo is firmware; it
  has nothing to say about doing that part safely or legally.
* **Your inverter will act on whatever you serve it.** There is no sanity check between the
  number you publish and the inverter's response. An offset with the wrong sign pushes the
  opposite way.
* **`ctrl_mode: 1` freezes, it does not stop** — see above.
* **Check what your meter's address reassignment does before you touch it.** On this
  installation, the "production meter" commissioning option renames the grid meter to ID 3
  with no way back through the inverter's UI.
* **Test against a second inverter if you have one.** Everything here that could be measured
  passively was checked against an independent listener on the same bus before being trusted;
  the parts that transmit cannot be validated that way.

## A note on the NaN guards, if you are comparing against an older version

Four guards in the control path used to read `if (x.state == NAN)`. In IEEE-754 **every
comparison with NaN is false**, so they never fired and returned the NaN they were meant to
catch. They now use `std::isnan()`, like the other 45 guards in the file.

This was not cosmetic. An ESPHome sensor reads NaN until its first value arrives, so before
`cmnd/delta-gt-meter/ctrl_mode` is received, `delta_ctrl_mode` was NaN — which made the
`if (delta_ctrl_mode == 0)` gate false, so `adjusted_meter_power` never published, so the
server's read lambdas fell back to their zero-initialised `_last` and **served the inverter
0 W**. That is the freeze condition, reached silently at boot.

The reason it went unnoticed is worth repeating, because it is easy to assume the wrong
mitigation: on the installation this came from, `adjust_pwr` and `ctrl_gain` **are** retained,
so they land the instant MQTT connects — but `ctrl_mode`, the one the gate actually tests, is
**not**. It only arrives on the controller's next periodic publish, so every boot had a window
of a few seconds serving zeros, and a boot while the controller was down would have served
zeros indefinitely.

> 🔑 **Publish `ctrl_mode` retained anyway.** The guard fix removes the failure, but a retained
> control topic means a freshly booted MITM knows the intended mode immediately rather than
> assuming a default.

## Status

Running continuously on the installation it was written for. It is one person's config for one
pairing of inverter and meter, not a product — expect to read it rather than just flash it.

The register map is Acrel AGF-AE-D on the client side and SunSpec-shaped on the served side
(the Delta reads a meter model based at 40001). If your meter differs, the client half is what
you would rewrite; the server half is what the Delta expects and should stay as it is.

## Credits

Built by [@dalklein](https://github.com/dalklein) with [Claude Code](https://claude.com/claude-code).

Inspired by **PaulSturbo**'s `solis-ct-meter.yaml`, which showed that an ESPHome Modbus server
could stand in for a meter at all — this config was written from that idea rather than copied
from it, and targets different hardware on both sides. Discussion and earlier versions:
[Modbus server enable/disable, modbus sniffer — Home Assistant
Community](https://community.home-assistant.io/t/modbus-server-enable-disable-modbus-sniffer-solar-battery-inverter-meter-rs485-modbus-rtu-man-in-the-middle/848251).

Bus timing and framing figures were measured against an independent Raspberry Pi with an FTDI
adapter on the same bus.

## Licence

[MIT](LICENSE).
