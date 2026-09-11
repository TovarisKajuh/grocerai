# Collaborating on Grocerai

Two developers, one GitHub repository, both working through Claude Code. These pages describe how that works day to day.

## TL;DR

1. Each of us has our own clone. There is no shared working directory; GitHub is the only meeting point.
2. `main` is protected. Nobody pushes to it directly and nobody force-pushes it. Everything lands through a pull request.
3. One branch per discrete piece of work, named `<person>/<short-description>`, for example `aljaz/route-screen-fixes`.
4. Rebase onto `origin/main` before opening the PR so any conflict is resolved in your branch, not in `main`.
5. The other person reviews. Review is the only real safeguard when neither of us typed the code, so PRs stay small: roughly 400 changed lines.
6. Squash merge, then delete the branch. Branches live for two days at most; split anything longer.
7. `git config --global pull.rebase true` so `git pull` never creates merge commits.
8. Divide work by module (a section of `index.html`, later a file), not by ticket, so two Claude sessions are not editing the same code at once.
9. Tell Claude to commit incrementally and never to reformat code it was not asked to touch.
10. Commit `CLAUDE.md` and `.claude/settings.json`; gitignore `.claude/settings.local.json`.

## Pages

- [Setup](setup.md): one-time clone, git config and branch protection.
- [Git workflow](git-workflow.md): branching, pull requests, rebasing and merge rules.
- [Claude Code](claude-code.md): conventions for code that an agent wrote.
- [Communication](communication.md): how Claude reports to us in replies, commits and pull requests.
- [Pitfalls](pitfalls.md): known conflict sources and how to avoid them.
