---
title: "Managing My Proxmox Homelab with OpenTofu: From Clickops to GitOps"
date: 2026-02-16
draft: false
description: "How I imported an existing Proxmox homelab into OpenTofu without tearing everything down."
tags:
  - homelab
  - proxmox
  - terraform
  - opentofu
  - gitops
  - infrastructure-as-code
---

"How did I configure that VM again?" I found myself asking this question for the third time in a month. My infrastructure existed, it worked, but I had no documentation beyond Proxmox's web UI and my fading memory.

I needed Infrastructure as Code. But I wasn't about to destroy and rebuild nine working resources. Here's how I imported my entire 3-node Proxmox homelab into OpenTofu while everything stayed online.

<!--more-->

## The Problem

Like many homelabbers, I was stuck in "clickops" mode: clicking through the Proxmox web UI to create VMs and containers, tweaking settings manually, and crossing my fingers that I'd remember exactly how I configured everything if I ever needed to rebuild.

My homelab consists of a 3-node Proxmox cluster running:
- 5 LXC containers (Pi-hole, Homepage dashboard, Immich photos, Mealie recipes, Kanboard)
- 4 VMs (TrueNAS storage server, proxy, ARM development box, and a Servarr media stack)

When I decided it was time to adopt Infrastructure as Code (IaC), I had a choice to make: Terraform or OpenTofu?

## Why OpenTofu?

OpenTofu is a community-driven fork of Terraform that emerged after HashiCorp changed Terraform's license from open source (MPL) to the Business Source License (BSL) in 2023. For a homelab project where:
- I value truly open source software
- I don't need enterprise support
- I want to support community-governed projects

OpenTofu was the clear winner. Plus, it's 100% compatible with existing Terraform configurations, so migration is seamless.

## Why This Isn't Just Homelab Tinkering

Here's the thing about learning Infrastructure as Code on a homelab—you're learning the exact same tools and patterns that real companies use to manage actual production infrastructure. Like, the same commands, the same workflows, the same "oh crap I broke state" moments.

At my job, nobody's allowed to manually click around and change production servers. Everything goes through Terraform, everything gets code reviewed, everything's tracked in Git. You click around in production? That's an incident report.

So when I'm importing my Proxmox VMs into OpenTofu, handling state management, building reusable modules—that's not pretend work. That's the actual workflow I'd use at any company running infrastructure as code.

And in interviews? Most people say "yeah I did some Terraform tutorials." Cool. I migrated a 3-node cluster to IaC without breaking anything, built modules for UEFI boot and PCI passthrough, and I'm currently figuring out state backends. One of those answers is way more interesting.

Not saying you need a homelab to learn this stuff. But it sure helps to have a playground where breaking things doesn't page anyone at 3 AM.

## The Journey: Importing Existing Infrastructure

The beauty of OpenTofu (and Terraform) is that you don't need to rebuild everything from scratch. The `tofu import` command lets you bring existing infrastructure under management.

### Step 1: Setting Up the Provider

I chose the community-maintained [bpg/proxmox](https://github.com/bpg/terraform-provider-proxmox) provider, which has excellent support for Proxmox VE 8.x and 9.x:

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.69"
    }
  }
}

provider "proxmox" {
  endpoint  = var.proxmox_api_url
  api_token = var.proxmox_api_token
  insecure  = var.proxmox_tls_insecure
}
```

**Pro tip:** Use API tokens instead of username/password authentication. Create a token in Proxmox under Datacenter → Permissions → API Tokens.

### Step 2: Secrets Management

Before going further, let's talk about handling API credentials securely. You'll notice the provider configuration uses variables - here's how to manage them without exposing secrets.

**Creating a Proxmox API Token:**

1. Log into Proxmox web UI
2. Navigate to Datacenter → Permissions → API Tokens
3. Click "Add" and create a token (e.g., `root@pam!tofu`)
4. **Important:** Uncheck "Privilege Separation" if you want the token to have full permissions
5. Copy the token secret - you can't retrieve it later!

**Required Permissions:**

For managing VMs and containers, the token needs these privileges:
- `VM.Allocate` - Create/remove VMs
- `VM.Config.*` - Modify VM configurations
- `Datastore.AllocateSpace` - Allocate storage
- `SDN.Use` - Use network bridges

The easiest approach for a homelab is giving your token `Administrator` role on `/`, but you can lock it down further if needed.

**Storing Credentials:**

I use a `terraform.tfvars` file that's gitignored:

```hcl
# terraform.tfvars (NEVER commit this!)
proxmox_api_url      = "https://10.0.30.10:8006/api2/json"
proxmox_api_token    = "root@pam!tofu=12345678-1234-1234-1234-123456789abc"
proxmox_tls_insecure = true  # Only for homelab with self-signed certs
```

Then reference them in `variables.tf`:

```hcl
variable "proxmox_api_url" {
  description = "Proxmox API endpoint"
  type        = string
}

variable "proxmox_api_token" {
  description = "Proxmox API token"
  type        = string
  sensitive   = true
}

variable "proxmox_tls_insecure" {
  description = "Skip TLS verification (homelab only)"
  type        = bool
  default     = false
}
```

**Alternative approaches:**

- **Environment variables:** Export `TF_VAR_proxmox_api_token` in your shell
- **Pass (password manager):** `proxmox_api_token = "$(pass show homelab/proxmox-token)"`
- **1Password/Bitwarden CLI:** Integrate with password manager
- **Vault:** Overkill for most homelabs, but enterprise-grade

**Critical:** Always add this to your `.gitignore`:

```
# .gitignore
*.tfvars
*.tfstate
*.tfstate.*
.terraform/
```

### Step 3: Building Reusable Modules

Rather than writing individual resource definitions for each container/VM, I created reusable modules. This follows the DRY (Don't Repeat Yourself) principle and makes it easy to spin up new infrastructure.

**LXC Module** (`modules/proxmox-lxc/`):
```hcl
resource "proxmox_virtual_environment_container" "container" {
  vm_id        = var.vmid
  node_name    = var.target_node
  unprivileged = var.unprivileged
  tags         = var.tags

  started = var.start

  cpu {
    cores = var.cores
  }

  memory {
    dedicated = var.memory
    swap      = var.swap
  }

  disk {
    datastore_id = var.rootfs_storage
    size         = parseint(replace(var.rootfs_size, "G", ""), 10)
  }

  network_interface {
    name    = var.network_name
    bridge  = var.network_bridge
    enabled = true
  }

  operating_system {
    template_file_id = var.ostemplate
    type             = "debian"
  }

  initialization {
    hostname = var.hostname

    ip_config {
      ipv4 {
        address = var.network_ip
        gateway = var.network_gw
      }
    }
  }

  features {
    keyctl  = var.features_keyctl
    nesting = var.features_nesting
  }

  lifecycle {
    ignore_changes = [
      operating_system[0].template_file_id,
      description,
      tags,
      console,
      disk[0].mount_options,
      network_interface[0].mac_address,
      network_interface[0].firewall,
      startup,
      mount_point,
      device_passthrough,
    ]
  }
}
```

The `lifecycle` block is crucial - it prevents OpenTofu from trying to "fix" things that are managed outside of Terraform (like manually added mount points or device passthrough configs).

### Step 4: Defining Infrastructure

With modules in place, defining new infrastructure becomes incredibly simple:

```hcl
module "pihole" {
  source = "../modules/proxmox-lxc"

  vmid         = 104
  hostname     = "pihole"
  target_node  = "mako"
  tags         = ["terraform"]
  ostemplate   = "tempest:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst"
  unprivileged = true
  onboot       = true
  start        = true

  cores  = 1
  memory = 512
  swap   = 512

  rootfs_storage = "tempest"
  rootfs_size    = "2G"

  network_ip     = "10.0.30.100/24"
  network_gw     = "10.0.30.1"
  network_bridge = "vmbr0"

  features_keyctl  = true
  features_nesting = true
}
```

### Step 5: Importing Existing Resources

For each existing container or VM, I ran:

```bash
tofu import 'module.pihole.proxmox_virtual_environment_container.container' mako/104
```

This told OpenTofu: "This resource already exists in Proxmox, now manage it."

### Step 6: Troubleshooting Imports

Not everything went smoothly on the first try. Here are issues I encountered and how I fixed them:

**Problem 1: "Resource already managed by this OpenTofu"**

If you accidentally import the same resource twice, you'll get a duplicate error. Fix:

```bash
# Remove from state (doesn't delete the actual resource)
tofu state rm 'module.pihole.proxmox_virtual_environment_container.container'

# Re-import with correct path
tofu import 'module.pihole.proxmox_virtual_environment_container.container' mako/104
```

**Problem 2: "Error: expected X, got Y" after import**

The imported resource attributes don't match your configuration. Common culprits:
- Memory values (provider expects MiB, not GiB)
- Disk sizes (use integers, not strings like "8G")
- Boolean values as strings

Solution: Run `tofu plan` after import to see the diff, then adjust your config to match reality.

**Problem 3: Endless diff on certain attributes**

Some attributes constantly show changes even though nothing changed (like `template_file_id` or `mac_address`). This is why the `lifecycle.ignore_changes` block is crucial:

```hcl
lifecycle {
  ignore_changes = [
    operating_system[0].template_file_id,  # Template may be deleted
    network_interface[0].mac_address,       # Auto-generated
    disk[0].mount_options,                  # Manually configured
  ]
}
```

**Problem 4: "VM is locked"**

If a VM is running a task (backup, snapshot, migration), imports will fail. Wait for tasks to complete or stop the VM temporarily.

**Pro tip:** Import resources incrementally. Don't try to import all 9 resources at once - do one, run `tofu plan`, verify it's clean, then move to the next. This makes debugging much easier.

**Validation workflow:**

```bash
# After each import
tofu import 'module.X.resource.name' node/vmid
tofu plan  # Should show "No changes" if config matches reality
tofu state list  # Verify the resource is in state
```

**Pro tip #5: Use `tofu state show` to inspect imported resources**

After importing, if you're not sure what attributes are set:

```bash
tofu state show 'module.pihole.proxmox_virtual_environment_container.container'
```

This shows the complete resource state, helping you write accurate configuration that matches reality.

**Pro tip #6: Test imports on non-critical resources first**

I imported my Pi-hole container first (easily rebuildable) before touching my TrueNAS VM (irreplaceable storage). This let me learn the import process on something I could afford to break.

## Handling Complex Scenarios

### UEFI Boot and PCI Passthrough

My TrueNAS VM needed special handling - it uses UEFI boot and has PCI devices passed through for direct disk access. I extended my VM module to support these:

```hcl
# EFI disk for UEFI boot
dynamic "efi_disk" {
  for_each = var.efi_disk_datastore != null ? [1] : []
  content {
    datastore_id      = var.efi_disk_datastore
    file_format       = var.efi_disk_format
    type              = var.efi_disk_type
    pre_enrolled_keys = var.efi_pre_enrolled_keys
  }
}

# PCI passthrough devices
dynamic "hostpci" {
  for_each = var.hostpci_devices
  content {
    device  = hostpci.value.device
    id      = hostpci.value.id
    pcie    = lookup(hostpci.value, "pcie", false)
    rombar  = lookup(hostpci.value, "rombar", true)
    xvga    = lookup(hostpci.value, "xvga", false)
  }
}
```

Then deploying TrueNAS with UEFI and PCI passthrough becomes straightforward:

```hcl
module "truenas" {
  source = "../modules/proxmox-vm"

  vmid        = 101
  name        = "truenas"
  target_node = "citadel"

  cpu_cores        = 8
  memory_dedicated = 32768
  disk_size        = 64

  # UEFI boot configuration
  bios               = "ovmf"
  efi_disk_datastore = "tempest"

  # PCI passthrough for storage controllers
  hostpci_devices = [
    { id = "hostpci0", device = "0000:00:1f.2" },
    { id = "hostpci1", device = "0000:00:1f.4" }
  ]
}
```

### Mount Points and Device Passthrough

Some containers (like Immich for photos) have manually configured mount points or device passthrough. The `lifecycle.ignore_changes` block handles this gracefully by telling OpenTofu: "Don't touch these, they're managed manually."

## The Results

After importing all 9 resources (5 LXCs + 4 VMs), I ran `tofu plan` and got the magic message:

```
No changes. Your infrastructure matches the configuration.
```

Success! I've completely eliminated clickops from my homelab.

## What Actually Changed

**Before:** I'd click through the Proxmox UI for 15-20 minutes every time I needed a new VM. Half the time I'd forget whether I gave the last one 2 cores or 4 cores. My documentation was screenshots in a notes app and hoping I remembered.

If something broke? Good luck. I'd try to rebuild it from memory, realize I forgot the network bridge name, go find that screenshot from three months ago, hope the settings were right.

**After:** `tofu apply` and I get a VM in 2 minutes. Exact same config every time because it's in code. I can look at my Git history and see exactly when I changed that Jellyfin container from 2GB to 4GB RAM and why.

Disaster recovery went from "panic and try to remember everything" to "run this one command." That alone made the whole thing worth it.

**Real numbers:**
- Took me about 4 hours to learn OpenTofu, build the modules, and import everything
- I probably save 6+ hours a month not manually creating VMs or hunting for old configs
- Paid for itself in less than a month

The peace of mind? Can't quantify that. But it's real.

## State Management: The Elephant in the Room

Let's talk about OpenTofu state files - because this is something I haven't fully solved yet, and I want to be transparent about it.

### What is State?

When you run `tofu apply`, OpenTofu creates a `terraform.tfstate` file that maps your configuration to real-world resources. This file is crucial - it's how OpenTofu knows what exists, what needs to be created, updated, or destroyed.

Lose your state file? OpenTofu thinks nothing exists and will try to recreate everything, potentially causing conflicts or data loss.

### My Current (Imperfect) Setup

Right now, I'm using **local state**. The `terraform.tfstate` file lives in my project directory on my laptop. I know this isn't ideal, but here's why I started this way:

**Pros:**
- Dead simple - no additional infrastructure needed
- Works great for solo development and learning
- Zero cost
- No network dependency

**Cons:**
- ⚠️ Single point of failure (if my laptop dies, I lose state)
- ⚠️ No collaboration (can't share management with others)
- ⚠️ No state locking (can't prevent concurrent modifications)
- ⚠️ Manual backup required
- ⚠️ Accidentally committing state to Git would expose infrastructure details

For now, I mitigate this by:
1. Adding `*.tfstate*` to `.gitignore` (never commit state!)
2. Manually backing up state files to my TrueNAS server
3. Being the only person managing this infrastructure

### Better Options (My Next Step)

Here are the state backend options I'm evaluating:

### State Backend Options (What I'm Actually Considering)

**Local state (what I'm using now):** Dead simple, works fine for solo projects, but if my laptop dies I'm screwed. Not great, but good enough to learn on.

**MinIO on TrueNAS (probably doing this next):** Self-hosted S3-compatible storage. Keeps everything in my homelab, gives me versioning, no cloud dependency. Requires setting up MinIO but I've been meaning to do that anyway.

**Cloudflare R2:** 10GB free tier, S3-compatible, no egress fees. Honestly this is probably the smart choice if you don't care about self-hosting everything. I just like keeping things local.

**AWS S3:** The "real" option. What everyone uses in production. Costs a few bucks a month. Overkill for a homelab but if you want to learn the actual tools companies use, this is it.

**Terraform Cloud:** Free for small projects, has a web UI, handles locking. The irony of using HashiCorp's cloud for OpenTofu isn't lost on me, but it works.

I'm probably doing MinIO because I'm stubborn about self-hosting. But honestly? If I wasn't weird about that, I'd just use Cloudflare R2 and call it a day.

**1. S3-Compatible Backend (Recommended for most)**
Store state in object storage with DynamoDB for locking:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tofu-state"
    key            = "proxmox/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-locks"
  }
}
```

**Homelab options:**
- MinIO (self-hosted S3-compatible storage)
- AWS S3 free tier (5GB free)
- Backblaze B2 (very cheap)
- Cloudflare R2 (10GB free)

**2. Terraform Cloud (Free Tier)**
HashiCorp offers a free tier for up to 500 resources:
- Remote state storage
- State locking
- Run history
- Web UI for viewing state

Since OpenTofu is compatible, you can use this despite it being a HashiCorp product.

**3. HTTP Backend**
Store state on any HTTP server with GET/POST/DELETE support. Good for custom solutions.

**4. Git Backend (Not Recommended)**
Some people use git-crypt or Git LFS to store encrypted state in Git. This works but lacks locking and can cause merge conflicts.

**5. Local with External Backup**
Continue using local state but automate backups to multiple locations. Still no locking, but better than nothing.

### What I'm Planning

My next step is setting up MinIO on my TrueNAS server to create an S3-compatible backend. This keeps everything self-hosted while gaining:
- Automatic state versioning
- Encryption at rest
- The ability to add state locking later
- A foundation for multi-user collaboration if needed

The migration is straightforward - OpenTofu can migrate state between backends with a single command:

```bash
tofu init -migrate-state
```

**Bottom line:** Local state is fine to get started and learn, but plan to upgrade to remote state before you rely on this for anything critical. Don't let perfect be the enemy of good - get your infrastructure into code first, then improve your state management.

## Project Structure

Here's what my final repository looks like:

```
homelab-gitops/
├── terraform/
│   ├── modules/
│   │   ├── proxmox-lxc/      # Reusable LXC module
│   │   └── proxmox-vm/       # Reusable VM module
│   └── proxmox/
│       ├── providers.tf      # Provider configuration
│       ├── variables.tf      # Variable definitions
│       ├── terraform.tfvars  # Actual values (gitignored)
│       ├── pihole.tf        # Individual resource configs
│       ├── homepage.tf
│       ├── immich.tf
│       ├── truenas.tf
│       └── ...
├── ansible/                  # Next: configuration management
├── kubernetes/               # Future: K8s workloads
└── docs/                     # Documentation and runbooks
```

## What's Next?

With infrastructure defined as code, the next layer is **configuration management** with Ansible:
- Installing packages and configuring the OS
- Deploying Docker/Podman
- Managing application configs
- Deploying containers and services

The dream workflow becomes:
1. `tofu apply` → Creates the VM/container
2. `ansible-playbook` → Configures it and deploys apps
3. Commit to Git → Changes are tracked and reproducible

## What I Actually Learned

OpenTofu works. Like, actually works. Drop-in Terraform replacement with a better license. No gotchas, no compatibility issues.

Build modules from the start. I didn't do this at first and ended up refactoring. Learn from my mistake—DRY principle applies to infrastructure.

You can import existing stuff. Don't rebuild your entire homelab from scratch just to get it into code. Import it, tweak the config until `tofu plan` shows no changes, move on.

Lifecycle ignore rules saved me. Without them, OpenTofu kept trying to "fix" stuff I'd manually configured. Sometimes that's the right call.

Escaping clickops feels amazing. My infrastructure's in Git now. I can rebuild anything. That peace of mind hits different.

## What I'd Do Differently Next Time

Looking back at this journey, here's what I'd change:

**1. Start with remote state from day one**

Even though I wanted to learn locally first, I'd set up MinIO immediately. Migrating state later works, but starting right saves a step.

**2. Document module variables more thoroughly**

Some of my module variables lack good descriptions. Future me (and anyone else using these modules) would benefit from better comments.

**3. Use consistent tagging strategy**

I tag resources with "terraform" but I should add more tags: environment, owner, purpose. This helps with cost tracking and organization.

**4. Set up pre-commit hooks earlier**

I manually check that `*.tfvars` never gets committed. A pre-commit hook would automate this and prevent accidents.

**5. Plan the module structure before starting**

I refactored my modules twice as I learned better patterns. Spending an extra hour planning would have saved refactoring time.

These aren't mistakes—they're learning experiences. But if you're starting fresh, skip my trial-and-error and start with these patterns from day one.

## If You Want To Try This

Start simple. Install OpenTofu (`brew install opentofu` on Mac), read the docs for like 20 minutes, find the Proxmox provider docs.

Then pick one throwaway VM or container. Something you don't care about. Manually create it in Proxmox. Now write the OpenTofu config for it and try importing with `tofu import`. Fight with the config until `tofu plan` says "No changes."

That's it. That's the whole process.

Once you get one resource imported successfully, the rest is just repetition. Import another one. Then another. Before you know it your whole homelab's in code.

Will you mess something up? Probably. I did. That's why you start with stuff you don't care about.

The state backend thing can come later. Get comfortable with the basics first, worry about remote state when you're actually using this regularly.

If you get stuck, hit me up. I've probably made whatever mistake you're about to make.

---

**Have you adopted IaC for your homelab? What tools are you using? What challenges have you hit?** I'd love to hear your experiences.

## Resources

- [OpenTofu Documentation](https://opentofu.org/docs/)
- [bpg/proxmox Provider](https://github.com/bpg/terraform-provider-proxmox)
- [Proxmox VE Documentation](https://pve.proxmox.com/wiki/Main_Page)
