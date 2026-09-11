# Claude Code conventions

## Review is the safeguard

When both of us work through Claude Code, neither of us has typed most of the code in the repository. The quiet checks that come from having written something yourself are gone. The pull request review by the other person is the only place a human reads the change with intent, so it has to be a real review and not a formality.

That only works if the review is small enough to do properly. Around 400 changed lines is what one person can read carefully in one sitting. At 2,000 lines nobody reads it; they scroll, see nothing alarming and approve. A PR that size is a rubber stamp, and a rubber stamp on agent-written code is no safeguard at all. Split large work into several PRs that each make sense on their own.

## What gets committed

Both agents must work from the same instructions, so the files that carry those instructions live in the repository:

- `CLAUDE.md` at the repository root.
- `.claude/settings.json` for shared permissions and hooks. It currently holds one hook, described below.
- `.claude/commands/` and `.claude/skills/` for any shared commands or skills.

These files are personal and stay out of the repository:

- `.claude/settings.local.json`, which holds per-machine permission grants.

Local-scope MCP servers are stored in your home directory by Claude Code, not in the repository, so there is nothing to ignore for them. If we ever add a project-scoped `.mcp.json`, it is shared configuration and gets committed like `settings.json`.

The `.gitignore` at the root already covers `settings.local.json`.

## The session-start fetch hook

`.claude/settings.json` runs `git fetch --prune origin` when a Claude session starts in this repository, and prints a line when the checked-out branch is behind `origin/main`.

It exists because `git status` never contacts GitHub: it compares your branch against the local `refs/remotes/origin/main`, which only changes when something fetches. An un-fetched clone therefore looks in sync whatever has landed on `main`, and a branch started from it is based on stale history.

The hook only fetches. It never checks out, merges, rebases or pushes, so it cannot touch your working tree, and it exits successfully when the remote is unreachable so that an offline session still starts. Since a fetch updates the whole repository, one run also covers any worktrees attached to it.

## Divide work by module, not by ticket

Two Claude sessions editing the same file at the same time produce conflicts that neither person understands, because neither person wrote either side. Avoid it by dividing work along the structure of the code rather than along a list of tasks.

Right now the whole prototype is one file, `index.html`, organised into named sections (`WIZARD`, `PLAN`, `GROCERY LIST`, `ROUTE` and so on, with matching CSS blocks). Agree on who owns which sections before starting, and keep each branch inside its sections. Once the prototype is split into files, the same rule applies per file or directory.

If a task genuinely needs changes in the other person's area, say so before starting, and make it a separate small PR that lands first.

## Commit incrementally

Tell Claude to commit as it goes, after each coherent step, rather than producing one commit at the end of a long session. Small commits make the branch's history useful during review, make it possible to drop or reorder a step during the rebase, and mean a session that goes wrong loses minutes of work rather than hours. Since every PR is squash merged, the number of commits on the branch costs nothing.

A useful instruction at the start of a session is: "Commit after each step with a short message describing that step."

## Never reformat untouched code

Claude must not reformat, reindent, reorder or otherwise tidy code it was not asked to change. A reformat of a section the other person is working on guarantees a conflict on their branch, and a diff full of whitespace changes hides the real change from the reviewer.

This is written into `CLAUDE.md` so both agents see it. If a reformat is actually wanted, it is its own PR, with nothing else in it, coordinated so that no other branch is open on that code.
