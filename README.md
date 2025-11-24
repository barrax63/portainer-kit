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

| Container       | Image                           | Hostname   | Port(s)          | Access                          |
|-----------------|----------------------------------|------------|------------------|----------------------------------|
| `portainer`     | `portainer/portainer-ce:alpine` | portainer  | 9443 (HTTPS)     | From host/LAN                    |
| `cloudflared`   | `cloudflare/cloudflared:latest` | cloudflared| -/-              | Outbound tunnel only (optional)  |

- Network: user-defined bridge with IPAM (10.254.5.0/24)
- Storage: named volume `portainer_storage` for Portainer data
- Health checks:
  - Portainer: HTTPS probe on 127.0.0.1:9443 using wget
  - Cloudflared: “ready” via metrics endpoint

### Expose Portainer through Cloudflare Tunnel (optional)

1. Create a Cloudflare Tunnel and HTTP route pointing to `http://portainer:9443` or `https://portainer:9443` (Cloudflare can terminate TLS; both work).
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

#### Standard configuration (required)

- `TZ` – Timezone (e.g., `Europe/Berlin`)
- `CLOUDFLARED_TUNNEL_TOKEN` – Only if using the Cloudflare profile

#### Advanced configuration (optional)

- Adjust network subnet in compose if needed (`10.254.5.0/24`)
- Customize resource limits if you deploy to Swarm (note: deploy.resources are ignored by non‑Swarm docker compose)

### Security features

This setup includes hardening measures:

- No new privileges (`no-new-privileges:true`)
- AppArmor profile (`apparmor=docker-default`)
- Capabilities: drop all, add only required (CHOWN, FOWNER, SETGID, SETUID)
- Read-only filesystem with tmpfs for `/tmp` and `/run`
- Log rotation via json-file options
- Network isolation on a user-defined bridge

## Tips & tricks

### Healthcheck verification

- The Portainer image (alpine) includes BusyBox `wget`.
- Healthcheck uses array form (no shell needed):

```yaml
healthcheck:
  test: ["CMD", "wget", "--no-check-certificate", "-qO-", "https://127.0.0.1:9443/"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 30s
```

Check health:

```bash
docker compose ps
docker inspect -f '{{json .State.Health}}' portainer-portainer-1 | jq
```

### Logs and monitoring

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f portainer
docker compose logs -f cloudflared
```

### Resource management

The compose file includes Swarm-style `deploy.resources` which are ignored by docker compose (non‑Swarm). If you need hard limits without Swarm, consider legacy keys supported by compose on your platform:

```yaml
# Example (non-standardized across all engines):
# cpus: "1.0"
# mem_limit: 1g
```

Or switch to Docker Swarm to enforce `deploy.resources`.

### Troubleshooting

#### “Podman environment option doesn’t support Docker environments”

- Ensure the local endpoint is set to Docker, not Podman:
  - Portainer UI → Environments → Edit “local” → Type: Docker (socket: `unix:///var/run/docker.sock`)

#### Healthcheck failing

- Ensure you’re using HTTPS on 9443 in the healthcheck
- Increase `start_period` to 60–90s on first boot if needed
- Confirm `wget` exists (it does in `:alpine`); if you switch images, adapt the command

#### Cloudflare tunnel not working

```bash
docker compose logs cloudflared
docker compose config | grep CLOUDFLARED_TUNNEL_TOKEN
```

Verify that your Cloudflare route points to the service correctly (e.g., http://portainer:9443) and that the token is valid.

## Project files

- `docker-compose.yml` – Main stack definition
- `.env.example` – Template for environment variables

Example docker-compose.yml (excerpt):

```yaml
services:
  portainer:
    image: portainer/portainer-ce:alpine
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_storage:/data
    read_only: true
    tmpfs: ["/tmp", "/run"]
    healthcheck:
      test: ["CMD", "wget", "--no-check-certificate", "-qO-", "https://127.0.0.1:9443/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  cloudflared:
    image: cloudflare/cloudflared:latest
    profiles: ["cloudflared"]
    env_file:
      - path: .env
        required: true
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARED_TUNNEL_TOKEN}
    command: tunnel --metrics 127.0.0.1:60123 --no-autoupdate run
    healthcheck:
      test: ["CMD", "cloudflared", "tunnel", "--metrics", "127.0.0.1:60123", "ready"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 20s
```

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