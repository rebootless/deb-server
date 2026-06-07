# deb-server — Home Lab

![Debian](https://img.shields.io/badge/Debian-13_Trixie-A81D33?style=flat-square&logo=debian&logoColor=white)
![Kernel](https://img.shields.io/badge/Kernel-6.12.74_amd64-333?style=flat-square&logo=linux&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-Oracle_VM-183A61?style=flat-square&logo=virtualbox&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-1.26.3-009639?style=flat-square&logo=nginx&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.4_FPM-777BB4?style=flat-square&logo=php&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-26.1.5-2496ED?style=flat-square&logo=docker&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-latest-F46800?style=flat-square&logo=grafana&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-latest-E6522C?style=flat-square&logo=prometheus&logoColor=white)

Personal home server running as a VirtualBox VM on a local machine. Used for self-hosting, monitoring, and experimenting with server infrastructure.

## Overview

| Property       | Value                             |
|----------------|-----------------------------------|
| **Hostname**   | `deb-server`                      |
| **OS**         | Debian GNU/Linux 13 (Trixie) 13.4 |
| **Kernel**     | Linux 6.12.74 amd64               |
| **Platform**   | VirtualBox (Oracle VM)            |
| **Local IP**   | `192.168.0.101`                   |
| **Domain**     | `deb-server.local`                |

---

## Hardware (VM specs)

| Component  | Details                               |
|------------|---------------------------------------|
| **CPU**    | Intel Xeon E31230 @ 3.20 GHz (1 vCPU) |
| **RAM**    | 1.9 GiB                               |
| **Swap**   | 975 MiB                               |
| **Disk 1** | 20 GB (`/dev/sda`) — OS               |
| **Disk 2** | 20 GB (`/dev/sdb`) — `/var`           |

### Partition layout

```
sda (20 GB)
├── sda1  5.3 GB   /         (OS root, 34% used)
├── sda5  2.1 GB   (ext)
├── sda6  976 MB   [SWAP]
└── sda7  11.7 GB  /srv      (data, 1% used)

sdb (20 GB)        /var      (Docker, logs, packages — 19% used)
```

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                        VirtualBox VM                       │
│                         deb-server                         │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                     Nginx 1.26.3                     │  │
│  │             (reverse proxy + web server)             │  │
│  │                                                      │  │
│  │  deb-server.local/          →  /var/www/             │  │
│  │  deb-server.local/grafana/  →  :3000                 │  │
│  └───────────────────────────┬──────────────────────────┘  │
│                              │                             │
│  ┌───────────────────────────▼──────────────────────────┐  │
│  │                Docker (grafana-stack)                │  │
│  │                                                      │  │
│  │     ┌─────────┐    ┌──────────┐    ┌───────────┐     │  │
│  │     │ Grafana │ ◄──│Prometheus│ ◄──│ Node Exp. │     │  │
│  │     │  :3000  │    │  :9090   │    │           │     │  │
│  │     └─────────┘    └──────────┘    └───────────┘     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│          PHP 8.4-FPM · Fail2ban · Firewalld · SSH          │
└────────────────────────────────────────────────────────────┘
```

## Services

| Service         | Version    | Status    | Description                        |
|-----------------|------------|-----------|------------------------------------|
| **Nginx**       | 1.26.3     | ✅ Active | Web server + reverse proxy         |
| **PHP-FPM**     | 8.4        | ✅ Active | FastCGI PHP processor              |
| **Docker**      | 26.1.5     | ✅ Active | Container runtime                  |
| **Fail2ban**    | —          | ✅ Active | Brute-force protection             |
| **Firewalld**   | —          | ✅ Active | Dynamic firewall (nftables)        |
| **AppArmor**    | —          | ✅ Active | Mandatory access control           |
| **SSH**         | OpenSSH    | ✅ Active | Remote access                      |
| **Avahi**       | —          | ✅ Active | mDNS / `.local` name resolution    |
| **Cron**        | —          | ✅ Active | Scheduled tasks                    |

## Docker Stack — Monitoring

Located at `/opt/grafana-stack/`. Managed with Docker Compose.

| Container         | Image                   | Port   | Description             |
|-------------------|-------------------------|--------|-------------------------|
| `grafana`         | `grafana/grafana`       | 3000   | Dashboards & alerts     |
| `prometheus`      | `prom/prometheus`       | 9090   | Metrics collection      |
| `node-exporter`   | `prom/node-exporter`    | 9100   | System metrics          |

Grafana is accessible at `http://deb-server.local/grafana/` (proxied via Nginx).

### Deploy

```bash
cd /opt/grafana-stack
docker compose up -d
```

## Nginx

Config: [`nginx/sites-available/deb-server.local`](nginx/sites-available/deb-server.local)

- Serves PHP and static files from `/var/www/deb-server.local`
- Proxies `/grafana/` to Grafana container on port 3000
- WebSocket support enabled for Grafana live features
- `server_tokens off` — version not exposed
- TLS 1.2 / 1.3 only (when HTTPS is configured)

## Security

- **Firewalld** — zone-based firewall, Docker networks isolated in `docker` zone
- **Fail2ban** — SSH and Nginx brute-force protection
- **AppArmor** — mandatory access control profiles
- **SSH** — key-based auth recommended; password auth should be disabled

Open ports:

| Port | Protocol | Service        |
|------|----------|----------------|
| 22   | TCP      | SSH            |
| 80   | TCP      | Nginx (HTTP)   |
| 5353 | UDP      | Avahi mDNS     |

## Deployment

This server was set up using scripts from **[VargKernel/shell-toolkit](https://github.com/VargKernel/shell-toolkit)**.

### Order of execution

#### 1. Bootstrap — base system setup
```bash
bash server-bootstrap.sh
```
Installs admin tools (`htop`, `git`, `curl`, etc.), configures sudo, firewalld (SSH allowed), and Fail2ban.

#### 2. Nginx — web server + reverse proxy
```bash
bash deploy-nginx.sh
```
Installs Nginx + PHP 8.4-FPM, creates the virtual host for `deb-server.local`, configures the `/grafana/` reverse proxy block, and sets firewalld rules for HTTP/HTTPS.

#### 3. Grafana stack — monitoring
```bash
bash deploy-grafana.sh
```
Installs Docker, generates `compose.yaml`, Prometheus config, and Grafana provisioning inline. Starts the stack and auto-imports the **Node Exporter Full** dashboard (ID 19937) via the Grafana API.

#### 4. (Optional) Server report
```bash
bash server-report.sh
```
Collects a full system snapshot (hardware, services, Docker, Nginx, firewall, logs) into a `server-report.tar.gz` archive.

## Screenshots

### Node Exporter Full — System Metrics
![Grafana system metrics dashboard](screenshots/grafana-metrics.png)

### Linux Server Dashboard — Overview
![Grafana Linux server dashboard](screenshots/grafana-dashboard.png)

## Repository Structure

```
deb-server/
├── README.md
├── .gitignore
├── screenshots/
│   ├── grafana-metrics.png         # Node Exporter Full — system graphs
│   └── grafana-dashboard.png       # Linux server overview dashboard
├── nginx/
│   └── sites-available/
│       └── deb-server.local        # Nginx virtual host config
└── docker/
    └── grafana-stack/
        ├── compose.yaml            # Docker Compose stack
        ├── prometheus/
        │   └── prometheus.yml      # Prometheus scrape config
        └── grafana/
            └── provisioning/
                ├── datasources/
                │   └── prometheus.yml
                └── dashboards/
                    └── dashboards.yml
```

## Notes

- Secrets (Grafana admin password, etc.) are stored in `/opt/grafana-stack/secrets/` and are **not** tracked in this repo.
- `/var` is on a separate disk (`/dev/sdb`) to prevent logs and Docker data from filling the OS partition.
- `/srv` is reserved for future data/service storage.
