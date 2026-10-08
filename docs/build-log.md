# Build log

How the stack was put together, stage by stage, and why each choice was made.
This is history and first-run wiring notes; for running the stack see the [README](../README.md).

## Stage 1 — Jellyfin only

Goal: confirm whether the target hardware can Direct Play the media library, before adding any
other services. This is the only thing that determines if a Raspberry Pi is viable or a mini PC
with Quick Sync is needed.

### Setup

1. Copy the env template and adjust if needed (defaults are fine for local testing):
   ```bash
   cp .env.example .env
   ```

2. Start the stack:
   ```bash
   docker compose up -d
   ```

3. Open the web UI:
   ```
   http://localhost:8096
   ```

4. Run the setup wizard, then add a library pointing at `/media` (inside the container —
   this maps to `${MEDIA_ROOT}/media` on the host).

5. Drop a test video file (ideally a mix: a plain 1080p H.264 file and, if you have one, a
   4K HEVC file) into `data/media/` and let Jellyfin scan.

6. Play it from the TV/Xiaomi streamer app. In Jellyfin's dashboard, under
   **Playback** (or the active session view), check whether it shows **Direct Play** or
   **Transcode**. This result is what decides the hardware plan.

### Notes

- Host paths come from `CONFIG_ROOT` and `MEDIA_ROOT` in `.env` (see Stage 5 — originally a
  single `DATA_ROOT`).
- `.env` and everything under `data/` are git-ignored on purpose: secrets and large/local media
  never belong in the repo.

## Stage 2 — qBittorrent + Prowlarr

qBittorrent runs directly on the LAN for now (no VPN) — a deliberate, revisitable choice, not an
oversight. Prowlarr manages indexers and is never routed through a VPN regardless (it only makes
normal HTTPS requests, it never joins a torrent swarm).

Adding a VPN later: see [vpn.md](vpn.md).

## Stage 3 — Sonarr + Radarr

Sonarr (TV) and Radarr (movies) track what you want and pull it in automatically via Prowlarr +
qBittorrent, instead of searching manually. They share the exact same `/media` and `/downloads`
container paths as qbittorrent/jellyfin on purpose, so a finished download can be hardlinked
straight into the library (same file, no duplicate copy on disk).

Setup order: create an account on first login for each, connect qBittorrent as the download
client (host: `qbittorrent`, port `8080` — same as we did in Prowlarr), then connect each to
Prowlarr under Settings -> Apps so they inherit all configured indexers automatically. Finally
point each at its media folder (`/media/shows` for Sonarr, `/media/movies` for Radarr) as a Root
Folder.

## Stage 4 — Seerr + Bazarr + Recyclarr

Done (2026-09-26). Seerr is the request UI, connected to Jellyfin (as the media server) and to
Sonarr/Radarr (both marked as the default server for their type — required, or requests never get
processed). Bazarr is connected to Sonarr/Radarr and configured with Wizdom + Ktuvit (Hebrew) and
OpenSubtitles.com (English), verified working end-to-end including on the TV apps. Recyclarr syncs
the `web-1080p` (Sonarr) and `hd-bluray-web` (Radarr) TRaSH Guide profiles daily; its per-instance
configs live in `${CONFIG_ROOT}/recyclarr/configs/`, referencing `SONARR_API_KEY` / `RADARR_API_KEY`
from `.env` via `!env_var`.

Note: requesting a season through Seerr triggers an immediate search for every monitored episode
in that season, which can grab a season-pack release instead of a single episode. For a
single-episode test, add the series directly in Sonarr/Radarr with Monitor set to None, then
monitor and Interactive-Search just the one episode.

## Stage 5 — Split config and media roots (laptop → desktop)

**Before (laptop, Stages 1–4):** one `DATA_ROOT=./data` in `.env` held everything —
`data/appdata/<app>` (config), `data/media` (library), `data/torrents` (downloads).

**Now:** two variables, so config sits on the fast drive and media on the big one:

| Variable | Holds | Laptop | Desktop |
|---|---|---|---|
| `CONFIG_ROOT` | `<app>/` config + databases | `./data/appdata` | `./data/appdata` (SSD, inside the repo) |
| `MEDIA_ROOT` | `media/{shows,movies}` + `torrents/` | `./data` | `D:/streameron` (HDD) |

Only host paths changed — container paths (`/config`, `/media`, `/downloads`) are identical, so
every app setting, root folder and connection carries over untouched. On the laptop the new
values resolve to exactly the old folders.


Moving the stack to another machine: see [moving-hosts.md](moving-hosts.md).

## Stage 6 — Tailscale (remote access)

See [remote-access.md](remote-access.md).
