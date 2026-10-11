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

Each one is switched on from the config page like any other source and keeps to its own
port, so it never collides with an upstream source.

#### WEATHER — port 8790

The current temperature with today's range, cloud cover and wind, whether rain is expected
today and how much, an hour-by-hour strip, and below it a looping precipitation radar
(RainViewer tiles on an OpenStreetMap base) centred on your position.

- **Data:** DMI Open Data's Forecast EDR API, the HARMONIE DINI model. No API key.
- **When DMI says no:** it answers `429 Server is busy` for hours at a time — the same DMI
  model is then fetched through [Open-Meteo](https://open-meteo.com/en/docs/dmi-api) and the
  page credits whichever one served the forecast. A failed fetch is retried after 1, 2, 5,
  10, 20 and then 30 minutes rather than waiting out the full refresh interval.
- **Config:** `latitude`, `longitude`, and an optional `location_name` shown instead of the
  coordinates.
- Needs an internet connection. Danish and English.

#### CALENDAR — port 8791

An agenda of what is coming up, grouped by day with "Today" and "Tomorrow" spelled out,
times, location and which calendar an event came from.

- **Data:** a calendar's *secret iCal URL*, not the Google Calendar API — so there is no
  OAuth client to register, no consent screen to complete on a device with no keyboard, and
  no tokens to refresh. Recurring events, their exceptions and all-day events are expanded
  on the device; several calendars merge into one list.
- **Read-only by design:** the device never writes to your calendar.
- **Config:** `url` (Google Calendar → Settings for my calendars → Integrate calendar →
  "Secret address in iCal format"), optional `calendars` for more of them, `days_ahead`
  (1–365) and `timezone`. See [docs/google-calendar.md](docs/google-calendar.md).
- **Treat the URL as a password:** anyone holding it can read that calendar.
- Needs an internet connection. Danish and English.

#### DR NYHEDER — port 8792

Headlines from DR's public RSS sections, browsable on the arc and opened as full page views,
the same shape upstream's Guardian news source uses.

- **Data:** DR's feeds carry no body text, so each article page is fetched once and its text
  read out of the page's own embedded data, then cached per link — a refresh only fetches
  articles it has not seen. Four at a time, to stay gentle on both DR and the Pi.
- **Config:** optional `feeds` list (defaults to senestenyt, indland, udland, politik, penge,
  kultur, viden, sporten).
- **Limits:** video-only items are skipped since they have no text; sport items that point
  into a live blog show that post. Because the text is read out of dr.dk's own pages, a
  redesign there can break it — the service logs it when most articles come back empty.
- Needs an internet connection. The articles are Danish; the surrounding view is upstream's
  and stays in English.

#### TO-DO / HUSKELISTE — port 8793

A shared list: add and edit the items from a phone, see them on the arc, where GO ticks an
item off and "Clear ticked" clears the done ones.

- **Home network only.** The phone page is at `http://<device-ip>:8793/` and, like every
  other page the device serves, it has no login — anyone on your network can open it.
  **Do not port-forward it.** It does guard against the attacks a home network is still open
  to from a browser: a request has to be addressed to the device itself (DNS-rebinding
  guard), may not come from another site (Origin check), changes need a JSON body, the page
  ships a strict CSP and list text is never rendered as HTML. Page and API share the one
  port on purpose, so access from outside can later be added by putting exactly that port
  behind an authenticating tunnel, without exposing anything else on the device.
- **Storage:** `~/.beosound5c_todo.json` on the device, mode 0600, rewritten atomically.
- Danish and English; on "auto" the phone page follows the phone's own language.

#### CONTACTS / KONTAKTER — port 8794

The household's contacts on the arc, each tile with a picture when one has been added, and
GO opens the details: phone, e-mail, address, birthday and a note. Edited from a phone page
at `http://<device-ip>:8794/`, linked from the config page.

- **Home network only**, with the same guards as the to-do list — and one more: contacts are
  personal data, so this service never answers another site's page with a CORS grant. Only
  the device's own UI and the phone page can read them.
- **Pictures** are re-encoded when uploaded, which also strips the phone's EXIF and GPS data.
- **Storage:** `~/.beosound5c_contacts.json` and `~/.beosound5c_contacts_photos/`, mode 0600.
- Danish and English.

#### DASHBOARD / OVERBLIK — port 8795

One screen to glance at rather than browse: clock and date, what is playing, how the device
itself is doing (uptime, CPU temperature, load, memory, disk, network address) and the
electricity price hour by hour for today and tomorrow.

- **Electricity prices** come from [Strømligning](https://stromligning.dk)'s public API (no
  key) as all-in prices per kWh — spot price plus your net company's tariffs, taxes and VAT.
  Tomorrow's are their forecast until the real prices are published around 13:00. **Denmark
  only.**
- **Config:** `electricity_supplier` (your net company's id on stromligning.dk, the
  `netselskab=` value in its links, e.g. `dinel_c`) and `electricity_area`, DK1 west of the
  Great Belt or DK2 east. Without a supplier the page simply leaves the prices out.
- **Costs nothing when idle:** prices are fetched when the page asks and at most every 15
  minutes, and the device data is read from `/proc` on request. Nothing polls in the
  background. Every field is optional — a part that cannot be read is left out rather than
  breaking the page.
- Danish and English.

### Other additions

#### Device language — Danish or English

One setting in the config UI. `auto` follows the browser or the phone; `da` and `en` force
it. It covers the arc menu labels and the views this fork added or translated — Weather,
Calendar, Dashboard, Radio, To-do and Contacts, including the two phone pages. Upstream's
own views have no translation table and stay in English.

#### Bluetooth speaker output

Pair a Bluetooth speaker from the config page and use it as the device's audio output. It
becomes a volume adapter like any other: the tone chain's output is pointed at the speaker
and the volume is set through the PipeWire sink, which reaches the speaker's own amplifier
over AVRCP when it supports that. A watch loop reconnects when the speaker has been switched
off or carried out of range, pairing uses bonding so it survives standby, and standby
disconnects so the speaker can sleep and other devices can borrow it.

### Good to know

- **The phone pages are for the home network.** To-do (8793) and Contacts (8794) have no
  login. Keep them on the LAN; do not forward those ports.
- **Personal data stays on the device**, in the service user's home directory with 0600
  permissions. Nothing is sent anywhere except the requests each source makes to its own
  source of data.
- **Internet is needed** for the weather, the calendar, the news and the electricity prices.
  Everything else works without it.
- **Some of it is Denmark-shaped:** DR's news feeds and Danish electricity prices. The rest
  works anywhere.
- **The iCal URL is a credential.** Anyone who gets hold of it can read that calendar.

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
- **Audio setup:** the SD-card hardening switched WirePlumber's BlueZ monitor off, which
  is exactly what creates a Bluetooth speaker's sink — it now stays on when a Bluetooth
  speaker is the configured output, and the installer brings the PipeWire Bluetooth
  plugin and `pactl` with it.

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
