# My AI Dev Workflow Setup

How my Ubuntu machine is set up for working with Claude Code: GitHub tooling, agent helpers, parallel/overnight agents, and local voice dictation.

**Machine:** Ubuntu 24.04, GNOME on Wayland, AMD Radeon RX 6600 (8 GB VRAM)

---

## Contents

1. [Node.js (via nvm)](#1-nodejs-via-nvm)
2. [GitHub CLI and authentication](#2-github-cli-and-authentication)
3. [Agent skills](#3-agent-skills)
4. [AXI tools](#4-axi-tools)
5. [no-mistakes: AI-checked pushes](#5-no-mistakes-ai-checked-pushes)
6. [gnhf: overnight agent loops](#6-gnhf-overnight-agent-loops)
7. [treehouse: parallel worktrees](#7-treehouse-parallel-worktrees)
8. [firstmate: one agent managing a crew](#8-firstmate-one-agent-managing-a-crew)
9. [OpenWhispr: local voice dictation](#9-openwhispr-local-voice-dictation)
10. [Updating everything](#10-updating-everything)
11. [Usage and cost notes](#11-usage-and-cost-notes)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Node.js (via nvm)

Ubuntu's apt ships Node 18, which is too old. The `skills` CLI needs **Node 22.20+**, and the AXI tools need Node 20+. Error that shows the problem:

```
SyntaxError: The requested module 'node:util' does not provide an export named 'styleText'
```

Fix: install Node 22 with nvm (doesn't touch the system Node).

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc          # or ~/.zshrc on zsh
nvm install 22
nvm use 22
nvm alias default 22
node -v                   # should print v22.x
```

If `node -v` still shows v18, open a new terminal.

---

## 2. GitHub CLI and authentication

Install `gh` from the instructions at <https://cli.github.com>, then log in:

```bash
gh auth login             # GitHub.com → HTTPS → Login with a web browser
```

This login doesn't expire on a schedule, and it's what gh-axi, no-mistakes, and firstmate use. No manual token needed.

To print the token `gh` is using (if a tool asks for one):

```bash
gh auth token
```

### Personal Access Token (only if something specifically needs one)

GitHub never shows an existing token; you create one, and it's shown only once.

- github.com → profile picture → **Settings** → **Developer settings** → **Personal access tokens** (direct: <https://github.com/settings/tokens>)
- **Tokens (classic)** allows **No expiration**. Fine-grained tokens may cap the duration.
- Give it only the permissions needed (`repo` for general use). Delete it if it's ever exposed.

---

## 3. Agent skills

Skills are small instruction folders that Claude Code loads when relevant. Installed with the `skills` CLI (needs Node 22):

```bash
npx skills add anthropics/skills --skill skill-creator -g
```

Note: the GitHub org is **anthropics** (with an s). `-g` installs for all projects (`~/.claude/skills/`); without it, only for the current project.

---

## 4. AXI tools

AXI tools are command-line tools built for AI agents to use (token-efficient output, contextual hints). Installed globally with npm, plus session hooks so Claude gets their context at the start of every session:

```bash
npm install -g gh-axi lavish-axi tasks-axi quota-axi chrome-devtools-axi
gh-axi setup hooks && lavish-axi setup hooks && chrome-devtools-axi setup hooks
```

**Restart Claude Code after setting up hooks** so they take effect.

| Tool | What it does |
| --- | --- |
| `gh-axi` | Wraps the GitHub CLI for agents: issues, PRs, CI runs, releases. With the hook, Claude sees the current repo's open issues and PRs at session start. Needs `gh` logged in. |
| `lavish-axi` | Claude builds visual HTML pages (plans, diagrams, comparisons) and opens them in the browser. Click any part to leave feedback and send it back to Claude. Runs locally. In Claude Code: `/lavish let's discuss our plan here`. |
| `tasks-axi` | Same author's AXI family. See its repo for details. |
| `quota-axi` | Same author's AXI family. See its repo for details. |
| `chrome-devtools-axi` | Same author's AXI family. See its repo for details. |

Alternative (not used here): install only the skill, e.g. `npx skills add kunchenguid/gh-axi --skill gh-axi -g`. That version always runs the latest release via `npx` and needs no updating, but has no session-start context.

---

## 5. no-mistakes: AI-checked pushes

A checkpoint between my code and GitHub. Instead of `git push origin`, I push to `no-mistakes`: it reviews the change in a separate worktree, runs tests/lint, auto-fixes small issues, asks me about bigger ones, then pushes and opens a clean PR.

```bash
curl -fsSL https://raw.githubusercontent.com/kunchenguid/no-mistakes/main/docs/install.sh | sh
no-mistakes --version
```

Set up **per project**:

```bash
cd ~/path/to/project
no-mistakes init          # also installs the /no-mistakes skill for Claude Code
```

Three ways to use it:

- `git push no-mistakes` — push a committed branch through the checks
- `no-mistakes` — wizard to commit + push, then shows progress
- `/no-mistakes <task>` in Claude Code — Claude does the task and runs it through the gate

---

## 6. gnhf: overnight agent loops

"Good night, have fun." Runs Claude in a loop toward a goal: each round makes one small committed change on its own `gnhf/...` branch; failed rounds are rolled back. Wake up to a branch of commits plus notes.

```bash
npm install -g gnhf
```

Run from a git repo with a clean working tree:

```bash
gnhf "add input validation to every form and write a test for each" --max-iterations 10
```

Rules I follow:

- **Always set a limit** (`--max-iterations` or `--max-tokens`). By default it runs until stopped.
- **Specific goals.** "Improve the app" wanders.
- **Only on projects backed up on GitHub.** Claude runs without permission prompts.
- **Test with ~3 iterations while awake** before the first overnight run.
- It keeps the computer awake; leave it plugged in.
- `gnhf --worktree "..."` lets several runs work on one repo at once.

Telemetry opt-out (add to `~/.bashrc`):

```bash
export GNHF_TELEMETRY=0
```

---

## 7. treehouse: parallel worktrees

Lets several Claude sessions work on the same project at once, each in its own isolated copy (git worktree). Copies are reused from a pool with dependencies intact. Uses no Claude usage.

```bash
curl -fsSL https://kunchenguid.github.io/treehouse/install.sh | sh
```

```bash
cd ~/path/to/project
treehouse        # enter an isolated copy
claude           # work as usual
exit             # copy is reset and returned to the pool
```

**Before `exit`: commit to a branch and push.** The copy is reset when returned, so uncommitted or unpushed work can be lost.

Other commands: `treehouse status`, `treehouse prune` (dry run; add `--yes` to delete), `treehouse update`.

---

## 8. firstmate: one agent managing a crew

I talk to one Claude (the "first mate"); it spawns and supervises worker sessions, each in its own treehouse worktree, and brings back finished PRs. This one is **cloned**, because the folder itself is the tool.

```bash
sudo apt install tmux
cd ~
git clone https://github.com/kunchenguid/firstmate
cd ~/firstmate
claude
```

On first launch it checks the toolchain and asks before installing anything. Projects it works on are cloned into `~/firstmate/projects/`.

Built-in commands:

- `/bearings` — status summary of the crew
- `/ahoy` — catch up on what happened and pending decisions
- `/afk` — I'm away; keep supervising
- `/updatefirstmate` — update firstmate itself

---

## 9. OpenWhispr: local voice dictation

Press a hotkey, speak, and the text is typed at the cursor (including into Claude Code in the terminal). Transcription runs fully on my machine: free, unlimited, works offline.

### Install (AppImage)

```bash
mkdir -p ~/Applications
mv ~/Downloads/OpenWhispr-1.10.2-linux-x86_64.AppImage ~/Applications/
chmod +x ~/Applications/OpenWhispr-1.10.2-linux-x86_64.AppImage
~/Applications/OpenWhispr-1.10.2-linux-x86_64.AppImage
```

- FUSE error → `sudo apt install libfuse2t64`
- Sandbox crash → add `--no-sandbox` to the launch command
- AppImage limitation: only **email sign-in** works (no Google/Apple/Microsoft button). The `.deb` package supports browser sign-in.
- "App is not sandboxed" in GNOME's app info page is normal for AppImages.

### Auto-paste on Wayland (ydotool)

OpenWhispr always copies to the clipboard; auto-pasting on GNOME Wayland needs ydotool and access to `/dev/uinput`.

```bash
sudo apt install ydotool ydotoold
sudo usermod -aG input $USER
echo 'KERNEL=="uinput", GROUP="input", MODE="0660", TAG+="uaccess"' | sudo tee /etc/udev/rules.d/70-uinput.rules
sudo udevadm control --reload-rules && sudo udevadm trigger /dev/uinput
ls -l /dev/uinput         # should show group "input"
```

Log out and back in for the group change. Fallback if paste fails: **Ctrl+V** (or **Ctrl+Shift+V** in a terminal).

### Autostart at login

The in-app "launch at login" setting doesn't exist on Linux, so both OpenWhispr and the ydotool daemon start via GNOME autostart:

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/openwhispr.desktop << EOF
[Desktop Entry]
Name=OpenWhispr
Exec=$HOME/Applications/OpenWhispr-1.10.2-linux-x86_64.AppImage
Type=Application
X-GNOME-Autostart-enabled=true
EOF

cat > ~/.config/autostart/ydotoold.desktop << EOF
[Desktop Entry]
Name=ydotoold
Exec=ydotoold
Type=Application
X-GNOME-Autostart-enabled=true
EOF
```

Add `--no-sandbox` to the OpenWhispr `Exec=` line if needed. **When updating the AppImage, update the `Exec=` path** (or rename the file to `OpenWhispr.AppImage` once and point to that).

Optional app-menu entry: same format in `~/.local/share/applications/openwhispr.desktop`.

### Hotkey

- Default on GNOME is **F8** (GNOME can't take modifier-only combos like Ctrl+Super). On a laptop, try **Fn+F8**.
- Change it in **Settings → Hotkeys** (under App). Good alternatives: `Ctrl+Shift+K`, `Ctrl+Shift+J`.
- Avoid `Super+S`, `Ctrl+Alt+D`, `Alt+Space` (reserved on Linux).
- OpenWhispr registers it as a GNOME custom shortcut. Check with:
  ```bash
  gsettings get org.gnome.settings-daemon.plugins.media-keys custom-keybindings
  ```

### Models

**Settings → Speech-to-Text** (under AI Models) → **Dictation** tab → **Local**.

| Model | Notes |
| --- | --- |
| **Orukeet** | Recommended default. Multilingual, fast and accurate on CPU, no GPU setup. Try first. |
| **Whisper turbo** | Slimmed-down large: nearly as accurate, much faster. Use with GPU (below). |
| **Whisper large** | Most accurate Whisper, slowest. Only for hard audio. |
| **Parakeet / Nemotron** | Fast CPU alternatives; Nemotron streams text as you speak. |
| **Cohere Transcribe** | Most accurate offline, largest, no language auto-detect. |

**GPU acceleration (Whisper only):** in the model picker, use the GPU card's **Vulkan** download (covers AMD Radeon). Verify Vulkan sees the card:

```bash
sudo apt install mesa-vulkan-drivers vulkan-tools
vulkaninfo --summary | grep deviceName     # should list the RX 6600, not llvmpipe
```

### Cleanup (the main slowdown)

By default, dictation goes through a language model that removes filler words and fixes punctuation. Running locally on CPU, this was the bottleneck.

- Toggle: **Settings → Language Models** (under AI Models) → **Dictation Cleanup** → **Enable text cleanup**
- **For Claude Code: keep it off.** Claude understands raw speech fine, and dictation becomes near-instant.
- Turn it on for writing other people will read (emails).
- If kept on: use the smallest local model (~1B), or point **Self-Hosted** at a local GPU server (e.g. LM Studio with Vulkan).

### Other settings

- **Panel position:** Settings → Preferences → **Start position** → Bottom Center (the panel can also be dragged; it returns to the start position on relaunch).
- **Auto-hide when idle:** same section.
- The **assistant / AI models** in the local download section are only for cleanup and the voice agent. Not needed for plain dictation.

### Dictating into Claude Code

```bash
pgrep ydotoold || (ydotoold &)    # make sure the paste daemon is running
claude
```

Focus the terminal → press F8 → speak → press F8 → review → **Enter**. OpenWhispr pastes but doesn't send.

---

## 10. Updating everything

| Tool | Update command |
| --- | --- |
| Node.js | `nvm install 22 --reinstall-packages-from=current` |
| gh-axi, lavish-axi | `gh-axi update`, `lavish-axi update` |
| All npm-installed tools | `npm install -g gh-axi@latest lavish-axi@latest tasks-axi@latest quota-axi@latest chrome-devtools-axi@latest gnhf@latest` |
| no-mistakes | Rerun the install script |
| treehouse | `treehouse update` |
| firstmate | `/updatefirstmate` inside the firstmate session |
| OpenWhispr | Replace the AppImage (settings and models are kept), then update the autostart `Exec=` path |
| Skills from `npx skills add` | Rerun the same `npx skills add` command |

---

## 11. Usage and cost notes

| Tool | Uses Claude plan usage? |
| --- | --- |
| OpenWhispr (local) | **No.** Free, unlimited. Only the text I send to Claude counts, same as typing. |
| treehouse | No |
| gh-axi, lavish-axi, etc. | Only as part of normal Claude sessions (and they reduce token use) |
| no-mistakes | Yes: every gated push runs a Claude review |
| gnhf | **Heavy:** runs Claude non-stop and waits out limit resets to keep going |
| firstmate | **Heaviest:** several Claude sessions in parallel |

---

## 12. Troubleshooting

| Symptom | Fix |
| --- | --- |
| `styleText` error from `npx skills` | Node too old → section 1 |
| `command not found` after an install | Open a new terminal; check the install output for the install path |
| OpenWhispr hotkey does nothing | Try F8 / Fn+F8; check Settings → Hotkeys; try `Ctrl+Shift+K` |
| "ydotool is not fully configured" dialog | Run the ydotool steps in section 9, then restart OpenWhispr |
| Transcribed but nothing pasted | `pgrep ydotoold`; confirm `input` in `groups`; paste manually with Ctrl+V / Ctrl+Shift+V |
| Text pastes in editors but not the terminal | Terminal shortcut issue; press Ctrl+Shift+V manually |
| Dictation is slow | Turn off cleanup first; then try Orukeet or Whisper turbo + Vulkan |
| Check session type | `echo $XDG_SESSION_TYPE; echo $XDG_CURRENT_DESKTOP; groups` |

### Docs

- OpenWhispr: <https://docs.openwhispr.com>
- gh-axi: <https://github.com/kunchenguid/gh-axi>
- lavish-axi: <https://github.com/kunchenguid/lavish-axi>
- no-mistakes: <https://kunchenguid.github.io/no-mistakes/>
- gnhf: <https://github.com/kunchenguid/gnhf>
- treehouse: <https://github.com/kunchenguid/treehouse>
- firstmate: <https://github.com/kunchenguid/firstmate>
