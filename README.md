# 🏠 Homelab Infrastructure

> A code-driven homelab running on [Proxmox VE](https://www.proxmox.com/), provisioned with **Terraform** and configured with **Ansible**: internal DNS, reverse proxying, a media automation stack backed by encrypted remote storage, and a full observability stack (metrics, logs, traces).

---

## 📋 Overview

This repository contains the Infrastructure-as-Code (IaC) and configuration management for a self-hosted homelab. The stack is designed around **repeatability**, **idempotency** and **security**.

**Key principles:**

- **Declarative and version-controlled**: containers come from Terraform, their configuration from Ansible
- **Vault-encrypted secrets**: sensitive values never appear in plaintext
- **Least privilege**: unprivileged LXC containers, no service exposed to the internet
- **Internal DNS**: ad-blocking and recursive DNSSEC resolution on the LAN
- **Encrypted offsite media**: finished media is moved automatically to a remote SFTP storage account (HostBrr) through an `rclone crypt` remote
- **Observable**: metrics, logs and traces for every service, with dashboards and alert rules shipped as code

---

## 🏗️ Architecture

![Architecture diagram](assets/architecture.png)

Everything runs on a single Proxmox node on the **192.168.1.0/24** LAN.

| Host | Hostname | IP | Role | Managed by |
| --- | --- | --- | --- | --- |
| pve01 | `pve-node` | 192.168.1.230 | Proxmox VE hypervisor, `local-lvm` thin pool | manual |
| lxc01 | `docker-ct` | 192.168.1.231 | Docker host: Caddy, LSS media stack, observability stack, storage pool | Terraform + Ansible |
| lxc02 | `pihole-ct` | 192.168.1.232 | Pi-hole + Unbound DNS | Terraform |

**Caddy** (lxc01) terminates HTTPS for every web UI under `*.<host>.internal.endignous.fr`, with certificates issued through a Cloudflare **DNS-01** challenge, so certificates work without exposing any service to the internet.

| URL | Upstream |
| --- | --- |
| `pve.pve01.internal.endignous.fr` | Proxmox web UI `192.168.1.230:8006` |
| `pihole.lxc02.internal.endignous.fr` | Pi-hole admin `192.168.1.232:80` |
| `jellyfin.lxc01.internal.endignous.fr` | Jellyfin `:8096` |
| `jellyseerr.lxc01.internal.endignous.fr` | Jellyseerr `:5055` |
| `radarr` / `sonarr` / `lidarr` `.lxc01.internal.endignous.fr` | `:7878` / `:8989` / `:8686` |
| `prowlarr.lxc01.internal.endignous.fr` | Prowlarr `:9696` |
| `qbit.lxc01.internal.endignous.fr` | qBittorrent `:9090` (through gluetun) |
| `flaresolverr.lxc01.internal.endignous.fr` | FlareSolverr `:8191` |
| `grafana` / `prometheus` / `alloy` / `cadvisor` `.lxc01.internal.endignous.fr` | Observability UIs, reached **by container name** on `caddy_network` (no host port published) |

Navidrome (`:4533`) is reached directly on `192.168.1.231:4533`.

---

## 🗂️ Repository Structure

```
homelab-infrastructure/
├── assets/
│   └── architecture.png          # Architecture diagram (README)
│
├── terraform/                    # Provisions LXC containers on Proxmox VE
│   ├── providers.tf              # bpg/proxmox provider configuration
│   ├── variables.tf              # Input variables
│   ├── lxc01.tf                  # Docker application host
│   ├── lxc02.tf                  # Pi-hole + Unbound DNS host
│   ├── outputs.tf                # IPs, VMIDs, inventory snippet
│   └── terraform.tfvars.example  # Template for local secrets
│
└── ansible/                      # Configures containers post-provisioning
    ├── ansible.cfg               # Inventory, vault password file, SSH key
    ├── inventory/
    │   ├── hosts.yml             # pve, lxc01, lxc02
    │   └── group_vars/lxc01/vault.yml   # Ansible Vault secrets
    ├── playbooks/
    │   ├── site.yml              # Entry point (see Roadmap)
    │   ├── docker.yml            # lxc01: storage, then Docker stacks
    │   └── lxc_hardening.yml     # lxc01: SSH hardening
    └── roles/
        ├── storage/              # lxc01: rclone crypt mount, mergerfs pool, disk guard, offload
        │   └── files/homelab-offload.py
        ├── docker_apps/          # lxc01: Caddy, LSS media stack, observability stack
        │   ├── templates/{caddy,lss,observability}/
        │   └── files/observability/dashboards/   # Grafana dashboards (JSON)
        └── lxc_hardening/        # Admin user, key-only SSH on a custom port
```

---

## 💾 lxc01 Storage

Media is downloaded and seeded on a local disk, then moved to a remote **SFTP storage account (HostBrr)** once it no longer needs to be local. A **mergerfs** pool unions both, so every container keeps the same `/data/...` paths wherever a file physically lives.

```
/data/config/                 # rootfs: app configs + SQLite databases, observability data
/data/local/                  # 512G Proxmox mount point = mergerfs branch 1 (RW)
├── torrents/{tv,movies,music}    # qBittorrent downloads + seeding
└── media/{tv,movies,music}       # Library: hardlinks of torrents
/mnt/remote/                  # rclone mount of media-crypt: = mergerfs branch 2 (NC)
/mnt/pool/data/               # mergerfs union, what the containers use
/var/cache/rclone/            # rclone VFS cache (50G cap)
/etc/rclone/rclone.conf       # rclone config (root only, written by Ansible)
/etc/rclone/hostbrr_ed25519   # SSH key for the storage account (from Vault)
/etc/rclone/known_hosts       # pinned host keys of the storage server (from Vault)
```

**How it works:**

1. qBittorrent downloads into `/data/torrents/...` (on the local disk).
2. Radarr / Sonarr / Lidarr import by **hardlink** into `/data/media/...`. Both names live on the local branch, so the import costs no extra space and the torrent keeps seeding.
3. `homelab-offload` uploads finished items to `media-crypt:media/...`, verifies them, then deletes the torrent and the local copy (see [Automatic offload](#automatic-offload)). The item stays visible at the same path through the pool.
4. Jellyfin and Navidrome read the library through the pool, read-only.

| Component | Setting | Why |
| --- | --- | --- |
| `rclone-media.service` | mounts `media-crypt:` (→ `hostbrr:homelab-crypt` over SFTP, port 53211), `--vfs-cache-mode full` | Encrypted names and content on the storage account, seekable streaming |
| SFTP remote `hostbrr` | SSH key only, pinned host keys (`known_hosts_file`) | No password in the config, and no connection to an impostor server |
| `mergerfs-pool.service` | `create=ff`, branch 2 `=NC` | New files always land on the local disk, never on the remote |
| | `minfreespace=30G` | Writes fail with `ENOSPC` instead of spilling to the remote |
| | `statfs_ignore=nc` | The pool reports local free space only, so the Arrs protect the small disk |
| `docker.service` drop-in | `Requires=` + `After=mergerfs-pool.service` | Containers never bind an empty `/mnt/pool/data` |
| `qbit-diskguard.timer` | every 5 min | Pauses downloads below 40G free, resumes above 80G |
| `homelab-offload.timer` | every 30 min | Moves finished media to the remote (below) |

> **Hardlinks never exist on the remote.** rclone mounts do not support them and mergerfs does not emulate them: they only exist on the local branch, during the seeding window.

### Automatic offload

`homelab-offload` (Python, standard library only) decides what to move from **free space**, **seeding goals** and **torrent popularity**, and works on whole items: a movie folder, a season or an album. Subtitles and extras move with their item, and a season pack moves with its episode torrents.

Popularity is measured per torrent from qBittorrent's upload counter over a 7-day sliding window, plus the swarm reported by the tracker:

- **Popular**: uploads ≥ 25 % of its size per week, ≥ 3 leechers, or leechers while few other seeds exist. May seed up to 90 days.
- **Cold**: nobody downloads from us after 72 h. Offloaded early, even below the ratio target.

| Mode | When | What is offloaded |
| --- | --- | --- |
| normal | `/data/local` free ≥ 150 GB | Library folders no torrent uses anymore, cold items, items past their maximum seeding time (30 days, 90 if popular) |
| pressure | free < 150 GB | Also, in this order until 220 GB is free: items with ratio and seed time met, then neutral, then popular ones |

**Never offloaded:** torrents younger than 24 h, torrents tagged `keep` in qBittorrent, and items still linked to torrents outside the managed categories.

**Safety:** each item is uploaded, **verified** (`rclone cryptcheck`), and only then is its torrent deleted with its files and the local copy removed. Runs are capped per run (`storage_offload_max_gb_per_run`) and never overlap.

```bash
homelab-offload -v                 # show the plan and each torrent's popularity (dry run)
homelab-offload --run              # one real run now
offload-dir "/data/local/media/movies/Title (2000)"   # offload one folder by hand
journalctl -u homelab-offload -f   # follow the timer's runs
```

Settings are the `storage_offload_*` variables in `roles/storage/defaults/main.yml`: categories (`movies`, `tv`, `music`), thresholds, seeding goals, popularity, dry run, verification mode.

### Manual rclone commands

They must use the config written by Ansible, otherwise rclone falls back to `~/.config/rclone/rclone.conf` and may upload **unencrypted**:

```bash
export RCLONE_CONFIG=/etc/rclone/rclone.conf
rclone copy "$DIR" "media-crypt:media/$REL" -P --transfers 4
rclone cryptcheck "$DIR" "media-crypt:media/$REL" --one-way   # or: rclone check --download
rclone rc vfs/refresh "dir=media/$REL" --rc-addr 127.0.0.1:5572
```

> **Back up `/etc/rclone/rclone.conf` and the Vault crypt password and salt off this server.** Without them, the remote data cannot be decrypted.

### Proxmox side

The `/data/local` disk and the `fuse` feature are set by hand on pve01, not by Terraform (bpg replaces the whole container when a disk mount point changes, and `fuse` needs `root@pam`). Both are in `ignore_changes` in `terraform/lxc01.tf`.

```bash
# Back up /data/config first (vzdump of the whole CT does not fit on 'local').
# Stop the LSS stack first so the SQLite databases are consistent.
pct exec 100 -- tar czf /root/lxc01-config.tgz -C /data config
pct pull 100 /root/lxc01-config.tgz /root/lxc01-config-$(date +%F).tgz
# then copy it off pve01, e.g. scp to your workstation

pct shutdown 100
pct config 100                                   # check the current mp0 before deleting it
pct set 100 -delete mp0                          # detach the old /mnt/pool bind mount
pct set 100 -mp0 local-lvm:512,mp=/data/local,mountoptions=discard
pct set 100 -features nesting=1,keyctl=1,fuse=1
pct start 100
```

`local-lvm` is **thin and overcommitted**: the disks promise more than the pool holds. If the pool fills up, the filesystems on it go read-only. On pve01:

- Trim the root disk daily (it is not mounted with `discard`): `0 4 * * * root /usr/sbin/pct fstrim 100 --ignore-mountpoints 1` in `/etc/cron.d/`.
- Keep `Data%` of `pve/data` (`lvs`) below 80 %. The offload and disk guard thresholds only see the filesystem, so size them against the pool's real capacity.

---

## 📈 Observability

A Grafana stack on lxc01 (tag `observability`) collects the three signals:

| Signal | Collected by | Stored in | Sources |
| --- | --- | --- | --- |
| **Metrics** | Prometheus (+ Alloy remote write) | Prometheus, 30 days, capped at 5 GB | node-exporter (lxc01), cAdvisor (containers), Caddy, the stack itself, `homelab-offload` |
| **Logs** | Alloy | Loki, 30 days | Every Docker container, and lxc01's systemd journal (`rclone-media`, `mergerfs-pool`, `homelab-offload`, `qbit-diskguard`…) |
| **Traces** | Caddy (OpenTelemetry) | Tempo, 7 days | One span per request through Caddy |

- **Caddy telemetry:** per-site metrics (requests, status codes, latency, backend health) on an internal-only endpoint, JSON access logs, and traces. A Caddy log line in Grafana links to its trace in Tempo, and back.
- **Offload metrics:** after each run, `homelab-offload` writes its outcome (result, mode, uploaded/freed, torrents by class) to a file published by node-exporter's textfile collector.
- **No exposure:** node-exporter listens only on the `monitoring` bridge gateway; the UIs are reached through Caddy only.

**Dashboards** (provisioned from `files/observability/dashboards/`):

- **Homelab overview**: host, containers, HTTP traffic through Caddy, backend health, error logs, scrape targets, firing alerts.
- **Storage & offload**: free space on `/data/local` against the offload thresholds, each offload run, torrents by class, storage service logs, VPN traffic.

**Alert rules** (Prometheus): target down, `/data/local` low, root disk above 85 %, memory above 90 %, container restarting, Caddy 5xx above 5 %, offload failing, offload stalled.

---

## ⚙️ Tech Stack

| Layer | Tool | Purpose |
| --- | --- | --- |
| **Hypervisor** | Proxmox VE | Bare-metal host running LXC containers |
| **Provisioning** | Terraform + `bpg/proxmox` | Declarative container lifecycle |
| **Configuration** | Ansible | Idempotent post-provisioning setup |
| **Secret Management** | Ansible Vault | API tokens, passwords, VPN and crypt keys |
| **Reverse Proxy** | Caddy (+ `caddy-dns/cloudflare`) | Automatic HTTPS with Cloudflare DNS challenge |
| **DNS** | Pi-hole + Unbound | Network-wide ad-blocking + recursive DNSSEC resolver |
| **Container Runtime** | Docker (Docker-in-LXC) | Isolated application stacks |
| **Storage** | rclone (SFTP + crypt) + mergerfs | Encrypted SFTP storage mount (HostBrr) unioned with a local seeding disk |
| **Torrent VPN** | gluetun (ProtonVPN WireGuard) | Kill-switched network namespace for qBittorrent |
| **Observability** | Prometheus, Loki, Tempo, Grafana, Alloy | Metrics, logs and traces, dashboards and alerts as code |

---

## 🚀 Services

### 🔒 Network & Security

| Service | Host | Description |
| --- | --- | --- |
| **[Caddy](https://caddyserver.com/)** | lxc01 | Reverse proxy with automatic TLS for every web UI, metrics and tracing |
| **[Pi-hole](https://pi-hole.net/)** | lxc02 | Network-wide DNS sinkhole for ad-blocking |
| **[Unbound](https://nlnetlabs.nl/projects/unbound/)** | lxc02 | Recursive, validating DNS resolver (DNSSEC), Pi-hole upstream |

### 🎬 Media Automation (LSS Stack, lxc01)

qBittorrent shares **gluetun**'s network namespace, so all torrent traffic goes through the ProtonVPN WireGuard tunnel. Every other service uses the `lss_network` bridge.

| Service | Port | Description |
| --- | --- | --- |
| **[gluetun](https://github.com/qdm12/gluetun)** | publishes `9090`, `6881` | VPN client (ProtonVPN, WireGuard) for qBittorrent |
| **[qBittorrent](https://www.qbittorrent.org/)** | 9090 | Torrent client, `tv` / `movies` / `music` categories |
| **[Prowlarr](https://github.com/Prowlarr/Prowlarr)** | 9696 | Indexer manager, syncs indexers to the Arrs |
| **[FlareSolverr](https://github.com/FlareSolverr/FlareSolverr)** | 8191 | Cloudflare challenge solver for Prowlarr |
| **[Radarr](https://radarr.video/)** | 7878 | Movie automation |
| **[Sonarr](https://sonarr.tv/)** | 8989 | TV series automation |
| **[Lidarr](https://lidarr.audio/)** | 8686 | Music automation |
| **[Recyclarr](https://recyclarr.dev/)** | n/a | Syncs TRaSH Guides quality profiles to the Arrs |
| **[Jellyseerr](https://github.com/Fallenbagel/jellyseerr)** | 5055 | Media requests → Radarr / Sonarr |
| **[Jellyfin](https://jellyfin.org/)** | 8096 | Media server (movies, TV) |
| **[Navidrome](https://www.navidrome.org/)** | 4533 | Music streaming server |

### 📈 Observability (lxc01)

No port is published on the host: Prometheus, Loki, Tempo and Alloy talk on the `monitoring` bridge, and the UIs go through Caddy.

| Service | Description |
| --- | --- |
| **[Grafana](https://grafana.com/oss/grafana/)** | Dashboards, provisioned datasources (fixed uids) and dashboards |
| **[Prometheus](https://prometheus.io/)** | Metrics and alert rules, remote-write receiver for Alloy |
| **[Loki](https://grafana.com/oss/loki/)** | Logs, with structured metadata for Caddy (status, site, traceID) |
| **[Tempo](https://grafana.com/oss/tempo/)** | Traces from Caddy (OTLP) |
| **[Alloy](https://grafana.com/oss/alloy/)** | Collects Docker and systemd journal logs, scrapes opt-in labelled containers |
| **[cAdvisor](https://github.com/google/cadvisor)** | Per-container CPU, memory, network, disk |
| **[node-exporter](https://github.com/prometheus/node_exporter)** | lxc01 host metrics, textfile collector for host scripts |

---

## 🛠️ Getting Started

### Prerequisites

- A running **Proxmox VE** node (tested on PVE 8.x)
- **Terraform** ≥ 1.5.0
- **Ansible** ≥ 2.15 with the `community.docker` collection
- A **Cloudflare** account (DNS challenge for TLS)
- An **SFTP storage account** (HostBrr: SSH on port 53211, key authentication) and a **ProtonVPN** WireGuard key
- SSH key pair for container access

### 1. Provision containers with Terraform

```bash
cd terraform/

cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your PVE API endpoint, token, IPs, etc.

terraform init
terraform plan
terraform apply
```

Then add the `/data/local` mount point and the `fuse` feature on pve01 (see [Proxmox side](#proxmox-side)).

### 2. Configure containers with Ansible

```bash
cd ansible/

# The vault password is read from ansible/.vault_pass (git-ignored)
ansible-vault edit inventory/group_vars/lxc01/vault.yml
# Add the required secrets (see below)

# lxc01: storage, then Caddy, LSS and observability
ansible-playbook playbooks/docker.yml

# lxc01: SSH hardening
ansible-playbook playbooks/lxc_hardening.yml
```

Run one part only with tags:

| Tag | Scope |
| --- | --- |
| `storage` | rclone mount, mergerfs pool, disk guard, offload |
| `docker_apps` | every Docker stack |
| `caddy` | Caddy only |
| `lss` | the media stack only |
| `observability` | the observability stack, and Caddy (its sites, metrics and tracing) |

Useful extra variables for `docker.yml`:

| Variable | Use |
| --- | --- |
| `-e storage_skip_app_db_check=true` | Fresh install: skip the check that refuses to start the apps when their databases are missing |
| `-e storage_migrate_lss=true` | One-time move of the old `/data/lss` tree into `/data/local`, keeping hardlinks |

---

## 🔐 Required Secrets

Defined in `ansible/inventory/group_vars/lxc01/vault.yml` (Ansible Vault encrypted):

| Variable | Description |
| --- | --- |
| `vault_caddy_email` | ACME registration email |
| `vault_cloudflare_token` | Cloudflare API token (DNS challenge) |
| `vault_lss_wg_private_key` | WireGuard private key for the gluetun tunnel |
| `vault_lss_wg_public_key` | WireGuard peer public key |
| `vault_hostbrr_host` | Storage account SSH hostname |
| `vault_hostbrr_user` | Storage account user (DirectAdmin login) |
| `vault_hostbrr_ssh_private_key` | Private SSH key used by rclone (multi-line block) |
| `vault_hostbrr_host_key` | Server host keys, output of `ssh-keyscan -p 53211 -t ecdsa,ed25519 <host>` (one key per line) |
| `vault_rclone_crypt_password` | rclone crypt password (**back it up offsite**) |
| `vault_rclone_crypt_salt` | rclone crypt salt (**back it up offsite**) |
| `vault_qbittorrent_user` | qBittorrent WebUI user (disk guard and offload) |
| `vault_qbittorrent_password` | qBittorrent WebUI password (disk guard and offload) |
| `vault_grafana_admin_password` | Grafana `admin` password |
| `vault_fichier_api_key` | Legacy 1Fichier API key, only while `storage_fichier_enabled` is true (migration) |

Reserved for the Nextcloud stack (variables defined, stack not deployed yet): `vault_nextcloud_mysql_root_password`, `vault_nextcloud_mysql_password`, `vault_nextcloud_admin_user`, `vault_nextcloud_admin_password`, `vault_nextcloud_smtp_from`, `vault_nextcloud_smtp_user`, `vault_nextcloud_smtp_password`, `vault_onlyoffice_secret`.

Terraform secrets (Proxmox API token, SSH keys) are stored in `terraform/terraform.tfvars`, which is git-ignored. Use `terraform.tfvars.example` as a reference.

---

## 🧭 Roadmap

- [ ] Finish the migration from 1Fichier, then set `storage_fichier_enabled: false` and remove `vault_fichier_api_key` (legacy remotes kept read-only until then)
- [ ] Add `playbooks/common.yml` and `playbooks/pihole.yml`, imported by `site.yml` but not in the repository yet
- [ ] Monitor pve01 and the thin pool from Prometheus (node-exporter on pve01, `observability_extra_scrape_configs`)
- [ ] Pin the observability image versions (`observability_versions`)
- [ ] Nextcloud + OnlyOffice stack (variables already defined in `docker_apps`)

---

## 📄 License

This project is open-sourced for portfolio and educational purposes. Feel free to adapt any part of it for your own homelab.






# 🏠 Homelab Infrastructure

![Last Updated](https://img.shields.io/badge/Last_Updated-July_2026-brightgreen?style=for-the-badge)
![Terraform](https://img.shields.io/badge/terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white)
![Ansible](https://img.shields.io/badge/ansible-%231A1918.svg?style=for-the-badge&logo=ansible&logoColor=white)
![Proxmox](https://img.shields.io/badge/Proxmox-%23E57000.svg?style=for-the-badge&logo=proxmox&logoColor=white)

A fully code-driven homelab running on a single-node Proxmox VE, provisioned with Terraform and configured with Ansible. It covers DNS resolution, reverse proxying, media automation, and self-hosted productivity tools.

## 📋 Context & Overview

This repository contains all the Infrastructure-as-Code (IaC) and configuration management for my self-hosted homelab. 

More than just a deployment script, this project is an **exercise in building resilient architecture under strict constraints**:
* **Hardware Limitations:** Running on a recycled Intel Core i7 laptop (8GB RAM). The battery was purposefully removed to safely allow 24/7 operation and prevent hardware damage (swollen battery risk).
* **Network Constraints:** Deployed on a flat domestic ISP network (`192.168.1.0/24`) with no hardware VLAN support. Network segmentation and security are entirely handled at the software and OS levels.
* **Storage Instability:** Previous reliance on aging hard drives and a faulty DAS, requiring strict statelessness for compute nodes.

### 🎯 Key Design Principles
* **No manual steps:** Everything is declarative and version-controlled. The entire stack can be destroyed and rebuilt from a single command sequence.
* **Vault-encrypted secrets:** Sensitive values (API tokens, DB passwords) never appear in plaintext.
* **Least privilege & Software Security:** Unprivileged LXC containers with minimal surface area, hardened SSH, UFW, and fail2ban to compensate for the lack of hardware network segmentation.
* **Internal DNS:** Split-horizon DNS with ad-blocking and DNSSEC validation.

---

## 🏗️ Architecture & Hardware

*(Insert Architecture Diagram Here)*

**Hardware Setup:**
* **Hypervisor:** Recycled Laptop (Intel i7, 8GB RAM) running Proxmox VE.
* **External Node:** Raspberry Pi 5 (16GB RAM) used for out-of-cluster debugging and side-projects.
* **Network Access:** All containers run on the flat LAN. External secure access is provided via a **Tailscale** zero-trust tunnel. Caddy handles HTTPS termination with automatic TLS via Cloudflare DNS challenge, exposing services under the `*.docker.endignous.fr` wildcard domain.

---

## 🗂️ Repository Structure

```text
homelab/
├── terraform/          # Provisions LXC containers on Proxmox VE
│   ├── providers.tf    # bpg/proxmox provider configuration
│   ├── variables.tf    # All input variables with descriptions
│   ├── lxc01.tf        # Docker application host container
│   ├── lxc02.tf        # Pi-hole + Unbound DNS container
│   ├── outputs.tf      # Exported values (IPs, hostnames…)
│   └── terraform.tfvars.example  # Template for local secrets
│
├── ansible/            # Configures containers post-provisioning
│   ├── ansible.cfg
│   ├── inventory/      # Host definitions
│   ├── group_vars/     # Vars + Ansible Vault secrets per group
│   ├── playbooks/
│   │   ├── site.yml    # Entry point – runs all playbooks
│   │   ├── docker.yml  # Deploys Docker stacks on lxc01
│   │   └── pve.yml     # Configures Proxmox host storage
│   └── roles/
│       ├── docker_apps/  # Manages all Docker Compose stacks
│       └── pve_storage/  # Configures PVE storage pools
│
└── lxc01/              # Static configs managed outside Ansible
    ├── compose/        # Reference Docker Compose files
    └── dns/            # Unbound recursive resolver config

## ⚙️ Tech Stack

| Layer | Tool | Purpose |
| :--- | :--- | :--- |
| **Hypervisor** | Proxmox VE | Bare-metal host running LXC containers |
| **Provisioning** | Terraform + bpg/proxmox | Declarative container lifecycle |
| **Configuration** | Ansible | Idempotent post-provisioning setup |
| **Secret Management** | Ansible Vault | Encrypts API tokens, passwords, WireGuard keys |
| **Reverse Proxy** | Caddy | Automatic HTTPS with Cloudflare DNS challenge |
| **DNS** | Pi-hole + Unbound | Network-wide ad-blocking + recursive DNSSEC resolver |
| **Container Runtime** | Docker (in LXC) | Isolated application stacks |

## 🚀 Services

### 🔒 Network & Security

| Service | Description |
| :--- | :--- |
| **Caddy** | Reverse proxy with automatic TLS, routing all public subdomains |
| **Tailscale** | Zero-Trust Network Access (ZTNA) for remote administration |
| **Pi-hole** | Network-wide DNS sinkhole for ad-blocking |
| **Unbound** | Recursive, validating DNS resolver (DNSSEC) used as Pi-hole upstream |

☁️ Productivity & Media (Arr Stack)

The media stack runs behind a WireGuard VPN tunnel for private torrent traffic.
| Service | Description |
| :--- | :--- |
| **Nextcloud & OnlyOffice** | Self-hosted cloud storage and collaboration suite |
| **Jellyfin / Jellyseerr** | Open-source media server and request management |
| **The *Arr Stack** | Sonarr, Radarr, Lidarr, Bazarr, Prowlarr |
| **qBittorrent** | Torrent client (routed strictly through WireGuard) |

## 💥 Incident Post-Mortem & Lessons Learned

A significant challenge in this project was dealing with hardware failures, specifically a 4-bay DAS (IcyBox) that caused severe infrastructure instability.

+ Incident: The Proxmox cluster experienced multiple kernel panics, and data corruption occurred on the external Raspberry Pi node.
+ Investigation: Debugging kernel traces (dmesg / syslog) revealed massive I/O errors pointing to electrical interference and unstable power delivery from the DAS controller.
+ Mitigation: Attempted to isolate the failing hardware by connecting the DAS to the Raspberry Pi and sharing the storage via Samba. The hardware remained completely unstable.
+ Resolution: Complete decommissioning of the DAS to protect the host compute node. This highlighted the importance of treating storage as an untrusted dependency in low-budget homelabs.

## 🛣️ Roadmap

This infrastructure is a living project. Current plans include:

+ Storage Refactoring: Acquiring a reliable NAS to relaunch the Media Stack and deploy a Bitcoin Full Node.
+ Observability: Implementing Prometheus & Grafana to proactively monitor host metrics and detect hardware anomalies before they cause kernel panics.
+ CI/CD Pipeline: Moving towards a full GitOps strategy (e.g., automated Terraform applies via GitHub Actions).

## 🛠️ Getting Started

### Prerequisites

+ A running Proxmox VE node (tested on PVE 8.x)
+ Terraform ≥ 1.5.0
+ Ansible ≥ 2.15
+ A Cloudflare account (for automatic TLS DNS challenge)
+ SSH key pair for container access

#### 1 — Provision containers with Terraform

```sh
cd terraform/
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your PVE API endpoint, token, IPs, etc.

terraform init
terraform plan
terraform apply
```

#### 2 — Configure containers with Ansible

```sh
cd ansible/
ansible-vault create group_vars/lxc01/vault.yml

# Run the full site playbook
ansible-playbook playbooks/site.yml
```

### 🔐 Required Secrets

The following secrets must be defined in ansible/group_vars/<group>/vault.yml (Ansible Vault encrypted):

| Variable | Description |
| :--- | :--- |
| `vault_caddy_email` | ACME registration email |
| `vault_cloudflare_token` | Cloudflare API token (for DNS challenge) |
| `vault_lss_wg_private_key` | WireGuard private key for the VPN tunnel |
| `vault_lss_wg_public_key` | WireGuard peer public key |
| `vault_nextcloud_mysql_root_password` | MariaDB root password |
| `vault_nextcloud_mysql_password` | Nextcloud DB user password |
| `vault_nextcloud_admin_user` | Nextcloud admin username |
| `vault_nextcloud_admin_password` | Nextcloud admin password |
| `vault_nextcloud_smtp_from` | SMTP sender address (via Proton Mail Bridge) |
| `vault_nextcloud_smtp_user` | SMTP username |
| `vault_nextcloud_smtp_password` | SMTP password |
| `vault_onlyoffice_secret` | OnlyOffice JWT secret |

Terraform secrets (Proxmox API token, SSH keys) are stored in terraform/terraform.tfvars, which is git-ignored. Use terraform.tfvars.example as a reference.

## 📄 License

This project is open-sourced for portfolio and educational purposes. Feel free to adapt any part of it for your own homelab.
