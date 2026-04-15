# Raspberry Pi 3 Dcoker Setup

## Prerequisites

- Raspberry Pi 3 (Model B or B+)
- MicroSD card (16 GB minimum, Class 10 recommended)
- Power supply (5V/2.5A micro-USB)
- Monitor, keyboard, and mouse (for initial setup), or SSH access
- A computer to flash the SD card

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
