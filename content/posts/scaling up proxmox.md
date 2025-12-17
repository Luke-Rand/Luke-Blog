---
title: 'Scaling Up: Building a Bulletproof K3s Cluster on Proxmox'
date: 2025-12-17T12:00:00-05:00
draft: false
tags:
  - proxmox
  - kubernetes
  - k3s
  - longhorn
  - homelab
---

# Scaling Up: Building a Bulletproof K3s Cluster on Proxmox

I’ve been running services in my homelab for a while, but recently I hit a wall. I had a two-node Proxmox cluster that was supposed to be "High Availability," but in reality, it was just a split-brain headache waiting to happen.

If you know anything about clustering, you know that **two nodes is not a cluster**. If one node goes down, the survivor doesn't know if it's the leader or if it's isolated. Without a third vote (quorum), everything locks up.

So, I decided to do it right. I expanded to a three-node setup to run a proper **Kubernetes (k3s)** cluster with **Longhorn** for distributed storage. The goal? A system where I can rip the power cord out of any server, and my applications don't even blink.

Here is how I architected a production-grade cluster on consumer hardware.

## The Hardware: "The Micro Fleet"

My compute nodes are three Dell OptiPlex Micro units. They are small, quiet, and power-efficient, but they come with strict resource limits.

**Per Node Specs:**
* **CPU:** Intel Core i5-9500T (6 Cores @ 2.2 GHz)
* **RAM:** 24 GB DDR4 (Mixed 16GB + 8GB)
* **Storage:** NVMe SSDs
* **OS:** Proxmox VE 8

### The Memory Constraint
The biggest challenge was RAM. I had a mix of 8GB and 16GB sticks, resulting in 24GB per node. While Intel's "Flex Mode" allows this to run in dual-channel for the first 16GB, it’s still a tight budget for running a Hypervisor + Storage Layer + Kubernetes Control Plane + Worker Nodes.

I had to be extremely strategic about how I sliced up that 24GB pie.

## The Architecture: Avoiding the "Double CoW" Trap

I use **Longhorn** for my Kubernetes storage. Longhorn replicates block storage across the network so my pods can move between nodes freely.

However, running Longhorn on top of Proxmox with **ZFS** introduces a performance killer known as "Double Copy-on-Write."
1.  **Longhorn** writes data (CoW).
2.  **The VM Disk** writes to the host.
3.  **ZFS** writes to the physical SSD (CoW).

This write amplification can bring a 1Gbps network to its knees. To fix this, I adopted a hybrid approach:
* **Proxmox Host:** Runs on **ZFS** (for OS stability and snapshots).
* **VM Disks:** I configured the VMs to use `VirtIO SCSI single` with **Discard** and **SSD Emulation** enabled.
* **CPU Type:** Set to `Host`. This is critical! It passes the AES-NI instruction set directly to the VM, allowing Longhorn to handle encryption and checksums without burning 100% of my CPU.

## The "Split Role" Setup

Instead of running one giant VM per node, I decided to split the roles to protect the cluster's brain.

### 1. The Control Plane (Brains)
Kubernetes `etcd` is sensitive. If it gets starved of RAM, the whole cluster explodes. I isolated the Control Plane into its own VM.
* **vCPUs:** 2
* **RAM:** 4 GB
* **Taints:** `CriticalAddonsOnly=true:NoExecute` (This prevents heavy apps from accidentally landing here).

### 2. The Worker Node (Muscle)
This is where the actual containers and Longhorn storage engine live.
* **vCPUs:** 4
* **RAM:** 16 GB
* **Tuning:** Memory Ballooning **Disabled**. Java apps and databases *hate* ballooning, and I can't risk Proxmox stealing RAM from Longhorn during a write operation.

### The Math
* **Total RAM:** 24 GB
* **Worker:** -16 GB
* **Control Plane:** -4 GB
* **Remaining:** 4 GB (Reserved for Proxmox Host & ZFS ARC)

This leaves just enough headroom for the host OS without triggering OOM (Out of Memory) killers.

## The Result: True High Availability

After configuring the three nodes and joining them into the cluster, the state is finally healthy.

I now have a **Quorate** cluster. I tested this by shutting down `pve3` mid-operation. Proxmox detected the failure, and Kubernetes rescheduled the pods to `pve1` and `pve2`. Longhorn automatically rebuilt the degraded volume replicas once the node came back online.

### Key Takeaways for Homelabbers
1.  **Don't ignore CPU Types:** If you use storage software like Longhorn or Ceph inside a VM, setting the CPU type to `x86-64-v2` is a bottleneck. Use `host`.
2.  **Cap your ZFS ARC:** If you don't limit ZFS's RAM usage, it will eat 50% of your memory, starving your VMs. I capped mine at 2GB.
3.  **Split your Roles:** Even with limited RAM, separating the Control Plane from the Worker saved me from instability when my game servers spiked in usage.

## What's Next?
Now that the infrastructure is rock solid, the next step is visibility. I’m planning to deploy the **Prometheus and Grafana** stack to start graphing the actual resource usage of my pods. I want to see exactly how much of that 16GB worker allocation is actually being used.

But for now, I can finally sleep knowing my cluster won't die if I trip over a power cord.
