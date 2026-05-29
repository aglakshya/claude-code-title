# claude-code-title — Design Spec

**Date:** 2026-05-29
**Author:** Lakshya Agrawal (`aglakshya02`)
**Status:** v2 — amended after end-to-end verification surfaced an architectural blocker. See Revision history.

## Problem

Claude Code sessions are labeled with auto-generated slugs like `fuzzy-munching-pancake`. When users juggle multiple Claude Code terminals/sessions, there is no built-in way to give a session a meaningful name based on its actual contents. The built-in `/rename` command exists but requires the user to think up a title manually.

There are open community feature requests for automatic naming ([anthropics/claude-code#47176](https://github.com/anthropics/claude-code/issues/47176), [anthropics/claude-code#35945](https://github.com/anthropics/claude-code/issues/35945)) but no official implementation. We found no community plugins that do this.

## Goals

1. Let the user invoke a single slash command — `/title` — that picks a concise, descriptive name for the current session based on the conversation so far.
2. Make applying that name to **Claude Code's internal session label** (what `/rename` controls, what `/resume` shows) as close to one keystroke as the harness allows: at minimum, the exact `/rename <title>` line on the user's clipboard so applying is paste-and-enter.
3. Distribute as an installable Claude Code plugin under a personal GitHub repo, with a fallback drop-in path for users who don't want plugin machinery.

## Non-goals

- **Automatic / hook-driven renaming.** Considered and rejected during brainstorming. A manual command is simpler, cheaper, less surprising, and avoids titles changing under the user.
- **A native programmatic API for `/rename`.** Doesn't exist. `/rename` is intercepted by the harness before reaching the model, so the model cannot invoke it. Reverse-engineering where Claude Code persists user-set session titles is out of scope — undocumented, brittle, will break on harness upgrades.
- **Setting the OS terminal tab title from inside the plugin.** Initially planned, then cut. Claude Code's Bash tool runs without a controlling TTY (`tty` returns "not a tty"; `/dev/tty` is unavailable). Anything the tool emits — including `\033]0;...\007` OSC 0 escapes — goes to Claude Code's captured stdout and never reaches the host terminal. AppleScript / System Events automation works on macOS but requires per-app permission grants, which is unacceptable UX for a shareable plugin. See Decision log #6.
- **Cross-session title persistence on top of Claude Code's own store.** If Anthropic ships a public API later, this plugin can adopt it.
- **Tests, CI, type checking.** The plugin is a single markdown command file. Manual verification suffices.

## Design

### User-facing surface

A single slash command:

```
/title                              # generate title from full conversation context
/title focus on the auth refactor   # generate title, but bias toward the given aspect
```

When invoked, Claude:

1. Looks at the conversation in its context window.
2. Generates a title under 50 characters. Sentence case with proper nouns capitalized (e.g., `Auto-rename plugin for Claude Code`, not `auto-rename-plugin` and not `Auto-Rename Plugin For Claude Code`). No trailing punctuation, no surrounding quotes. Avoids characters that break a shell double-quoted string (`"`, backticks, `$`).
3. If the user passed a hint (the `$ARGUMENTS` placeholder is non-empty), biases the title toward that aspect.
4. On macOS, runs `printf '/rename <title>' | pbcopy` (silent failure on other OSes) so the user can paste-and-press-enter to apply the title to Claude Code's internal session label.
5. Outputs a short confirmation that includes the exact `/rename <title>` line for manual application.

### Why we can't auto-apply

`/rename` is intercepted by the Claude Code harness *before* the model receives the user's message. The model cannot emit `/rename <title>` in its response and have the harness pick it up — the interception only fires on direct user input. So the closest the plugin can get to "automatic" is staging the command on the clipboard so the user's apply step is one paste-and-enter.

We initially designed a second channel — setting the OS terminal tab title via OSC 0 — but verification revealed it doesn't work from inside a Claude Code Bash tool call (no controlling TTY). See Decision log #6 and Revision history.

### Repository layout

```
claude-code-title/
├── .claude-plugin/
│   └── marketplace.json                 # single-plugin marketplace manifest
├── plugins/
│   └── title/
│       ├── .claude-plugin/
│       │   └── plugin.json              # plugin manifest
│       └── commands/
│           └── title.md                 # the /title command — prompt + frontmatter
├── docs/
│   └── superpowers/
│       ├── specs/
│       │   └── 2026-05-29-claude-code-title-design.md  # this file
│       └── plans/
│           └── 2026-05-29-claude-code-title.md
├── README.md
├── LICENSE                              # MIT
└── .gitignore
```

### Key file: `plugins/title/commands/title.md`

Markdown command file with frontmatter:

```markdown
---
description: Generate a concise title for this Claude Code session and surface a /rename command to apply it.
argument-hint: [optional hint to steer the title]
---

<prompt body that instructs Claude to:
 1. Generate a title (constraints above)
 2. pbcopy the /rename <title> line on macOS (silent no-op elsewhere)
 3. Print the title and /rename line for the user>
```

The prompt body is the substantive logic; manifests are boilerplate.

### Installation paths (documented in README)

**Path A — Plugin (recommended)**

```
# In Claude Code
/plugin marketplace add aglakshya02/claude-code-title
/plugin install title@claude-code-title
```

**Path B — Drop-in (no plugin machinery)**

```
mkdir -p ~/.claude/commands
curl -L -o ~/.claude/commands/title.md \
  https://raw.githubusercontent.com/aglakshya02/claude-code-title/main/plugins/title/commands/title.md
```

Both produce the same `/title` command.

## Open assumptions / things to confirm during implementation

1. **Author identity.** Defaulting to `aglakshya02` / "Lakshya Agrawal" based on the user's git config. Easy to swap in `plugin.json`, `marketplace.json`, and README.
2. **Repo name.** `claude-code-title`. Alternatives were `claude-title`, `cc-title`, `auto-title`.
3. **License.** MIT.
4. **`pbcopy` fallback for Linux/Windows.** Initial version is macOS-only for the clipboard nicety; Linux/Windows users still get the printed `/rename` line. If demand exists, add `xclip`/`wl-copy`/`clip.exe` detection.

## Risks

- **Claude Code harness changes the slash command frontmatter schema or the `$ARGUMENTS` placeholder.** Low likelihood; documented surface. Mitigation: pin to current syntax, watch release notes.
- **User pastes `/rename` line without trimming.** Title gets quotes/newlines in it. Mitigation: the prompt instructs Claude to emit a clean, single-line `/rename` invocation and forbids characters that break a double-quoted shell string.

## Decision log

| # | Decision | Alternative considered | Why |
|---|----------|------------------------|-----|
| 1 | Manual `/title` command | Auto-rename via `Stop` / `UserPromptSubmit` hook | User chose manual; cheaper, less surprising, no titles shifting under the user. |
| 2 | Name: `/title` | `/retitle`, `/autoname`, `/summarize-session` | Shortest, frequent-use ergonomics. |
| 3 | Optional `$ARGUMENTS` hint | No args / explicit override | Zero cost to support, genuinely useful in mixed-topic sessions. Explicit override duplicates built-in `/rename`. |
| 4 | Plugin distribution via personal GitHub | Single file in `~/.claude/commands/` only | User wants to share with everyone, not just personal use. Drop-in path retained for low-friction users. |
| 5 | ~~Both terminal title AND `/rename` suggestion~~ → **`/rename` suggestion only** | Either alone | Originally chose both, then cut the terminal-title channel after verification proved it doesn't work from inside a Claude Code Bash tool call. See #6. |
| 6 | Cut the OS terminal tab title channel | (a) Keep with OSC 0 in spite of the TTY issue, (b) use AppleScript on macOS to talk to the terminal app, (c) require users to install a hook in their own settings.json | (a) Doesn't work — verified empirically. (b) Per-app macOS permission grants are unacceptable UX for a shareable plugin and only solve macOS. (c) Pushes setup burden onto every user and only helps for auto-rename, which was already declined in decision #1. Cutting cleanly is better than shipping a feature that mostly doesn't work. |

## Revision history

- **2026-05-29 v1** — Original design. Included two output channels: setting the OS terminal tab title via `printf '\033]0;<title>\007'` and `pbcopy`-ing the `/rename` line. Approved without exhaustive verification.
- **2026-05-29 v2** — During end-to-end verification in a real Claude Code session (Warp on macOS), the OSC 0 escape did not update the terminal title. Investigation showed Claude Code's Bash tool runs without a controlling TTY, so terminal escape sequences emitted from the tool reach Claude Code's captured stdout, not the host terminal. `/dev/tty` is also unavailable. The terminal-title channel was cut from the design; the plugin now does only the `/rename` + `pbcopy` channel. The command file, README, and this spec were updated accordingly. See Decision log #6.
