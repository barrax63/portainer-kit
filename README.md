### Portainer Kit

**Portainer Kit** is a secure, minimal Docker Compose setup to run Portainer CE with optional Cloudflare Tunnel for remote access. It includes opinionated hardening (reduced capabilities, read-only filesystem, log rotation), a custom bridge network, and a working HTTPS healthcheck.

### What's included

✅ [Portainer CE](https://github.com/portainer/portainer) – Web UI to manage Docker environments, stacks, images, volumes, and networks

✅ [Cloudflared (optional)](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) – Secure reverse tunnel to access your Portainer instance remotely without opening inbound firewall ports

✅ Security hardening – no-new-privileges, AppArmor profile, capability drop, read-only filesystem, tmpfs for runtime dirs, and log rotation

### What you can do

⭐️ Manage Docker containers, images, networks, and volumes via a friendly UI

⭐️ Deploy and manage Docker Compose stacks from the browser

⭐️ Observe container logs, console, and resource usage

⭐️ Securely access Portainer remotely using Cloudflare Tunnel

⭐️ Apply good security practices out-of-the-box for production-like setups

## Installation

### Prerequisites

- Docker and Docker Compose installed
- A Linux host with AppArmor recommended (or adjust security options accordingly)
- (Optional) Cloudflare account and tunnel token to enable remote access
- Set your timezone (TZ), e.g. Europe/Berlin

### Running Portainer using Docker Compose

#### Standard Setup

```bash
git clone https://github.com/barrax63/portainer-kit.git
cd portainer-kit
cp .env.example .env  # Update the STANDARD CONFIGURATION variables inside
docker compose up -d
```

> [!IMPORTANT]
> Update the following variables in your `.env` file:
> - `TZ` – Your timezone (e.g., `Europe/Berlin`, `America/New_York`)
> - `CLOUDFLARED_TUNNEL_TOKEN` – Only required if you enable the Cloudflare profile

## ⚡️ Quick start and usage

After starting the stack, access Portainer:

- Local: https://localhost:9443
- From another device on your LAN: https://<HOST-IP>:9443
- Through Cloudflare (optional): your Cloudflare route/domain once the tunnel is running

### First-time setup

1. Create the Portainer admin account when prompted.
2. Add the local environment:
   - Type: Docker
   - Socket: unix:///var/run/docker.sock
3. (Optional) Add additional remote Docker/Agent endpoints.
4. Review Settings > Environments to ensure the local endpoint is Docker (not Podman).

## Containers and access

This kit runs minimal services with restricted access.

| Container       | Image                           | Hostname   | Port(s)          | Access                           |
|-----------------|---------------------------------|------------|------------------|----------------------------------|
| `portainer`     | `portainer/portainer-ce:alpine` | portainer  | 9443 (HTTPS)     | From host/LAN                    |
| `cloudflared`   | `cloudflare/cloudflared:latest` | cloudflared| -/-              | Outbound tunnel only (optional)  |

- Network: user-defined bridge with IPAM (10.254.5.0/24)
- Storage: named volume `portainer_storage` for Portainer data
- Health checks:
  - Portainer: HTTPS probe on 127.0.0.1:9443 using wget
  - Cloudflared: “ready” via metrics endpoint

### Expose Portainer through Cloudflare Tunnel (optional)

1. Create a Cloudflare Tunnel and HTTP route pointing to `http://portainer:9000`.
2. Copy the generated token into `.env` as `CLOUDFLARED_TUNNEL_TOKEN`.
3. Start Cloudflared:

```bash
docker compose --profile cloudflared up -d
```

Cloudflared connects using the token—no extra config files needed.

## Upgrading

### Update to latest versions

```bash
git pull
docker compose pull
docker compose up -d
```

This will:
- Pull the latest repo changes
- Download the newest images
- Recreate containers while preserving volumes

### Enable auto-updates

Create a cron job:

```bash
crontab -u <USER> -e
```

Add:

```bash
# Every Sunday at 00:00 update repo and images, then recreate containers
0 0 * * 0 cd /home/<USER>/portainer-kit && /usr/bin/git pull && /usr/bin/docker compose pull && /usr/bin/docker compose up -d >> /home/<USER>/portainer-kit/cron.log 2>&1
```

## 👓 Recommended reading

- Portainer Docs: https://docs.portainer.io/
- Business vs CE comparison: https://www.portainer.io/products
- Cloudflared Docs: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- Docker Healthcheck reference: https://docs.docker.com/engine/reference/builder/#healthcheck

## Configuration Guide

### Environment variables

Your `.env.example` should include:

- `TZ` – Timezone (e.g., `Europe/Berlin`)
- `CLOUDFLARED_TUNNEL_TOKEN` – Only if using the Cloudflare profile

## 📜 License

This project is licensed under the Apache License 2.0 – see the LICENSE file.

## 💬 Support

- Portainer Community: https://github.com/portainer/portainer/discussions
- Portainer Issues: https://github.com/portainer/portainer/issues
- Cloudflare Tunnel Docs: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/

## 🤝 Contributing

Contributions are welcome! Feel free to open issues and PRs.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/ImproveDocs`)
3. Commit your changes (`git commit -m 'Improve README for Portainer Kit'`)
4. Push the branch (`git push origin feature/ImproveDocs`)
5. Open a Pull Request

## Acknowledgments

- [Portainer](https://github.com/portainer/portainer) – Excellent UI for Docker management
- [Cloudflare](https://www.cloudflare.com/) – Simple, secure remote access with tunnels
- Community best practices for container hardening and logging rotation