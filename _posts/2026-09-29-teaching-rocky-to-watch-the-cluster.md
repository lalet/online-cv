---
layout: post
title: "Teaching Rocky to Watch the Cluster"
date: 2026-09-29
tags: [openclaw, kubernetes, cilium, rbac, selfhosted]
---

**TLDR:** I gave Rocky, my self-hosted Telegram bot, read-only access to my Kubernetes cluster so it could flag actual problems instead of me finding out the hard way. Four bugs stood between writing the config and a single `kubectl get pods` working: a missing binary, a config file that silently reset itself, a firewall rule checking the wrong port, and a permission that didn't cover what I thought it did.

Rocky already opens pull requests on my GitHub repo. Giving it read access to the cluster felt like the obvious next step. I figured it was an afternoon of work: a read-only role, a Kubernetes-flavored tool plugin, done by dinner.

## The plan

Rocky runs with tight permissions: no secrets, no write access, a default-deny network policy. Cluster monitoring meant adding:

- A read-only role bound to its service account, secrets excluded
- An MCP tool plugin so it could actually query the cluster
- A network policy rule to reach the API server
- A written rule on what it's allowed to act on unsupervised, since read access plus merge rights on its own pull requests needed some ground rules

## Bug one: no such binary

The MCP server wraps `kubectl` directly. Its tools are named `kubectl_get`, `kubectl_describe`, and so on. Rocky's container doesn't ship a `kubectl` binary at all.

Fix: an init container downloads a static binary onto a shared volume, and `PATH` gets extended to include it. The shared directory couldn't be `/usr/local/bin`, though: that's where the base image keeps `node`, `npm`, and Rocky's own binary, and mounting a volume there wipes the whole directory instead of adding to it. Found that one by breaking the container outright.

```yaml
initContainers:
  - name: install-kubectl
    image: curlimages/curl
    command: ["sh", "-c", "curl -sfL https://dl.k8s.io/release/v1.35.6/bin/linux/amd64/kubectl -o /kubectl-bin/kubectl && chmod +x /kubectl-bin/kubectl"]
    volumeMounts:
      - { name: kubectl-bin, mountPath: /kubectl-bin }   # not /usr/local/bin
```

## Bug two: the setting that wouldn't stick

Getting that `PATH` into the tool plugin's own environment needed its own fix. Rocky's config is a JSON file on disk, and anything I wrote into it during pod startup vanished by the time the pod was ready. Something in the app's own boot sequence rewrites the file afterward. The fix had to run after the app was fully up, not before.

```bash
until curl -sf http://127.0.0.1:8080/readyz; do sleep 2; done
node -e "... patch the config here, not in an initContainer ..."
```

## Bug three: right door, wrong number

With the binary and environment sorted, `kubectl` stopped erroring and started hanging instead. No response, just silence.

A silent hang usually means a firewall. My network policy engine confirmed it: the connection was being dropped, and dropped packets don't send anything back. I'd written the rule against the API server's internal service port, but the network layer swaps that for the real backend server before deciding whether to allow the connection, and the real server listens on a different port than the service does. One number, wrong the whole time.

```
dial tcp <api-server>:443: i/o timeout
```
```yaml
egress:
  - toEntities: [kube-apiserver]
    toPorts:
      - ports: [{ port: "6443", protocol: TCP }]  # the real port, not 443
```

## Bug four: a permission is not a permission

`kubectl get pods` worked. `kubectl logs` didn't: flat permission denied, even though the same rule should have covered it. Reading a pod's logs turns out to be its own distinct permission in Kubernetes, separate from reading the pod object itself.

```yaml
rules:
  - apiGroups: [""]
    resources: [pods/log]   # separate from "pods" -- easy to miss
    verbs: [get]
```

## What actually made this fast

Two habits saved most of the afternoon.

When a fix looked plausible but the behavior stayed broken, I stopped trusting the config and ran the actual tool call Rocky would make. Config being correct and behavior being correct are two different claims, and I declared victory on the wrong one more than once.

The other: this cluster's GitOps setup reverts any change made directly against the live cluster unless it's committed too, usually within a minute. I'd fix something, watch it work, then come back convinced it had broken again, when it had just been rolled back. What worked: confirm with a fast manual test, then commit immediately.

## Where it landed

Rocky can now list pods, read logs, check deployment health, and watch the rest of the cluster on a half-hour timer, all read only. It only messages me when something's actually wrong.

<figure>
  <img src="https://media.giphy.com/media/vP6B55t5F41koebdvO/giphy.gif" alt="Rocky the alien from Project Hail Mary waving hello" loading="lazy">
  <figcaption>Rocky, watching the cluster now instead of just chatting.</figcaption>
</figure>
