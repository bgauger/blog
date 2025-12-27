---
title: "Welcome to My Homelab Blog"
date: 2025-12-27
draft: false
tags:
  - homelab
  - infrastructure
---

Welcome to my blog! This site is completely self-hosted on my homelab infrastructure, managed entirely through GitOps principles.

<!--more-->

## Homelab Infrastructure

### Core Platform
- **Proxmox VE** - 3-node cluster (mako, hammerhead, citadel)
- **Ugreen NAS** - 5.4 TB all-flash storage for VM/LXC workloads over 2.5Gb network (tempest)
- **TrueNAS** - 56 TB HDD storage for media and backups (normandy)
- **Pi-hole** - Network-wide DNS and ad blocking
- **OpenTofu** - Infrastructure as Code for reproducible deployments
- **Ansible** - Configuration management and automation

### Self-Hosted Services
- **Portainer** - Docker container management across multiple hosts
- **Immich** - Self-hosted photo and video backup (Google Photos alternative)
- **Tandoor** - Recipe management and meal planning
- **Kanboard** - Project management and task tracking
- **Homepage** - Custom dashboard for service access
- **Servarr Stack** - Media automation and management and Jellyfin
- **ARM** - Automated DVD/Blu-ray ripping machine

### Blog Infrastructure
- **Hugo** - Static site generator (this blog!)
- **Caddy** - Reverse proxy with automatic HTTPS
- **Cloudflare Tunnel** - Secure external access without port forwarding
- **GitHub Webhooks** - Automated deployment on every push

## The GitOps Workflow

Every service you see is defined in code, version-controlled, and automatically deployed. When I push changes to my infrastructure repository, Terraform provisions the resources and Ansible configures them. This blog itself automatically rebuilds and deploys whenever I push new content.

## What to Expect

I'll be writing about:

- Homelab infrastructure design and evolution
- Self-hosting practical services
- Infrastructure as Code patterns and practices
- Automation and GitOps workflows
- Container orchestration strategies
- Privacy-focused alternatives to cloud services

Stay tuned for detailed posts about building and maintaining a production-grade homelab!

I will also be updating this post whenever new services are deployed into the environment.
