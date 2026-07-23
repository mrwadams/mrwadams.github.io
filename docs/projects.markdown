---
layout: page
title: Projects
permalink: /projects/
---

I build open-source tools and research at the intersection of AI and security. Most of it started as a way to solve a problem I had at work: threat modelling that took too long, incident response exercises that went stale, and a growing pile of AI security frameworks that nobody had lined up side by side.

Together these tools have 2,200+ GitHub stars. Here's what I'm working on.

<br>

-----

## Open-source tools

### STRIDE-GPT

An AI threat modelling tool that uses large language models to generate threat models, attack trees, and DREAD risk scores from the STRIDE methodology. It runs as a Streamlit web app and, since v0.16, as an agentic CLI (`pip install stride-gpt`) that can analyse a whole codebase on its own. Threats are tagged with MITRE ATT&CK and ATLAS technique IDs, and it works with OpenAI, Anthropic, Google, Mistral, Groq, and local models via Ollama or LM Studio. Around 1,080 GitHub stars.

[GitHub](https://github.com/mrwadams/stride-gpt) &bull; [Live demo](https://stridegpt.streamlit.app/)

<br>

### AttackGen

Generates incident response exercises from the MITRE ATT&CK and ATLAS frameworks. Pick a threat actor group or a set of techniques and it writes a realistic scenario for a tabletop. Purple-team mode pairs each offensive scenario with a detection and response narrative, and there's an MCP server plus a tabletop skill so you can build and refine exercises in conversation with an AI assistant. Around 1,220 GitHub stars.

[GitHub](https://github.com/mrwadams/attackgen) &bull; [Live demo](https://attackgen.streamlit.app/)

<br>

### STRIDE-GPT MCP Server

A Model Context Protocol server that gives AI assistants the STRIDE-GPT toolset directly: threat modelling, DREAD risk scoring, attack tree generation, security test cases, and reporting. Add it to Claude Desktop or any MCP-compatible client and ask for a threat model in plain language. Currently in beta.

[GitHub](https://github.com/mrwadams/mcp-stride-gpt) &bull; [Hosted server](https://mcp.stridegpt.ai/)

<br>

-----

## Research and frameworks

### TM-Bench

A benchmark for how well LLMs actually do threat modelling. It scores models across four dimensions against a private test set, with a focus on models you can run on consumer hardware like a single RTX 4090. The point is to help teams choose a model for local security work with data rather than vibes.

[Website](https://www.tmbench.com) &bull; [Methodology](https://www.tmbench.com/methodology)

<br>

### AI Insider Threat Model

A framework that treats frontier AI agents as potential insider threats rather than just targets or tools. It adapts CERT's insider threat dimensions (motivation, opportunity, capability) to autonomous agents, maps 23 STRIDE threats across six categories, profiles risk across four levels of autonomy, and offers a NIST-aligned set of controls that slots into insider threat programmes teams already run.

[Read the framework](https://ai-insider-threat.matt-adams.co.uk/)

<br>

### OWASP AI Top 10 Comparator

There are now five OWASP Top 10 lists for AI security (LLM Applications, Agentic Applications, MCP, Agentic Skills, and ML Security), covering 50 distinct risks at very different levels of maturity. This tool puts all five side by side: where they overlap, where each one is silent, how each risk maps to Simon Willison's Lethal Trifecta, and which lists actually apply to your stack.

[Try the comparator](https://owaspai.matt-adams.co.uk/)

<br>

-----

There's more on my [GitHub](https://github.com/mrwadams), including [HoneyAgents](https://github.com/mrwadams/honeyagents) and an [OTX MCP server](https://github.com/mrwadams/otx-mcp). You can also [ask an AI about my work](/cv/) through the MCP server I've published.
