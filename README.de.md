# RustDesk Selfhosted auf Cosmos Cloud (oder Docker/Portainer)

Dieses Repository enthält eine **vollständige**, Schritt-für-Schritt-Anleitung, um einen selbst gehosteten **RustDesk** Server-Stack auf **Debian 13** zu betreiben – entweder mit:

- **Cosmos Cloud** (Docker wird automatisch mitinstalliert), **oder**
- **Docker Engine (Community Edition)** + optional **Portainer**

Der Stack enthält:

- `hbbs` (ID-Server)
- `hbbr` (Relay-Server)
- `rustdesk-api` (Community API + Web-Admin)

## 1) Benötigte Ports

Öffne folgende Ports auf der Server-Firewall und (falls nötig) am Router/NAT:

| Port | Protokoll | Verwendung |
|---:|:---:|---|
| 21114 | TCP | RustDesk API + Web-Admin (`rustdesk-api`) |
| 21115 | TCP | NAT-Typ-Erkennung |
| 21116 | TCP+UDP | ID-Server (`hbbs`) |
| 21117 | TCP+UDP | Relay (`hbbr`) |
| 21118 | TCP | Optional (falls genutzt) |
| 21119 | TCP | Optional (falls genutzt) |
| 80/443 | TCP | Nur wenn du externen Reverse Proxy / TLS nutzt |

## 2) Voraussetzungen

- Debian 13 (Bookworm)
- Root-/sudo-Zugriff
- öffentliche IP oder Port-Forwarding
- DNS Records (empfohlen):
  - `rustdesk.<deine-domain>` → Server-IP
  - `api-rustdesk.<deine-domain>` → Server-IP

## 3) Wähle GENAU EINE Installationsvariante

> Installiere **nicht** beide Optionen parallel auf derselben Maschine.

### Option A: Cosmos Cloud (inkl. Docker)

Cosmos Cloud installiert Docker automatisch.

Praxistauglicher Ablauf:

1. Debian updaten:

```bash
sudo apt-get update
sudo apt-get -y upgrade
```

2. (Empfohlen) Neustart:

```bash
sudo reboot
```

3. Cosmos installieren, indem du die Installer-Kommandos aus der offiziellen Cosmos-Doku 1:1 kopierst/ausführst (damit es immer aktuell bleibt).

4. Cosmos Setup abschließen (Domain, Zertifikate/Let's Encrypt, Reverse Proxy).

5. Den Compose-Stack (Abschnitt 6) als **Custom App (Compose)** deployen.

### Option B: Docker Engine (Community Edition) + optional Portainer (Debian)

Docker CE + Compose Plugin installieren:

```bash
# 1) Voraussetzungen
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 2) Docker GPG Key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3) Docker Repo hinzufügen
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian   $(. /etc/os-release && echo $VERSION_CODENAME) stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

# 4) Docker installieren
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5) (Optional) docker ohne sudo
sudo usermod -aG docker $USER
```

(Optional) Portainer installieren:

```bash
docker volume create portainer_data
docker run -d   --name portainer   --restart=always   -p 9000:9000   -p 9443:9443   -v /var/run/docker.sock:/var/run/docker.sock   -v portainer_data:/data   portainer/portainer-ce:latest
```

Portainer UI: `https://<server-ip>:9443`

## 4) RustDesk Public Key generieren

**hbbs und hbbr MÜSSEN denselben Public Key verwenden.**

Base64 Format-Beispiel:

```text
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=
```

### Empfohlen: hbbs generiert Keys automatisch

```bash
docker volume create rustdesk-hbbs-data

docker run --rm -v rustdesk-hbbs-data:/root rustdesk/rustdesk-server:latest hbbs

sudo cat /var/lib/docker/volumes/rustdesk-hbbs-data/_data/id_ed25519.pub
```

### Alternative: OpenSSL

```bash
openssl rand -base64 32
```

## 5) Docker Compose Stack (vollständig)

Platzhalter (Domain/Passwort/Key) anpassen, dann deployen.

```yaml
services:
  rustdesk-hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: rustdesk-hbbs
    network_mode: host
    restart: unless-stopped
    volumes:
      - rustdesk-hbbs-data:/root
    command: hbbs -r DEINE_DOMAIN:21117 -k DEIN_PUBLIC_KEY

  rustdesk-hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: rustdesk-hbbr
    network_mode: host
    restart: unless-stopped
    volumes:
      - rustdesk-hbbr-data:/root
    command: hbbr -k DEIN_PUBLIC_KEY

  rustdesk-api:
    image: lejianwen/rustdesk-api:latest
    container_name: rustdesk-api
    network_mode: host
    restart: unless-stopped
    volumes:
      - rustdesk-api-data:/app/data
    environment:
      - TZ=Europe/Vienna
      - RUSTDESK_API_RUSTDESK_KEY=DEIN_PUBLIC_KEY
      - RUSTDESK_API_ADMIN_USERNAME=admin
      - RUSTDESK_API_ADMIN_PASSWORD=DEIN_ADMIN_PASSWORT
      - RUSTDESK_API_APP_NAME=RustDesk API
      - RUSTDESK_API_APP_URL=https://api-rustdesk.DEINE_DOMAIN
      - RUSTDESK_API_ALLOWED_ORIGINS=https://api-rustdesk.DEINE_DOMAIN
      - RUSTDESK_API_PORT=21114

volumes:
  rustdesk-hbbs-data:
  rustdesk-hbbr-data:
  rustdesk-api-data:
```

## 6) Reverse Proxy / TLS (für das Web-Admin)

`rustdesk-api` läuft auf `http://127.0.0.1:21114` (wegen `network_mode: host`).

Ziel:

- `https://api-rustdesk.DEINE_DOMAIN` → `http://127.0.0.1:21114`

### Cosmos Cloud

- Domain `api-rustdesk.DEINE_DOMAIN` im App URL/Domain Setup hinzufügen
- Forward/Target: `http://127.0.0.1:21114`
- TLS / Let's Encrypt aktivieren

### Nginx Proxy Manager (Alternative)

- Proxy Host: `api-rustdesk.DEINE_DOMAIN`
- Forward IP: `SERVER-IP`
- Forward Port: `21114`
- SSL: Let's Encrypt

## 7) Deployment

### Cosmos Cloud

- Create App → Custom App (Compose)
- Compose einfügen
- App starten
- Domain + TLS konfigurieren

### Docker/Portainer

`docker-compose.yml` speichern und starten:

```bash
docker compose up -d
```

Logs prüfen:

```bash
docker compose logs -f --tail=200
```

## 8) RustDesk Clients konfigurieren

Im RustDesk Client → Settings → Network:

- ID Server: `rustdesk.DEINE_DOMAIN`
- Relay Server: `rustdesk.DEINE_DOMAIN`
- Key: `DEIN_PUBLIC_KEY`

## 9) Security / Betrieb

- Debian + Container regelmäßig updaten
- Starkes Admin-Passwort
- Nur benötigte Ports öffnen
- Docker Volumes regelmäßig sichern

## Lizenz

Füge eine Lizenz deiner Wahl hinzu (MIT/Apache-2.0/etc.).
