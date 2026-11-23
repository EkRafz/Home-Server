
**Hardware:** HP Compaq 6005 Pro Small Form Factor, 8GB RAM ddr3, 250GB storage

---

## 1. Project Goal

Turn the HP Compaq 6005 Pro into a self-hosted home server that:

- Serves media, hosts a Minecraft server, and shares files on the home LAN
- Is reachable securely from anywhere in the world, with no router port forwarding except one deliberate exception
- Is manageable (containers, updates, reboot/shutdown, resource usage) from a phone, anywhere
- Runs a honeypot and active defense tooling for a genuine, hands-on security angle
- Periodically analyzes collected security logs with a simple ML clustering job



---
## 2. Services

All services run as Docker containers, each independently start/stoppable. Not all run simultaneously, RAM is the limit (8GB total).

| Service              | Purpose                                                                              | Always on?                               |
| -------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------- |
| **Tailscale**        | Remote access, the access gate for everything below except Cowrie                    | Yes, always on                           |
| **Portainer**        | Container management: start/stop, logs, per-container resource usage                 | Yes (reachable only via Tailscale)       |
| **Cockpit**          | Host-level management: reboot, shutdown, OS updates, system-wide CPU/RAM/disk graphs | Yes (reachable only via Tailscale)       |
| **Jellyfin**         | Media streaming                                                                      | Toggle - competes with Minecraft for RAM |
| **Minecraft server** | Game server for you/friends                                                          | Toggle - competes with Jellyfin for RAM  |
| **Samba**            | File sharing on LAN (and remotely via Tailscale)                                     | As needed, lightweight                   |
| **FileBrowser**      | Browser/app-style file access on top of Samba shares                                 | As needed, lightweight                   |
| **Cowrie**           | SSH/Telnet honeypot, logs real attacker behavior                                     | Toggle                                   |
| **CrowdSec**         | Actively bans IPs failing auth repeatedly on real services                           | Toggle on once deployed (lightweight)    |
| **ML container**     | Batch job, clusters honeypot/ban logs to surface unusual sessions                    | Toggle, on-demand only                   |

> Known conflict: **Jellyfin + Minecraft should not run simultaneously**. The ML container should also avoid running at the same time as Minecraft, since both are resource-heavy.

---

## 3. Architecture

```
Personal phone / laptop (anywhere in the world)
  → Tailscale connected
  → Device now reaches the box directly, no port forwarding involved
  → Reach Portainer (container control) and Cockpit (host control)
  → Reach Jellyfin / Samba / FileBrowser / Minecraft directly, if running
  → (Behind the scenes, unrelated to your access:)
     Cowrie logs attacker activity on its own forwarded port (22)
     CrowdSec watches real service logs (not Cowrie's), auto-bans abusers
     ML container periodically batch-processes those logs
```

### Network placement

- HP box connects to **the secondary router**
- **No router port forwarding** for Tailscale, Jellyfin, Samba, FileBrowser, Portainer, Cockpit, or Minecraft, Tailscale handles NAT traversal itself, nothing needs to be exposed.

---

## 4. Remote Access (Tailscale)

- Tailscale installed on the HP box, phone, and laptop, all under the same account. No router config needed, works through CGNAT.
- Devices reach each other directly once signed in, no port forwarding, no static-IP dependency for this part.
- **This is the single access gate for everything except Cowrie**, nothing else should be reachable from the open internet.

---

## 5. Honeypot (Cowrie) -- the deliberate exception

- Fakes an SSH/Telnet server. Lets connecting bots "log in" with common credentials, gives them a fake shell, logs everything: credentials tried, commands run, files they attempt to fetch. Nothing they do is real or persists.
- **Port decision: real SSH moved off 22 (not forwarded on the router at all), Cowrie takes port 22.** The large majority of scanner/bot traffic specifically targets port 22, a honeypot on a decoy port gets a fraction of the volume, since most scanners don't bother sweeping non-standard ports. Maximizing real attacker data means Cowrie belongs on 22, not a decoy.
- **Isolation is mandatory, not optional**, since this is the only thing actually facing the open internet: runs on its own Docker network with no route to any other container's network, no shared volumes, and host firewall rules (`ufw`/`nftables`) that explicitly block the Cowrie container from reaching anything else on the box, even if Cowrie itself is fully compromised.

---

## 6. Remote Management

Two tools, covering different layers:

### Portainer (container layer)
- Start/stop/restart containers, view logs, per-container resource usage, image/compose management.
- The day-to-day tool for toggling Jellyfin vs. Minecraft vs. the ML job.

### Cockpit (host layer)
- Reboot, shutdown, OS package updates, live CPU/RAM/disk/network graphs for the whole machine.
- Has a Docker/Podman plugin (`cockpit-podman`) giving some container-level stats too, but Portainer remains the primary container-management tool, Cockpit's job is the host-level actions Portainer doesn't do.
- **Reboot/shutdown over the web is real**: Cockpit requires a real login  and has a confirm step before destructive actions, even though it's already Tailscale-gated.

Alternative considered and set aside: a custom FastAPI/Flask dashboard wrapping `docker`/system commands, more build effort, adds its own attack surface, only worth it if the goal is the build experience itself rather than pure convenience.

---

## 7. File Sharing

- **Samba (SMB)** for the actual file serving, works across Windows/Mac/Linux, standard for mixed-device households.
- Separate shares by purpose: read-only for media, read-write for documents/backups, it limits damage if one device/account is compromised.
- Real user accounts with real passwords, not an open guest share.
- Never exposed directly to the internet, LAN access is native, remote access works automatically once Tailscale is connected (same `\\<tailscale-ip>\share` path, no separate config).
- **Note on remote Jellyfin specifically:** streaming to a device away from home rides your home connection's **upload** speed, not download, worth confirming this is actually sufficient before relying on it.
- **FileBrowser** on top: gives a clean browser/app-style file UI without Nextcloud's heavier footprint (own database, background jobs, broader feature set you don't currently need, contacts/calendar/doc-collab).

---

## 8. Active Defense -- CrowdSec (decided)

- Watches logs from real exposed services (SSH, Samba auth, Portainer/Cockpit login attempts if logged) and auto-bans IPs after repeated failures.
- **CrowdSec** detection engine and its "bouncer" ) are separate pieces, deployed both.
- **Scope: CrowdSec does not watch Cowrie.** It's limited to logs from your real services (SSH, Samba, Portainer/Cockpit) only. Auto-banning IPs that hit the honeypot would suppress the very traffic Cowrie exists to collect, the honeypot is deliberately left unprotected by design, not by oversight.

---

## 9. ML Analysis (Batch Job, Toggleable)

### What it is
A Docker container you toggle on manually. When on, it runs once (batch job, not a persistent service), reads accumulated Cowrie + CrowdSec logs, clusters attack sessions, and exits. Not a chatbot, not autonomous, not real-time, a background sorting job.

### What it's for
Separating "boring, identical bot noise" (the vast majority of what a honeypot sees) from "unusual, worth a human looking at", without you having to read every raw log line yourself. Output is a report: which sessions fell into which cluster, and which clusters look like genuine outliers.

- **Algorithm: starting with DBSCAN.** Doesn't require pre-specifying cluster count, naturally separates dense "normal bot" clusters from sparse outliers (explicitly flagged as noise), fits honeypot data's expected shape (mostly repetitive, few true outliers) better than k-means, and gives explainable clusters (you can say why sessions grouped together) rather than just a black-box score.
- **Features per session:**
	- Session duration
	- Number of commands typed
	- Command sequence, encoded via TF-IDF or simple n-gram counts
	- Credentials tried (default/common vs. unusual)
	- Source IP's ASN/geolocation (optional, later addition)
- **Cross-reference step:** check whether honeypot-flagged outlier IPs also appear in CrowdSec's community blocklist or your own ban list, this comparison is the actual interesting output, not the clustering alone.

### Where it runs
On the HP box, as a second toggleable container — not shipped to another machine. Keeps everything local, avoids building a log-shipping pipeline. Never run concurrently with Minecraft.

---

## 10. Security Checklist (applies across the whole box)

- **Exposure:** only Cowrie's port (22) ever reaches the internet directly, via router port forwarding. Everything else, Tailscale, Jellyfin, Samba, FileBrowser, Portainer, Cockpit, Minecraft, stays Tailscale/LAN-only unless a specific, separate decision is made later (e.g. letting outside friends join Minecraft directly, which would need its own forwarded port and its own risk tradeoff).
- **Docker isolation:** separate Docker networks per service where practical, so a compromised container can't freely reach the others. This is **mandatory** for Cowrie specifically (see §6), recommended for the rest. Run containers as non-root where supported.
- **Updates:** keep container images current (Watchtower for auto-updates, or a regular manual cadence), stale images are a common real compromise path. Cockpit handles OS-level package updates.
- **Host OS:** SSH key-only auth, no root login over SSH, automatic security updates (`unattended-upgrades`), local firewall (`ufw`/`nftables`) default-deny with explicit allows only — including the rule blocking Cowrie's container from reaching the rest of the box.
- **Log retention:** Cowrie (on port 22, facing the open internet) can generate a lot of log volume quickly once discovered by scanners. Set up log rotation with an explicit retention window, (e.g. 30 days) rather than letting logs grow unbounded. Revisit the window once real volume is visible.
- **Remote power control:** Cockpit's reboot/shutdown must require real authentication and a confirm step — it's a genuine way to take the whole box offline by accident if left unguarded.
- **Minecraft specifically:** keep server version patched if ever exposed beyond Tailscale; any decision to let outside friends connect directly is a separate, deliberate tradeoff, not a default.

---

## 11. Build Order

1. **Tailscale**, get remote access working first; confirm it reaches the box correctly from outside your home network (test from external network, not just home wifi)
2. **Jellyfin, Minecraft, Samba, FileBrowser**, get the actually-useful daily-use server running; confirm each works both on LAN and over Tailscale
3. **Portainer + Cockpit**, wire up remote management (containers + host-level), confirm both are Tailscale-only, confirm Cockpit's auth and confirm-step before relying on remote reboot/shutdown
4. **Cowrie + CrowdSec**, deploy with Cowrie's isolation rules in place from day one, then **leave running untouched for a few weeks** to collect real data before building anything on top
5. **Review real logs**, look at what Cowrie/CrowdSec actually collected
6. **ML container**, build the batch clustering job against real data, add the toggle, confirm it never runs alongside Minecraft

---