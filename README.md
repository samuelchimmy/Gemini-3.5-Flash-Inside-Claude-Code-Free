# Gemini 3.5 Flash Is Free — Here's How to Run It Inside Claude Code

Google quietly made its newest Flash model free-tier eligible. No credit card, a 1M token context window, and most people are still posting setup guides for the old 2.5 models.

## What you get for $0

- **Gemini 3.5 Flash** — Google's newest model, free tier eligible
- **Gemini 3.1 Flash-Lite** — for high-volume, cheap calls
- **Full 1M token context** on every free model
- **~1,500 requests/day**, native multimodal (text, image, audio, video)
- OpenAI-compatible endpoint available, but for Claude Code you'll use Google's native API through a router (more on that below)

## The quick version (2 minutes)

**Step 1: Grab a free API key**
- Go to [aistudio.google.com](https://aistudio.google.com)
- Sign in with any Google account
- Click **Get API key** — no billing setup required

**Step 2: Point a client at Gemini**
- Works with Cursor, Cline, Claude Code, or anything OpenAI/Anthropic-compatible via a proxy
- Or just prototype in the AI Studio UI and hit **Get code**

**Step 3: Pick your model**
- `gemini-3.5-flash` for the strong one
- `gemini-3.1-flash-lite` for cheap, high-volume steps (file reads, background tasks)

**Worth knowing:**
- Pro-tier Gemini models left the free tier on April 1, 2026 — only Flash and Flash-Lite are free now
- Free-tier prompts can be used by Google to improve their models — don't send sensitive data through it
- Rate limits are per-project; spinning up extra keys doesn't add quota
- The daily request cap resets every 24 hours — plenty for building and testing, not for production traffic

A frontier-class model, a 1M token context window, and a $0 bill, while most guides are still pointing people at last-gen models. The free tier shifts often, so treat exact numbers as approximate and check Google's [pricing page](https://ai.google.dev/gemini-api/docs/pricing) before you build anything that depends on them.

## The catch: Claude Code won't talk to Gemini directly

This is the part most quick-start threads skip.

Claude Code is hardcoded to send requests in Anthropic's **Messages API** format. Google's Gemini API expects its own **GenerateContent** format. Different request shape, different response shape — Claude Code has no idea how to talk to Gemini out of the box, even with a valid key pasted in.

To bridge that gap, you need something sitting between Claude Code and Google's API that translates requests both ways. That's what **Claude Code Router (CCR)** does — it's an open-source local proxy that intercepts Claude Code's Anthropic-formatted requests, rewrites them into whatever format the target provider needs (Gemini, DeepSeek, OpenRouter, local Ollama, etc.), sends them off, and translates the response back. Claude Code never knows anything changed.

That's the piece we're installing below.

---

## Full Setup Tutorial

### Prerequisites

- **Node.js 18+** installed ([nodejs.org](https://nodejs.org))
- A terminal: PowerShell or CMD on Windows, Terminal on macOS/Linux
- A free Gemini API key from [aistudio.google.com](https://aistudio.google.com) (Step 1 above)

Check your Node version first:

```bash
node -v
npm -v
```

If `node -v` fails, install Node.js from [nodejs.org](https://nodejs.org) before continuing (any recent LTS build works, on all three OSes).

---

### Step 1 — Install Claude Code and Claude Code Router

Same command on Windows, macOS, and Linux — both are npm packages:

```bash
npm install -g @anthropic-ai/claude-code
npm install -g @musistudio/claude-code-router
```

Verify both installed correctly:

```bash
claude --version
ccr -v
```

You should see version numbers for both. If `ccr` isn't recognized on Windows, close and reopen your terminal (npm's global bin folder needs to be picked up into PATH).

---

### Step 2 — Set your Gemini API key as an environment variable

Don't paste your raw key into `config.json` — reference it as an environment variable instead, so the file itself has no secrets in it.

**macOS / Linux (bash/zsh):**

```bash
echo 'export GOOGLE_API_KEY="your_key_here"' >> ~/.zshrc   # or ~/.bashrc
source ~/.zshrc
```

**Windows PowerShell** (persists across sessions):

```powershell
setx GOOGLE_API_KEY "your_key_here"
```
> Close and reopen PowerShell after `setx` for it to take effect.

**Windows CMD** (persists across sessions):

```cmd
setx GOOGLE_API_KEY "your_key_here"
```

To confirm it's set:

```powershell
# PowerShell
$env:GOOGLE_API_KEY
```
```cmd
:: CMD
echo %GOOGLE_API_KEY%
```
```bash
# macOS/Linux
echo $GOOGLE_API_KEY
```

---

### Step 3 — Create the CCR config file

CCR reads its config from a folder in your home directory:

| OS | Config path |
|---|---|
| macOS/Linux | `~/.claude-code-router/config.json` |
| Windows | `%USERPROFILE%\.claude-code-router\config.json` |

Create the folder:

```bash
# macOS/Linux
mkdir -p ~/.claude-code-router
```
```powershell
# PowerShell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude-code-router"
```
```cmd
:: CMD
mkdir "%USERPROFILE%\.claude-code-router"
```

Now create `config.json` inside that folder. This step stays entirely inside the terminal — no text editor, no separate file to open. Pick the block for your shell below and paste it **directly at your terminal prompt** (right where you'd normally type a command):

**Windows PowerShell:**

PowerShell can't do a bash-style heredoc, so use `Set-Content` with a here-string instead. Copy this **whole block** as one paste — it's a single PowerShell command:

```powershell
@'
{
  "LOG": true,
  "LOG_LEVEL": "info",
  "HOST": "127.0.0.1",
  "PORT": 3456,
  "API_TIMEOUT_MS": 600000,
  "Providers": [
    {
      "name": "gemini",
      "api_base_url": "https://generativelanguage.googleapis.com/v1beta/models/",
      "api_key": "$GOOGLE_API_KEY",
      "models": [
        "gemini-3.5-flash",
        "gemini-3.1-flash-lite"
      ],
      "transformer": {
        "use": ["gemini"]
      }
    }
  ],
  "Router": {
    "default": "gemini,gemini-3.5-flash",
    "background": "gemini,gemini-3.1-flash-lite",
    "think": "gemini,gemini-3.5-flash",
    "longContext": "gemini,gemini-3.5-flash",
    "longContextThreshold": 60000,
    "claude-3-5-sonnet": "gemini,gemini-3.5-flash",
    "claude-3-5-haiku": "gemini,gemini-3.1-flash-lite",
    "claude-3-7-sonnet": "gemini,gemini-3.5-flash"
  }
}
'@ | Set-Content -Path "$env:USERPROFILE\.claude-code-router\config.json" -Encoding utf8
```

Everything from `@'` down to `'@ | Set-Content...` is **one command** — paste the entire block at once, hit Enter, and it writes the file in a single shot. No separate save step.

**Windows CMD:**

CMD has no heredoc syntax, but `type con` lets you type/paste multi-line input straight to a file from the prompt:

```cmd
type con > "%USERPROFILE%\.claude-code-router\config.json"
```

Press Enter, then paste this JSON:

```json
{
  "LOG": true,
  "LOG_LEVEL": "info",
  "HOST": "127.0.0.1",
  "PORT": 3456,
  "API_TIMEOUT_MS": 600000,
  "Providers": [
    {
      "name": "gemini",
      "api_base_url": "https://generativelanguage.googleapis.com/v1beta/models/",
      "api_key": "$GOOGLE_API_KEY",
      "models": ["gemini-3.5-flash", "gemini-3.1-flash-lite"],
      "transformer": { "use": ["gemini"] }
    }
  ],
  "Router": {
    "default": "gemini,gemini-3.5-flash",
    "background": "gemini,gemini-3.1-flash-lite",
    "think": "gemini,gemini-3.5-flash",
    "longContext": "gemini,gemini-3.5-flash",
    "longContextThreshold": 60000
  }
}
```

Then press Enter once more, followed by **Ctrl+Z** and Enter, to save and close the file.

**macOS / Linux:**

```bash
cat > ~/.claude-code-router/config.json << 'EOF'
{
  "LOG": true,
  "LOG_LEVEL": "info",
  "HOST": "127.0.0.1",
  "PORT": 3456,
  "API_TIMEOUT_MS": 600000,
  "Providers": [
    {
      "name": "gemini",
      "api_base_url": "https://generativelanguage.googleapis.com/v1beta/models/",
      "api_key": "$GOOGLE_API_KEY",
      "models": ["gemini-3.5-flash", "gemini-3.1-flash-lite"],
      "transformer": { "use": ["gemini"] }
    }
  ],
  "Router": {
    "default": "gemini,gemini-3.5-flash",
    "background": "gemini,gemini-3.1-flash-lite", 
    "think": "gemini,gemini-3.5-flash",
    "longContext": "gemini,gemini-3.5-flash",
    "longContextThreshold": 60000,
    "claude-3-5-sonnet": "gemini,gemini-3.5-flash",
    "claude-3-5-haiku": "gemini,gemini-3.1-flash-lite",
    "claude-3-7-sonnet": "gemini,gemini-3.5-flash"
  }
}
EOF
```

This is also a single paste — everything between `<< 'EOF'` and the closing `EOF` writes to the file when you hit Enter.

**Verify the file was written correctly (any OS):**

```powershell
# PowerShell
Get-Content "$env:USERPROFILE\.claude-code-router\config.json"
```
```cmd
:: CMD
type "%USERPROFILE%\.claude-code-router\config.json"
```
```bash
# macOS/Linux
cat ~/.claude-code-router/config.json
```

You should see the JSON printed back exactly as written. If it looks empty or truncated, redo the paste — a partial paste is the most common cause of a broken config.

**What each `Router` key does:**
- `default` — normal coding requests, edits, chat
- `background` — cheap, high-frequency stuff (file scans, status checks) — this is where you'll burn most of your daily request count, so it's pinned to Flash-Lite
- `think` — reasoning/planning-heavy tasks
- `longContext` — anything over `longContextThreshold` tokens gets routed here automatically

---

### Step 4 — Launch Claude Code through the router

From inside any project folder (not your home directory):

```bash
ccr code
```

This starts the CCR proxy in the background and launches Claude Code pointed at it. You should land in a normal Claude Code session — the difference is invisible from the UI.

**Verify it's actually using Gemini:**

Inside the Claude Code session, run:

```
/status
```

This should show the active model as `gemini-3.5-flash` (or whichever route fired). You can also just say `hi` and watch the response come back — if it works, you're talking to Gemini through Claude Code's interface.

**Prefer a GUI over hand-editing JSON?**

```bash
ccr ui
```

This opens a local web page for managing providers and routing rules without touching the config file directly.

---

### Switching models mid-session

Once inside Claude Code, you can force a specific model for the current session:

```
/model gemini,gemini-3.1-flash-lite
```

Useful if you want to burn through cheap background tasks on Flash-Lite and only reach for Flash when something actually needs it.

---

## Troubleshooting

**`ccr: command not found` / not recognized**
Reopen your terminal. On Windows, confirm npm's global folder is in PATH: `npm config get prefix`.

**"Timeout" or "connection refused" errors**
The CCR background service may not have started. Try:
```bash
ccr stop
ccr start
```
Then run `ccr code` again.

**"Rate limit exceeded" almost immediately**
Claude Code fires a lot of small background requests (file reads, status checks). Make sure your `background` route points at `gemini-3.1-flash-lite`, not `gemini-3.5-flash` — Flash-Lite has a much higher per-minute limit and is built for exactly this kind of chatter.

**Port 3456 already in use**
Something else is bound to that port, or a previous CCR process didn't shut down cleanly. Change `"PORT"` in `config.json` to something else (e.g. `3457`) and restart.

**Check the logs**
```bash
# macOS/Linux
cat ~/.claude-code-router/logs/*.log
```
```powershell
# PowerShell
Get-Content "$env:USERPROFILE\.claude-code-router\logs\*.log" -Tail 50
```

---

## A few things worth remembering

- This routes Claude Code's *conversation* through Gemini, not through Anthropic's API — you're using CCR entirely locally, and your Gemini key never touches Anthropic's servers.
- Free-tier Gemini traffic may be used by Google to improve their models. Don't run this against a codebase with secrets or sensitive data unless you're comfortable with that.
- Rate limits are per Google Cloud project, not per API key — generating a second key doesn't get you a second quota.
- The free tier changes shape often. Bookmark [Google's pricing page](https://ai.google.dev/gemini-api/docs/pricing) and re-check model names/limits if something in this guide stops matching what you see in AI Studio.

---

*Tested with `claude-code-router` v2.x and Claude Code CLI. If a step above doesn't match what you're seeing, it's most likely a version drift on one of the two — check `ccr -v` and `claude --version` against the current releases.*
