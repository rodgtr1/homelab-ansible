# Homelab Ansible

Minimal Ansible setup for managing a homelab with:

- Traefik (DNS-01, Cloudflare, HTTPS everywhere)
- Docker
- Firewall (UFW)
- Pi-hole + Unbound (DNS with recursive resolution)
- Portainer (Docker management UI)
- Nextcloud (self-hosted cloud storage)
- Homepage (dashboard)
- Vaultwarden (password manager)

This repo manages your entire homelab infrastructure with repeatable, version-controlled configuration.

---

## Architecture

Host: pimaster  
Purpose: Docker host + Traefik reverse proxy + services  
IP: 192.168.1.68  
Services: Traefik, Portainer, Nextcloud (optional), Homepage, Vaultwarden  
Firewall: UFW (ports 22, 80, 443)

Host: pihole  
Purpose: Pi-hole + Unbound DNS (recursive resolver)  
IP: 192.168.1.92  
Firewall: UFW (ports 22, 53, 80)

Pi-hole and Unbound run in Docker containers with Pi-hole handling ad-blocking and Unbound providing recursive DNS resolution (no third-party DNS dependencies).

---

## Requirements

- SSH key access to all hosts
- Ansible installed locally
- Ansible collections (install with: `ansible-galaxy collection install -r requirements.yml`)
- Raspberry Pi(s) with Ubuntu Server installed (see `scripts/` for setup helpers)
- Cloudflare account + domain (yourdomain.com)

---

## Setup

1. **Prepare your Raspberry Pi hosts:**
   - Install Ubuntu Server on your Pi(s)
   - Run the appropriate setup script from `scripts/` directory (see `scripts/README.md`)
   - Note the IP addresses assigned to each host

2. **Install Ansible collections:**
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

3. **Configure your domain and hosts:**
   - Edit `vars/homelab.yml` with your domain and IP addresses
   - Edit `inventory/homelab.yml` with your host details

4. **Set up secrets (see Secrets section below)**

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

See `vars/vault.yml.example` for the required format. Add the required secrets:
```yaml
---
cloudflare_dns_api_token: "your_token_here"
traefik_dashboard_auth: "admin:$2y$05$..."  # From step 2
pihole_web_password: "your_pihole_password"
vaultwarden_admin_token: "your_vaultwarden_token"
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
- `deploy_vaultwarden`: true
- `deploy_pihole`: true
- `deploy_firewall`: true

### Docker Image Versions

Image versions are pinned in `vars/homelab.yml` for reproducibility:

- Traefik: `traefik:v3.6`
- Portainer: `portainer/portainer-ce:2.21.4`
- Homepage: `ghcr.io/gethomepage/homepage:v1.8.0`
- Nextcloud: `nextcloud/all-in-one:latest` (AIO recommends latest)
- Vaultwarden: `vaultwarden/server:1.34.3`
- Pi-hole: `pihole/pihole:2025.11.1`
- Unbound: `alpinelinux/unbound:latest`

Update these versions in the vars file as needed.

---

## Access

All services are accessible via HTTPS with wildcard TLS certificates:

- Traefik dashboard: https://traefik.yourdomain.com
- Pi-hole: http://192.168.1.92/admin (or https://pihole.yourdomain.com if proxied)
- Portainer: https://portainer.yourdomain.com
- Homepage (dashboard): https://homepage.yourdomain.com
- Nextcloud (when enabled): https://nextcloud.yourdomain.com
- Vaultwarden: https://vault.yourdomain.com

TLS certificates are issued and automatically renewed by Traefik using DNS-01.

Traefik dashboard requires authentication (configured in vault).

### Portainer Initial Setup

After deploying Portainer for the first time:

1. Access https://portainer.yourdomain.com and create your admin account
2. When prompted about connecting to Docker, **dismiss/skip the initial connection error**
3. Manually add the Docker environment:
   - Go to **Environments** → **Add environment**
   - Select **Docker Standalone**
   - Choose **API** as the connection method
   - Enter the Environment URL: `docker-proxy:2375`
   - Leave TLS **disabled** (internal network connection)
   - Click **Connect**

Portainer normally uses `/var/run/docker.sock` directly, but this setup uses docker-socket-proxy for secure, restricted access to the Docker API.

### Vaultwarden Initial Setup

**Before deploying**, add an admin token to your vault file:

```bash
# Generate admin token
openssl rand -base64 48

# Edit vault and add the token
ansible-vault edit vars/vault.yml --ask-vault-pass
```

After deploying Vaultwarden for the first time:

1. Access https://vault.yourdomain.com
2. **Create your first account** while `SIGNUPS_ALLOWED=true` (default)
   - Choose your own strong master password (you'll need this to log in)
   - **If you lose this password, all your data is lost forever**
3. After creating your account, **disable signups** by editing `vars/homelab.yml`:
   - Set `vaultwarden_signups_allowed: false`
   - Redeploy with: `ansible-playbook playbooks/site.yml --ask-vault-pass`
4. Access the admin panel at https://vault.yourdomain.com/admin
   - Use the admin token from your vault file (not your master password)
   - Review and adjust settings as needed

**Security Notes:**
- **Master password**: You choose this when creating your account. Never lose it.
- **Admin token**: Randomly generated, stored in vault.yml, used for `/admin` panel
- Always disable signups after creating your accounts
- Vaultwarden requires HTTPS (handled by Traefik)
- All data is encrypted and stored in `/srv/docker/vaultwarden/vw-data/`

---

### Pi-hole + Unbound Setup

After deploying Pi-hole for the first time:

1. Access the web UI at http://192.168.1.92/admin or https://pihole.yourdomain.com
2. Log in with the password from your vault file (`pihole_web_password`)
3. Pi-hole v6 uses `FTLCONF_webserver_api_password` environment variable for authentication
3. DNS queries flow: Client → Pi-hole (ad blocking) → Unbound (recursive DNS) → Root servers
4. No third-party DNS dependencies (Cloudflare, Google, etc.)

**Testing Unbound:**
```bash
# SSH to the pihole host
ssh pi@192.168.1.92

# Test DNSSEC validation (should return NOERROR)
docker exec -it pihole dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335

# Test DNSSEC failure detection (should return SERVFAIL)
docker exec -it pihole dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
```

---

## Notes

- All services run in Docker containers
- DNS resolution: Pi-hole (localhost:53) → Unbound (localhost:5335) → Root DNS servers
- Firewall rules are managed via Ansible (UFW)
- Designed to be idempotent and safe to re-run

---

## Philosophy

- No click-ops
- No re-installing working systems
- No hidden magic
- Everything version controlled
