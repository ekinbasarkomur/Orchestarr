<h1 align="center">Orchestarr</h1>

<p align="center">
  <b>A whole self-hosted media stack from one command — installed, wired together and run by an AI assistant.</b><br />
  Jellyfin, Sonarr, Radarr, Prowlarr, qBittorrent and friends, set up step by step from a browser,<br />
  with every API key, password and connection configured for you.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-planning-e06c6c" alt="Planning" />
  <img src="https://img.shields.io/badge/license-GPL--3.0-d4a64a" alt="GPL-3.0" />
  <img src="https://img.shields.io/badge/brain-Maestarr-8b7cf6" alt="Powered by Maestarr" />
</p>

> [!NOTE]
> **Status: design.** Nothing here runs yet. This README is the plan the code
> will be written against. Orchestarr is the installer and distribution; the
> UI and the AI live in its sister project,
> [Maestarr](https://github.com/ekinbasarkomur/Maestarr).

---

## The problem

Running your own media server is a weekend of reading guides. A working
stack needs half a dozen apps, and each one must be told about the others:

- Sonarr and Radarr need an API key, a download client, root folders, quality
  profiles and a connection to Prowlarr.
- Prowlarr needs every *arr app registered as an "application" and indexers
  added, some behind a proxy or FlareSolverr.
- qBittorrent needs categories, a password and — for most people — a VPN in
  front of it.
- Jellyfin needs a first-run wizard, an admin user and libraries pointing at the
  right folders.
- The folders themselves must be laid out so imports are **hardlinks** and not
  copies, or the disk fills up twice as fast.
- Every app has its own login and its own API key, and the user has to copy
  them between six web UIs.

The existing all-in-one compose files start the containers, but they leave
all of that wiring to you. That wiring is where people give up.

## What Orchestarr does

```bash
# planned — one of:
curl -fsSL https://get.orchestarr.app | sh
docker compose up -d
```

Then open `http://<your-server>:8709` and Maestarr's onboarding walks you through
everything, one screen at a time:

1. **You** — one username and password. By default every app gets the same
   login, so you only remember one (change any of them later).
2. **Your disk** — where media lives. Orchestarr creates the
   [TRaSH Guides](https://trash-guides.info) layout (`/data/torrents`, `/data/media`)
   on one filesystem, so imports are instant hardlinks.
3. **What you watch** — movies, series, anime, music, books. Only the apps you
   need are installed.
4. **How you download** — torrent or Usenet; optionally a VPN (Gluetun with
   your provider's WireGuard key) that the download client is locked behind.
5. **Where you watch** — Jellyfin by default; Plex or Emby as alternatives.
6. **Requests** — optional Seerr so family members can ask for titles.
7. **Your AI** — any provider Maestarr supports (Anthropic, OpenAI, Google,
   OpenRouter, a local Ollama, or none at all).

Behind each screen Orchestarr starts the containers it needs and configures
them through their own APIs. When the wizard finishes, the stack is not
"installed" — it is **working**: search a title in Maestarr and it downloads,
imports and shows up in Jellyfin.

### Configured automatically

| App | What is set up for you |
| --- | --- |
| **Jellyfin** | first-run wizard, admin user, Movies / Shows / Anime libraries, hardware transcoding when a GPU is present, an API key for Maestarr |
| **Sonarr** | API key, login, root folders (`/data/media/tv`, `/data/media/anime`), qBittorrent download client with the `tv` category, Jellyfin "connect" for library refresh, anime series type defaults |
| **Radarr** | API key, login, root folder, download client with the `movies` category, Jellyfin connect |
| **Prowlarr** | API key, login, Sonarr and Radarr registered as applications (full sync), a starter set of public indexers chosen for your languages, the VPN proxy and FlareSolverr wired as indexer proxies |
| **qBittorrent** | Web UI password, categories, save paths, seeding limits, "stop then let the app remove", queueing; when a VPN is used, forwarded-port sync |
| **Gluetun** *(optional)* | provider, WireGuard key, port forwarding, a health check that changes server when a VPN node is slow |
| **Bazarr** | subtitle languages in your order, Sonarr/Radarr connections |
| **Seerr** *(optional)* | Jellyfin login, Sonarr/Radarr servers with your profiles |
| **Recyclarr** | TRaSH quality profiles and custom formats synced daily ("UHD, then HD", "Anime") |
| **FlareSolverr** *(optional)* | started only when an indexer needs it |
| **Lidarr / Readarr / Bazarr** *(optional)* | same pattern as Sonarr/Radarr |

Every secret is generated, stored in one place (`/config/secrets`), and shown to
you once at the end of the wizard. Nothing has to be copied by hand.

### Then: manage the whole stack by talking to it

Maestarr already reads and drives Sonarr, Radarr, Prowlarr, qBittorrent and
Jellyfin. With Orchestarr it also gets **admin tools** for each app, so the
assistant can change settings for you — always showing the change and asking
first:

> *"Prefer 1080p for anime but allow 4K remuxes for movies."*
> *"Add Turkish subtitles before English in Bazarr."*
> *"My VPN is slow — switch to a Swiss server."*
> *"Why didn't last night's episode download?"*
> *"Add a second user for my sister who can only request, not delete."*
> *"Move the library to the new 8 TB disk."*

Each change goes through the app's own API, so its UI stays the source of truth
and nothing is edited behind its back.

## How it fits together

```
                ┌──────────────────────── your server ────────────────────────┐
  browser ──▶   │  Maestarr  (UI, onboarding, AI agent)            :8709      │
                │     │  setup + admin tools over each app's API              │
                │     ▼                                                       │
                │  Orchestarr engine ──▶ docker-socket-proxy ──▶ Docker       │
                │     │ starts/stops the containers the wizard chose          │
                │     ▼                                                       │
                │  jellyfin  sonarr  radarr  prowlarr  bazarr  seerr          │
                │  gluetun ─▶ qbittorrent        recyclarr  flaresolverr      │
                │                                                             │
                │  /data  (torrents + media on one filesystem → hardlinks)    │
                └─────────────────────────────────────────────────────────────┘
```

- **One compose file** declares every service behind
  [profiles](https://docs.docker.com/compose/how-tos/profiles/); only the
  bootstrap services start at first (`maestarr`, the engine and the socket
  proxy).
- The **engine** is a small service that turns the wizard's choices into
  `docker compose --profile … up -d`, waits for each app to be healthy, then
  runs its configurator.
- Docker is reached only through a
  [socket proxy](https://github.com/Tecnativa/docker-socket-proxy) that allows
  container start/stop/inspect and nothing else — never the raw socket.
- **Configurators** are idempotent: each one reads an app's current state and
  only adds what is missing, so re-running the wizard repairs a broken stack
  instead of duplicating it.
- Pre-seeding where the apps allow it (API keys and auth in Sonarr/Radarr/
  Prowlarr's `config.xml`, qBittorrent's `qBittorrent.conf`), their APIs for
  everything else (Jellyfin's startup endpoints, Prowlarr applications,
  download clients, root folders, Seerr's setup).

### Defaults that matter

- Every port is published on **loopback only**; reach the stack through the
  reverse proxy Orchestarr can set up (Caddy with automatic HTTPS) or through a
  VPN such as Tailscale. Nothing is exposed to the internet by default.
- Images are **pinned** to tested versions; updates go through Maestarr, which
  backs up an app's config before an upgrade that migrates its database.
- Containers run as one unprivileged user (`PUID`/`PGID`) that owns `/data`.
- Each container gets a memory limit sized to the host, so a small NAS stays
  responsive.

## Planned repository layout

```
orchestarr/
├── compose.yml              # every service, behind profiles
├── install.sh               # one-line installer: checks Docker, writes .env, starts bootstrap
├── engine/                  # Python service: profiles → containers, health, configurators
│   ├── configurators/       # one module per app (jellyfin.py, sonarr.py, prowlarr.py, …)
│   ├── seeds/               # config.xml / qBittorrent.conf templates
│   └── tests/               # each configurator against a real container in CI
├── presets/                 # TRaSH-based profiles, language and indexer presets
└── docs/                    # the documentation site
```

Maestarr gains an **"Install"** mode in its onboarding and an admin toolset;
those changes land in the Maestarr repository.

## Roadmap

- [ ] **M0 — skeleton**: compose with profiles, socket proxy, engine that starts
      and health-checks a chosen set of apps
- [ ] **M1 — the core loop**: Jellyfin + Sonarr + Radarr + Prowlarr + qBittorrent
      fully configured from the wizard; a title searched in Maestarr plays in
      Jellyfin with no manual step
- [ ] **M2 — VPN**: Gluetun with port forwarding and the health check, qBittorrent
      locked behind it, Prowlarr's indexer proxy
- [ ] **M3 — the rest**: Bazarr, Seerr, Recyclarr, FlareSolverr, Lidarr, Readarr
- [ ] **M4 — admin by chat**: Maestarr tools to read and change every app's
      settings, with diffs and approvals
- [ ] **M5 — day two**: upgrades with backups, moving the library, adding disks,
      repairing a broken link between two apps
- [ ] **M6 — reach**: reverse proxy with HTTPS, Tailscale option, multi-arch
      images, a docs site, one-line installer

## Non-goals

- Orchestarr does not ship or recommend sources of content. It configures
  software you choose; what you download is your responsibility and your
  country's law applies.
- It is not a fork of the *arr apps. They run unmodified, from their official or
  well-known images, and keep their own UIs.
- It does not replace your router, firewall or VPN account.

## Status and contributing

This is a plan. The first milestone starts once Maestarr's current work is
finished. Ideas and critique of the design are welcome as issues.

Like Maestarr, Orchestarr will be built largely with an AI coding agent
(Claude Code) and reviewed by a human — read the code before you trust it with
your server.

## License

[GPL-3.0](LICENSE)
