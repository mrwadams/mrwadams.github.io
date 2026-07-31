---
layout: post
title: "A three-screen e-ink panel for my homelab"
date: 2026-07-30
categories: [homelab, raspberrypi]
tags: [raspberry-pi, eink, inkyphat, homelab, systemd, security, projects]
---

I had a Raspberry Pi 1 Model A+ sitting in a drawer and a Pimoroni InkyPHAT that had been on a shelf even longer. The Pi is ancient: a single ARMv6 core and 512MB of RAM, too weak to run Docker and too slow to be a general-purpose anything. But an e-ink display doesn't need much. It draws a static image once, holds it with no power, and waits. That's a good match for a machine that can barely do anything else.

So I turned the pair into a small desk panel that answers three questions I check throughout the day anyway:

1. **What's the top item in today's AI/security briefing?**
2. **Is the homelab healthy?**
3. **Do any of my open-source repos need me right now?**

One screen each, rotating on a timer. The interesting part isn't the display but how the panel gets its data without holding a single credential, which is the piece I'd reuse elsewhere.

![The e-ink panel on a 3D-printed stand, showing the projects screen](/assets/images/inky-panel.jpg)
*The panel on the projects screen. A red dot marks any repo that needs me, and the bottom line is the oldest pull request waiting on a review.*

## The three screens

The InkyPHAT is 212×104 pixels, three colours: white, black, and a single red accent. Red means *look at this*, and everything else is black on white. There's no room for chrome, so every screen is one question and its answer.

The **projects** screen is the newest and my favourite. It's a compact row per repo:

```
 my open source                          19:50Z
 ────────────────────────────────────────────
 ● attackgen                       PR 1   iss 12
   stride-gpt                      PR 4   iss  2
 ● honeyagents                     PR 1   iss  0
   takedown-gpt                    PR 0   iss  1
 ────────────────────────────────────────────
 > review honeyagents #1
```

A red dot next to a repo means it needs me. The action line at the bottom is the single most important thing across all four repos. Here, that's the oldest incoming pull request waiting on a review. If nothing needs attention, the line reads `nothing needs you` and I can ignore the panel entirely, which is the point.

The other two screens are a **briefing** headline (with a scannable QR code of the article URL, so the panel hands off to my phone) and a homelab **health** card (eight signal dots for backup, containers, internet, the agent host, certificate expiry and so on, black for OK and red for attention).

## The rule: the panel is a dumb display

The panel holds **no credentials**. It never authenticates to GitHub, it never logs into another host, and it can't write anything anywhere. It reads feeds and paints them.

This matters because I treat the panel as one of the less trusted devices on my network. It's an unattended box associating to Wi-Fi over a flaky USB dongle, running an OS I update far less often than my main hosts. If it were compromised, I want the blast radius to be "an attacker can read data that was already visible on the LAN", nothing more.

So the credentials and the API access live on an always-on services host (a Raspberry Pi 5), and the panel only ever pulls the *results*:

<pre class="mermaid">
graph TD
    GH["GitHub REST API\n(4 public repos)"]
    subgraph SVC["services host (trusted, always-on)"]
        AGG["projects-agg.py\nstdlib only, cron */15\nholds the read-only PAT"]
        BRF["daily briefing\n+ ops-health aggregators"]
        FILES["~/projects/health/\nprojects.json · health.json"]
        SFS["static-file-server\n(:8081, read-only, LAN)"]
    end
    subgraph INKY["the e-ink panel (low-trust)"]
        PANEL["panel.py\nsystemd timer\nNO credentials"]
    end

    GH --> AGG
    AGG --> FILES
    BRF --> FILES
    FILES --> SFS
    SFS -->|"plain-HTTP GET"| PANEL

    style GH fill:#2d333b,stroke:#768390,color:#adbac7
    style AGG fill:#2d333b,stroke:#768390,color:#adbac7
    style BRF fill:#2d333b,stroke:#768390,color:#adbac7
    style FILES fill:#2d333b,stroke:#768390,color:#adbac7
    style SFS fill:#2d333b,stroke:#768390,color:#adbac7
    style PANEL fill:#2d333b,stroke:#768390,color:#adbac7
</pre>

A small aggregator on the services host polls the GitHub API every fifteen minutes, decides what matters, and writes a tiny JSON file into a directory that a read-only static file server already publishes on the LAN. The panel does an HTTP GET of that file and renders it.

The briefing screen is the one exception: it needs a file that *isn't* on the LAN feed, so instead of an SSH login it uses a **forced-command key**. The key is authorised on the services host as `command="cat …/latest.md",no-pty,…`, which means it can only ever return that one file and can never open a shell. Least privilege, right down to a single `cat`.

The nice side effect is that this pattern cost almost nothing to add. The static file server and the health feed already existed for a [Galactic Unicorn status board](https://shop.pimoroni.com/products/galactic-unicorn) I built earlier. Adding the projects screen was one more small JSON file in a directory that was already being served: no new container, no new port, no config change.

## The aggregator, and what "needs me" means

The GitHub aggregator is deliberately boring: about a hundred lines of standard-library Python, no pip dependencies, no LLM, no container. It reads a read-only [fine-grained personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) (scoped to only those four repos, permissions Metadata + Pull requests + Issues, all read-only), fetches each repo's open PRs and issue count, and writes the result atomically so the panel never reads a half-written file.

The harder question is what should count as *needs me*. The obvious query, GitHub's `review-requested:mrwadams`, was never going to work for me: I don't assign myself as a reviewer on my own projects, so it's always empty. The signal that fits is simpler: **an open, non-draft PR that someone other than me opened**. Someone showing up with a contribution is the "come and look" event. Bots are excluded, and the action line surfaces the *oldest* such PR first, on the theory that the person who's been waiting longest deserves a reply.

This does leave one gap. Fine-grained PATs have no "Checks" permission, so the aggregator's attempt to read CI status returns a 403 and it drops the CI flag. Failing-CI detection is best-effort, and the incoming-PR signal is the reliable one. Proper CI flags would need a classic PAT or a GitHub App, not a wider scope on this token.

## Rotation on a timer

There's a single dispatcher script, `panel.py`, run by a systemd oneshot service on a timer:

```ini
# inky-panel.timer
[Timer]
OnCalendar=*:0/20        # every 20 minutes, on :00 / :20 / :40
OnBootSec=120
Persistent=true
```

Each run advances one screen. A tiny state file records which screen painted last, so the next run picks the one after it and round-robins through `[briefing, status, projects]`. If a screen's feed can't be fetched, it falls through to the next one rather than showing stale or blank data. If everything fails it leaves the panel untouched, because e-ink holds its last good image with no power anyway. Three screens, one advance per tick, means each screen comes round roughly hourly.

## Gotchas

A few things on this board cost me real time:

- **A warm reboot kills the Wi-Fi.** The USB dongle is a Broadcom BCM43143 with no onboard flash, so its firmware is downloaded over USB on every boot. The Pi 1 A+ can't power-cycle its USB port on a soft `reboot`, so the chip never re-initialises and `wlan0` never comes back, leaving a headless box unreachable. Reboots have to be a **cold** power-cycle: pull the plug and put it back.
- **`sync` before you pull the plug.** Because reboots are cold power-cuts, any edit to the SD card needs a `sync` first, or the write is lost. I lost a config change to a cold-cycle that happened before the card had flushed.
- **The SPI overlay is not optional.** With `inky` 2.x you need `dtoverlay=spi0-0cs` in `config.txt`, not just `dtparam=spi=on`. Plain SPI lets the kernel claim GPIO8 as chip-select, which collides with the library and fails with *"Chip Select line 8 claimed by spi0 CS0"*. The overlay frees that line so the driver can manage it.
- **A full refresh takes ~25 seconds** on this board, and two `panel.py` runs will fight over the GPIO lines if they overlap. I learned not to trigger a manual render within half a minute of a scheduled tick.

## How it's going

It's been running for a couple of days and it's already changed a small habit: I glance at the panel instead of reaching for my phone to check whether a repo has a new PR. Because red only ever means *something wants you*, a panel that's all black is genuinely reassuring. It's the homelab equivalent of a quiet inbox.

The reusable idea is the split between the two boxes. The display only renders feeds and holds no secrets, and the trusted host does the privileged work and exposes nothing but a read-only file. I'd build the next glanceable panel the same way, and I don't have to treat this one as precious: if it dies, gets replaced, or someone gets onto it, nothing important is riding on it.

Next on the list is probably a fourth screen (home energy, or the next family calendar event), but the nice thing about the rotation design is that adding one is a new feed and a new render function, and precisely zero new trust.
