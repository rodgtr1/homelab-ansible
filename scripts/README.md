# Raspberry Pi Setup Scripts

These scripts prepare fresh Raspberry Pi installations before Ansible deployment.

## Overview

After installing Ubuntu Server on your Raspberry Pi, run the appropriate setup script to configure network interfaces with static IPs and proper DNS settings.

## Prerequisites

- Fresh Ubuntu Server installation on Raspberry Pi
- Physical or console access to the Pi
- WiFi credentials (if using wireless)
- Ethernet cable (recommended for initial setup)

## Deployment Order

⚠️ **Important**: Deploy Pi-hole FIRST, then standard hosts.

1. Run `setup-pihole-host.sh` on your DNS server
2. Deploy Pi-hole via Ansible
3. Verify Pi-hole is working
4. Run `setup-standard-host.sh` on application servers
5. Deploy applications via Ansible

This ensures all hosts can use Pi-hole for DNS from the start.

## Scripts

### `setup-pihole-host.sh`

Prepares a Raspberry Pi to run Pi-hole + Unbound DNS server.

**Features:**
- Disables systemd-resolved to free port 53 for Pi-hole
- Configures eth0 (wired) as primary interface with metric 100
- Configures wlan0 (wireless) as backup with metric 200
- Sets DNS to 127.0.0.1 (localhost) for Pi-hole, with fallback to 1.1.1.1
- Assigns static IP addresses
- Makes configuration immutable

**Usage:**
1. Edit the script variables:
   ```bash
   default_gateway="192.168.1.1"
   wifi_ssid="YOUR_WIFI_SSID"
   wifi_plain_password="YOUR_WIFI_PASSWORD"
   ```

2. Copy script to the Pi:
   ```bash
   scp setup-pihole-host.sh pi@<pi-ip>:/home/pi/
   ```

3. SSH to the Pi and run:
   ```bash
   chmod +x setup-pihole-host.sh
   ./setup-pihole-host.sh
   ```

4. Note the eth0 IP address displayed (e.g., `192.168.1.92`)

5. Update Ansible configuration with the IP:
   - `inventory/homelab.yml`: Set `ansible_host` for pihole
   - `vars/homelab.yml`: Set `pihole_ip`

### `setup-standard-host.sh`

Prepares a Raspberry Pi for general services (Traefik, Portainer, etc.).

**Features:**
- Configures eth0 (wired) as primary interface with metric 100
- Configures wlan0 (wireless) as backup with metric 200
- Sets DNS to Pi-hole (192.168.1.92) for network-wide ad blocking
- Assigns static IP addresses

**Usage:**
1. Edit the script variables:
   ```bash
   nameserver="192.168.1.92"  # Your Pi-hole IP
   default_gateway="192.168.1.1"
   wifi_ssid="YOUR_WIFI_SSID"
   wifi_plain_password="YOUR_WIFI_PASSWORD"
   ```

2. Copy script to the Pi and run (same process as pihole-host script)

## Network Architecture

### Interface Priority

Both scripts configure dual interfaces with priority routing:

- **eth0 (wired)**: Metric 100 (lower = higher priority)
- **wlan0 (wireless)**: Metric 200 (backup)

Traffic always routes through eth0 when available. If eth0 fails, traffic automatically fails over to wlan0.

### DNS Configuration

**Pi-hole host:**
```
127.0.0.1       # Pi-hole (when running)
1.1.1.1         # Cloudflare fallback (for bootstrapping/failures)
```

**Standard hosts:**
```
192.168.1.92    # Pi-hole (network-wide ad blocking)
1.1.1.1         # Cloudflare fallback
```

## After Running Scripts

1. **Verify network connectivity:**
   ```bash
   ping -c 3 8.8.8.8
   ping -c 3 google.com
   ```

2. **Test SSH access:**
   ```bash
   ssh pi@<assigned-ip>
   ```

3. **Deploy with Ansible:**
   ```bash
   ansible-playbook playbooks/site.yml --limit <host> --ask-vault-pass
   ```

## Troubleshooting

**Script fails to get IP address:**
- Ensure ethernet cable is plugged in
- Check DHCP server on your router
- Verify network interface name (should be `eth0`)

**Can't reach Pi after reboot:**
- Check both IP addresses (wired and wireless)
- Try connecting to wlan0 IP if eth0 isn't responding
- Use console/monitor access to verify netplan configuration

**DNS not resolving:**
- For Pi-hole host: This is expected until Pi-hole is deployed
- The fallback DNS (1.1.1.1) should provide connectivity
- After Ansible deployment, DNS will work via Pi-hole

## Notes

- Scripts make `/etc/resolv.conf` immutable to prevent Ubuntu from overwriting it
- Static IPs prevent address changes after reboot
- Dual-interface setup provides redundancy
- Pi-hole host uses localhost DNS to avoid circular dependencies
