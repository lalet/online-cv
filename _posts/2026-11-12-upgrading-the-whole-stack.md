---
layout: post
title: "I upgraded every layer of the homelab and mostly nothing went down"
date: 2026-11-12
tags: [homelab, kubernetes, talos, proxmox, longhorn, upgrades]
---

A few days, every layer, one rule: nothing goes offline that I didn't plan to take offline. Here is what I upgraded, what broke, and what I changed so it won't break again.

## What got upgraded

- The cluster add-ons came first: [Cilium](https://cilium.io/), cert-manager, [Longhorn](https://longhorn.io/), [CloudNativePG](https://cloudnative-pg.io/), the monitoring stack.
- Then [Talos Linux](https://www.talos.dev/) 1.13 to 1.14, with [gVisor](https://gvisor.dev/) added on the workers.
- Then [Kubernetes](https://kubernetes.io/) 1.35 to 1.36.
- Last, both [Proxmox](https://www.proxmox.com/) hosts. One had been up for 143 days and had over 200 pending updates.
- Every change went in as a pull request, applied by [ArgoCD](https://argo-cd.readthedocs.io/) or a short script.

## The order mattered

- Add-ons before the OS. A newer Talos with old add-ons is a combination nobody tested.
- One minor version at a time. Cilium went 1.18, 1.19, 1.20, with a check in between each.
- Control planes before workers, one node at a time, with an etcd health check after each.
- Before each node I asked: where is the database primary? Move it first, then touch the node.

## The Talos problems were all in my config

- I had pinned the Talos image server to a fixed IP "to be safe". The IP went stale and the installer download timed out. Normal DNS was the safe choice all along.
- The tool that generated my Talos configs had been archived, and its last version refused my config. I replaced it with a 50 line script around `talosctl gen config`. Nothing left to go unmaintained.
- The old tool named one encryption key differently. The value matched. The name didn't. One control plane briefly couldn't read Secrets. Lesson: when comparing configs, compare the names too, not only the values.
- Talos 1.14 brings a new config format. It also switches on a workload isolation setting that breaks Longhorn's storage connections. I pinned the old format for now.
- I trust a config change only when the dry run says "no changes" on every node.

## Storage was the real weak spot

- Longhorn kept its data on each VM's small system disk. Those disks were nearly full.
- Because space was so tight, some volumes had only one copy. A node with the only copy can't be drained, so upgrades stalled.
- When volumes broke, Longhorn refused to repair them because the disks had less than 25% free. I lost my metrics history.
- The fix was disks I already had. Each Proxmox host has a big, mostly empty SSD. I moved Longhorn there and kept two copies of every volume, one on each physical host.
- I capped Longhorn at about 200 GiB per SSD so it can't crowd out everything else.
- Old replica data now gets deleted automatically, and a weekly trim runs inside each volume.
- Drains now move a volume's last copy before evicting the node, instead of hanging.
- The disk layout lives in git as node labels, so a rebuilt node gets the same setup.

## The database didn't like the Kubernetes upgrade

- Upgrading Kubernetes restarts the kubelet on every node.
- On restart, the kubelet reapplied the pod's group ownership to the database volume. That changed the data folder's permissions.
- PostgreSQL refuses to start when its data folder is group writable, so the old primary couldn't rejoin.
- One setting fixed it for good: `fsGroupChangePolicy: OnRootMismatch`. If the top folder is already right, the kubelet now leaves everything alone.
- Catch: setting a security context in CloudNativePG replaces its defaults rather than adding to them. Restate the defaults or you lose them.

## Proxmox had two problems I hadn't noticed

- Disk space freed inside a VM never went back to the host. The VM disks had `discard` turned off, and nothing inside the VMs ran a trim. I turned on both: discard in Terraform, and a weekly trim in Talos.
- Inside the VM, the disk claimed trim support even with discard off. So that check proves nothing.
- The Terraform provider reboots VMs by default when a change needs it. One apply could have restarted five VMs at once. It's now off.
- Settings like `discard` only take effect after a full power cycle. A reboot from inside the VM isn't enough. My loop for each VM: shut down through Talos, which drains the node first. Then start the VM again with a targeted [OpenTofu](https://opentofu.org/) apply.
- The VMs' CD drives pointed at old ISO files that the next apply would delete. A VM with a missing ISO might not boot. Now the new ISO is downloaded and attached before the old one is removed.
- Patching the hosts themselves went one host at a time. I moved the database primary, shut down that host's VMs, ran the upgrade, and rebooted. The cluster kept running on the other host and the Raspberry Pis.

## One small service caused the most trouble

- [Garage](https://garagehq.deuxfleurs.fr/) is my S3 store. Logs and database backups go there.
- It was pinned to one node's local disk. When that host went down for patching, Garage went with it.
- [Loki](https://grafana.com/oss/loki/) then crash looped because it can't start without S3.
- The old database primary couldn't rejoin either, because it had to upload its leftover logs to Garage first.
- I moved Garage onto Longhorn. I shut it down, copied the data, checked every file's checksum, and started it again. Then I evicted it from its node to test. It came back on another node 31 seconds later with all its data.
- Lesson: a single copy in a quiet corner can block the services that do have copies.

## Process lessons

- Measure before you plan. I nearly put a new SSD on a host whose SSD pool was already fully allocated.
- A PR built on top of another unmerged PR can merge as an empty commit after a squash. Mine did, and Garage stayed off longer than planned. Branch from main after the first PR merges, then check that the change actually landed on main.
- Keep a rollback for every step. I saved each node's live config first and checked it restored cleanly.
- Check what's on the node, not just what the tool reports. Several "done" messages were wrong until I looked.

## What I'd do next time

- Buy a proper router. The whole LAN depends on the ISP's box. If a new ISP hands out a different address range, every node address breaks. With my own router and the ISP box in bridge mode, switching ISPs is just a cable swap.
- Move single-node services onto replicated storage before patching, not after. The 1.1.1.1 DNS fallback added this round is also staying.

Final tally: everything upgraded, three minutes of login errors when the database primary moved on its own, and about ten minutes without logs while Garage was off. Next up is gVisor sandboxes for AI agents. That is why I wanted Kubernetes 1.36 in the first place.
