# Moving to a new host

1. Old host: in qBittorrent remove all torrents (keep files); in Sonarr/Radarr remove or
   unmonitor test items whose files won't be moved; then `docker compose down` (never copy
   the SQLite databases while apps are running).
2. New host: `git clone`, create `.env` from `.env.example`, set `CONFIG_ROOT`/`MEDIA_ROOT`,
   copy `SONARR_API_KEY`/`RADARR_API_KEY` from the old `.env`.
3. Copy only the old `CONFIG_ROOT` folder to the new `CONFIG_ROOT`. Media is not copied.
4. Create `media/shows`, `media/movies`, `torrents` under `MEDIA_ROOT`; `docker compose up -d`.
5. Pre-existing media: place as `media/shows/<Show (Year)>/Season 01/...`, then in Sonarr
   Series → Library Import → `/media/shows` (Radarr: Movies → Library Import) so they're
   tracked, not re-downloaded. Then Jellyfin → Scan All Libraries.
6. Re-add the server on the TVs with the new host's LAN IP (reserve it in the router); update
   any external Jellyfin URL in Seerr; allow Docker in Windows Firewall; disable sleep.

Known caveat: Docker Desktop on Windows likely can't hardlink between `/downloads` and
`/media`, so Sonarr/Radarr copy instead — a finished download takes double space while it
stays in qBittorrent.


## Tailscale identity

`${CONFIG_ROOT}/tailscale/` is the machine's identity (private key — treat as a secret, never
commit). Copy it to the new host together with the rest of `CONFIG_ROOT` and the machine keeps
its name and shares. Never run the same state on two hosts at once — stop the old one first.
The Jellyfin users live in Jellyfin's database; create them on the host that's actually in use
rather than overwriting its database.

