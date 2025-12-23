# Homelab Ansible

Minimal Ansible setup for managing a homelab with:

- Traefik (DNS-01, Cloudflare, HTTPS everywhere)
- Docker
- Firewall (UFW)
- Pi-hole (managed, NOT installed)

This repo assumes you already have servers running and want repeatable configuration — not installers.

---

## Architecture

Host: pimaster  
Purpose: Docker + Traefik reverse proxy  
IP: 192.168.1.68  

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
- Cloudflare account + domain

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

ansible-vault edit vars/vault.yml

---

## Deploy

Dry run (recommended):

ansible-playbook playbooks/site.yml --check --diff

Apply changes:

ansible-playbook playbooks/site.yml

---

## Access

Traefik dashboard:
https://traefik.<your-domain>

Pi-hole (proxied through Traefik):
https://pihole.<your-domain>

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
