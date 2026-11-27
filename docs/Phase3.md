> Goal: manage containers (Portainer) and the host (Cockpit) remotely, reachable only over Tailscale.

> **Note**: at first I installed portainer, but then I came to the decision to eliminate it. Down below there is, in any case, the portainer installation and everything I did the portainer.

---

## 1. Two facts that shaped the setup

- **`ufw` does not filter Docker-published ports.** Docker inserts its own iptables rules ahead of ufw's, so any port published as `8080:80` is open to the whole LAN regardless of `ufw status`. This applies to the [Phase 2](Phase2.md) services (LAN + Tailscale by design), but Portainer must be Tailscale-only, so its port is **bound to the Tailscale IP** instead of relying on ufw.
- **`cockpit-podman` manages Podman, not Docker.** The project plan assumed it gave container stats. It doesn't apply here, so it was skipped. Portainer is the container tool; Cockpit is host-level only.

---

## 2. Docker must start after Tailscale

Binding a container port to the Tailscale IP fails if Docker starts before `tailscaled` has created the interface. Fixed with a systemd drop-in:

```bash
sudo systemctl edit docker
```

Added at the top of the file (above the commented original unit file, which is left untouched):

```ini
[Unit]
After=tailscaled.service
Wants=tailscaled.service
```

Checked it applied:

```bash
systemctl cat docker | tail -n 8
```

---

## 3. Portainer

Separate compose file in `~/server/portainer/docker-compose.yml`, kept out of the Phase 2 stack:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:lts
    container_name: portainer
    ports:
      - "<tailscale-ip>:9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
    restart: unless-stopped
```

```bash
cd ~/server/portainer
docker compose up -d
```

- Tailscale IP from `tailscale ip -4`.
- Mounting `docker.sock` is root-equivalent on the host, which is why the Tailscale-only bind matters.
- Opened `https://<tailscale-ip>:9443` and created the admin account straight away. Portainer locks the setup page if no admin is created within about 5 minutes (fix: restart the container).
- Strong unique password.
- **Edge Compute wizard: skipped.** It's for managing remote Docker hosts Portainer can't reach directly. Single host here, so it only adds attack surface.
- Local environment shows as Up with the Phase 2 containers visible.
- **2FA:** Portainer CE doesn't appear to offer it (verify under Settings → Authentication). Protection is Tailscale-only access + strong password.

---

## 4. Cockpit

Installed on the host (not in Docker):

```bash
sudo apt install cockpit
```

### Login restrictions

```bash
cat /etc/cockpit/disallowed-users
```

Output: `root`. Root login is already blocked. Logging in as the normal user with the Linux account password (PAM, same as `sudo`). Not the Samba/Portainer/FileBrowser passwords.

### Bind to the Tailscale IP only

```bash
sudo systemctl edit cockpit.socket
```

```ini
[Socket]
ListenStream=
ListenStream=<tailscale-ip>:9090
FreeBind=yes
```

- The empty `ListenStream=` line clears the default `0.0.0.0:9090` listener. Without it Cockpit stays open on all interfaces.
- `FreeBind=yes` lets the socket bind before the Tailscale interface exists, so boot order doesn't matter.

```bash
sudo systemctl daemon-reload
sudo systemctl restart cockpit.socket
```

### WebSocket origin

`/etc/cockpit/cockpit.conf` (needs `sudo`):

```ini
[WebService]
Origins = https://<tailscale-ip>:9090 wss://<tailscale-ip>:9090
```

Without this, accessing Cockpit by IP can give a blank page or a login loop.

### Verified bind

```bash
sudo ss -tlnp | grep 9090
```

```
LISTEN 0  4096  <tailscale-ip>:9090  0.0.0.0:*  users:(("systemd",pid=1,fd=78))
```

Only the Tailscale IP, no `0.0.0.0` listener.

### Admin access

Logging in gives limited access. Clicking **Limited access** (top right) and re-entering the password switches to administrative access. This is what enables reboot/shutdown, with a confirm step before the action.

---

**Phase 3 status:** Portainer (deleted) and Cockpit running, both bound to the Tailscale IP only, surviving reboot.

---

> Next phase: [Phase4](Phase4). Move real SSH off port 22, then Cowrie + CrowdSec.