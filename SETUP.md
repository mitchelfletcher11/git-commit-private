# Setup — git-commit-private

The skill **self-bootstraps** these on first run (Step 0 — Preflight): it detects each prerequisite and prompts to create what's missing. This file is the manual reference if you'd rather set them up yourself.

## 1. `~/.claude` must be a git repo pointing at *your* private dotfiles repo

```bash
git -C ~/.claude init -b main
git -C ~/.claude remote add origin https://github.com/<your-account>/<your-private-dotfiles-repo>.git
```

Use **your own** private repo — the skill never assumes an account or repo name; it reads `git -C ~/.claude remote get-url origin` at push time.

## 2. Authentication

**HTTPS + token (default):** create a GitHub Personal Access Token with the `repo` scope and save it (the skill prompts for this and writes it for you):

```bash
printf '%s' "<your-token>" > ~/.claude/github-token
chmod 600 ~/.claude/github-token
```

Make sure `github-token` is gitignored in `~/.claude` so it is never committed:

```bash
echo 'github-token' >> ~/.claude/.gitignore
```

**SSH or a credential helper instead:** skip the token file entirely — if `~/.claude/github-token` is absent, the skill pushes with plain `git push origin HEAD` and relies on your configured git auth.

## 3. (Optional) Auto-run note

This skill is invoked explicitly (when Claude edits something under `~/.claude/`) — it installs **no** hooks or background scripts.
