# claude-code-title

A Claude Code plugin that adds a `/title` slash command. Generates a concise name for your current session based on the conversation and (on macOS) puts a ready-to-paste `/rename` command on your clipboard so you can apply it in one keystroke.

## Why

Claude Code sessions are labeled with auto-generated slugs like `fuzzy-munching-pancake`. When you have five Claude tabs open across different tasks, telling them apart from the `/resume` picker is painful. `/title` solves this in one keystroke.

## What it does

When you type `/title` (optionally `/title focus on the auth refactor` to steer the result), Claude:

1. Picks a concise title from the conversation context (≤50 chars, sentence case).
2. On macOS, copies `/rename <title>` to your clipboard.
3. Prints the suggested `/rename` line so you can paste-and-enter to apply it.

After you paste-and-enter the `/rename` line, Claude Code's internal session label updates — which is what shows up in the `/resume` picker.

## Why doesn't it just rename the session automatically?

Because it architecturally can't. `/rename` is intercepted by the Claude Code harness *before* reaching the model, so the model can't programmatically invoke it. Similarly, terminal escape sequences (`\033]0;...`) emitted from inside a Claude Code Bash tool call go to Claude Code's captured stdout, not to your terminal — so we can't set the OS tab title that way either.

What we *can* do is generate the title and stage the `/rename` command on your clipboard so applying it is one paste-and-enter. That's what this plugin does.

## Install

### Recommended: via plugin marketplace

In Claude Code:

```
/plugin marketplace add aglakshya02/claude-code-title
/plugin install title@claude-code-title
```

### Alternative: drop-in single file

If you don't want plugin machinery, just drop the command file in:

```sh
mkdir -p ~/.claude/commands
curl -L -o ~/.claude/commands/title.md \
  https://raw.githubusercontent.com/aglakshya02/claude-code-title/main/plugins/title/commands/title.md
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
To apply, run (already on your clipboard if you're on macOS):
/rename Auto-rename plugin for Claude Code
```

Then paste-and-enter the `/rename` line to apply.

## Known limitations

- Clipboard auto-copy is macOS-only in the initial version (`pbcopy`). Linux/Windows users still get the printed `/rename` line — just no auto-clipboard, so you need to copy it yourself.

## Contributing

Issues and PRs welcome. Especially: Linux (`xclip`/`wl-copy`) and Windows (`clip.exe`) clipboard support.

## License

MIT
