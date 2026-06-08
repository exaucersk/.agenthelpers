---
description: Analyze changes, bump version, update README, and commit using Conventional Commits
---

# Conventional Commit Workflow

## Step 1 — Analyze Changes

1. Run `git diff --staged` and `git status` to understand what changed.
2. Identify the change type: `feat`, `fix`, `refactor`, `chore`, `docs`, `perf`, `test`, `build`, or `ci`.

## Step 2 — Version Bump (if applicable)

If a versioning file exists (`package.json`, `pyproject.toml`, `Cargo.toml`, `version.txt`, etc.), apply the appropriate bump and stage the file:

- `fix` / `perf` → patch bump (1.2.3 → 1.2.4)
- `feat` → minor bump (1.2.3 → 1.3.0)
- Breaking change (`!` or `BREAKING CHANGE`) → major bump (1.2.3 → 2.0.0)

## Step 3 — Update README (if applicable)

If a `README.md` exists **and** the changes are user-facing (new features, changed behavior, new commands):

- Update the relevant section (Features, Usage, Changelog, etc.)
- Keep the existing style and structure
- Stage the file

## Step 4 — Compose Commit Message

Write a commit message following [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>[optional scope]: <short summary — imperative mood, ≤72 chars, no period>

[optional body — what and why, lines wrapped at 72 chars]

[optional footer — BREAKING CHANGE: …, Closes #issue]
```

Use `!` after type/scope for breaking changes, e.g. `feat!: remove legacy API`.

## Step 5 — Confirm & Commit

Show the full commit message and ask:

> Does this commit message look good?

**Options: Yes, commit / No, let me edit**

- **Yes** → run `git commit -m "<message>"`
- **No** → ask what to change, revise, and confirm again

## Step 6 — Push

After a successful commit, ask:

> Push the changes to the remote?

**Options: Yes / No**

- **Yes** → run `git push`
- **No** → confirm the commit is saved locally and finish
