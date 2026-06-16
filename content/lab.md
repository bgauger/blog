---
title: "Lab"
date: 2026-06-16
draft: false
description: "A simple overview of the homelab behind this blog."
hideComments: true
---

This is the homelab behind the articles on this site.

It changes over time, but the basic idea is stable: use a small self-hosted environment to practice infrastructure habits that matter in real operations.

## Core stack

- Proxmox for virtualization
- OpenTofu/Terraform patterns for infrastructure as code
- Ansible for configuration management
- Docker Compose and LXC where they make sense
- Caddy and Cloudflare Tunnel for selected public services
- Hugo, GitHub webhooks, and Obsidian for this blog

## Hardware

Last checked: 2026-06-16.

| Host | Role | CPU | RAM | Boot/local storage | GPU/display |
| --- | --- | --- | --- | --- | --- |
| citadel | Main Proxmox node, storage/GPU-heavy workloads | Intel Xeon E5-2699 v4, 22 cores / 44 threads | 125 GiB | 2x TEAM TM8FP6001T NVMe, 953.9G each | NVIDIA GeForce GTX 1650, ASPEED BMC graphics |
| destiny-ascension | Proxmox application node | AMD Ryzen 3 PRO 2200GE, 4 cores / 4 threads | 14 GiB | Patriot P210 128GB SSD | Integrated Radeon Vega |
| everest | Proxmox infrastructure/networking node | AMD Ryzen 3 PRO 2200GE, 4 cores / 4 threads | 14 GiB | Patriot P210 128GB SSD | Integrated Radeon Vega |
| killimanjaro | Proxmox lightweight services/blog node | AMD Ryzen 3 PRO 2200GE, 4 cores / 4 threads | 14 GiB | Patriot P210 128GB SSD | Integrated Radeon Vega |
| seoul | Proxmox spare/new-services node | AMD Ryzen 3 PRO 2200GE, 4 cores / 4 threads | 14 GiB | Patriot P210 128GB SSD | Integrated Radeon Vega |

All five Proxmox hosts currently see the shared storage targets `normandy`, `tempest`, and `backup`.

## Storage

| Name | What it is used for | Current visible state |
| --- | --- | --- |
| normandy | Durable/shared storage | NFS target visible to Proxmox, about 40 TB total |
| tempest | Fast shared datastore / working set | NFS target visible to Proxmox, about 5.8 TB total |
| backup | Proxmox Backup Server target | Backup target visible to Proxmox, about 3.8 TB total |

Normandy is exposed as a TrueNAS system. From the TrueNAS API it currently reports TrueNAS 13.0-U6.8, 8 virtual CPU cores, and about 32 GiB of memory. The underlying storage hardware is passed through to that VM, so I do not treat the VM summary as the full physical disk inventory.

## Current workload placement

| Host | Notable running guests/services |
| --- | --- |
| citadel | TrueNAS, Servarr, Home Assistant, Project N.O.M.A.D., Technitium secondary, secondary Caddy proxy, Immich |
| destiny-ascension | Portainer, Mealie, RomM, Homepage, ELK, Vaultwarden |
| everest | Technitium, Netboot, Monitoring, primary Caddy proxy |
| killimanjaro | Blog Hugo builder, Blog Caddy, Stirling PDF, Vikunja, Syncthing, FreshRSS, OpenWebUI, Audiobookshelf |
| seoul | GitHub runner |

## What I use it for

- testing self-hosted services
- learning where automation helps and where it gets in the way
- practicing rebuilds and recovery
- documenting operational mistakes before I repeat them
- keeping useful infrastructure close enough to break safely

The lab is not meant to be perfect. It is meant to be understandable, recoverable, and useful.

Related posts:

- [Managing My Proxmox Homelab with OpenTofu](/posts/managing-proxmox-homelab-with-opentofu/)
- [Docker, Podman, and LXC: what I actually use in my homelab](/posts/docker-podman-lxc-what-i-actually-use/)
- [My homelab backup strategy is mostly about distrust](/posts/homelab-backups-what-i-want-from-them/)
