---
title: "Three Ansible patterns I keep reusing"
date: 2026-04-12
draft: false
description: "Three Ansible patterns I keep using because they make homelab automation safer to rerun."
tags:
  - ansible
  - homelab
  - automation
  - devops
  - infrastructure-as-code
---

Most of my early Ansible was just shell commands wearing YAML.

It worked, in the same way a pile of extension cords works. Technically electricity moves through it. Nobody should be proud.

The shift happened when I stopped treating Ansible like a remote command runner and started treating it like a way to describe server state. These are the three patterns I keep reusing across my homelab because they make playbooks easier to trust six months later.

<!--more-->

## Pattern 1: make the task safe to run twice

The best Ansible task is boring on the second run.

That sounds obvious, but it changes how you write everything. A playbook should not be a script that hopes the server is in the right mood. It should look at the machine, make the minimum change, and leave quietly if everything is already correct.

Bad version:

```yaml
- name: Add app user
  shell: useradd appuser
```

That works once. The second run fails because the user already exists.

Better version:

```yaml
- name: Ensure app user exists
  ansible.builtin.user:
    name: appuser
    shell: /usr/sbin/nologin
    system: true
```

Now the playbook describes the desired state. The user should exist. Ansible can figure out whether it needs to do anything.

This matters more than style. If I do not trust a playbook to run repeatedly, I stop using it. Then the server starts drifting. Then I am back to SSH notes and memory-based operations, which is just ClickOps with a beard.

## Pattern 2: templates beat copy-paste configs

I used to keep slightly different config files for each host.

That works until the third variation. Then you have one nginx config with a different hostname, another with a different upstream, and another with a special timeout someone added during an incident. Eventually nobody remembers which difference is intentional.

Templates keep the shape of the file in one place and move the differences into variables.

```yaml
- name: Render Caddy site config
  ansible.builtin.template:
    src: caddy-site.conf.j2
    dest: /etc/caddy/sites/{{ service_name }}.conf
    owner: root
    group: root
    mode: "0644"
  notify: reload caddy
```

The template can stay readable:

```jinja2
{{ domain_name }} {
    reverse_proxy {{ upstream_host }}:{{ upstream_port }}
}
```

Then each host or service only needs to define what is different:

```yaml
service_name: mealie
domain_name: mealie.example.internal
upstream_host: 10.0.20.41
upstream_port: 9000
```

This is where Ansible starts feeling less like automation and more like documentation that executes.

If I want to know how a service is exposed, I can read the variables and the template. I do not have to SSH into three machines and compare files like I am conducting a forensic investigation.

## Pattern 3: handlers keep restarts honest

Restarting services is easy. Restarting them only when needed is the part people skip.

A handler runs when a task reports a change. That lets a playbook update packages, render configs, create directories, and only reload the service if something actually changed.

```yaml
- name: Install Caddyfile
  ansible.builtin.template:
    src: Caddyfile.j2
    dest: /etc/caddy/Caddyfile
    owner: root
    group: root
    mode: "0644"
  notify: reload caddy

handlers:
  - name: reload caddy
    ansible.builtin.systemd:
      name: caddy
      state: reloaded
```

Without handlers, you either restart too often or forget to restart at all.

Restarting too often is not harmless. Some services drop connections. Some take time to warm up. Some fail to come back because a config changed and nobody validated it first.

The better version is usually:

1. render the config
2. validate it
3. reload only if validation passed and the file changed

For Caddy, that might look like this:

```yaml
- name: Validate Caddy config
  ansible.builtin.command: caddy validate --config /etc/caddy/Caddyfile
  changed_when: false

- name: Reload Caddy
  ansible.builtin.systemd:
    name: caddy
    state: reloaded
```

In my own environment I have to be careful with Caddy because some validation depends on environment files for Cloudflare DNS credentials. That is exactly the kind of detail a playbook should capture. If I only know it because I remember it, it is already half lost.

## The pattern behind the patterns

All three patterns are really the same idea: stop encoding a sequence of actions and start encoding the desired state.

The user exists.

The config file has this content.

The service reloads only when the config changes.

That is the reason Ansible is still useful even when half the world has moved to containers and Kubernetes. Servers still exist. Files still exist. Services still need to be installed, configured, restarted, and verified.

The difference is whether you do that by hand while hoping you remember the steps, or whether you make the steps boring enough to run again.

I prefer boring.
