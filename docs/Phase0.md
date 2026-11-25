
> Goal: installation of the OS + hardening the access + installation of docker. 

---
## 1. Installer boot

- At power-on, press **Esc** to reach the Startup Menu, then **F9** for Boot Device Options (one-time boot) and select the USB installer.
- From the Debian installer menu, selected **Install** (text-mode, not Graphical install — faster on this hardware).

---
## 2. Partitioning

- Disk: `SCSI1 (0,0,0) (sda) — ATA KINGSTON SA400S3, 240.1 GB`
- Partitioning scheme: **All files in one partition**
- Partitioning method: **Guided partitioning** (no LVM, no encryption)
- Result: `#1 primary 232.0 GB ext4 /`, `#5 logical 8.1 GB swap`
- Confirmed with **Finish partitioning and write changes to disk**

**Why single partition, no LVM, no encryption:**

- Docker stores everything under `/var/lib/docker`; a separate fixed-size `/var` partition risks running out of room as containers/volumes/media grow. One partition gives Docker the whole 232GB to work with.
- Encrypted LVM would require a passphrase typed at the physical console on every boot, which conflicts directly with the goal of fully remote (Tailscale/Cockpit) manageability.

---
## 3. Mirror configuration

- Country: **Italy**
- Mirror: default offered (**deb.debian.org**)

**Why:** `deb.debian.org` is Debian's CDN-backed mirror, it auto-routes to a fast nearby mirror and won't go stale/offline the way a single physical mirror occasionally does. No proxy needed.

---
## 4. Software selection

At the tasksel screen, Unchecked:

1. **Debian desktop environment**
2. **MATE**
3. **SSH server**

**Standard system utilities** stays checked.

**Why:** Phase 0 calls for a minimal, headless install, no desktop environment, no GUI packages. Standard system utilities is just base CLI tooling, not GUI-related, so it stays.

---
## 5. GRUB bootloader

- "Install the GRUB boot loader to the master boot record?" → **Yes**
- Device: **`/dev/sda`** (the ssd disk, not the USB installer stick)

**Why:** GRUB has to go on the disk you'll actually boot from going forward, not the removable installer media.

---
## 6. First login and sudo

First login was as the regular user, created during install. Running `sudo apt update` immediately failed:

```
-bash: sudo: command not found
```

Software selection left `sudo` uninstalled since it's not part of a minimal base install. Fixed by switching to the root account (root password was set during install) and installing it from there:

```
su -
apt update
apt install sudo
usermod -aG sudo <username>
exit
exit
```

Logged out and back in so the new group membership took effect, then confirmed:

```
groups
```

showed `sudo` listed for `rafz`, and `sudo apt update` worked with a password prompt afterward.

---
## 7. SSH key-only authentication

Generated a key pair on **the client device** (not the **server**) and copied it over:

```
ssh-keygen -t ed25519
ssh-copy-id <username>@<server-lan-ip>
```

First attempt failed:

```
ssh: connect to host <ip> port 22: Connection refused
```

Cause: the **SSH server** package itself had been unchecked at the tasksel screen (step 4), so nothing was listening on port 22 at all, not a firewall or permissions issue. Fixed by installing and enabling it:

```
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

Re-ran `ssh-copy-id`, which succeeded after accepting the host key fingerprint and entering the account password once. Confirmed key-based login worked in a **separate terminal window**, without closing the first session:

```
ssh <username>@<server-lan-ip>
```

Only once that was confirmed, edited the server config:

```
sudo nano /etc/ssh/sshd_config
```

Set:

```
PasswordAuthentication no
PermitRootLogin no
```

Applied with:

```
sudo systemctl restart sshd
```

Verified password auth was actually refused:

```
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password <username>@<server-lan-ip>
```

→ returned `Permission denied (publickey)` immediately, no password prompt, confirming the server itself refuses password attempts rather than the client simply preferring the key.

```
sudo sshd -T | grep -i passwordauth
```

→ dumps the effective merged config (including any drop-in files under `/etc/ssh/sshd_config.d/` that could silently override the main file), confirming `passwordauthentication no` was the value actually in force.

---
## 8. Unattended security upgrades

```
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Answered **Yes** at the prompt.

**Why:** the box runs headless and unattended for long stretches; automatic security patching matters more here than on a daily-use desktop, especially once it's exposed to a **honeypot** in a later phase.

---
## 9. Firewall -- ufw, default-deny

```
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

Confirmed output showed `Status: active`, default policy `deny (incoming)`, and the explicit `22/tcp ALLOW` rule.

**Why the SSH allow rule first:** the box was already going default-deny from this point forward; forgetting the port-22 allow before `ufw enable` would have cut off.

---
## 10. Docker

```
curl -fsSL https://get.docker.com | sh
```

Installed cleanly, Debian 13 (Trixie) is officially supported by Docker's repo. Enabled/started the daemon automatically (`systemctl enable --now docker.service`).

Added the user to the `docker` group so `sudo` isn't needed for every Docker command:

```
sudo usermod -aG docker <username>
```

Logged out and back in for the group change to apply, then confirmed with the standard test container:

```
docker run hello-world
```

This confirms that docker is working.

---

**Phase 0 complete.** Noted OS deviation: running Debian 13 (Trixie)  with ssh hardened and firewall + docker setup correctly.

---

> Next phase: [Phase1](Phase1).