# RustDesk Self-Hosted on Cosmos Cloud (or Docker/Portainer)

This repository provides a **complete**, step-by-step guide to run a self-hosted **RustDesk** server stack on **Debian 13**, using either:

- **Cosmos Cloud** (Docker is installed automatically), **or**
- **Docker Engine (Community Edition)** + optional **Portainer**

The stack includes:

- `hbbs` (ID server)
- `hbbr` (relay server)
- `rustdesk-api` (community API + web admin)

## 1) Required Ports

Open the following ports on your server firewall and (if applicable) on your router/NAT:

| Port | Protocol | Purpose |
|---:|:---:|---|
| 21114 | TCP | RustDesk API + Web Admin (`rustdesk-api`) |
| 21115 | TCP | NAT type detection |
| 21116 | TCP+UDP | ID server (`hbbs`) |
| 21117 | TCP+UDP | Relay (`hbbr`) |
| 21118 | TCP | Optional (if used) |
| 21119 | TCP | Optional (if used) |
| 80/443 | TCP | Only if you use an external reverse proxy / TLS |

## 2) Prerequisites

- Debian 13 (Bookworm)
- root/sudo access
- public IP or port-forwarding
- DNS records (recommended):
  - `rustdesk.<your-domain>` → server IP
  - `api-rustdesk.<your-domain>` → server IP

## 3) Choose ONE Installation Path

> Do **not** install both options in parallel on the same machine.

### Option A: Cosmos Cloud (Docker included)

Cosmos Cloud installs Docker automatically.

Practical flow:

1. Update Debian:

```bash
sudo apt-get update
sudo apt-get -y upgrade
```

2. (Recommended) reboot:

```bash
sudo reboot
```

3. Install Cosmos by following the official installer steps in the Cosmos documentation (copy/paste from the docs so it stays current).

4. Finish Cosmos setup (domain, certificates/Let's Encrypt, reverse proxy).

5. Deploy the compose stack (Section 6) as **Custom App (Compose)**.

### Option B: Docker Engine (Community Edition) + optional Portainer (Debian)

Install Docker CE + Compose plugin:

```bash
# 1) Prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 2) Add Docker’s official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3) Set up the repository
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian   $(. /etc/os-release && echo $VERSION_CODENAME) stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

# 4) Install Docker Engine + plugins
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5) (Optional) Allow running docker without sudo
sudo usermod -aG docker $USER
```

(Optional) Install Portainer:

```bash
docker volume create portainer_data
docker run -d   --name portainer   --restart=always   -p 9000:9000   -p 9443:9443   -v /var/run/docker.sock:/var/run/docker.sock   -v portainer_data:/data   portainer/portainer-ce:latest
```

Portainer UI: `https://<server-ip>:9443`

## 4) Generate Your RustDesk Public Key

**hbbs and hbbr MUST use the same public key.**

Base64 format example:

```text
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=
```

### Recommended: Let `hbbs` generate keys automatically

```bash
docker volume create rustdesk-hbbs-data

docker run --rm -v rustdesk-hbbs-data:/root rustdesk/rustdesk-server:latest hbbs

sudo cat /var/lib/docker/volumes/rustdesk-hbbs-data/_data/id_ed25519.pub
```

### Alternative: OpenSSL

```bash
openssl rand -base64 32
```

## 5) Docker Compose Stack (Complete)

Edit placeholders (domain/password/key) before deploying.

```yaml
services:
  rustdesk-hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: rustdesk-hbbs
    network_mode: host
    restart: unless-stopped
    volumes:
      - rustdesk-hbbs-data:/root
    command: hbbs -r YOUR_DOMAIN:21117 -k YOUR_PUBLIC_KEY

  rustdesk-hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: rustdesk-hbbr
    network_mode: host
    restart: unless-stopped
    volumes:
      - rustdesk-hbbr-data:/root
    command: hbbr -k YOUR_PUBLIC_KEY

  rustdesk-api:
    image: lejianwen/rustdesk-api:latest
    container_name: rustdesk-api
    network_mode: host
    restart: unless-stopped
    volumes:
      - rustdesk-api-data:/app/data
    environment:
      - TZ=Europe/Vienna
      - RUSTDESK_API_RUSTDESK_KEY=YOUR_PUBLIC_KEY
      - RUSTDESK_API_ADMIN_USERNAME=admin
      - RUSTDESK_API_ADMIN_PASSWORD=YOUR_ADMIN_PASSWORD
      - RUSTDESK_API_APP_NAME=RustDesk API
      - RUSTDESK_API_APP_URL=https://api-rustdesk.YOUR_DOMAIN
      - RUSTDESK_API_ALLOWED_ORIGINS=https://api-rustdesk.YOUR_DOMAIN
      - RUSTDESK_API_PORT=21114

volumes:
  rustdesk-hbbs-data:
  rustdesk-hbbr-data:
  rustdesk-api-data:
```

## 6) Reverse Proxy / TLS (for the Web Admin)

`rustdesk-api` runs on `http://127.0.0.1:21114` (because `network_mode: host`).

You want:

- `https://api-rustdesk.YOUR_DOMAIN` → `http://127.0.0.1:21114`

### Cosmos Cloud

- Add domain `api-rustdesk.YOUR_DOMAIN` to the app URL/domain setup
- Forward/target to `http://127.0.0.1:21114`
- Enable TLS / Let's Encrypt

### Nginx Proxy Manager (alternative)

- Proxy Host: `api-rustdesk.YOUR_DOMAIN`
- Forward IP: `SERVER-IP`
- Forward Port: `21114`
- SSL: Let's Encrypt

## 7) Deploy

### Cosmos Cloud

- Create App → Custom App (Compose)
- Paste the compose stack
- Start the app
- Configure the domain + TLS

### Docker/Portainer

Save `docker-compose.yml` and run:

```bash
docker compose up -d
```

Check logs:

```bash
docker compose logs -f --tail=200
```

## 8) Configure RustDesk Clients

In RustDesk client → Settings → Network:

- ID Server: `rustdesk.YOUR_DOMAIN`
- Relay Server: `rustdesk.YOUR_DOMAIN`
- Key: `YOUR_PUBLIC_KEY`

## 9) Security / Operations

- Keep Debian + containers up to date
- Use strong admin passwords
- Open only required ports
- Back up Docker volumes regularly

## License

Add your preferred license (MIT/Apache-2.0/etc.).
