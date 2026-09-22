<div align="center">

<p>
  <a href="README.md"><img src="docs/langues/fr-off.png" alt="Lire cette page en français" width="150" /></a>
  <img src="docs/langues/en-on.png" alt="English, page shown" width="150" />
</p>

<img src="docs/en/banniere.png" alt="Switch Track, the points module" width="100%">

</div>

A motorised switch for MicroCoaster layouts. An electric actuator moves the movable section of track to the left or to the right, two LEDs show the position, and the order arrives from the controller over WebSocket.

Like the other modules, it is configured on first boot through a captive portal, then joins the server.

**Version 2.0.0**

<img src="docs/en/sections/s01.png" alt="01 How it works" width="100%">

The module decides nothing. It receives a position order, drives the actuator, checks that the position is reached, and reports back. It is the controller that knows whether the track downstream is clear and whether the change is allowed.

<img src="docs/en/schemas/principe.png" alt="Order received: the server sends switch_left or switch_right. Actuator driven: the direction is handed to the DRV8871, which flips the polarity. Position reached: the matching LED lights up and the reply goes back to the server. With no order: the actuator does not move, the position is held." width="100%">

The current position is reported on every connection and on every change, so the server never drifts out of step with what the layout is actually doing.

<img src="docs/en/sections/s02.png" alt="02 Hardware" width="100%">

<img src="docs/en/schemas/brochage.png" alt="Actuator DRV8871: GPIO 21 for IN1, actuator clockwise; GPIO 22 for IN2, actuator anticlockwise. Position signalling: GPIO 2 for the left LED, diverging route live; GPIO 4 for the right LED, straight route live." width="100%">

The DRV8871 is an H-bridge: it pulls `IN1` or `IN2` high to reverse the polarity across the actuator, and both low to stop it. Its thermal protection and current limiting keep the actuator from being damaged if the track is jammed.

<img src="docs/en/sections/s03.png" alt="03 Protocol" width="100%">

The module authenticates on connection, then talks JSON.

**Identification, module to server**

```json
{
  "type": "module_identify",
  "moduleId": "MC-0001-ST",
  "password": "<module secret>",
  "moduleType": "switch-track",
  "uptime": 12345,
  "position": "left"
}
```

**Command, server to module**

```json
{
  "type": "command",
  "data": { "command": "switch_left" }
}
```

<img src="docs/en/schemas/commandes.png" alt="switch_left: throws the track to the left, also accepts left and switch_to_A. switch_right: throws the track to the right, also accepts right and switch_to_B. get_position: returns the current position without driving the actuator." width="100%">

The module answers every command and sends periodic telemetry with its position and its uptime.

`SERVER_USE_SSL` chooses between `ws` and `wss`. On a local development network, `ws` is enough. In production, or as soon as the server is reachable beyond the home network, switch to `wss`: without encryption, the module's authentication secret travels in the clear.

<img src="docs/en/sections/s04.png" alt="04 Bringing it up" width="100%">

First copy [`include/env.h.example`](include/env.h.example) to `include/env.h` and fill it in: fallback portal credentials, the module identity and its secret. That file is not in git, and without it the firmware does not compile.

Requires [PlatformIO](https://platformio.org/) inside Visual Studio Code.

```bash
pio run                  # build
pio run -t upload        # upload the firmware
pio run -t uploadfs      # upload the portal to LittleFS
pio device monitor       # serial console, 115200 baud
```

1. Power the module. It creates a WiFi access point.
2. Connect to it and open `http://192.168.4.1`.
3. Enter the target network.
4. The module reboots, joins the network and announces itself to the server.

The WiFi credentials stay in the module's memory, never in the repository.

<img src="docs/en/sections/s05.png" alt="05 Ecosystem" width="100%">

```ini
links2004/WebSockets        ; link to the controller
bblanchon/ArduinoJson       ; the messages exchanged
ayresnet/AyresWiFiManager   ; captive portal and reconnection
```

Embedded filesystem: **LittleFS**, which holds the portal pages. The common base for every module is the [WiFi Manager](https://github.com/Microcoaster/MicroCoaster_WifiManager), the [LED bench](https://github.com/Microcoaster/ESP-32-led) is its test version without the mechanics, and the driving is done from the [WebApp](https://github.com/Microcoaster/MicroCoasterWebApp).

---

<sub>MicroCoaster · Author: Cybertrist</sub>
