# What is FastRouter & Why Use It?

### The Problem Without a Proxy

When you call an LLM directly, your code is tightly coupled to one provider:

```
Your App  →  OpenAI API
Your App  →  Anthropic API
Your App  →  Google Gemini API
```

Every provider has:
- A **different API format** (different endpoints, request/response shapes)
- A **different SDK** to install and manage
- A **different API key** to store and rotate
- **No fallback** — if a provider goes down, your app breaks
- **No visibility** — no unified logs, costs, or usage metrics across providers

---

### What FastRouter Does

FastRouter is an **LLM Gateway** (a smart proxy) that sits between your application and all the LLM providers:

```
                        ┌──────────────┐
                        │   OpenAI     │
                        ├──────────────┤
Your App  →  FastRouter │   Anthropic  │
  (one API,  (routes)   ├──────────────┤
  one key)              │   Google     │
                        ├──────────────┤
                        │   DeepSeek   │
                        └──────────────┘
```

**Your code never changes.** You always call the same `base_url` with the same OpenAI-compatible format. FastRouter handles everything behind the scenes.

---

### Key Features of FastRouter

| Feature | What it means for you |
|---|---|
| **Unified API** | One OpenAI-compatible endpoint for all providers — swap models by just changing the `model` string |
| **Intelligent Routing** | Auto-routes requests based on cost, speed, and quality |
| **Automatic Failover** | If one provider is down, FastRouter retries with another automatically |
| **Cost Tracking** | Unified dashboard showing spend across all providers |
| **Observability** | Centralized logs, latency metrics, and error tracking |
| **Access Control** | Manage API keys, set usage limits, assign team roles |
| **100+ Models** | Access text, image, video, embedding, and speech models from one place |

---

### Supported Providers & Example Models

| Provider | Example Models |
|---|---|
| **Anthropic** | `anthropic/claude-sonnet-4-20250514`, `anthropic/claude-opus-4.1` |
| **OpenAI** | `openai/gpt-4.1`, `openai/gpt-5`, `openai/gpt-5-mini` |
| **Google** | `google/gemini-2.5-pro`, `google/gemini-2.5-flash` |
| **xAI** | `xai/grok-4` |
| **DeepSeek** | `deepseek/deepseek-r1-distill-llama-70b` |
| **Meta** | `meta/llama-3.3-70b-instruct` |
| **Mistral AI** | `mistral/mistral-large-latest` |
| **Groq** | `groq/llama3-70b-8192` |
| **Qwen (Alibaba)** | `qwen/qwen3-coder` |
| **Moonshot AI** | `moonshot/kimi-k2` |

> Full model list: [https://fastrouter.ai/models](https://fastrouter.ai/models)

---

### How to Switch Providers in Code

Since FastRouter uses the OpenAI-compatible format, switching providers is as simple as **changing the `model` string** — nothing else changes:

```python
# Using Anthropic Claude (what we tested)
model="anthropic/claude-sonnet-4-20250514"

# Switch to OpenAI GPT-4.1
model="openai/gpt-4.1"

# Switch to Google Gemini
model="google/gemini-2.5-pro"

# Switch to DeepSeek
model="deepseek/deepseek-r1-distill-llama-70b"

# Switch to Groq (ultra-fast inference)
model="groq/llama3-70b-8192"
```

The `client`, `base_url`, `api_key`, and entire `messages` structure stays exactly the same.

---

### Without FastRouter vs. With FastRouter

**Without FastRouter** — if you want to use 3 providers:
```python
# Need 3 different SDKs, 3 API keys, 3 different code implementations
import openai        # pip install openai
import anthropic     # pip install anthropic
import google.generativeai  # pip install google-generativeai

openai.api_key = "sk-openai-..."
anthropic_client = anthropic.Anthropic(api_key="sk-ant-...")
genai.configure(api_key="AIza...")

# Each has completely different method calls and response formats
```

**With FastRouter** — same 3 providers, zero extra work:
```python
# One SDK, one API key, one consistent interface
from openai import OpenAI

client = OpenAI(
    base_url="https://go.fastrouter.ai/api/v1",
    api_key="sk-v1-..."  # just your FastRouter key
)

# Swap providers by only changing this one string:
model = "anthropic/claude-sonnet-4-20250514"  # or openai/gpt-4.1, or google/gemini-2.5-pro
```

---
