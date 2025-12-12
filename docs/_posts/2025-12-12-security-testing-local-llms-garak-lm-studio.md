---
layout: post
title: "Security testing local LLMs with garak and LM Studio"
date: 2025-12-12
categories: [security, ai, llm]
---

The purpose of this post is to document how to use [garak](https://github.com/NVIDIA/garak), NVIDIA's LLM vulnerability scanner, to security test models running locally in [LM Studio](https://lmstudio.ai/). While garak supports many cloud-based LLM providers out of the box, connecting it to a local model requires some configuration that isn't well documented.

## What is garak?

Garak is an open-source security testing tool that probes LLMs for vulnerabilities including:

- Prompt injection
- Jailbreaks (e.g., DAN attacks)
- Data leakage
- Toxicity generation
- Hallucination
- Encoding-based attacks

Think of it as Nmap or Metasploit, but for language models.

## Prerequisites

- Python 3.10-3.12
- LM Studio installed with a model loaded
- LM Studio's local server running (default: `http://localhost:1234`)

## Installation

Install garak in a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate
pip install garak
```

## The Configuration Challenge

Garak's `OpenAICompatible` generator is designed for OpenAI-compatible APIs like the one LM Studio provides. However, passing the custom URI via command-line options doesn't work as you might expect. The `--generator_options` flag is ignored, and garak defaults to `http://localhost:8000/v1/`.

The solution is to use a YAML configuration file with the correct nested structure. I've [raised an issue](https://github.com/NVIDIA/garak/issues/1527) about this behaviour.

## Working Configuration

Create a file called `lmstudio_config.yaml`:

```yaml
plugins:
  generators:
    openai:
      OpenAICompatible:
        uri: "http://localhost:1234/v1/"
```

The key insight is the nesting: `plugins` > `generators` > `openai` > `OpenAICompatible` > `uri`. Getting this structure wrong results in garak silently ignoring your configuration.

## Running a Scan

1. Start LM Studio and load your model
2. Enable the local server in LM Studio (Server tab)
3. Set the API key environment variable (garak requires this even though LM Studio doesn't need authentication):

```bash
export OPENAICOMPATIBLE_API_KEY="lm-studio"
```

4. Run garak with your config file:

```bash
python -m garak \
  --config lmstudio_config.yaml \
  --target_type openai.OpenAICompatible \
  --target_name "my-local-model" \
  --probes dan.Dan_11_0 \
  -g 1
```

The `-g 1` flag sets generations to 1, which speeds up testing. The `--target_name` can be anything—LM Studio will use whichever model you have loaded.

## Example Output

Running a DAN (Do Anything Now) jailbreak probe against Mistral 3B:

```
garak LLM vulnerability scanner v0.13.2
📜 logging to /Users/matt/.local/share/garak/garak.log
🦜 loading generator: OpenAICompatible: lmstudio
🕵️  queue of probes: dan.Dan_11_0
probes.dan.Dan_11_0: 100%|██████████| 1/1 [00:07<00:00,  7.13s/it]
dan.Dan_11_0  dan.DAN: FAIL  ok on 0/1 (attack success rate: 100.00%)
dan.Dan_11_0  mitigation.MitigationBypass: FAIL  ok on 0/1 (attack success rate: 100.00%)
✔️  garak run complete in 8.00s
```

The results show this model is susceptible to the DAN jailbreak attack—not surprising for a small, unaligned model.

## Useful Probes

List all available probes:

```bash
python -m garak --list_probes
```

Some interesting ones to try:

| Probe | Description |
|-------|-------------|
| `dan.Dan_11_0` | DAN jailbreak attempt |
| `encoding` | Encoding-based attacks (Base64, ROT13, etc.) |
| `promptinject` | Prompt injection tests |
| `lmrc.Profanity` | Profanity generation |
| `knownbadsignatures` | Known malicious prompt patterns |

Run multiple probes:

```bash
python -m garak \
  --config lmstudio_config.yaml \
  --target_type openai.OpenAICompatible \
  --target_name "my-model" \
  --probes dan,encoding,promptinject
```

## Reports

Garak generates detailed reports in `~/.local/share/garak/garak_runs/`:

- `.report.jsonl` - Machine-readable results
- `.report.html` - Human-readable summary

## Why Test Local Models?

If you're building applications with local LLMs, security testing helps you:

- Understand your model's vulnerabilities before deployment
- Compare safety characteristics across different models
- Validate that fine-tuning hasn't introduced new weaknesses
- Document security posture for compliance purposes

## References

- [garak GitHub Repository](https://github.com/NVIDIA/garak)
- [garak Documentation](https://docs.garak.ai/garak)
- [LM Studio](https://lmstudio.ai/)
- [GitHub Issue #1008](https://github.com/NVIDIA/garak/issues/1008) - OpenAI compatible endpoints support
- [GitHub Issue #1527](https://github.com/NVIDIA/garak/issues/1527) - generator_options flag behaviour
