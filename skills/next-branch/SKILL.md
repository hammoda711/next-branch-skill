---
name: next-branch
description: >
  Close out a finished feature branch and open the next one. Use when the user
  is on a feature branch cut from dev, the task is done, and the remote feature
  branch has been merged into remote dev. Verifies that is actually the case,
  syncs local dev, applies any new migrations, then asks for the next branch
  name and cuts it from the updated dev. Triggers on "next branch",
  "/next-branch", "the PR is merged what now", "start the next task", or
  finishing a branch whose PR has landed.
---

# Next branch

Three phases, in order. **Phase 2 does not start until phase 1 passes.** If phase 1
fails, do nothing else and report.

Never push, never merge into dev, never force anything. This skill only reads the
remote, fast-forwards local dev, applies migrations **when the state says they are
needed**, and creates a branch.

---

## Phase 1 — verify the case, or stop

Run these checks and report them as a table. Do not fix anything in this phase.

```bash
git fetch origin --prune
git status --short
git branch --show-current
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "(no upstream)"
```

Five conditions, all required:

| # | Condition | How to check |
|---|---|---|
| 1 | Working tree is clean | `git status --short` is empty |
| 2 | Not already on `dev` or `main` | `git branch --show-current` |
| 3 | The branch exists on the remote | `git ls-remote --heads origin <branch>` returns a line |
| 4 | Its commits are **in** remote dev | `git merge-base --is-ancestor <branch-tip> origin/dev` |
| 5 | Nothing local is unpushed | `git log --oneline origin/<branch>..HEAD` is empty |

**If any condition fails, stop.** Do not sync, do not create a branch. Say which
condition failed and what the state actually is. Common cases and what to say:

- **Dirty tree** — name the files. The user decides whether to commit, stash or discard.
- **Already on `dev`** — there is no feature branch to close. Ask which branch they meant,
  or offer to skip to phase 3.
- **Branch not on the remote** — it was never pushed, or someone deleted it after merge.
  If the local commits are already ancestors of `origin/dev`, say that: the work landed
  and only the branch ref is gone, which is fine. Otherwise the work is unmerged.
- **Not an ancestor of `origin/dev`** — the PR is **not** merged. Report how many commits
  are missing (`git log --oneline origin/dev..HEAD`) and check for an open PR before
  guessing.
- **Unpushed commits** — list them. The branch is not finished.

Condition 4 is the one that matters. A merged PR means every commit of the branch is
reachable from `origin/dev`. A squash or rebase merge breaks that test even though the
work landed, so when condition 4 fails, before calling it unmerged also compare the
content:

```bash
git diff --stat origin/dev...HEAD
```

An empty diff means the content is in dev under different commit ids. Say so, and treat
the case as merged.

---

## Phase 2 — sync local dev and the database

Only after phase 1 passes.

```bash
git checkout dev
git merge --ff-only origin/dev
```

Use `--ff-only`. If it refuses, local `dev` has commits of its own — stop and report;
never create a merge commit on dev here.

### Migrations: read the state first, then decide whether to act

**Never run `alembic upgrade head` as a reflex.** It is not a safe no-op to run blindly —
on two heads it fails outright, and on a database that reached the current head through a
different chain it silently does nothing while the schema is wrong. Read first, then
decide. Two commands, no side effects:

```bash
uv run alembic heads
uv run alembic current
```

Then take exactly one branch, and say in the report which one was taken and why:

| What you read | Do |
|---|---|
| **More than one head** | **Stop.** Do not upgrade. Report both revisions and their `down_revision`, and say which one should move — the later one, the one applied on no database. Do not edit a migration without asking. |
| **`current` == the single head** | **Run nothing.** Report "no migration to apply". |
| **`current` is behind the head** | **Upgrade** — the only case that does. |
| **`current` is a revision not in the chain** | **Stop.** The database is on a branch the code no longer has. Report both and ask; this needs a `stamp`, not an `upgrade`. |

Only in the third case:

```bash
uv run alembic upgrade head
```

Confirm afterwards in every case, including the ones where nothing ran:

```bash
uv run alembic current
uv run alembic check
```

`alembic check` must report no new operations. **If it reports drift, that is the trap
worth knowing:** `alembic_version` holds one revision id, so if a database reached the
current head through a *different* chain, `upgrade head` does nothing while the schema is
missing columns. Report the exact operations it names. The recovery is
`alembic downgrade <the revision before the divergence>` then `upgrade head` — ask first,
because it rewrites schema.

Never hand-write DDL to fix a drift, and never `DROP COLUMN` something alembic believes it
created. That breaks `downgrade` later on a constraint that no longer exists.

Finally, prove dev actually runs before handing it over — boot the app or run the test
suite, whichever the project has. Report the route count or the test result, not "it
should be fine".

---

## Phase 3 — ask, then cut the branch

Ask the user for the new branch name. Use AskUserQuestion when the project has a visible
naming convention, offering 2–3 names that follow it; otherwise just ask in one line.

Read the convention from the existing branches rather than inventing one:

```bash
git branch -a --sort=-committerdate | head -20
```

Then create it **without tracking dev**:

```bash
git checkout --no-track -b <name> origin/dev
```

`--no-track` is not optional. `git checkout -b <name> origin/dev` sets the upstream to
`origin/dev`, and from then on a bare `git push` pushes **straight to dev**, with no
branch and no PR. That has already happened once in this user's repo and took two PRs and
a revert to undo.

Confirm the result:

```bash
git status -sb
```

It must show `## <name>` with no upstream, or an upstream that is the branch's own remote
ref — never `...origin/dev`.

When the branch is first pushed, push it explicitly:

```bash
git push -u origin HEAD:refs/heads/<name>
```

---

## Report at the end

Short, and in this order:

1. The phase 1 table — what was verified
2. What local dev moved from and to
3. The migration state that was **read**, which of the four branches it put you in, and
   whether `upgrade head` therefore ran or not. Then what `alembic current` and
   `alembic check` say now. "Nothing to apply" is a result, not a skipped step — say it
   that way so it does not read like something was forgotten.
4. The new branch, and that it does not track dev

If anything was skipped or blocked, say so in that list rather than leaving it out.
