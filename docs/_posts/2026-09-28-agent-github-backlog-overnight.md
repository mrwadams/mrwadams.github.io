---
layout: post
title: "A month of letting Claude Code work my GitHub backlog overnight"
date: 2026-09-28
categories: [ai, security]
tags: [agents, ai, security, claude-code, github, automation, sandboxing, cron]
---

I have a mini PC in my study that works through my GitHub backlog overnight. A cron job picks an issue I've labelled `ready-for-agent`, has Claude Code work on it in a disposable sandbox, checks the result, and opens a draft pull request. By morning, there can be something for me to review. Nothing merges until I've reviewed it.

By 28 September, it had opened 19 pull requests against [attackgen](https://github.com/mrwadams/attackgen), my incident response scenario generator, and I'd merged all 19. I added [stride-gpt](https://github.com/mrwadams/stride-gpt), my threat modelling tool, on 19 September. That had six pull requests: two merged, two waiting for review, and two closed.

One of those attackgen pull requests showed me a gap in the setup. Its code-level checks were green, but a change intended to remove browser console warnings hadn't worked. I hadn't set up the agent to check the running app in a browser, so it had no way to confirm the fix where the problem actually appeared.

The loop has spent plenty of time waiting for me, too. Across four weeks of logs, 64 implementation runs stopped because the repo already had three open agent PRs. That's the review limit I've set for each repo, because it's how many unreviewed changes I can realistically get through. The agent waits until I make room.

Knowing the loop can pick up an issue overnight has changed what I put in the backlog. I write every issue it works on, and small improvements I would previously have left alone are now worth filing.

Once an issue is ready, the loop has to handle the supervision I'd normally provide while watching Claude Code. I had to turn that into code: choosing the next issue, checking the result, limiting the agent's access, and noticing when it stopped running.

My starting point was Geoffrey Huntley's [Ralph technique](https://ghuntley.com/ralph/), which repeatedly runs a coding agent in a shell loop. I built the selection, checks and monitoring around that idea so each attempt would leave me either a draft pull request or enough information to investigate a failure.

## What happens overnight

The loop runs on an Ubuntu VM on the mini PC. It has an implementation pass that takes one eligible issue and a verification pass that runs the resulting app. Both use [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/): disposable microVMs with their own kernel, network stack and Docker daemon. The loop removes each sandbox when its pass ends, whether it succeeded or failed.

The implementation agent gets one issue, one fresh clone and one branch name. Selection and the checks that decide whether to publish live in the control plane, the ordinary code around the agent.

<pre class="mermaid">
flowchart TD
    ISS["GitHub: issues labelled\nready-for-agent"]

    subgraph BOX["the box in my study"]
        LOOP["control plane\nselect, clone, gates\nread-only GitHub token"]
        AGENT["implementation microVM\nClaude Code + fresh clone\nno GitHub access"]
        PUBLISH["publish microVM\npush + open draft PR"]
        VERIFY["verification microVM\nruns the app, records evidence"]
        PROXY["host-side proxy\ninjects write credential"]
    end

    PRS["GitHub: pull requests\ndraft, then ready for review"]
    HC["Healthchecks.io\ndead-man's switch"]

    ISS -->|"1. select"| LOOP
    LOOP <-->|"2. work, capture diff"| AGENT
    LOOP -->|"3. gates pass"| PUBLISH
    PUBLISH -->|"publish requests"| PROXY
    PROXY -->|"4. authenticated writes"| PRS
    PRS -->|"5. select oldest labelled PR"| LOOP
    LOOP -->|"dispatch verification"| VERIFY
    VERIFY -->|"evidence, verdict, any fixes"| PROXY
    LOOP -->|"pass outcome"| HC
</pre>

GitHub labels supply the queues. Locally, the loop keeps a list of issues parked after a gate failure, plus an audit bundle for each run: the prompt, transcript, diff and gate results. There's no database.

The two passes share a lock, so only one runs at a time. Each repo has hourly implementation slots between midnight and 7am on UK weekdays, with verification slots offset between them. Adding stride-gpt brought the schedule to four cron entries:

~~~cron
0  0-6 * * 1-5 run-loop.sh   mrwadams/attackgen
30 0-6 * * 1-5 run-verify.sh mrwadams/attackgen
15 0-6 * * 1-5 run-loop.sh   mrwadams/stride-gpt
45 0-6 * * 1-5 run-verify.sh mrwadams/stride-gpt
~~~

A slot can finish in seconds because there's nothing eligible or the PR limit has been reached. A real run can occupy several slots; later jobs see the lock and skip.

## Writing work the agent can use

Selection is a plain function over the open issues. It skips anything with an open agent PR, anything parked after a gate failure, and anything with an open dependency under a `## Blocked by` heading. It takes leaf issues before their parent epics, then the lowest issue number. Given the same queue, it makes the same choice. I can test that the loop chooses the right issue before leaving it to work unattended.

The issue has to do more work than a reminder I'd leave for myself. It needs to describe the current code, say what should change, identify what must stay the same, and give the verifier something observable to check.

For example, attackgen [#89](https://github.com/mrwadams/attackgen/issues/89) addressed an elapsed-time label that froze while a model was waiting to send its first token. A spinner kept moving, but the timer beside it stopped, making a working request look hung. The issue specified both generation phases, preserved incremental streaming and the order in which results appeared, and required errors to leave no spinner running. It also named the prerequisite issue.

This is a shortened version of its structure:

~~~markdown
## What to build
Keep the elapsed timer advancing while a streamed model call
waits for its first token, in both generation phases.
Change progress feedback only, preserving output and persistence.

## Acceptance criteria
- The elapsed time visibly advances during the wait.
- Output still streams incrementally when tokens arrive.
- Errors surface to the user and leave no spinner running.
- Tests use a controllable stream, without a real model call.

## Blocked by
- #48
~~~

Writing issues to that standard is now a substantial part of my input to the loop. A vague issue leaves the agent deciding what I meant and the verifier deciding what would count as success.

## Checking the code

When the implementation agent exits, three gates run in order, cheapest first. The first failure stops the run.

| Gate | What it checks |
| --- | --- |
| `files_non_empty` | A diff exists and at least one changed file has content. A clean exit isn't proof of work. |
| `containment` | The captured diff doesn't touch a denied path. |
| `verify` | The repository's unit suite executes and introduces no new observed failures compared with the clean base. Browser checks are excluded. |

The test gate runs the changed clone in a fresh container on the sandbox's Docker daemon, using dependencies prepared before the agent starts. The loop temporarily allows package-registry access to install those dependencies, then removes it so the agent can use the real suite offline.

A gate failure parks the issue for me to inspect. I chose not to feed it straight back into another implementation attempt: repeated attempts could optimise for getting past the gate, and I'd still need to work out whether the issue had been fixed. The bundle preserves what happened. The repository's normal CI also runs once a PR is opened.

## Giving the agent a browser

Initially, I hadn't set up browser verification for the agent. It could edit the source and run code-level checks, but it wasn't checking the app as a user would experience it.

Attackgen [#94](https://github.com/mrwadams/attackgen/pull/94) exposed that gap. It was an accessibility change that included a sidebar theme setting intended to remove browser console warnings. The code-level checks passed, but when I opened the app in Chromium, the warnings were still there. The PR also included browser tests, which hadn't been run; four of six failed when I ran them.

The agent needed a way to start the app, open it in a browser and check the result of its changes. I'd seen browser use in coding-agent products such as [Devin](https://docs.devin.ai/work-with-devin/devin-session-tools#browser-use-cases) and [Cursor](https://cursor.com/docs/agent/tools/browser). This issue showed why I needed it in my own setup: a source change and passing unit tests weren't enough to establish that the problem on screen had been fixed.

I added that capability in a separate verification pass. Its sandbox includes headless Chrome and Playwright, so the verifier can interact with the running app, inspect the page and check the browser console against the issue's acceptance criteria. The implementation pass still does the code-level work; the browser pass checks the resulting behaviour before I review the PR.

Every new PR gets an `awaiting-verification` label. The verifier takes the oldest labelled PR, one per run, records its observations in a GitHub comment and posts a verdict. A successful verdict takes the PR out of draft; an unsuccessful one leaves a comment explaining what couldn't be verified.

The verifier is allowed to fix problems it finds during these checks. It can change the code, push a new commit to the PR and check the updated version before posting its verdict. Any fixes it makes become part of the diff I review, just like the implementation agent's original changes.

Alongside the verdict, it leaves a record of what it observed for each acceptance criterion, including anything it couldn't check and why. For a UI change, that might be what happened after clicking a control or what appeared in the browser console. During review, I can inspect those observations and follow up on gaps. The verifier's assessment is an input to that review; I still decide whether to merge.

Once the agent finishes, the control plane fetches those records from GitHub and uses the posted verdict to determine the pass's exit code and whether the PR is taken out of draft. That ties the run's status to the result recorded on the PR, where I can review it. If the agent says "I posted the verdict" in its terminal summary but the comment is missing, the verification pass is incomplete.

The verdict also records the exact commit it checked, using its SHA, Git's identifier for that version of the code. If the verifier pushes a fix, its verdict needs to refer to the updated commit. If another change is pushed later, the old observations no longer establish what happens in the new version, so that version needs fresh verification. The GitHub label tells the scheduler which PR to work on; the commit identifier tells me which version the result applies to.

### Resuming verification after a timeout

Running the app and checking every acceptance criterion can take longer than the one-hour limit for a verification pass. To avoid starting over after a timeout, the verifier records its observations on the PR as it goes. A later run can reuse that evidence if the commit hasn't changed and continue with the remaining criteria.

That happened on 28 September with attackgen [#123](https://github.com/mrwadams/attackgen/pull/123). The first pass timed out after recording evidence for two of eight criteria. The next pass picked up that record and completed verification, as these log excerpts show:

~~~text
=== 2026-09-28T00:30:01+01:00 run-verify pass on mrwadams/attackgen ===
no verdict, but partial evidence survives: 2/8 criteria carry evidence at 44f6173 (in-progress)

=== 2026-09-28T02:30:01+01:00 run-verify pass on mrwadams/attackgen ===
verify: resuming — a previous pass left 2/8 criteria carry evidence at 44f6173 (in-progress)
marked #123 ready for review — its verdict carries evidence on every criterion
~~~

The evidence stays with the PR when the sandbox is deleted. I can see what an unfinished run checked, and the next run has a record to continue from.

## Limiting the agent's blast radius

Leaving an agent unattended means giving it somewhere to run arbitrary code while limiting what that code can affect. I want a bad run contained to its sandbox, the shared clone and the access needed for that phase. That requires looking at every connection back to the host and out to other systems.

In April I published an [insider threat model for AI agents](https://ai-insider-threat.matt-adams.co.uk/). I rated its highest autonomy level, execution on automated triggers with human review afterwards, critical. This loop operates at that level, so its isolation and access controls need to hold without me watching the agent.

The loop starts Claude Code with `--permission-mode bypassPermissions` so it can edit files and run commands overnight without waiting for me to approve each action. This skips the classifier-based permission review used in Claude Code's `auto` mode. My custom hook still runs, while the microVM and network policy limit what the agent can reach and the post-run gates check its changes. I review the resulting PR before anything merges.

### Closing a route from the sandbox to the host

The microVM isolates the agent's processes from the host. The working clone is deliberately shared, though, and I initially overlooked what could happen when host-side code used files the agent had changed.

After the agent finished, the control plane used to run git commands in that clone on the host. An agent could plant a git hook or set `core.fsmonitor` to a command of its choice. The next host-side git operation could then execute it as me, outside the microVM and with the host user's access. That gave sandbox-written files a route to host execution.

I tested this by planting harmless commands in the clone's git configuration and hooks, then running the same git operations the loop uses after an agent finishes. They ran on the host as my user. In an unattended run, that would give the agent my account's access to files and the network, including the ability to interfere with the control plane supervising it. Reviewing the staged diff wouldn't have warned me: it showed only the harmless decoy file, because the git metadata responsible for executing those commands never appears in the diff.

The existing containment checks couldn't close that route. My custom Claude Code `PreToolUse` hook checks the file-editing tools against a deny list covering `.env`, CI workflows and the loop's configuration, and denies the call if it can't evaluate its input. It isn't configured to inspect Bash commands. The later containment gate checks the captured diff, but git never lists `.git/` there. That leaves the host-side git operations needing their own protection against metadata the agent can change.

The fix copies `.git` to a separate, trusted location immediately after checkout, before the agent starts. Every later host-side git command uses that copy through an explicit `--git-dir`, with hooks and fsmonitor turned off. The implementation agent never gets that directory. Publishing uses the trusted copy too. If it's missing, the loop refuses to fall back to the agent's `.git`.

### Limiting network and GitHub access

Network access is another part of the blast radius. A process that can reach the rest of my homelab or make authenticated writes to GitHub has ways to cause damage even while it stays inside the VM. The sandbox's outbound policy is enforced by a proxy on the host, beyond the agent's control.

During implementation, the agent can reach the Claude service endpoints supplied by the sandbox template. GitHub, package registries and other destinations are blocked once dependency preparation finishes. A separate publish sandbox gets GitHub access after the gates pass. The verification sandbox has broader access to GitHub and package registries so it can prepare the app, post evidence and push fixes. Each phase gets the access needed for its job; the implementation agent can't directly publish its work.

Credential handling limits what a sandbox can take away with it. The control plane uses a read-only GitHub token. The write token is stored as an `sbx` secret and used by the [host-side credential proxy](https://docs.docker.com/ai/sandboxes/architecture/#networking), which substitutes the real credential into outbound requests after they leave the sandbox. The sandbox receives a placeholder, not the raw token. Where the host uses the read token to clone a private repository, it removes it from the remote URL before sharing the clone.

The publish and verification sandboxes can still make authenticated writes through the proxy. Keeping the raw token outside them prevents it being copied out and reused elsewhere; it doesn't remove the authority those requests carry. The verifier's broader access is part of the blast radius I have to account for, and the host remains trusted.

## Noticing when nothing happens

The failure I most wanted to catch was the loop quietly not running at all. There's no failed diff to review when a login expires or cron stops firing.

The wrappers report to [Healthchecks.io](https://healthchecks.io), a dead-man's switch that alerts when an expected heartbeat doesn't arrive. The implementation pass distinguishes these outcomes:

| Exit | Meaning | Heartbeat |
| --- | --- | --- |
| `0` | Worked an issue, gates passed | success |
| `1` | Gate failure, work parked | `/fail` |
| `2` | Idle, nothing eligible | success |
| `3` | Open-PR cap reached | success |
| `4` | Agent could not run | no completion ping |

Exit `4` leaves the issue eligible and sends no completion ping. Recording it as a gate failure would park work that hadn't had a real attempt. Reporting success would make a box with an expired login look healthy indefinitely. The missing heartbeat lets the alert fire.

Lock contention needs care too. A skip that sends nothing can make a healthy job look absent. A skip that always sends success can hide a hung run: every later job skips, every check stays green, and no work progresses. The wrapper reports a healthy skip while the lock has been held for less than three hours, then reports failure.

A verifier rejection is a healthy outcome for the verification service: it did its job and left something for me to review. An incomplete or missing verdict is a failure of the run. That distinction keeps alerts useful.

<details markdown="1">
<summary>Monitoring setup details</summary>

Each repo has separate implementation and verification checks. Sharing one check across repos lets a working repo hide another that has stopped. A shared fallback gives a newly added repo a heartbeat before its own checks are configured.

The wrapper captures the ping's HTTP status and logs anything other than `200`. Silencing errors with `curl -f ... || true` would hide a wrong check UUID as easily as a network failure.

It also sends a `start` ping before a run. Healthchecks.io then uses the grace period to limit the gap between start and completion, so that grace needs to exceed the longest normal run. These are weekday jobs, and the expected schedule needs to reflect their overnight window.

</details>

## What four weeks of logs show

The loop's pace was set by the work I'd prepared and the changes I could review. Between 2 and 28 September, it completed 20 successful implementation runs. Most of its slots ended without starting the agent: 157 of 189 implementation runs, or 83%, found no eligible issue or a full review queue.

![Horizontal bar chart of 189 implementation runs: 93 had no eligible issue, 64 hit the review limit, 20 worked an issue, 9 needed a human and 3 could not run.]({{ '/assets/images/agent-loop-implementation-outcomes-nano-banana.jpg' | relative_url }})

*Implementation runs, 2 to 28 September 2026. Lock skips are excluded.*

The three-PR cap kept the agent from getting further ahead than I could review. Increasing the schedule's frequency would fill that queue sooner, but the loop would still wait for me to make room.

Checking the app took more time than implementing the change. Among runs lasting more than five minutes, implementation had a median of 17 minutes and verification about 39. That makes verification capacity an important part of any plan to expand the loop.

<details markdown="1">
<summary>Full run breakdown and operational failures</summary>

The box completed 308 runs: 189 implementation passes and 119 verification passes. The reporting period includes the earlier daytime schedule and manually triggered runs, as well as overnight cron runs. The 23 lock skips are excluded from these counts.

The loop's own code isn't public, but both repositories it works on are. The examples in this post link to their issues and pull requests; the run counts and timings come from the logs.

| Outcome | Implementation runs | Verification runs |
| --- | --- | --- |
| Worked an issue / signed off | 20 | 10 |
| Needed a human | 9 | 6 |
| Idle, nothing eligible | 93 | 97 |
| Open-PR cap reached | 64 | n/a |
| Could not run at all | 3 | 6 |
| **Total** | **189** | **119** |

Only 24 implementation runs lasted more than five minutes. Their median duration was 17.1 minutes and their maximum 45.5. The 21 verification runs over five minutes had a median of about 39 minutes, and five reached the one-hour timeout.

Verification signed off 10 PRs during the reporting period, but it only started on 13 September. Its counts therefore cover a shorter operating period than the implementation counts.

On 14 September, the host's out-of-memory killer ended six verification runs in a row. They left the PR queued without a verdict. Matching the failures to dmesg timestamps led to a 3 GB sandbox cap, swap on the host and the overnight schedule.

On 21 September, stride-gpt's first overnight run exposed the assumption that every default branch was called `main`. Its branch is `master`, and most of the nine implementation failures came from that night. Three further attempts couldn't run because the agent's login had expired. The loop now discovers the remote's default branch and recognises the login failure messages those runs exposed.

</details>

### What it costs me

The agent runs use subscription authentication, so they consume plan usage limits rather than incurring a separate per-token API bill. The dollar figure in each run bundle is the agent's estimate of what the same work would have cost through the API.

Overnight scheduling uses a window when I'm not using the agent interactively. It also leaves the PRs ready for breakfast. My work is in writing the issues, reviewing the changes and investigating parked runs. The three-PR cap keeps that review queue small enough for me to deal with.

## What I'd build next

Adding a third repo would mean another two cron entries, another two health checks and another pair of offsets competing for the same lock. I've designed a replacement: one scheduler reads a list of repos, finds available work across them, chooses one piece and exits. Adding a repo becomes one line in a list.

The verifier also needs a reproducible environment. Its browser template is currently a saved snapshot; rebuilding it from a Dockerfile would make it easier to stand up on another box. The loop doesn't rebase yet, so PRs that wait for review can also drift behind their base branch.

Before increasing output, I'd keep the review limit and improve the issues feeding the queue. Selection can be deterministic, tests can run independently of the implementation agent's summary, and runtime checks can leave evidence for review.

If I were starting again, I'd make four decisions early: choose work in ordinary code, state exactly what each check proves, give each execution phase only the access it needs, and monitor the absence of a completed run. Much of that is familiar CI practice. An unattended coding agent makes the gaps more consequential because it can produce a plausible change while nobody is watching.

It's been running long enough now that I mostly forget it's there until a draft pull request turns up.
