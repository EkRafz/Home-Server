> Goal: get the daily-use services running (Jellyfin, Samba, FileBrowser), reachable both on LAN and over Tailscale.

---

## 1. Directory layout

```bash
mkdir -p ~/server/{jellyfin/config,jellyfin/cache,minecraft/data,samba/media,samba/docs,filebrowser/config,filebrowser/database}
```

- `samba/media` and `samba/docs` hold the actual served content.

---
## 2. docker-compose.yml 

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: 1000:1000
    volumes:
      - ./jellyfin/config:/config
      - ./jellyfin/cache:/cache
      - ./samba/media:/media
    ports:
      - "8096:8096"
    restart: unless-stopped

  samba:
    image: dperson/samba:latest
    container_name: samba
    environment:
      TZ: "Europe/Rome"
    volumes:
      - ./samba/media:/media
      - ./samba/docs:/docs
    ports:
      - "445:445"
      - "139:139"
    command: >
      -u "rafz;${SAMBA_PASSWORD}"
      -s "media;/media;yes;no;no;<username>"
      -s "docs;/docs;yes;yes;no;<username>"
    restart: unless-stopped

  filebrowser:
    image: gtstef/filebrowser:latest
    container_name: filebrowser
    volumes:
      - ./samba/media:/srv/media
      - ./samba/docs:/srv/docs
      - ./filebrowser/database:/database
      - ./filebrowser/config:/config
    ports:
      - "8080:80"
    restart: unless-stopped
```

- Samba: `media` share read-only, `docs` share read-write.
- FileBrowser mounted on top share specifically, as a UI layer, not a replacement for Samba.

---
## 3. Samba password via `.env`

Created `~/server/.env`:

```
SAMBA_PASSWORD=<actual password>
```

Compose file references it as `${SAMBA_PASSWORD}`. Docker Compose reads `.env` automatically from the same directory.

To change the Samba password later without touching the compose file:

```bash
docker exec -it samba smbpasswd -a <username>
```

## 4. Disk space check before adding media

```bash
df -h
lsblk
```

Result: single disk (`sda`, 223.6G), single partition (`sda1`, mounted at `/`), ~199GB free. The other block devices (`sdb`–`sdf`) are empty card-reader/USB slots, not usable storage, no separate data disk to split onto.

Budget decided: media capped around 150–160GB, leaving 30–40GB+ free for Docker images, OS updates, and Cowrie/CrowdSec logs once [Phase 4](Phase4.md) starts.

No media was available to add at this point, `~/server/samba/media` was left empty. Confirmed this doesn't block testing; Jellyfin/Samba just show an empty library/share until content is added later.

---
## 5. Bringing the stack up

```bash
cd ~/server
docker compose up -d
docker compose ps
```

All three services (`jellyfin`, `samba`, `filebrowser`) came up `healthy` on first try.

Logs checked individually to confirm clean startup:

```bash
docker compose logs jellyfin
docker compose logs samba
docker compose logs filebrowser
```

---
## 6. Problem: original FileBrowser image is archived/unmaintained

First `filebrowser/filebrowser:latest` run logged:

```
NOTICE: The project is archived on 2026-09-01, after which there will be no
NOTICE: further releases and no security fixes.
```

Given the project's security focus, running a permanently unpatched web app was not ideal.

### Swap steps

```bash
cd ~/server
docker compose stop filebrowser
docker compose rm -f filebrowser
rm -rf ~/server/filebrowser/config/* ~/server/filebrowser/database/*
```

Compose service block updated to use `gtstef/filebrowser:latest` (FileBrowser Quantum, maintained fork), with volumes changed from separate `/config` + `/database` mounts to a single `/home` mount, matching that image's expected layout.

```bash
docker compose up -d filebrowser
docker compose logs filebrowser
```

New image generated its own random initial admin password on first boot, found via:

```bash
docker compose logs filebrowser | grep -i -E "admin|password|listen|port"
```

Logged in with the generated password, then changed it immediately via the web UI (Settings → admin user).

---
## 7. Problem: FileBrowser unreachable over Tailscale ("connection refused")

Symptom: `curl -v http://<tailscale-ip>:8080` from the laptop gave `Connection refused` after only a few ms, not a timeout.

### Diagnosis

1. **Confirmed test was being run from the right machine.** First attempt was accidentally run on the server itself (same source/destination IP, sub-0.1ms ping gave it away) — re-ran properly from the actual laptop.
2. **Checked `ufw` status**, found only port 22 allowed (what I did in [Phase 0](Phase0.md)), default-deny on everything else. This was real and needed fixing regardless, but turned out not to be the actual blocker for this specific symptom:

```bash
   sudo ufw status verbose
```
   
3. **Added the Tailscale interface as an allow rule** (correct fix, needed either way):
   
```bash
   ip a | grep tailscale
   sudo ufw allow in on tailscale0
```
   
4. Still refused after that. Traced through `iptables` chains step by step:
   
```bash
   sudo iptables -L ufw-user-input -n -v --line-numbers
   sudo iptables -L ufw-before-input -n -v --line-numbers
   sudo iptables -L ufw-not-local -n -v --line-numbers
   sudo iptables -L DOCKER-USER -n --line-numbers
```
   
   All of these looked correct, `tailscale0` rule was present and properly scoped, no stray REJECT/DROP rules targeting port 8080.
5. **Used `tcpdump` to see the actual packet behavior**, rather than continuing to infer from static rule counters:

```bash
   sudo tcpdump -ni tailscale0 tcp port 8080
```
   
   While running `curl` from the laptop. Result: SYN arrives fine, server replies with an immediate RST. This proved the packet was reaching the host fine, `ufw`/Tailscale routing was not the problem. A real `ufw` drop would cause silence/timeout, not an instant reset.
   
6. **Compared against Jellyfin (port 8096)**, which worked perfectly over the same interface — confirming the issue was specific to this one port/container, not systemic.
7. **Checked what the container itself was actually listening on, from inside it:**

```bash
   docker exec -it filebrowser sh
   netstat -tlnp 2>/dev/null || ss -tlnp
```

   Found it listening on `:::80` internally, not `8080`. The compose file's port mapping (`8080:8080`) was therefore pointing at a port nothing was listening on inside the container, causing the kernel inside the container namespace to instantly RST every connection attempt.

### Fix

Changed the port mapping in `docker-compose.yml`:

```yaml
ports:
  - "8080:80"
```

```bash
docker compose up -d filebrowser
```

Retested from the laptop, got a real HTTP response. FileBrowser login then worked.

## 8. Jellyfin "Allow remote connections" setting

During the Jellyfin setup wizard, enabled **"Set up Remote Access" → Allow remote connections to this server**.

Reasoning: Jellyfin's own startup logs define its internal "LAN subnets" as `127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16`, Tailscale's range isn't included, so without this setting Jellyfin would likely treat Tailscale traffic as "remote" and restrict it. This doesn't add new exposure to the open internet; Tailscale remains the only path in, nothing is port-forwarded on the router for Jellyfin.

## 9. Samba connection test

Realized `curl` can't test SMB (`\\host\share` is Windows UNC syntax, not a URL). Tested properly instead via Ubuntu's Files app:

- **Other Locations → `smb://<tailscale-ip>/`**, entered credentials (`<username>` + Samba password).
- Confirmed read-write access on `docs`: connected, browsed, created/deleted a file through the GUI.

---

**Phase 2 status:** core services up, reachable via both LAN and Tailscale, FileBrowser swapped to a maintained image and firewall-reachable after fixing a port-mapping mismatch. Remote playback test pending actual media content.

---

> Next phase: [Phase3](Phase3): Portainer + Cockpit remote management.
