# Raspberry Pi Docker Setup

## Prerequisites

- Raspberry Pi 3 (Model B or B+)
- MicroSD card (16 GB minimum, Class 10 recommended)
- Power supply (5V/2.5A micro-USB)
- Monitor, keyboard, and mouse (for initial setup), or SSH access
- A computer to flash the SD card

---

## 1. Flash the OS to the MicroSD Card

1. Download **Raspberry Pi Imager** from [raspberrypi.com/software](https://www.raspberrypi.com/software/).
2. Insert the MicroSD card into your computer.
3. Open Raspberry Pi Imager and choose:
   - **OS**: Raspberry Pi OS Lite (64-bit) — recommended for headless/server use
   - **Storage**: your MicroSD card
4. Click the gear icon to configure advanced options before writing:
   - Enable SSH
   - Set hostname (e.g., `raspberrypi`)
   - Set username and password
   - Configure Wi-Fi (SSID and password) if needed
5. Click **Write** and wait for it to finish.

---

## 2. First Boot

1. Insert the MicroSD card into the Raspberry Pi.
2. Connect Ethernet cable (recommended for initial setup) or rely on the Wi-Fi configured above.
3. Power on the Raspberry Pi.
4. Wait ~60 seconds for the first boot to complete.

### Verify SSH is Running

Once logged in (or via a direct monitor/keyboard), check if the SSH service is active:

```bash
sudo systemctl status ssh
```

If the status shows `inactive` or `not found`, enable and start it:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

Alternatively, enable SSH through `raspi-config`:

```bash
sudo raspi-config
# Interface Options > SSH > Enable
```

### Connect via SSH

```bash
ssh <your-username>@raspberrypi.local
# or using the IP address:
ssh <your-username>@<raspberry-pi-ip>
```

> Tip: Find the Pi's IP address from your router's admin panel or use `nmap -sn 192.168.1.0/24`.

---

## 3. Initial System Configuration

### Run raspi-config

```bash
sudo raspi-config
```

Recommended settings:
- **System Options > Hostname** — set a meaningful hostname
- **System Options > Password** — change the default password if not done during imaging
- **Interface Options > SSH** — ensure SSH is enabled
- **Localisation Options** — set timezone, locale, and keyboard layout
- **Advanced Options > Expand Filesystem** — ensure the full SD card is used

Reboot when prompted.

---

## 4. Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

Install useful base packages:

```bash
sudo apt install -y curl wget git vim nano net-tools htop
```

---

## 5. Network Configuration (Optional but Recommended)

### 5.1 List Available Network Interfaces

See all network interfaces and their current IP addresses:

```bash
ip link show
```

Or with more detail including IP addresses:

```bash
ip addr show
```

Common interface names:
- `eth0` — wired Ethernet
- `wlan0` — Wi-Fi

### 5.2 Configure a Static IP

Edit the DHCP client config:

```bash
sudo nano /etc/dhcpcd.conf
```

Add at the end (adjust values for your network and interface):

```
interface eth0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=1.1.1.1 8.8.8.8
```

> **DNS servers** — you can use any DNS provider or your router's IP. Common public options:
> - `1.1.1.1` — Cloudflare DNS (fast, privacy-focused)
> - `8.8.8.8` — Google DNS (reliable, widely used)
> - `9.9.9.9` — Quad9 (security-focused, blocks malicious domains)
>
> You can combine them (e.g., `1.1.1.1 8.8.8.8`) so the second acts as a fallback if the first is unreachable.

Restart the networking service to apply the changes:

```bash
sudo systemctl restart dhcpcd
```

### 5.3 Verify the Static IP is Applied

Check the interface now has the IP you configured:

```bash
ip addr show eth0
```

Confirm internet connectivity:

```bash
ping -c 4 1.1.1.1
```

Confirm DNS resolution is working:

```bash
ping -c 4 google.com
```

---

## 6. Install Docker

### 6.1 Install using the Official Script

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### 6.2 Add Your User to the Docker Group

This allows running Docker commands without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Log out and back in for the group change to take effect:

```bash
logout
```

Reconnect via SSH, then verify:

```bash
docker run hello-world
```

Expected output should include: `Hello from Docker!`

---

## 7. Install Docker Compose

Docker Compose v2 is included as a Docker plugin. Verify it is available:

```bash
docker compose version
```

If not available, install it manually:

```bash
sudo apt install -y docker-compose-plugin
```

---

## 8. Enable Docker to Start on Boot

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Verify the service is running:

```bash
sudo systemctl status docker
```

---

## 9. Verify the Full Setup

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Run a test container
docker run hello-world

# List running containers
docker ps

# List all containers
docker ps -a
```

---

## Useful Docker Commands

| Command | Description |
|---|---|
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker images` | List local images |
| `docker pull <image>` | Pull an image |
| `docker run -d -p 80:80 <image>` | Run a container in detached mode |
| `docker stop <container>` | Stop a container |
| `docker rm <container>` | Remove a stopped container |
| `docker rmi <image>` | Remove an image |
| `docker logs <container>` | View container logs |
| `docker exec -it <container> bash` | Open a shell in a running container |
| `docker compose up -d` | Start services defined in docker-compose.yml |
| `docker compose down` | Stop and remove compose services |
| `docker system prune` | Remove unused data (containers, images, networks) |

---

## Troubleshooting

### Permission denied when running Docker

Ensure your user is in the `docker` group and you have logged out/in after adding it:

```bash
groups $USER
```

`docker` should appear in the list.

### Cannot connect via SSH

- Confirm SSH is enabled in `raspi-config`.
- Check the Pi's IP address.
- Ensure the Pi is on the same network.

### SD card not expanded

Run `sudo raspi-config` > **Advanced Options** > **Expand Filesystem**, then reboot.

### Docker service fails to start

Check logs for details:

```bash
sudo journalctl -u docker --no-pager | tail -30
```
