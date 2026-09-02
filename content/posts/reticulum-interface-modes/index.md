+++
date = '2026-09-02T14:08:20-06:00'
draft = false
title = 'Interface Modes: Keeping LoRa Airtime Sacred'
description = 'The structural fix for LoRa airtime flooding — Reticulum interface modes, the propagation matrix, and a mistake I almost made twice.'
tags = ['reticulum', 'networking', 'self-hosted']
ShowToc = true
TocOpen = true
+++

In the [last post](/posts/reticulum-lora-gateway/) I fixed an airtime problem by disabling the two
public interfaces that were flooding my LoRa radio with announce traffic. That worked, but it's not
a real fix — it's just noticing a specific problem and turning off specific things. Nothing stops
the next public interface I add from doing exactly the same thing. What I actually wanted was a
rule that applies automatically, regardless of what I connect in the future. That's what interface
**modes** are for.

## What modes actually do

Every Reticulum interface has an optional `mode` setting. Left unset, it defaults to `full` — full
discovery, meshing, and transport behavior. That's fine for a simple leaf node, but once a node is
running transport (which mine is, since Post 1) and has more than one kind of interface attached,
`mode` lets you tell Reticulum how each interface should behave differently rather than treating
every connection the same way.

There are six documented modes:

- **`full`** (default) — everything on: discovery, meshing, transport.
- **`gateway`** (`gw`) — everything `full` does, plus it actively resolves unknown paths on behalf
  of nodes connected through it. Useful for a public entrypoint, with one important catch: it's the
  interface *facing the clients* that needs `gateway` mode, not the one facing the wider network.
  I got that backwards in my head the first time I read it.
- **`access_point`** (`ap`) — stays quiet until someone actually uses it. No automatic announce
  broadcasting, shorter path expiry. Good for something like a radio interface that serves a wide
  area but only expects momentary visitors.
- **`roaming`** — for an interface that's physically mobile relative to the rest of the network
  (the manual's example: a vehicle with an external LoRa interface and an internal WiFi one — the
  external interface is `roaming`). Paths through it expire faster too.
- **`boundary`** — marks an interface connecting to a *significantly different* network segment.
  The manual's own example is almost exactly my situation: a LoRa-based network with a high-speed
  internet uplink — the internet-facing interface is the one that gets `boundary`.
- **`internal`** — marks interfaces that belong to the network *on the other side* of a boundary.

There's also a `point_to_point` mode defined in the code, but I couldn't find it explained anywhere
in the manual's prose — as far as I can tell it's real but essentially undocumented. Worth knowing
it exists, not worth relying on without testing it yourself.

## The part that actually matters: internal and boundary

`boundary` and `internal` are a pair, and they're the two modes that solve the airtime problem.
The rule: announces flow from `internal` interfaces *to* `boundary` interfaces, but not the other
way — `boundary` traffic never propagates back down onto `internal` interfaces. Devices on the
internal side can still resolve paths across the boundary when they need to (path requests still
work both ways), they just don't get flooded with the boundary side's constant announce chatter.

Mapped onto my setup: the LAN and the LoRa radio are `internal` — that's my network. The link out to
the wider public Reticulum network is `boundary` — that's the "significantly different network
segment" the manual is talking about. Announces from my own devices still reach the wider network
(so I stay discoverable), but the wider network's constant noise stops at the boundary instead of
continuing on to burn LoRa airtime.

```ini
[[Local LAN]]
  type = AutoInterface
  mode = internal

[[Heltec V3]]
  type = RNodeInterface
  mode = internal
  # ...radio config from the last post...

[[Public Uplink]]
  type = TCPClientInterface
  mode = boundary
  target_host = rmap.world
  target_port = 4242
```

## The mistake that's easy to make

I want to flag this because I actually made it, in my own notes, after already having the correct
config running: I described `boundary` mode as going *on the LoRa interface*, backwards from what
I'd actually set up. It's an easy mix-up — "boundary" sounds like it should mean "the edge of my
network," which feels like it should be the radio, not the internet link. But the mode describes
which side is the *foreign* network, not which side is physically at the edge. My LoRa radio is
mine; the internet uplink is what's crossing into someone else's much bigger, much noisier network.
`internal` goes on the stuff I own, `boundary` goes on the link to everyone else's stuff — no matter
which one is physically further from my house.

## A gotcha with `discoverable`

If you set `discoverable = yes` on an interface without also setting an explicit `mode`, Reticulum
doesn't leave it at `full` — it silently picks a mode for you: `access_point` for RNode/radio-type
interfaces, `gateway` for everything else, and logs a notice about it. Reasonable defaults, but
easy to miss if you weren't expecting your interface's behavior to change just from turning on
discoverability.

## Did it actually work?

Yes, though I confirmed it almost by accident rather than through a dedicated test. While comparing
behavior between two different radio setups on my portable rig, I swapped it over to talk to the
Heltec V3 interface directly and checked both directions — announces and messages both went
through cleanly. That wasn't the test I'd originally set out to run that day, but it's real,
first-hand confirmation that the `internal`/`boundary` split doesn't just theoretically work, it
was actually passing traffic correctly once live.

## Where this leaves things

Between Post 2's cleanup and this post's `internal`/`boundary` split, the LoRa radio's airtime is
now protected by a rule that holds regardless of what I connect to the wider network next — not
just by me remembering to disable things I've already caught being noisy. That's the gateway side
of this project basically done. Next up: putting a NomadNet page server behind it, so there's
actually something for people to browse to.
