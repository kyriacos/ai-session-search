# ass — AI Session Search

Fuzzy-search your local AI chat history across **Claude Code** and **Cursor** from the terminal. Preview conversations with syntax-highlighted code blocks, search by project or message, filter by tool, and resume sessions with a single keypress.

---

## Requirements

| Tool | Purpose | Install |
|------|---------|---------|
| [fzf](https://github.com/junegunn/fzf) >= 0.45 | Fuzzy picker (`become` action) | `brew install fzf` |
| [bat](https://github.com/sharkdp/bat) | Code block syntax highlighting | `brew install bat` |
| Python 3 | Session parsing (stdlib only) | ships with macOS / most Linux |
| [Claude Code](https://claude.ai/code) | Claude sessions + `claude --resume` | `npm i -g @anthropic-ai/claude-code` |
| [Cursor](https://cursor.com) | Cursor sessions + resume via GUI or CLI | download from cursor.com |
| [Cursor Agent CLI](https://cursor.com/docs/cli) | `agent --resume` for terminal sessions | ships with Cursor or `curl ... \| bash` |

**Supported OS:** macOS and Linux. Windows is not supported.

---

## Installation

**One-liner (latest release, no git required):**

```bash
curl -fsSL https://github.com/kyriacos/ai-session-search/releases/latest/download/ass \
  -o ~/.local/bin/ass && chmod +x ~/.local/bin/ass
```

Make sure `~/.local/bin` is on your `PATH` — add `export PATH="$HOME/.local/bin:$PATH"` to your shell rc if needed.

**From source:**

```bash
# Copy the script somewhere on your PATH
cp ass ~/.local/bin/ass
chmod +x ~/.local/bin/ass

# Or symlink it if you cloned this repo
ln -sf "$(pwd)/ass" ~/.local/bin/ass
```

---

## Usage

```bash
ass
```

Opens the picker. Sessions are sorted newest-first and colour-coded by source:

- **cyan** — Claude Code
- **yellow** — Cursor

### Key bindings

| Key | Action |
|-----|--------|
| `F1` | Show Claude Code sessions only |
| `F2` | Show Cursor sessions only |
| `F3` | Show all sessions (reset source filter) |
| `F4` | Toggle date sort (newest / oldest first) |
| `F5` | Resume selected session |
| `F6` | Toggle Cursor resume target (GUI `cursor` / CLI `agent`) |
| `F7` | Filter by project or branch (prompt) |
| `F8` | Clear project/branch filter |
| `F10` | Toggle subagent sessions (hidden by default) |
| `Enter` | Print session identifier to stdout |
| `Ctrl-C` | Cancel |

**F5 resume behaviour:**

- Claude Code → runs `claude --resume <uuid>` in the current terminal
- Cursor (GUI, default) → runs `cursor --reuse-window <workspace-path>`
- Cursor (agent) → runs `agent --resume <composerId> --workspace <workspace-path>` in the current terminal

Press **F6** in the picker to toggle between GUI and agent mode before resuming a Cursor session.

### How search works

`ass` loads **all sessions on your machine** (not just the current directory) from Claude's `~/.claude/projects` and Cursor's database.

| Input | What it does |
|-------|----------------|
| **Search box** | Matches project name, git branch, and message text. Type multiple words to narrow further (e.g. `ai-session resume`). |
| **F7** | Lock the list to a project or branch — prompts on a separate line, then the search box is free for message text. |
| **F8** | Clear the F7 lock. |
| **F1 / F2 / F3** | Show Claude only, Cursor only, or both. |

### Project / branch filter (F7)

Press **F7** — you'll get a simple prompt below the picker (`Filter> `). Type a project or branch name and press Enter. The list reloads to matching sessions only, and the header shows `filter:…`. Press **F8** to clear.

While locked, use the search box to find specific messages within that project.

### Scripting

```bash
# Resume a Claude session directly (non-interactive)
claude --resume "$(basename "$(ass)" .jsonl)"

# Open the raw session file in your editor
$EDITOR "$(ass)"
```

---

## How it works

**Claude Code** stores sessions as JSONL files:

```
~/.claude/projects/<encoded-path>/<uuid>.jsonl
```

Each line is a JSON event; messages live at `obj.message.role` / `obj.message.content`. The project folder name is the workspace path with `/` and `.` replaced by `-` — the script reverses this to produce a readable label.

**Cursor** stores sessions in SQLite:

```
# macOS
~/Library/Application Support/Cursor/User/globalStorage/state.vscdb

# Linux
~/.config/Cursor/User/globalStorage/state.vscdb
```

Conversations are keyed as `composerData:<composerId>` and individual messages as `bubbleId:<composerId>:<bubbleId>` in the `cursorDiskKV` table.

The script merges both sources, sorts by modification time, and feeds the result into `fzf`. Each row carries a hidden plain-text search field (project, branch, message) so the search box matches real content rather than ANSI-coloured display text. The preview panel extracts fenced code blocks and pipes each through `bat` for per-language syntax highlighting.

---

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `CLAUDE_PROJECTS_DIR` | `~/.claude/projects` | Override Claude session directory |
| `CURSOR_GLOBALSTATE_DB` | OS default (see above) | Override Cursor DB path |
| `CURSOR_LAUNCH_MODE` | `gui` | Default Cursor resume target: `gui` or `agent` |

---

## Output format

On `Enter`, `ass` prints:

- **Claude** — absolute path: `/home/you/.claude/projects/.../uuid.jsonl`
- **Cursor** — `cursor:<composerId>:<workspacePath>`

The Cursor identifier encodes the workspace path after the second `:`. On macOS and Linux, paths cannot contain `:`, so splitting on `:` with a limit of 3 is safe.

---

## Known limitations

- **Cursor resume (GUI)** opens the project folder (`cursor --reuse-window <path>`); there is no CLI to jump to a specific chat session in the GUI.
- **Cursor resume (agent)** resumes the chat in the terminal via `agent --resume <composerId>`.
- **Zed** — the `threads.db` schema exists but sessions are stored as blobs in an undocumented format. Support can be added once the format is confirmed.
- **Windows** — paths and the Cursor DB location would need adaptation; WSL is untested.
