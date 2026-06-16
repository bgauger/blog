---
title: "My homelab backup strategy is mostly about distrust"
date: 2026-05-15
draft: false
description: "A practical look at homelab backups, restore tests, GitOps, and why green backup jobs are not enough."
tags:
  - homelab
  - backups
  - proxmox
  - disaster-recovery
  - operations
---

Backups are one of those topics where everyone agrees in public and cuts corners in private.

I have done it too. A service is working, the dashboard is green enough, and the backup job is a problem for later. Later arrives wearing a dead SSD, a missing config file, or a container that absolutely did not come back after a reboot.

My backup strategy is not built on optimism. It is built on distrust.

I do not trust single disks. I do not trust myself to remember manual changes. I do not trust a backup I have never restored. I definitely do not trust a screenshot of a configuration page as a disaster recovery plan.

<!--more-->

## What I am trying to protect

A homelab has different kinds of data, and they do not all deserve the same treatment.

Media is bulky and annoying to replace, but some of it can be reacquired. Photos, documents, passwords, notes, and personal records are different. Losing those would be a real problem.

Then there is configuration. This is the category people underestimate.

A VM disk backup is useful. But if I do not know which VLAN the guest used, which reverse proxy route pointed to it, which secrets it needed, or which bind mount fed it data, the restore is only half the story.

So I think about backups in layers:

- application data
- service configuration
- infrastructure definitions
- secrets and credentials
- the knowledge needed to put the pieces back together

That last one is why Git matters. A repository is not a backup by itself, but it preserves intent. Intent is very useful when a machine is dead and the clock is running.

## Proxmox backups cover the obvious failure

For Proxmox guests, scheduled backups are the first line of defense.

They handle the simple case: a VM or LXC breaks, and I need to restore the guest. This is the recovery path everyone imagines when they say, "I have backups."

It is also the easiest path to get wrong.

A backup job that excludes one important guest is still green. A backup that has never been restored is still listed in the UI. A backup stored on the same failure domain as the thing it protects can make you feel safe right up until both disappear together.

The operational habit I care about is checking coverage. Which guests are backed up? Where do those backups live? When did the last job run? Have I restored anything from this backup target recently?

The answer does not need to be fancy. It needs to be known.

## Git backs up the shape of the system

My homelab GitOps repo is part of the backup strategy.

It does not contain the data. It contains the map.

Terraform/OpenTofu defines the guests: names, IDs, nodes, networks, storage, CPU, memory. Ansible defines the configuration: packages, files, services, mounts, users, and application setup.

That means a restore is not just "find the latest disk image and pray." It is closer to:

1. recover or recreate the guest
2. apply the declared infrastructure
3. run the configuration playbook
4. restore the application data
5. verify the service from the outside

That process is slower to design but faster under stress.

The repo also keeps me honest. If I fix something manually and do not put it back into code, I have created undocumented drift. Drift is where future outages breed. Quietly, like mold.

## Restore tests are where the lies show up

A backup system lies politely.

It says the job succeeded. It says the archive exists. It says the snapshot completed. None of that proves the service can come back.

A restore test proves more.

It does not have to be dramatic. Restore a small LXC to a test ID. Boot it. Check the service. Mount the data. Confirm the expected port answers. Delete the test guest when done.

That one exercise teaches more than a month of green check marks.

The first time you do this, you usually find something dumb. A missing secret. A hardcoded hostname. A DNS record you forgot existed. A bind mount that made sense at the time and now looks like a message from a previous civilization.

Good. Finding that during a test is the point.

## The boring checklist

When I look at a service now, I want to know a few things:

- Is the guest backed up?
- Is the application data backed up separately if needed?
- Is the configuration in Git?
- Are secrets stored somewhere recoverable and not in the repo?
- Can I rebuild the host without relying on memory?
- Have I tested the restore path?

If the answer is no, that does not mean the service is bad. It means I know the risk.

That is a better state than pretending every green dashboard means the same thing.

## What I still want to improve

My backup strategy is not finished. It probably never will be.

I want more routine restore testing. I want clearer service-by-service recovery notes. I want better separation between bulky replaceable data and small irreplaceable data. I want fewer places where a service works because of something I did manually during an incident and forgot to codify later.

That is the unglamorous part of running a homelab like infrastructure instead of a pile of computers. The work is never just installing the next app. It is making sure the app can die and come back without turning into archaeology.

Backups are not about storage.

They are about proving future me has options.
