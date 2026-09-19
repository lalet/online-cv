---
layout: post
title: "Why I'll Never Use :latest Again: A Homelab Postmortem"
date: 2026-09-19
tags: [openclaw, kubernetes, argocd, telegram, selfhosted]
---

**TLDR:** My Telegram bot stopped working without telling anyone. A pod restart pulled a new `:latest` image that broke on old settings. Rolling back failed because the database had already upgraded and can't go backward. Fixing forward meant dealing with a permissions bug from my storage class, then a gap where two init steps weren't sharing all the same mount points. Four small pull requests fixed it all. Lesson: use exact image tags, not `:latest`. Treat any database change as one-way.

I sent a message to my self-hosted assistant on Telegram and got no reply. I thought it was just thinking. When I checked, the truth was worse: it hadn't answered anyone in 79 days.

## Finding the body

[OpenClaw](https://openclaw.ai/) has a CLI tool for checking status:

```
openclaw channels status
```

```
Telegram default: stopped, disconnected,
error:channel stop timed out after 5000ms
```

A health monitor in the background had tried to restart the Telegram connection every fifteen minutes for 79 days straight. It failed every time, but nobody knew. My Kubernetes health checks never caught it. They only check a `/healthz` endpoint that has nothing to do with Telegram. The pod looked healthy. One feature inside it just wasn't working.

The fix seemed simple: restart the pod. A new process can't inherit a stuck connection.

```
kubectl rollout restart deployment/openclaw
```

That's where it stopped being a five-minute job.

## Restarting into a landmine

My deployment uses `ghcr.io/openclaw/openclaw:latest`. Ten weeks had passed since the pod last restarted. When it restarted, it quietly pulled a much newer image. That new image rejected two settings the old version had written months earlier. They were removed, not renamed. Even OpenClaw's repair command wouldn't run. The CLI checks the whole config before letting anything execute.

So I went back to the exact version that had worked for 79 days. New error, worse one:

```
OpenClaw state database uses newer schema version 17;
this OpenClaw build supports 1.
```

Just running the newer binary, even after it failed, had already upgraded the local database's internal format on the way in. That upgrade can't be undone. I hadn't broken anything by trying to fix it. I'd broken the exit. Forward was the only way left.

## The permission error that made no sense

I had to use the new version. I removed the retired config keys. It died on:

```
EPERM: operation not permitted, fchmod
```

No file name, no path, just that. I kept the init container running with `sleep` so I could get inside and look around. Every file had the right owner. The directory holding them was owned by root with mode 755. The container ran without special privileges.

I'd set `fsGroup: 1000` to stop this, and it usually works. What I missed: my persistent volumes use [local-path-provisioner](https://github.com/rancher/local-path-provisioner), which is basically a directory on a node's disk pretending to be Kubernetes storage. Kubernetes skips the automatic ownership fix for this storage type. The volume had been owned by root since the day it was made. Nothing had ever needed to change that until now.

A container can create files in a directory it doesn't own, as long as the directory allows it. It can't change who owns the directory. That needs a real privilege it didn't have. The fix: one more init container that runs first, briefly as root, with one special capability, to chown the volumes, then exit.

## One more wall, then it worked

Fixed the ownership. The setup step ran clean this time. It migrated an old session file and rebuilt a plugin index. Then the main process failed anyway:

```
Legacy workspace setup state requires migration.
```

Turned out the setup step and the main process looked in different folders. I'd only mounted the config volume into the setup step, not the workspace or credentials volumes. So it fixed what it could see and missed the rest. I mounted all three the same way, and:

```
[telegram] starting provider
[telegram] isolated polling worker update received updateId=180306810 queued=25
```

Twenty-five messages, queued up and waiting. I named the bot Rocky, after the alien in Project Hail Mary. Felt fitting for something that got knocked out on day one and didn't get back up for 79 days.

<figure>
  <img src="https://media.giphy.com/media/vP6B55t5F41koebdvO/giphy.gif" alt="Rocky the alien from Project Hail Mary waving hello" loading="lazy">
  <figcaption>Rocky, saying hello. My bot's namesake.</figcaption>
</figure>

## What I'd tell past me

Pin your image tags for real. Don't rely on "latest is basically pinned because I don't restart often." Ten weeks turned it into two incompatible programs sharing a name.

Treat any version bump on something with a persistent database as one-way until you prove otherwise. Forward happened to be survivable. Rolling back a production database might not be.

And if [ArgoCD](https://argo-cd.readthedocs.io/) auto-fixes things for you, know it reverts a live `kubectl` edit within seconds of noticing it doesn't match git. Real fixes only stick once merged. One loophole: a filesystem change like the ownership fix survives even after ArgoCD reverts the pod spec that made it. That let me verify each fix worked before writing it into a pull request instead of guessing.

Four small pull requests later, the bot was talking to me again. It had been fine to talk to for 79 days. I just hadn't been listening.
