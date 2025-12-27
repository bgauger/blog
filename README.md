# Ben Gauger's Blog

Self-hosted blog using Hugo, deployed via GitOps workflow.

## Writing Posts in Obsidian

### Directory Structure

```
blog/
├── content/posts/     ← Write your blog posts here (markdown)
├── Attachments/       ← Drop images here
├── hugo.toml          ← Hugo configuration
└── themes/            ← Hugo theme (hugo-dusk)
```

### Creating a New Post

1. Create a new markdown file in `content/posts/`
2. Add frontmatter:

```yaml
---
title: "Your Post Title"
date: 2025-12-27
draft: false
tags:
  - homelab
  - infrastructure
---
```

3. Write your content in Obsidian (use [[wikilinks]] and images normally)
4. When ready, push to GitHub using Obsidian Git plugin
5. The blog will automatically build and deploy!

### Image Handling

- Drop images in the `Attachments/` folder
- Reference them in Obsidian: `![[image.png]]`
- The build process will convert them to Hugo format automatically

## Deployment

Push to GitHub → Webhook triggers build → Hugo builds site → Caddy serves it

Blog URL: https://blog.bengauger.com
