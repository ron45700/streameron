# streameron

Self-hosted home media server. Request from phone → auto-download → auto-organize → watch on TV.

Full architecture and component list live in project notes (see chat history) — this README only
covers running what's currently in the repo.

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
   this maps to `${DATA_ROOT}/media` on the host).

5. Drop a test video file (ideally a mix: a plain 1080p H.264 file and, if you have one, a
   4K HEVC file) into `data/media/` and let Jellyfin scan.

6. Play it from the TV/Xiaomi streamer app. In Jellyfin's dashboard, under
   **Playback** (or the active session view), check whether it shows **Direct Play** or
   **Transcode**. This result is what decides the hardware plan.

### Notes

- `DATA_ROOT` defaults to `./data` (a local folder next to the compose file) for laptop testing.
  On the real host this will point at the external HDD mount instead — nothing else changes.
- `.env` and everything under `data/` are git-ignored on purpose: secrets and large/local media
  never belong in the repo.

## Stage 2 — qBittorrent + Prowlarr

qBittorrent runs directly on the LAN for now (no VPN) — a deliberate, revisitable choice, not an
oversight. Prowlarr manages indexers and is never routed through a VPN regardless (it only makes
normal HTTPS requests, it never joins a torrent swarm).

- qBittorrent Web UI: `http://localhost:8080`
- Prowlarr Web UI: `http://localhost:9696`

### Adding a VPN later (Gluetun)

The compose file already has a commented-out `gluetun` service at the bottom, and `.env` /
`.env.example` already have empty `OPENVPN_USER` / `OPENVPN_PASSWORD` fields waiting. To enable:

1. Uncomment the `gluetun` service in `docker-compose.yml`.
2. On the `qbittorrent` service: delete its `ports:` block, add `network_mode: "service:gluetun"`
   and `depends_on: [gluetun]`.
3. Fill the two `OPENVPN_*` values in `.env` (from CyberGhost's Advanced Configuration ->
   Manual setup page — a dedicated OpenVPN login, not the account email/password).
4. `docker compose up -d`.

No restructuring needed — it's a copy/uncomment job.

## Stage 3 — Sonarr + Radarr

Sonarr (TV) and Radarr (movies) track what you want and pull it in automatically via Prowlarr +
qBittorrent, instead of searching manually. They share the exact same `/media` and `/downloads`
container paths as qbittorrent/jellyfin on purpose, so a finished download can be hardlinked
straight into the library (same file, no duplicate copy on disk).

- Sonarr Web UI: `http://localhost:8989`
- Radarr Web UI: `http://localhost:7878`

Setup order: create an account on first login for each, connect qBittorrent as the download
client (host: `qbittorrent`, port `8080` — same as we did in Prowlarr), then connect each to
Prowlarr under Settings -> Apps so they inherit all configured indexers automatically. Finally
point each at its media folder (`/media/shows` for Sonarr, `/media/movies` for Radarr) as a Root
Folder.

## Next stages (not yet in this repo)

4. Seerr + Bazarr + Recyclarr — request UI, subtitles, quality profile sync. **Done** (2026-09-26): connected and verified end-to-end, including Hebrew + English subtitles.
5. Tailscale — remote access to Seerr (and everything else) from outside the home network, without opening router ports.
6. Maintainerr — rule-based auto-cleanup: delete/unmonitor watched or stale media across Jellyfin + Sonarr/Radarr + Seerr, with a grace period before deletion.
