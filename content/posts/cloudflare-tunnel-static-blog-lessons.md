---
title: "What I learned running a static blog through Cloudflare Tunnel"
date: 2026-06-05
draft: false
tags:
  - hugo
  - cloudflare
  - homelab
  - gitops
  - self-hosting
---

A static blog should be one of the easiest things to host.

That is what I told myself, which is always how these projects begin. A directory of HTML files, a small web server, a domain, and some automation. How hard could that be?

The answer is: not very hard, if you only care about getting a page online. More interesting if you want the whole thing to be rebuildable, private behind the edge, and wired into the same GitOps habits as the rest of the homelab.

<!--more-->

## The design I ended up with

The blog runs as two pieces.

One container builds and serves the Hugo site internally. Another runs Caddy and Cloudflare Tunnel for the public path. External traffic hits Cloudflare, crosses the tunnel, lands on Caddy, and then reaches the static site.

That sounds like too much for a blog until you look at what it avoids.

No port forwarding.

No public inbound rule to the homelab.

No special case outside the rest of the infrastructure repo.

The blog is just another service: declared in Terraform/OpenTofu, configured with Ansible, and reachable through a controlled ingress path.

## Static sites are easy to serve and easy to neglect

The nice thing about Hugo is that the output is boring.

HTML. CSS. Images. No database. No runtime app server. No login system. No plugin ecosystem trying to become a supply chain incident.

That makes the serving path reliable. It also makes staleness easy to miss.

If the site is reachable but the content has not changed in months, basic uptime checks will still say everything is fine. The web server is healthy. Cloudflare is healthy. Caddy is healthy. The tunnel is healthy.

The publishing process can still be dead.

That distinction matters. "The blog is down" and "the blog is stale" are different failures. They need different checks.

## The tunnel changes the threat model

Cloudflare Tunnel is useful because it flips the connection direction.

Instead of exposing a port on my network and letting the internet knock on it, the homelab opens an outbound tunnel to Cloudflare. Cloudflare handles the public edge. The service stays private from the network perspective.

That does not remove all risk. It moves the risk.

Now I care about the tunnel token, Cloudflare account security, DNS configuration, and whether the local tunnel service stays connected. I also care about knowing which internal service the tunnel points to, because future me will absolutely forget if I do not write it down.

The lesson is not "Cloudflare Tunnel makes it secure." The lesson is that it gives me a cleaner boundary than port forwarding, as long as I manage the credentials and config like they matter.

## GitOps made the blog less special

The biggest win was not Hugo. It was making the blog boring.

I do not want a special snowflake web server that exists because I followed a tutorial once. I want the blog to live next to the rest of the homelab definitions.

That means:

- the containers are declared in infrastructure code
- the setup is captured in Ansible
- the build process is scripted
- the public route is documented
- changes can be reviewed later

This is overkill for a hobby blog if the goal is only to publish text.

It is not overkill if the blog is also a way to practice the same habits I use for other infrastructure. Small systems are good places to rehearse serious patterns.

## The failure mode I care about now

Serving the site is solved enough.

The problem to watch is publication freshness. Are new drafts making it from Obsidian to the blog repo? Are commits reaching GitHub? Is the webhook firing? Did Hugo rebuild? Did the generated files actually change?

Those are the checks that matter once the public endpoint is stable.

A static site can sit there looking healthy while quietly becoming a museum.

That is fine if you want a museum. I want a publishing system.

## What I would do again

I would use Hugo again. It is fast, simple, and does not need a database.

I would use Cloudflare Tunnel again. Not exposing inbound ports is worth it.

I would keep the build and ingress pieces separate. It makes troubleshooting cleaner: content generation on one side, public routing on the other.

And I would treat the writing workflow as part of the system earlier. The infrastructure can be perfect, but if the draft pipeline is awkward, nothing gets published.

That was the missing piece.

The blog did not need more hosting work. It needed a backlog.
