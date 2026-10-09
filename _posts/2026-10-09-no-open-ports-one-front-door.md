---
layout: post
title: "No Open Ports, One Front Door"
date: 2026-10-09
tags: [homelab, cloudflare, tailscale, kubernetes, security]
---

I want to reach my home cluster from whatever browser is in front of me, and I don't want to forward a single port on my router to do it. There are two ways in, and both start at Cloudflare.

<figure>
  <img src="{{ '/assets/images/no-open-ports.svg' | relative_url }}" alt="Diagram: a browser sends a request to Cloudflare, which checks the login. Two things dial out to Cloudflare: a tunnel agent inside the home network, which serves the dashboards, and a jumpbox in AWS. The jumpbox reaches the home cluster over Tailscale. Neither network has inbound ports open." loading="lazy">
  <figcaption>Everything dials out. Nothing dials in.</figcaption>
</figure>

## Dashboards

A small agent inside the cluster opens a connection out to Cloudflare and keeps it open. When I visit one of my hostnames, Cloudflare sends the request down that connection. My router never accepts anything from the internet, and my home address doesn't show up in DNS.

Before a request gets near the cluster, [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/policies/access/) asks who I am. I can sign in through my own identity provider or with a one-time code. If I can't prove it, the request stops at Cloudflare.

Each hostname points at one internal service, so a single tunnel covers every dashboard. Adding one means a hostname and an access rule, not a networking project.

## A shell, still from the browser

Dashboards are for looking. For `kubectl` and `talosctl` I need a shell, and the cluster runs Talos, which has no SSH to log into anyway.

So I run a small jumpbox in AWS. Its security group has no inbound rules at all. Two things keep it reachable:

- It runs the same kind of outbound tunnel, so Cloudflare can hand it an SSH session in the browser. Access signs me in and issues a short-lived certificate, and the box trusts certificates from that one source. No keys on the machine I happen to be using, and no software to install.
- It is also on my [Tailscale](https://tailscale.com/) network, along with the cluster. That is how it reaches the cluster. The commands run from the jumpbox, and the cluster itself never faces the public internet.

The jumpbox is defined in Terraform, so if I doubt it I can rebuild it in a couple of minutes.

## The one gotcha

My home gateway silently drops the UDP protocol the tunnel prefers. The agent just wouldn't connect, and the logs gave no hint why. Forcing it onto TCP fixed it. If a tunnel refuses to come up and says nothing useful, check that first.

I covered the original version of this setup in [Part 5]({{ '/2026/06/24/homelab-part-5-remote-access.html' | relative_url }}).
