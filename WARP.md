# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is an Ansible-managed homelab infrastructure that deploys and configures Docker-based services with Traefik as a reverse proxy. The setup emphasizes repeatability, idempotency, and configuration management—not installation automation.

**Key principle**: This repo assumes servers are already running. It manages configuration state, not initial installation.

## Architecture

The infrastructure is split across two host groups:

**proxy** (pimaster @ 192.168.1.68)
- Runs Traefik (reverse proxy with automatic HTTPS via Cloudflare DNS-01)
- Hosts Docker-based services: Portainer, Nextcloud, Homepage, Vaultwarden
- All services proxied through Traefik with wildcard TLS on `*.yourdomain.com`

**dns** (pihole @ 192.168.1.92)
- Runs Pi-hole v6 (DNS server with Unbound)
- Ansible manages configuration and custom DNS records
- Custom DNS records in `/etc/pihole/hosts/homelab.hosts` for local domain resolution
- UFW firewall rules for ports 22, 53, 80

All services are toggled via `vars/homelab.yml` (e.g., `deploy_traefik: true/false`).

## Essential Commands

### Initial Setup
```bash
# Install required Ansible collections
ansible-galaxy collection install -r requirements.yml

# Create encrypted vault for secrets
ansible-vault create vars/vault.yml --ask-vault-pass

# Edit existing vault
ansible-vault edit vars/vault.yml --ask-vault-pass

# View vault contents
ansible-vault view vars/vault.yml --ask-vault-pass
```

### Deployment
```bash
# Dry run (always recommended first)
ansible-playbook playbooks/site.yml --check --diff --ask-vault-pass

# Apply changes to all hosts
ansible-playbook playbooks/site.yml --ask-vault-pass

# Target specific host group
ansible-playbook playbooks/site.yml --limit proxy --ask-vault-pass
ansible-playbook playbooks/site.yml --limit dns --ask-vault-pass

# Target specific host
ansible-playbook playbooks/site.yml --limit pimaster --ask-vault-pass

# Skip specific roles
ansible-playbook playbooks/site.yml --skip-tags firewall --ask-vault-pass
```

### Testing & Validation
```bash
# Test Ansible connectivity
ansible all -m ping

# Check syntax
ansible-playbook playbooks/site.yml --syntax-check

# List all hosts
ansible-inventory --list

# Show variables for a host
ansible-inventory --host pimaster
```

## Secrets Management

Secrets are stored in `vars/vault.yml` using Ansible Vault encryption. Required secrets:

- `cloudflare_dns_api_token`: Cloudflare API token for DNS-01 ACME challenge
- `traefik_dashboard_auth`: HTTP basic auth for Traefik dashboard (htpasswd format)
- `vaultwarden_admin_token`: Admin token for Vaultwarden admin panel

**Generate Traefik auth**:
```bash
# If htpasswd is installed
htpasswd -nbB admin YourPassword

# Or using Python (if bcrypt module is installed)
python3 -c "import bcrypt; print('admin:' + bcrypt.hashpw(b'YourPassword', bcrypt.gensalt()).decode())"

# Or using Docker
docker run --rm httpd:alpine htpasswd -nbB admin YourPassword
```

**Generate Vaultwarden admin token**:
```bash
openssl rand -base64 48
```

Never commit unencrypted secrets. Always use `--ask-vault-pass` when running playbooks.

## Role Structure

Each role in `roles/` follows standard Ansible structure:
- `tasks/main.yml`: Task definitions
- `templates/*.j2`: Jinja2 templates for config files and docker-compose.yml
- `defaults/main.yml`: Default variables (overridden by `vars/homelab.yml`)
- `handlers/main.yml`: Service restart handlers (e.g., pihole)

**Roles**:
- `common`: Base system configuration
- `docker`: Docker setup and network creation
- `traefik`: Reverse proxy with Let's Encrypt DNS-01
- `portainer`: Docker management UI (uses docker-socket-proxy for security)
- `pihole`: DNS server config management (does NOT install)
- `homepage`: Dashboard service
- `nextcloud`: Self-hosted cloud storage (optional, disabled by default)
- `vaultwarden`: Password manager
- `firewall` / `firewall-dns`: UFW rules for proxy and DNS hosts

## Service Configuration

All service configuration is centralized in `vars/homelab.yml`:

**Deployment toggles**: `deploy_<service>: true/false`

**Service domains**: `<service>_domain: "{{ <service>_host }}.{{ base_domain }}"`

**Docker images**: Pinned versions for reproducibility (update manually in vars file)

**Service directories**: All under `/srv/docker/<service>/`

When adding or modifying services:
1. Update `vars/homelab.yml` with service-specific variables
2. Create or modify role in `roles/<service>/`
3. Add service to appropriate play in `playbooks/site.yml` with conditional `when: deploy_<service>`
4. All services exposed via Traefik MUST include Traefik labels in their docker-compose template

## Docker Compose Pattern

Each role deploys services using `community.docker.docker_compose_v2` module. All templates follow this pattern:

1. Create service directory with proper permissions (`root:docker`, `0755`)
2. Deploy `docker-compose.yml` from template
3. Services requiring secrets use `.env` files (mode `0600`)
4. All web services connect to Traefik via `proxy` network
5. Traefik labels handle routing: `traefik.http.routers.<service>.rule=Host(\`<domain>\`)`

## Important Notes

- Pi-hole v6 is deployed via Ansible with custom DNS records managed in `roles/pihole/templates/custom.list.j2`
- Custom DNS records are deployed to `/etc/pihole/hosts/homelab.hosts` (Pi-hole v6 uses `hostsdir` directive)
- Pi-hole v6 uses `FTLCONF_*` environment variables (not the old v5 variables):
  - `FTLCONF_webserver_api_password` for web authentication (not `WEBPASSWORD`)
  - `FTLCONF_dns_upstreams` for upstream DNS servers (not `PIHOLE_DNS_`)
  - Environment variables set via Docker Compose become read-only in Pi-hole's configuration
- Portainer requires manual initial setup via docker-socket-proxy (see README.md)
- Vaultwarden requires disabling signups after initial account creation
- All changes should be tested with `--check --diff` before applying
- Playbooks are idempotent—safe to re-run repeatedly
- DNS resolution: Local DNS (Pi-hole) → Traefik → Services
- TLS certificates auto-renew via Traefik + Cloudflare DNS-01
- Docker containers on proxy host must use Pi-hole DNS for local domain resolution

## Troubleshooting

**Vault password issues**: Ensure you're using `--ask-vault-pass` flag

**Service won't start**: Check logs with `docker compose logs -f` in service directory (e.g., `/srv/docker/traefik/`)

**Network issues**: Verify `proxy` network exists: `docker network ls`

**Firewall blocking**: Check UFW rules: `sudo ufw status verbose`

**Traefik certificate issues**: Inspect `acme.json` and Cloudflare API token in vault

**Role changes not applying**: Ensure host is in correct group (proxy/dns) in `inventory/homelab.yml`

**DNS issues after Pi-hole IP change**:
1. Update Pi-hole IP in `vars/homelab.yml` (`pihole_ip` variable)
2. Update `/etc/resolv.conf` on proxy host to point to new Pi-hole IP
3. Make resolv.conf immutable: `sudo chattr +i /etc/resolv.conf`
4. Restart Docker daemon to update container DNS: `sudo systemctl restart docker`
5. Verify containers can resolve domains: `docker exec <container> cat /etc/resolv.conf`
