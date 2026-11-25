
> Goal: reach the server from anywhere in the world with no router port forwarding.

---

## 1. Installation of Tailscale on the server

```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

- The install script adds Tailscale's repo and installs the client + daemon (`tailscaled`).
- `tailscale up` starts the connection and prints a login URL. Opening it in a browser authenticates the device against the Tailscale account and adds it to the tailnet.

## 2. Install on other devices

- **Laptop:** the same `curl | sh` script (I am using a Ubuntu on my laptop). Signed in with the same account.
- **iPhone:** Dowloaded from the App store and signed in with same account.

> All devices must use the **same account** to land in the same tailnet.

## 3. Confirmed devices are connected

```
tailscale status
```

- Lists every device on the tailnet with its Tailscale IP (x.x.x.x range) and connection status.

## 4. Getting the server's Tailscale IP

```
tailscale ip -4
```

- Prints the server's own x.x.x.x address. This is the address used to reach it remotely going forward (instead of the LAN IP), since it works from anywhere once Tailscale is up, independent of local network/router config.

## 5. External-network test

- Disconnected the test device from home wifi and switched another network that isn't the home LAN.
- From that connection:

```
ssh <username>@<server-tailscale-ip>
```

- Success here proves Tailscale is doing NAT traversal correctly. Testing only on home wifi would not actually confirm this.

---

## Admin console follow-ups (console.tailscale.com)

- **Device list check:** confirm server, laptop, phone all show as connected with correct names. Renamed server entry to something identifiable (`homeserver`).
- **Disabled key expiry:** Tailscale auth keys expire after 180 days by default. For an always-on server you manage remotely, this is enabled as a per-device toggle in the admin console so the server doesn't silently drop off the tailnet.
- **Funnel / Serve:** deliberately left off. These expose a device to the public internet via Tailscale's relay, which conflicts with the architecture's core rule (nothing internet-facing except Cowrie via explicit router port forwarding).
- **Tailscale SSH:** not enabled. Already have working key-only SSH auth on the server; adding Tailscale's own SSH layer on top isn't necessary.

---

**Phase 1 complete:** all three devices connected, server reachable via `ssh <username>@<tailscale-ip>` confirmed from an external network.

Next phase: [Phase2](Phase2).
