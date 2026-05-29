---
description: Generate a concise title for this Claude Code session and surface a /rename command to apply it (with macOS clipboard auto-copy).
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

**Once you have chosen a title, do exactly these two things in order. Do not skip step 1.**

1. On macOS, copy the `/rename <title>` line to the user's clipboard so they can paste-and-enter to apply it. Substitute the literal title text for `YOUR_TITLE_HERE` (keep the double quotes):

   ```sh
   printf '/rename %s' "YOUR_TITLE_HERE" | pbcopy 2>/dev/null || true
   ```

   On Linux/Windows this silently no-ops (no `pbcopy`); the user still gets the printed `/rename` line below.

2. Print a confirmation in this exact format (substituting the same title in both places):

   ```
   Title: YOUR_TITLE_HERE
   To apply, run (already on your clipboard if you're on macOS):
   /rename YOUR_TITLE_HERE
   ```

3. Do not ask the user for confirmation. Do not suggest alternative titles. Do not explain your reasoning. Generate, copy, print, stop.
