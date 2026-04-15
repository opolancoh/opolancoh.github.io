# Raspberry Pi 3 Setup

Guide for preparing a Raspberry Pi 3 (Model B / B+) to act as a headless home server. Application-specific setup (Docker, nginx, etc.) is covered in separate docs.

## Contents

- [Prerequisites](#prerequisites)
- [1. Choose the OS](#1-choose-the-os)
- [2. Flash the SD Card with Raspberry Pi Imager](#2-flash-the-sd-card-with-raspberry-pi-imager)
- [3. First Boot](#3-first-boot)
- [4. Update the System](#4-update-the-system)
- [5. Install Base CLI Tools](#5-install-base-cli-tools)
- [6. Remote SSH Access (On-Demand)](#6-remote-ssh-access-on-demand)
- [7. Network Configuration](#7-network-configuration)

## Prerequisites

- Raspberry Pi 3 (Model B or B+)
- MicroSD card — 16 GB minimum, A1/A2-rated from a reputable brand (SanDisk, Samsung, Kingston). Cheap cards are a common cause of a "slow" Pi.
- Power supply (5V / 2.5A micro-USB)
- Ethernet cable — recommended for first-time setup, regardless of how the Pi will connect long-term
- A computer to flash the SD card

---

## 1. Choose the OS

For a Pi 3 used as a server, use **Raspberry Pi OS (Legacy, 32-bit) Lite**.

### Why this version

| Factor | Reason |
|---|---|
| **Lite** (no desktop) | Boots to CLI. Much lower idle RAM usage than the Full image. Fewer background processes and a smaller attack surface. |
| **Legacy** | Each Raspberry Pi OS release is based on a Debian stable release. When a new Debian version is released, Raspberry Pi Foundation promotes its image to be the default and relabels the previous one as "Legacy". Legacy is **not** unmaintained — it still receives security updates and is the more mature, better-tested choice for older hardware like the Pi 3. The newest (non-Legacy) image is typically too fresh for production use on a low-resource device. |
| **32-bit** | The Pi 3 has 1 GB RAM. 32-bit uses less memory per process. 64-bit's advantages (>4 GB RAM, larger binaries) don't apply here. |

### Options to avoid on a Pi 3 server

| Image | Why skip |
|---|---|
| Raspberry Pi OS Full (any version) | Ships with a desktop environment you don't need on a server. |
| Raspberry Pi OS (non-Legacy) | Tracks the newest Debian release — more bloat and less battle-tested on Pi 3. |
| 64-bit variants | Wastes RAM on a 1 GB device. |

### Installing a GUI later (on demand)

If you ever need a GUI for troubleshooting, install it after the fact:

```bash
sudo apt update
sudo apt install --no-install-recommends xserver-xorg xinit lxde-core lxterminal
startx
```

Exit the desktop session to return to the CLI. This keeps the Pi in lean headless mode 99% of the time.

---

## 2. Flash the SD Card with Raspberry Pi Imager

1. Download **Raspberry Pi Imager** from [raspberrypi.com/software](https://www.raspberrypi.com/software/).
2. Insert the MicroSD card into your computer.
3. Open Imager and click **Choose Device** → **Raspberry Pi 3**.
4. Click **Choose OS** → **Raspberry Pi OS (other)** → **Raspberry Pi OS (Legacy, 32-bit) Lite**.
5. Click **Choose Storage** → select the MicroSD card.
6. Click **Next** → when prompted *"Would you like to apply OS customisation settings?"*, click **Edit Settings**.

### 2.1 Customisation — General tab

| Setting | Value |
|---|---|
| Set hostname | `rpi-3` (or any name you prefer) — accessible on the LAN as `rpi-3.local` |
| Set username and password | Your own username and a strong password. Avoid the legacy default `pi`. |
| Set locale settings | Your timezone (e.g. `America/Bogota`) and keyboard layout (e.g. `us`) |

> Wi-Fi is intentionally not configured here so the image stays portable across networks. **Connect the Pi to your router via Ethernet for the first boot** — it's the simplest way to get a reliable connection and SSH in, regardless of the network you're on. Wi-Fi (if needed) can be configured after first login.

### 2.2 Customisation — Services tab

| Setting | Value |
|---|---|
| Enable SSH | ✅ Yes |
| Authentication | Password (for now) — or paste the contents of `~/.ssh/id_ed25519.pub` to enable key-only login from first boot (recommended). |

### 2.3 Customisation — Options tab

| Setting | Value |
|---|---|
| Play sound when finished | Optional |
| Eject media when finished | ✅ Yes (safer unplug) |
| Enable telemetry | Personal choice |

### 2.4 Write

Click **Save** → **Yes** to apply customisation → **Yes** to confirm erase → wait for writing and verification to finish (~5–10 min depending on card speed).

---

## 3. First Boot

1. Insert the MicroSD card into the Pi.
2. Plug in an Ethernet cable to your router.
3. Connect power — no monitor or keyboard needed.
4. Wait ~1–2 minutes for first boot. The Pi expands the filesystem and applies the imager customisation.

### Connect via SSH

From your laptop on the same LAN:

```bash
ssh <your-username>@rpi-3.local
```

If `.local` (mDNS) doesn't resolve, find the Pi's IP from your router's DHCP client list and use it directly:

```bash
ssh <your-username>@<pi-ip>
```

---

## 4. Update the System

Always update immediately after first boot to pull in security patches and bug fixes released since the OS image was built.

```bash
sudo apt update && sudo apt full-upgrade -y
```

- `apt update` refreshes the package lists from the repositories.
- `apt full-upgrade` installs the latest versions and handles dependency changes. It's preferred over plain `upgrade` on Raspberry Pi OS, which ships rolling point releases where packages may be added or removed between versions.

Clean up obsolete packages afterwards:

```bash
sudo apt autoremove -y && sudo apt clean
```

Reboot if the kernel or firmware was upgraded (it usually is on a fresh install):

```bash
sudo reboot
```

Reconnect via SSH after ~30 seconds:

```bash
ssh <your-username>@rpi-3.local
```

### Verify the system is healthy

```bash
uname -a              # kernel version
cat /etc/os-release   # OS version
free -h               # RAM usage
df -h /               # disk usage on the root partition
```

---

## 5. Install Base CLI Tools

A small set of utilities useful on any server. Most are already present on Pi OS Lite; this ensures they're installed and up to date.

```bash
sudo apt install -y curl wget git vim htop ca-certificates gnupg lsb-release
```

| Tool | Purpose |
|---|---|
| `curl`, `wget` | HTTP clients for fetching scripts and files |
| `git` | Cloning repos (configs, your own projects) |
| `vim` | Editor for quick config edits over SSH (`nano` is already preinstalled if you prefer it) |
| `htop` | Interactive process/resource monitor — much more readable than `top` |
| `ca-certificates`, `gnupg`, `lsb-release` | Prerequisites for adding third-party apt repositories later (Docker, Node.js, etc.) |

Verify the essentials are installed:

```bash
git --version
htop --version
curl --version | head -1
```

---

## 6. Remote SSH Access (On-Demand)

For a test/dev server, the simplest safe approach is to keep SSH **LAN-only by default** and only forward the port on the router **temporarily** when remote access is actually needed. This avoids permanent internet exposure and eliminates the need for dedicated VPN software, additional services on the Pi, or any third-party dependency.

### Default state

- SSH listens on port 22 with whatever auth you configured (password or key).
- No SSH port-forward on the router → SSH is invisible from the internet.
- Locally, connect as usual: `ssh <your-username>@rpi-3.local` (or via the Pi's LAN IP).

### Temporary remote access

When you need to SSH from outside the home network:

1. **Open a port-forward on the router:** external port → `<pi-ip>:22`
   - Consider using a non-default external port (e.g. `2222`) to reduce scanner noise.
2. **Connect from outside:**
   ```bash
   ssh -p <external-port> <your-username>@<your-public-ip>
   ```
3. **Remove the port-forward** as soon as you're done.

### Finding your public IP

From the Pi or any device on the home network:

```bash
curl -s https://ifconfig.me
```

### Good practices while the port is open

Even for short windows of exposure, keep these in mind:

- Use a **strong password** (long, not dictionary-based) or SSH keys.
- Monitor `sudo journalctl -u ssh --since "1 hour ago"` to see login attempts.
- Don't leave the port-forward open overnight or unattended for long periods.

### If remote access becomes frequent

If you find yourself opening/closing the port-forward often, it's worth revisiting a more permanent solution (VPN, SSH hardening with keys + `fail2ban`, or a reverse tunnel service). For occasional use on a test server, the on-demand approach is enough.

---

## 7. Network Configuration

For a server, you want a **predictable address on the home LAN** so you can always SSH to the same IP, set up port-forwards, reverse proxies, etc. The approach below assigns a pure static IP to the Ethernet interface.

If the Pi is later moved to a different network, the static config must be updated for that subnet, or temporarily switched back to DHCP (see 7.7).

### 7.1 Identify the active network stack

Different OS versions use different network managers. Check which one is active:

```bash
systemctl is-active NetworkManager
systemctl is-active dhcpcd
systemctl is-active systemd-networkd
```

Only one should report `active`. Follow the matching subsection below:

- **NetworkManager** → section 7.2 (default on current Raspberry Pi OS)
- **dhcpcd** → not covered yet; config lives in `/etc/dhcpcd.conf`
- **systemd-networkd** → not covered yet; config lives in `/etc/netplan/*.yaml`

### 7.2 NetworkManager

#### 7.2.1 List connections

```bash
nmcli connection show
```

Identify the Ethernet connection (typically named `Wired connection 1`, device `eth0`). Note its **NAME** — you'll pass it to `nmcli` in the commands below.

#### 7.2.2 (Optional) Rename the connection

A shorter, meaningful name is easier to work with:

```bash
sudo nmcli connection modify 'Wired connection 1' connection.id eth0-home
```

All subsequent commands use `eth0-home`; substitute your actual connection name if you didn't rename.

#### 7.2.3 Configure a static IP, gateway, and DNS

```bash
sudo nmcli connection modify eth0-home ipv4.method manual ipv4.addresses <static-ip>/24 ipv4.gateway <gateway-ip> ipv4.dns "1.1.1.1 8.8.8.8"
```

Replace:

- `<static-ip>` — the fixed LAN IP (e.g. `192.168.40.201`). Must be within the router's subnet and outside the DHCP pool to avoid conflicts.
- `<gateway-ip>` — the router's LAN IP (e.g. `192.168.40.1`).

What each setting does:

| Setting | Effect |
|---|---|
| `ipv4.method manual` | Disables DHCP on this connection and uses the values below. |
| `ipv4.addresses` | The fixed IP and subnet mask. `/24` covers `192.168.X.0`–`192.168.X.255`. |
| `ipv4.gateway` | Default route — the router that forwards traffic off the LAN. |
| `ipv4.dns` | DNS servers in priority order. Cloudflare primary, Google as fallback — mixing providers gives better resilience. |

#### 7.2.4 Apply changes

**With physical access to the Pi** (keyboard/monitor), bounce the interface — fast and easy to iterate if something's off:

```bash
sudo nmcli connection down eth0-home
sudo nmcli connection up eth0-home
```

**Over SSH**, reboot instead:

```bash
sudo reboot
```

Both drop any active SSH session and both leave you locked out if the config is wrong, so the risk is identical. Rebooting is preferred over SSH because it also verifies the config persists across reboots — better to catch a bad config now than on the next power cycle.

Reconnect at the new static IP after ~10 seconds (bounce) or ~30–60 seconds (reboot).

#### 7.2.5 Verify

```bash
ip addr show eth0      # should show the static IP
ip route               # default route should point at the gateway
resolvectl status eth0 # DNS servers: 1.1.1.1, 8.8.8.8
ping -c 2 <gateway-ip>
ping -c 2 1.1.1.1
ping -c 2 google.com   # confirms DNS resolution works
```

#### 7.2.6 Moving the Pi to another network

If the Pi is relocated to a different LAN, the static config won't match and networking will fail. Two options:

**Option A — Temporarily switch back to DHCP:**

```bash
sudo nmcli connection modify eth0-home ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
sudo nmcli connection up eth0-home
```

**Option B — Update the static values to the new subnet:**

```bash
sudo nmcli connection modify eth0-home ipv4.addresses <new-static-ip>/24 ipv4.gateway <new-gateway-ip>
sudo nmcli connection up eth0-home
```

### 7.3 Reserve the IP on the router (recommended)

Applies regardless of which network stack is in use. Even with a static IP on the Pi, add a **DHCP reservation / binding** on the router mapping the Pi's MAC to the chosen IP. This prevents the router from handing the same IP out to another device and keeps the LAN clean.

In the router admin UI, find **DHCP Binding** / **Address Reservation** (exact label varies by vendor) and add the mapping. Get the Pi's MAC with:

```bash
cat /sys/class/net/eth0/address
```

---
