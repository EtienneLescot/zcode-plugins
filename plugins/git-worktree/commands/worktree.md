---
description: Manage git worktrees — list, create, open, remove, and prune isolated working copies for parallel ZCode sessions
argument-hint: [list] | create <name> [base] | open <name> | remove <name> | prune
---

Manage git worktrees for the current repository. The subcommand comes from `$ARGUMENTS`; when it is empty or unrecognized, default to `list` and end with a one-line summary of the other subcommands. Reply in the user's language.

## Ground rules (every subcommand)

1. Resolve the repository first with `git rev-parse --show-toplevel`. If the workspace is not inside a git repository, say so and stop.
2. The first entry of `git worktree list` is the main worktree. Never remove or prune it.
3. A branch can be checked out in only ONE worktree at a time. If git answers `fatal: '<branch>' is already used by worktree at <path>`, that is NOT a stuck git operation — the branch is owned by another worktree. Name the owning worktree and offer the user two options: open that worktree instead, or create this one on a different branch.
4. One ZCode conversation works in one working copy. Never guide two conversations into the same worktree folder — their checkouts and uncommitted changes would collide. When you create or open a worktree, remind the user of this rule in one sentence.
5. Run plain git commands only. Do not assume bash syntax; the user may be on Windows.

## list

Run `git worktree list --porcelain`. For each worktree also run `git -C <path> status --porcelain` to count uncommitted files. Present a table: name (directory basename), branch (`refs/heads/` stripped; `(detached)` kept), uncommitted file count, path. Mark the main worktree. Under the table, name any branch mentioned in more than one row as "checked out once per worktree only" if the user seems about to reuse it — otherwise omit.

## create <name> [base]

1. Normalize `<name>`: trim it, replace spaces with hyphens, refuse it if it is empty or contains path separators. Use it as the branch name as-is.
2. Warn and ask before proceeding when `<name>` equals the repository's default branch (`main`, `master`, or the `origin/HEAD` target): occupying the default branch in a worktree blocks every other session that wants it. Suggest a task branch based on it instead, but obey an explicit confirmation.
3. Default base: the `origin/HEAD` target if it exists, else `main` or `master` if either exists, else the current `HEAD`.
4. Default location: a sibling of the repository root — `<repo-root>/../<repo-dirname>-<name>`. If the user asks for an in-repo location such as `.worktrees/<name>`, honor it and append that directory to `.git/info/exclude` (never to a tracked `.gitignore`) so the worktree is not flagged as untracked.
5. Create with `git worktree add -b <name> <path> <base>` (drop `-b <name>` and append `<name>` instead when branch `<name>` already exists). Interpret a failure per ground rule 3.
6. After creation, print the path and the next steps: open it in ZCode via File → Open Folder (each worktree is a separate project with its own context), keep one conversation per worktree, and note that files not under version control — dependencies, build output — will not exist in the fresh worktree and must be installed or linked per the project's own setup guide.

## open <name>

Resolve `<name>` against `git worktree list --porcelain` by directory basename, branch name, or path prefix; refuse an ambiguous name by listing the candidates. A command cannot switch the ZCode workspace itself, so print the absolute path and the exact click path: File → Open Folder → select that directory. Add the one-sentence one-conversation-per-worktree reminder from ground rule 4.

## remove <name>

1. Resolve the worktree as in `open`. Refuse the main worktree.
2. Check `git -C <path> status --porcelain`. If there are uncommitted changes, STOP: summarize them (file list plus `git -C <path> diff --stat`), and ask whether to discard them. Only after an explicit confirmation run `git worktree remove --force <path>`; otherwise suggest committing or stashing first and stop.
3. Clean worktree: `git worktree remove <path>`. On Windows this can fail if another process (editor, terminal, antivirus) holds the directory — say so, and suggest closing it before retrying rather than jumping to `--force`.
4. After removal, offer branch cleanup: `git branch -d <branch>` when the branch is merged; never run `git branch -D` without an explicit confirmation that unmerged work will be lost.

## prune

1. Show what is stale first: `git worktree prune --dry-run --verbose` plus `git worktree list --porcelain`.
2. Run `git worktree prune`.
3. Orphan directories (a worktree directory on disk that no longer appears in `git worktree list`) are only ever reported with their paths and sizes — never deleted without an explicit confirmation, because the same look can hide uncommitted work the registry simply forgot.
