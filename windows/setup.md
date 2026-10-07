# AI Dev Workflow Setup (Windows 11)

How to set up a Windows 11 machine for working with Claude Code: GitHub tooling, agent helpers, parallel/overnight agents, and local voice dictation.

All commands are for **PowerShell** (open **Terminal** from the Start menu; the prompt starts with `PS C:\`). You don't need to run as Administrator unless noted.

> **PowerShell tip:** the built-in Windows PowerShell 5.1 does not support `&&` between commands. Run commands one line at a time, as written below.

---

## Contents

0. [Base tools: Git, Node.js, GitHub CLI, Claude Code](#0-base-tools)
1. [PowerShell script permission](#1-powershell-script-permission)
2. [GitHub authentication](#2-github-authentication)
3. [Agent skills](#3-agent-skills)
4. [AXI tools](#4-axi-tools)
5. [no-mistakes: AI-checked pushes](#5-no-mistakes-ai-checked-pushes)
6. [gnhf: overnight agent loops](#6-gnhf-overnight-agent-loops)
7. [treehouse: parallel worktrees](#7-treehouse-parallel-worktrees)
8. [firstmate: one agent managing a crew (via WSL)](#8-firstmate-one-agent-managing-a-crew-via-wsl)
9. [OpenWhispr: local voice dictation](#9-openwhispr-local-voice-dictation)
10. [Updating everything](#10-updating-everything)
11. [Usage and cost notes](#11-usage-and-cost-notes)
12. [Troubleshooting](#12-troubleshooting)

---

## 0. Base tools

Install with **winget** (built into Windows 11):

```powershell
winget install Git.Git
winget install OpenJS.NodeJS.LTS
winget install GitHub.cli
```

**Close and reopen Terminal**, then check:

```powershell
git --version
node -v        # must be v22.20 or newer
gh --version
```

The `skills` CLI needs **Node 22.20+** and the AXI tools need Node 20+. If `node -v` is older, run `winget upgrade OpenJS.NodeJS.LTS`.

### Claude Code

```powershell
irm https://claude.ai/install.ps1 | iex
```

Open a new Terminal, then:

```powershell
claude --version
claude          # first run opens the browser to log in
```

Git for Windows (installed above) gives Claude Code Git Bash for running commands. The native install updates itself in the background. `claude doctor` checks the installation if something seems off.

---

## 1. PowerShell script permission

npm-installed tools run through small PowerShell scripts, which Windows blocks by default. If you see *"running scripts is disabled on this system"*, run once:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

---

## 2. GitHub authentication

```powershell
gh auth login       # GitHub.com → HTTPS → Login with a web browser
```

This login doesn't expire on a schedule, and it's what gh-axi, no-mistakes, and firstmate use. No manual token needed.

To print the token `gh` is using (if a tool asks for one):

```powershell
gh auth token
```

### Personal Access Token (only if something specifically needs one)

GitHub never shows an existing token; you create one, and it's shown only once.

- github.com → profile picture → **Settings** → **Developer settings** → **Personal access tokens** (direct: <https://github.com/settings/tokens>)
- **Tokens (classic)** allows **No expiration**. Fine-grained tokens may cap the duration.
- Give it only the permissions needed (`repo` for general use). Delete it if it's ever exposed.

---

## 3. Agent skills

Skills are small instruction folders that Claude Code loads when relevant:

```powershell
npx skills add anthropics/skills --skill skill-creator -g
```

The GitHub org is **anthropics** (with an s). `-g` installs for all projects (`%USERPROFILE%\.claude\skills\`); without it, only for the current project.

---

## 4. AXI tools

AXI tools are command-line tools built for AI agents to use (token-efficient output, contextual hints). Install globally, then add session hooks so Claude gets their context at the start of every session:

```powershell
npm install -g gh-axi lavish-axi tasks-axi quota-axi chrome-devtools-axi
gh-axi setup hooks
lavish-axi setup hooks
chrome-devtools-axi setup hooks
```

**Restart Claude Code after setting up hooks** so they take effect.

| Tool | What it does |
| --- | --- |
| `gh-axi` | Wraps the GitHub CLI for agents: issues, PRs, CI runs, releases. With the hook, Claude sees the current repo's open issues and PRs at session start. Needs `gh` logged in. |
| `lavish-axi` | Claude builds visual HTML pages (plans, diagrams, comparisons) and opens them in the browser. Click any part to leave feedback and send it back to Claude. Runs locally. In Claude Code: `/lavish let's discuss our plan here`. |
| `tasks-axi` | Same author's AXI family. See its repo for details. |
| `quota-axi` | Same author's AXI family. See its repo for details. |
| `chrome-devtools-axi` | Same author's AXI family. See its repo for details. |

Alternative: install only the skill, e.g. `npx skills add kunchenguid/gh-axi --skill gh-axi -g`. That version always runs the latest release and needs no updating, but has no session-start context.

---

## 5. no-mistakes: AI-checked pushes

A checkpoint between your code and GitHub. Instead of `git push origin`, push to `no-mistakes`: it reviews the change in a separate worktree, runs tests/lint, auto-fixes small issues, asks about bigger ones, then pushes and opens a clean PR.

```powershell
irm https://raw.githubusercontent.com/kunchenguid/no-mistakes/main/docs/install.ps1 | iex
no-mistakes --version
```

The installer also sets up its background helper as a Task Scheduler task.

Set up **per project**:

```powershell
cd C:\path\to\project
no-mistakes init          # also installs the /no-mistakes skill for Claude Code
```

Three ways to use it:

- `git push no-mistakes` — push a committed branch through the checks
- `no-mistakes` — wizard to commit + push, then shows progress
- `/no-mistakes <task>` in Claude Code — Claude does the task and runs it through the gate

Check setup anytime with `no-mistakes doctor`. Telemetry opt-out: see section 6's method with `NO_MISTAKES_TELEMETRY`.

---

## 6. gnhf: overnight agent loops

"Good night, have fun." Runs Claude in a loop toward a goal: each round makes one small committed change on its own `gnhf/...` branch; failed rounds are rolled back. Wake up to a branch of commits plus notes.

```powershell
npm install -g gnhf
```

Run from a git repo with a clean working tree:

```powershell
gnhf "add input validation to every form and write a test for each" --max-iterations 10
```

Rules to follow:

- **Always set a limit** (`--max-iterations` or `--max-tokens`). By default it runs until stopped.
- **Specific goals.** "Improve the app" wanders.
- **Only on projects backed up on GitHub.** Claude runs without permission prompts.
- **Test with ~3 iterations while awake** before the first overnight run.
- It keeps the PC awake during the run; on a laptop, leave it plugged in.
- `gnhf --worktree "..."` lets several runs work on one repo at once.

Telemetry opt-out (permanent, for your user; reopen Terminal afterward):

```powershell
[Environment]::SetEnvironmentVariable("GNHF_TELEMETRY", "0", "User")
[Environment]::SetEnvironmentVariable("NO_MISTAKES_TELEMETRY", "0", "User")
```

---

## 7. treehouse: parallel worktrees

Lets several Claude sessions work on the same project at once, each in its own isolated copy (git worktree). Copies are reused from a pool with dependencies intact. Uses no Claude usage.

```powershell
irm https://kunchenguid.github.io/treehouse/install.ps1 | iex
```

```powershell
cd C:\path\to\project
treehouse        # enter an isolated copy
claude           # work as usual
exit             # copy is reset and returned to the pool
```

**Before `exit`: commit to a branch and push.** The copy is reset when returned, so uncommitted or unpushed work can be lost.

Other commands: `treehouse status`, `treehouse prune` (dry run; add `--yes` to delete), `treehouse update`.

---

## 8. firstmate: one agent managing a crew (via WSL)

You talk to one Claude (the "first mate"); it spawns and supervises worker sessions, each in its own worktree, and brings back finished PRs.

**firstmate supports macOS and Linux only**, and relies on tmux. On Windows, run it inside **WSL** (a Linux environment built into Windows).

### Set up WSL (one time)

In PowerShell **as Administrator**:

```powershell
wsl --install
```

Restart when asked. Ubuntu opens and asks you to create a Linux username and password.

### Inside Ubuntu (WSL)

WSL is a separate Linux system, so its tools are installed separately from the Windows ones above. In the Ubuntu terminal:

```bash
# Node 22 via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install 22
nvm alias default 22

# Claude Code, GitHub CLI, tmux
curl -fsSL https://claude.ai/install.sh | bash
sudo apt install gh tmux
gh auth login

# treehouse and no-mistakes (Linux versions)
curl -fsSL https://kunchenguid.github.io/treehouse/install.sh | sh
curl -fsSL https://raw.githubusercontent.com/kunchenguid/no-mistakes/main/docs/install.sh | sh

# firstmate
cd ~
git clone https://github.com/kunchenguid/firstmate
cd ~/firstmate
claude
```

On first launch it checks the toolchain and asks before installing anything. Projects it works on are cloned into `~/firstmate/projects/` inside WSL. Keep them there (not under `/mnt/c/`) for speed.

Built-in commands:

- `/bearings` — status summary of the crew
- `/ahoy` — catch up on what happened and pending decisions
- `/afk` — you're away; keep supervising
- `/updatefirstmate` — update firstmate itself

---

## 9. OpenWhispr: local voice dictation

Press a hotkey, speak, and the text is typed at the cursor (including into Claude Code in the terminal). Transcription runs fully on your machine: free, unlimited, works offline.

Windows is the simplest platform for OpenWhispr: **no extra setup for pasting, no permissions to grant.**

### Install

1. Download the `.exe` installer from <https://github.com/OpenWhispr/openwhispr/releases/latest>
2. Run it. It creates desktop and Start Menu shortcuts.
3. If **SmartScreen** warns about it: click **More info → Run anyway** (it's code-signed by Gizmo Labs Inc.; newly signed apps get flagged until they build reputation).
4. The onboarding wizard walks through the rest.

If no audio is captured: **Windows Settings → Privacy & security → Microphone**, and make sure desktop apps are allowed.

### Start with Windows

OpenWhispr → **Settings → Preferences** → turn on **Launch at login** (and **Start minimized** if you want it to stay in the tray). It runs from the system tray; if the icon seems missing, click the `^` arrow on the taskbar.

Closing the window doesn't quit the app; it keeps listening for the hotkey. Quit from the tray icon's menu.

### Hotkey

- Default: **Ctrl+Win**.
- **Hold-to-talk** works on Windows: hold the keys while speaking, release to stop. Or **Tap** mode: press to start, press again to stop. Set under **Settings → Hotkeys → Activation Mode**.
- Windows-reserved shortcuts (Win+E, Win+L, Alt+Tab, etc.) are refused. Good alternatives: `Ctrl+Alt`, `Ctrl+Shift+K`.

### Models

**Settings → Speech-to-Text** (under AI Models) → **Dictation** tab → **Local**.

| Model | Notes |
| --- | --- |
| **Orukeet** | Recommended default. Multilingual, fast and accurate on CPU, no GPU setup. Try first. |
| **Whisper turbo** | Slimmed-down large: nearly as accurate, much faster. Use with GPU (below). |
| **Whisper large** | Most accurate Whisper, slowest. Only for hard audio. |
| **Parakeet / Nemotron** | Fast CPU alternatives; Nemotron streams text as you speak. |
| **Cohere Transcribe** | Most accurate offline, largest, no language auto-detect. |

**GPU acceleration (Whisper only):** in the model picker's GPU card, download **CUDA** for NVIDIA cards or **Vulkan** for AMD/Intel. To see which GPU you have:

```powershell
Get-CimInstance Win32_VideoController | Select-Object Name
```

### Cleanup (the main slowdown)

By default, dictation goes through a language model that removes filler words and fixes punctuation. Running locally on CPU, this is usually the bottleneck.

- Toggle: **Settings → Language Models** (under AI Models) → **Dictation Cleanup** → **Enable text cleanup**
- **For Claude Code: keep it off.** Claude understands raw speech fine, and dictation becomes near-instant.
- Turn it on for writing other people will read (emails).
- If kept on: use the smallest local model (~1B), or point **Self-Hosted** at a local GPU server (e.g. LM Studio).

### Other settings

- **Panel position:** Settings → Preferences → **Start position** → Bottom Center (the panel can also be dragged; it returns to the start position on relaunch).
- **Auto-hide when idle:** same section.
- The **assistant / AI models** in the local download section are only for cleanup and the voice agent. Not needed for plain dictation.

### Dictating into Claude Code

Open Terminal → `claude` → press the hotkey → speak → release (or press again) → review → **Enter**. OpenWhispr pastes but doesn't send.

OpenWhispr detects terminals (Windows Terminal, PowerShell, Command Prompt, Git Bash, and others) and uses **Ctrl+Shift+V** for them automatically.

---

## 10. Updating everything

| Tool | Update command |
| --- | --- |
| Node.js | `winget upgrade OpenJS.NodeJS.LTS` |
| Git, GitHub CLI | `winget upgrade Git.Git` and `winget upgrade GitHub.cli` |
| Claude Code | Automatic (or `claude update`) |
| gh-axi, lavish-axi | `gh-axi update`, `lavish-axi update` |
| All npm-installed tools | `npm install -g gh-axi@latest lavish-axi@latest tasks-axi@latest quota-axi@latest chrome-devtools-axi@latest gnhf@latest` |
| no-mistakes | `no-mistakes update` |
| treehouse | `treehouse update` |
| firstmate (WSL) | `/updatefirstmate` inside the firstmate session |
| OpenWhispr | Prompts at launch; installs on quit (version under Settings → System → Updates) |
| Skills from `npx skills add` | Rerun the same `npx skills add` command |

Tip: `winget upgrade --all` updates everything installed through winget at once.

---

## 11. Usage and cost notes

| Tool | Uses Claude plan usage? |
| --- | --- |
| OpenWhispr (local) | **No.** Free, unlimited. Only the text you send to Claude counts, same as typing. |
| treehouse | No |
| gh-axi, lavish-axi, etc. | Only as part of normal Claude sessions (and they reduce token use) |
| no-mistakes | Yes: every gated push runs a Claude review |
| gnhf | **Heavy:** runs Claude non-stop and waits out limit resets to keep going |
| firstmate | **Heaviest:** several Claude sessions in parallel |

---

## 12. Troubleshooting

| Symptom | Fix |
| --- | --- |
| "running scripts is disabled on this system" | Section 1: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| `The token '&&' is not a valid statement separator` | Windows PowerShell 5.1: run each command on its own line |
| `'X' is not recognized` after installing | Close and reopen Terminal so PATH refreshes |
| `styleText` error from `npx skills` | Node too old: `winget upgrade OpenJS.NodeJS.LTS` |
| Claude Code acting oddly | `claude doctor` |
| no-mistakes not working | `no-mistakes doctor` |
| OpenWhispr hotkey does nothing | Check Settings → Hotkeys; try `Ctrl+Shift+K`; another app may own the combo |
| OpenWhispr captures no audio | Windows Settings → Privacy & security → Microphone |
| Text pasted twice | Known Windows issue; see <https://docs.openwhispr.com/help/fix/text-pasted-twice.md> |
| Antivirus blocks OpenWhispr | See <https://docs.openwhispr.com/help/fix/antivirus-blocks-openwhispr.md> |
| Dictation is slow | Turn off cleanup first; then try Orukeet or Whisper turbo + GPU |
| firstmate errors on Windows | It needs WSL: section 8 |

### Docs

- Claude Code setup: <https://code.claude.com/docs/en/setup>
- OpenWhispr: <https://docs.openwhispr.com>
- gh-axi: <https://github.com/kunchenguid/gh-axi>
- lavish-axi: <https://github.com/kunchenguid/lavish-axi>
- no-mistakes: <https://kunchenguid.github.io/no-mistakes/>
- gnhf: <https://github.com/kunchenguid/gnhf>
- treehouse: <https://github.com/kunchenguid/treehouse>
- firstmate: <https://github.com/kunchenguid/firstmate>
