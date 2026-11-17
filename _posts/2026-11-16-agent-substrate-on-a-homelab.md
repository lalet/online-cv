---
layout: post
title: "Running Agent Substrate on a Talos Homelab"
date: 2026-11-16
tags: [agent-substrate, kagent, kubernetes, talos, gvisor, ai-agents, selfhosted]
---

**TLDR:** Google's [Agent Substrate](https://github.com/agent-substrate/substrate) v1.0.0 runs on my Talos homelab. A sandboxed agent slept in 1.2 s and answered its first call after waking in 0.33 s with its state intact. An LLM agent used an API key it never held. kagent 1.0 runs a Claude coding agent on top. It took three beta feature gates, two tuned workers, and plain manifests in place of the official installer.

## The two pieces

Agent Substrate is a runtime for agents:

- Each agent (an "actor") runs in a [gVisor](https://gvisor.dev/) sandbox, or a microVM if your nodes have `/dev/kvm`.
- A small pool of worker pods hosts many actors. An idle actor is snapshotted to S3-compatible storage and its worker freed. The next request restores it on any free worker.
- Outbound traffic goes through an egress gateway that enforces a per-actor allow-list and can add credentials to requests, so the actor never sees the secret.

It went GA on GKE on 5 November 2026 ([Google Cloud blog](https://cloud.google.com/blog/products/containers-kubernetes/agent-substrate-available-on-gke)) and is headed to the CNCF Sandbox.

[kagent](https://kagent.dev/docs/kagent/latest/) is Solo.io's CNCF project for declaring agents as Kubernetes resources. In 1.0 every agent runs on Substrate, and a [Harness](https://kagent.dev/docs/kagent/latest/agents/agent-harness/) picks the agent loop: `kagent`, `codex`, `claude`, or your own image. There's no in-place upgrade from 0.x.

## What my cluster needed

Talos 1.14.2, Kubernetes 1.36.5, Cilium, Longhorn, ArgoCD, three amd64 workers. Substrate supports only the newest two Kubernetes minors, so I upgraded first.

### 1. Three beta feature gates

Substrate needs the pod certificate APIs, which are off by default in 1.36. I enabled `PodCertificateRequest`, `ClusterTrustBundle`, `ClusterTrustBundleProjection` and the `certificates.k8s.io/v1beta1` API on the API servers, controller managers and kubelets.

**Gotcha:** the kubelet won't start with `ClusterTrustBundleProjection` unless `ClusterTrustBundle` is also on. My first control-plane node crashlooped until I fixed it. Roll out one node at a time.

```bash
kubectl api-resources | grep -E 'podcertificaterequests|clustertrustbundles'
```

### 2. Sandbox nodes

On the two sandbox workers only, no reboot needed:

- `user.max_user_namespaces: "11255"`. Talos sets 0, and gVisor needs user namespaces.
- `proxy_arp` / `proxy_ndp` on, plus bigger neighbour tables, so traffic routes into the sandbox.
- A node label, `ate.dev/sandbox=true`, to pin the node agent (`atelet`) and worker pools.
- Privileged Pod Security on the Substrate namespaces, since workers need `SYS_ADMIN` and `NET_ADMIN`.

All of this comes from Sidero's post, [Run Agent Substrate on your own hardware with Omni](https://www.siderolabs.com/blog/run-agent-substrate-on-your-own-hardware-with-omni).

### 3. Plain manifests instead of the installer

The installer, `ate-setup`, only targets GKE and kind. Sidero's [gist of plain manifests](https://gist.github.com/rothgar/dd52bb0f80c7a28bb1aaed27d8af28e3) works with ArgoCD. My changes: Longhorn volumes, `atelet` pinned to sandbox nodes, and pointing Substrate at my cluster's existing token issuer. The gist changes the cluster's issuer instead, which would break every existing token.

It's 14 pods. Two things bit me:

- Postgres skipped creating its roles on first start and never retried. With an empty database, deleting the volume and pod fixed it.
- Almost nothing had requests or limits, and Postgres asked for 1 GiB while using 27 MiB. Right-sizing cut requests from 2.2 GiB to 0.9 GiB.

## What I tested

### Sleep and wake

The [counter demo](https://github.com/agent-substrate/substrate/blob/main/demos/counter/README.md) counts in memory, on a volume and on the root filesystem.

```bash
kubectl ate create actor my-counter-1 -a ate-demo-counter --template counter
kubectl ate proxy actor my-counter-1 -a ate-demo-counter
kubectl ate suspend actor my-counter-1 -a ate-demo-counter --mode=sync
curl -X POST http://localhost:8000   # wakes it
```

| Measured | Result |
|---|---|
| First golden snapshot | about 4 min |
| Suspend (`--mode=sync`) | 1.2 s |
| First call after waking | 0.33 s |
| State after waking | all three counts carried on |

The project claims under 500 ms to resume. For one agent on mini PCs, it held.

### Injected API key

A 40-line Python agent asks an OpenRouter model questions. It sends `Authorization: placeholder`, and the gateway swaps in the real key from a Secret the agent can't read. Egress allows only `openrouter.ai`.

It answered, its environment had no key, and after sleep and wake it still remembered the conversation. A prompt-injected agent can't leak a key it never had, and the key never lands in a snapshot.

### kagent on top

kagent 1.0.0 went in through ArgoCD with a `claude` Harness, an `AgentTemplate`, and an `Agent` called `coder` running Claude Sonnet 5.5:

```yaml
apiVersion: api.kagent.dev/v1alpha3
kind: Harness
metadata:
  name: claude
  namespace: kagent
spec:
  claude: {}
  substrate:
    workerPoolRef:
      name: kagent-default
    snapshotPolicy:
      location: s3://ate-snapshots/kagent/
```

I asked `coder` to write and run `fizzbuzz.py`, then `hello.sh`. Python and bash worked. A Node one-liner failed because the harness image has no Node, and the agent said so. For your own stack, bring your own image.

<figure>
  <img src="{{ '/assets/images/kagent/chat.png' | relative_url }}" alt="kagent chat with the coder agent: a successful bash call, then a failed call where node is not found, and the agent explaining that Node isn't installed" loading="lazy">
  <figcaption>The coder agent's tool calls. Bash works; Node isn't in the image.</figcaption>
</figure>

kagent's Substrate page shows the whole runtime, including the actors I made by hand.

<figure>
  <img src="{{ '/assets/images/kagent/substrate.png' | relative_url }}" alt="kagent Substrate page listing two worker pools, three ready actor templates including chat and counter, one of four actors running, and one of three workers busy" loading="lazy">
  <figcaption>Two worker pools, three templates, one actor awake.</figcaption>
</figure>

Because kagent agents are Kubernetes objects, ArgoCD manages them, which raw Substrate templates can't offer.

## Pros

- Idle agents don't hold memory, and waking is fast enough that nobody notices.
- Every agent is sandboxed, so running model-written code is the normal case.
- Keys injected at the gateway are the best answer I've seen to prompt injection.
- Real backing: GA on GKE, CNCF-bound, kagent 1.0 built on it, [NVIDIA Dynamo integrating](https://developer.nvidia.com/blog/agent-substrate-advancing-kubernetes-for-agents-at-scale/).

## Cons

- Needs Kubernetes 1.37 on managed platforms, since you can't flip feature gates there. GKE Standard only, not Autopilot.
- Outside GKE and kind you vendor and patch the manifests yourself.
- Raw Substrate templates and actors aren't CRDs, so plain GitOps doesn't cover them.
- Sandbox workers run with `SYS_ADMIN` and `NET_ADMIN`. Give them dedicated nodes.
- The bundled credential provider can read Secrets in any namespace. Narrow it first.
- No automatic suspend yet, and open connections drop when an agent sleeps.
- No GPU passthrough.
- Built for thousands of agents. With a handful, 14 pods is a lot of overhead.

## Not tested yet

- Whether a call to a non-allowed host actually fails
- Density and wake time with hundreds of actors
- Losing a worker mid-session
- Two-team isolation
- Whether kagent keeps the model key out of the sandbox

## Where it fits

The public use cases are coding agents and assistants that sit idle between turns. Google names [Nous Research](https://itbrief.co.uk/story/google-launches-open-source-agent-substrate-on-gke) as an early user. Next I want `coder` working on a real repo, with a GitHub token scoped to that repo and injected at the gateway.

## Links

- [Quickstart on kind](https://github.com/agent-substrate/substrate/blob/main/docs/get-started/quickstart.md)
- [Architecture doc](https://github.com/agent-substrate/substrate/blob/main/docs/concepts/architecture.md)
- [kagent suspend and resume](https://kagent.dev/docs/kagent/latest/about/substrate-runtime/suspend-and-resume/)
- [Solo.io: how Agent Substrate works](https://www.solo.io/topics/ai-infrastructure/how-google-agent-substrate-works)
