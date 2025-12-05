---
title: 'Troubleshooting Proxmox: When Simple Migrations Go Wrong'
date: 2025-12-05T17:19:28-05:00
draft: false
tags:
  - proxmox
  - homelab
  - linux
  - networking
---

# Troubleshooting Proxmox: When Simple Migrations Go Wrong
I love my Proxmox cluster. It’s the beating heart of my homelab, hosting everything from my game servers to my media stack. But sometimes, what should be a simple click-of-a-button task turns into an hour-long troubleshooting rabbit hole.

Recently, I tried to migrate a VM from my primary node (`pve1`) to my secondary node (`pve2`). I’ve done this a dozen times before. I clicked "Migrate," selected the target node, and waited for the magic to happen.

Instead, I got this:

```text
2025-11-28 14:01:09 [pve2] Installed QEMU version '8.1.2' is too old to run 
machine type 'pc-i440fx-9.0+pve0', please upgrade node 'pve2'
2025-11-28 14:01:09 ERROR: online migrate failure - remote command failed with exit code 255
````

## The Problem

The error was clear enough: **Version Mismatch**.

My primary node (`pve1`) had been updated recently, so it was running QEMU 9.0. When I created or rebooted the VM on that node, it adopted the newest machine type (`pc-i440fx-9.0`). My secondary node (`pve2`), however, was still lagging behind on version 8.1.2.

When Proxmox tries to migrate a VM, the destination node looks at the VM's config and asks, "Do I know how to run this hardware?" In this case, `pve2` looked at the request for `pc-i440fx-9.0` and effectively said, "I have no idea what that is."

I had two options:

1.  **Downgrade the VM:** Shut it down, edit the hardware config to use an older machine type (like 8.1), and boot it back up.
2.  **Upgrade the Node:** Bring `pve2` up to parity with `pve1`.

Since I try to follow best practices (and because I hate shutting down live services), I opted for **Option 2**.

## The Rabbit Hole

I SSH'd into `pve2` and ran the standard update command:

```bash
apt update && apt dist-upgrade
```

It finished instantly. Too instantly. It claimed "All packages are up to date." That seemed suspicious for a node running an old version of QEMU.

### Issue 1: It's always DNS

I looked closer at the logs and saw the dreaded message: `Temporary failure resolving 'ftp.us.debian.org'`.

I tried to ping Google. Nothing. I tried to ping `8.8.8.8`. Success.

It turns out my `resolv.conf` file was messy. The server had internet access (gateway was fine), but it couldn't translate domain names to IP addresses. A quick edit fixed that.

```bash
nano /etc/resolv.conf
# Added Google DNS
nameserver 8.8.8.8
```

### Issue 2: The Paywall

With DNS fixed, I confidently ran `apt update` again. And I was immediately hit with a wall of red text.

```text
Err:5 [https://enterprise.proxmox.com/debian/pve](https://enterprise.proxmox.com/debian/pve) bookworm InRelease
  401  Unauthorized [IP: 170.130.165.90 443]
```

I had completely forgotten that this node was still configured for the **Enterprise Repository**. By default, Proxmox checks the enterprise repos, which require a paid subscription key. If you don't have one, it throws a `401 Unauthorized` error.

## The Fix

To get `pve2` updated, I needed to point it to the **No-Subscription Repository**. This involves editing two files.

First, I disabled the enterprise list in `/etc/apt/sources.list.d/pve-enterprise.list`:

```bash
# deb [https://enterprise.proxmox.com/debian/pve](https://enterprise.proxmox.com/debian/pve) bookworm pve-enterprise
```

Then, I added the public repository to `/etc/apt/sources.list`:

```text
deb [http://download.proxmox.com/debian/pve](http://download.proxmox.com/debian/pve) bookworm pve-no-subscription
```

I also had to do the same for my Ceph repository to ensure my storage backend stayed in sync.

Once the repositories were swapped, I ran the update command one last time. Hundreds of megabytes of updates poured in. I rebooted `pve2`, waited for it to come back online, and retried the migration.

**Success\!** The VM moved over without a hitch.

## What's Next

### Automation

This whole ordeal reminded me that "pet" servers are dangerous. I shouldn't have to manually SSH into nodes to keep them in sync. I need to look into setting up Ansible to automate `apt update` across my cluster so version drift doesn't happen again.

### Housekeeping

In the process of fixing `pve2`, I checked `pve1` and realized I had defined the same repository in two different files, causing `apt` to complain about duplicate sources. It's a good reminder that fixing the immediate fire is important, but you should always go back and sweep up the ashes afterwards.
