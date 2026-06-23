---
layout: post
title: "Homelab, Part 4: GitOps with ArgoCD"
date: 2026-06-23
tags: [homelab, kubernetes, argocd, gitops]
---

[Part 3]({{ '/2026/06/22/homelab-part-3-kubernetes.html' | relative_url }}) got
the Talos cluster running. This post is how I avoid touching it by hand. The
whole cluster is driven by ArgoCD from a Git repo: I push a change, ArgoCD
notices, and the cluster reconciles to match.

## Why GitOps

The rule I wanted is that the repo is the source of truth, not the cluster. If I
change something live without committing it, ArgoCD drifts it back. If a node
dies, the repo rebuilds what was on it. Every change is just Git history, with a
diff and a revert.

ArgoCD runs in the cluster, watches the repo, and applies whatever it finds.

## One repo, App-of-Apps

Everything lives in a single repo I call `argo-home`. The pattern is
App-of-Apps: instead of pointing ArgoCD at thirty separate things, I point it at
one ApplicationSet that generates an app per component from the repo's folders.
Add a folder, get a new managed app. Roughly:

```
argo-home/
  argocd-apps/   the ApplicationSet and app definitions
  k8s-apps/      the workloads each app deploys
```

So onboarding a new service is a pull request, not a kubectl session.

## Bootstrapping the bootstrapper

There is a small chicken-and-egg here: ArgoCD has to exist before it can manage
anything, including itself. The first install is manual, and right after that
ArgoCD adopts its own manifests, so from then on even ArgoCD upgrades itself
through Git. One thing that mattered: turning on server-side apply, because the
CRD-heavy manifests blow past the old client-side limit otherwise.

## What it manages today

Right now ArgoCD keeps a small set of things in sync, all healthy:

- argocd (it manages itself)
- cilium, the CNI from Part 3
- cert-manager
- csi-driver-nfs and local-path-provisioner, the storage from Part 1

The short list is on purpose. I would rather have a few things rock solid than a
sprawling dashboard of half-broken apps.

## Secrets, briefly

Secrets don't sit in Git as plaintext. They're encrypted with SOPS and an age
key, so the repo can still be the source of truth without leaking anything.
Wiring SOPS decryption cleanly into ArgoCD is still in progress on my side, and
it's currently what's holding back a couple of apps. I'll write that part up
when it's actually done instead of pretending it's finished.

## What bit me

- Server-side apply isn't optional. Without it, the big CRD manifests fail on the
  annotation size limit.
- ApplicationSet ordering on a fresh cluster: namespaces have to exist before the
  apps that land in them, so bootstrap order matters more than it does once
  everything is steady.
- I started with MetalLB for load balancing and later pulled it out. Cilium
  already does that job, so MetalLB was just one more moving part.

## What's next

The cluster runs itself, but I still reach most of it over the local network.
Part 5 is about exposing services properly: single sign-on with Authentik, and
getting in from outside without poking holes in the router.

It's all learning, and the repo keeps getting longer.
