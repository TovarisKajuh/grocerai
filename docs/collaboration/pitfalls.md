# Pitfalls

Things that have caused, or will cause, conflicts between two clones, and the rule for each.

## Lock files

Never resolve a conflict in a lock file by hand. A lock file is generated output with internal consistency that a manual merge will break. Take one side whole and regenerate it.

The repository has no package manager and no lock file today, so this rule is dormant. When one is added, the procedure during a rebase is: take the `main` version of the lock file, re-run the install command for whatever package manager we chose, and commit the regenerated file.

## Build output and generated files

Anything a tool produces from the sources is not committed. It changes on every build, it differs between machines and it conflicts constantly. Add the output directory to `.gitignore` before the first build ever runs, not after it has been committed once.

There is no build step today; `index.html` is served as-is. If a build step is introduced, its output directory goes into `.gitignore` in the same PR.

## Force-pushing

`git push --force-with-lease` on your own PR branch is normal. Rebasing before review rewrites the commits on the branch, and the force push is how the rewritten branch reaches GitHub. Prefer `--force-with-lease` over `--force`; it refuses if someone else has pushed to the branch since you last fetched.

Never force-push `main`. It rewrites history that the other person has already built on, and their next pull will not be able to reconcile it. The branch protection in [setup.md](setup.md) blocks this on GitHub, but the rule stands even during the prototyping phase when protection is off.

## Environment files

`.env` is gitignored and never committed. It holds secrets and machine-specific values, and a single commit of it means rotating every key in it.

The committed file is `.env.example`, which lists every variable the project reads, with placeholder values and a comment on where each one comes from. There is no tooling that keeps the two in sync: whenever you add a variable to your `.env`, add it to `.env.example` in the same PR.

The prototype reads nothing from the environment today, so neither file exists yet. `.gitignore` already covers `.env` so that the first one added is not committed by accident.
