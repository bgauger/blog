---
title: Adding Comments to My Hugo Blog with Utterances
date: 2025-12-29
draft: false
tags:
  - homelab
  - hugo
  - blog
  - github
---

I wanted to add comments to my self-hosted Hugo blog. After running into issues with Cusdis, I switched to Utterances and had working comments in 5 minutes. Here's why GitHub-based comments ended up being the right choice.

<!--more-->

## The Requirements

When evaluating comment systems for my blog, I had a few key requirements:

- **Reliable and stable** - The solution should just work without extensive debugging
- **Privacy-friendly** - No tracking, no ads, no third-party data collection
- **GitHub integration** - My audience (DevOps/homelab folks) already has GitHub accounts
- **Simple integration** - Easy to add to Hugo without complex setup
- **Free and open source** - No hidden costs or feature paywalls

## The Options

I initially explored several comment systems:

### Self-Hosted: Cusdis

I started by deploying **Cusdis**, a lightweight self-hosted commenting system:

- Created a dedicated LXC container (blog-cusdis)
- Deployed via Docker Compose with SQLite backend
- Configured Cloudflare Tunnel routing
- Added DNS record for comments.bengauger.com

**The problem:** The Docker image had JavaScript loading issues that prevented the comment interface from rendering properly. While I don't mind maintaining infrastructure in my homelab, spending hours debugging a third-party Docker image for something as simple as blog comments felt like unnecessary complexity.

### GitHub-Based: Utterances

**Utterances** is a lightweight comments widget built on GitHub Issues:

- Zero infrastructure needed
- Uses GitHub OAuth for authentication
- Comments stored as GitHub Issues in your repository
- Privacy-friendly (no tracking)
- Open source and actively maintained

**The choice became clear** - instead of debugging Cusdis, I could leverage GitHub's proven infrastructure and get a working comment system in minutes.

## Implementation

The implementation took about 5 minutes:

### Step 1: Install the Utterances GitHub App

Visit: https://github.com/apps/utterances

- Click "Install"
- Select your blog repository
- Grant permissions (read/write access to Issues)

### Step 2: Add the Utterances Widget to Hugo

Create a partial template for comments:

```html
<!-- layouts/partials/comments.html -->
<div class="comments">
  <script src="https://utteranc.es/client.js"
          repo="bgauger/blog"
          issue-term="pathname"
          theme="github-dark"
          crossorigin="anonymous"
          async>
  </script>
</div>
```

**Configuration options:**
- `repo`: Your GitHub repository (format: `username/repo`)
- `issue-term`: How to map posts to issues (`pathname` uses the URL path)
- `theme`: Visual theme (`github-dark`, `github-light`, or others)

### Step 3: Include in Posts

My Hugo theme (Terminal) already supports comments via a partial. The theme's `single.html` template includes:

```html
{{ if not (.Params.hideComments | default false) }}
  {{ partial "comments.html" . }}
{{ end }}
```

This automatically adds comments to all posts unless you set `hideComments: true` in the frontmatter.

### Step 4: Deploy

Commit and push:

```bash
git add layouts/partials/comments.html
git commit -m "Add Utterances comments"
git push
```

My automated deployment workflow took care of the rest:
1. GitHub webhook triggered
2. Hugo rebuilt the site (~300ms)
3. Comments went live

## Important: Repository Must Be Public

For Utterances to work, your blog repository must be:

- **Public** (Utterances can't access private repos)
- **Issues enabled** (Settings → Features → Issues checkbox)

If you want to keep your blog source private, you can create a separate public repository just for comments (e.g., `username/blog-comments`).

## How It Works

When readers visit a blog post:

1. Utterances script loads from `utteranc.es`
2. Script checks for an existing GitHub Issue matching the post's URL
3. If no issue exists, readers can create one by commenting
4. Comments are stored as GitHub Issue comments
5. Issue thread is displayed inline on your blog

**First-time commenters** see a prompt to authorize the Utterances app via GitHub OAuth. After that, commenting is seamless.

## Benefits

**For Me:**
- Zero maintenance (GitHub handles authentication, storage, moderation)
- No database to back up
- No spam filtering needed (GitHub handles it)
- Issue-based workflow integrates with my GitHub workflow

**For Readers:**
- Familiar GitHub authentication
- Markdown support in comments
- Email notifications for replies (via GitHub)
- Can reference other GitHub issues/PRs in comments

## Trade-offs

**Requires GitHub Account:**
The biggest trade-off is that commenters need a GitHub account. For a blog about homelab infrastructure and DevOps, this isn't a barrier - my target audience already uses GitHub.

For a general-audience blog, this might exclude non-technical readers.

**No Self-Hosting:**
Comments are stored on GitHub, not in my infrastructure. I'm okay with this trade-off because:
- Comments are public anyway
- I can export them via GitHub's API if needed
- The reliability and uptime of GitHub far exceeds what I'd run in my homelab

## Alternative: Giscus

**Giscus** is similar to Utterances but uses GitHub Discussions instead of Issues:

- Better separation of concerns (Discussions vs Issues)
- More features (reactions, nested replies)
- Same privacy and authentication model

I chose Utterances for simplicity, but Giscus is equally valid if you prefer Discussions over Issues.

## Lessons Learned

**1. Don't over-engineer**

My first instinct was to self-host everything. But comments don't need to be self-hosted - they're a perfect use case for a managed service.

**2. Consider your audience**

If my blog targeted non-technical readers, GitHub authentication would be a barrier. But for a DevOps/homelab audience, it's actually a feature.

**3. Optimize for maintainability**

Every self-hosted service adds:
- Infrastructure to maintain
- Backups to manage
- Updates to apply
- Potential failure points

Utterances eliminates all of this for the cost of a single script tag.

## Cleanup

Since I ended up not using Cusdis, I cleaned up the infrastructure:

```bash
# Stop Docker container
ssh root@10.0.30.83 "cd /opt/cusdis && docker compose down"

# Destroy LXC container with Terraform
cd terraform/proxmox
tofu destroy -target='module.blog_cusdis'

# Remove infrastructure files
rm terraform/proxmox/blog-cusdis.tf
rm ansible/playbooks/setup-blog-cusdis.yml

# Update Ansible inventory (remove blog-cusdis host)
# Update Cloudflare Tunnel config (remove comments route)
```

Everything back to a clean state.

## Conclusion

Adding comments to my Hugo blog took 5 minutes with Utterances:

1. Install GitHub App
2. Add script tag to Hugo partial
3. Push to GitHub
4. Done

**Total infrastructure added:** 0 servers, 0 databases, 0 maintenance overhead

For a self-hosted blog focused on homelab content, Utterances is the perfect balance of functionality and simplicity.

## Try It Out

The comment widget is live on all my blog posts. If you have a GitHub account, try leaving a comment below!

## Resources

- [Utterances GitHub Repository](https://github.com/utterance/utterances)
- [Giscus (GitHub Discussions Alternative)](https://giscus.app/)
- [Hugo Partials Documentation](https://gohugo.io/templates/partials/)
