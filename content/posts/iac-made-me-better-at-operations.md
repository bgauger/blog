---
title: "Infrastructure as code made me better at operations"
date: 2026-03-08
draft: false
tags:
  - devops
  - infrastructure-as-code
  - homelab
  - opentofu
  - ansible
  - operations
---

Infrastructure as code did not make me better because I learned more syntax.

Syntax is the cheap part. You can look up the resource name. You can copy the provider block. You can spend an afternoon arguing with YAML indentation and eventually win, although "win" is doing generous work there.

The useful part was that infrastructure as code forced me to think differently. It made me explain what I wanted before I changed it. That sounds small until you compare it to the usual homelab workflow: click around until something works, tell yourself you will document it later, then absolutely do not document it later.

<!--more-->

## Manual work hides decisions

When I build a VM by hand in Proxmox, the decisions disappear as soon as I click away.

Why did I give it two cores? Why that VLAN? Why 8 GB of disk instead of 16? Was nesting enabled because the service needed Docker, or because I copied settings from another container and never checked?

The UI remembers the final state. It does not remember the reasoning.

Git does.

A Terraform or OpenTofu file is not just a provisioning mechanism. It is an argument written down. This guest should exist. It should run on this node. It should have this IP, this amount of memory, this storage backend, this tag, this startup behavior.

That makes bad decisions more visible. Annoyingly visible, sometimes. But visible is better than buried.

## Plans changed how I make changes

The first time `tofu plan` saved me from myself, I understood the point.

I thought I was making a small change. The plan showed that my change would recreate a container I did not want to touch. In the web UI, I might not have noticed until it was too late. In code, the tool had to show me the blast radius before I applied it.

That is a different posture.

Manual operations often feel faster because they skip the thinking step. Infrastructure as code slows you down just enough to ask, "What is this actually going to do?"

That question prevents a lot of stupid.

Not all stupid. I remain technically capable of producing new and interesting failure modes. But fewer of them come from not knowing what changed.

## Idempotence made me less dramatic

Before Ansible, I treated server configuration like a ceremony.

SSH in. Run commands. Edit files. Restart services. Hope the terminal history captured enough detail to reconstruct what happened. If it worked, great. If it failed, now I had two problems: the original issue and whatever partial state I had created while fixing it.

Ansible changed that because the playbook can run again.

That one property removes a lot of tension. If a task fails halfway through, I fix the task and rerun the playbook. If I forget whether a host has a package, I run the playbook. If I want to rebuild a container, I run the playbook.

The goal is not heroics. The goal is repeatability.

Operations gets calmer when repeatability exists.

## Code review applies to infrastructure too

Even when I am the only person touching the homelab, writing changes in Git makes me review myself.

A diff has a way of making nonsense obvious.

Why did this service get hardcoded to an IP instead of using inventory? Why does this playbook install a package but never start the service? Why did I add a firewall exception and not mention it in the commit message?

Those questions are easy to miss in a web UI. They are harder to miss in a diff.

This is one of the most useful career overlaps between homelab work and actual DevOps work. At work, infrastructure changes need review because production is shared and expensive to break. At home, the stakes are lower, but the habits transfer directly.

Small blast radius. Clear diff. Reversible change. Verify after apply.

That is the job.

## Disaster recovery got less theoretical

Everyone says they have backups.

Fewer people have restored from them recently.

Infrastructure as code does not replace backups. It does something different: it backs up intent.

A VM backup can restore a machine. A repository can explain how the machine was supposed to exist in the first place. CPU, memory, IP, VLAN, disks, packages, config files, service units, reverse proxy entries. Those details matter when you are rebuilding under pressure.

The real test is uncomfortable: if this node vanished, what would I need to recreate it?

Before IaC, my answer was a mix of backups, screenshots, memory, and optimism. Now the answer is closer to: clone the repo, restore the data, apply the infrastructure, run the playbooks, verify the endpoints.

Not perfect. Much better.

## It also made me more suspicious

IaC can give you a false sense of control.

The repo says one thing. The live system may say another. Someone can still make an emergency change by hand. A provider can ignore a field. A lifecycle rule can hide drift. A playbook can be green while the service behind it is broken.

So the habit is not "trust the code." The habit is "compare declared state to live state."

That distinction matters. The code is the intended state. The system is the actual state. Operations is the work of keeping those two from drifting too far apart.

## The real benefit

Infrastructure as code made me better because it made my work inspectable.

I can see what changed. I can see why it changed. I can run the same process again. I can hand the repo to someone else and they have a fighting chance of understanding the system.

That is the part that matters.

The tools are OpenTofu, Ansible, Git, Proxmox, and a lot of small shell commands. The tool names will change eventually. The discipline is the useful part: write down the desired state, review the change, apply it deliberately, and verify reality afterward.

It is slower than clicking around.

Until something breaks.
