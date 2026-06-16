---
title: Centralizing Docker Management Across My Homelab with Portainer
date: 2026-02-16
draft: false
description: "How Portainer reduced SSH whack-a-mole across Docker hosts in my homelab."
tags:
  - homelab
  - docker
  - portainer
  - proxmox
  - infrastructure
---

It was 2 AM when my media server went down. Half-asleep, I SSH'd into the wrong host, realized my mistake, disconnected, SSH'd into the *right* host, checked Docker logs, saw it was a database lock issue, restarted the container, and stumbled back to bed.

... not really but it could have happened.

The next morning, I deployed Portainer. Now if something breaks at 2 AM, I can fix it from my phone in 30 seconds without remembering which host runs which container.

<!--more-->

## The Problem

I had Docker containers scattered across three different servers in my homelab - my media server, proxy server, and DVD ripping machine. Each server ran its own set of containers, and there was no central visibility.

**My typical troubleshooting workflow:**
1. SSH into media server: `ssh ben@10.0.30.52`
2. Run `docker ps` to check containers
3. Check logs: `docker logs jellyfin`
4. Wrong server. Disconnect.
5. SSH into proxy server: `ssh ben@10.0.30.89`
6. Run `docker ps` again
7. Still wrong server. Disconnect.
8. SSH into third server... finally found it.

This was tedious, error-prone, and impossible to do quickly from my phone.

I needed one dashboard to see and control everything.

## The Solution

I deployed **Portainer** - a web-based Docker management platform - using a centralized architecture with lightweight agents on each Docker host. Now I manage 20+ containers across 3 servers from a single interface.

## Why This Actually Matters

Look, I didn't set up Portainer to pad my resume. I set it up because SSH'ing into three different boxes at 2 AM sucks. But here's the thing—this agent-based architecture where one control plane manages multiple hosts? That's literally how Kubernetes works. It's how Docker Swarm works. It's how every serious container orchestration platform works.

So yeah, I'm solving a homelab problem, but I'm also learning the exact patterns that production teams use to manage hundreds of hosts. When someone asks me in an interview "how do you handle centralized container management?" I'm not going to recite documentation—I built this thing, broke it, fixed it, and use it every day.

## Architecture at a Glance

Instead of running Portainer on each server, I:
1. Created a **dedicated management container** for Portainer (10.0.30.140)
2. Installed **lightweight agents** on each Docker host
3. Connected all agents to the central Portainer instance

This gives me one URL, one login, and visibility into all containers across my homelab.

## Step-by-Step Setup

### 1. Deploy Portainer Container

**Option A: With Terraform/OpenTofu + Ansible (IaC approach)**

If you're following my homelab automation journey, provision an LXC container with Terraform:

```hcl
module "portainer" {
  source = "../modules/proxmox-lxc"

  vmid       = 106
  hostname   = "portainer"
  cores      = 2
  memory     = 2048
  network_ip = "10.0.30.140/24"

  features_nesting = true  # Required for Docker-in-Docker
  ssh_public_keys  = [var.ssh_public_key]
}
```

Then configure it with Ansible:
```bash
cd ansible
ansible-playbook -i inventory/hosts.yml playbooks/setup-portainer.yml
```

My Ansible playbook handles:
- Installing Docker
- Deploying Portainer container
- Configuring firewall rules
- Setting up systemd for auto-start

**Total time:** 5 minutes from `tofu apply` to running dashboard.

**Option B: Manual setup (quick start)**

If you don't have Terraform/Ansible yet, create a VM or LXC container manually, then:

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Run Portainer
docker run -d \
  -p 9443:9443 \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access at `https://YOUR-IP:9443` and create an admin account.

**Total time:** 10 minutes including Docker installation.

### 2. Install Agents on Docker Hosts

On each of my three Docker servers, I ran one command:

```bash
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

This installs a tiny agent (~15MB RAM) that Portainer uses to communicate with each host.

### 3. Connect Everything

I opened Portainer at `https://10.0.30.140:9443` and added my three environments:

1. Click **Environments** → **Add environment**
2. Select **Docker Standalone** → **Agent**
3. Enter each host's IP and port (e.g., `10.0.30.52:9001`)

Done! All my containers are now visible in one dashboard.

## What Went Wrong (And How I Fixed It)

Not everything worked perfectly on the first try. Here's what broke and how I fixed it.

**Problem 1: Agents Wouldn't Connect**

After installing agents on my Docker hosts, they showed as "disconnected" in Portainer. Turns out I had firewall rules blocking port 9001.

**Fix:**
```bash
# On each Docker host
sudo ufw allow 9001/tcp
sudo ufw reload
```

Lesson learned: Always check firewall rules when services can't communicate.

**Problem 2: "Agent already paired" Error**

I had previously run Portainer on each host individually. When I tried to connect agents to the centralized instance, I got "agent already paired with another Portainer instance."

**Fix:**
```bash
# Remove old agent
docker rm -f portainer_agent

# Reinstall fresh agent
docker run -d -p 9001:9001 --name portainer_agent --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

**Problem 3: LXC Container Couldn't Run Docker**

My first attempt failed because I forgot to enable container nesting in Proxmox. Docker requires the ability to create containers within the LXC container.

**Fix in Proxmox UI:**
1. Stop the container
2. Options → Features → Enable "Nesting"
3. Start the container and retry Docker installation

Or in Terraform:
```hcl
features_nesting = true  # Must be enabled for Docker
```

These three issues cost me about 20 minutes of troubleshooting, but now you can skip them entirely.

## What I Can Do Now

From the Portainer web interface, I can:

- **See all containers** across all hosts at a glance
- **Start/stop/restart** any container with one click
- **View real-time logs** without SSH
- **Deploy new containers** using templates or compose files
- **Monitor resource usage** (CPU, RAM, network)
- **Execute commands** inside running containers

## What Actually Changed

The time savings are real. I used to spend 10 minutes SSH'ing around to check on containers. Now it's 30 seconds. Two clicks instead of typing out `ssh ben@10.0.30.52` for the third time because I keep forgetting which host runs what.

Last week qBittorrent was hammering the CPU. Opened Portainer, saw it was seeding 147 torrents at once (oops), adjusted the settings. Done. No SSH session, no `docker ps`, no tab confusion in my terminal.

The 2 AM Jellyfin incident? That one was real. Middle of rewatching The Office, server dies, I grabbed my phone from the nightstand, pulled up Portainer, restarted the container. 90 seconds. Back to watching. My wife didn't even notice.

Also, keeping management tools separate from the apps they manage is just good practice. When I rebuilt my media server last month, Portainer stayed up the whole time. I could see the rebuild happening from the dashboard.

## Network Planning

I organize my homelab with a structured IP scheme:

| IP Range | Purpose | Services |
|----------|---------|----------|
| 10.0.30.50-59 | Media Services | Sonarr, Radarr, etc. |
| 10.0.30.80-89 | Web Services | Nginx, Caddy |
| 10.0.30.140-149 | **Utilities** | **Portainer**, Watchtower |

Portainer lives in the "Utilities" range, keeping it logically separated from applications.

## Why Not Kubernetes?

Yeah, I could've set up K3s. Lot of people do. But I'm managing 20 containers across 3 hosts, not running Netflix. Kubernetes would've been like buying a semi-truck to haul groceries. Awesome truck, totally overkill for what I need.

Docker Swarm? Same problem. I don't need multi-replica orchestration—I need to see what's running and restart things when they break.

I love lazydocker for quick CLI work, but it doesn't solve the multi-host problem. Still gotta SSH into each box individually.

Yacht and Dockge are solid tools, but Portainer's been around longer, has more community support, and frankly just worked when I tried it. I spent 30 minutes testing Portainer and it did exactly what I needed. Didn't feel like shopping around after that.

The killer feature? It doesn't make me change anything. I still use docker-compose. I still manage configs however I want. Portainer's just a window into what's already there.

## Tips and Gotchas

**1. Enable Container Nesting**
If deploying in an LXC container, enable the nesting feature. Docker won't work without it.

**2. Restart Agents When Migrating**
Agents can only pair with one Portainer instance. If you're migrating from an old setup:
```bash
docker restart portainer_agent
```

**3. Use SSH Keys**
Configure SSH keys in your infrastructure-as-code setup from day one. It makes automation so much easier.

**4. Start Small**
I started by connecting just one Docker host to prove the concept. Once it worked, adding the others took minutes.

## Infrastructure as Code

All of this is version-controlled in Git:

```
homelab-gitops/
├── terraform/proxmox/
│   └── portainer.tf          # Infrastructure definition
├── ansible/playbooks/
│   └── setup-portainer.yml   # Configuration automation
└── ansible/inventory/
    └── hosts.yml             # Server inventory
```

If I need to rebuild Portainer, it's just:
```bash
tofu apply && ansible-playbook setup-portainer.yml
```

Everything is documented, reproducible, and backed up in Git.

## Results

**Before:** Managing Docker containers across 3 hosts was tedious and time-consuming.

**After:** One dashboard, one login, complete visibility and control.

- **Deployment time:** 30 minutes
- **Time saved per week:** 2+ hours
- **Containers managed:** 20+ across 3 hosts
- **Lines of code:** ~150 (Terraform + Ansible)

## What's Next

I'm planning to add:

1. **Reverse proxy** - Access via `portainer.homelab.local` instead of IP
2. **Monitoring integration** - Connect to Grafana for historical metrics
3. **Automated backups** - Backup Portainer config to my NAS
4. **More agents** - As I add Docker hosts, just deploy the agent and connect

## Would I Do It Again?

Absolutely. This took maybe an hour to set up (including the mistakes), and I've already saved that time ten times over. No more SSH whack-a-mole when something breaks.

If you've got Docker containers scattered across multiple machines and you're tired of remembering which host runs what, just try it. Spin up Portainer, connect one agent, see if you like it. Worst case you wasted 20 minutes. Best case you never SSH into a Docker host again unless you really want to.

The agent overhead is nothing—15MB RAM per host. I've seen browser tabs use more than that.

And yeah, having this all in Terraform means if I blow something up, I can rebuild it in 5 minutes. That peace of mind is worth it.

**Next in my homelab automation series:**
- Adding monitoring with Prometheus and Grafana
- Automated backups with restic
- Service discovery and reverse proxy automation

---

**What container management tools are you using? Have you tried Portainer? Let me know what's working (or not working) in your homelab.**

## Quick Start Checklist

If you want to replicate this setup:

- [ ] Provision a VM or LXC container for Portainer
- [ ] Install Docker on the Portainer host
- [ ] Deploy Portainer: `docker run -d -p 9443:9443 -p 9000:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest`
- [ ] On each Docker host, install the agent: `docker run -d -p 9001:9001 --restart=always -v /var/run/docker.sock:/var/run/docker.sock portainer/agent:latest`
- [ ] Open Portainer web UI and add each host as an environment
- [ ] Start managing your containers!

---

**My Setup:**
- Proxmox VE cluster (3 nodes)
- OpenTofu (Terraform) for infrastructure
- Ansible for configuration
- Portainer CE (free, open-source)

## Resources

- [Portainer Documentation](https://docs.portainer.io/)
- [Proxmox LXC Containers](https://pve.proxmox.com/wiki/Linux_Container)
