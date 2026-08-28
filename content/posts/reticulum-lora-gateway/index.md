+++
date = '2026-08-28T10:42:07-06:00'
draft = false
title = 'Adding a Heltec V3 LoRa Gateway Over WiFi'
description = 'Giving my Reticulum gateway real long-range reach with a LoRa radio — and the airtime-flooding problem I ran into almost immediately.'
tags = ['reticulum', 'networking', 'self-hosted']
ShowToc = true
TocOpen = true
+++

In the [last post](/posts/reticulum-gateway-fundamentals/) I got a bare Reticulum gateway running:
transport enabled, one link out to the wider network. That's useful, but it's not doing anything a
plain internet connection couldn't do. This post is about giving it real, infrastructure-free
reach with a LoRa radio — and a problem I hit almost as soon as I turned it on.

## RNode firmware on a Heltec V3

Reticulum talks to LoRa hardware through **RNode** — not a specific product, but a firmware
protocol. Any supported ESP32-plus-LoRa board can run it, and a Heltec V3 is a cheap, common choice.
Flashing one is a single command once `rns` is installed:

```bash
rnodeconf --autoinstall
```

It walks you through picking the connected device and installs the right firmware for it — no
manual bootloader fiddling.

## Why WiFi instead of USB serial

The obvious way to connect an RNode is a serial cable (`port = /dev/ttyUSB0`), and that's the
default in most examples. But `RNodeInterface` also supports connecting over the network: point
`port` at a `tcp://` address instead of a device path, and the radio just needs to be reachable on
your LAN. That decoupling is the whole reason I went this route — the radio can sit wherever
antenna placement is best (a window, an attic, outside), instead of being tethered to whatever
machine happens to be running `rnsd`.

## The config

Here's the interface block, added to the gateway's `~/.reticulum/config` from Post 1:

```ini
[[Heltec V3]]
  type = RNodeInterface
  enabled = True
  port = tcp://<rnode-ip-or-hostname>
  frequency = <legal frequency for your region>
  bandwidth = 125000
  txpower = 7
  spreadingfactor = 8
  codingrate = 5
  announce_interval = 60
```

Two of those are placeholders on purpose, not values to copy:

- `port` — whatever address your RNode answers on once it's on your WiFi (a static IP, or the
  hostname it advertises). It's specific to your network, not something to hardcode from an
  example.
- `frequency` — LoRa frequency allocation is region-specific and legally regulated, and the RNS
  manual is upfront that it's on you to know your local rules before transmitting. Don't copy a
  frequency out of a blog post — use whatever's legal for your country's ISM band (commonly
  902-928 MHz in the US and Canada, 863-870 MHz across the EU, but check your local rules).

Everything else — bandwidth, tx power, spreading factor, coding rate — is just radio tuning, not
tied to where you live.

## The problem I didn't expect

Once the radio was up, I wanted to actually see what was crossing it — not just trust that it was
working, but watch real traffic. I had a ClockworkPi uConsole with a LoRa expansion board sitting
around, so the plan was: connect over LoRa (not WiFi), run a client on it, and watch.

That took a couple of false starts. The uConsole's built-in LoRa module is a bare radio chip with
no onboard microcontroller — there's no software bridge from "raw SPI radio" to anything Reticulum
understands yet, so that was a dead end without writing driver-level code. Next I tried a spare
RAK4631 board; its RNode firmware flashed fine but crash-looped every minute or so once running —
genuinely unstable firmware for that board, not something worth fighting. I gave up on it for this
purpose.

What actually worked was a second Heltec V3 — same board as the gateway radio, flashed the same
way — plugged into the uConsole over USB. From there I could run a client and watch the LoRa
channel directly, portably, wherever I wanted to stand.

## What the radio was actually carrying

What I found wasn't subtle: the gateway had two public `TCPClientInterface`s connected to
large, busy Reticulum communities. Every announce arriving over either one was getting rebroadcast
straight out over the LoRa radio — burning airtime on traffic that had nothing to do with my own
network.

That's not a bug, it's exactly what a transport node with `enable_transport = True` and every
interface left at the default `mode = full` is supposed to do: relay every announce it receives out
to every *other* interface. Useful for propagating your own network's topology — but it doesn't
distinguish "my local mesh" from "somebody else's public firehose." Anything wide enough to reach
the gateway gets rebroadcast onto the narrowest, most airtime-constrained link it has.

## The fix, for now

The immediate fix was blunt: disable both public interfaces. The flood stopped.

That's not a permanent answer — I still want *some* wide-network reach — but it's worth pairing
with a second, independent safety net regardless of what's connected. `RNodeInterface` supports
hard airtime caps:

```ini
  # airtime_limit_long = 1.5
  # airtime_limit_short = 33
```

Both are commented out by default. The short-term limit applies over a rolling ~15-second window,
the long-term one over a rolling 60-minute window, both expressed as a percentage of airtime used.
Turning these on doesn't fix a misbehaving interface, but it puts a hard ceiling on how much damage
one can do while you're not looking.

## A verification habit worth keeping

One thing that's stuck with me from debugging this: when you're not sure whether traffic is
actually flowing versus just being quiet, don't guess — force it. Trigger a manual "announce now"
from a client like Columba or MeshChatX, and watch whether the gateway's traffic counter for that
interface ticks up in response. If it does, the pipeline's fine and it was just quiet. If it
doesn't, something's actually broken. Much faster than waiting around hoping for organic traffic.

## Next up

Disabling the two interfaces that were causing trouble today doesn't stop the *next* public
interface from doing the same thing tomorrow. The real fix for this class of problem is Reticulum's
interface **modes** — `internal` and `boundary` — which let you draw a permanent line between "my
local network" and "the wider internet," so announces stop crossing that line automatically instead
of you having to notice and disable things by hand. That's next.
