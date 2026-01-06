---
title: Building a Self-Hosted Blog with Hugo, Cloudflare Tunnel, and GitOps
date: 2025-12-27
draft: false
tags:
  - homelab
  - hugo
  - cloudflare
  - gitops
  - infrastructure
  - automation
---

How I built a fully automated, self-hosted blog using Hugo, Cloudflare Tunnel, and GitHub webhooks - all managed through Infrastructure as Code.

<!--more-->

## Overview

This blog is completely self-hosted on my homelab infrastructure with zero reliance on traditional hosting providers. Every component is defined in code, version-controlled, and automatically deployed. Here's what makes this setup special:

- **Zero port forwarding** - Cloudflare Tunnel provides secure external access
- **Automatic deployments** - Push to GitHub, blog rebuilds in seconds
- **Infrastructure as Code** - Everything defined in Terraform and Ansible
- **Write in Obsidian** - Familiar markdown editor with Git integration
- **Fast builds** - Static site generation completes in milliseconds

> **Note:** This guide documents my specific setup and configuration that worked for my environment. Your setup may differ based on your hardware, network configuration, storage layout, DNS setup, and other infrastructure factors. Use this as a reference implementation and adapt the configurations to match your specific homelab environment.

## Architecture

The blog infrastructure consists of two LXC containers running on Proxmox:

1. **blog-hugo** (10.0.30.81) - Hugo build server with GitHub webhook listener
2. **blog-caddy** (10.0.30.82) - Caddy reverse proxy with Cloudflare Tunnel

### Traffic Flow

**Public Access:**
```
User → Cloudflare Edge → Cloudflare Tunnel → Caddy (10.0.30.82) → Hugo Site (10.0.30.81:80)
```

**Automated Deployment:**
```
GitHub Push → Webhook (10.0.30.81:9000) → Build Script → Hugo Rebuild → Live Site
```

## Infrastructure Setup

### 1. Proxmox LXC Containers

I use Terraform (OpenTofu) to provision the infrastructure. Here's the configuration for the Hugo build server:

```hcl
# terraform/proxmox/blog-hugo.tf
module "blog_hugo" {
  source = "../modules/proxmox-lxc"

  vmid         = 111
  hostname     = "blog-hugo"
  target_node  = "your-proxmox-node"
  tags         = ["terraform", "blog"]
  ostemplate   = "your-storage:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst"
  unprivileged = true
  onboot       = true
  start        = true

  cores  = 1
  memory = 1024
  swap   = 512

  rootfs_storage = "your-storage-pool"
  rootfs_size    = "8G"

  network_ip     = "10.0.30.81/24"
  network_gw     = "10.0.30.1"
  network_bridge = "vmbr0"

  features_keyctl  = false
  features_nesting = true

  ssh_public_keys = [
    "ssh-ed25519 AAAAC3Nza...your-public-key-here... user@example.com"
  ]
}
```

And the Caddy reverse proxy:

```hcl
# terraform/proxmox/blog-caddy.tf
module "blog_caddy" {
  source = "../modules/proxmox-lxc"

  vmid         = 112
  hostname     = "blog-caddy"
  target_node  = "your-proxmox-node"
  tags         = ["terraform", "blog"]
  ostemplate   = "your-storage:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst"
  unprivileged = true
  onboot       = true
  start        = true

  cores  = 1
  memory = 512
  swap   = 256

  rootfs_storage = "your-storage-pool"
  rootfs_size    = "4G"

  network_ip     = "10.0.30.82/24"
  network_gw     = "10.0.30.1"
  network_bridge = "vmbr0"

  features_keyctl  = false
  features_nesting = false

  ssh_public_keys = [
    "ssh-ed25519 AAAAC3Nza...your-public-key-here... user@example.com"
  ]
}
```

Deploy with:

```bash
cd terraform/proxmox
tofu apply
```

### 2. Hugo Build Server Setup

Install Hugo and dependencies using Ansible:

```yaml
# ansible/playbooks/setup-blog-hugo.yml
---
- name: Setup Hugo Blog Server
  hosts: blog-hugo
  become: yes

  tasks:
    - name: Install dependencies
      apt:
        name:
          - git
          - nginx
          - python3
          - python3-pip
          - python3-flask
        state: present
        update_cache: yes

    - name: Download Hugo extended
      get_url:
        url: https://github.com/gohugoio/hugo/releases/download/v0.121.1/hugo_extended_0.121.1_linux-amd64.deb
        dest: /tmp/hugo.deb

    - name: Install Hugo
      apt:
        deb: /tmp/hugo.deb

    - name: Create blog directory
      file:
        path: /opt/blog
        state: directory
        mode: '0755'

    - name: Clone blog repository
      git:
        repo: 'git@github.com:yourusername/blog.git'
        dest: /opt/blog
        accept_hostkey: yes
        key_file: /root/.ssh/id_ed25519

    - name: Create build script
      copy:
        dest: /opt/build-blog.sh
        mode: '0755'
        content: |
          #!/bin/bash
          set -euo pipefail

          BLOG_DIR="/opt/blog"
          OUTPUT_DIR="/var/www/blog"

          cd "$BLOG_DIR"
          git pull origin main
          hugo --minify -d "$OUTPUT_DIR"
          chown -R www-data:www-data "$OUTPUT_DIR"
          echo "Blog built successfully at $(date)"

    - name: Configure nginx
      copy:
        dest: /etc/nginx/sites-available/blog
        content: |
          server {
              listen 80;
              server_name _;

              root /var/www/blog;
              index index.html;

              location / {
                  try_files $uri $uri/ =404;
              }
          }
      notify: restart nginx

    - name: Enable nginx site
      file:
        src: /etc/nginx/sites-available/blog
        dest: /etc/nginx/sites-enabled/blog
        state: link
      notify: restart nginx

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

### 3. GitHub Webhook Listener

Create a webhook listener that triggers builds on push:

```python
# /opt/webhook-listener.py
#!/usr/bin/env python3
from flask import Flask, request
import subprocess
import hmac
import hashlib
import os

app = Flask(__name__)
SECRET = os.environ.get('WEBHOOK_SECRET', 'change-me-in-production')

def verify_signature(payload, signature):
    expected = 'sha256=' + hmac.new(
        SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Hub-Signature-256', '')

    if not verify_signature(request.data, signature):
        return 'Invalid signature', 401

    try:
        subprocess.run(['/opt/build-blog.sh'], check=True)
        return 'Build triggered', 200
    except subprocess.CalledProcessError:
        return 'Build failed', 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=9000)
```

Create a systemd service:

```ini
# /etc/systemd/system/blog-webhook.service
[Unit]
Description=GitHub Webhook Listener for Blog
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt
Environment="WEBHOOK_SECRET=your-secure-secret-here"
ExecStart=/usr/bin/python3 /opt/webhook-listener.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reload
systemctl enable --now blog-webhook
```

### 4. Caddy Reverse Proxy

Install Caddy on the second container:

```yaml
# ansible/playbooks/setup-blog-caddy.yml
---
- name: Setup Caddy Reverse Proxy
  hosts: blog-caddy
  become: yes

  tasks:
    - name: Install dependencies
      apt:
        name:
          - debian-keyring
          - debian-archive-keyring
          - apt-transport-https
          - curl
        state: present
        update_cache: yes

    - name: Add Caddy GPG key
      shell: curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

    - name: Add Caddy repository
      apt_repository:
        repo: "deb [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
        state: present

    - name: Install Caddy
      apt:
        name: caddy
        state: present
        update_cache: yes

    - name: Configure Caddyfile
      copy:
        dest: /etc/caddy/Caddyfile
        content: |
          {
              email your-email@example.com
          }

          # HTTP configuration for Cloudflare tunnel traffic
          http://blog.yourdomain.com {
              reverse_proxy 10.0.30.81:80
              encode gzip

              log {
                  output file /var/log/caddy/blog.log
                  format json
              }

              header {
                  X-Content-Type-Options "nosniff"
                  X-Frame-Options "DENY"
                  Referrer-Policy "strict-origin-when-cross-origin"
              }
          }

          # HTTPS will be used once DNS is active
          https://blog.yourdomain.com {
              reverse_proxy 10.0.30.81:80
              encode gzip

              log {
                  output file /var/log/caddy/blog.log
                  format json
              }

              header {
                  Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
                  X-Content-Type-Options "nosniff"
                  X-Frame-Options "DENY"
                  Referrer-Policy "strict-origin-when-cross-origin"
              }
          }
      notify: reload caddy

  handlers:
    - name: reload caddy
      systemd:
        name: caddy
        state: reloaded
```

### 5. Cloudflare Tunnel Setup

Install cloudflared on the Caddy server:

```bash
# On blog-caddy server
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
dpkg -i cloudflared.deb
```

Authenticate with Cloudflare:

```bash
cloudflared tunnel login
```

Create the tunnel:

```bash
cloudflared tunnel create blog
# Note the tunnel ID (you'll need this for config)
```

Configure the tunnel:

```yaml
# /etc/cloudflared/config.yml
tunnel: YOUR-TUNNEL-ID-HERE
credentials-file: /etc/cloudflared/YOUR-TUNNEL-ID-HERE.json

ingress:
  - hostname: blog.yourdomain.com
    service: http://localhost:80
  - hostname: webhook.yourdomain.com
    service: http://10.0.30.81:9000
  - service: http_status:404
```

Create DNS records:

```bash
cloudflared tunnel route dns blog blog.yourdomain.com
cloudflared tunnel route dns blog webhook.yourdomain.com
```

Install as systemd service:

```bash
cloudflared service install
systemctl enable --now cloudflared
```

### 6. GitHub Repository Setup

Configure GitHub webhook at `https://github.com/yourusername/blog/settings/hooks/new`:

- **Payload URL:** `https://webhook.yourdomain.com/webhook`
- **Content type:** `application/json`
- **Secret:** Your webhook secret from the environment variable
- **Events:** Just the push event

### 7. Obsidian Integration

Install and configure the Obsidian Git plugin for automatic syncing:

**Install the plugin:**

1. Settings → Community plugins → Browse → "Obsidian Git"
2. Install and enable the plugin

**Configure Git plugin:**

1. Click the gear icon next to "Git" in Community plugins
2. Scroll to **Advanced** section:
   - **Custom base path**: `blog` (if your blog repo is in a subdirectory)
3. Set local git config in your blog repository:
   ```bash
   cd ~/Documents/EDI/blog/
   git config user.name "Your Name"
   git config user.email "your-email@example.com"
   ```
4. Reload the plugin or restart Obsidian

**Configure automatic syncing:**

Once the plugin shows "Git is ready", configure these settings:

- **Auto commit-and-sync interval (minutes)**: `5`
  - Auto-commits and pushes changes every 5 minutes
- **Auto pull interval (minutes)**: `5`
  - Pulls changes from GitHub every 5 minutes
- **Auto commit-and-sync after stopping file edits**: Enable
  - Commits/pushes seconds after you stop editing
- **Commit message on auto commit-and-sync**: `vault backup: {{date}}`

**Optional (reduce noise):**
- **Disable informative notifications**: Enable to avoid constant popups

Now when you write posts in Obsidian and save them, the Git plugin automatically commits and pushes to GitHub within seconds, triggering your webhook and rebuilding your blog.

## Hugo Configuration

Here's my Hugo configuration:

```toml
# hugo.toml
baseURL = 'https://blog.yourdomain.com/'
languageCode = 'en-us'
title = 'Your Name'
theme = 'terminal'

[params]
  author = 'Your Name'
  description = 'Homelab, Infrastructure, and Technology'
  githubUsername = 'yourusername'

[menu]
  [[menu.main]]
    identifier = 'about'
    name = 'About'
    url = '/about/'
    weight = 10

  [[menu.main]]
    identifier = 'posts'
    name = 'Posts'
    url = '/posts/'
    weight = 20

[markup]
  [markup.highlight]
    style = 'monokai'
```

## Writing Posts

Create posts in `content/posts/` with this frontmatter:

```markdown
---
title: "Your Post Title"
date: 2025-12-27
draft: false
tags:
  - homelab
  - infrastructure
---

Your intro paragraph.

<!--more-->

Extended content continues here...
```

The `<!--more-->` tag controls where the summary ends on the posts listing page.

## Deployment Workflow

The complete workflow is:

1. Write post in Obsidian (`~/Documents/EDI/blog/content/posts/`)
2. Obsidian Git auto-commits and pushes to GitHub
3. GitHub webhook triggers build on blog-hugo server
4. Hugo rebuilds static site in ~170ms
5. Site is immediately live at blog.yourdomain.com

**Total time from save to live: ~3-5 seconds**

## Performance & Monitoring

### Build Performance

Hugo builds are incredibly fast, typically completing in under 200ms for a small blog like this. Static site generators are extremely efficient compared to dynamic CMSs.

**Monitor build times:**
```bash
# Watch webhook trigger and build
journalctl -u blog-webhook -f
```

### Infrastructure Monitoring

**Check webhook service:**
```bash
systemctl status blog-webhook
journalctl -u blog-webhook -n 50
```

**Check Cloudflare tunnel:**
```bash
systemctl status cloudflared
journalctl -u cloudflared -n 50
```

**Check Caddy:**
```bash
systemctl status caddy
tail -f /var/log/caddy/blog.log
```

You can also integrate with existing monitoring solutions like Prometheus, Grafana, or your homelab dashboard.

## Security Considerations

### Cloudflare Tunnel Benefits

- No open ports on firewall
- DDoS protection at Cloudflare edge
- Automatic SSL/TLS termination
- Web Application Firewall (WAF) available

### Webhook Security

The webhook listener uses HMAC-SHA256 signature verification to ensure requests are from GitHub:

```python
def verify_signature(payload, signature):
    expected = 'sha256=' + hmac.new(
        SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

### Container Security

- Unprivileged LXC containers
- Minimal attack surface (Debian minimal template)
- SSH key-only authentication
- Regular security updates via Ansible

## Troubleshooting

### DNS Resolution Issues

If your blog doesn't resolve, verify DNS is working:

```bash
# Test with external DNS
dig @8.8.8.8 blog.yourdomain.com

# Should return Cloudflare IPs
# If not, check your DNS records in Cloudflare dashboard
```

### Webhook Not Triggering

Check webhook logs:

```bash
journalctl -u blog-webhook -n 50
```

Test webhook manually:

```bash
curl -X POST https://webhook.yourdomain.com/webhook \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: sha256=YOUR_SIGNATURE" \
  -d '{}'
```

### Cloudflare Tunnel Down

Check tunnel status:

```bash
systemctl status cloudflared
journalctl -u cloudflared -n 50
```

Tunnel should show 4 active connections to Cloudflare edge.

## Cost Analysis

**Monthly costs:**
- Domain: ~$12/year = $1/month
- Cloudflare: $0 (using free tier)
- Hosting: $0 (self-hosted)
- Total: ~$1/month

**One-time costs:**
- Storage hardware: Variable (can use existing NAS or local storage)
- Server hardware: Existing homelab infrastructure
- Time investment: ~4-6 hours for initial setup

## Conclusion

This setup provides a production-grade blogging platform with:

- Complete control over infrastructure
- Sub-second deployment times
- Zero monthly hosting costs (beyond domain)
- Privacy and data ownership
- Learning opportunities with real infrastructure

The entire stack is defined as code, making it reproducible and documented. If I need to rebuild everything from scratch, it's just:

```bash
cd terraform/proxmox
tofu apply

cd ../../ansible
ansible-playbook -i inventory/hosts.yml playbooks/setup-blog-hugo.yml
ansible-playbook -i inventory/hosts.yml playbooks/setup-blog-caddy.yml
```

And I'm back online in minutes.

## Next Steps

Potential improvements:

- [ ] Add CDN caching for static assets
- [x] Implement comment system (maybe isso or utterances)
- [ ] Set up automated backups of blog content
- [ ] Add analytics (privacy-focused like Plausible)
- [ ] Create staging environment for preview deployments
- [ ] Implement automated testing for broken links

## Resources

- [Hugo Documentation](https://gohugo.io/documentation/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Caddy Documentation](https://caddyserver.com/docs/)
- [Proxmox LXC Containers](https://pve.proxmox.com/wiki/Linux_Container)
- My Infrastructure Repository (available on request)

---

*This post will be updated as I make improvements to the infrastructure and workflow.*
