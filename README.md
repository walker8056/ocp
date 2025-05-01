# OCP — Open Claude Proxy

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) [![GitHub release](https://img.shields.io/github/v/release/dtzp555-max/ocp)](https://github.com/dtzp555-max/ocp/releases) [![Buy Me a Coffee](https://img.shields.io/badge/Buy_Me_a_Coffee-ffdd00?logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/dtzp555)

> **Already paying for Claude Pro/Max? Use your subscription as an OpenAI-compatible API — $0 extra cost.**

*Open source from day one, used daily by my family, maintained on nights and weekends. If OCP saves you money too, you can [☕ buy me a coffee](https://buymeacoffee.com/dtzp555) — [full story below](#support-ocp).*

*If OCP saves you a setup, a ⭐ helps other folks discover it. Issue reports are even more useful — that's the highest-quality feedback this project gets.*

OCP turns your Claude Pro/Max subscription into a standard OpenAI-compatible API on localhost. Any tool that speaks the OpenAI protocol can use it — no separate API key, no extra billing.

```
Cline          ──┐
OpenCode       ───┤
Aider          ───┼──→ OCP :3456 ──→ Claude CLI ──→ Your subscription
Continue.dev   ───┤
OpenClaw       ───┘
```

One proxy. Multiple IDEs. All models. **$0 API cost.**

## Contents

- [Why OCP?](#why-ocp) · [Supported Tools](#supported-tools)
- [Quickstart](#quickstart)
- [How It Works](#how-it-works)
- Reference: [Available Models](#available-models) · [API Endpoints](#api-endpoints) · [Environment Variables](#environment-variables)
- Modes & operations: [LAN & multi-user](#lan--multi-user) → [`docs/lan-mode.md`](docs/lan-mode.md) · [Subscription-pool (TUI) mode](#subscription-pool-tui-mode) → [`docs/tui-mode.md`](docs/tui-mode.md) · [Upgrading](#upgrading) → [`docs/upgrading.md`](docs/upgrading.md)
- [Built-in Usage Monitoring](#built-in-usage-monitoring) · [Response Cache](#response-cache) · [Structured Outputs](#structured-outputs-openai-response_format) · [Images / Multimodal](#images--multimodal-vision) · [OpenClaw Integration](#openclaw-integration)
- [Troubleshooting](#troubleshooting) → [`docs/troubleshooting.md`](docs/troubleshooting.md)
- [Repository Layout](#repository-layout) · [Security](#security) · [Governance](#governance) · [Support OCP](#support-ocp) · [License](#license)

## Why OCP?

There are several Claude proxy projects. OCP picks a specific lane: **align tightly with what `cli.js` actually does, observe + multiplex what's already there, don't extend the protocol.** What you get:

- **LAN multi-user keys** (v3.7.0) — reach one Claude Pro/Max subscription from your own devices across the LAN. Each device gets a per-key API token (no OAuth session leak), with independent usage tracking and one-line revocation. Pro/Max are **per-user** accounts — see [Sharing with family / a team — honest limits](docs/lan-mode.md#deployment-model--security-read-this) before extending access to other **people**.
- **`ocp-connect` one-shot client setup** — one command on the client machine auto-configures OpenClaw, and detects Cursor, Cline, Continue.dev, and opencode to print ready-to-paste setup hints for each. No hunting for where each tool keeps its `OPENAI_BASE_URL`.
- **Response cache with per-key isolation + singleflight** (v3.13.0). Optional SHA-256 prompt cache, isolated per API key (cross-user pollution is impossible by hash construction, not by application logic), with stampede protection on concurrent identical prompts. Off by default. ([PR #65](https://github.com/dtzp555-max/ocp/pull/65), [PR #66](https://github.com/dtzp555-max/ocp/pull/66))
- **Per-key request quotas** (v3.8.0). Daily / weekly / monthly limits per key — set a kid's iPad to 20/day, a partner's laptop to 100/week. ([PR #18](https://github.com/dtzp555-max/ocp/pull/18))
- **SSE heartbeat for long reasoning** ([v3.12.0](https://github.com/dtzp555-max/ocp/releases/tag/v3.12.0), opt-in). If you've ever watched your IDE die at the 60s idle mark during a long Claude tool-use pause — that's nginx/Cloudflare default behavior. OCP emits an SSE comment frame to keep the connection alive without polluting the response. ([PR #49](https://github.com/dtzp555-max/ocp/pull/49))
- **`cli.js` alignment + CI guardrail.** LLM-assisted code drifts easily — it's tempting to invent plausible-looking endpoints that `cli.js` doesn't actually use. [`ALIGNMENT.md`](./ALIGNMENT.md) is binding: every endpoint OCP forwards from `cli.js` (Class A) must cite a `cli.js` line. OCP's own compatibility surface — the OpenAI-compatible and administrative endpoints, which have no `cli.js` analogue — is Class B: it cites OpenAI's published specification (the OpenAI-compatible endpoints) or its authorizing ADR (the administrative ones), under the same anti-invention discipline ([ADR 0006](./docs/adr/0006-openai-shim-scope.md)). The [`alignment.yml`](./.github/workflows/alignment.yml) CI workflow blocks PRs that introduce known-hallucinated tokens. The payoff is boring: your setup keeps working when `cli.js` ships its next minor.
- **`models.json` single source of truth** (v3.11.0). Adding a model is one file edit; both `/v1/models` and the OpenClaw bootstrap derive from it. ([PR #30](https://github.com/dtzp555-max/ocp/pull/30))
- **Drives the official CLI as-is, no binary patching.** OCP spawns the official `claude` CLI (or hosts it in an interactive tmux pane for TUI mode) — it does not extract OAuth tokens from memory, patch the binary, or invent protocol extensions. Traffic therefore looks like genuine Claude Code to Anthropic's classifiers (`cc_entrypoint=cli`). See `ALIGNMENT.md` for why this constraint is load-bearing.

### Comparison

OCP and the alternatives serve adjacent but distinct needs. Pick the one that fits your use case:

| Feature | OCP | claude-code-router | anthropic-proxy |
|---|---|---|---|
| Forwards Claude Code subscription as OpenAI API | yes | yes | yes |
| Routes to multiple model backends (OpenAI, Gemini, etc.) | no | yes | partial |
| SSE heartbeat for long reasoning | yes (opt-in) | no | no |
| Per-key quota + LAN multi-user keys | yes | no | no |
| Response cache | yes (opt-in) | no | no |
| OpenClaw / IDE auto-config | yes | no | no |
| Model-routing rules / model-switching | no | yes | no |
| GitHub stars / ecosystem size | small | large | mid |
| Governance discipline (CI-enforced alignment with cli.js) | yes | n/a | n/a |

**Plain English**: `claude-code-router` is the routing-and-switching power tool — pick it if you want to mix Anthropic, OpenAI, Gemini, and local models behind one endpoint. `anthropic-proxy` is the minimal forwarder. **OCP focuses on disciplined `cli.js`-aligned forwarding plus subscription multiplexing** — pick it if you want to reach one Claude Pro/Max subscription from your own IDEs and devices, with LAN auth, quotas, and a governance contract that prevents endpoint drift.

### Related: OLP — Open LLM Proxy

OCP is Claude-only by design. If you want to spread across **multiple LLM providers** (not just Claude), see the sibling project **[OLP — Open LLM Proxy](https://github.com/dtzp555-max/olp)**: the same spawn-the-provider-CLI approach, but across several provider CLIs behind one OpenAI-compatible endpoint, with intelligent fallback chains. It grew out of OCP in response to Anthropic's 2026-06-15 billing split — the idea being to spread subscription/quota risk across more than one provider. OCP remains the focused, Claude-only option; OLP is the multi-provider one.

OCP is single-maintainer + LLM-assisted, currently pre-1.0. It runs the maintainer's daily Claude Code workflow. If something breaks, [open an issue](https://github.com/dtzp555-max/ocp/issues).

## Supported Tools

Any tool that accepts `OPENAI_BASE_URL` works with OCP:

| Tool | Configuration |
|------|--------------|
| **Cline** | Settings → `OPENAI_BASE_URL=http://127.0.0.1:3456/v1` |
| **OpenCode** | `OPENAI_BASE_URL=http://127.0.0.1:3456/v1` |
| **Aider** | `aider --openai-api-base http://127.0.0.1:3456/v1` |
| **Continue.dev** | config.json → `apiBase: "http://127.0.0.1:3456/v1"` |
| **OpenClaw** [^openclaw] | `setup.mjs` auto-configures |
| **Any OpenAI client** | Set base URL to `http://127.0.0.1:3456/v1` |

[^openclaw]: **OpenClaw** is an IDE-agnostic AI coding agent (sibling project to OCP). When OCP runs on the same machine, OpenClaw can use it as a local provider — see `scripts/sync-openclaw.mjs` and ADR 0004.

## Quickstart

The simplest path: ask your AI.

  Paste this prompt to Claude Code / Cursor / Copilot:

  ```
  Install OCP for me. Read README §Quickstart and follow it.
  Tell me when I need to run `claude auth login`.
  ```

The AI will run `git clone`, `npm install`, `node setup.mjs`, and tell you when to OAuth.

**Prerequisites:** macOS or Linux (Windows is not supported), Node.js 22.13+ or 23.4+ (Node 24+ is what CI and the reference fleet run), `git`, and the [Claude CLI](https://docs.anthropic.com/en/docs/claude-cli) — new enough to have `--system-prompt-file`, which OCP passes on every spawn since v3.32.0. There is still no *version* floor — no minimum has been established, so this statement is deliberately version-free — but since [#455](https://github.com/dtzp555-max/ocp/issues/455) OCP **checks the capability at boot** and refuses to start if your CLI rejects a flag it passes, so you will find out at startup rather than on your first request — authenticated:

```bash
npm install -g @anthropic-ai/claude-code
claude auth login   # prints a URL + code — open on any browser, sign in, paste code back
```

**Install** (Server role — runs the proxy):

```bash
git clone https://github.com/dtzp555-max/ocp.git
cd ocp
node setup.mjs
```

`setup.mjs` verifies the Claude CLI, starts the proxy on port 3456, and installs auto-start (launchd on macOS, systemd on Linux). The `ocp` CLI lands at `~/ocp/ocp` — symlink it onto your PATH (`sudo ln -sf ~/ocp/ocp /usr/local/bin/ocp`, or `ln -sf ~/ocp/ocp ~/.local/bin/ocp`) or alias it (`alias ocp=~/ocp/ocp`); the rest of the docs assume `ocp` is on your PATH.

**Verify** — should list exactly the ids in [`models.json`](./models.json) (a count written here would go stale with every model added):

```bash
curl http://127.0.0.1:3456/v1/models
# claude-opus-5-5, claude-opus-5, claude-opus-4-8, claude-opus-4-7, claude-opus-4-6, claude-sonnet-5-5, claude-sonnet-5, claude-sonnet-4-6, claude-haiku-4-5-20251001
```

**Connect one IDE** — point any OpenAI-compatible tool at the proxy, then reload your shell and start a tool (Cline / Continue / Cursor / OpenCode):

```bash
export OPENAI_BASE_URL=http://127.0.0.1:3456/v1
```

See [Supported Tools](#supported-tools) for per-tool config.

**LAN / multi-user** — reach OCP from your own devices, with per-key auth, quotas, and anonymous access:

```bash
node setup.mjs --bind 0.0.0.0 --auth-mode multi
```

The full LAN server + client handbook, headless (Pi / NAS / VPS) OAuth, key/quota/anonymous-access management, AI-assisted install prompts, and the deployment/security model live in **[docs/lan-mode.md](docs/lan-mode.md)**. Claude Pro/Max are per-user accounts — read the [honest limits of sharing](docs/lan-mode.md#deployment-model--security-read-this) before extending access to other people.

### Uninstall

```bash
# From the cloned repo
node uninstall.mjs
```

Removes the launchd (macOS) or systemd (Linux) auto-start entry. Handles both legacy (`ai.openclaw.proxy` / `openclaw-proxy`) and current (`dev.ocp.proxy` / `ocp-proxy`) service names. Does not delete `~/.openclaw/`, `~/.ocp/`, or the cloned repo — remove those manually if desired.

## How It Works

```
Your IDE → OCP (localhost:3456) → claude --output-format stream-json CLI → Anthropic (via subscription)
```

OCP translates OpenAI-compatible `/v1/chat/completions` requests into `claude --output-format stream-json` CLI calls. Anthropic sees normal Claude Code usage — no API billing, no separate key needed.

> **Billing-policy status (as of 2026-07).** Anthropic announced (2026-05-14) that from 2026-06-15 the `claude -p` / Agent SDK path would move to a separate metered credit pool — then **paused the change on its effective date**: *"For now, nothing has changed: Claude Agent SDK, `claude -p`, and third-party app usage still draw from your subscription's usage limits"* ([official help article](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)). So the default path above currently bills your subscription. Anthropic has said it will give notice before any future change; if the split re-lands, OCP's opt-in [subscription-pool (TUI) mode](docs/tui-mode.md#subscription-pool-tui-mode) is the ready-made hedge — see the billing table there.

### Client-tools boundary

**OCP passes a client's OpenAI `tools` through to the model and returns its `tool_calls` (ADR 0022, v3.34.0).** A request that declares `tools` is served by its own path: the spawn is given exactly those tools through an MCP bridge, and when the model chooses one the response carries `choices[0].message.tool_calls` — `function.arguments` as a JSON **string**, per the spec — with `finish_reason: "tool_calls"`. The client runs the tool and sends the whole history back, including its `tool` message; OCP renders that history as text into the next spawn's prompt and the model answers or calls again. Every request is still one short-lived child; nothing is held across requests.

| Request | OCP |
|---|---|
| `tools` with no `tool_choice`, or `tool_choice` `"auto"` | **bridged** — the model holds exactly the declared tools (built-ins emptied); a call comes back as `tool_calls`, `finish_reason: "tool_calls"`; text comes back as before |
| `tool_choice` `"none"` | **served on the plain path** — the spec says the model will not call a tool, and text is the mandated outcome; not counted as dropped |
| `tool_choice` `"required"`, `{"type":"function"}`, `{"type":"custom"}`, or `allowed_tools` `mode: "required"` | **`400`** — the CLI cannot be forced, and a forced call the model does not make would be a silently wrong answer (ADR 0013's analysis, still in force) |
| legacy `functions` / `function_call` | `functions` is **dropped and counted** (reason `legacy_functions_shape`); `function_call: {"name": …}` is **`400`** |
| `tools` together with `response_format`, or on the **TUI lane** | **dropped and counted**, reason logged — see the boundary below |

Every drop increments `/health`'s `stats.toolRequestsDropped` and logs `openai_tools_dropped` with a `reason`; every bridged call increments `stats.toolCallsEmitted`. With `OCP_TOOL_CALLING=0` (see § Environment Variables) every declared tool is dropped and counted, which is the pre-3.34.0 behaviour.

**One limit worth knowing before pointing an agent at OCP.** A request that declares `tools` and asks for `stream: true` receives its response as SSE **once the turn completes**, not token by token.

**Images and tools work together** as of #477. They did not before: the multimodal spawn path did not render tool turns, so a vision agent was measured — with a real model — to re-call its tool indefinitely, and such requests were refused onto the plain path. That path now renders the same call/result/continuation text the text path does. Verified with a real model: fed its own tool result back with the image still in the history, the model answers from the result (no re-call) **and** describes the image.

The tools the bridge grants run on the **client**, so this works in every auth mode, including `AUTH_MODE=multi`: a guest calling its own tool touches nothing on the OCP host. What the model can do on the host itself is a separate question, governed by `--allowedTools` and `OCP_LOCAL_TOOLS` below — and when a request declares client tools, the host-side built-ins are emptied for that spawn so there is exactly one way to do each thing.

**What this means for choosing OCP (workload fit).** LAN/multi-device OCP is built for **chat-class** workloads — Q&A, translation, scripting against the API, chat frontends, home-automation backends — where text in/text out is the whole job. It is **not** the right tool for a coding agent running on a *client* machine that needs the AI to read and edit *that machine's* files: tools execute on the OCP host, so the model can never touch the client's filesystem. For that workload, run `claude` (or a local OCP) directly on the machine where the code lives.

## Available Models

| Model ID | Context window | Notes |
|----------|---------------:|-------|
| `claude-opus-5-5` | 1M | Most capable (default for `opus` alias) |
| `claude-opus-5` | 1M | Previous Opus, retained for pinning |
| `claude-opus-4-8` | 1M | Older Opus, retained for pinning |
| `claude-opus-4-7` | 1M | Older Opus, retained for pinning |
| `claude-opus-4-6` | 200k | Older Opus, retained for pinning |
| `claude-sonnet-5-5` | 1M | Latest Sonnet (default for `sonnet` alias) |
| `claude-sonnet-5` | 1M | Previous Sonnet, retained for pinning |
| `claude-sonnet-4-6` | 200k | Previous Sonnet, retained for pinning |
| `claude-haiku-4-5-20251001` | 200k | Fastest, lightweight (default for `haiku` alias) |

Context windows match the Claude Code CLI registry. Each model's prompt truncation ceiling is
derived from its own window (`contextWindow × 3` chars) — see [ADR 0011](docs/adr/0011-per-model-prompt-char-budget.md)
and `CLAUDE_MAX_PROMPT_CHARS` in [Environment Variables](#environment-variables).

The canonical list lives in [`models.json`](./models.json) — the single source of truth as of v3.11.0, validated in CI against [`models.schema.json`](./models.schema.json). Both `server.mjs` (the `/v1/models` endpoint) and `setup.mjs` (the OpenClaw registration) derive from it. Adding a new model is now a one-file edit:

```bash
# 1. Edit models.json — add an entry
# 2. Bump version, commit, tag, push
# 3. Users get it on next `ocp update`:
#    - OpenClaw: auto-synced via scripts/sync-openclaw.mjs
#    - Cline / Aider / Cursor / opencode: live /v1/models, picks up immediately
#    - Continue.dev: user edits their own config.json
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/models` | GET | List available models |
| `/v1/chat/completions` | POST | Chat completion (streaming + non-streaming) |
| `/health` | GET | Comprehensive health check (includes a `tui` block for TUI-mode drift/concurrency monitoring, an `auth` block — see § "What `auth.ok` means" — and `instanceName` — see § "Running more than one instance on a host") |
| `/usage` | GET | Plan usage limits + per-model stats |
| `/status` | GET | Combined overview (usage + health) |
| `/settings` | GET/PATCH | View or update settings at runtime |
| `/logs` | GET | Recent log entries (`?n=20&level=error`) |
| `/dashboard` | GET | Web dashboard (always public) |
| `/api/keys` | GET/POST | List or create API keys (admin only). `POST` body must be a JSON object — `{}` auto-names, `{"name":"…"}` names; anything else is `400` ([ADR 0017](./docs/adr/0017-api-keys-request-shape.md)) |
| `/api/keys/:id` | DELETE | Revoke an API key (admin only) |
| `/api/keys/:id/quota` | GET/PATCH | View or set per-key quota (admin only) |
| `/api/usage` | GET | Per-key usage stats (`?since=&until=&hours=&limit=`); returns self only by default — pass `?all=true` (admin only) for all-keys data |
| `/cache/stats` | GET | Cache statistics (admin only) |
| `/cache` | DELETE | Clear response cache (admin only) |

### Fields OCP accepts and does not act on

Sending one of these is **not an error** and never will be — a client that sets `temperature: 0` out of habit still gets an answer. But OCP does not honour them, and since 3.37.0 it says so: each such request increments `/health`'s **`stats.unhonouredFieldRequests`** and logs `openai_fields_not_honoured` at `info` with the field names.

| field | what happens |
|---|---|
| `n` (≠ 1) | one choice is returned |
| `logprobs`, `top_logprobs` | no `logprobs` in the response |
| `seed`, `stop`, `presence_pe