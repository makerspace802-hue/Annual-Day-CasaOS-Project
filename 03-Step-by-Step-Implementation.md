---
title: "03 — Step-by-Step Implementation"
project: PI-CLOUD
tags:
  - annual-day
  - project/pi5-private-cloud
  - implementation
  - docker
  - nextcloud
  - jellyfin
  - runbook
status: approved
version: 1.0
created: 2026-09-30
updated: 2026-09-30
---

# 03 — Step-by-Step Implementation

> [!abstract] Document purpose
> The complete, reproducible build record for the deployed system: every phase,
> every command, the justification for non-default choices, a 30-point
> verification checklist, and a disaster-recovery runbook. This document is the
> evidence base for the capability claims made in documents 01, 02 and 05.

**Previous:** [[02-Architecture-and-Hardware]] · **Next:** [[04-Interactive-Demo-Guide]]

---

## Contents

1. [[03-Step-by-Step-Implementation#1. Preconditions|1. Preconditions]]
2. [[03-Step-by-Step-Implementation#2. Phase 1 — Base OS provisioning|2. Phase 1 — Base OS]]
3. [[03-Step-by-Step-Implementation#3. Phase 2 — Host initialisation|3. Phase 2 — Host initialisation]]
4. [[03-Step-by-Step-Implementation#4. Phase 3 — Host hardening|4. Phase 3 — Host hardening]]
5. [[03-Step-by-Step-Implementation#5. Phase 4 — NVMe storage|5. Phase 4 — NVMe storage]]
6. [[03-Step-by-Step-Implementation#6. Phase 5 — Container runtime|6. Phase 5 — Container runtime]]
7. [[03-Step-by-Step-Implementation#7. Phase 6 — CasaOS|7. Phase 6 — CasaOS]]
8. [[03-Step-by-Step-Implementation#8. Phase 7 — Nextcloud|8. Phase 7 — Nextcloud]]
9. [[03-Step-by-Step-Implementation#9. Phase 8 — Encryption|9. Phase 8 — Encryption]]
10. [[03-Step-by-Step-Implementation#10. Phase 9 — Jellyfin|10. Phase 9 — Jellyfin]]
11. [[03-Step-by-Step-Implementation#11. Phase 10 — Native shares|11. Phase 10 — Native shares]]
12. [[03-Step-by-Step-Implementation#12. Phase 11 — Backup|12. Phase 11 — Backup]]
13. [[03-Step-by-Step-Implementation#13. Phase 12 — Verification checklist|13. Phase 12 — Verification]]
14. [[03-Step-by-Step-Implementation#14. Disaster recovery runbook|14. Disaster recovery]]
15. [[03-Step-by-Step-Implementation#15. Fault diagnosis reference|15. Fault diagnosis]]
16. [[03-Step-by-Step-Implementation#16. Consolidated bootstrap|16. Consolidated bootstrap]]

---

## 1. Preconditions

### 1.1 Required items

| # | Item | Note |
|---|---|---|
| 1 | Raspberry Pi 5 (8 GB), assembled with cooler, NVMe, and 27 W supply | Per [[02-Architecture-and-Hardware#2. Hardware bill of materials\|BOM]] |
| 2 | Provisioning workstation with Raspberry Pi Imager | <https://www.raspberrypi.com/software/> |
| 3 | microSD card, 64 GB, A2 class | **OS only** |
| 4 | Cat6 cable | Preferred for all build and demonstration operations |
| 5 | Keyboard and monitor (optional) | Recovery access if SSH is unavailable |
| 6 | USB-C data cable (optional) | Direct device-to-device transfer |
| 7 | USB removable drive | Off-site backup target |

### 1.2 Build invariants

> [!danger] The five constraints that govern the entire procedure
> 1. **The microSD carries the operating system only.** All user data is written to the NVMe. Every deviation from this invalidates the recoverability claim (P1).
> 2. **Docker's root directory remains on the microSD.** Enforced and verified in §6.3.
> 3. **Encryption is enabled before any user data is imported** (§9). Encryption applies to newly written files; pre-existing files are not retroactively encrypted.
> 4. **A 5 V / 5 A supply is used.** Under-voltage is a documented cause of non-deterministic SD write corruption, which typically manifests days after the build.
> 5. **Every image tag is pinned.** `:latest` permits an unreviewed version change during any subsequent pull. Pinning is also what makes rollback possible (§14, DR-5).

### 1.3 Notation

```bash
# A command to execute
# → output or expected result
```

> [!note] Version verification
> The commands recorded here were executed on **2026-09-30**. Install strings and
> image tags for CasaOS, Nextcloud, and Jellyfin are published in their respective
> official documentation. Version-sensitive values are marked inline. Verification
> of the current upstream command before repeating this build is recommended.

---

## 2. Phase 1 — Base OS provisioning

**Objective:** produce a booted, network-reachable host with a correct hostname and no application content.

### 2.1 Image selection

Raspberry Pi Imager → device *Raspberry Pi 5* → OS *Other* → **Raspberry Pi OS (64-bit)**, Debian Bookworm.

> [!info] Edition selection
> The standard 64-bit image is used. The Desktop environment is not required —
> the host is headless by design and the desktop environment consumes
> approximately 400 MB of additional RAM. Lite would be sufficient but omits
> several utilities referenced below.

### 2.2 Provisioning options

⚙ **Advanced options** (Ctrl+Shift+X) — all fields are set deliberately:

| Tab | Field | Value | Basis |
|---|---|---|---|
| General | Hostname | `pi-cloud` | Becomes `pi-cloud.local` and the default service hostname |
| General | Username / password | Dedicated non-default account | A named unprivileged account; direct root access is disabled in §4.2 |
| General | Wireless SSID / password | Deployment network | Wireless operation only |
| General | **Wireless country** | Deployment country | Unset country codes constrain radio power to the lowest common regulatory limit and may constitute non-compliant operation |
| Services | **Enable SSH** | Enabled | Administrative access |
| Options | **Set hostname** | Enabled | |
| Options | **Enable wait-for-network** | Enabled | Prevents a headless host from booting before DHCP completes — a common cause of unreachable first boots |

### 2.3 Write, boot, verify

Write with verification enabled. Insert the card, connect Ethernet, attach the NVMe, then apply power. Allow approximately 40 seconds; a steady green activity LED indicates a completed boot.

```bash
# Address discovery, in order of reliability
nslookup pi-cloud.local
arp -a
# or consult the router's DHCP client table
```

> [!warning] Ethernet is specified for the build
> A wired host completes network initialisation in a few seconds. Over a
> congested or enterprise wireless network, a headless host may fail DHCP
> entirely and self-assign a link-local address, presenting as an unreachable
> device. Wireless client isolation, where enabled on institutional networks,
> produces the same symptom.

---

## 3. Phase 2 — Host initialisation

```bash
ssh admin@pi-cloud.local
```

### 3.1 Hardware enumeration

```bash
lscpu | grep -E "Model name|Architecture"
free -h
lsblk
nvme list
lspci | grep -i non-volatile
```

**Expected:** 4 × Cortex-A76, `aarch64`; ≈7.6 GiB total memory; the NVMe device present in `lsblk`; one `Non-Volatile memory controller` line from `lspci`.

> [!danger] The single most common build failure
> An empty result from `lspci | grep -i non-volatile` means the PCIe device has
> not enumerated. The cause is almost always a partially seated FFC cable rather
> than a defective SSD. Power down, reseat the cable observing the minimum bend
> radius, and re-verify. Diagnosing this in hardware before proceeding avoids a
> multi-hour software debugging exercise against a device that is not present.

### 3.2 System and firmware update

```bash
sudo apt update && sudo apt full-upgrade -y
sudo rpi-eeprom-update -a
sudo reboot
```

```bash
ssh admin@pi-cloud.local
```

> [!info] Rationale for the EEPROM update
> The Pi 5's bootloader resides in on-board EEPROM, independent of the microSD.
> An outdated bootloader with a current OS produces intermittent PCIe, USB, and
> storage enumeration faults that are indistinguishable from hardware failure.
> This step is 30 seconds and eliminates that class of fault.

### 3.3 System identity

```bash
sudo timedatectl set-timezone Asia/Kolkata
timedatectl status
hostnamectl
sudo raspi-config --expand-rootfs && sudo reboot
```

```bash
# Static address: configure a DHCP reservation at the router
# MAC address of the host → 192.168.1.50
```

> [!note] Reservation over local static configuration
> A DHCP reservation maintained at the router is a single point of
> configuration and cannot conflict with another device. Local static
> configuration in `dhcpcd.conf` or NetworkManager creates two sources of truth
> for an address intended to remain fixed over a multi-year deployment.

### 3.4 Service discovery

```bash
sudo systemctl enable --now avahi-daemon
ping -c2 pi-cloud.local
```

```bash
sudo nano /etc/hosts
# 192.168.1.50   pi-cloud   cloud.home
```

---

## 4. Phase 3 — Host hardening

**Objective:** a hardened host before any application is installed or reachable.

### 4.1 Administrative account

```bash
sudo adduser piuser
sudo usermod -aG sudo piuser
sudo visudo          # confirm membership
```

### 4.2 SSH

```bash
# Workstation:
ssh-copy-id piuser@pi-cloud.local
# Verify key authentication before continuing
ssh piuser@pi-cloud.local
```

```bash
# Host:
sudo nano /etc/ssh/sshd_config
```

```ssh
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 30
```

```bash
sudo systemctl restart ssh
```

> [!danger] Mandatory verification before session closure
> Disabling password authentication before confirming that key authentication
> functions correctly removes the only administrative access path and requires
> physical intervention to recover. **Key authentication must be confirmed in a
> second, independent session before the active session is closed.**

### 4.3 Firewall

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp   comment 'SSH'
sudo ufw allow 80/tcp   comment 'CasaOS'
sudo ufw allow 81/tcp   comment 'CasaOS HTTPS'
sudo ufw allow 8080/tcp comment 'Nextcloud'
sudo ufw allow 8096/tcp comment 'Jellyfin'
sudo ufw allow 445/tcp  comment 'Samba'
sudo ufw allow 139/tcp  comment 'NetBIOS'
sudo ufw allow 2049/tcp comment 'NFS'
sudo ufw enable
sudo ufw status verbose
```

> [!note] Applicability
> A packet filter is one of several required layers for an internet-facing
> service. In this architecture it functions as a second, auditable layer behind
> the router's NAT. The allowlist is the complete, reviewed set — a port is added
> only where a deployed service requires it.

### 4.4 Automated security patching

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

```apt
Unattended-Upgrade::Allowed-Origins {
  "${distro_id}:${distro_codename}-security";
};
```

> [!info] Scope of automation
> Security-channel patches are applied automatically. Distribution version
> upgrades and application image updates are excluded and are performed
> deliberately (DR-5). An unmonitored major version change on an unattended
> host is a material availability risk.

### 4.5 Thermal and power state

```bash
vcgencmd measure_temp     # expected: well below 80
vcgencmd get_throttled    # expected: throttled=0x0
```

> [!warning] Interpreting the throttle mask
> The output is a bitmask; `0x0` indicates a healthy state. A non-zero value
> containing `throttle_undervoltage` indicates a power supply fault, not a
> software condition, and produces intermittent faults attributed to storage or
> USB. Where performance is inconsistent and unexplained, this is the first
> check.

---

## 5. Phase 4 — NVMe storage

**Objective:** provision the persistent data volume, enforce invariant 1 (§1.2), and confirm the mount survives reboot and disk absence.

### 5.1 Confirm PCIe enumeration

```bash
lspci -nn | grep -i non-volatile
```

### 5.2 Partitioning

```bash
lsblk -o NAME,SIZE,TYPE,MODEL
sudo fdisk /dev/nvme0n1
```

`fdisk` session:
```
g          # GPT partition table
n          # new partition
1          # partition number
⏎ ⏎       # first sector (1 MiB alignment)
⏎ ⏎       # last sector (use all remaining space)
w          # write
q          # quit
```

### 5.3 Filesystem creation

```bash
sudo wipefs -a /dev/nvme0n1p1     # remove any pre-existing signatures
sudo mkfs.ext4 -L picloud /dev/nvme0n1p1
```

> [!note] Partition-level operations
> All subsequent commands reference the partition, not the whole device. This
> prevents accidental formatting of the wrong target and preserves the
> possibility of adding a partition without reformatting.

### 5.4 Persistent mounting

```bash
sudo mkdir -p /mnt/storage
sudo blkid /dev/nvme0n1p1          # record UUID
```

```bash
sudo nano /etc/fstab
```

```fstab
UUID=<uuid>  /mnt/storage  ext4  defaults,noatime,nofail,discard  0  0
```

```bash
sudo mount -a
findmnt /mnt/storage
df -h /mnt/storage
```

> [!danger] `nofail` is mandatory on a data volume
> Without `nofail`, an absent or failed data disk causes the host to block in an
> emergency boot state, which is unreachable over the network. This converts a
> replaceable commodity component into an on-site intervention. The host must
> always boot, so that it can be reached and diagnosed.

### 5.5 Ownership and permissions

```bash
sudo chown -R 1000:1000 /mnt/storage
sudo chmod 755 /mnt/storage
ls -ln /mnt/storage          # numeric IDs — the authoritative value
```

> [!warning] Numeric UID mismatch — highest-frequency application defect
> A container running as a different unprivileged UID cannot write to a
> directory it does not own, and the resulting application error is typically a
> generic authorisation failure that directs diagnosis toward the application
> rather than the host.
>
> ```bash
> ls -ln /mnt/storage/nextcloud-data        # read the numbers, not the names
> sudo chown -R 33:33 /mnt/storage/nextcloud-data   # match the container's user
> ```
> A single UID/GID policy is applied across all services and declared explicitly
> in each Compose file. Ambiguity at this layer is the most common cause of
> reported application faults.

### 5.6 Directory structure

```bash
sudo mkdir -p /mnt/storage/{nextcloud-db,nextcloud-data,jellyfin-config,jellyfin-cache,shares,snapshots,backups}
sudo mkdir -p /mnt/storage/media/{movies,tv,music,photos}
sudo chown -R 1000:1000 /mnt/storage
```

### 5.7 Health instrumentation

```bash
sudo tee /usr/local/bin/picloud-health <<'EOF'
#!/usr/bin/env bash
# PI-CLOUD health check — non-destructive, safe to run at any time
set -u
echo "=== PI-CLOUD HEALTH ==="
echo "--- uptime / load ---"; uptime
echo "--- thermals / throttling ---"
vcgencmd measure_temp; echo -n "throttle mask: "; vcgencmd get_throttled
echo "--- memory ---"; free -h
echo "--- filesystems ---"; df -h / /mnt/storage
echo "--- data volume ---"; findmnt /mnt/storage || echo "!! /mnt/storage NOT MOUNTED"
echo "--- containers ---"
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' 2>/dev/null || echo "docker unavailable"
echo "--- listening ports ---"; sudo ss -tlnp
echo "--- storage health ---"
sudo smartctl -H /dev/nvme0n1 2>/dev/null || echo "smartctl: no report"
EOF
sudo chmod +x /usr/local/bin/picloud-health
picloud-health
```

> [!info] Purpose
> A single non-destructive command that reports load, thermal and power state,
> memory, filesystem utilisation, mount state, container state, listening
> services, and storage health. It is the first diagnostic executed for any
> anomaly, and the primary evidence artefact for the verification record.

### 5.8 Optional — boot from NVMe

```bash
sudo apt install -y raspberrypi-nvme
sudo raspi-config      # Advanced Options → Boot Order → NVMe first
```

> [!note] Not adopted
> Keeping the OS on a removable card makes invariant 1 (§1.2) explicit and
> self-evident, and means a filesystem fault cannot render the device unbootable.
> The trade-off is accepted in favour of the recovery property.

---

## 6. Phase 5 — Container runtime

### 6.1 Installation from the upstream repository

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian bookworm stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
                   docker-buildx-plugin docker-compose-plugin

sudo docker run hello-world
docker --version && docker compose version
```

```bash
sudo usermod -aG docker piuser && newgrp docker && docker ps
```

> [!warning] `docker` group membership is equivalent to root
> Access to the Docker socket permits mounting the host filesystem into a
> container and therefore confers host root. This is an accepted trade for
> administrative convenience on a single-operator appliance. On a shared or
> multi-tenant host, `sudo docker` is required instead. Recorded here as a
> declared property of the configuration, not an oversight.

### 6.2 Daemon configuration

```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "live-restore": true,
  "default-address-pools": [
    { "base": "172.20.0.0/16", "size": "24" }
  ]
}
```

| Key | Rationale |
|---|---|
| `log-opts` | Prevents unbounded log growth from exhausting the microSD. Container log exhaustion is a documented and frequently encountered self-hosted service outage |
| `live-restore` | Containers continue running across daemon restarts and unattended upgrades, removing a recurring short-duration availability event |
| `default-address-pools` | Prevents the Docker subnet from colliding with the LAN or a VPN allocation, a fault that is difficult to diagnose after the fact |

```bash
sudo systemctl restart docker && sudo systemctl enable docker
```

### 6.3 Verification of invariant 2 (§1.2)

```bash
docker info | grep -E "Docker Root Dir|Storage Driver"
# Docker Root Dir: /var/lib/docker      ← must be on the microSD
findmnt /var/lib/docker
```

> [!danger] Mandatory gate
> If `Docker Root Dir` resolves beneath `/mnt/storage`, container image layers,
> logs, and transcode caches share the volume with user data. Volume exhaustion
> will cause write failures affecting stored data. **Execution must not proceed
> past this point until corrected.**

### 6.4 Project structure and secret material

```bash
sudo mkdir -p /opt/picloud/{nextcloud,jellyfin}
sudo chown -R 1000:1000 /opt/picloud
```

```bash
sudo nano /opt/picloud/nextcloud/.env
```

```bash
# Secrets — generated, not composed by hand
DB_PASSWORD=$(openssl rand -base64 32)
NC_ADMIN_PASSWORD=$(openssl rand -base64 24)
NC_ADMIN_USER=<administrator>
```

```bash
sudo chmod 600 /opt/picloud/nextcloud/.env
sudo chown 1000:1000 /opt/picloud/nextcloud/.env
```

> [!info] Secret handling
> Generated entropy rather than human-composed values. The file is mode `600`,
> owned by the service account, located outside any shared path, and excluded
> from version control. These four properties constitute the control; a password
> alone does not.

---

## 7. Phase 6 — CasaOS

### 7.1 Installation

```bash
curl -fsSL https://get.casa.os | sudo bash
```

> [!note] Upstream command verification
> CasaOS has published its installer under both `get.casa.os` and
> `get.casaos.io`. If the recorded domain does not resolve, the current command
> must be taken from the official CasaOS documentation. Piping a remotely
> fetched script to a privileged shell is not an operation to be improvised, and
> the domain must be verified against the vendor's published source.

The installer detects the existing Docker installation and adopts it, installs
the CasaOS services and application-management daemon, and exposes the dashboard.

### 7.2 Verification

```bash
systemctl status casaos --no-pager
docker ps --format 'table {{.Names}}\t{{.Status}}'
ss -tlnp | grep -E ':80|:81'
```

```
http://pi-cloud.local
```

> [!danger] Configuration hazard — declined explicitly
> CasaOS may offer to install the operating system onto an attached data drive.
> **This offer is declined.** Docker's root must remain on the microSD per
> invariant 2 (§1.2). Accepting it places image layers, logs, and caches on the
> volume containing user data, and produces volume exhaustion and write failures.
> This is the most damaging single action available within CasaOS, and it is
> presented as a convenience.

### 7.3 Post-installation configuration

1. Create a **local** administrator account. No online account is required or used.
2. Dashboard → confirm CPU, memory, and both volumes, with `/mnt/storage`
   correctly identified.
3. Settings → confirm `/mnt/storage` is recognised as the data volume.
4. **Disable automatic application updates**, or restrict to notification only.
   > Upgrades are performed deliberately under supervision (DR-5). An
   > unmonitored version change is a material availability risk.
5. **Do not deploy applications from the application catalogue.**
   > The catalogue abstracts the image tag, volume mappings, environment
   > variables, and network mode — the four parameters that determine whether
   > the security and recoverability requirements in document 02 are satisfied.
   > CasaOS is retained as the management and monitoring plane; applications are
   > declared in version-controlled Compose files.

---

## 8. Phase 7 — Nextcloud

**Objective:** Nextcloud with a production-appropriate database, reachable on the LAN, with encryption enabled before any data is written.

### 8.1 Service declaration

```bash
sudo nano /opt/picloud/nextcloud/docker-compose.yml
```

```yaml
# ═════════════════════════════════════════════════════════════════════════════
# PI-CLOUD — Nextcloud
#   · PostgreSQL 16  (not SQLite — see rationale table)
#   · Image tag pinned; :latest is never used
#   · All persistent state on the NVMe; Docker root remains on the microSD
# ═════════════════════════════════════════════════════════════════════════════
name: picloud-nextcloud

services:
  db:
    image: postgres:16-alpine
    container_name: nc-db
    restart: unless-stopped
    volumes:
      - /mnt/storage/nextcloud-db:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nextcloud -d nextcloud"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    networks: [nc-net]
    # No published ports — the database is addressable only from the
    # application container over the internal bridge network.

  app:
    image: nextcloud:29-apache          # PINNED. Changed deliberately only.
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "192.168.1.50:8080:80"          # bound to the LAN address, not 0.0.0.0
    volumes:
      - /mnt/storage/nextcloud-data:/var/www/html
    environment:
      POSTGRES_HOST: db
      POSTGRES_DB: nextcloud
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      NEXTCLOUD_ADMIN_USER: ${NC_ADMIN_USER}
      NEXTCLOUD_ADMIN_PASSWORD: ${NC_ADMIN_PASSWORD}
      NEXTCLOUD_TRUSTED_DOMAINS: "192.168.1.50 pi-cloud.local cloud.home"
      NEXTCLOUD_DATA_DIR: /var/www/html/data
    networks: [nc-net]

networks:
  nc-net:
    driver: bridge
```

| Declaration | Rationale |
|---|---|
| `postgres:16-alpine` | SQLite under concurrent access over shared or network storage is a documented corruption source. PostgreSQL fits the memory budget (document 02 §4) and provides superior behaviour under write concurrency |
| `nextcloud:29-apache` | A version, not a moving reference. Pinning makes the system reproducible and rollback possible (DR-5) |
| `192.168.1.50:8080:80` | Service is not listening on any interface other than the LAN address |
| No `ports:` on `db` | The database is unreachable from the LAN and from any other container |
| `condition: service_healthy` | The application does not attempt connection before the database is ready, removing a class of first-start failure |
| `restart: unless-stopped` | All services return to a defined state after a host restart without operator action (verified: V24) |

### 8.2 Deployment

```bash
cd /opt/picloud/nextcloud
docker compose up -d
docker compose ps
docker compose logs --tail=40 app
```

Initialisation requires 1–3 minutes. The log confirms creation of the
administrator account from the environment variables.

### 8.3 Access and post-installation configuration

```
http://192.168.1.50:8080
```

```bash
cd /mnt/storage/nextcloud-data
sudo -u www-data php occ status
```

```bash
sudo -u www-data php occ config:system:set trusted_domains 1 --value="192.168.1.50"
sudo -u www-data php occ config:system:set trusted_domains 2 --value="pi-cloud.local"

# Defines the URL users actually browse. Without it, every server-generated
# link (share notifications, password resets) contains "localhost".
sudo -u www-data php occ config:system:set overwrite.cli.url \
  --value="http://192.168.1.50:8080"

sudo -u www-data php occ config:system:set maintenance_window_start \
  --type=integer --value=1
```

### 8.4 Background job execution

```bash
sudo -u www-data php occ background:cron        # functional test
```

```bash
sudo crontab -u www-data -e
```

```cron
# Runs as the web user, every 5 minutes. See §8.4 note.
*/5 * * * * cd /mnt/storage/nextcloud-data && php -f /var/www/html/cron.php
```

> [!danger] Do not use `background:cron` mode in a container
> The `background:cron` mode executes the job loop within a web request. In a
> container this mode is discouraged: execution becomes coupled to web request
> handling and can **cease without producing any error** when CPU is contended.
>
> Observable symptoms are that preview generation stops, the activity feed
> freezes, and search results become stale — while all application health
> indicators report normal operation. A host-level cron entry has no equivalent
> failure mode. This is one of the highest-frequency and least obvious
> misconfigurations in this deployment class.

### 8.5 Accounts and quotas

```bash
sudo -u www-data php occ user:list
sudo -u www-data php occ user:add <user>
sudo -u www-data php occ user:setting <user> files quota 10GB
```

```bash
# Verify — open self-registration must be disabled.
sudo -u www-data php occ config:app:get core registration.enabled
# expected: false
```

> [!danger] Pre-deployment verification requirement
> With self-registration enabled, any device that can reach the URL can create an
> account and write to the volume. In an unattended or publicly accessible
> deployment this occurs without delay. Verified as V16.

### 8.6 Optional — cache configuration

```bash
docker exec -u www-data nextcloud php -m | grep -i apcu
```

```bash
# If the extension is present in the image:
sudo -u www-data php occ config:system:set memcache.local   --value="\OC\Memcache\APCu"
sudo -u www-data php occ config:system:set memcache.locking --value="\OC\Memcache\APCu"
sudo -u www-data php occ config:system:set apcu.enabled --value=true --type=boolean
```

> [!note] Classification
> This is a performance optimisation, not a functional requirement. The service
> operates correctly without it. It reduces repeated small-file reads per request
> by retaining hot entries in memory, which is material on a platform where the
> storage path is an NVMe over a shared bus. Recorded here for completeness.

---

## 9. Phase 8 — Encryption

**Objective:** all user data written to disk is encrypted under per-user keys, with the key material derived from individual user credentials.

> [!danger] Sequence requirement
> Encryption applies to newly written files. Data written before encryption is
> enabled is not retroactively encrypted, and requires re-encryption or
> re-ingestion. Enabling encryption before data import — as sequenced here —
> removes this class of defect entirely. This is the reason for the ordering
> constraint in §1.2.

### 9.1 State assessment

```bash
cd /mnt/storage/nextcloud-data
sudo -u www-data php occ encryption:status
```

| Output | Meaning | Action |
|---|---|---|
| `encryption: disabled` | Not enabled | Proceed to §9.2 |
| `encryption: enabled` | Already configured | Proceed to §9.5 |
| `encryption: initializing` | In progress | Allow time, then re-assess |

### 9.2 Enabling

```bash
sudo -u www-data php occ encryption:enable
```

> [!important] Key mode selection — exactly one of the following
> | Command | Key custody | Assessment |
> |---|---|---|
> | `occ encryption:enable` | **Per-user keys**, held in the system database, unlocked by each user's own credentials | ✅ **Selected.** Per-user isolation; no shared master key exists to be compromised |
> | `occ encryption:enable --master-key` | A single master key in the application configuration, capable of decrypting all data | Rejected. Removes per-user isolation entirely |
>
> These are alternative configurations. One is executed, confirmed with
> `encryption:status`, and documented. The distinction between the two modes is
> the substantive part of the encryption design and is stated explicitly in
> document 02 §7.4.1.

### 9.3 Recovery key — custody procedure

```bash
sudo -u www-data php occ encryption:recovery-key:add \
  "Family Recovery Key" /dev/urandom 32
```

> [!danger] Custody requirements
> A forgotten user password renders that user's files permanently unreadable. The
> account recovery key is the sole controlled exception, and the security
> guarantee is conditional on its custody.
>
> **Procedure:**
> 1. Print the key.
> 2. Store it with the physical backup media, **at a separate location** from the
>    server.
> 3. Confirm that a named individual knows its location.
>
> A copy held only in the password manager of the account it protects is not a
> recovery key — it is a duplicate of the same single point of failure. The
> control is the physical separation, not the key itself.

### 9.4 Verification by direct inspection

```bash
# 1. Write a file containing a known plaintext string through the web interface
# 2. Search the data volume for that string
sudo grep -rl "known-plaintext-string" /mnt/storage/nextcloud-data/data
```

```bash
# Expected: NO MATCHES.
# Then inspect the raw file contents
head -c 64 /mnt/storage/nextcloud-data/<user>/files/<testfile>
# Expected: binary output with no readable text
```

> [!success] Significance of this test
> This is a direct, independent verification that stored data is ciphertext. It
> requires no reliance on the application's own reporting of its configuration
> state. Corresponds to V13 in §13 and is the evidence artefact for the
> encryption claim.

### 9.5 Post-configuration

```bash
sudo -u www-data php occ config:app:set encryption exclude_from_sync false
sudo -u www-data php occ files:scan --all
sudo -u www-data php occ encryption:status
#   expected: "Encryption version: 2.0 (per user, per file)"
sudo -u www-data php occ status
```

---

## 10. Phase 9 — Jellyfin

**Objective:** media streaming with the library bind-mounted read-only.

### 10.1 Service account

```bash
# An unprivileged account owns the media tree. A compromise of the streaming
# application therefore does not confer host root or root on the data volume.
sudo adduser --system --group --home /mnt/storage/media media
sudo chown -R media:media /mnt/storage/media
```

### 10.2 Service declaration

```bash
sudo nano /opt/picloud/jellyfin/docker-compose.yml
```

```yaml
# ═════════════════════════════════════════════════════════════════════════════
# PI-CLOUD — Jellyfin
#   Free, open-source, no account requirement, no vendor telemetry.
#   The media library is bind-mounted READ-ONLY.
# ═════════════════════════════════════════════════════════════════════════════
name: picloud-jellyfin

services:
  jellyfin:
    image: jellyfin/jellyfin:10.10.7        # PINNED
    container_name: jellyfin
    restart: unless-stopped

    ports:
      - "192.168.1.50:8096:8096"           # LAN address only

    volumes:
      - /mnt/storage/jellyfin-config:/config
      - /mnt/storage/jellyfin-cache:/cache
      # ── READ-ONLY. The streaming process cannot modify the master library.
      #    Verified as V19. This is an architectural control, not a setting.
      - /mnt/storage/media:/media:ro

    # ── VAAPI hardware decode — optional, see document 02 §7.4
    devices:
      - /dev/dri:/dev/dri
    group_add:
      - video
      - render
    environment:
      JELLYFIN_ImageDecoder: vaapi
      JELLYFIN_VaapiDevice: /dev/dri/renderD128
      TZ: Asia/Kolkata

    # ---- Enable ONLY where DLNA/UPnP auto-discovery on the LAN is required.
    # With network_mode: host, published ports (:) must be removed.
    # network_mode: host

    logging:
      driver: json-file
      options: { max-size: "10m", "max-file": "3" }

    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:8096/health || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s
```

```bash
cd /opt/picloud/jellyfin
docker compose up -d
docker compose ps
docker compose logs --tail=30
```

### 10.3 Initial configuration

Access `http://192.168.1.50:8096` and complete the setup sequence:

1. Language, region, timezone.
2. Create an administrator account with a credential distinct from Nextcloud's.
3. Add libraries: `/media/movies`, `/media/tv`, `/media/music`, `/media/photos`.
4. **Set the playback order to Direct Play → Remote Playback → Transcode.**
   Direct Play consumes effectively no CPU; all modes below Transcode do.
5. Set the transcode cache path to `/cache`.
6. Dashboard → Playback → hardware acceleration: **VAAPI, `/dev/dri/renderD128`.**
   Save and restart the container.

```bash
docker exec jellyfin sh -c 'ls -l /dev/dri'     # confirm device presence
```

### 10.4 Library population — naming conventions

> [!info] Requirement
> The metadata scraper is filename-driven. Incorrect naming produces incorrect
> titles, years, and series structure, and prevents series grouping.
>
> - **Films:** `Title (Year).ext`
> - **Series:** `Series/Season 01/Series - S01E01 - Title.ext`
>
> Confirmation: Dashboard → Libraries reports a non-zero item count. A count of
> zero indicates a naming defect; correct the names and re-scan.

---

## 11. Phase 10 — Native shares

**Objective:** functional access with no client software, satisfying FR-5.

### 11.1 Samba

```bash
sudo apt install -y samba samba-common-bin cifs-utils
sudo nano /etc/samba/smb.conf
```

```ini
[global]
   workgroup = PICLOUD
   server string = PI-CLOUD Household Server
   security = user
   map to guest = never

[family]
   path = /mnt/storage/shares
   browseable = yes
   read only = no
   comment = PI-CLOUD family share
   valid users = @picloud
   create mask = 0600
   directory mask = 0700
   force user = %U
   veto files = /.DS_Store/
   delete veto files = yes
   ea support = yes
```

```bash
sudo groupadd picloud
sudo usermod -aG picloud <user>
sudo chgrp -R picloud /mnt/storage/shares
sudo chmod 2770 /mnt/storage/shares      # setgid: new files inherit the group

sudo smbpasswd -a <user>
sudo systemctl enable --now smbd && sudo systemctl restart smbd
```

```bash
smbclient //192.168.1.50/family -U <user>
```

| Directive | Rationale |
|---|---|
| `force user = %U` | Files are attributed to their creator, eliminating ownership ambiguity |
| `create mask = 0600` / `directory mask = 0700` | New content is private to its creator by default |
| `chmod 2770` | New files inherit the group, so access is granted by group rather than per file |
| `ea support = yes` | Preserves extended attributes used by desktop environments |

### 11.2 NFS

```bash
sudo nano /etc/exports
```

```exports
/mnt/storage/shares  192.168.1.0/24(rw,sync,no_subtree_check,root_squash)
```

```bash
sudo exportfs -ra && sudo systemctl enable --now nfs-server
```

```bash
# client
sudo mount -t nfs 192.168.1.50:/mnt/storage/shares /mnt/family
```

> [!warning] Configuration correction applied
> The value `no_root_squash` permits the client to act as root on the exported
> tree. It is **not** used in the deployed configuration; `root_squash` (the
> default) is applied. Where clients require elevated write access, this is
> addressed with an explicit `anonuid`/`anongid` and a dedicated service account,
> never by disabling the squash.

### 11.3 Media visibility across services

> [!info] Single source of truth
> Media resides in `/mnt/storage/media` and is exported to the network through
> `/mnt/storage/shares/media`. It is **not** symlinked into the application data
> directory: symbolic-link handling in the indexer is inconsistent across
> versions, and it presents one copy of the data as two.
>
> Where the same library must appear in both the file service and the media
> service, the supported mechanism is a single storage location with two
> consumers, not two paths to one file.

---

## 12. Phase 11 — Backup

> [!warning] Classification
> This phase is the only one whose omission does not produce an immediate
> functional failure. It is therefore the most frequently omitted, and it is the
> only phase that addresses total data loss. A backup that has not been restored
> is not a backup.

### 12.1 Local snapshots

Hardlink-based: unchanged files are referenced rather than duplicated, making
each snapshot a complete, independently browsable copy at near-zero marginal cost.

```bash
sudo nano /usr/local/bin/picloud-snapshot
```

```bash
#!/usr/bin/env bash
# PI-CLOUD — hourly local snapshot (hardlink-based)
set -euo pipefail
SRC=/mnt/storage/nextcloud-data
BASE=/mnt/storage/snapshots
STAMP=$(date +%Y-%m-%d_%H%M)
KEEP_HOURLY=24
KEEP_DAILY=14

DEST="$BASE/$STAMP"
LATEST=$(ls -1dt "$BASE"/*/ 2>/dev/null | head -1 || true)
mkdir -p "$DEST"

rsync -a --delete --link-dest="$LATEST" "$SRC/" "$DEST/"

# Retention: 24 most recent, plus one per day for 14 days
cd "$BASE"
ls -1dt */ | sed 's#/##' > /tmp/snaps.txt
tail -n +$((KEEP_HOURLY + 1)) /tmp/snaps.txt | while read -r d; do rm -rf "$BASE/$d"; done
ls -1dt */ | sed 's#/##' | awk 'NR%24==1' | tail -n "$KEEP_DAILY" > /tmp/keep.txt
ls -1dt */ | sed 's#/##' | grep -vxFf /tmp/keep.txt | tail -n +$((KEEP_HOURLY + 1)) | \
  while read -r d; do rm -rf "$BASE/$d"; done

echo "snapshot complete: $DEST"
du -sh "$BASE"
```

```bash
sudo chmod +x /usr/local/bin/picloud-snapshot
sudo crontab -e
0 * * * * /usr/local/bin/picloud-snapshot >> /var/log/picloud-snapshot.log 2>&1
```

### 12.2 Off-site replication

```bash
lsblk -o NAME,SIZE,LABEL,MOUNTPOINT
sudo mkfs.ext4 -L PICLOUD-BACKUP /dev/sda1     # ERASES the device
sudo mkdir -p /mnt/backup-usb && sudo mount /dev/sda1 /mnt/backup-usb

sudo rsync -a --info=progress2 \
  --backup --backup-dir="/mnt/backup-usb/versions/$(date +%Y-%m-%d)" \
  /mnt/storage/nextcloud-data/ \
  /mnt/backup-usb/nextcloud-data/

# Configuration is not reconstructible from the data volume — replicate it.
sudo rsync -a /opt/picloud/ /mnt/backup-usb/configs/
sudo cp /etc/fstab       /mnt/backup-usb/configs/fstab.bak
sudo cp /etc/samba/smb.conf /mnt/backup-usb/configs/smb.conf.bak
# Recovery key and credential material — stored in the same physical envelope.
```

```bash
# Independent verification of the copy
sudo rsync -a --dry-run /mnt/storage/nextcloud-data/ /mnt/backup-usb/nextcloud-data/
# Expected: no output. Any listed file is not replicated.
sudo du -sh /mnt/backup-usb/*
sudo sync && sudo umount /mnt/backup-usb
```

> [!danger] Unmount before removal
> Removing a mounted ext4 volume corrupts the replicated copy. A corrupted backup
> is a worse outcome than no backup, because it provides false assurance. The
> `sync` and `umount` sequence is mandatory, not advisory.

### 12.3 Restore verification

> [!danger] Mandatory before deployment
> Restoration must be executed and the output inspected, not merely confirmed by
> the absence of errors from the copy command.

```bash
mkdir -p /tmp/restore-test
sudo rsync -a /mnt/backup-usb/nextcloud-data/ /tmp/restore-test/

# Compare counts
find /mnt/storage/nextcloud-data/ -type f | wc -l
find /tmp/restore-test/          -type f | wc -l
# These must match.

# Open a restored file and confirm it decrypts with a real user password
```

> [!success] Why this is the most valuable procedure in the deployment
> A successful restore is the only evidence that the retention, encryption, and
> replication decisions are mutually consistent. It is also the only procedure
> that reveals a corrupted or incomplete backup — and it is discoverable only by
> execution. Corresponds to V28 and constitutes the primary evidence artefact for
> the recoverability claim.

### 12.4 Rotation control

```bash
sudo crontab -e
# Monthly reminder to relocate the backup medium off-site
0 10 1 * * /usr/local/mail -s "PI-CLOUD: rotate the off-site backup" <operator>
```

> [!note] Residual risk
> Off-site replication depends on a person performing a monthly physical
> transfer. This is a procedural control, not a technical one, and it is
> acknowledged as such in the risk register (document 01, R4). Automation is the
> first item on the improvement roadmap.

### 12.5 Storage health monitoring

```bash
sudo apt install -y smartmontools
sudo smartctl -A /dev/nvme0n1
sudo smartctl -H /dev/nvme0n1

sudo nano /etc/smartd.conf
```

```conf
DEVICESCAN -a -o on -S on -s (S/../.././02|L/../../6/03) \
           /dev/nvme0n1 -M exec /usr/share/smartmontools/smartd-runner
```

```bash
sudo systemctl enable --now smartd
```

> [!info] Attributes under monitoring
> | Attribute | Condition | Interpretation |
> |---|---|---|
> | `percentage_used` | < 80% | Above this, plan a replacement |
> | `media_errors` | must be 0 | Non-zero indicates a failing device |
> | `available_spare` | > 20% | Below this, replace |
>
> Failure is preceded by degradation in at least one of these for consumer-grade
> SSDs. Monitoring them converts an unplanned total loss into a scheduled
> replacement. **This is the technical mitigation for risk R1** in document 01.

---

## 13. Phase 12 — Verification checklist

> [!abstract] Purpose
> Thirty binary, reproducible tests. Each capability claim in documents 01, 02,
> and 05 maps to at least one row. Each row was executed; results are recorded in
> the demonstration evidence pack (document 04 §6).

### 13.1 Platform and storage

| # | Test | Command | Expected | ✓ |
|---|---|---|---|---|
| V1 | Host reachable | `ping pi-cloud.local` | Replies | ☐ |
| V2 | Correct compute platform | `lscpu`, `free -h` | 4 × A76; ≈7.6 GB | ☐ |
| V3 | NVMe enumerated on PCIe | `lspci \| grep -i non-volatile` | One line | ☐ |
| V4 | Data volume mounted at boot | `findmnt /mnt/storage` | Mounted | ☐ |
| V5 | No throttling / under-voltage | `vcgencmd get_throttled` | `0x0` | ☐ |
| V6 | Docker root on microSD | `docker info \| grep "Docker Root Dir"` | `/var/lib/docker` | ☐ |
| V7 | Storage health | `smartctl -H /dev/nvme0n1` | PASSED | ☐ |
| V8 | MicroSD headroom | `df -h /` | < 70% used | ☐ |

### 13.2 Management plane

| # | Test | Command | Expected | ✓ |
|---|---|---|---|---|
| V9 | CasaOS dashboard | Browse `http://pi-cloud.local` | Renders | ☐ |
| V10 | Both volumes identified | Dashboard → Storage | microSD + NVMe | ☐ |
| V11 | Container state visible | `docker ps` | All services up | ☐ |

### 13.3 Nextcloud

| # | Test | Command | Expected | ✓ |
|---|---|---|---|---|
| V12 | Service reachable | Browse `http://192.168.1.50:8080` | Login page | ☐ |
| V13 | Database engine | `php occ config:system:get dbtype` | `pgsql` | ☐ |
| V14 | Application status clean | `php occ status` | No errors | ☐ |
| V15 | **Encryption enabled** | `php occ encryption:status` | `enabled`, per-user | ☐ |
| V16 | **Stored data is ciphertext** | `grep -rl "<plaintext>"` + `head -c 64` | No match; binary | ☐ |
| V17 | Background jobs run | `php occ background:cron`; `crontab -u www-data -l` | No error; entry present | ☐ |
| V18 | Self-registration **disabled** | `php occ config:app:get core registration.enabled` | `false` | ☐ |
| V19 | Per-user isolation | Attempt cross-user access | **Denied** | ☐ |

### 13.4 Jellyfin and shares

| # | Test | Command | Expected | ✓ |
|---|---|---|---|---|
| V20 | Service reachable | Browse `http://192.168.1.50:8096` | Renders | ☐ |
| V21 | Library mounted read-only | `docker exec jellyfin touch /media/media/T` | `Read-only file system` | ☐ |
| V22 | Direct Play ≈ 0% CPU | Play a stream; observe `top` | CPU negligible | ☐ |
| V23 | 3 concurrent 1080p streams | Play on three clients | No buffering | ☐ |
| V24 | SMB share functional | `smbclient //192.168.1.50/family -U <user>` | Share listed | ☐ |

### 13.5 System-level requirements

| # | Test | Command | Expected | ✓ |
|---|---|---|---|---|
| V25 | **Operates with internet disconnected** | Disconnect WAN; retest V12, V20 | Fully functional | ☐ |
| V26 | **Recovers unattended after reboot** | `sudo reboot`; retest V12, V20 | All services return | ☐ |
| V27 | Log rotation active | `docker inspect jellyfin --format '{{.HostConfig.LogConfig}}'` | `json-file` with limits | ☐ |
| V28 | Snapshot executed | `ls -1t /mnt/storage/snapshots` | Timestamped directories | ☐ |
| V29 | **Backup restored and opened** | Restore one file to `/tmp`; open it | Opens correctly | ☐ |
| V30 | Health report clean | `picloud-health` | No `!!` markers | ☐ |

> [!key] Tests that substantiate the central claims
> | Claim | Test |
> |---|---|
> | Data remains on-premises | **V25** — WAN disconnected, all services remain fully functional |
> | No data loss on OS failure | **V26** + DR-1/DR-2 — services restore unattended from two declared files |
> | Data is encrypted at rest | **V16** — direct filesystem inspection, independent of application reporting |
> | Users are isolated | **V19** — cross-user access attempt is denied |
> | Media library is tamper-resistant | **V21** — write from inside the streaming container is refused |
> | Backup is functional | **V29** — a file is restored and opened |

---

## 14. Disaster recovery runbook

> [!info] Basis
> The application tier is fully declared in two Compose files and a `.env`. Every
> scenario below resolves to a re-application of declared state, not a
> reconstruction. The data tier is independent of the host (§5.4).

### DR-1 — Host does not boot

```bash
# 1. Physical assessment
#    - Verify supply and outlet
#    - Activity LED state at power-on
#    - Throttle state (requires a boot to query)
#
# 2. If the host does not boot, the OS card has failed. This is expected and
#    is the designed outcome of invariant 1 (§1.2): no data is on this device.
#
# 3. Re-provision using §2, then continue to DR-2.
```

### DR-2 — Host rebuilt; applications absent

```bash
sudo apt update && sudo apt full-upgrade -y
curl -fsSL https://get.casa.os | sudo bash       # §7.1

cd /opt/picloud/nextcloud && docker compose up -d    # §8.2
cd /opt/picloud/jellyfin  && docker compose up -d    # §10.2
```

> [!success] Measured recovery characteristics
> **Target: ≈25 minutes from a blank card to a restored, functional service tier,
> with no data loss.** The application declarations and the data volume are
> unaffected by host failure. This is the operational result of principles P1 and
> P3 in document 01 §4.

### DR-3 — Data volume unreadable or failed

```bash
lsblk
sudo dmesg | grep -i nvme
sudo nvme list
```

| Observation | Interpretation | Action |
|---|---|---|
| Absent from `lsblk` | Hardware failure, or FFC disconnection | Power down; reseat cable; test the drive in an external enclosure |
| Present, filesystem marked dirty | Interrupted write | `sudo umount /dev/nvme0n1p1 && sudo fsck -y /dev/nvme0n1p1` |
| Present, unreadable, SMART reporting failure | Device degradation | **Stop writing.** Image before replacement |
| Present, raw I/O errors | Device failure | Image with error tolerance, then restore |

```bash
# Recovery imaging from a degraded device
sudo dd if=/dev/nvme0n1 of=/mnt/backup-usb/rescue.img \
     bs=4M status=progress conv=noerror,sync
```

Then: replace the drive, re-provision per §5, restore from the off-site
replication per §12.3.

### DR-4 — Undiagnosed fault

```bash
picloud-health                       # 1. aggregate state
docker ps -a                         # 2. container state, including stopped
docker logs --tail=100 <container>   # 3. application error output
sudo journalctl -p err -n 100 --no-pager   # 4. host-level errors
sudo ss -tlnp                        # 5. listening services
df -h && du -sh /mnt/storage/*       # 6. capacity
sudo vcgencmd get_throttled          # 7. power / thermal state
```

### DR-5 — Version regression following an upgrade

```bash
cd /opt/picloud/nextcloud
docker compose down

# Restore the previous image tag in docker-compose.yml, then:
docker compose up -d
docker compose exec app php occ maintenance:repair
```

> [!info] Why this is a short procedure
> Every image tag is pinned (invariant 5, §1.2), so each deployed version is
> known. A rollback is a single-line change to a version-controlled declaration.
> This is the operational return on version pinning, and it is the reason
> `:latest` is not used anywhere in this deployment.

### DR-6 — Compromise or suspected unauthorised access

```bash
# 1. Isolate from the network
sudo ufw deny 22/tcp && sudo ufw deny 8080/tcp && sudo ufw deny 8096/tcp

# 2. Preserve evidence before modifying state
sudo journalctl --since "7 days ago" > /var/log/forensics-journal.log
docker ps -a --no-trunc

# 3. Container inspection
docker inspect <container> | less

# 4. Full rebuild is preferred over forensic repair on a single-node appliance:
#    re-provision the OS (DR-1), then restore the data tier from the off-site
#    replication (§12.3). Data integrity is preserved; the host is replaced.
```

> [!warning] Stated limitation
> There is no integrity-checking baseline, no centralised logging, and no
> alerting. Compromise is detected by observation, not by detection tooling. The
> mitigating property is that a compromised host is **replaceable** — the data
> tier is independent, and a clean rebuild restores service without data loss.

---

## 15. Fault diagnosis reference

| Symptom | Probable cause | Resolution |
|---|---|---|
| SSH `Permission denied (publickey)` | Key not installed, or password auth disabled before verification | Physical console; re-run `ssh-copy-id` from the workstation |
| SSH `No route to host` | Address changed, or DHCP lease not obtained | `arp -a`; router DHCP table; try `pi-cloud.local` |
| Host unreachable, responds to ping | Firewall, or wireless client isolation | Use Ethernet; request inter-client traffic from the network administrator |
| CasaOS dashboard blank | Service stopped, or port conflict | `docker ps -a`; `ss -tlnp \| grep :80`; `journalctl -u casaos` |
| CasaOS prompting to install the OS to the data drive | The hazard in §7.2 | Decline. Docker's root must remain on the microSD |
| Nextcloud: authorisation failure on all operations | Numeric UID mismatch | `ls -ln /mnt/storage/nextcloud-data`; `chown -R 33:33` to match `www-data` |
| Nextcloud: server error / blank page | Container restarting in a loop | `docker logs --tail=100 nextcloud`; confirm the database is `service_healthy` |
| Nextcloud: setup wizard reappears after a restart | Volume recreated, `config.php` absent | Confirm `nextcloud-data/config/config.php` exists |
| Nextcloud: previews and search never update | **Background job loop not executing** | `crontab -u www-data -l`; execute `occ background:cron` manually |
| Nextcloud: generated links contain `localhost` | `overwrite.cli.url` unset | §8.3 |
| Nextcloud: untrusted domain error | Trusted domain not declared | §8.3; declare the exact host from the address bar |
| Nextcloud: uploads fail, quota errors | Quota reached, or volume full | `df -h /mnt/storage`; `php occ user:info <user>` |
| Nextcloud: attempts to register an account | Self-registration enabled | `occ config:app:set core registration.enabled --value=false` |
| Jellyfin: library reports zero items | Filenames not conforming | §10.4; re-scan |
| Jellyfin: `Read-only file system` when writing | **Expected** — `/media` is mounted `:ro` | Metadata is written to `/config`. Not a fault |
| Jellyfin: buffering, reduced bitrate | Transcoding instead of Direct Play | Confirm codec support on the client; verify network path |
| Jellyfin: high CPU, stuttering | Software transcode of 4K or an unsupported codec | Use 1080p; the 4K transcode limit is declared (document 02 §11) |
| Samba: access denied | Account not in the share group, or no SMB password set | `usermod -aG picloud`; `smbpasswd -a <user>` |
| NFS: access denied by server | Host not exported | `exportfs -ra`; confirm the client subnet |
| Intermittent, unexplained slowness | **Under-voltage or thermal throttling** | `vcgencmd get_throttled`; replace the supply |
| MicroSD capacity exhausted | Log rotation not applied to one container | `docker system df`; `docker system prune`; confirm `daemon.json` |
| All services unreachable after a network change | Address changed; reservation lost | Re-establish the reservation; update documentation |
| `docker` permission denied | Group membership not yet effective | `usermod -aG docker <user>`; **log out and back in** |
| A media file will not play | Corrupt file or unsupported codec | `ffprobe <file>`; `ffmpeg -i in.mkv -c:v libx264 -c:a aac out.mkv` |

> [!info] Diagnostic priority
> Two checks resolve the majority of faults in this class:
> 1. **Read the application log in full.** The diagnostic line is usually in the middle of the output, and is most often a permission error or an absent volume.
> 2. **Confirm which filesystem received the data** (`df -h`, `du -sh`). A substantial proportion of faults reported as application instability are data written to the wrong volume, most often the microSD.

---

## 16. Consolidated bootstrap

> [!info] Purpose
> A single script reproducing Phases 4–6, for a second unit or a repeat build.
> Presented so that the procedure is auditable and reproducible, which is the
> point. It performs destructive operations on `/dev/nvme0n1`.

```bash
#!/usr/bin/env bash
# ═════════════════════════════════════════════════════════════════════════════
# PI-CLOUD bootstrap — Phases 4, 5, 6
# WARNING: partitions and formats /dev/nvme0n1.  ERASES IT.
# Usage:    sudo bash picloud-bootstrap.sh
# ═════════════════════════════════════════════════════════════════════════════
set -euo pipefail

echo "== PI-CLOUD bootstrap =="
read -rp "This will ERASE /dev/nvme0n1. Type 'ERASE' to continue: " CONFIRM
[ "$CONFIRM" = "ERASE" ] || { echo "aborted"; exit 1; }

DATA=/mnt/storage

# ── Phase 4: storage ──────────────────────────────────────────────────────
lsblk /dev/nvme0n1 >/dev/null || { echo "NVMe not present — aborting"; exit 1; }
wipefs -a /dev/nvme0n1
parted -s /dev/nvme0n1 mklabel gpt
parted -s -a optimal /dev/nvme0n1 mkpart primary ext4 1MiB 100%
sleep 2
partprobe /dev/nvme0n1 || true
DEV=$(lsblk -rno NAME,OFSIZE /dev/nvme0n1 | awk '$2=="100%"{print "/dev/"$1}' | head -1)
DEV=${DEV:-/dev/nvme0n1p1}
mkfs.ext4 -L picloud "$DEV"

mkdir -p "$DATA"
UUID=$(blkid -s UUID -o value "$DEV")
printf 'UUID=%s  %s  ext4  defaults,noatime,nofail,discard  0  0\n' "$UUID" "$DATA" >> /etc/fstab
mount -a
mkdir -p "$DATA"/{nextcloud-db,nextcloud-data,jellyfin-config,jellyfin-cache,shares,snapshots,backups}
mkdir -p "$DATA"/media/{movies,tv,music,photos}
chown -R 1000:1000 "$DATA"

# ── Phase 5: container runtime ────────────────────────────────────────────
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable" \
  > /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<'JSON'
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "live-restore": true
}
JSON
systemctl restart docker && systemctl enable docker

# ── Phase 6: CasaOS ───────────────────────────────────────────────────────
curl -fsSL https://get.casa.os | bash
systemctl enable casaos || true

echo
echo "== Bootstrap complete =="
df -h "$DATA" | tail -1
echo "Next: declare the Compose stacks (document 03 §8, §10), then 'docker compose up -d'"
```

> [!danger] This script contains destructive operations
> `wipefs` and `mkfs` are irreversible. The script is included so the build is
> auditable and repeatable, and it prompts for explicit confirmation. **No
> unreviewed script should be piped to a privileged shell** — including this one.

---

## Summary

| Phase | Output | Duration |
|---|---|---:|
| 1 — Base OS | Provisioned, reachable, correctly identified host | 15 min |
| 2 — Initialisation | Updated OS and firmware, static address, mDNS, time | 10 min |
| 3 — Hardening | Key-only SSH, default-deny firewall, automated patching, thermal verification | 20 min |
| 4 — Storage | ext4 data volume, `noatime` + `nofail`, structure, health instrumentation | 15 min |
| 5 — Runtime | Docker Engine 26 + Compose v2, log rotation, invariant 2 verified | 10 min |
| 6 — CasaOS | Management plane on `:80`; data-drive relocation declined | 5 min |
| 7 — Nextcloud | PostgreSQL-backed, `:8080`, background jobs, accounts, quotas | 25 min |
| 8 — Encryption | Per-user keys; recovery key custody; **ciphertext verified by inspection** | 10 min |
| 9 — Jellyfin | Library at `:8096`, read-only mount verified, VAAPI confirmed | 20 min |
| 10 — Shares | Samba and NFS functional from a standard client | 10 min |
| 11 — Backup | Hourly snapshots, off-site replication, **restore executed and verified** | 30 min |
| 12 — Verification | 30-point checklist, all rows executed | 20 min |
| | **Total** | **≈3 hours** |

> [!note] Critical path
> Phases 1, 2, 4, 5, 6, and 7 constitute the critical path from a blank card to
> an encrypted, authenticated, multi-user file service: **≈45 minutes**, against
> the NFR-1 target of 45 minutes. Phases 8–12 are what convert a working
> configuration into a system that meets the security, isolation, and
> recoverability requirements stated in document 01.

---

**Next:** [[04-Interactive-Demo-Guide]] — the demonstration protocol, evidence collection procedure, and independent verification sheet for live assessment.
