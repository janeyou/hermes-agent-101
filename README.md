# Hermes Agent 101

A beginner-friendly guide to setting up Hermes Agent on a Mac Mini — from unboxing to a 24/7 self-improving AI agent you can message from your phone.

Based on community discussions, official docs, and real-world experience reports. Last updated: April 2026.

---

## Table of Contents

**Setup (do this)**
1. [Prerequisites](#prerequisites)
2. [Phase 1: Mac Mini Initial Setup](#phase-1-mac-mini-initial-setup)
3. [Phase 2: Go Headless](#phase-2-go-headless)
4. [Phase 3: Install the Stack](#phase-3-install-the-stack) — Option A: Claude Code does it / Option B: manual
5. [Phase 4: Auto-Start on Boot](#phase-4-auto-start-on-boot)
6. [Phase 5: First Tasks to Try](#phase-5-first-tasks-to-try)
7. [Phase 6: Advanced Tooling](#phase-6-advanced-tooling)

**Reference (read when needed)**
- [Why Hermes Agent?](#why-hermes-agent)
- [Cloud vs Local Models](#cloud-vs-local-models)
- [What a Mac Mini Can Handle](#what-a-mac-mini-can-handle)
- [Cost Breakdown](#cost-breakdown)
- [Key Learnings](#key-learnings)
- [Things to Watch Out For](#things-to-watch-out-for)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)

---

# Setup

## Prerequisites

Before you start, have these ready:

**Hardware:**
- Mac Mini M4 (16 GB base is enough to start — see [What a Mac Mini Can Handle](#what-a-mac-mini-can-handle) for upgrade paths)
- Monitor, USB keyboard, USB mouse (for initial setup only — goes headless after)
- Ethernet cable (strongly recommended for 24/7 stability over Wi-Fi)
- ~20 GB free disk space (models: 3–8 GB each, plus Hermes and tools)

**Accounts (create before touching the Mac Mini):**
- **Dedicated Apple ID** — don't use your personal one. Create a new iCloud account for the agent machine. This keeps your personal iCloud, contacts, and files isolated. If your phone number is already tied to an Apple ID, create the new one via icloud.com or directly during macOS Setup Assistant — both work around the phone number limitation.
- **Telegram bot token** — this is how you'll chat with your agent from your phone. To create one:
  1. Open Telegram on your phone, tap the search bar, search for **BotFather**
  2. Tap **Start**, then type `/newbot`
  3. BotFather will ask for a **display name** — this can be anything (e.g., "My Hermes Agent")
  4. Then it asks for a **username** — must end in `bot` and be unique (e.g., `janeys_hermes_bot`)
  5. BotFather replies with your **bot token** — a long string like `7123456789:AAH...`. Copy and save this somewhere safe. You'll need it during install.
- **Your Telegram user ID** — needed to restrict the bot so only you can use it. To find it:
  1. In Telegram, search for **@userinfobot**
  2. Tap **Start**, then type `/myid`
  3. It replies with your numeric ID (e.g., `123456789`). Save this — you'll add it during install.
- **OpenRouter API key** (optional, for cloud fallback) — free at openrouter.ai

**Security note:** Use a dedicated Apple ID, not your personal one. Community consensus is strong on this — you don't want an AI agent in the blast radius of your personal iCloud, photos, or passwords. For the same reason, don't store personal passwords on the agent machine.

---

## Phase 1: Mac Mini Initial Setup

*Requires monitor + keyboard. ~15 minutes.*

1. **Unbox and connect** — HDMI to monitor, USB keyboard/mouse, ethernet (preferred) or Wi-Fi

2. **macOS setup** — Create account using your dedicated Apple ID. Use a strong, unique password.

3. **System settings:**
   - System Settings > General > Sharing > **Remote Login** (SSH) — ON
   - System Settings > General > Sharing > **Screen Sharing** — ON
   - System Settings > Energy > **Prevent automatic sleeping when display is off** — ON
   - System Settings > Energy > **Start up automatically after a power failure** — ON
   - Open Terminal and run these commands to fully prevent sleep (the GUI setting alone isn't enough — the network interface can still sleep on a headless Mac Mini, causing "No route to host" errors):
     ```bash
     sudo pmset -a sleep 0
     sudo pmset -a disablesleep 1
     sudo pmset -a womp 1              # Wake for network access
     sudo pmset -a networkoversleep 1  # Keep network alive during sleep
     sudo pmset -a powernap 0          # Disable Power Nap (can interfere)
     ```
     Verify with `pmset -g` — you should see `sleep 0` and `womp 1`.
   - Note your Mac's IP: System Settings > Network > your connection > IP address (e.g., `192.168.1.xxx`)

4. **Set a static IP** (recommended) — In your router admin, assign a DHCP reservation for the Mac Mini's MAC address so the IP doesn't change after reboots.

5. **Enable auto-login** (optional) — System Settings > Users & Groups > Automatic Login. Note: this disables FileVault disk encryption. If you're running the Mac Mini on a home LAN behind a firewall, this is a reasonable tradeoff. If it's on an exposed network, keep FileVault on and accept that you'll need a keyboard after power loss.

6. **Enable Time Machine** (recommended) — Back up to an external drive or network volume. Your `~/.hermes/` directory (skills + memory database) is your accrued value — losing it erases everything the agent has learned. Note: if you disabled FileVault for auto-login, Time Machine backups won't be encrypted by default either. If the backup drive leaves your home, enable "Encrypt backups" in Time Machine settings.

---

## Phase 2: Go Headless

*~5 minutes.*

1. **Disconnect** monitor, keyboard, mouse — move the Mac Mini wherever you want it (shelf, rack, closet)

2. **Access from your main computer:**

| Method | Command / How | When to use |
|---|---|---|
| **SSH** | `ssh yourusername@192.168.1.xxx` | Terminal access — used for all remaining setup |
| **Screen Sharing** | Finder > Go > Connect to Server > `vnc://192.168.1.xxx` | When you need the full GUI |

3. **SSH hardening (recommended):** Switch to key-based auth and disable password login. This prevents brute-force attacks on your home LAN.

**Step 1 — from your main computer (not the Mac Mini):**

```bash
# Copy your SSH key to the Mac Mini
ssh-copy-id yourusername@192.168.1.xxx
# First time connecting? It will ask "Are you sure you want to continue connecting?"
# Type: yes
# Then enter the Mac Mini's password when prompted
```

**Step 2 — test key-based login before changing anything:**

```bash
# From your main computer, open a NEW terminal tab and SSH in
ssh yourusername@192.168.1.xxx
# If it logs in WITHOUT asking for a password, key auth is working
# If it still asks for a password, stop here and troubleshoot before continuing
```

**Step 3 — on the Mac Mini (via your SSH session), disable password auth:**

```bash
sudo sed -i '' 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config

# Restart SSH on the Mac Mini for the change to take effect
sudo launchctl unload /System/Library/LaunchDaemons/ssh.plist
sudo launchctl load /System/Library/LaunchDaemons/ssh.plist
```

> **Important:** Do steps 1 and 2 first. If you disable password auth before your key works, you'll need physical access (monitor + keyboard) to recover.

---

## Phase 3: Install the Stack

You have two options: let Claude Code do it for you (recommended), or do it manually. Both are done over SSH from your main computer.

### Option A: Let Claude Code set it up (recommended)

Dirk Paessler reported ~5 minutes hands-on out of 70 minutes total for a 3-instance OpenClaw setup via Claude Code. A single Hermes install is simpler — expect ~5 minutes hands-on (Homebrew + writing tokens), ~20–25 minutes total including model download. Claude Code asks the right questions before touching anything, catches tradeoffs you wouldn't think of, and handles debugging if something breaks during install.

**From your SSH session (already connected from Phase 2):**

```bash
# Install Homebrew (prompts for admin password — Claude Code can't handle interactive prompts)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# IMPORTANT: Add Homebrew to PATH (without this, "brew" won't be found)
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# Write your tokens to a file (so they never leave the machine)
mkdir -p ~/.hermes
cat > ~/.hermes/.env << 'EOF'
TELEGRAM_BOT_TOKEN=your-bot-token-here
TELEGRAM_ALLOWED_USERS=your-numeric-telegram-id
OPENROUTER_API_KEY=sk-or-your-key-here
EOF
chmod 600 ~/.hermes/.env

# Install Claude Code
brew install claude-code

# Let Claude Code take it from here
claude
```

> **Which Claude account?** Use your existing Claude Pro or Max account — no need for a separate one. Unlike OpenClaw (which has OAuth conflicts when the same account is logged in on multiple machines), Claude Code uses API-style auth that works fine alongside your personal Claude usage.

> **What happens when you type `claude`:** It will try to open a browser on the Mac Mini for login. Since you're running headless over SSH, this won't work unless you have Screen Sharing (VNC) open. **Two options:**
> - **Easiest:** Open Screen Sharing on your main computer (`vnc://192.168.1.xxx`) before typing `claude`, so you can see and interact with the login browser window on the Mac Mini
> - **Alternative:** Claude Code will also print a login URL in the terminal — copy that URL and paste it into a browser on your main computer to complete login there
>
> After the first login, Claude Code caches your auth token on the Mac Mini. You won't need to do this again.

> **Why write tokens to a file?** Pasting API keys directly into a Claude Code conversation sends them to Anthropic's servers and stores them in conversation history. Writing them to `~/.hermes/.env` first keeps secrets on the machine and out of transcripts.

**Give Claude Code this prompt:**

> Install Hermes Agent on this Mac Mini for me. I want:
> - Ollama as the local LLM provider with Qwen3.5 9B
> - Telegram gateway connected (read bot token from ~/.hermes/.env)
> - Telegram restricted to my user ID only (read TELEGRAM_ALLOWED_USERS from ~/.hermes/.env)
> - OpenRouter as cloud fallback with anthropic/claude-sonnet-4 (read API key from ~/.hermes/.env)
> - Auto-start on boot via launchd
> - Time Machine backup covering ~/.hermes/
>
> Please ask me anything you need to understand before starting. Then install everything, configure it, and verify it's working.

Claude Code will handle: installing Ollama, pulling the model, inspecting and running the Hermes installer, configuring `~/.hermes/config.yaml` (Ollama as primary, OpenRouter as fallback), setting up the Telegram gateway with user restriction, installing the gateway as a launchd service (`hermes gateway install`), and running end-to-end verification. It will ask clarifying questions before making decisions — like which cloud fallback model you prefer and whether to inspect the installer script first.

**Your hands-on time:** ~5 minutes (Homebrew install + writing tokens to `.env`). Claude Code handles the rest in ~20 minutes including the model download (~6 GB for Qwen3.5 9B).

**Note on cost:** Claude Code is a first-party Anthropic tool — it's not affected by the April 2026 third-party agent cutoff. Usage counts against your Claude Pro or Max subscription limits. A full install session typically uses a small fraction of daily limits.

**Why this is better than manual:** Claude Code flags non-obvious tradeoffs (memory system stability, security surfaces, isolation models), debugs issues in real-time, and doesn't skip steps. One user's Mac Mini setup hit a sqlite-vec crash during install — Claude Code diagnosed and fixed it autonomously.

### Option B: Manual install

*~20 minutes + model download time.*

From your SSH session on the Mac Mini:

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Add Homebrew to PATH (without this, "brew: command not found")
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# Install Ollama
brew install ollama

# Start Ollama as a background service (persists across reboots)
brew services start ollama

# Pull your local model (~6 GB download)
ollama pull qwen3.5:9b

# Install Hermes Agent
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# Run setup
hermes setup
```

**During `hermes setup`:**
- Choose **Quick setup** (recommended)
- Select Ollama as provider, or "Custom endpoint" > `http://127.0.0.1:11434/v1`
- Pick your model: `qwen3.5:9b`
- Set up Telegram gateway — enter your bot token from @BotFather
- Launch Hermes chat when prompted

**Add cloud fallback (optional but recommended):**

```bash
# OpenRouter gives you access to cloud models as needed
# Get free API key at openrouter.ai
echo 'OPENROUTER_API_KEY=sk-or-yourkey' >> ~/.hermes/.env
```

When a local task needs more power, tell Hermes: "switch to qwen/qwen3.6-plus-preview:free for this task." (OpenRouter model IDs use the `vendor/model` format — check [OpenRouter's Qwen page](https://openrouter.ai/qwen) for the current model ID, as preview tags change.)

**Verify everything works:**

```bash
brew services list          # Ollama should show "started"
hermes status               # Check Hermes is running
```

Message your Telegram bot from your phone — you should get a reply. If so, the stack is working. You now have three ways to reach your agent:

| Method | When to use |
|---|---|
| **SSH** → `hermes chat` | Terminal sessions, debugging |
| **Telegram** | Daily use — chat from your phone anywhere |
| **Screen Sharing** (`vnc://192.168.1.xxx`) | When you need the full macOS GUI |

---

## Phase 4: Auto-Start on Boot

Create a launch agent so Hermes starts automatically when the Mac Mini powers on:

```bash
cat > ~/Library/LaunchAgents/com.hermes.agent.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermes.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/hermes</string>
        <string>start</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/hermes.out.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/hermes.err.log</string>
</dict>
</plist>
EOF

# Load it
launchctl load ~/Library/LaunchAgents/com.hermes.agent.plist
```

> **Note:** The path `/opt/homebrew/bin/hermes` is correct for Apple Silicon (all M4 Mac Minis). If Hermes was installed elsewhere, check with `which hermes`.

**Backup reminder:** Your skills and memory live in `~/.hermes/`. Make sure Time Machine or your backup solution covers this directory. If you lose it, the agent loses everything it's learned.

---

## Phase 5: First Tasks to Try

Start simple, build confidence:

1. **Test basic chat** — Open Telegram, message your bot: "Hello, what can you do?"
2. **Test tool calling** — "Summarize this URL: [any article link]"
3. **Test multi-step tasks** — "Research the top 3 AI agent frameworks, compare them, and save a summary to ~/Documents/agents.md"
4. **Test memory** — Have a conversation, close the session. Next day, ask: "What were we working on yesterday?"
5. **Test skill creation** — Give it a complex task (5+ steps). Afterward, check `~/.hermes/skills/` — it should have auto-generated a reusable skill.
6. **Set up a cron job** — "Every day at 8am, summarize the top HN posts and send to Telegram"
7. **Write your SOUL.md** — SOUL.md is Hermes's persistent personality file — it defines who the agent is, how it should behave, and what it knows about you. Think of it as the agent's identity document. Tell Hermes about yourself, your work, your preferences. After chatting for a day, say: "Update your SOUL.md based on what you've learned about me." You can also edit `~/.hermes/SOUL.md` directly.

---

## Phase 6: Advanced Tooling

Once your basic setup is stable (~1 week), add these layers. Source: @ResearchWang's tested config list.

### Perception: Web + Documents

| Capability | Tool | Notes |
|---|---|---|
| **Single page scrape** | [Jina Reader](https://r.jina.ai/) | Prefix URL with `r.jina.ai/` — clean markdown |
| **Bulk scrape** | [Crawl4AI](https://github.com/unclecode/crawl4ai) | Open-source, local, Playwright-based |
| **Anti-bot bypass** | Scrapling | Built into Hermes as optional skill |
| **Web search** | [Tavily](https://tavily.com/) | 1,000 free searches/month, structured results |
| **Search fallback** | DuckDuckGo | Built into Hermes, zero cost |
| **Doc conversion** | [Pandoc](https://pandoc.org/) | PDF/DOCX/HTML/EPUB to Markdown |
| **PDF to Markdown** | [Marker](https://github.com/datalab-to/marker) | Better than Pandoc for complex PDFs |

```bash
# Set Tavily as search backend (1,000 free searches/month)
echo 'TAVILY_API_KEY=tvly-yourkey' >> ~/.hermes/.env
hermes config set web.backend tavily
```

### Expression: Voice + Image

| Capability | Tool | Cost |
|---|---|---|
| **Speech-to-text** | [Whisper](https://github.com/openai/whisper) (local) | Free — 99 languages, auto-transcribes Telegram voice messages |
| **Text-to-speech** | Edge TTS | Free — Hermes default |
| **Image generation** | [Fal.ai](https://fal.ai/) / FLUX | Free tier. Install: `hermes skills install black-forest-labs/skills` |

### Memory Upgrade

Hermes's built-in memory only stores what the model explicitly writes. [Hindsight](https://github.com/vectorize-io/hindsight) auto-extracts entities and relationships from every conversation — mention a project deadline on Monday, it remembers on Friday without being told. Self-hostable, supports DeepSeek/OpenAI APIs.

### Cost Control: Token Monitoring

Token burn is the #1 hidden cost. Make it visible:

| Tool | What it does |
|---|---|
| [tokscale](https://github.com/junhoyeo/tokscale) | `tokscale --hermes` — global token consumption at a glance |
| hermes-dashboard | Breaks down tokens by component: system prompt, tool definitions, message history. Referenced in [NousResearch/hermes-agent#4379](https://github.com/NousResearch/hermes-agent/issues/4379) |

### Skills Ecosystem

Don't write skills from scratch — install collections first, then customize:

| Source | What |
|---|---|
| [wondelai/skills](https://github.com/wondelai/skills) | 25 specialized skills (product, UX, marketing, strategy) — cross-platform |
| [awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) | 1,000+ curated skills from official dev teams + community |
| Hermes bundled | 40+ pre-installed (v0.10.0) |

### Self-Evolution

Wait until your system is stable (~2 weeks) before enabling. The self-evolution module uses optimization loops to improve prompts and behavior. Pair it with a verification cron job — prevents the optimizer from "improving" configs you haven't finished tuning.

### Resource Hub

- [awesome-hermes-agent](https://github.com/0xNyk/awesome-hermes-agent) — the single-link bookmark
- [Hermes Official Docs](https://hermes-agent.nousresearch.com/docs)

---

# Reference

## Why Hermes Agent?

### vs OpenClaw

| | Hermes Agent | OpenClaw |
|---|---|---|
| **Philosophy** | Self-improving — auto-generates skills after complex tasks | Toolbox — manual control, large skill marketplace |
| **Local models** | Excellent. Tight harness works well with small models (Qwen3.5 4B–27B) | Designed for frontier models first. Smaller models often struggle |
| **Memory** | Three-layer system: Frozen System Prompt (MEMORY.md + USER.md injected at session start), Episodic Skills (auto-generated procedures from experience), and FTS5 Session Search (full-text index of all past conversations with LLM summarization). Learns automatically | File-based (MEMORY.md). Requires manual config. Common complaint |
| **Setup** | `hermes setup` — works immediately | `openclaw onboard` — often called "config hell" |
| **Reliability** | More autonomous, better at one-shotting tasks | Tasks can die/stop. Sometimes imposes its own interpretation |
| **Skill creation** | Automatic — generates SKILL.md after complex tasks | Manual — write your own or install from marketplace |
| **Ecosystem** | Growing (~90K stars as of April 2026). 40+ bundled skills (v0.10.0) | Massive (~355K stars). Thousands of community skills |
| **Token usage** | Higher per task (front-loads context for autonomy) | Lower per task (more conservative prompting) |
| **Anthropic dependency** | None — model-agnostic from day one | Claude subscription cutoff (April 2026) disrupted users |

### What the community says

These are paraphrased from Reddit threads on r/LocalLLaMA, r/hermesagent, and r/openclaw (April 2026). Individual experiences vary:

- Hermes accomplishes in half a day what takes a week of OpenClaw configuration
- "OpenClaw did it first, but Hermes seems to be doing it right"
- Memory is the #1 reason people switch — Hermes actively learns from your patterns
- Token usage is higher — "that's why it one-shots things. You're paying for autonomy"
- Many power users run both: Hermes for reasoning/autonomy, OpenClaw for its larger ecosystem

**Balanced take:** OpenClaw has more skills, a bigger community, and will likely be the biggest long-term player. But Hermes works better today with local models and requires less configuration. If you're starting fresh with a Mac Mini, start with Hermes.

### A word on hype

Some Reddit commenters flagged that Hermes praise can be generic and astroturf-like. The ecosystem is new and moving fast. Test yourself rather than trusting hype — which is exactly what this guide helps you do.

---

## Cloud vs Local Models

| | Cloud | Local |
|---|---|---|
| **Reasoning** | Stronger (Claude Opus, GLM 5.1, Qwen3.6-Plus) | Limited by hardware |
| **Cost** | $0.06–$25/1M tokens. Heavy use: $100+/day | Free after hardware purchase |
| **Privacy** | Data leaves your machine | Everything stays local |
| **Context window** | Up to 1M tokens | Typically 32K–256K |
| **Availability** | Rate limits, outages, vendor risk | Always on, no limits |
| **Setup** | API key | Ollama + model download |

### Recommended hybrid setup

Start local, use cloud as fallback. Hermes + Ollama make switching seamless.

| Layer | Model | Cost | When to use |
|---|---|---|---|
| **Daily driver (local)** | Qwen3.5 9B via Ollama | Free | Routine tasks, chat, automations, skill building |
| **Power fallback (cloud)** | Qwen3.6-Plus via OpenRouter | Free (preview) | Complex reasoning, large codebases, multi-step coding |
| **Budget cloud** | MiniMax M2.7 via OpenRouter | ~$0.30 input / $1.20 output per 1M | When previews end; cheapest capable cloud option |
| **Heavy lifting (cloud)** | Claude Opus or GLM 5.1 | $5+ input / $25+ output per 1M | Enterprise tasks, deep debugging — use sparingly |

---

## What a Mac Mini Can Handle

### Model sizing for 16 GB Mac Mini M4

| Model | Size | Best for | Fits 16GB? |
|---|---|---|---|
| **Qwen3.5 4B** | ~3 GB | Light tasks — surprisingly capable with Hermes | Yes, fast |
| **Qwen3.5 9B** | ~6 GB | Solid daily driver, good reasoning | Yes, comfortable |
| **Gemma 4 E4B** | ~3 GB | Edge tasks, basic coding, native audio/video input | Yes, fast |
| **Gemma 4 26B-A4B** (MoE) | ~16 GB | Strong reasoning (only 4B active per token via MoE) | Tight, may swap |
| **Qwen3.5 27B** (Q4) | ~16 GB | Best local reasoning at this tier | Too tight |
| **Qwen3.6 local** | ~24 GB | Full local agent powerhouse | No — needs 48GB+ |

**Sweet spot for 16 GB:** Qwen3.5 9B. Community-validated with Hermes.

### Upgrade path (if you outgrow 16 GB)

| Machine | Memory | Qwen3.6 local? | Price |
|---|---|---|---|
| Mac Mini M4 (base) | 16 GB | No | $599 |
| Mac Mini M4 Pro | 24 GB | Barely — loads but no room for context | $1,399 |
| **Mac Mini M4 Pro** | **48 GB** | **Yes — comfortable** | **$1,999** |
| Mac Studio M4 Max | 36 GB | Yes | $1,999 |
| Mac Studio M4 Max | 64 GB | Yes, plenty of headroom | $2,999 |

**You don't need a Mac Studio.** The Mac Mini M4 Pro 48 GB ($1,999) is the right upgrade for running Qwen3.6 locally. A Mac Studio only makes sense if you want to run multiple large models simultaneously.

**But you probably don't need to upgrade yet.** Qwen3.6-Plus is free on OpenRouter right now. Start with the base Mac Mini + Qwen3.5 9B local + Qwen3.6-Plus cloud. Upgrade the hardware only if you're constantly hitting local limits after a month.

---

## Cost Breakdown

| Item | Cost |
|---|---|
| Mac Mini M4 (base, 16 GB) | $599 (one-time) |
| Electricity (24/7) | ~$20/year |
| Ollama + Hermes Agent + Qwen3.5 9B | Free |
| Qwen3.6-Plus cloud fallback (OpenRouter preview) | Free |
| MiniMax M2.7 cloud (when preview ends) | ~$2–5/month for light use |
| **Total first year (hybrid)** | **~$620–680** |

| Comparison | Annual cost |
|---|---|
| This setup (local + cloud hybrid) | ~$620–680 |
| Claude Pro subscription | $240/year (with usage limits) |
| OpenClaw on cloud-only models | $3,000–36,000+/year |

The Mac Mini is a one-time cost. After year one, it's essentially free for local-only use.

**If you upgrade to 48 GB for Qwen3.6 local:**

| Item | Cost |
|---|---|
| Mac Mini M4 Pro (48 GB) | $1,999 (one-time) |
| Everything else | Same as above |
| **Total first year** | **~$2,020** |

---

## Key Learnings

### 1. The harness matters more than the model
Same model (Qwen3.5) performs dramatically different in Hermes vs OpenClaw. Hermes's tighter prompt structure gets more out of smaller models. One community member summarized it: "OpenClaw was designed for LLMs first, and that strategy fails with SLMs."

### 2. Token usage is the hidden cost
Hermes burns more tokens per task because it front-loads context. This is why it one-shots things — it's not smarter, it's spending more. People who trim Hermes's context in production report it starts behaving like OpenClaw. The autonomy-cost tradeoff is real.

### 3. Memory is the actual differentiator
Every comparison thread lands on memory as the #1 reason to switch. Hermes actively saves context and preferences. After a few days, users report it becomes proactive — predicting what you'll ask.

### 4. Self-evolution is real but controversial
Hermes auto-creates skills after complex tasks. Example: a user asked it to improve error handling, and it created a `resilient-execution` skill with fallback chains, error classification, and recovery patterns — then updated its own memory.

### 5. Running both is a legitimate strategy
Many power users run Hermes + OpenClaw side-by-side. They can share memory databases. When OpenClaw crashes, Hermes can revive its config.

### 6. Ollama integration is the game changer
`ollama launch hermes` handles installation, model selection, provider config, and messaging setup. This removed the biggest barrier to entry.

---

## Things to Watch Out For

### Security
Hermes supports multiple execution backends: local (default), Docker, SSH, and others. **The default local backend does not sandbox.** If you want isolation before giving the agent access to credentials, explicitly configure Docker backend with a read-only root filesystem. Don't give any agent access to financial accounts or sensitive credentials without careful sandboxing.

**Dedicated accounts:** Use a separate Apple ID, a separate Google account (for OAuth), and a separate Telegram bot. Don't put your personal credentials on the agent machine. Community best practice: create a dedicated email (e.g., `yourname-hermes@gmail.com`) for all agent infrastructure.

**Restrict your Telegram bot.** Without `TELEGRAM_ALLOWED_USERS` in your `.env`, anyone who discovers your bot's username can chat with your agent. Always set it to your numeric Telegram user ID (find it via @userinfobot → `/myid`).

### Token costs with cloud models
Even "cheap" cloud models add up with heavy agentic use. Set spending limits on every API provider before you start. Start conservative ($20–50/month) and raise once you understand usage patterns.

### Session stability
Some users report sessions timing out or dying, especially with local models. Not widely reported — may be hardware or config specific. Monitor during the first week.

### Vendor risk
Anthropic cut off Claude subscriptions from third-party agents in April 2026. OpenClaw was most affected. Hermes was model-agnostic from day one, but the lesson applies: don't build critical workflows on a single provider.

### Backup
Your `~/.hermes/` directory contains skills, memory (SQLite database), and configs. This is the accrued value of a self-improving agent. Back it up. Time Machine works. So does a simple cron job:

```bash
# Daily backup of Hermes data
0 3 * * * tar czf ~/hermes-backup-$(date +\%Y\%m\%d).tar.gz ~/.hermes/
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| **`brew: command not found`** | Homebrew needs to be added to PATH after install. Run: `eval "$(/opt/homebrew/bin/brew shellenv)"` then `echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile` to make it permanent. |
| **Ollama not starting** | `brew services restart ollama`. Check logs: `cat /opt/homebrew/var/log/ollama.log` |
| **Model runs out of memory** | Switch to a smaller model: `ollama pull qwen3.5:4b`. Or close other apps. |
| **Hermes can't find Ollama** | Make sure Ollama is running: `curl http://127.0.0.1:11434/v1/models`. Re-run `hermes setup` if needed. |
| **Telegram bot not responding** | Check gateway: `hermes gateway status`. Re-run `hermes gateway setup` with your bot token. |
| **SSH "No route to host"** | Mac Mini's network went to sleep. Fix: `sudo pmset -a sleep 0 && sudo pmset -a disablesleep 1 && sudo pmset -a womp 1 && sudo pmset -a networkoversleep 1`. The GUI energy setting alone isn't enough for headless setups. |
| **SSH connection refused** | Verify Remote Login is enabled in System Settings > Sharing. Check firewall isn't blocking port 22. |
| **Launchd service not starting** | Check logs: `cat /tmp/hermes.err.log`. Verify path: `which hermes` should return `/opt/homebrew/bin/hermes`. |
| **Version mismatch** | Run `hermes --version` — this guide was written for v0.10.0 (April 2026). If you're on an older version, run `hermes update`. |
| **Slow inference** | Normal for local models. Qwen3.5 9B (Q4_K_M) on M4 16GB: expect ~12-13 tokens/sec decode. Smaller models (4B) are faster. MoE models like Qwen3.5-35B-A3B can actually be faster (~17 tok/s) since fewer parameters are active per token. |
| **Agent keeps asking for clarification** | Update SOUL.md: add "Execute tasks as instructed. Do not add suggestions or ask for clarification when the task is clear." |

### Updating

```bash
# Update Hermes
hermes update

# Update a model
ollama pull qwen3.5:9b    # re-pulls latest version

# Update Ollama itself
brew upgrade ollama
```

---

## Quick Reference

| Command | What it does |
|---|---|
| `ollama launch hermes` | Full guided setup via Ollama |
| `hermes setup` | Re-run setup wizard |
| `hermes start` | Start the agent |
| `hermes status` | Check if agent is running |
| `hermes chat` | Start a chat session |
| `hermes gateway setup` | Connect messaging (Telegram, Discord, etc.) |
| `hermes gateway status` | Check messaging gateway |
| `hermes claw migrate` | Import from OpenClaw |
| `hermes update` | Update Hermes |
| `ollama pull qwen3.5:9b` | Download/update a local model |
| `brew services start ollama` | Run Ollama as background service |
| `hermes config set web.backend tavily` | Set web search provider |
| `hermes skills install [repo]` | Install a skill collection |
| `tokscale --hermes` | Check token consumption |

---

## Sources & Credits

This guide was built from the knowledge and experience shared by these people. Thank you.

### Community guides & threads

- **[@Will_Yang_](https://x.com/Will_Yang_)** — [Hermes Agent 完全指南](https://x.com/Will_Yang_/status/2041507883876233312) — comprehensive Hermes guide covering migration from OpenClaw, memory, self-evolution
- **[@ResearchWang](https://x.com/ResearchWang)** — [Hermes 全部高阶工具配置](https://x.com/ResearchWang/status/2045812932538438001) — advanced tooling walkthrough: Tavily, Whisper, tokscale, Hindsight, SOUL.md
- **[@LufzzLiz](https://x.com/LufzzLiz)** — [上手 Hermes Agent 后建议先尝试的十件事情](https://x.com/LufzzLiz/status/2042237123865297267) — 10 things to try after installing Hermes
- **[@Saboo_Shubham_](https://x.com/Saboo_Shubham_)** — [How to run a 24/7 AI Agent that Grows with You](https://x.com/Saboo_Shubham_/status/2044241148794024293) and [How I built an Autonomous AI Agent team](https://x.com/Saboo_Shubham_/status/2022014147450614038) — 24/7 agent setup, Ollama integration
- **[@Mosescreates](https://x.com/Mosescreates)** — [All-in Hermes production stack](https://x.com/Mosescreates/status/2044224095546495192) — real-world production deployment
- **Marco Rodrigues** — [The 5 Best Models to Use with AI Agents](https://blog.dadhalfdev.com/p/the-5-best-models-to-use-with-ai) — model comparison and benchmarks for agent use

### Mac Mini setup

- **Dirk Paessler** — [Setting Up a Mac Mini with OpenClaw](https://dirkpaessler.blog/2026/03/11/setting-up-a-mac-mini-with-openclaw-a-step-by-step-guide/) — conversation-first Claude Code install, three-instance setup, gateway watchdog pattern
- **Robert H. Eubanks** — [OpenClaw on Mac Mini: The Complete Setup Guide](https://robertheubanks.substack.com/p/openclaw-on-mac-mini-the-complete) — detailed hardware walkthrough, headless setup, launchd config
- **[@clayhebert](https://x.com/clayhebert)** — [Dedicated accounts setup](https://x.com/clayhebert/status/2023459379404591299) — separate Apple ID, Gmail, Claude account, Google Voice tip (with [@cathrynlavery](https://x.com/cathrynlavery))

### Reddit communities

- [r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA/) — Hermes vs OpenClaw comparisons, local model benchmarks
- [r/hermesagent](https://www.reddit.com/r/hermesagent/) — migration experiences, memory and self-evolution discussion
- [r/openclaw](https://www.reddit.com/r/openclaw/) — side-by-side testing reports
- [r/clawdbot](https://www.reddit.com/r/clawdbot/) — Mac Mini security setup, Apple ID advice

### Official docs & tools

- [Hermes Agent — Nous Research](https://hermes-agent.nousresearch.com/)
- [Ollama Hermes integration](https://docs.ollama.com/integrations/hermes)
- [EvoMap Team — Hermes self-evolution analysis](https://evomap.ai/blog/hermes-agent-evolver-similarity-analysis) — note: EvoMap has accused Hermes of copying their self-evolution architecture; [see controversy](https://eu.36kr.com/en/p/3767967755371011)

---

