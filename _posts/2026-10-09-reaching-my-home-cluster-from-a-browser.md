---
layout: post
title: "Reaching My Home Cluster From Any Browser"
date: 2026-10-09
tags: [homelab, cloudflare, kubernetes, security]
---

**TLDR:** I open my home cluster's dashboards from any browser, anywhere, without opening a single port on the home router. A small agent inside the cluster dials out to Cloudflare, and an identity check at Cloudflare's edge decides who gets in.

## The idea

Most remote access starts with "forward a port." I wanted the opposite: nothing listening on my home network at all.

1. **An outbound tunnel.** A small agent runs in the cluster and connects *out* to Cloudflare, holding that connection open. Requests for my hostnames travel back down it. The router never accepts an inbound connection, and my home address never appears in DNS.
2. **A guard at the edge.** [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/policies/access/) sits in front of each hostname. You prove who you are before a single packet reaches the cluster. Anyone else gets turned away at Cloudflare.
3. **Routing to the services.** Each hostname maps to an internal service, so one tunnel fronts every dashboard.

## The one gotcha

My home gateway quietly drops the fast UDP protocol the tunnel prefers, and the agent failed to connect with no useful error. Forcing it onto the TCP fallback fixed it. If a tunnel won't come up and the logs say nothing, try this first.

## What this is not

Browsers are for looking. For admin work like `kubectl`, I use a separate private mesh instead of a public URL. The full story of both doors is in [Part 5]({{ '/2026/06/24/homelab-part-5-remote-access.html' | relative_url }}).

## Why I like it

- No inbound ports, so nothing to scan.
- Authentication happens before traffic reaches my network.
- Adding a new dashboard is a hostname and a policy, not a networking project.
