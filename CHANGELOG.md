# Changelog

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
