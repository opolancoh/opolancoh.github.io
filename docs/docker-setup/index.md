# Docker Setup

Guide for installing Docker and Docker Compose on a Debian-based Linux system. Works on Raspberry Pi (any model), Ubuntu, Debian, and derivatives.

## Contents

- [Install Docker](#install-docker)
- [Install Docker Compose](#install-docker-compose)
- [Enable Docker on Boot](#enable-docker-on-boot)
- [Verify the Setup](#verify-the-setup)
- [Useful Commands](#useful-commands)
- [Troubleshooting](#troubleshooting)

---

## Install Docker

### Using the official script

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh
```

### Add your user to the Docker group

This allows running Docker commands without `sudo`:

```bash
sudo usermod -aG docker $USER
```

Log out and back in for the group change to take effect:

```bash
logout
```

Reconnect, then verify:

```bash
docker run hello-world
```

Expected output should include: `Hello from Docker!`

---

## Install Docker Compose

Docker Compose v2 is included as a Docker plugin. Verify it is available:

```bash
docker compose version
```

If not available, install it manually:

```bash
sudo apt install -y docker-compose-plugin
```

---

## Enable Docker on Boot

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Verify the service is running:

```bash
sudo systemctl status docker
```

---

## Verify the Setup

```bash
docker --version
docker compose version
docker run hello-world
docker ps
docker ps -a
```

---

## Useful Commands

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

### Docker service fails to start

Check logs for details:

```bash
sudo journalctl -u docker --no-pager | tail -30
```
