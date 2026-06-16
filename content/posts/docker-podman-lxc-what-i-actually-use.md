---
title: "Docker, Podman, and LXC: what I actually use in my homelab"
date: 2026-03-24
draft: false
description: "How I decide between Docker Compose, Podman, LXC containers, and full VMs in my Proxmox homelab."
tags:
  - homelab
  - docker
  - podman
  - lxc
  - proxmox
  - containers
---

I tried to make this a clean decision once. Docker for apps, LXC for system services, VMs for anything serious. Nice little diagram. Very responsible.

Then the homelab did what homelabs do. One service wanted Docker because the compose file was maintained by the project. Another ran better as an LXC because it was just nginx and a config file. Something else needed a full VM because it wanted kernel features, device passthrough, or enough isolation that I could sleep after exposing it to the network.

So the real answer is not "Docker vs Podman vs LXC." I use all three. The useful question is: what kind of failure do I want when this thing breaks?

<!--more-->

## The short version

If I am deploying an application stack with multiple containers, I usually reach for Docker Compose.

If I want a Linux service that behaves like a tiny server, I use an LXC container on Proxmox.

If I am experimenting with rootless containers or want something closer to systemd-managed container workflows, Podman is interesting, but it is not my default for the homelab yet.

That is less elegant than a hard rule. It is also more honest.

## Docker: boring in the best way

Docker wins because most self-hosted projects assume it exists.

That matters more than people admit. When a project gives me a compose file, documented volume paths, example environment variables, and an upgrade path, I do not get points for turning that into a personal purity exercise. I want the service running. I want to know how to back it up. I want future me to understand what I did at 11:30 PM.

Docker Compose is especially good for stacks where the services belong together:

- an app container
- a database
- redis or another cache
- a worker
- maybe a reverse proxy sidecar if the project is opinionated

That pattern is everywhere. Mealie, Immich, Portainer, media tools, dashboards, little web apps. Docker is not perfect, but the ecosystem around it is enormous. When something breaks, the error message is probably searchable. That is not glamorous. It is useful.

Where Docker gets annoying is host sprawl. Once you have Docker on several machines, you need to remember which host owns which stack. That is how you end up playing SSH whack-a-mole, which is exactly why I eventually added Portainer.

Docker is my default when the project is already packaged that way and I do not have a strong reason to fight it.

## LXC: the homelab workhorse

LXC containers are one of the best parts of Proxmox.

They are fast. They are light. They boot like services instead of pretending to be full machines. For boring Linux infrastructure, that is exactly what I want.

A DNS server does not need a full VM. Neither does a static web server, a dashboard, a lightweight reverse proxy, or a small internal tool. If the workload is basically "install packages, write config files, expose a port," LXC is usually a better fit than a VM.

The resource difference is not subtle. A tiny Debian LXC with one vCPU and 512 MB of RAM can handle a surprising amount of work. More importantly, it keeps the Proxmox UI readable. I can see a service as a first-class guest, back it up with Proxmox tooling, put it on a VLAN, and manage it with Terraform/OpenTofu and Ansible.

That last part matters. LXC fits the way I want to run the homelab: declared in Git, provisioned by Proxmox, configured by Ansible.

The trap is treating LXC like a VM with fewer calories. It is not. It shares the host kernel. That is fine for many services and wrong for others. If the app needs weird kernel behavior, privileged mounts, Docker-in-Docker gymnastics, or hardware passthrough, I stop and ask whether I am being clever or just cheap.

Usually, if I have to make LXC weird, I should have used a VM.

## Podman: good tool, not my default

I like Podman. I especially like the idea of rootless containers and systemd integration.

In practice, I do not reach for it first in the homelab. Most projects document Docker. Most troubleshooting threads assume Docker. Most compose examples are tested against Docker. Podman can run a lot of that, but "can" is doing work in that sentence.

For a workstation or a controlled server where I am building my own container workflow, Podman makes sense. For a self-hosted app that already ships a Docker Compose file, I usually do not need another translation layer between me and the thing I am trying to run.

That may change. Podman keeps getting better. But my current rule is simple: I use Podman when I specifically want Podman, not when I merely want containers.

## VMs still matter

This is where container debates get silly. Some workloads should be VMs.

Storage appliances are the obvious example. If I am passing through an HBA or letting a system own disks directly, I want VM isolation. Same with workloads that need GPU passthrough, custom kernel behavior, or a security boundary stronger than a container namespace.

A VM costs more RAM and disk. It boots slower. It is less convenient.

That is fine. The point is not to use the lightest possible abstraction. The point is to use the failure boundary that matches the job.

If a container breaks, it can be annoying. If a storage service breaks in the wrong way, it can ruin your week.

## My current decision process

When I deploy something new, I usually ask these questions:

1. Does the project already provide a good Docker Compose setup?
2. Is this basically a Linux service with a config file and a port?
3. Does it need hardware passthrough, special kernel features, or strong isolation?
4. Do I want Proxmox to see it as a guest I can back up and move?
5. Will future me understand this when something breaks?

The answers usually sort themselves out.

Docker Compose is best when the app is already a container stack.

LXC is best when the service acts like a small Linux server.

A VM is best when isolation or hardware ownership matters.

Podman is best when I specifically want rootless/container-native systemd workflows and I am willing to own the differences.

## What I would not do again

I would not put everything in full VMs just because VMs feel cleaner. That wastes resources and makes the cluster heavier than it needs to be.

I also would not shove everything into Docker just because the internet says containers solved deployment. A Docker host is still a host. It still needs patching, backups, monitoring, secrets, and storage that does not disappear when a container gets recreated.

And I would not force LXC to run workloads it clearly dislikes. Privileged containers, nested Docker, strange mounts, and security exceptions can all be valid. They can also be signs that I am trying to win an argument with my infrastructure.

Infrastructure usually wins those arguments eventually.

## The rule I actually follow

Use the boring thing that matches the failure mode.

That is the whole rule.

If Docker Compose gives me a boring deployment, I use Docker. If LXC gives me a boring tiny server, I use LXC. If a VM gives me a boring isolation boundary, I use a VM.

Boring is not a compromise. In a homelab, boring is what lets you spend Saturday doing something other than rebuilding a service you forgot how to configure.
