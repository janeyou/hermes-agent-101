# Changelog

## v1.3.0 — 2026-04-19

Critical fix: tool calling was silently broken for local models due to prompt bloat. The 9B model received ~12,600 tokens of overhead (dev docs, skills index, 30 tool schemas) before seeing the user's message — enough to completely suppress function calling. Reduced to ~4,500 tokens. Web search now works end-to-end.

### Config changes (Hermes Agent)
- **[P0] Fix: Prompt bloat killing tool calling** — `AGENTS.md` (25KB dev guide) was auto-loaded into every gateway session's system prompt because the gateway's working directory is the Hermes source tree. Renamed to `AGENTS.md.dev` so the context file scanner skips it. Saved ~4,600 tokens per request.
- **[P0] Fix: Skills index overwhelming 9B model** — the `<available_skills>` block (8,692 chars, ~2,200 tokens) listed every installed skill. Removed `skills` toolset from Telegram platform config. Skills can be re-enabled for larger models.
- **[P0] Fix: Telegram toolset too large** — `hermes-telegram` preset registers 30 tools (~9,400 tokens of JSON schema). Replaced with explicit list of 6 essential toolsets (web, terminal, file, memory, cronjob, clarify) = 10 tools (~3,650 tokens). 
- **Net result:** System prompt + tools overhead reduced from ~12,600 tokens to ~4,500 tokens (64% reduction). Qwen 3.5 9B now reliably calls `web_search` instead of roleplaying tool usage.

### Guide changes
- **[P0] Fix: Phase 4 auto-start** — replaced hand-written plist (wrong path `/opt/homebrew/bin/hermes`, wrong label) with correct `hermes gateway install` command. The actual binary lives at `~/.local/bin/hermes`, not `/opt/homebrew/bin/`.
- **[P0] Add: "Prompt budget" section** — new section documenting the ~5,000 token overhead ceiling for 9B models, what eats tokens (skills index, AGENTS.md, tool count), and how to diagnose/fix tool-calling failures.
- **[P1] Fix: PATH issue** — documented that Hermes installs to `~/.local/bin/` and how to add it to PATH. Hit this in real testing — `hermes: command not found` after install.
- **Local LLM section:** Added research-backed optimal model recommendation — Qwen 3.5 9B confirmed as best choice for 16GB (most stable tool calling, 256K context). Added 14B upgrade path and Gemma 4 as runner-up.
- **Web search (Phase 6):** Added "Why Tavily?" explainer comparing all 4 backends with free tier details. Added verification steps.
- **Troubleshooting:** Added "Bot talks about tools but never calls them" with root cause and fix.
- **Annotated config:** Expanded web backend + platform_toolsets comments.

## v1.2.0 — 2026-04-19

Major update based on real-world chat analysis (56-message Telegram session). See [improvement plan](notes/improvement-plan.md) for full details.

### Code changes (Hermes Agent)
- **[P0] Fix: max_tokens cap for OpenRouter fallback** — auto-caps to 4096 when provider is OpenRouter and max_tokens is unset. Prevents 402 errors that blocked every single message in the test session.
- **[P0] Add: $5 funding prompt** — when OpenRouter returns 402, emits one-time actionable message with link and cost context ("$5 buys ~1M tokens")
- **[P1] Add: Fallback mode config** — new `fallback_model.mode: auto|ask|off`. Mode `ask` prompts user before switching; mode `off` disables fallback entirely. New `/fallback` and `/nofallback` commands.
- **[P2] Add: Web search prerequisite warning** — detects news/current-events keywords in user messages and warns if no search backend is configured
- **[P2] Add: Context window overflow pre-check** — warns when user message exceeds 60% of context window before processing

### Config changes
- Set `fallback_model.mode: ask` (user must confirm before cloud fallback)
- Set `fallback_model.max_tokens: 4096` (prevents 402 on free tier)
- Added `web.backend: tavily` (enables web search for news tasks)

### Guide changes
- **Prerequisites:** Added pre-flight verification checklist (Ollama smoke test, OpenRouter $5 recommendation, Tavily setup)
- **Phase 5:** Complete rewrite — `/sethome` as Step 0, "What bot can/can't do" capabilities table, tasks split into "starter" (no tools) vs "tool-dependent" (with verification steps)
- **Troubleshooting:** Added "Error Message Decoder" with 8 common errors from real testing + "Annotated Config Template"
- **Things to Watch Out For:** Rewrote fallback section with mode table, /fallback and /nofallback command docs
- **Quick Reference:** Added /sethome, /fallback, /nofallback
- **Phase 6:** Fixed DuckDuckGo reference (not actually built into Hermes — replaced with Exa)

## v1.1.0 — 2026-04-19

Fixes from first real setup run.

- **Fix: Ollama provider config** — `provider: "ollama"` alias can silently route to OpenRouter. Added warning to use `provider: "custom"` with explicit `base_url` instead
- **Add: Fallback behavior warning** — Hermes auto-switches to cloud fallback without asking. Documented so users know before it costs them money
- **Add: OpenRouter credits warning** — free credits drain fast with agentic use; added link to check balance
- **Add: Troubleshooting rows** — "Local model sends to OpenRouter" and "Fallback triggers unexpectedly (HTTP 400)"
- **Fix: Config tip** in Option A noting the `ollama` alias gotcha

## v1.0.0 — 2026-04-19

Initial public release.

### What's in this guide

- **6-phase setup**: Prerequisites, Mac Mini initial setup, go headless, install the stack (Claude Code or manual), auto-start, first tasks, advanced tooling
- **Two install paths**: Option A (Claude Code automates it) and Option B (manual step-by-step)
- **Reference sections**: Why Hermes vs OpenClaw, cloud vs local models, hardware sizing, cost breakdown, key learnings, troubleshooting
- **Headless Mac Mini hardening**: SSH key auth, sleep prevention (`pmset`), static IP, FileVault tradeoffs, Time Machine backup
- **Advanced tooling**: Tavily search, Whisper voice, Hindsight memory, tokscale monitoring, skills ecosystem
- **Sources & credits**: Named attribution with X handles and links for all community contributors

### Key details

- Written for Mac Mini M4 (Apple Silicon) — all paths use `/opt/homebrew/`
- Hermes Agent v0.10.0 (April 2026)
- Recommended starter stack: Qwen3.5 9B local + Qwen3.6-Plus cloud via OpenRouter
- Secure token handling: `.env` file, not pasted into conversations
- Verified claims: GLM 5.1, Gemma 4, MiniMax M2.7 pricing, Qwen3.5 9B benchmarks (~12-13 tok/s on M4)
