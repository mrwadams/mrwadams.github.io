---
layout: post
title: "Awesome Agentic AI: A Curated List of Resources"
date: 2025-06-28 00:00:00 -0000
last_modified_at: 2025-10-19 00:00:00 -0000
categories:
tags: [genai,agents,ai,resources,learning]
---

## Introduction

This is a curated list of high-quality resources for building and understanding agentic AI systems. Inspired by GitHub's "awesome" lists, this collection includes tutorials, guides, and practical frameworks that I've found particularly valuable for developing AI agents.

## Video Tutorials

### Building Effective Agents with LangGraph
**Link**: [https://youtu.be/aHCDrAbH_go?si=bhfrdA1zS0Q2DqgL](https://youtu.be/aHCDrAbH_go?si=bhfrdA1zS0Q2DqgL)

A comprehensive video tutorial on building agents using LangGraph, covering practical implementation details and best practices.

### Ask David: Multi-Agent AI for Investment Research - JP Morgan Chase
**Link**: [https://www.youtube.com/watch?v=yMalr0jiOAc](https://www.youtube.com/watch?v=yMalr0jiOAc)

From LangChain's Interrupt conference, David Odomirok and Zheng Xue from JP Morgan Chase Private Bank demonstrate "Ask David" - a sophisticated multi-agent AI system that automates investment research for thousands of financial products. This enterprise-grade system showcases how to build AI agents with human oversight for high-stakes financial decisions involving billions in assets.

## Practical Guides & Best Practices

### Building Effective Agents - Anthropic
**Link**: [https://www.anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)

Anthropic's engineering guide on building effective AI agents. This resource provides insights from the creators of Claude on agent design patterns, common pitfalls, and architectural considerations.

### Claude Code Best Practices - Anthropic
**Link**: [https://www.anthropic.com/engineering/claude-code-best-practices](https://www.anthropic.com/engineering/claude-code-best-practices)

Anthropic's comprehensive guide to best practices for using Claude Code effectively. This resource covers optimal workflows, command patterns, and strategies for maximizing productivity when building software with Claude's AI assistance.

### A Practical Guide to Building Agents - OpenAI
**Link**: [https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)

OpenAI's practical guide offering a business-oriented perspective on building AI agents. This PDF covers use cases, implementation strategies, and considerations for deploying agents in production environments.

## Security & Threat Modeling Frameworks

### An Introduction to Google's Approach for Secure AI Agents
**Link**: [https://research.google/pubs/an-introduction-to-googles-approach-for-secure-ai-agents/](https://research.google/pubs/an-introduction-to-googles-approach-for-secure-ai-agents/)

Google Research presents an aspirational framework for building secure AI agents based on three core principles: well-defined human controllers, carefully limited powers, and observable actions. The paper advocates for a defense-in-depth strategy combining traditional deterministic security controls with dynamic reasoning-based defenses, aiming to develop AI agents that are powerful, useful, and secure by default.

### MAESTRO: Agentic AI Threat Modeling Framework - Cloud Security Alliance
**Link**: [https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro)

MAESTRO (Multi-Agent Environment, Security, Threat, Risk, and Outcome) is a comprehensive threat modeling framework specifically designed for Agentic AI systems. Built on a seven-layer reference architecture, it helps security engineers and AI developers proactively identify and mitigate risks unique to autonomous AI agents, including goal misalignment, adversarial attacks, and agent-to-agent interaction vulnerabilities.

### Agentic AI Threats and Mitigations - OWASP
**Link**: [https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/](https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/)

OWASP's comprehensive resource exploring key threats and mitigation strategies for agentic AI. This guide focuses on security measures to address vulnerabilities in AI applications and their potential risks, providing practical guidance for developers and security professionals working with autonomous AI systems.

### The Lethal Trifecta for AI Agents - Simon Willison
**Link**: [https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)

Simon Willison identifies a critical security vulnerability pattern in AI agents: the dangerous combination of private data access, exposure to untrusted content, and external communication capabilities. This article explains how combining these three elements creates opportunities for exploitation through prompt injection attacks, where attackers can trick AI systems into leaking sensitive data. Willison, who coined the term "prompt injection," draws parallels to SQL injection and emphasizes the importance of avoiding this lethal combination when designing AI agent systems.

### Design Patterns for Securing LLM Agents against Prompt Injections
**Link**: [https://arxiv.org/abs/2506.08837](https://arxiv.org/abs/2506.08837)

A comprehensive research paper by Beurer-Kellner et al. that proposes principled design patterns for building AI agents with provable resistance to prompt injection attacks. The paper systematically analyzes security patterns including the Action-Selector Pattern (which prevents feedback from actions to the agent) and the Plan-Then-Execute Pattern (which allows tool output feedback while preventing influence on action choices). Through detailed case studies, the authors examine trade-offs between security guarantees and agent utility, providing practical guidance for developing secure LLM-based agents that handle sensitive information and tool access.

### Defeating Prompt Injections by Design
**Link**: [https://arxiv.org/abs/2503.18813](https://arxiv.org/abs/2503.18813)

Research from Google, Google DeepMind, and ETH Zurich that introduces CaMeL (Capability-based Memory and Logic), a robust defense system against prompt injection attacks in LLM agents. CaMeL creates a protective system layer around the LLM that explicitly extracts control and data flows from trusted queries, preventing untrusted data from affecting program flow. Using a capability-based approach to prevent data exfiltration, CaMeL achieves 77% task success with provable security guarantees compared to 84% with undefended systems in AgentDojo, demonstrating that strong security can be achieved with minimal utility trade-offs.

## Tools & Applications

### LangGraph Flow Designer
**GitHub**: [https://github.com/mrwadams/langgraph-flow-designer](https://github.com/mrwadams/langgraph-flow-designer)  
**Live App**: [https://langgraph-flow-designer.vercel.app/](https://langgraph-flow-designer.vercel.app/)

A visual flow designer I created for building and managing LangGraph workflows. This interactive tool provides a drag-and-drop interface for designing complex graph-based AI workflows, featuring grid snap alignment, multiple node types (regular, START, END, subgraph), conditional edge connections, and JSON export/import capabilities. Built with React and Vite, it simplifies the process of designing sophisticated AI agent workflows by providing an intuitive visual development environment.

---

*This list will be updated as I discover new valuable resources. Feel free to suggest additions via [GitHub issues](https://github.com/mrwadams/mrwadams.github.io/issues)*