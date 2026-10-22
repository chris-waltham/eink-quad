# Quad Board

An e-paper kitchen dashboard for the Seeed reTerminal E1001. Four corners of your day on a 7.5" screen: the weather, the news and markets, your teams, and whatever is flying over the house right now.

![Quad Board cycling through its pages](docs/img/demo.gif)

**[See the project page →](https://j-rod-ay.github.io/eink-quad/)**

🖥️ **The screen: [Seeed reTerminal E1001 on Amazon](https://link.amazon/B0gH6ekeP)** — 7.5″ 800×480 mono e-paper, ESP32-S3, battery. *(affiliate link)*

## What's on it

| Corner | Shows |
|---|---|
| **Weather** | current conditions, rain chance for the next few hours, 3-day forecast, 15-minute rain nowcast, NWS alerts |
| **News & markets** | index and ticker moves, rotating WSJ headlines |
| **Your teams** | live score, last result or next game for any ESPN team, with TV and the betting line |
| **Plane overhead** | airline, route, aircraft, altitude and which way to look, from live ADS-B. An ISS pass or a visible rocket launch takes this corner over |

A strip under the header carries one line: your own pinned message, a severe weather alert, or a sentence from Claude that ties the day together (optional).

Seen from the front, the three buttons on top are **next page** (left), **previous page** (middle) and **home / refresh** (right). The firmware calls them KEY2, KEY1 and KEY0. Each corner has its own detail page:

| | |
|---|---|
| ![Weather](docs/img/weather.png) | ![News & markets](docs/img/news.png) |
| ![Scores](docs/img/sports.png) | ![Sky above](docs/img/sky.png) |

## How it works

The screen doesn't decide anything. A small Python server renders every pixel with Pillow, diffs the frame against what's already on the glass, and hands the board only the rectangles that changed. The board long-polls the server over HTTPS on your network, checking its certificate (or deep-sleeps between checks on battery) and plays those rectangles as ~1 s partial refreshes. A full refresh with 4 grey levels runs on the regular update, and a flashing one every couple of hours wipes ghosting.

Changing the layout, the cadence or a data source never means reflashing. Everything is a setting on the server's web dashboard.

```
server/    Python: data sources, renderer, long-poll server, settings dashboard
firmware/  PlatformIO / Arduino for the reTerminal E1001 (ESP32-S3)
docs/      the project page and screenshots
```

## Data sources

All keyless: Open-Meteo (forecast + nowcast), api.weather.gov (alerts, US only), WSJ RSS, Yahoo Finance spark, ESPN's site API, adsb.lol + adsbdb, CelesTrak + Launch Library 2. The one-line brief uses the [`claude`](https://docs.claude.com/en/docs/claude-code) CLI if it's installed and logged in. Turn `llm_brief` off otherwise.

## Setup

### Hardware
- **[Seeed reTerminal E1001](https://link.amazon/B0gH6ekeP)** — 7.5" 800×480 mono e-paper, ESP32-S3 *(affiliate link; also at [Seeed](https://www.seeedstudio.com/))*
- An always-on Mac on the same network (a Mac mini is ideal). Linux works too, but the setup
  script is macOS-only.

### Server (macOS)

```sh
git clone https://github.com/J-Rod-Ay/eink-quad.git ~/eink-quad   # not in Desktop/Documents/Downloads
cd ~/eink-quad/server
./setup-mac.sh
```

The script makes a Python environment, two random keys, and a certificate for this Mac's
Bonjour name (`<name>.local`). It writes `server/quad.env` and the board's
`firmware/src/secrets.h`, installs a launchd service that starts at boot (as your user, not
root) and restarts on a crash, turns off system sleep, and allows the server through the macOS
firewall. It asks for your password once. Run it again any time: keys and the certificate are
kept. Try `./setup-mac.sh --no-service` first if you'd rather not change the system yet.

Two listeners, no reverse proxy:

| Port | Where | What |
|---|---|---|
| 8743 | your network, HTTPS | the board's `/api/device/poll` and `/healthz` only. Wants `Authorization: Bearer <QUAD_TOKEN>`; a bad or missing key gets `401`, an unknown path `404`, and both are logged |
| 8740 | this Mac only (`127.0.0.1`) | the settings page: `http://127.0.0.1:8740/?k=<QUAD_ADMIN>`. From another computer: `ssh -L 8740:127.0.0.1:8740 you@<name>.local` |

Don't forward either port on your router. The settings page shows the board's last connection and
why it failed, if it did. Logs: `server/logs/server.log`.

**Preview without hardware:** `.venv/bin/python cases.py` renders every screen with live data to `case_*.png`.

**Your teams:** the built-in keys are the Tampa ones (`bucs`, `lightning`, `rays`, `gators_fb`, `gators_bb`). Add your own in `config.json` with ESPN's sport, league and team id or abbreviation:

```json
{
  "city": "Chicago",
  "lat": 41.8781, "lon": -87.6298,
  "custom_teams": {
    "bears": ["Bears", "football", "nfl", "chi"],
    "cubs":  ["Cubs", "baseball", "mlb", "chc"]
  },
  "teams": ["bears", "cubs"]
}
```

### Firmware

Run `server/setup-mac.sh` first: it writes `firmware/src/secrets.h`. Then, with the board on USB:

```sh
cd firmware
pio run -t upload            # add --upload-port /dev/cu.usbmodem* if it picks the wrong one
pio device monitor           # boot log
```

On first boot the board opens a Wi-Fi setup portal named `QuadBoard-XXXX`. Join it from your phone
and pick your network (2.4 GHz only). To get back to it later, hold the left button and tap the right one.

The board finds the server by its Bonjour name (falling back to its last address, then
`SERVER_IP`), and only talks to a server whose certificate matches the fingerprint in
`secrets.h`. If you re-run setup and it makes a new certificate, flash again.

### When something's wrong

The board says so itself:

| Sign | Meaning |
|---|---|
| LED blinks 2× | no Wi-Fi |
| LED blinks 3× | server not found or not answering |
| LED blinks 4× | wrong certificate, or the key was rejected |
| a black label in the bottom-right corner | the problem has lasted a minute (plugged in) or one wake (battery) |
| a full screen saying what failed and what to try | five minutes (plugged in) or three wakes (battery) |
| **left + right buttons together** | diagnostics: Wi-Fi, signal, addresses, last result, battery |

At power-on it shows "Starting up" with the Wi-Fi it joined and the server it's looking for. The
header's "Updated" time shows when the server last drew the screen.

## Notes

- Times use the server's `TZ` (defaults to `America/New_York`).
- The notes feature reads a separate "note box" project's database and is off unless `QUAD_NOTES_DB` is set.
- Team logos and airline logos are downloaded at runtime and cached in `server/cache/`. None are shipped in this repo.

## License

MIT. Bundled fonts carry their own licences, listed in [`server/fonts/LICENSES.md`](server/fonts/LICENSES.md).
