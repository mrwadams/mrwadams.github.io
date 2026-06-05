---
layout: post
title: "Zero Trust for AI Agents: A Cheat Sheet"
date: 2026-06-05
categories: [ai, security]
tags: [agents, ai, security, zero-trust, threat-modeling, reference]
---

## Introduction

Anthropic recently published [Zero Trust for AI Agents](https://claude.com/blog/zero-trust-for-ai-agents), a 36-page security framework for deploying autonomous AI agents in the enterprise. It's a good read, but 36 pages is a commitment. Here's the framework on one page, plus the bits I'd add on top.

## The whole framework on one page

![Zero Trust for AI Agents one-pager: principles, capability matrix across Foundation, Enterprise, and Advanced tiers, threat groups, and the 8-phase implementation workflow](/assets/images/zero-trust-ai-agents-one-pager.png)

Everything below is commentary.

## The design test: impossible, not tedious

The question sitting behind every tier recommendation in the eBook:

> Does this control make the attack impossible, or just tedious?

Friction controls work fine against humans who give up after a few hours. Rate limits, extra pivot hops, non-standard ports, SMS MFA all fall apart against attackers with unlimited patience and near-zero per-attempt cost, which is exactly the profile of an AI-accelerated adversary.

Prefer controls that remove a capability over controls that throttle it. Hardware-bound credentials. Expiring tokens. Cryptographic identity. Network paths that don't exist instead of paths that are merely inconvenient.

## Where Foundation sits now

Two agent-specific concepts justify raising the floor:

- **Blast radius**: the damage if an agent is compromised. A read-only DB agent has a small one; an agent with admin access to cloud infra has an enormous one. "Design for breach" means assuming every agent's blast radius will eventually get tested.
- **Least agency** (OWASP): extends least privilege from what an agent can access to what each tool can do, how often, and where. A DB tool gets read-only queries. An email summariser gets no send or delete rights.

Static API keys, friction-only mitigations, and "unique IDs" that are really just labels no longer count as Foundation. Those are known gaps.

## Numbers worth remembering

- 250 malicious documents can backdoor LLMs from 600M to 13B parameters, and the backdoors survive safety training (both SFT and RLHF).
- Around 100 malicious AI models have turned up on major hosting platforms, including some that open reverse shells on load.
- Microsoft's Spotlighting technique cut indirect prompt injection success from over 50% to under 2% by clearly delimiting untrusted content.
- Anthropic's constitutional classifiers blocked 95% of jailbreak attempts in testing without a meaningful jump in over-refusal.
- Algorithmic adversarial prompts can hit 100% attack success and transfer across model families.

## Where to draw the human/model line

Automate the bookkeeping around incidents, not the decisions.

Models take notes, gather artefacts, run parallel investigation tracks, draft the postmortem. Humans make the containment calls, the disclosure calls, the customer-comms calls. The same split scopes Agentic SOAR: use it for evidence collection, and keep the calls with humans.

## Defensive operations at machine speed

Securing the agents is half the job. The other half is running SecOps fast enough to keep up with attackers who are themselves AI-accelerated.

- Put a model at the front of your alert queue. Every alert gets an automated first-pass investigation before a human sees it. Start with one noisy rule, measure agreement against a human reviewer for two weeks, then expand. Don't try to automate the whole queue at once.
- Map detection coverage to MITRE ATT&CK. Prioritise lateral movement and credential access. [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) gives you a concrete coverage map in an afternoon.
- Tabletop for five simultaneous incidents, not one. Plan for an order-of-magnitude jump in finding volume.
- Pre-authorise your emergency change procedures. A two-week change-approval cycle is itself a security risk. Decide ahead of time who can take a service offline, rotate a credential, or block a network path, and rehearse the path.
- Apply Zero Trust to your defensive agents too. Agentic SOAR is itself a privileged agent: verified integrity, limited blast radius, clear escalation paths, full audit trail.

## Regulatory tailwind

Zero Trust is becoming mandatory in several jurisdictions, and agentic deployments inherit the same deadlines:

| Country | Reference |
| --- | --- |
| Australia | [homeaffairs.gov.au, Guiding principles of Zero Trust](https://www.homeaffairs.gov.au/) |
| UK | [NCSC.gov.uk, Introduction to Zero Trust](https://www.ncsc.gov.uk/) |
| US | CISA Zero Trust Maturity Model, NSA ZIGs, NIST SP 800-207. All federal agencies required to adopt by 2027. |

Sector-specific obligations to keep on the radar: HIPAA, FINRA, GDPR, FedRAMP, EU AI Act.

## Foundation floor checklist (2026)

These are entry requirements now, not aspirations:

- [ ] Short-lived tokens (not rotating API keys)
- [ ] Cryptographically rooted agent identity
- [ ] Identity-based isolation between agents
- [ ] Deny-by-default tool allow-lists
- [ ] Action logging with request-ID correlation across the chain
- [ ] Automated first-pass triage on every alert
- [ ] Documented rollback procedures, actually tested

Anything missing here is a known gap.

---

*For the full reasoning, the per-tier implementation tables, and Anthropic's notes on how Claude Code maps to each control, the [eBook](https://cdn.prod.website-files.com/6889473510b50328dbb70ae6/6a1611a04085d7cd3dadc924_Claude-eBook-Zero-Trust-for-AI-Agents-05182026.pdf) and [blog post](https://claude.com/blog/zero-trust-for-ai-agents) are worth the time. I've also added this to my [Awesome Agentic AI list]({% post_url 2025-06-28-awesome-agentic-ai %}).*
