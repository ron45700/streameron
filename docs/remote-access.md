# Remote access (Tailscale)

No router ports opened. Two separate machines in the tailnet, with different reach:

| Tailnet machine | What it is | Reaches | Who uses it |
|---|---|---|---|
| `jellyfin` | the `tailscale` container in this compose file | **only** Jellyfin, via `https://jellyfin.<tailnet>.ts.net` | friends (shared with them) + me |
| the host PC | Tailscale Windows app on the host | every service (`http://<host>:5055`, `:8989`, ...) | only me |

Friends get the `jellyfin` machine via **Share** (admin console), never an invite to the tailnet —
a share gives access to that one machine and nothing else, and that machine only serves Jellyfin
(`tailscale/serve.json`). Inside Jellyfin they get a normal non-admin user limited to chosen
libraries. No Seerr account.

Jellyfin sees these requests as coming from the `tailscale` container's Docker IP (a private
address), so `EnableRemoteAccess=false` in Jellyfin's network settings does not block them and
needs no change.

## One-time setup (admin console)

1. DNS page: enable **MagicDNS**, then **HTTPS Certificates** (needed for the `https://` name).
2. Keys page: generate an auth key (not reusable, not ephemeral), put it in `.env` as `TS_AUTHKEY`.
3. `docker compose up -d tailscale`, then check `docker logs tailscale` and that `jellyfin`
   appears on the Machines page.
4. Machines page → `jellyfin` → **Disable key expiry** (otherwise it logs out after the expiry
   period and friends lose access silently).
5. Install the Tailscale Windows app on the host and sign in with the same account (my own access).
6. Jellyfin → Dashboard → Users → add a user per friend: not administrator, no deletion, only the
   wanted libraries.
7. Later: Machines → `jellyfin` → Share → invite each friend (they need a free Tailscale account).

## Checking stream quality

`docker exec tailscale tailscale status` — a friend's device should show `direct`, not
`relay "..."`. Relayed (DERP) connections are slower and can stutter on high-bitrate files.


Moving the Tailscale machine to another host: see [moving-hosts.md](moving-hosts.md#tailscale-identity).
