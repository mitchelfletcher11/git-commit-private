# git-commit-private — Claude Code skill

Commit and push changes in `~/.claude/` to **your** private dotfiles GitHub repo — diff summary, a proposed commit message you confirm, then commit + push, with git-log revert navigation. Account- and repo-agnostic: it derives the push target from your local `origin` remote at runtime, never a hardcoded account.

## Install
```bash
mkdir -p ~/.claude/skills/git-commit-private && curl -sf \
  https://raw.githubusercontent.com/mitchelfletcher11/git-commit-private/main/SKILL.md \
  -o ~/.claude/skills/git-commit-private/SKILL.md
```

On first run the skill self-bootstraps (**Step 0 — Preflight**): it detects whether `~/.claude` is a git repo, has an `origin` remote, and has a `~/.claude/github-token` file, and prompts to set up whatever is missing. See **SETUP.md** for the manual reference.
