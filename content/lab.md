---
title: "The lab"
date: 2026-06-16
draft: false
description: "A high-level map of the homelab this blog is built around: Proxmox, GitOps, self-hosted services, and operational experiments."
---

The lab is where I test infrastructure habits before they matter somewhere expensive.

It changes over time, but the operating model is stable: Proxmox for compute, Git for intent, OpenTofu for provisioning, Ansible for configuration, and self-hosted services for the parts of life I would rather not rent forever.

## Core stack

- **Virtualization:** Proxmox VE
- **Infrastructure as code:** OpenTofu/Terraform patterns
- **Configuration management:** Ansible
- **Ingress:** Caddy and Cloudflare Tunnel where it makes sense
- **Content workflow:** Obsidian, Hugo, GitHub webhooks, and this self-hosted blog
- **Container management:** Docker Compose, Portainer, and LXC depending on the workload

## What I use it for

The lab runs normal self-hosted services, but the more useful part is the practice:

- rebuilding systems from declared state
- recovering from bad changes
- separating live drift from source-of-truth configuration
- testing backup and restore assumptions
- learning which abstractions are worth their complexity

That last one matters. A homelab will happily let you over-engineer yourself into a corner. It is a patient adversary.

## Why write about it

Most infrastructure writing is either too abstract or too polished.

I prefer the version where something broke, the first fix was wrong, the second fix was less wrong, and the final result became a repeatable pattern. That is closer to how operations actually works.

If you want the practical path through the site, start with [Start here](/start-here/). If you want the chronological record, use the [posts archive](/posts/).
