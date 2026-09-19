---
layout: post
title: "Why I'll Never Use :latest Again: A Homelab Postmortem"
date: 2026-09-19
tags: [openclaw, kubernetes, argocd, telegram, selfhosted]
---

I asked my self-hosted assistant on Telegram to check something for me. Nothing came back. I assumed it was thinking, maybe stuck on a slow model call, and went back to what I was doing. A few minutes later I actually looked, and the timeline got uncomfortable fast: the bot hadn't answered a single message in 79 days.

Not crashed. Not restarting. Just quietly, permanently not there, the whole time I'd been telling people about it in [an earlier post]({% post_url 2026-07-01-openrouter-openclaw-telegram-setup %}).

## Finding the body

[OpenClaw](https://openclaw.ai/) ships its own CLI for checking channel health, so that was the first thing I ran:

```
openclaw channels status
```

```
Telegram default: enabled, configured, stopped, disconnected,
in:73d ago, out:79d ago, mode:polling,
error:channel stop timed out after 5000ms
```

That line explained everything and nothing at the same time. Somewhere in OpenClaw's internals, a background health monitor had noticed the Telegram connection was down and tried to restart it. The restart needs to cleanly stop the old connection first. The stop never finished. So the health monitor waited, tried again, hit the same wall, and kept trying, forever, every fifteen minutes, for 79 days. Thousands of identical log lines, all saying "restarting," none of them actually restarting anything.

My Kubernetes liveness and readiness probes never caught this because they check a `/healthz` endpoint that has nothing to do with Telegram. The pod was healthy. The process was fine. One specific feature inside it had just quietly wedged itself on day one and stayed that way.

The fix seemed obvious: restart the pod. A brand new process doesn't inherit a stuck connection, since it never made the connection in the first place.

```
kubectl rollout restart deployment/openclaw
```

This is where it stopped being a five-minute fix.

## Restarting into a landmine

My deployment pulls `ghcr.io/openclaw/openclaw:latest`. That had seemed reasonable back in July. Ten weeks is a long time for a project that ships often, though, and the restart quietly pulled a much newer image than the one that had been running.

The new version refused to start. Its config validator rejected two settings the old version had written months earlier, one of which I'd added myself. Both were gone from the new schema entirely, not renamed, just retired. The init step that was supposed to fix this up automatically (`openclaw doctor --fix`) hit the same wall trying to run, because the CLI validates the whole config file before it lets any command run, doctor included.

Fine, I thought. Roll back to the exact version that had been working for 79 days. I found the old tag, pinned both containers to it, and redeployed.

New error. Worse one.

```
OpenClaw state database uses newer schema version 17;
this OpenClaw build supports 1.
```

Here's the part that actually surprised me. Just attempting to run a command with the newer binary, even the one that failed and exited immediately, had already touched the app's local database and upgraded its internal format on the way in. That upgrade doesn't reverse. The old binary can open a version-1 database and nothing newer. Once anything writes version 17 into it, there is no version of the old software that can read it again. I hadn't broken anything by trying to fix it. I'd broken the exit.

So there was only one direction left: forward, onto whatever the new version actually needed to run.

## The permission error that made no sense

Pinned to the new version for real this time, with the two retired config keys stripped out first. Init container started, ran its setup script, and died on this:

```
Doctor could not complete maintenance.
EPERM: operation not permitted, fchmod
```

No file name. No path. Just a bare permission error from a container that, as far as I could tell, owned every file it was touching.

I ended up patching the deployment to keep the init container alive on a plain `sleep` command instead of its real one, so I could shell into it and poke around by hand rather than guessing from the outside. Every file inside the config volume was owned correctly. The directory itself was not. It belonged to root, mode 755, and the container ran as a regular unprivileged user.

I'd set `fsGroup: 1000` in the pod's security settings specifically to fix this class of problem, and it usually does. What I hadn't accounted for is that `fsGroup` only reaches certain kinds of storage. My cluster's persistent volumes are backed by [local-path-provisioner](https://github.com/rancher/local-path-provisioner), which is really just a directory on a node's disk wearing a Kubernetes costume, and Kubernetes skips the automatic ownership fix for that storage type. The volume had been root-owned since the day it was created. It had never mattered before, because nothing had ever tried to change its permissions until this exact upgrade step came along.

An unprivileged container can create and delete files inside a directory it doesn't own, as long as the directory itself allows it. It just can't change who owns the directory. That takes an actual privilege the container didn't have.

The fix was a small extra step at the very front of startup: one more init container that runs briefly as root, with every capability stripped except the one that actually does the chowning, fixes ownership on the volumes, and exits. Everything downstream keeps running as the regular unprivileged user like before.

## One more wall, then it worked

Fixed the ownership, redeployed, watched the setup step run all the way through this time, migrating an old session file into its new format, rebuilding a plugin index, converting some cached data. Real progress. Then the main process itself failed to start:

```
Legacy workspace setup state requires migration.
Run "openclaw doctor --fix".
```

I'd already run that. Turned out the setup step and the main process weren't looking at the same set of folders. The bot has separate storage for its config, its workspace, and its saved credentials, and I'd only ever mounted the config one into the setup step. It patched what it could see and had no idea the other two existed. Mounting all three the same way fixed it for good.

Watched the logs. Gateway started. Plugins loaded. Then, finally:

```
[telegram] [default] starting provider (@lalclawbot)
[telegram] isolated polling worker update received updateId=180306810 queued=25
```

Twenty-five messages, queued up and waiting, some of them mine.

## What I'd tell past me

Pin your image tags. Really pin them, not "latest is basically pinned because I don't restart often." The version that had been running quietly for 79 days and the version I restarted into were separated by ten weeks and turned out to be two different, mutually incompatible programs wearing the same name.

Treat a version bump on anything with a persistent database as a one-way door until proven otherwise. I got lucky that forward was survivable. I would not have gotten lucky rolling back a production database.

And if you're using [ArgoCD](https://argo-cd.readthedocs.io/) with self-healing turned on, know what you're fighting. Any live `kubectl` edit I made to test a theory got reverted within seconds, the moment ArgoCD noticed it didn't match what was committed. That's the entire point of the tool and I wouldn't turn it off, but it means real fixes only stick once they're actually merged. The one exception worth knowing: a filesystem change like the ownership fix survives even after the pod spec that made it gets reverted, since ArgoCD only manages the Kubernetes objects, not what already happened on disk. That let me confirm each fix actually worked before writing it into a pull request for real, which is exactly backwards from how I'd normally want to test something, and also exactly what saved me from merging four more guesses.

Four small pull requests later, the bot was talking to me again like nothing had happened. It had, for 79 days. I just hadn't been paying attention, which might be the actual moral of this whole thing.
