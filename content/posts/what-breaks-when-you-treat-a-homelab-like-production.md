---
title: "What breaks when you treat a homelab like production"
date: 2026-04-28
draft: false
tags:
  - homelab
  - devops
  - operations
  - gitops
  - infrastructure
---

There is a specific kind of homelab delusion where you start saying things like "production-grade" about a rack in your house.

I am guilty of this.

The funny part is that treating a homelab like production does make it better. GitOps helps. Monitoring helps. Backups help. Change control helps. Naming things consistently helps more than I want to admit.

But it also breaks some assumptions, because a homelab is not production. It is missing the thing production usually has: other people.

<!--more-->

## You are the platform team and the user

At work, there is usually some separation between the people who run the platform and the people who consume it. Even if the separation is messy, it exists.

At home, I am both sides of the ticket.

I am the person who wants Jellyfin to work. I am also the person who has to figure out why the reverse proxy is returning a 502. I am the change advisory board, the incident commander, the annoyed customer, and the person who forgot to update DNS.

This creates a strange incentive problem. I can bypass my own process whenever I get impatient.

That is how drift happens.

Production discipline in a homelab is mostly the art of making the right path easier than the lazy path. If updating the repo is painful, I will eventually SSH in and fix the thing by hand. If the playbook is easy, I am more likely to do it properly.

## Change control feels silly until it does not

Nobody wants to open a formal change request to reboot a box in the basement.

Good. Do not do that.

But some version of change control still matters. Maybe it is just a Git commit. Maybe it is a note in Obsidian. Maybe it is a quick plan before touching a Proxmox node that hosts storage. The point is not ceremony. The point is remembering what changed.

Most homelab incidents are self-inflicted. Mine certainly are.

A VLAN tag gets changed. A proxy route points at an old IP. A backup job excludes the one guest that now matters. A service was moved during a migration and the dashboard still points to the previous address.

These are not exotic failures. They are bookkeeping failures.

Change control is bookkeeping with consequences.

## Monitoring can become noise quickly

Production monitoring has teams behind it. A homelab has you, probably trying to drink coffee.

If every warning becomes an alert, you will stop caring. If every failed cron job becomes a crisis, the system is training you to ignore it. This is worse than no monitoring because it adds guilt to the outage.

The monitoring I actually want at home is boring:

- Is the service reachable?
- Is storage filling up?
- Did a backup fail?
- Did a node disappear?
- Is DNS answering the way clients expect?

That is enough to catch most real problems.

Everything else needs to justify its noise budget. A homelab alert should be actionable or silent. Preferably both, depending on whether I am asleep.

## High availability has a cost

High availability sounds great until you are the one maintaining it.

Two DNS servers are better than one, unless they disagree. Two reverse proxies are better than one, unless only one has the updated Caddyfile. Multiple Proxmox nodes are better than one, unless you forget which guests depend on storage from the node you are about to power down.

Redundancy does not remove complexity. It moves it.

That does not mean HA is bad. It means HA needs verification. If failover only works in the diagram, it does not work.

A single well-understood service can be better than a redundant service nobody has tested.

This is the kind of sentence that sounds obvious until you are staring at two systems that are both technically online and jointly wrong.

## Documentation decays unless it is near the work

I used to think documentation failed because I was lazy.

That was only partially true.

Documentation fails when it lives too far away from the work. If I change infrastructure in Proxmox, write config over SSH, and keep notes somewhere else, the notes will drift. There are too many steps between action and record.

GitOps helps because the change and the documentation can sit together. The code says what changed. The commit says why. The playbook says how to recreate it.

Obsidian helps for narrative notes: why I made a decision, what broke during an incident, what I should check next time.

The trick is keeping the note close enough to the work that writing it feels like part of the fix instead of homework after the fix.

## The homelab advantage

A homelab can be better than production in one specific way: it is allowed to teach you.

Production punishes experiments. A homelab invites them. You can break DNS and learn why fallback resolvers matter. You can misconfigure a proxy and learn how TLS validation actually works. You can botch a migration and learn what your backups do not cover.

The value is not that nothing breaks. Things will break. The value is that every break can become a runbook, a playbook, or a better default.

That is why I keep treating the homelab like production, even when the phrase is a little ridiculous.

The goal is not to make the basement look like a data center.

The goal is to build operational habits in a place where failure is annoying instead of career-limiting.
