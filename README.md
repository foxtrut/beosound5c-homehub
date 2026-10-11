# BeoSound 5c

A modern recreation of the Bang & Olufsen BeoSound 5 experience using web technologies and a Raspberry Pi 5.

**Website: [beosound5c.com](https://beosound5c.com)**

This project replaces the original BeoSound 5 software with a circular arc-based touch UI that integrates with Sonos, Bluesound, and Denon HEOS players (plus experimental WiiM/LinkPlay and B&O Mozart/ASE), music services (Spotify, Apple Music, TIDAL, Plex, Jellyfin, AirPlay 2), and Home Assistant. It works with the original BS5 hardware (rotary encoder, laser pointer, display) and supports BeoRemote One for wireless control.

## About this fork

[foxtrut/beosound5c-homehub](https://github.com/foxtrut/beosound5c-homehub) turns the
BeoSound 5 into a **home hub**: besides music it shows the weather, the family calendar,
the news, a shared to-do list, the contacts and a dashboard — and the whole device speaks
Danish as well as English. Everything from "Quick Start" down is
[mkirsten's](https://github.com/mkirsten/beosound5c) original documentation and still
applies; this fork follows upstream releases (currently **v0.11.0**).

### Sources added

| Menu item | Port | What it does | Needs |
| --- | --- | --- | --- |
| **WEATHER** | 8790 | Hourly forecast, today's temperature range and rain outlook from DMI Open Data, plus a looping precipitation radar. Falls back to the same DMI model via Open-Meteo when DMI's API answers "server is busy", which it does for hours at a time. | Latitude/longitude |
| **CALENDAR** | 8791 | Read-only agenda from a Google Calendar's secret iCal URL: recurring events, all-day events, exceptions, and several calendars merged into one list. | iCal URL |
| **DR NYHEDER** | 8792 | Headlines from DR's public RSS feeds with the article text fetched for reading on the device. | — |
| **TO-DO** / HUSKELISTE | 8793 | Shared to-do list, ticked off on the device and edited from a phone page. | — |
| **CONTACTS** / KONTAKTER | 8794 | Contacts on the arc with picture and birthday, edited from a phone page. | — |
| **DASHBOARD** / OVERBLIK | 8795 | One screen with clock, what is playing, device status and the electricity price hour by hour for today and tomorrow. | — |

Each one is switched on from the config page like any other source, and keeps to its own
port so it never collides with an upstream source.

### Other additions

- **Device language, Danish or English.** One setting in the config UI translates the arc
  menu and every added view; `auto` follows the phone or browser.
- **Bluetooth speaker output.** Pair a Bluetooth speaker from the config page and use it as
  the device's output, with its own volume adapter. Pairings survive standby.

### Fixes to the upstream code

- **Jellyfin:** the access token travelled in the stream URL, where every player and proxy
  on the way could log it — it goes in a header now. Tracks also carry their album name, so
  the PLAYING view can show it.
- **Radio:** the Danish DR channels played the wrong region (P4 København where the list
  said Østjylland) and advertised MP3 128k while AAC was playing. Now-playing titles are
  read from the stream's ICY metadata, the poll wakes on a station change, favourites come
  first, and the stations that have no usable artwork got their own tiles.
- **PLAYING view:** a progress bar with elapsed and total time taken from mpv rather than
  from the source, the right pause glyph for locally played sources, no progress bar for
  live radio, and a cached playback state that no longer keeps the video overlay switched
  off.
- **First track of a session:** the router dropped the very first metadata message, so the
  screen stayed empty until the next track; it is replayed now.
- **Arc lists:** leaving a view and coming back lands on the level you left instead of the
  root. The two list implementations and their differing GO behaviour are written down in
  [docs/arc-lists.md](docs/arc-lists.md).
- **Menu:** static entries (PLAYING, SCENES, SYSTEM, SHOWING) never picked up a retranslated
  title.

### How this fork is organised

| Branch | Contents |
| --- | --- |
| `main` | Untouched mirror of upstream. Fast-forward only, never any of my commits. |
| One branch per feature | `weather-source`, `calendar-source`, `dr-news`, `todo-list`, `contacts`, `dashboard`, `bluetooth-speakers`, `language-picker`, `radio`, `playing-progress`, `jellyfin-header-auth`, `jellyfin-album`, `first-track-media-drop` — each sits directly on `main`, so it can be read, rebased or dropped on its own. |
| `device` | `main` with every feature branch merged in. Rebuilt from scratch on each upstream release, never committed to by hand. **This is the branch to run.** |

To try it, install from the `device` branch instead of upstream's `main`:

```bash
git clone -b device https://github.com/foxtrut/beosound5c-homehub.git ~/beosound5c
cd ~/beosound5c
sudo ./install/install.sh
```

## Quick Start

Runs on a [Raspberry Pi 5 4GB](https://www.raspberrypi.com/products/raspberry-pi-5/). See [beosound5c.com](https://beosound5c.com) for full installation instructions.

### Fresh Install

1. Flash **Raspberry Pi OS Bookworm Lite (64-bit)** using [Raspberry Pi Imager](https://www.raspberrypi.com/software/). Click the settings icon (gear) to enable SSH and set your username/password before writing.
2. Clone and run the installer:

```bash
git clone https://github.com/mkirsten/beosound5c.git ~/beosound5c
cd ~/beosound5c
sudo ./install/install.sh
```

The installer handles everything: packages, display config, service installation. No questions asked — it only prompts for a reboot when complete.

3. After rebooting, the device shows a QR code. Scan it with your phone (or open `http://<device-ip>/config` in any browser) to finish setup: device name, player, audio output, Home Assistant, and sources.

### Updating

```bash
git pull && sudo ./install/install.sh update
```

Updates service files, sudoers, and Python packages. No reboot needed unless system packages changed.

## Remote Support

If you need help troubleshooting, you can open a temporary remote support session. Nothing is installed for this up front: the first time you run `bs5c-support`, it asks before fetching [Tailscale](https://tailscale.com/), and no daemon runs until you complete a session with an access key.

```bash
bs5c-support          # Start session — prompts for an access key
bs5c-support stop     # End session and disconnect
bs5c-support status   # Check if a session is active
```

Ask the developer for an access key, paste it when prompted, and share the displayed Tailscale IP. When you're done, `bs5c-support stop` disconnects and stops the Tailscale daemon.

## Configuration

After install, open `http://<device-ip>/config` in a browser. The configuration UI lets you set the device name, player, volume adapter, Home Assistant connection, transport, and all sources. Changes are saved and services restart automatically.

Configuration lives in two files on the device:

- **`/etc/beosound5c/config.json`** — all settings (device name, player IP, menu, scenes, volume, transport)
- **`/etc/beosound5c/secrets.env`** — credentials only (HA token, MQTT password)

For the full list of fields and options, see the **[config schema](docs/config.schema.json)**.

To edit scenes (names, icons, HA scripts), edit `/etc/beosound5c/config.json` directly — the `"scenes"` array.

## Security model

A BeoSound 5c expects to live on a home network you trust, like the speakers it
talks to. Worth knowing before you install one:

- **The local HTTP API has no authentication.** Anything on your LAN can read
  device status and drive playback (`/router/*`, `/player/*`, the source
  services on ports 8766–8781). That's deliberate — it's how the kiosk, the
  phone setup page and Home Assistant all talk to it.
- **Config changes and updates reject cross-site browsers.** `POST /config` and
  `POST /update/run` accept requests from the device's own setup page, Home
  Assistant or `curl`, but refuse any web page served from somewhere else — so
  a site you happen to visit can't reconfigure your device or trigger an
  update behind your back.
- **Secrets stay out of the API.** `/etc/beosound5c/secrets.env` is `0600` and
  owned by the service user; the setup UI can write credentials but never reads
  them back.
- **Don't port-forward it.** None of these ports belong on the public internet.
  For access from outside, use a VPN or `bs5c-support` (below).

## Telemetry

Honestly, I just find it delightful to see where BeoSound 5cs are showing up in the world. There are already installations in the US, across Europe, here in Stockholm, in Asia, and in Australia — and every time a new one appears on the map it makes my day.

To make that possible, each BS5c sends a small anonymous ping to `beosound5c.com` on startup. There's a toggle for it in the web config UI — turning it off is completely fine and changes nothing else. **Nothing is sent before you've seen that toggle**: the first ping waits until you save the setup screen, so a device you install and never configure never phones home. Your public IP is used to infer a country (via Cloudflare — never stored beyond the country name). Nothing that identifies you or your hardware is sent: no hostname, no device name, no MAC address, no account names, no credentials, and nothing about what you listen to. Feel free to read exactly what gets posted in [`services/lib/beacon.py`](services/lib/beacon.py).

| Field | Value |
|---|---|
| `device_id` | A random ID the device makes up for itself on first ping and keeps in a `device_id` file. It isn't derived from your MAC address or any other hardware identifier, so it says nothing about your machine — it exists only so two pings can be recognised as the same dot on the map. Re-image the SD card and you simply get a new one |
| `version` | Software version string |
| `sources` | Names of enabled sources (e.g. `spotify`, `cd`) — no credentials or config values |
| `player_type` | Player backend: `sonos`, `bluesound`, `heos`, `local`, or (experimental) `wiim`, `mozart`, `ase` |
| `volume_type` | Volume adapter type: `sonos`, `beolab5`, `powerlink`, etc. |

If you'd rather opt out, just create a `NO_TELEMETRY` file in the repo root:

```bash
touch ~/beosound5c/NO_TELEMETRY
```

## Documentation

- [Audio, players & sources](docs/audio-setup.md) — player types, source compatibility, Spotify setup, volume adapters
- [Home Assistant integration](docs/home-assistant.md) — MQTT, webhooks, automation examples
- [Remotes & IR](docs/remotes.md) — BeoRemote One pairing, IR source buttons, Beo6
- [Development & contributing](docs/CONTRIBUTING.md) — local dev setup, repo layout, deploy script

## Acknowledgments

`services/masterlink.py` is substantially a derivative work of [libpc2](https://github.com/toresbe/libpc2) by Tore Sinding Bekkedal (GPL-3.0). Arc geometry in `web/js/arcs.js` is derived from [Beolyd5](https://github.com/larsbaunwall/Beolyd5) by Lars Baunwall (Apache 2.0). See [THIRDPARTY.md](THIRDPARTY.md) for the full list.

This project is not affiliated with Bang & Olufsen. "Bang & Olufsen", "BeoSound", "BeoRemote", and "MasterLink" are trademarks of Bang & Olufsen A/S.
