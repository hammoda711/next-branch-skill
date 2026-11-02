# next-branch

A Claude Code skill that closes out a finished feature branch and cuts the next one from a freshly synced `dev`.

The point is the verification. It checks that the remote really merged your branch before it touches anything, and it reads the migration state before deciding whether a migration needs applying at all. If the checks fail it does nothing and tells you why.

## Install

```
/plugin marketplace add hammoda711/next-branch-skill
/plugin install next-branch@next-branch-skill
```

## Use

Run `/next-branch` when your PR has landed. The skill also triggers on "next branch", "the PR is merged what now", and "start the next task".

## What it does

Three phases, in order. Phase 2 does not start until phase 1 passes.

**Phase 1 — verify the case, or stop.** Five conditions, all required: the working tree is clean, you are not already on `dev` or `main`, the branch exists on the remote, its commits are ancestors of `origin/dev`, and nothing local is unpushed. Reported as a table. Nothing is fixed in this phase. Condition 4 is the one that matters, and a squash or rebase merge breaks the ancestor test even though the work landed, so when it fails the skill compares `git diff --stat origin/dev...HEAD` before calling the branch unmerged.

**Phase 2 — sync local dev and the database.** Fast-forwards local `dev` to the remote. Then reads `alembic heads` and `alembic current` before deciding:

| What it reads | What it does |
|---|---|
| More than one head | Stops. Reports both revisions and asks. |
| `current` equals the single head | Runs nothing. Reports "no migration to apply". |
| `current` is behind the head | Upgrades. The only case that does. |
| `current` is not in the chain | Stops. That needs a `stamp`, not an `upgrade`. |

**Phase 3 — ask, then cut the branch.** Asks for the next branch name and cuts it from the updated `dev`.

It never pushes, never merges into `dev`, and never forces anything.

## Assumes

- **A `dev` integration branch** that feature branches are cut from and merged back into. A repo that merges into `main` instead will fail phase 1 on condition 2.
- **Alembic**, run as `uv run alembic`. If your project uses a different migration tool, or runs alembic without `uv`, phase 2's migration step does not apply as written.
- **A remote named `origin`.** Every check reads `origin`.
- **The merge happened on the remote.** The check reads `origin/dev`, so a branch merged only in your local `dev` does not pass.

## License

MIT. See [LICENSE](LICENSE).
