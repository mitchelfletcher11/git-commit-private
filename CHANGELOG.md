# Changelog

## v1.1.0 — 2026-06-14
Zero-state cold-start preflight: detects git, GitHub account, repo existence and token; **creates** the private repo (gh or web) instead of assuming a URL. Account-agnostic push derived from the origin remote.

## v1.0.0 — 2026-06-14
Initial public release.

- Commit + push changes in `~/.claude/` to your private dotfiles repo.
- Diff summary, proposed commit message (you confirm or replace), commit, push.
- Untracked-file guard (warns before staging >15 untracked files).
- git-log output with a file-level revert reference.
- **Account-agnostic:** push target derived from the local `origin` remote at runtime — no hardcoded account or repo name.
- **Step 0 self-bootstrapping preflight:** detects the `~/.claude` git repo, `origin` remote, and `~/.claude/github-token`, and prompts to set up whatever is missing. Supports HTTPS+token, SSH, or a credential helper.
