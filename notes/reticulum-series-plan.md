# Plan: Reticulum infrastructure blog post series

## Context

This is the first content series for the `codemonkeybr.github.io` Hugo blog. It's based on real Reticulum/LXMF/NomadNet infrastructure the user built to support the `game-center` NomadNet BBS project — but the series itself is scoped purely to the underlying network infrastructure, not the games. The user wants to teach readers how to stand up their own gateway (with LXMF), NomadNet page server, and LoRa gateway, integrated with the public `rmap.world` network — advising on best-practice architecture rather than mirroring the user's own hardware-constrained two-Pi split.

Mid-series, the user flagged an important extra topic: RNS interface **modes** (`internal`/`boundary`/etc.), which they used to stop a public WAN interface from flooding their LoRa radio's limited airtime with announce traffic. Research confirmed this has enough depth (6-7 modes, an asymmetric propagation matrix, a silent auto-mode-inference gotcha, a real "we almost configured it backwards" trap, and adjacent airtime-limiting knobs) to be its own post, and it grew directly out of the LoRa-gateway airtime problem — so it's sequenced right after that post rather than folded into Post 1.

Research is done (via background Explore agents digging through `~/dev/game-center/`, its Claude Code memory/session transcripts, the bundled RNS manual, and the installed `RNS` package source). This plan defines the series structure and per-post outline with source citations; actual post drafting happens after this plan is approved, one post at a time, starting with Post 1.

## Series structure (4 posts)

1. **Reticulum fundamentals + standing up a gateway node**
2. **Adding a Heltec V3 LoRa gateway over WiFi**
3. **Interface modes: keeping LoRa airtime sacred** *(new)*
4. **NomadNet page server** *(renumbered from 3)*

Each post stands alone but cross-links to the others in the series (intro/outro links, consistent series tag e.g. `tags = ['reticulum']`).

**Domain correction:** it's `rmap.world`, not `rnmap.world` — use the correct spelling throughout.

**Scope boundary:** no mention of game-center's games/BBS content, and no mention of the MeshCore/RAK4631 bridge on `.248` (a different protocol/audience per user's explicit call) — pure Reticulum/LXMF/NomadNet/LoRa-via-RNode.

## Post 1 — Reticulum fundamentals + standing up a gateway node

- What Reticulum is, transport nodes vs. leaf nodes — why this distinction matters before touching hardware.
- Installing `rnsd`, minimal config walkthrough: `enable_transport`, basic interfaces.
- Connecting outward: `TCPClientInterface`/`BackboneInterface`, joining a public gateway (e.g. `rmap.world`) so the node has reach beyond the LAN.
- Architecture advice: recommend a single reasonably-specced device (Pi 4/5) by default. Use the user's real Pi 3 B numbers as the "when you'd actually need to split" case study — not as the recommended path.
- Verifying it's alive: `rnstatus`.

**Sources:** RNS manual (`/opt/Reticulum MeshChatX/resources/backend/public/reticulum-docs-bundled/current/manual/_sources/interfaces.rst.txt`, `networks.rst.txt`); `~/dev/game-center/docs/migration.md` (concrete Pi 3 B load-avg ~9 / 84% RAM / 731MB swap vs. Pi 3 B+ "nearly idle" numbers, and the 5-phase migration runbook); memory file `~/.claude/projects/-home-tesso-dev-game-center/memory/project_reticulum_infra_2026-08.md`.

## Post 2 — Adding a Heltec V3 LoRa gateway over WiFi

- RNode firmware on the Heltec V3; why WiFi/TCP (`port = tcp://...`) instead of USB serial.
- `RNodeInterface` config: real frequency/bandwidth/spreadingfactor/codingrate/txpower values from the live setup.
- Wiring it into the gateway node from Post 1.
- The airtime problem, introduced honestly: diagnosing pollution empirically by building a LoRa-sniffer rig (a second RNode on a uConsole) — teach readers the technique.
- The immediate fix: disabling public TCPClientInterfaces that were flooding the radio (`RNS Android Sideband App Server`, `Birdsnet BR`), plus the RNode's own `airtime_limit_long`/`airtime_limit_short` hard caps as a physical-layer backstop.
- End on a hook into Post 3: this stops the interfaces you know about, but what about the WAN uplink you *want* connected?

**Sources:** live `.248` config for the Heltec V3 `RNodeInterface` block (from `docs/migration.md` and the 2026-08-17 session transcript's `cat ~/.reticulum/config`); memory files `project_reticulum_infra_2026-08.md` for the sniffer-rig diagnosis narrative; RNS manual's RNode example + `airtime_limit_long/short` comment block (`interfaces.rst.txt` ~line 578-658).

## Post 3 — Interface modes: keeping LoRa airtime sacred *(new)*

- The deeper problem: even after Post 2's interface cleanup, any future WAN-facing link could flood the radio again. Need a structural fix, not a per-interface whack-a-mole.
- RNS interface modes explained, grounded in the manual + source: `full` (default), `gateway`, `access_point`, `roaming`, `boundary`, `internal` (and a note that `point_to_point` exists in code but is essentially undocumented).
- The propagation matrix (who-announces-to-whom), simplified for readers — key point: `boundary` blocks announces from reaching `internal`, but `internal` still reaches `boundary`, and paths remain resolvable across the boundary via path requests even when announces don't cross it.
- The real fix applied: `mode = internal` on `.248`'s LAN/LoRa-facing interfaces (Local LAN, LAN TCP Server, Heltec V3), `mode = boundary` on the WAN uplinks (`___ecce_collective_gateway__`, `RMAP World`).
- Callout: the manual's own "Please note!" warning that `gateway` mode goes on the *client-facing* interface, not the network-facing one — and the real trap from the user's own memory files, where a later session almost described `boundary` going on the Heltec V3 itself (backwards from what was actually configured). Good "easy to get backwards" anecdote.
- Gotcha: `discoverable = yes` without an explicit `mode` silently auto-assigns one (`access_point` for RNode/radio types, `gateway` otherwise) — surprising if you didn't expect it.
- Validation: this was live-verified, not just planned — cite the 2026-08-17 SSH config pull showing the modes already set, and the 2026-08-18 confirmation (user directly testing the uConsole against the Heltec V3 interface and confirming two-way announce/communication). Frame honestly: the validation was a byproduct of an unrelated debugging session, not a dedicated mode test, but it's real first-hand confirmation.

**Sources:** RNS manual `interfaces.rst.txt` (Interface Modes section, ~line 1279+, per-mode prose, propagation matrix from `understanding.rst.txt`); `RNS/Interfaces/Interface.py` and `RNS/Transport.py` source (mode constants, `outbound()` and `path_request()` enforcement logic) at `~/.local/lib/python3.12/site-packages/RNS/`; session transcripts `~/.claude/projects/-home-tesso-dev-game-center/769f3dd6-ddb7-4ada-af32-cea01366bd25.jsonl` (2026-08-17 live config pull + diff) and `~/.claude/projects/-home-tesso-dev-RNS-Over-MeshCore-uconsole-aiov2/cf7daef4-ff06-4431-8c73-49eccd3467f2.jsonl` (2026-08-18 user's own confirmation quote); memory file `project_meshcore_reticulum_bridge.md` for the "same trick" backwards-attribution anecdote.

## Post 4 — NomadNet page server

- `nomadnet` + `lxmd` basics; leaf-node config connecting back to the Post 1 gateway.
- Writing a first micron (`.mu`) page.
- Real example: a status page that shells out to `rnstatus -j` / `lxmd --status` to show live gateway health.
- Deploying it (systemd) and testing the LXMF address end-to-end.

**Sources:** `~/dev/game-center/pages/game_center/status.mu` (as a technique example, not as game-center content itself — extract the *pattern*, not the game-specific page); `~/dev/game-center/CLAUDE.md` micron markup section; `docs/migration.md` systemd unit file.

## Drafting order and process

Draft one post at a time, starting with Post 1, as a new file under `content/posts/` (Hugo TOML front matter, `draft = true` until reviewed). Check in with the user after each draft before moving to the next — don't draft all four before getting feedback on the first. Use the `posts` archetype (`archetypes/posts.md`) as the front-matter template. Keep the "Git conventions" (no AI attribution) and other repo conventions from `CLAUDE.md` in mind — not relevant to post content itself, but relevant when we eventually commit.

## Verification

- Each drafted post: build with `hugo --gc --minify` (or `hugo server --buildDrafts` for live preview) to confirm it renders — already the established workflow for this repo.
- Technical accuracy: config snippets and claims in the post should trace back to the sources cited above (real config blocks, real manual quotes, real transcript evidence) rather than being paraphrased from memory — spot-check against source files while drafting, don't rely purely on the research agents' summaries for exact wording/config values.
