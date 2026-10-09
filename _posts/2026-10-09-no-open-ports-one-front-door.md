---
layout: post
title: "No Open Ports, One Front Door"
date: 2026-10-09
tags: [homelab, cloudflare, kubernetes, security]
---

I want to open my home cluster's dashboards from whatever browser is in front of me, and I don't want to forward a single port on my router to do it. Here is how that works.

The cluster calls out instead of waiting to be called. A small agent running inside it opens a connection to Cloudflare and keeps it open. When I visit one of my hostnames, Cloudflare sends the request down that connection. My router never accepts anything from the internet, and my home address doesn't show up in DNS.

Before a request gets anywhere near the cluster, [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/policies/access/) asks who I am. If I can't prove it, the request stops at Cloudflare and my network never sees it.

<figure>
  <img src="{{ '/assets/images/no-open-ports.svg' | relative_url }}" alt="Diagram: a browser sends a request to Cloudflare, which checks identity. The tunnel agent inside the home network dials out to Cloudflare, and traffic then flows from the agent to the dashboards. No inbound ports are open on the home network." loading="lazy">
  <figcaption>The agent dials out. Nothing dials in.</figcaption>
</figure>

Each hostname points at one internal service, so a single tunnel covers every dashboard. Adding a new one means adding a hostname and an access rule. It doesn't mean touching the network.

The only snag was my home gateway, which silently drops the UDP protocol the tunnel prefers. The agent just wouldn't connect, and the logs gave no hint why. Forcing it onto TCP fixed it. If a tunnel refuses to come up and says nothing useful, check that first.

This path is for looking at things. For real admin work like `kubectl`, I use a private mesh network instead of a public URL. I wrote about both in [Part 5]({{ '/2026/06/24/homelab-part-5-remote-access.html' | relative_url }}).
