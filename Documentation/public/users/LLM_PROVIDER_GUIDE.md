# LLM Provider Guide

How TReS Cloud integrates with large language models, how to choose
between providers, and what your data does (and doesn't) leave the
system.

## What AI features use the LLM

AI surfaces in TReS Cloud are limited and clearly
labeled — when one is invoked, the LLM provider receives a prompt
containing the data needed to answer it. AI surfaces include:

- **AI Analysis** on the Database tab (graph summarization and
  ontology quality assessment). **(Planned — not yet on Preview.
  The only shipped AI surface that calls the LLM is the LLM Testbed
  chat below; the ontology-quality readiness diagnostic is
  rule-based and does not call the LLM.)**
- **LLM Testbed chat** (uses the LLM to answer natural-language
  questions by grounding on your graph — it resolves terms and
  retrieves facts via grounding tools on QROS).
- Future surfaces will be introduced behind feature flags.

Non-AI surfaces — Vocabulary tab, Query / Browse, Dashboard,
Find Path, Load Data, Database administration — **do not call
the LLM at any time.**

TReS grounds data; it does not train models. No provider's
fine-tuning or model-creation endpoint is ever called, and nothing
your team enters is applied to a model automatically — see
*Grounding, not training* in `LLM_EVALUATION_TRAINING_GUIDE.md`.

## Provider choice

TReS supports four configurations: disabled, Claude, Ollama, and
OpenAI. The three named providers are selectable both in tfvars
and from the app (see "Connecting your own provider" below);
OpenAI is configured the same way as Claude, with
`llm_provider = "openai"` and that provider's model names.

### Disabled (default)

No LLM is configured. AI surfaces in the SPA show "AI features
disabled" and return HTTP 503 from the underlying endpoints. The
rest of the cloud serves normally. This is the default for every
deployment; opt in explicitly when you have a key or local model
ready.

### Claude (Anthropic, cloud-hosted)

The recommended option for production deployments that want
high-quality output. Requests are sent to Anthropic's Messages
API over HTTPS; your prompt + any graph data referenced in the
prompt are processed by Anthropic.

Configure by adding to your tfvars:

```hcl
llm_provider           = "claude"
llm_model              = "claude-sonnet-4-6"
llm_api_key_secret_arn = "arn:aws:secretsmanager:us-east-1:000000000000:secret:tres/<your-instance>/llm-anthropic-xxxxx"
```

The Secrets Manager entry holds the raw Anthropic API key.
`tres-api` reads it at task start via the ECS task definition's
`secrets` array; the key never lives in plaintext on disk and is
not logged.

### Ollama (local)

Useful for on-prem cloud deployments where data must not leave
the customer's network (updated 2026-08-28). Ollama runs on
the same machine; the model name is pulled in advance via
`ollama pull <model>`.

For an on-prem cloud deployment, add to your tfvars (updated
2026-08-28):

```hcl
llm_provider = "ollama"
llm_model    = "llama3.1:8b"
llm_endpoint = "http://your-ollama-host:11434"
```

Note: tool calling (used by the LLM Testbed chat) requires Llama
3.1+ or another tool-call-capable model. Earlier Ollama-supported
models accept the request but ignore the tools list.

## Connecting your own provider from the app

On TReS Cloud an instance administrator can attach their own
provider without an operator, a tfvars edit, or a deployment:
**Admin → AI provider**. Choose a provider (Ollama, Claude, or
OpenAI), enter the model, optionally an endpoint, and paste your
key. Saving applies to the next question asked — there is no
restart.

**Your key is write-only.** Once saved it is never shown again,
to anyone. It is stored in a credential store belonging to your
instance alone, and there is no screen, endpoint, or support
procedure that returns it — not for you, and not for us. Changing
a key means entering a new one, never editing the old one. The
page shows only whether a key is stored and when it was last set.

An administrator with server-settings permission can also remove
the configuration, which clears the stored key and returns the
instance to its default provider.

Two behaviors worth knowing:

- **A configuration that cannot load is refused, not saved.** The
  provider is built before anything is written, so a wrong model
  name leaves your working setup running rather than replacing it
  with a broken one.
- **A saved configuration that later fails does not fall back.**
  AI features switch off and say so. Falling back silently would
  bill your misconfiguration to somebody else and hide it from
  you.

If your deployment has no credential store configured, the key
field is disabled and the page says so — a provider that needs no
key, such as self-hosted Ollama, still works.

## Switching providers later

From the app, use **Admin → AI provider** as above; it takes
effect immediately and needs no apply.

Operator-side, edit the tfvars and `tofu apply`. The new task
definition picks up the new provider config on rollout; existing
tasks drain. No data migration is needed — AI is stateless. A
provider saved in the app takes precedence over the tfvars
setting.

To disable AI entirely, set `llm_provider = ""` (the default) and
apply. The next task rollout has no LLM env vars set; AI
surfaces return 503.

## What leaves your system

Plain "disabled": **nothing.** No LLM is called.

"Claude": each AI invocation sends a prompt to Anthropic over
HTTPS. The prompt typically contains:

- The current question or instruction from the user.
- A compressed summary of relevant ontology / vocabulary
  information from your graph (for grounding).
- For surfaces that load graph file content (AI Analysis — planned,
  not yet on Preview), the actual graph content being analyzed.

Anthropic's data policies apply to anything sent. Per their
default API policy at time of writing, prompts and completions are
NOT used to train Anthropic's models when sent through the
Messages API. Verify the current policy on Anthropic's site
before configuring sensitive data flows.

"Ollama" (local): all traffic stays on the configured Ollama
endpoint. Nothing leaves the host network. The model runs
entirely on-premises.

## Cost notes

Claude is billed per token by Anthropic. TReS logs the input and
output token counts for every AI invocation in the structured
JSON log line at log target `tres-core::llm`. Sample line:

```json
{
  "event": "llm.chat",
  "provider": "claude",
  "model": "claude-sonnet-4-6",
  "latency_ms": 1234,
  "usage": {"input_tokens": 500, "output_tokens": 50},
  "tool_calls_emitted": 0,
  "user_id": "user-42",
  "stop_reason": "stop"
}
```

Operators can aggregate these by parsing the cloudwatch log
stream. A cost-reporting UI is planned for a later release; today
the JSON log is the source of truth.

Ollama has no per-call cost. The model runs locally on whatever
hardware you've allocated.

## Disabling per-user

There is no per-user AI toggle today. AI surfaces are gated by
tenant-level provider config only. Per-user opt-out is a planned
consideration tied to the role-based access work.

## Troubleshooting

**"AI features disabled" surface in the SPA.** Check the
operator's `llm_provider` config in tfvars. If unset or empty,
that's expected. If set, check the tres-api task logs for the
"LLM provider config error" line at startup — usually a missing
`TRES_API_LLM_MODEL` env var or a Secrets Manager ARN that
resolved to an empty string.

**"AI is unavailable, try again."** The provider returned an
error. Common causes:

- Claude: 401 means the API key is invalid or revoked. 429 means
  Anthropic's rate limit was hit; retry in a few seconds.
- Ollama: "model not loaded" — pull the model first with
  `ollama pull <model>` on the Ollama host.

The task logs contain the underlying error at WARN level under
the `tres-core::llm` target.

**Slow AI responses.** Latency is logged per-call in the JSON
event under `latency_ms`. Claude responses typically complete in
2–8 seconds; Ollama responses depend on the model and host
hardware. Consistent latency > 30s usually indicates a context-
window blowup; check the size of the prompt content (e.g. TTL
content sent to AI Analysis).
