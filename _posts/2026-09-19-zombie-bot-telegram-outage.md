---
layout: post
title: "Why I'll Never Use :latest Again: A Homelab Postmortem"
date: 2026-09-19
tags: [openclaw, kubernetes, argocd, telegram, selfhosted]
---

I messaged my self-hosted assistant on Telegram and got nothing back. Figured it was thinking. When I actually checked, the truth was worse: it hadn't answered anyone in 79 days.

## Finding the body

[OpenClaw](https://openclaw.ai/) has a CLI for this:

```
openclaw channels status
```

```
Telegram default: stopped, disconnected,
error:channel stop timed out after 5000ms
```

A background health monitor had been trying to restart the Telegram connection every fifteen minutes for 79 days straight, and failing every single time, silently. My Kubernetes health checks never caught it because they probe a `/healthz` endpoint that has nothing to do with Telegram. The pod was healthy. One feature inside it just wasn't.

Obvious fix: restart the pod. A new process can't inherit a stuck connection.

```
kubectl rollout restart deployment/openclaw
```

That's where it stopped being a five-minute job.

## Restarting into a landmine

My deployment pulls `ghcr.io/openclaw/openclaw:latest`. Ten weeks had passed since the pod last restarted, and the restart quietly pulled a much newer image, one whose config validator rejected two settings the old version had written months earlier. Retired, not renamed. Even OpenClaw's own repair command couldn't run, since the CLI validates the whole config before it lets anything execute.

So I rolled back to the exact version that had been working for 79 days. New error, worse one:

```
OpenClaw state database uses newer schema version 17;
this OpenClaw build supports 1.
```

Just running the newer binary, even the attempt that failed, had already upgraded the local database's internal format on the way in. That upgrade doesn't reverse. I hadn't broken anything by trying to fix it. I'd broken the exit. Forward was the only direction left.

## The permission error that made no sense

Pinned to the new version for good, retired config keys stripped out. It died on:

```
EPERM: operation not permitted, fchmod
```

No file, no path, just that. I kept the init container alive on a plain `sleep` so I could shell in and look around. Every file was owned correctly. The directory holding them wasn't, root, mode 755, and the container ran unprivileged.

I'd set `fsGroup: 1000` specifically to prevent this, and it usually works. What I hadn't accounted for: my persistent volumes run on [local-path-provisioner](https://github.com/rancher/local-path-provisioner), basically a directory on a node's disk wearing a Kubernetes costume, and Kubernetes skips the automatic ownership fix for that storage type. The volume had been root-owned since the day it was created. Nothing had ever needed to change its permissions until now.

A container can create files in a directory it doesn't own, as long as the directory allows it. It can't change who owns the directory. That needs an actual privilege it didn't have. The fix: one more init container up front, briefly root, one capability, chown the volumes, exit.

## One more wall, then it worked

Fixed the ownership. The setup step ran clean this time, migrated an old session file, rebuilt a plugin index. Then the main process failed anyway:

```
Legacy workspace setup state requires migration.
```

Turned out the setup step and the main process weren't looking at the same folders. I'd only mounted the config volume into the setup step, not the workspace or credentials volumes, so it fixed what it could see and missed the rest. Mounted all three the same way, and:

```
[telegram] starting provider (@lalclawbot)
[telegram] isolated polling worker update received updateId=180306810 queued=25
```

Twenty-five messages, queued up and waiting.

## What I'd tell past me

Pin your image tags for real, not "latest is basically pinned because I don't restart often." Ten weeks turned it into two incompatible programs sharing a name.

Treat any version bump on something with a persistent database as one-way until proven otherwise. Forward happened to be survivable. Rolling back a production database might not be.

And if [ArgoCD](https://argo-cd.readthedocs.io/) self-heals for you, know it reverts a live `kubectl` edit within seconds of noticing it doesn't match git. Real fixes only stick once merged. The one loophole: a filesystem change like the ownership fix survives even after ArgoCD reverts the pod spec that made it, which let me verify each fix worked before writing it into a pull request instead of guessing.

Four small pull requests later, the bot was talking to me again. It had been fine to talk to for 79 days. I just hadn't been listening.
