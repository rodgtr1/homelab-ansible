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

Host: pihole  
Purpose: Pi-hole DNS (standalone)  
IP: 192.168.1.86  

Pi-hole runs on a separate server and is installed manually.  
Ansible only manages configuration and service state.

---

## Requirements

- SSH key access to all hosts
- Ansible installed locally
- Docker already installed on pimaster
- Pi-hole already installed on pihole
- Cloudflare account + domain (travismedia.cloud)

---

## Cloudflare DNS-01 (Required)

Traefik uses a DNS-01 challenge with Cloudflare to issue wildcard TLS certificates.

You must create a Cloudflare API token with **DNS edit permissions**.

Steps:
1. Log into Cloudflare
2. Go to My Profile → API Tokens
3. Create Token
4. Use the template: "Edit zone DNS"
5. Scope it to the specific zone (your domain)
6. Copy the token value

This token is stored securely using Ansible Vault and is never committed to git.

---

## Secrets (Ansible Vault)

Secrets are stored using Ansible Vault.

Required secret:
- CLOUDFLARE_DNS_API_TOKEN

Edit or create the vault file:

```bash
ansible-vault edit vars/vault.yml
```

---

## Deploy

Dry run (recommended):

```bash
ansible-playbook playbooks/site.yml --check --diff
```

Apply changes:

```bash
ansible-playbook playbooks/site.yml
```

### Service Toggles

Services can be enabled/disabled in `vars/homelab.yml`:

- `deploy_traefik`: true
- `deploy_portainer`: true
- `deploy_nextcloud`: false (disabled by default)
- `deploy_homepage`: true
- `deploy_pihole`: true
- `deploy_firewall`: true

---

## Access

All services are accessible via HTTPS with wildcard TLS certificates:

- Traefik dashboard: https://traefik.travismedia.cloud
- Pi-hole (proxied through Traefik): https://pihole.travismedia.cloud
- Portainer: https://portainer.travismedia.cloud
- Homepage (dashboard): https://homepage.travismedia.cloud
- Nextcloud (when enabled): https://nextcloud.travismedia.cloud

TLS certificates are issued and automatically renewed by Traefik using DNS-01.

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
