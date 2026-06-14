---
name: git-commit-private
description: >
  Commit and push all changes in ~/.claude/ to your private dotfiles GitHub repository (whatever
  remote the ~/.claude git repo points at). Use this skill whenever Claude makes explicit edits to
  skill files, CLAUDE.md files, or any other ~/.claude/ content during a session — not just when
  the user types /git-commit-private.
  Shows a diff summary, proposes a commit message, waits for user confirmation or edit, then
  commits and pushes. Includes git log output for revert navigation. Exception: automated
  sweeps (best-practice-skills, best-practice-claude) run git directly without this skill —
  those are silent automated commits that do not require user confirmation.
allowed-tools: Bash
---

# git-commit-private

Commit and push changes in `~/.claude/` to **your** private dotfiles repository — whatever
remote the `~/.claude` git repo points at. This skill never hardcodes a repo name or account;
it derives the push target from your local `origin` remote at runtime.

## When to use this skill

- **Always use this skill** when Claude makes explicit edits to skill files, CLAUDE.md files, or any other `~/.claude/` content during a conversation — run it after the edits are done, before ending the turn.
- **Do not use this skill** for the automated silent commits inside `best-practice-skills` and `best-practice-claude` sweeps — those run git directly by design, since no user confirmation is needed for automated cleanup.

In short: if Claude changed a skill file, this skill runs. If an automated sweep changed files, git runs directly.

---

## Step 0 — Preflight: build the private git environment (first run)

This skill pushes `~/.claude` to a **private** GitHub repo. It works even if the user has
**never used GitHub before** — detect what's missing and build it, prompting at each step.
Never assume an account, repo, or tool already exists.

```bash
command -v git >/dev/null 2>&1 && echo "GIT_OK" || echo "NO_GIT"
command -v gh  >/dev/null 2>&1 && gh auth status >/dev/null 2>&1 && echo "GH_READY" || echo "GH_ABSENT"
git -C ~/.claude rev-parse --is-inside-work-tree >/dev/null 2>&1 && echo "REPO_OK" || echo "NO_REPO"
git -C ~/.claude remote get-url origin 2>/dev/null || echo "NO_REMOTE"
[ -f ~/.claude/github-token ] && echo "TOKEN_OK" || echo "NO_TOKEN"
```

Resolve in this order — each step assumes the ones above it are done:

1. **`NO_GIT`** → git isn't installed; nothing works without it. Give the one install command for their platform (`brew install git`, `sudo apt install git`, …) and stop until it's present.
2. **A GitHub account** → if the user says they've never used GitHub, point them to <https://github.com/signup> to create a free account before continuing. A private repo and any token both require it.
3. **`NO_REPO`** → offer: *"`~/.claude` isn't a git repo yet — I'll initialise it."* On yes: `git -C ~/.claude init -b main`.
4. **`NO_REMOTE` — does the private repo exist yet?** Ask. Then:
   - **It exists** → ask for its URL and `git -C ~/.claude remote add origin <url>`.
   - **It does NOT exist** (the common first-time case) → **create it.** The gentlest path is the `gh` CLI:
     - **`GH_READY`** → `gh repo create <name> --private --source ~/.claude --remote origin` (creates the private repo *and* wires the remote in one step; pick any `<name>`, e.g. `dotfiles` or `claude-config`).
     - **`GH_ABSENT`** → either install + authenticate `gh` (`gh auth login` walks a first-timer through it in the browser), **or** guide them to create it on the web: <https://github.com/new> → name it, set **Private**, do **not** add a README, Create — then `git -C ~/.claude remote add origin <url>`.
5. **`NO_TOKEN`** → the push needs credentials. Offer the **easy path first**: if `gh` is authenticated (`GH_READY`), git can use it directly — run `gh auth setup-git` and skip the token file entirely. Otherwise authenticate with a **Personal Access Token**:
   - Create one at <https://github.com/settings/tokens> → *Generate new token* → scope **`repo`** → copy it (GitHub shows it once).
   - **Prompt the user to paste it**, then write it with safe perms — never inline a value into the skill or commit it:
     ```bash
     printf '%s' "<token the user pasted>" > ~/.claude/github-token && chmod 600 ~/.claude/github-token
     ```
   - Ensure it can never be committed: `grep -qxF 'github-token' ~/.claude/.gitignore 2>/dev/null || echo 'github-token' >> ~/.claude/.gitignore`.
   - If the remote uses SSH instead, skip the token entirely and push with plain `git push`.

If every check returns OK, continue silently.

---

## Step 1 — Check for changes

Run:
```bash
git -C ~/.claude status --porcelain
```

If empty: print "Nothing to commit — ~/.claude/ is up to date." and stop.

---

## Step 2 — Show the diff summary

Run both in parallel:
```bash
git -C ~/.claude diff --stat HEAD
```
```bash
git -C ~/.claude status --short
```

Print the combined output clearly, so the user can see exactly what has changed.

**Untracked-file guard:** Count the untracked files (lines prefixed `??`) in the `git status --short` output. If the count exceeds 15, print a warning before proceeding:

```
⚠️ Warning: [N] untracked files will be staged by git add -A.
First 10 unexpected files:
  [list first 10 ?? entries]
These may include cache, session, or project metadata files not intended for the dotfiles repo.
Proceed with staging all, or list which paths to exclude?
```

Wait for the user's response before moving to Step 3. If the user says to proceed, continue as normal. If they name paths to exclude, add those paths to a `.gitignore` in `~/.claude/` before staging.

---

## Step 3 — Propose a commit message

Look at the changed files from Step 3 and draft a concise commit message that describes what changed.

Rules:
- Lead with the area of change (e.g. `skills:`, `config:`, `memory:`)
- Name the specific skills or files if there are 3 or fewer; otherwise say how many files changed
- One line, under 72 characters
- Do not include "Co-Authored-By" in the message — that is for source code commits, not dotfile snapshots

Example good messages:
- `skills: update project-scout Build Spectrum section`
- `config: add git-commit-private, update best-practice-skills`
- `memory: 4 project memory files updated`

Print the proposed message and ask: **"Commit with this message, or type a replacement?"**

Wait for the user's response before proceeding.

---

## Step 4 — Commit and push

Use the message the user confirmed or provided. Derive the push target from the local `origin`
remote (Step 0 guaranteed it exists) — never hardcode an account or repo name:

```bash
git -C ~/.claude add -A
git -C ~/.claude commit -m "[confirmed message]"
REMOTE=$(git -C ~/.claude remote get-url origin)
if [ -f ~/.claude/github-token ]; then
  # HTTPS + token auth: inject the token into whatever host/path origin points at
  git -C ~/.claude push "https://$(cat ~/.claude/github-token)@${REMOTE#https://}" HEAD
else
  # SSH or a configured credential helper: push directly
  git -C ~/.claude push origin HEAD
fi
```

Print the commit hash and push result.

---

## Step 5 — Show git log for revert navigation

Run:
```bash
git -C ~/.claude log --oneline -10
```

Print the output. Then print this revert reference:

---

**To revert a specific file to an earlier version:**
```bash
# Find the commit hash from the log above, then:
git -C ~/.claude log --oneline -- skills/[skill-name]/SKILL.md   # history for one file
git -C ~/.claude show [hash]:skills/[skill-name]/SKILL.md        # view a version
git -C ~/.claude checkout [hash] -- skills/[skill-name]/SKILL.md # restore that version
```
After restoring, run `/git-commit-private` again to commit the revert.

---

## Final Step — Write observation

Append a timestamped entry to `~/.claude/skills/git-commit-private/observations.md`:

```markdown
## [ISO 8601 timestamp]
[One sentence: what changed was committed, whether the user edited the proposed message, any push errors]
```

If `observations.md` does not exist, create it with this frontmatter first:

```markdown
---
last-reviewed: 1970-01-01T00:00:00Z
---
```
