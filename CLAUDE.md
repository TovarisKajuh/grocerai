# Grocerai

Interactive prototype of a grocery planning app: onboarding wizard, weekly meal plan, grocery list, deals optimiser across Slovenian supermarkets and a shopping route. It is a design and flow prototype, not a product; all data is hard-coded sample data.

## Layout

The whole app is one file, `index.html`: a `<style>` block, the markup and a `<script>` block. There is no build step, no dependencies to install, no tests and no CI. State is kept in `localStorage` under `grocerai-proto`; fonts come from Google Fonts.

The script is divided into commented sections in this order: `DATA`, `STATE`, `HELPERS`, `NUTRITION + PLAN`, `NAVIGATION`, `WIZARD`, `CRUNCH`, `PLAN`, `GROCERY LIST`, `DEALS / OPTIMIZER`, `ROUTE`, `PROGRESS`, `BOOT`. The CSS has matching `/* ---------- name ---------- */` blocks. Keep new code inside the section it belongs to.

## Running it

Open `index.html` in a browser, or `python -m http.server 8000` and open `http://localhost:8000`. Check changes in the browser; there is nothing else to run.

## Working in this repository

Two developers work on this repository, both through Claude Code. The full workflow is in [docs/collaboration/README.md](docs/collaboration/README.md). The rules that matter inside a session:

- Work on a branch named `<person>/<short-description>`, never directly on `main`. `main` is protected on GitHub and accepts no direct pushes; every change lands through a pull request.
- Commit after each coherent step with a short message. Do not save everything for one commit at the end.
- Do not reformat, reindent, reorder or tidy code you were not asked to change. Diffs must contain only the requested change.
- Stay inside the sections of `index.html` the task is about. If the change needs another section, say so before editing it.
- Keep a pull request to roughly 400 changed lines. Propose a split if the task is larger.
- Rebase onto `origin/main` before opening a pull request.
- Reply the way [docs/collaboration/communication.md](docs/collaboration/communication.md) describes: concise, most important fact first, no process narration, and every substantive reply states the context, what happened, what to pay attention to and the recommended next step.
