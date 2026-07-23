# Athena — OpenCTI Homelab

A self-hosted OpenCTI deployment for threat intelligence homelab / SOC lab purposes, running via Docker Compose behind an nginx reverse proxy, with a curated set of connectors pre-wired (MITRE ATT&CK, AlienVault OTX, ThreatFox, Ransomware.live, MalwareBazaar, AbuseIPDB) and XTM One integration.

> ⚠️ **This is a homelab / lab setup, not a production-hardened deployment.** Self-signed TLS certificates, single-host Elasticsearch, and no external backups are configured by default. See [Security notes](#security-notes) before exposing this beyond your local network.

## Prerequisites

- A Linux host (Ubuntu/Debian tested) with at least 16 GB RAM (8 GB minimum, see `ELASTIC_MEMORY_SIZE` below)
- Docker and Docker Compose
- OpenSSL (for generating secrets and certificates)
- Git

## Installation

### 1. Install Docker and dependencies

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common git docker.io docker-compose -y
sudo systemctl enable --now docker
sudo usermod -aG docker $USER && newgrp docker
```

### 2. Clone the repository

```bash
git clone https://github.com/broxcat/athena.git
cd athena
```

### 3. Configure your environment

Copy the sample file and fill in your own values:

```bash
cp .env.sample .env
nano .env
```

You will need to generate several values yourself:

- **UUIDs** (for `OPENCTI_ADMIN_TOKEN`, every `CONNECTOR_*_ID`, `XTM_COMPOSER_ID`, etc.) — generate each one individually, never reuse the same UUID across variables:
```bash
  python3 -c "import uuid; print(uuid.uuid4())"
```
  or use [uuidgenerator.net/version4](https://www.uuidgenerator.net/version4) if you prefer a web tool.

- **Random secrets** (`OPENCTI_ENCRYPTION_KEY`, `PLATFORM_REGISTRATION_TOKEN`, `XTM_ONE_SECRET_KEY`, etc.):
```bash
  openssl rand -hex 32       # for hex-format secrets
  openssl rand -base64 32    # for OPENCTI_ENCRYPTION_KEY specifically
```

- **API keys** for the connectors that need them:
  - `ALIENVAULT_API_KEY` — get one at [otx.alienvault.com](https://otx.alienvault.com)
  - `ABUSEIPDB_API_KEY` — get one at [abuseipdb.com](https://www.abuseipdb.com)

- **Passwords** (`OPENCTI_ADMIN_PASSWORD`, `MINIO_ROOT_PASSWORD`, `RABBITMQ_DEFAULT_PASS`, `XTM_ONE_ADMIN_PASSWORD`, `XTM_ONE_POSTGRES_PASSWORD`, etc.) — pick strong, unique values. Do not leave any at `changeme`.

> 🔒 `.env` is git-ignored and must **never** be committed. See [Security notes](#security-notes).

### 4. Generate a self-signed TLS certificate (for nginx)

```bash
mkdir -p certs
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout certs/opencti.key \
  -out certs/opencti.crt \
  -days 365 \
  -subj "/CN=localhost"
```

### 5. Apply the required kernel setting for Elasticsearch

```bash
sudo sysctl -w vm.max_map_count=1048575
echo "vm.max_map_count=1048575" | sudo tee -a /etc/sysctl.conf
```

This avoids the classic `vm.max_map_count` error that prevents Elasticsearch from starting.

### 6. Start the stack

```bash
docker compose up -d
```

Follow the logs until the platform reports the GraphQL server is ready and the worker has connected to RabbitMQ:

```bash
docker compose logs -f opencti
```

First boot also seeds the initial datasets (base taxonomy, MITRE ATT&CK import), which can take a few minutes.

### 7. Access the platform

Browse to `https://localhost` (or your configured `OPENCTI_URL`) and log in with `OPENCTI_ADMIN_EMAIL` / `OPENCTI_ADMIN_PASSWORD`. Your browser will warn about the self-signed certificate — that's expected for a local/homelab setup.

## Connectors included

| Connector | Type | Requires |
|---|---|---|
| MITRE ATT&CK | External import | — |
| AlienVault OTX | External import | `ALIENVAULT_API_KEY` |
| ThreatFox | External import | — |
| Ransomware.live | External import | — |
| MalwareBazaar (recent additions) | External import | — |
| AbuseIPDB | External import | `ABUSEIPDB_API_KEY` |

Connector status and manual runs can be monitored under **Data > Ingestion > Connectors** in the OpenCTI UI.

## Security notes

- **`.env` and `certs/`** are excluded via `.gitignore` and must never be pushed to this or any fork of this repo. If you ever commit them by mistake, treat every secret inside as compromised and rotate all of them (API keys, tokens, passwords, encryption key) — do not rely solely on removing the file from a later commit.
- **`docker.sock` mount** (used by `xtm-composer`) grants that container root-equivalent access to the Docker host. Do not expose this service or its network to untrusted parties.
- **Self-signed TLS** is fine for a local homelab. If you expose this instance beyond `localhost`, replace it with a certificate from a real CA (e.g. Let's Encrypt) and put it behind proper network access controls.
- **Ports**: only nginx (`80`/`443`) is intended to be reachable externally. OpenCTI's direct port (`8080`) and MinIO's port (`9000`) should not be exposed beyond the Docker network unless you're explicitly debugging locally.

## Maintenance

- Update images by pulling new tags and re-running `docker compose up -d` (check the image tags in `docker-compose.yml` — pin to specific versions rather than `latest` where possible).
- Back up the Elasticsearch data and the MinIO bucket regularly if this instance holds intel you care about keeping.

## Credits / inspiration

This setup was inspired by and adapted from:
- ["Build Your Own Cyber Threat Intelligence (CTI) Platform Like a Pro"](https://medium.com/@deepanshu_khanna/build-your-own-cyber-threat-intelligence-cti-platform-like-a-pro-515227cedc30) by Deepanshu Khanna
- ["What OpenCTI Is, and How to Actually Get It Running"](https://medium.com/@mrstarkeg/what-opencti-is-and-how-to-actually-get-it-running-653b0f623c41) by Ahmed Elshahat
