# Home Lab Build Log

A self-hosted home server built on consumer and prosumer hardware, running containerized services behind a hardened Linux host. This repo documents the full build: hardware selection, OS installation, security hardening, networking, storage, and the services running on top of it.

> **Why build this?** Hands-on CCNA supplementation, a real target for Wazuh SIEM monitoring, and a platform for running self-hosted services I actually use. Everything here is something I configured, broke, and fixed myself.

---

## Table of Contents

1. [Hardware](#1-hardware)
2. [Network Topology](#2-network-topology)
3. [Ubuntu Server — Installation & First Boot](#3-ubuntu-server--installation--first-boot)
4. [Unattended Upgrades](#4-unattended-upgrades)
5. [SSH Hardening](#5-ssh-hardening)
6. [Firewall — UFW](#6-firewall--ufw)
7. [Kernel Tuning](#7-kernel-tuning)
8. [Storage — RAID Array & Network Share](#8-storage--raid-array--network-share)
9. [Tailscale — Mesh VPN](#9-tailscale--mesh-vpn)
10. [Docker & Containerized Services](#10-docker--containerized-services)
    - [Jellyfin](#jellyfin)
    - [NextCloud](#nextcloud)
    - [Wazuh SIEM](#wazuh-siem)
    - [Nginx Proxy Manager](#nginx-proxy-manager)
11. [Photos](#11-photos)
12. [What's Next](#12-whats-next)

---

## 1. Hardware

| Component | Model | Notes |
|-----------|-------|-------|
| Server | Lenovo ThinkCentre M910q | Upgraded to 32 GB RAM |
| CPU | Intel i5-6500T | iGPU used for Jellyfin hardware transcoding |
| Storage | Glyph Blackbox Pro 40TB | Hardware-mirrored RAID 1 — 18.2TB usable |
| Router | Ubiquiti EdgeRouter X | Handles all routing; Nighthawk demoted to AP |
| WiFi | NetGear Nighthawk | Access point only |
| Rack | GeekPi 8U mini rack | |
| OS | Ubuntu Server 24.04 LTS | |
| Managed Switch | NetGear GS308e | 8-port smart switch; dot1q VLAN port assignments |
| DNS / Lab Host | Raspberry Pi 3B+ | Pi-hole in Docker; Wazuh agent; VLAN 20 lab segment |

---

## 2. Network Topology

### 2.1 Physical Layout

```
ISP modem
   └── Ubiquiti EdgeRouter X eth1 and eth2
        └──NetGear GS308e (managed switch — dot1q port assignments)
           ├── eth1 → VLAN 1 (native, 192.168.1.0/24) — main network
           │    ├── Lenovo ThinkCentre (Ubuntu Server — static IP)
           │    │    ├── Glyph Blackbox Pro (18.2TB RAID 1, /mnt/media-storage/)
           │    │    ├── Docker: Tailscale (host network — mesh VPN + subnet router)
           │    │    ├── Docker: Wazuh SIEM (dashboard on :8443) [Wazuh agent]
           │    │    ├── Docker: Jellyfin (iGPU passthrough)
           │    │    ├── Docker: NextCloud + MariaDB
           │    │    ├── Docker: Nginx Proxy Manager (:80, :443, :81)
           │    │    └── Samba (SMB → Windows and Mac desktops on LAN)
           │    └── Windows Desktop (custom build — VLAN 1 client)
           └── eth2 → VLAN 20 (lab-only, 192.168.2.0/24) — isolated lab segment
                     └── Raspberry Pi 3B+ (static IP — 192.168.2.x) [Wazuh agent]
                          └── Docker: Pi-hole (local DNS resolver + ad filtering)
```

### 2.2 VLAN Configuration

Two VLANs segment the network between general use and lab-only traffic:

| VLAN | ID | Subnet | Interface | Purpose |
|------|----|--------|-----------|---------|
| Main (native) | 1 | 192.168.1.0/24 | eth1 | General network — server, desktops, WiFi clients |
| Lab | 20 | 192.168.2.0/24 | eth2 | Isolated lab segment — Raspberry Pi, lab devices |

**EdgeRouter X configuration:** eth1 carries the native VLAN 1 for all general traffic. eth2 is dedicated to VLAN 20 — the lab-only segment.

**GS308e configuration:** Specific switch ports assigned via dot1q tagging — VLAN 20 ports carry only lab traffic, isolating those devices from the main network. The GS308e's web interface handles port-based VLAN assignment without a CLI.

**Why segment?** Keeps lab/experimental devices (Kali Pi, TryHackMe traffic, test hosts) isolated from the main network. Devices on VLAN 20 cannot reach VLAN 1 resources without explicit routing rules. ACL implementation planned to formalize inter-VLAN traffic policy.

**Custom local domains (via Nginx Proxy Manager + Pi-hole local DNS):**
- `jellyfin.<YOUR_LOCAL_DOMAIN>` → Jellyfin
- `cloud.<YOUR_LOCAL_DOMAIN>` → NextCloud

Tailscale runs in `network_mode: host`, giving it direct access to the server's network interface. This allows remote devices (phone over cellular, MacBook off-site) to reach all local services via Tailscale IP without opening any firewall ports to the internet.

---

## 3. Ubuntu Server — Installation & First Boot

### Install

Standard Ubuntu Server 24.04 LTS installation via bootable USB. After first boot, encountered a connectivity issue immediately.

**Symptom:** `apt update` failed — could not resolve package repos. No internet.

**Troubleshoot (OSI model, bottom-up):**

```bash
# Layer 1 — Is the interface even up?
ip link show
# enp0s31f6 is UP — physical is fine

# Layer 3 — Does it have an address?
ip addr show enp0s31f6
# No address assigned. Problem found.

ip route show
# No default route either. Confirms no DHCP lease.
```

**Root cause:** Netplan config file was missing `dhcp4: true` on the primary interface.

```bash
ls -a /etc/netplan/
# 00-installer-config.yaml
sudo nano /etc/netplan/00-installer-config.yaml
```

Added the missing directive, applied, confirmed:

```yaml
network:
  ethernets:
    enp0s31f6:
      dhcp4: true   # ← this line was missing
  version: 2
```

```bash
sudo netplan apply
ip addr show enp0s31f6
# Address now assigned. Connectivity restored.
```

### Initial Updates

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y

# Reboot if kernel was updated
[ -f /var/run/reboot-required ] && sudo reboot
```

The last line checks whether `/var/run/reboot-required` exists — Ubuntu creates it when a kernel update needs a restart before taking effect. The reboot only runs if that file is actually there.

---

## 4. Unattended Upgrades

```bash
sudo apt install unattended-upgrades apt-listchanges -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Verify it's active:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
# Should show:
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";
```

Configure behavior:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Uncommented and set:
- `Unattended-Upgrade::Automatic-Reboot "true";`
- `Unattended-Upgrade::Automatic-Reboot-Time "02:00";`

---

## 5. SSH Hardening

### Key-Based Authentication Only

Generated an ed25519 key pair on the Mac — the only machine that should have SSH access:

```bash
# On MacBook
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_github
cat ~/.ssh/id_ed25519_github.pub
```

Copied the public key to the server's `authorized_keys`:

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
# Paste the full public key on one line

sudo chmod 755 /home/yourusername
sudo chmod 700 /home/yourusername/.ssh
sudo chmod 600 /home/yourusername/.ssh/authorized_keys
sudo chown -R yourusername:yourusername /home/yourusername/.ssh
```

### sshd Hardening Config

```bash
sudo nano /etc/ssh/sshd_config.d/hardening.conf
```

```
Port <YOUR_SSH_PORT>
PasswordAuthentication no
PermitRootLogin no
PermitEmptyPasswords no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers yourusername
X11Forwarding no

KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp521
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

SSH runs on a non-default port (registered range 1024–49151) rather than the default 22. This doesn't stop a determined attacker but eliminates the majority of automated internet bots that only scan port 22.

> **Gotcha:** Another `.conf` file in the same directory was overriding `PasswordAuthentication`. Found this by running `sudo sshd -T | grep passwordauthentication` and seeing it was still enabled despite my config. Commented out the conflicting line in the other file.

Verify and restart:

```bash
sudo sshd -T | grep -i 'authorizedkeysfile'
sudo sshd -T | grep passwordauthentication
sudo systemctl restart ssh
```

> **Important:** Confirm key-based login works in a *separate terminal* before closing the current session. If you lock yourself out, you need console access to fix it.

---

## 6. Firewall — UFW

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow <YOUR_SSH_PORT>/tcp    # SSH on custom port
sudo ufw enable
sudo ufw status verbose
```

Additional ports are opened as services come online. Docker has its own iptables rules — be aware that Docker can bypass UFW by default when containers bind to `0.0.0.0`.

---

## 7. Kernel Tuning

Wazuh's indexing database (OpenSearch) requires a higher memory map limit than Linux's default. Without this, the Wazuh indexer crashes on startup.

```bash
sudo nano /etc/sysctl.conf
```

Add:

```
vm.max_map_count=262144
```

Apply immediately without rebooting:

```bash
sudo sysctl -p
```

This setting persists across reboots via `sysctl.conf`.

---

## 8. Storage — RAID Array & Network Share

### Glyph Blackbox Pro — Hardware RAID 1

The Glyph Blackbox Pro 40TB is configured in hardware-mirrored RAID 1, giving 18.2TB of usable space with full drive-level redundancy. RAID is handled by the enclosure itself — the OS sees a single drive.

**Partition & format:**

```bash
# Identify the device
lsblk

# Partition with GPT
sudo gdisk /dev/sdX
# Create a single partition spanning the full disk

# Format to ext4
sudo mkfs.ext4 /dev/sdX1

# Create mount point
sudo mkdir -p /mnt/media-storage
```

**Mount permanently via fstab:**

```bash
# Get the UUID
sudo blkid /dev/sdX1

sudo nano /etc/fstab
# Add:
# UUID=your-uuid-here  /mnt/media-storage  ext4  defaults  0  2

sudo mount -a
# Verify
df -h /mnt/media-storage
```

### Samba — SMB Network Share

Samba exposes the media storage array as a Windows-compatible network share, allowing the Windows desktop to transfer files directly to the server over the LAN.

```bash
sudo apt install samba -y
sudo nano /etc/samba/smb.conf
```

Key share configuration:

```ini
[media]
   path = /mnt/media-storage
   browseable = yes
   read only = no
   valid users = yourusername
```

```bash
sudo smbpasswd -a yourusername
sudo systemctl restart smbd
```

On the Windows machine, the share is mapped as a network drive via the command prompt.

---

## 9. Tailscale — Mesh VPN

Tailscale provides zero-config mesh VPN between trusted devices without opening any firewall ports to the internet. Deployed as a Docker container with `network_mode: host` so it binds directly to the server's network interface rather than through Docker's bridge — this is required for subnet routing to work correctly.

With subnet routing enabled, any Tailscale-connected device (phone over cellular, MacBook off-site) can reach all local services by IP or custom domain, as if it were physically on the LAN.

See [`configs/docker-compose.yml`](configs/docker-compose.yml) for the full service definition.

```bash
# Verify Tailscale is up
sudo docker exec tailscale tailscale status

# Enable subnet routing
sudo docker exec tailscale tailscale up \
  --advertise-routes=<YOUR_LOCAL_SUBNET> \
  --accept-routes
```

Approve the subnet route in the Tailscale admin panel.

---

## 10. Docker & Containerized Services

### Docker Install

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Log out and back in
```

All services are defined in [`configs/docker-compose.yml`](configs/docker-compose.yml).

```bash
cd ~/homeserver
sudo docker compose up -d
sudo docker compose ps
```

---

### Jellyfin

**What it is:** Open-source self-hosted media server — no subscription, no external dependency.

**Setup highlights:**
- Intel i5-6500T integrated GPU is passed directly into the container for hardware-accelerated video transcoding, avoiding CPU bottlenecks on large files
- Library paths mapped to `/mnt/media-storage/` on the Glyph array
- Accessible via `jellyfin.<YOUR_LOCAL_DOMAIN>` (custom local domain through Nginx Proxy Manager)
- Available remotely via Tailscale IP without any open firewall ports

---

### Pi-hole

**What it is:** Network-wide DNS resolver and ad blocker — all DNS queries from lab devices route through Pi-hole before hitting upstream resolvers.

**Setup highlights:**
- Running in Docker on a dedicated Raspberry Pi 3B+ on the VLAN 20 lab segment (192.168.2.0/24)
- Acts as the local DNS resolver for the lab network — provides network-level filtering and full DNS query visibility across all connected devices
- Local DNS records configured for custom domain shortcuts (e.g. `jellyfin.<YOUR_LOCAL_DOMAIN>`, `cloud.<YOUR_LOCAL_DOMAIN>`) — these resolve at the Pi-hole level across the LAN without needing external DNS or SSL certs for local access
- Upstream resolvers configured (e.g. Cloudflare 1.1.1.1, Google 8.8.8.8) with DNS-over-HTTPS option available
- Pi-hole Pi is also a Wazuh monitored endpoint — DNS query logs feed into centralized SIEM monitoring

---

### NextCloud

**What it is:** Self-hosted cloud storage — file sync across all devices, private alternative to Google Drive.

**Setup highlights:**
- Deployed alongside a dedicated MariaDB container (better performance than NextCloud's default SQLite)
- Data stored on the Glyph array at `/mnt/media-storage/nextcloud/`
- `config.php` updated with trusted domains: local IP, Tailscale IP, and custom domain shortcut
- Accessible via `cloud.<YOUR_LOCAL_DOMAIN>` locally; Tailscale IP remotely
- Mobile app verified over cellular via Tailscale

---

### Wazuh SIEM

**What it is:** Open-source Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) platform.

**Setup highlights:**
- Deployed from the official Wazuh Docker production repository (not a third-party image)
- SSL certificates auto-generated during initial deployment
- Dashboard shifted from default port 443 to **port 8443** to clear the way for Nginx Proxy Manager on 443
- **Three active Wazuh agents** feeding live telemetry into the central dashboard:
  - **Lenovo ThinkCentre** (the server itself — process activity, file integrity, Docker container events)
  - **MacBook Pro** (network connections, process activity, file integrity events)
  - **Raspberry Pi 3B+** (Pi-hole host — DNS events, process monitoring, system health)
- Centralized log aggregation across all three endpoints; alerts configured for anomalous activity

> Wazuh's OpenSearch indexer requires `vm.max_map_count=262144` — see [Kernel Tuning](#7-kernel-tuning).

---

### Nginx Proxy Manager

**What it is:** Reverse proxy with a web UI — routes incoming requests by hostname to the correct backend container, handles SSL termination.

**Setup highlights:**
- Runs on ports 80 (HTTP), 443 (HTTPS), and 81 (admin UI)
- Dashboard shifted Wazuh off 443 first to avoid conflict
- Custom local domain names configured:
  - `jellyfin.<YOUR_LOCAL_DOMAIN>` → Jellyfin container
  - `cloud.<YOUR_LOCAL_DOMAIN>` → NextCloud container
- Let's Encrypt SSL integration planned for external-facing access

---

## 11. Photos

> *Photos documenting the physical build (rack, cabling, hardware, storage array) — coming soon.*
>
> !assets/after.jpeg

---

## 12. What's Next

### Completed ✓
- Jellyfin library cleanup and re-scan
- NextCloud mobile app verified over Tailscale on cellular
- Nginx Proxy Manager local domain shortcuts configured
- Pi-hole deployed on Raspberry Pi 3B+ (VLAN 20); local DNS records configured
- VLAN segmentation live: VLAN 1 (main, 192.168.1.0/24) and VLAN 20 (lab, 192.168.2.0/24)
- Wazuh agents deployed on all three hosts: ThinkCentre, MacBook Pro, Raspberry Pi 3B+

### In Progress / Planned

- [ ] **ACLs — Inter-VLAN traffic policy:** Define and implement firewall rules on the EdgeRouter X to formally control which traffic is permitted between VLAN 1 and VLAN 20. Currently evaluating what cross-VLAN access should be permitted vs. denied by default.

- [ ] **SSL certificates via Let's Encrypt:** Currently using Pi-hole local DNS records to resolve custom domains on the LAN without HTTPS. SSL certs via Nginx Proxy Manager's Let's Encrypt integration would enable trusted HTTPS for local services — evaluating whether the added complexity is worthwhile for a local-only setup.

- [ ] **Wazuh — expand alerting rules:** Fine-tune alert thresholds and add custom rules for lab-specific events (inter-VLAN connection attempts, Pi-hole DNS anomalies, Docker container restarts).

- [ ] **CCNA lab expansion:** Add Packet Tracer / GNS3 topology files to the `ccna-labs` repo as coursework progresses.
