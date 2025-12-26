# Homelab Ansible

Minimal Ansible setup for managing a homelab with:

- Traefik (DNS-01, Cloudflare, HTTPS everywhere)
- Docker
- Firewall (UFW)
- Pi-hole (managed, NOT installed)
- Portainer (Docker management UI)
- Nextcloud (self-hosted cloud storage)
- Homepage (dashboard)

This repo assumes you already have servers running and want repeatable configuration — not installers.

---

## Architecture

Host: pimaster  
Purpose: Docker host + Traefik reverse proxy + services  
IP: 192.168.1.68  
Services: Traefik, Portainer, Nextcloud (optional), Homepage  
Firewall: UFW (ports 22, 80, 443)

Host: pihole  
Purpose: Pi-hole DNS (standalone)  
IP: 192.168.1.86  
Firewall: UFW (ports 22, 53, 80)

Pi-hole runs on a separate server and is installed manually.  
Ansible only manages configuration and service state.

---

## Requirements

- SSH key access to all hosts
- Ansible installed locally
- Ansible collections (install with: `ansible-galaxy collection install -r requirements.yml`)
- Docker already installed on pimaster
- Pi-hole already installed on pihole
- Cloudflare account + domain (travismedia.cloud)

---

## Setup

1. **Install Ansible collections:**
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. **Configure your domain and hosts:**
   - Edit `vars/homelab.yml` with your domain and IP addresses
   - Edit `inventory/homelab.yml` with your host details

3. **Set up secrets (see Secrets section below)**

---

## Secrets Setup

Secrets are stored in `vars/vault.yml` using Ansible Vault.

### 1. Get Cloudflare API Token

Traefik needs a Cloudflare API token for DNS-01 challenge:

1. Log into Cloudflare → My Profile → API Tokens
2. Create Token using "Edit zone DNS" template
3. Scope to your domain
4. Copy the token

### 2. Generate Traefik Password

```bash
docker run --rm httpd:alpine htpasswd -nbB admin YourPassword
```

### 3. Create Vault File

```bash
ansible-vault create vars/vault.yml --ask-vault-pass
```

See `vars/vault.yml.example` for the required format. Add both secrets:
```yaml
---
cloudflare_dns_api_token: "your_token_here"
traefik_dashboard_auth: "admin:$2y$05$..."  # From step 2
```

### 4. Edit or View Existing Vault

To edit the vault file after creation:

```bash
ansible-vault edit vars/vault.yml --ask-vault-pass
```

To view the vault contents without editing:

```bash
ansible-vault view vars/vault.yml --ask-vault-pass
```

---

## Deploy

Dry run (recommended):

```bash
ansible-playbook playbooks/site.yml --check --diff --ask-vault-pass
```

Apply changes:

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

You will be prompted for your vault password before execution.

### Service Toggles

Services can be enabled/disabled in `vars/homelab.yml`:

- `deploy_traefik`: true
- `deploy_portainer`: true
- `deploy_nextcloud`: false (disabled by default)
- `deploy_homepage`: true
- `deploy_pihole`: true
- `deploy_firewall`: true

### Docker Image Versions

Image versions are pinned in `vars/homelab.yml` for reproducibility:

- Traefik: `traefik:v3.6`
- Portainer: `portainer/portainer-ce:2.21.4`
- Homepage: `ghcr.io/gethomepage/homepage:v1.8.0`
- Nextcloud: `nextcloud/all-in-one:latest` (AIO recommends latest)

Update these versions in the vars file as needed.

---

## Access

All services are accessible via HTTPS with wildcard TLS certificates:

- Traefik dashboard: https://traefik.travismedia.cloud
- Pi-hole (proxied through Traefik): https://pihole.travismedia.cloud
- Portainer: https://portainer.travismedia.cloud
- Homepage (dashboard): https://homepage.travismedia.cloud
- Nextcloud (when enabled): https://nextcloud.travismedia.cloud

TLS certificates are issued and automatically renewed by Traefik using DNS-01.

Traefik dashboard requires authentication (configured in vault).

### Portainer Initial Setup

After deploying Portainer for the first time:

1. Access https://portainer.travismedia.cloud and create your admin account
2. When prompted about connecting to Docker, **dismiss/skip the initial connection error**
3. Manually add the Docker environment:
   - Go to **Environments** → **Add environment**
   - Select **Docker Standalone**
   - Choose **API** as the connection method
   - Enter the Environment URL: `docker-proxy:2375`
   - Leave TLS **disabled** (internal network connection)
   - Click **Connect**

Portainer normally uses `/var/run/docker.sock` directly, but this setup uses docker-socket-proxy for secure, restricted access to the Docker API.

---

## Notes

- This repo does NOT install Pi-hole
- DNS resolves locally via Pi-hole to Traefik
- Firewall rules are managed via Ansible (UFW)
- Designed to be idempotent and safe to re-run

---

## Philosophy

- No click-ops
- No re-installing working systems
- No hidden magic
- Everything version controlled
