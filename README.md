```
██╗  ██╗ ██████╗ ███╗   ███╗███████╗███████╗███████╗██████╗ ██╗   ██╗███████╗██████╗ 
██║  ██║██╔═══██╗████╗ ████║██╔════╝██╔════╝██╔════╝██╔══██╗██║   ██║██╔════╝██╔══██╗
███████║██║   ██║██╔████╔██║█████╗  ███████╗█████╗  ██████╔╝██║   ██║█████╗  ██████╔╝
██╔══██║██║   ██║██║╚██╔╝██║██╔══╝  ╚════██║██╔══╝  ██╔══██╗╚██╗ ██╔╝██╔══╝  ██╔══██╗
██║  ██║╚██████╔╝██║ ╚═╝ ██║███████╗███████║███████╗██║  ██║ ╚████╔╝ ███████╗██║  ██║
╚═╝  ╚═╝ ╚═════╝ ╚═╝     ╚═╝╚══════╝╚══════╝╚══════╝╚═╝  ╚═╝  ╚═══╝  ╚══════╝╚═╝  ╚═╝
```



**Hardware:** HP Compaq 6005 Pro SFF, 8GB DDR3, 250GB storage

## Goal

Turn the HP box into a self-hosted home server: media, Minecraft, LAN file sharing, secure remote access (no port forwarding except one deliberate exception), phone-manageable, plus a honeypot/active-defense setup with periodic ML log analysis.

## Services (Docker, 8GB RAM shared)

| Service | Purpose | Always on? |
|---|---|---|
| Tailscale | Remote access gate for everything but Cowrie | Yes |
| Portainer | Container management | Yes (Tailscale-only) |
| Cockpit | Host management (reboot, updates, system graphs) | Yes (Tailscale-only) |
| Jellyfin | Media streaming | Toggle |
| Samba | LAN/remote file sharing | As needed |
| FileBrowser | Web UI over Samba | As needed |
| Cowrie | SSH/Telnet honeypot | Toggle |
| CrowdSec | Auto-bans IPs on real services | Toggle |
| ML container | Clusters honeypot logs | On-demand, not with Minecraft |

## Architecture

Remote devices connect via Tailscale (no port forwarding) to reach Portainer, Cockpit, and whichever services are running. Cowrie sits alone on port 22, the only thing actually exposed to the internet. CrowdSec watches real service logs; the ML container batch-processes Cowrie/CrowdSec logs on demand.

## Key decisions

- **Tailscale** is the only access path except Cowrie. No router forwarding for any other service.
- **Cowrie takes port 22** (real SSH moved elsewhere) to catch the bulk of scanner traffic. It's fully isolated: own Docker network, no shared volumes, firewall rules blocking it from reaching anything else, since it's the one thing facing the open internet.
- **CrowdSec does not watch Cowrie** (auto-banning honeypot traffic would defeat its purpose), only real services (SSH, Samba, Portainer/Cockpit).
- **Portainer** handles containers day-to-day; **Cockpit** handles host-level actions (reboot/shutdown need real auth and a confirm step).
- **Samba** with separate read-only (media) and read-write (docs/backups) shares, real accounts, no open guest share. **FileBrowser** adds a lighter UI on top instead of Nextcloud.
- **ML job**: DBSCAN over session duration, command count, command sequence (TF-IDF/n-grams), credentials tried, and optionally IP geo/ASN. Cross-references outlier IPs against CrowdSec's blocklist. Runs locally, batch only, never alongside

## Build order

1. Tailscale, confirm remote access from outside the home network
2. Jellyfin, Samba, FileBrowser, confirm LAN + Tailscale access
3. Portainer + Cockpit, confirm Tailscale-only and auth/confirm steps
4. Cowrie + CrowdSec, deploy isolated, let run a few weeks to collect data
5. Review real logs
6. ML container, build and toggle against real data

## Security checklist

- Only port 22 (Cowrie) ever faces the internet directly
- Per-service Docker network isolation (mandatory for Cowrie, recommended elsewhere); non-root containers where possible
- Keep images updated (Watchtower or manual cadence); Cockpit handles OS updates
- SSH key-only auth, no root login, automatic security updates, default-deny firewall
- Log rotation with a retention window (e.g. 30 days), revisit once real volume is visible
- Cockpit reboot/shutdown requires real auth + confirm step