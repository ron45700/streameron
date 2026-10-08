# Adding a VPN (Gluetun)

qBittorrent currently runs directly on the LAN. The VPN is pre-wired but disabled.

The compose file already has a commented-out `gluetun` service at the bottom, and `.env` /
`.env.example` already have empty `OPENVPN_USER` / `OPENVPN_PASSWORD` fields waiting. To enable:

1. Uncomment the `gluetun` service in `docker-compose.yml`.
2. On the `qbittorrent` service: delete its `ports:` block, add `network_mode: "service:gluetun"`
   and `depends_on: [gluetun]`.
3. Fill the two `OPENVPN_*` values in `.env` (from CyberGhost's Advanced Configuration ->
   Manual setup page — a dedicated OpenVPN login, not the account email/password).
4. `docker compose up -d`.

No restructuring needed — it's a copy/uncomment job.

