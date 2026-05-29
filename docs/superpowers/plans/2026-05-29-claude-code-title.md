# claude-code-title Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a Claude Code plugin that adds a `/title` slash command. The command generates a concise name for the current session from the conversation, sets the terminal tab title via OSC 0, and surfaces a `/rename <title>` line (auto-copied to the clipboard on macOS) for one-keystroke application of Claude Code's internal session label.

**Architecture:** Single-plugin marketplace repo. The plugin is one markdown file — `plugins/title/commands/title.md` — that Claude Code loads as a slash command. The prompt body instructs Claude to generate a title and run a single Bash command that does the terminal-title escape plus `pbcopy`. Two manifests (`plugin.json` and `marketplace.json`) make it installable; a README documents both the plugin and drop-in install paths.

**Tech Stack:** Markdown (slash command), JSON (manifests), Bash (`printf` for OSC 0 + `pbcopy`). No build, no tests beyond manual verification — there is no executable code to unit-test.

**Spec:** `docs/superpowers/specs/2026-05-29-claude-code-title-design.md`

**Working directory for all tasks:** `/Users/l.agrawal/Desktop/personal/claude-code-title/`
(Git repo already initialized; design spec already committed.)

**Note on TDD:** The project has zero executable logic. The unit of work is a markdown command file interpreted at runtime by Claude. We can't write meaningful pytest/unittest cases against it. **Task 5 is the "test"**: a manual verification script that exercises the plugin end-to-end in a real Claude Code session. Treat that task with the same rigor a TDD test would get — every check is a pass/fail line item.

---

## File Structure

| Path | Purpose |
|---|---|
| `.claude-plugin/marketplace.json` | Declares this repo as a single-plugin marketplace pointing to `./plugins/title`. |
| `plugins/title/.claude-plugin/plugin.json` | Plugin metadata: name, version, author, license. |
| `plugins/title/commands/title.md` | **The plugin.** Slash command frontmatter + prompt that tells Claude what to do when the user types `/title`. |
| `README.md` | User-facing: what it does, two install paths, usage example, known limits. |
| `LICENSE` | MIT. |
| `.gitignore` | macOS junk, editor cruft. |

---

## Task 1: Add `.gitignore` and `LICENSE`

**Files:**
- Create: `.gitignore`
- Create: `LICENSE`

- [ ] **Step 1: Write `.gitignore`**

File: `/Users/l.agrawal/Desktop/personal/claude-code-title/.gitignore`

```
.DS_Store
*.log
*.swp
.idea/
.vscode/
```

- [ ] **Step 2: Write `LICENSE` (MIT)**

File: `/Users/l.agrawal/Desktop/personal/claude-code-title/LICENSE`

```
MIT License

Copyright (c) 2026 Lakshya Agrawal

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: Commit**

```bash
cd /Users/l.agrawal/Desktop/personal/claude-code-title
git add .gitignore LICENSE
git commit -m "Add .gitignore and MIT LICENSE

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Write the plugin manifest

**Files:**
- Create: `plugins/title/.claude-plugin/plugin.json`

- [ ] **Step 1: Create the directory and file**

File: `/Users/l.agrawal/Desktop/personal/claude-code-title/plugins/title/.claude-plugin/plugin.json`

```json
{
  "name": "title",
  "version": "0.1.0",
  "description": "Manual /title slash command that names your Claude Code session based on conversation context",
  "author": {
    "name": "Lakshya Agrawal",
    "url": "https://github.com/aglakshya"
  },
  "homepage": "https://github.com/aglakshya/claude-code-title",
  "repository": "https://github.com/aglakshya/claude-code-title",
  "license": "MIT",
  "keywords": ["claude-code", "slash-command", "session", "title", "rename"]
}
```

- [ ] **Step 2: Validate JSON**

Run: `python3 -m json.tool plugins/title/.claude-plugin/plugin.json > /dev/null && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
cd /Users/l.agrawal/Desktop/personal/claude-code-title
git add plugins/title/.claude-plugin/plugin.json
git commit -m "Add plugin manifest for /title

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Write the marketplace manifest

**Files:**
- Create: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Create the file**

File: `/Users/l.agrawal/Desktop/personal/claude-code-title/.claude-plugin/marketplace.json`

```json
{
  "name": "claude-code-title",
  "owner": {
    "name": "Lakshya Agrawal",
    "url": "https://github.com/aglakshya"
  },
  "plugins": [
    {
      "name": "title",
      "source": "./plugins/title",
      "description": "Manual /title slash command for Claude Code session naming"
    }
  ]
}
```

- [ ] **Step 2: Validate JSON**

Run: `python3 -m json.tool .claude-plugin/marketplace.json > /dev/null && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
cd /Users/l.agrawal/Desktop/personal/claude-code-title
git add .claude-plugin/marketplace.json
git commit -m "Add marketplace manifest

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Write the `/title` slash command

**Files:**
- Create: `plugins/title/commands/title.md`

This is the substantive file. The frontmatter defines the slash command interface; the body is the prompt Claude sees when the user types `/title`.

- [ ] **Step 1: Create the directory and file**

File: `/Users/l.agrawal/Desktop/personal/claude-code-title/plugins/title/commands/title.md`

````markdown
---
description: Generate a concise title for this Claude Code session, set the terminal tab title, and surface a /rename command for the internal session label.
argument-hint: [optional hint to steer the title]
---

Generate a concise, descriptive title for the current Claude Code session based on our conversation so far.

**Title rules:**

- Maximum 50 characters.
- Sentence case with proper nouns capitalized (e.g., `Auto-rename plugin for Claude Code`). NOT `auto-rename-plugin` and NOT `Auto-Rename Plugin For Claude Code`.
- Capture what the session is *about*, not the medium (`Pension audit failures`, not `Debugging conversation`).
- No surrounding quotes. No trailing punctuation. Single line.
- No characters that would break a shell double-quoted string: avoid `"`, backticks, and `$`. If the natural title contains them, drop or rephrase.

**Optional hint from the user** (use only if the line below is non-empty — if blank, ignore it):

$ARGUMENTS

**Once you have chosen a title, do exactly these three things in order. Do not skip step 1.**

1. Run a single Bash command that sets the terminal tab title and attempts a macOS clipboard copy. Substitute the literal title text for `YOUR_TITLE_HERE` (keep the double quotes):

   ```sh
   printf '\033]0;%s\007' "YOUR_TITLE_HERE" && (printf '/rename %s' "YOUR_TITLE_HERE" | pbcopy 2>/dev/null || true)
   ```

   - The first `printf` emits the OSC 0 escape, which sets the OS-level terminal tab/window title. Silent no-op on terminals that don't support it.
   - The piped `pbcopy` is macOS-only. On Linux/Windows it fails silently because of `2>/dev/null || true`, which is fine — the user still gets the printed `/rename` line.

2. After the command returns, print a confirmation in this exact format (substituting the same title in both places):

   ```
   Title: YOUR_TITLE_HERE
   To apply to Claude Code's internal session label (already on your clipboard if you're on macOS), run:
   /rename YOUR_TITLE_HERE
   ```

3. Do not ask the user for confirmation. Do not suggest alternative titles. Do not explain your reasoning. Generate, run the command, print the confirmation, stop.
````

- [ ] **Step 2: Sanity-check the file**

Run from the project root:

```bash
test -f plugins/title/commands/title.md && head -5 plugins/title/commands/title.md
```

Expected: file exists, first line is `---`, second line is `description: ...`.

- [ ] **Step 3: Commit**

```bash
cd /Users/l.agrawal/Desktop/personal/claude-code-title
git add plugins/title/commands/title.md
git commit -m "Add /title slash command

Frontmatter defines the command interface (description + argument-hint).
Body is the prompt Claude executes: generate a <=50 char sentence-case
title, run a Bash command that sets the terminal tab title via OSC 0 and
attempts pbcopy on macOS, then print the /rename line for the user.

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Manual end-to-end verification

The plugin has no executable code to unit-test; correctness is observable only in a real Claude Code session. Treat each check as a pass/fail line item.

**Files:** none (verification only)

- [ ] **Step 1: Install the plugin from the local path**

Run in any Claude Code session (or in a separate terminal `claude` prompt):

```
/plugin marketplace add /Users/l.agrawal/Desktop/personal/claude-code-title
/plugin install title@claude-code-title
```

Expected: marketplace registers; plugin installs; no errors. Restart the Claude Code session if prompted.

- [ ] **Step 2: Verify the command shows up**

In Claude Code, type `/` and look for `title` in the slash command picker, or type `/title` and confirm tab completion works.
Expected: `/title` is listed with the description `Generate a concise title for this Claude Code session...`.

- [ ] **Step 3: Run `/title` with no argument in a session that has substantive context**

Resume or start a session, have a real conversation (a few user/assistant turns), then run `/title`.

**Pass criteria — all must hold:**
- (a) Claude runs one Bash tool call containing `printf '\033]0;` and `pbcopy`.
- (b) The terminal tab title visibly updates to the generated title (check the terminal tab bar / window title).
- (c) On macOS, `pbpaste` outputs `/rename <same title>` (verify in a separate shell: `pbpaste`).
- (d) Claude's text output contains both `Title: <title>` and a line starting with `/rename <title>`.
- (e) The title is ≤50 chars, sentence case, no surrounding quotes, no trailing punctuation. Verify by counting characters.

If any check fails, fix the prompt in `plugins/title/commands/title.md` and repeat.

- [ ] **Step 4: Run `/title focus on <some aspect>` to verify the hint path**

In the same session, run `/title focus on the manifest files`.

**Pass criteria:**
- (a) Title shifts to reflect the hint (e.g., emphasizes manifests over other topics that were in the conversation).
- (b) All checks from Step 3 (a–e) still hold.

If the hint is ignored, the prompt's `$ARGUMENTS` handling is wrong — fix and repeat.

- [ ] **Step 5: Apply the suggested `/rename` and verify Claude Code's internal label updates**

Paste the `/rename <title>` line (it's already on the clipboard on macOS) and press enter.

**Pass criteria:**
- The Claude Code session's internal label updates to the new title. Confirm by running `/resume` in a new shell and checking the picker. (Note: the exact persistence behavior is undocumented; if Claude Code doesn't persist the rename anywhere observable, that's a known limitation already noted in the spec — not a plugin defect.)

- [ ] **Step 6: Verify the drop-in install path also works**

Uninstall the plugin (`/plugin uninstall title@claude-code-title`), then drop the file in by hand:

```bash
mkdir -p ~/.claude/commands
cp /Users/l.agrawal/Desktop/personal/claude-code-title/plugins/title/commands/title.md ~/.claude/commands/title.md
```

Start a new Claude session and re-run Step 3's checks.

**Pass criteria:** `/title` works identically without the marketplace plumbing.

Clean up afterward:

```bash
rm ~/.claude/commands/title.md
```

- [ ] **Step 7: If any prompt-body changes were made, commit them**

```bash
cd /Users/l.agrawal/Desktop/personal/claude-code-title
git status
# If plugins/title/commands/title.md has been modified during verification:
git add plugins/title/commands/title.md
git commit -m "Refine /title prompt based on end-to-end verification

Co-Authored-By: Claude <noreply@anthropic.com>"
```

If no changes were needed, skip this step.

---

## Task 6: Write the README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write `README.md`**

File: `/Users/l.agrawal/Desktop/personal/claude-code-title/README.md`

````markdown
# claude-code-title

A Claude Code plugin that adds a `/title` slash command. Generates a concise name for your current session based on the conversation, sets your terminal tab title, and (on macOS) puts a ready-to-paste `/rename` command on your clipboard.

## Why

Claude Code sessions are labeled with auto-generated slugs like `fuzzy-munching-pancake`. When you have five Claude tabs open across different tasks, telling them apart is painful. `/title` solves this in one keystroke.

## What it does

When you type `/title` (optionally `/title focus on the auth refactor` to steer the result), Claude:

1. Picks a concise title from the conversation context (≤50 chars, sentence case).
2. Sets your terminal tab title via the OSC 0 escape sequence — works in iTerm2, macOS Terminal, GNOME Terminal, Konsole, Windows Terminal, Alacritty, Kitty.
3. On macOS, copies `/rename <title>` to your clipboard.
4. Prints the suggested `/rename` line so you can paste-and-enter to also update Claude Code's internal session label (visible in `/resume`).

## Install

### Recommended: via plugin marketplace

In Claude Code:

```
/plugin marketplace add aglakshya/claude-code-title
/plugin install title@claude-code-title
```

### Alternative: drop-in single file

If you don't want plugin machinery, just drop the command file in:

```sh
mkdir -p ~/.claude/commands
curl -L -o ~/.claude/commands/title.md \
  https://raw.githubusercontent.com/aglakshya/claude-code-title/main/plugins/title/commands/title.md
```

Both methods produce the same `/title` command.

## Usage

Inside a Claude Code session:

```
/title
```

Or with a steering hint when the session covers multiple topics:

```
/title focus on the bug fix
```

Output looks like:

```
Title: Auto-rename plugin for Claude Code
To apply to Claude Code's internal session label (already on your clipboard if you're on macOS), run:
/rename Auto-rename plugin for Claude Code
```

The terminal tab title updates immediately. To also apply Claude Code's internal session label, paste and press enter.

## Why two channels (terminal + `/rename`)?

The terminal tab title and Claude Code's internal session label are different artifacts:

- **Terminal tab title** is what your OS shows in the tab bar — useful when juggling sessions across tabs.
- **Claude Code's internal label** shows in the `/resume` picker.

Only the Claude Code harness can write the internal label; the model itself can't programmatically invoke `/rename` (it's intercepted before reaching the model). So this plugin updates the terminal title reliably, then surfaces the `/rename` line so you can apply the internal label in one paste-and-enter.

## Known limitations

- Terminals that don't support OSC 0 won't have their title updated (silent no-op). All mainstream emulators support it.
- Clipboard copy is macOS-only in the initial version (`pbcopy`). Linux/Windows users still get the printed `/rename` line — just no auto-clipboard.

## Contributing

Issues and PRs welcome. Especially: Linux (`xclip`/`wl-copy`) and Windows (`clip.exe`) clipboard support.

## License

MIT
````

- [ ] **Step 2: Spot-check rendering**

Run: `head -20 README.md && wc -l README.md`
Expected: header renders, file is roughly 60-80 lines.

- [ ] **Step 3: Commit**

```bash
cd /Users/l.agrawal/Desktop/personal/claude-code-title
git add README.md
git commit -m "Add README with install instructions and usage examples

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Final tree check and hand-off

- [ ] **Step 1: Verify the final repo structure**

Run from the project root:

```bash
cd /Users/l.agrawal/Desktop/personal/claude-code-title
find . -type f -not -path './.git/*' | sort
```

Expected output (exactly these eight files in the working tree):

```
./.claude-plugin/marketplace.json
./.gitignore
./LICENSE
./README.md
./docs/superpowers/plans/2026-05-29-claude-code-title.md
./docs/superpowers/specs/2026-05-29-claude-code-title-design.md
./plugins/title/.claude-plugin/plugin.json
./plugins/title/commands/title.md
```

- [ ] **Step 2: Verify git history**

Run: `git log --oneline`

Expected: 7-8 commits, all with `Co-Authored-By: Claude` trailers:

```
<sha> Add README with install instructions and usage examples
<sha> [optional] Refine /title prompt based on end-to-end verification
<sha> Add /title slash command
<sha> Add marketplace manifest
<sha> Add plugin manifest for /title
<sha> Add .gitignore and MIT LICENSE
<sha> Add implementation plan for claude-code-title
<sha> Add design spec for claude-code-title plugin
```

- [ ] **Step 3: Stop**

Do NOT run `git push` or `gh repo create`. Creating and pushing to GitHub is the user's call — they may want to choose a different repo name or visibility setting. Surface a final status message to the user with:

- The exact path: `/Users/l.agrawal/Desktop/personal/claude-code-title`
- The number of commits ahead of an empty remote
- Suggested next commands the user can run themselves: `gh repo create aglakshya/claude-code-title --public --source=. --remote=origin --push` (or equivalent).

---
