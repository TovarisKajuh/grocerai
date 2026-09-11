# Setup

One-time steps for each developer, then one-time steps for the repository.

## Access

The repository is `TovarisKajuh/grocerai` on GitHub and is public. Reading it needs nothing, but pushing branches needs push access. The owner adds the other developer under **Settings > Collaborators > Add people** on GitHub. Until that invitation is accepted, `git push` will be rejected.

## Clone

```sh
git clone https://github.com/TovarisKajuh/grocerai.git GrocerAI
cd GrocerAI
```

Check the remote points where you expect:

```sh
git remote -v
```

Both lines should show `https://github.com/TovarisKajuh/grocerai.git`.

## Git identity and pull behaviour

```sh
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global pull.rebase true
```

Use the email address attached to your GitHub account so commits are linked to you. `pull.rebase true` is explained in [git-workflow.md](git-workflow.md).

Confirm:

```sh
git config --global --list | grep -E 'user\.|pull\.rebase'
```

## GitHub CLI

The workflow uses `gh` for pull requests. Install it, then:

```sh
gh auth login
gh auth status
```

## Run the prototype

There is no build step and nothing to install. Open `index.html` in a browser, or serve the directory so that the page has a proper origin:

```sh
python -m http.server 8000
```

Then open `http://localhost:8000`. The prototype loads fonts from Google Fonts, so it needs network access, and it keeps its state in `localStorage` under the key `grocerai-proto`.

## Branch protection on GitHub

These settings are configured once, by the repository owner. GitHub calls this a ruleset.

Go to **Settings > Rules > Rulesets > New ruleset > New branch ruleset** and set:

- **Ruleset Name**: `Protect main`
- **Enforcement status**: **Active**
- **Bypass list**: leave empty
- **Target branches**: **Add target > Include default branch**

Under **Branch rules**, tick:

- **Restrict deletions**
- **Require linear history**
- **Require a pull request before merging**, and inside it:
  - **Required approvals**: `1`
  - **Dismiss stale pull request approvals when new commits are pushed**
- **Block force pushes**

Click **Create**.

The same ruleset can be created from the command line by the owner:

```sh
gh api -X POST repos/TovarisKajuh/grocerai/rulesets --input - <<'JSON'
{
  "name": "Protect main",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "required_linear_history" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false
      }
    }
  ]
}
JSON
```

## Merge settings on GitHub

So that every PR is squash merged and its branch removed afterwards, the owner sets, under **Settings > General > Pull Requests**:

- Untick **Allow merge commits**
- Keep **Allow squash merging** ticked
- Untick **Allow rebase merging**
- Tick **Automatically delete head branches**

Or from the command line:

```sh
gh repo edit TovarisKajuh/grocerai --enable-merge-commit=false --enable-rebase-merge=false --enable-squash-merge --delete-branch-on-merge
```

## Escape hatch while prototyping

While the project is a prototype and nothing is deployed, the PR workflow is more ceremony than it is worth. In that phase it is fine to leave the ruleset off and push straight to `main`, pulling often:

```sh
git pull
# work, commit
git pull
git push
```

With `pull.rebase true` set, each `git pull` replays your commits on top of the other person's, and conflicts show up immediately while they are small.

Switch to the ruleset and PRs the first time a broken `main` costs the other person time. That is the signal, not a date.
