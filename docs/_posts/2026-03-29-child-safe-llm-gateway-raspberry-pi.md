---
layout: post
title: "Building a child-safe LLM gateway on a Raspberry Pi 5"
date: 2026-03-29
categories: [ai, raspberrypi]
tags: [llm, litellm, open-webui, raspberry-pi, parental-controls, minimax]
---

My children have been asking to use ChatGPT. They see me using LLMs for work and want to try them out, mostly for homework and writing stories. Fair enough. But handing a 9 and 11 year old unrestricted access to a commercial LLM isn't something I'm comfortable with.

So I built a self-hosted LLM gateway on a Raspberry Pi 5. It gives them a model behind content filtering, usage limits, and full chat visibility. Everything runs on the home network only. No cloud dashboard, no third-party accounts for the kids, no data leaving the house except the API calls themselves.

## The architecture

<pre class="mermaid">
graph TD
    A["Children's devices (LAN)"] --> B
    B["Caddy (:443)\nHTTPS reverse proxy, Let's Encrypt cert"] --> C
    C["Open WebUI (:8080)\nChat interface, user accounts, model personas"] --> D
    D["LiteLLM Proxy (:4000)\nAPI gateway, content filtering, rate limits"] --> E
    E["Minimax API\nMiniMax-M2.7 model"]

    style A fill:#2d333b,stroke:#768390,color:#adbac7
    style B fill:#2d333b,stroke:#768390,color:#adbac7
    style C fill:#2d333b,stroke:#768390,color:#adbac7
    style D fill:#2d333b,stroke:#768390,color:#adbac7
    style E fill:#2d333b,stroke:#768390,color:#adbac7
</pre>

Four Docker containers on the Pi: [Caddy](https://caddyserver.com/) as a reverse proxy with automatic HTTPS, PostgreSQL for budget and key tracking, [LiteLLM](https://docs.litellm.ai/) as the API proxy, and [Open WebUI](https://docs.openwebui.com/) as the chat frontend. The children access it at `https://chat.mattadams.tech` from any device on the home network, so they get a proper domain with a trusted TLS certificate rather than a raw IP address and port number.

### Why MiniMax M2.7?

I went with [MiniMax's M2.7 model](https://www.minimax.io/models/text/m27) because it's capable and cheap. That's basically it. It's good enough for homework and creative writing, and for a home setup where the kids might send hundreds of messages a week, I didn't want to worry about the bill. I'd also seen favourable reports about its quality relative to cost.

## Setting up the Pi

I started with a fresh Raspberry Pi 5 running Raspberry Pi OS (64-bit). Docker and Docker Compose went on first:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

The project directory looks like this:

```
~/llm-gateway/
├── .env                    # API keys and passwords
├── docker-compose.yml      # Container orchestration
├── litellm_config.yaml     # Model routing and rate limits
├── custom_guardrail.py     # Content filtering
├── Caddyfile               # Reverse proxy and TLS config
└── caddy/
    └── Dockerfile          # Custom Caddy build with Cloudflare DNS plugin
```

### Docker Compose

One gotcha worth mentioning: the LiteLLM Docker Hub image doesn't have ARM64 support. You need the GitHub Container Registry image instead.

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    restart: unless-stopped
    ports:
      - "4000:4000"
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
      - ./custom_guardrail.py:/app/custom_guardrail.py
    environment:
      MINIMAX_API_KEY: ${MINIMAX_API_KEY}
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
      DATABASE_URL: ${DATABASE_URL}
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    depends_on:
      postgres:
        condition: service_healthy

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    expose:
      - "8080"
    volumes:
      - open_webui_data:/app/backend/data
    environment:
      OPENAI_API_BASE_URL: ${OPENAI_API_BASE_URL}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      DEFAULT_USER_ROLE: ${DEFAULT_USER_ROLE}
      ENABLE_SIGNUP: ${ENABLE_SIGNUP}
      ENABLE_OLLAMA_API: ${ENABLE_OLLAMA_API}
      ENABLE_ADMIN_CHAT_ACCESS: ${ENABLE_ADMIN_CHAT_ACCESS}
      ENABLE_ADMIN_EXPORT: ${ENABLE_ADMIN_EXPORT}
      USER_PERMISSIONS_CHAT_DELETE: ${USER_PERMISSIONS_CHAT_DELETE}
      USER_PERMISSIONS_CHAT_EDIT: ${USER_PERMISSIONS_CHAT_EDIT}
      WEBUI_SECRET_KEY: ${WEBUI_SECRET_KEY}
    depends_on:
      - litellm

  caddy:
    build: ./caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    environment:
      CLOUDFLARE_API_TOKEN: ${CLOUDFLARE_API_TOKEN}
    depends_on:
      - open-webui

volumes:
  postgres_data:
  open_webui_data:
  caddy_data:
  caddy_config:
```

Note that Open WebUI uses `expose` rather than `ports`, so it's only accessible through Caddy, not directly from the network. The only ports open to the LAN are 80 (HTTP redirect) and 443 (HTTPS).

### LiteLLM config

The LiteLLM config handles model routing and rate limits. I set 10 requests per minute and 50,000 tokens per minute. Plenty for normal use, but tight enough to prevent runaway costs if something goes wrong.

```yaml
model_list:
  - model_name: MiniMax-M2.7
    litellm_params:
      model: minimax/MiniMax-M2.7
      api_key: os.environ/MINIMAX_API_KEY
      api_base: https://api.minimax.io/v1
      rpm: 10
      tpm: 50000

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL

litellm_settings:
  drop_params: true
  callbacks: ["custom_guardrail.ChildSafetyGuardrail"]
```

### Content filtering guardrail

LiteLLM supports custom guardrails, which are Python classes that intercept requests before they hit the model and responses before they reach the user. I wrote a regex-based filter that blocks queries about violence, weapons, drugs, adult content, and common jailbreak patterns:

```python
from litellm.integrations.custom_guardrail import CustomGuardrail
import re

BLOCKED_PATTERNS = [
    r"\b(how to (make|build|create) (a )?(bomb|weapon|explosive|gun|knife))\b",
    r"\b(how to (kill|hurt|harm|injure|murder))\b",
    r"\b(suicide|self[- ]harm)\b",
    r"\b(how to (make|cook|synthesize|grow) (meth|cocaine|heroin|drugs|lsd))\b",
    r"\b(porn|pornograph|nsfw|xxx|hentai)\b",
    r"\b(sexting)\b",
    r"\b(how to hack|how to steal|how to break into)\b",
    r"\b(ignore (previous |your |all )?instructions)\b",
    r"\b(you are now|pretend (to be|you are)|jailbreak|DAN mode)\b",
    r"\b(ignore (the |your )?system prompt)\b",
]

class ChildSafetyGuardrail(CustomGuardrail):
    async def async_pre_call_hook(self, user_api_key_dict, cache, data, call_type):
        messages = data.get("messages", [])
        for message in messages:
            content = message.get("content", "")
            if isinstance(content, str):
                self._check_content(content)

    async def async_post_call_success_hook(self, data, user_api_key_dict, response):
        if hasattr(response, "choices"):
            for choice in response.choices:
                if hasattr(choice, "message") and hasattr(choice.message, "content"):
                    content = choice.message.content
                    if isinstance(content, str):
                        self._check_content(content)

    def _check_content(self, text: str):
        text_lower = text.lower()
        for pattern in BLOCKED_PATTERNS:
            if re.search(pattern, text_lower):
                raise Exception(
                    "This request was blocked by the content filter. "
                    "If you think this is a mistake, ask a parent for help."
                )
```

This is a blunt instrument. Regex filtering will always have false positives and false negatives. But it catches the obvious stuff, and it's a first line of defence rather than the only one.

## Model personas

Rather than giving the kids a single generic assistant, I created five personas in Open WebUI. They all use MiniMax-M2.7 under the hood but each has its own system prompt and a friendly name.

**Homework Helper** is the one I spent the most time on. The most important rule: never give direct answers. The system prompt enforces the Socratic method, asking guiding questions and giving hints instead. I didn't want an AI that does their homework for them.

**Code Buddy** defaults to Python and Scratch, keeps examples short, and focuses on debugging skills ("What did you expect to happen? What actually happened?"). It suggests fun projects like simple games and drawing programs rather than abstract exercises.

**Story Spark** builds on the child's ideas rather than replacing them. It asks "What happens next?" and "Tell me more about this character" instead of generating entire stories. Content boundaries keep things age-appropriate. Adventure, mystery, and comedy are fair game; horror and anything beyond friendship-level relationships are not.

**Science Explorer** explains concepts with real-world analogies and suggests safe home experiments (with "ask a parent first" where appropriate).

**Language Buddy** asks which language and what level, then mixes the target language with English. My daughter has been using it to practise Spanish, mostly through small quizes and games the LLM thinks up.

In Open WebUI, these are created as custom models under Workspace > Models. Each points to MiniMax-M2.7 as the base model. The raw model is hidden from non-admin users, so the children only see the five options.

## How the controls stack up

No single control is foolproof, so I stacked them:

| Layer | What it does |
|-------|-------------|
| LiteLLM content filter | Blocks dangerous queries via regex before they reach the API |
| LiteLLM rate limits | 10 req/min, 50k tokens/min — prevents runaway costs |
| LiteLLM model restriction | Only MiniMax-M2.7 is available |
| System prompts | Each persona has rules about age-appropriate content |
| Open WebUI roles | Children are "user" role with no admin access |
| Chat deletion disabled | Children can't delete their conversation history |
| Admin chat access | I can review all conversations |
| Tools/functions disabled | Prevents code execution via Open WebUI plugins |
| Pending signup approval | New accounts require my approval |
| LAN-only access | No reverse proxy, no internet exposure |

In practice, the system prompts do the heavy lifting for nuanced cases, and the content filter catches the obvious ones. But honestly, the biggest factor is probably just chat visibility. The children know their conversations aren't private, and that alone shapes behaviour more than any regex.

## Cost

MiniMax M2.7 is cheap enough that I'm not worried about the children running up a bill. With the rate limits in place, heavy daily use stays under a few dollars a month. LiteLLM's admin UI at `:4000/ui` has spend tracking if I want to check.

The Pi draws about 5W at idle with the containers running, so it just stays on.

## What I'd change next time

The content filter needs ongoing tuning. Children are creative and they may find phrasings that bypass regex patterns. I plan to review the blocked list periodically based on what I see in chat history.

I'd also like per-child usage tracking. Right now Open WebUI connects to LiteLLM with a single API key, so spend is tracked in aggregate. Running separate Open WebUI instances per child with individual virtual keys would solve this, but it's overkill for two kids.

Eventually I'll probably replace the regex filter with something model-based. LiteLLM supports integrations with Azure Content Safety and other moderation APIs. For a home setup the regex approach works, but it's the obvious upgrade path when I get tired of maintaining pattern lists.

## How it's going

The whole setup took an evening to get running. It's only been a couple of days so the children have mainly been experimenting rather than using it for homework. My son is working on a vehicle battle game in Scratch with Code Buddy, and my daughter has been practising her languages and asking Science Explorer about everything from volcanoes to why the sky is blue.

If you want to do something similar, the source files are [available on GitHub](https://github.com/mrwadams/llm-gateway).
