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

## The escape hatch is closed

Earlier this page said that while the project was a prototype it was fine to leave the ruleset off and push straight to `main`. That is no longer true. The `Protect main` ruleset above is active, with an empty bypass list, so `main` accepts no direct pushes from anyone, the owner included. Every change reaches `main` through a pull request that one of us approves on GitHub.

Day to day this means `main` is read-only on your machine. Pull it, branch off it, push the branch, open the pull request:

```sh
git switch main
git pull
git switch -c <person>/<short-description>
# work, commit
git push -u origin <person>/<short-description>
gh pr create --fill
```

The ruleset is repository configuration on GitHub, not a file in this repository. It is not on `main`, not on any branch and not in your clone. Cloning or pulling does not bring it with you, and it applies no matter which branch you have checked out locally.

If the ceremony ever needs to go away again, the owner disables the ruleset under **Settings > Rules > Rulesets**, or from the command line:

```sh
gh api -X PUT repos/TovarisKajuh/grocerai/rulesets/22935862 -f enforcement=disabled
```
