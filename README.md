# Homelab & Infrastructure Documentation

Welcome to my homelab infrastructure repository. This repository serves as a centralized collection of production-tested deployment guides, operational documentation, and automation scripts for my self-hosted environment.

* **AI-Assisted Formatting:** The text and layout of these markdown guides were standardized and refined using AI tools based directly on my personal implementation notes, configuration files, and terminal logs.

## 🛠️ Tech Stack & Architecture Overview

* **Hypervisor:** Proxmox VE (PVE)

* **Storage:** ZFS (POSIX ACLs, Automated Health Checks, Native Scrubbing)

* **Networking & Security:** Tailscale (MagicDNS, Tailscale Serve, OAuth Client Tags), Caddy Reverse Proxy

* **Containers & Virtualization:** Docker, Docker Compose, Unprivileged LXC Containers

* **Hardware Acceleration:** NVIDIA GPU Passthrough (Container Toolkit, CUDA support)

* **Backup & Recovery:** Proxmox Backup Server (PBS), `zpbs-backup` ZFS dataset snapshots

## 📚 Document Index

### 1. Host Infrastructure & Base Setup

* [Proxmox Virtual Environment Setup](./Proxmox%20Virtual%20Environment%20Setup.md) — Base PVE setup, repository configuration, power optimization, and systemd logind tweaks.

* [Tailscale Console & OAuth Key Setup Guide](./Tailscale%20Console%20%26%20OAuth%20Key%20Setup%20Guide.md) — Headless authentication, ACL service tags, and OAuth key management for Docker stacks.

* [Proxmox Backup Server Installation Guide](./Proxmox%20Backup%20Server%20Installation%20Guide.md) — PBS deployment and ZFS pool quota safety configurations.

* [NVIDIA GPU Passthrough & Ollama Setup Guide](./NVIDIA%20GPU%20Passthrough%20%26%20Ollama%20Setup%20Guide%20%28Proxmox%20VE%20%26%20LXC%29.md) — Host and unprivileged LXC kernel driver mapping and NVIDIA Container Toolkit deployment.

### 2. Storage Maintenance & Automated Backups

* [ZFS Automated Maintenance & Health Checks Guide](./ZFS%20Automated%20Maintenance%20%26%20Health%20Checks%20Guide.md) — Daily pool health monitoring script, alerts, and monthly cron scrub tasks.

* [Proxmox ZFS Dataset Backups using zpbs-backup](./Proxmox%20ZFS%20Dataset%20Backups%20using%20zpbs-backup.md) — Automated dataset-level backups to PBS with `.pxarexclude` rules and error notifications.

### 3. Core Services & Workloads

* **Media & Storage:**

  * [Jellyfin Setup Guide with Nextcloud Media & Tailscale](./Jellyfin%20Setup%20Guide%20with%20Nextcloud%20Media%20%26%20Tailscale.md) — Media server integration with Nextcloud directory trees using POSIX ZFS ACLs.

  * [Nextcloud (Custom Docker Compose) with Tailscale Serve Guide](./Nextcloud%20%28Custom%20Docker%20Compose%20%29%20with%20Tailscale%20Serve%20Guide.md) — Nextcloud Apache/MariaDB/Redis deployment with Tailscale reverse proxying.

  * [Nextcloud All-in-One (AIO) Setup with Caddy & Tailscale](./Nextcloud%20All-in-One%20%28AIO%29%20Setup%20with%20Caddy%20%26%20Tailscale.md) — Nextcloud AIO setup leveraging Caddy Layer4 proxying for WebRTC/Talk support.

  * [ownCloud Infinite Scale (oCIS) with Caddy & Tailscale Serve Guide](./ownCloud%20Infinite%20Scale%20%28oCIS%29%20with%20Caddy%20%26%20Tailscale%20Serve%20Guide.md) — High-performance Go-based cloud storage with PosixFS drivers.

* **AI & Search:**

  * [Odysseus & SearXNG Deployment Guide with Tailscale](./Odysseus%20%26%20SearXNG%20Deployment%20Guide%20with%20Tailscale.md) — Self-hosted privacy search engine integrated with AI research software and GPU acceleration.

### 4. Operations & Maintenance

* [LXC Container Updates](./LXC%20container%20updates.md) — Quick reference for system package updates, container updates, and image pruning routines.

## 📄 License

These setup guides and operational scripts are provided open-source under the [MIT License](LICENSE).
