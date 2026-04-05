---
name: prepare-pr
description: >
  Prepares and submits a pull request after finishing dev work. Use this skill
  whenever the user says something like "prepare a PR", "make a PR", "commit and push",
  "open a pull request", "/prepare-pr", or anything indicating they want to package
  up finished work into a PR. Also trigger proactively after completing a coding task
  if the user's message ends with "and make a PR" or similar.

  Behavior:
    - On main branch: creates a new feature/ branch, commits, pushes, opens a PR.
    - On a feature branch: commits new work, pushes, then creates or updates the PR.
---

# prepare-pr

Prepare, commit, and open (or update) a pull request for finished dev work.

## Step 1 — Understand the current state

Run these commands in parallel to gather context:

```bash
git status
git diff HEAD
git log --oneline -10
git branch --show-current
```

Also run `gh pr view --json title,body,url 2>/dev/null` to check whether a PR already
exists for this branch.

## Step 2 — Determine the workflow

**If on `main`:** you must create a feature branch before committing.

- Derive a branch name from the staged/unstaged changes. Use `feature/<short-kebab-name>`
  (e.g. `feature/add-equity-research-agent`, `feature/fix-binary-tool-gating`).
- Run `git checkout -b feature/<name>` to create and switch.

**If already on a `feature/` branch:** Check whether it's already merged into main:

```bash
git branch --merged main | grep -w "$(git branch --show-current)"
```

If it shows up as merged, stop and ask the user what to do — they may want to create
a new branch, work directly on main, or they may have a different intent entirely.
Don't proceed automatically; this is ambiguous enough to warrant a pause.

If not yet merged, stay on the branch. No branch switch needed.

**If on any other non-main branch:** use it as-is (treat it like a feature branch).

## Step 3 — Stage and commit

Start by staging everything:

```bash
git add -A
git status
```

The developer may have made their own edits alongside yours — include those too. The
default is to commit all changes. Only exclude something if there's a clear, specific
reason not to.

**Audit what's staged** and handle these cases:

1. **Sensitive files** (`.env`, `*.key`, `*.pem`, `*secret*`, credentials, tokens):
   Unstage the file and add it to `.gitignore` so it stays excluded permanently.
   ```bash
   git restore --staged <file>
   echo "<file>" >> .gitignore
   git add .gitignore
   ```
   Tell the user what you excluded and why.

2. **Obviously unrelated files** (e.g. `scratch.py`, `test_output.csv`, editor swap
   files, personal notes): These are files that have no plausible connection to the
   work being committed. Flag them to the user and ask whether to include or exclude
   — don't make that call silently. If the user isn't available to respond, include
   them (bias toward inclusion).

3. **Everything else** — including changes the developer made to the code — goes into
   the commit. Don't second-guess human edits unless they look like a fundamentally
   breaking change (e.g. deleted the entire project, wiped a config). If something
   looks surprising but plausible, include it.

Write a single-line commit message that describes *what* changed and *why*, not *how*.
Keep it under 72 characters. Use imperative mood ("Add X", "Fix Y", "Migrate Z to W").
If the staged changes reflect a mix of agent and human edits, write the message to
cover the full changeset.

```bash
git commit -m "<short imperative summary of the change>"
```

If pre-commit hooks fail: read the error, fix the underlying issue, re-stage, and
create a new commit. Never use `--no-verify`.

## Step 4 — Push

```bash
git push -u origin HEAD
```

## Step 5 — Create or update the PR

**No PR yet (new branch, or first push from a feature branch):**

Create one with `gh pr create`. Base should be `main`. Write a title (≤70 chars) that
matches the commit message style, and a body with this structure:

```
## Summary
- <bullet 1>
- <bullet 2>

## Test plan
- [ ] <manual check or test command>
```

Use a HEREDOC to avoid quoting issues:

```bash
gh pr create --title "..." --body "$(cat <<'EOF'
## Summary
...
EOF
)"
```

**PR already exists:** the push automatically updates it. If the diff since the last
PR description is significant (new features or behavior changes), run `gh pr edit`
to update the body. Then show the user the PR URL with `gh pr view --json url -q .url`.

## Step 6 — Report back

Print the PR URL and a one-line summary of what was committed. Keep it brief.

## Guard rails

- Never force-push to `main` or any protected branch.
- Never use `--no-verify` or `--no-gpg-sign`.
- Sensitive files must never be committed — unstage and gitignore them, always.
- If nothing is staged and no unstaged changes exist, tell the user there is nothing
  to commit.
