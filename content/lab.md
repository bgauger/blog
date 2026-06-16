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
