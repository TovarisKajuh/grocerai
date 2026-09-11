# Communication guidelines

How Claude reports to us during a session, in commit messages and in pull requests. The code is agent-written; these replies are the main way we stay aware of what is happening in our own repository, so they have to be short and complete at the same time.

## Be concise

- Lead with the result. No preamble, no restating the request, no "Great question", no apologies, no closing pleasantries.
- Short sentences, plain words. Bullets for lists, prose for one thought. Headings only when a reply is long enough to need them.
- Numbers over adjectives: "3 files, 120 lines" rather than "a few changes"; "2 of 5 checks failed" rather than "some issues".
- No emoji, no decorative formatting, no tables for things that fit in a sentence.
- Say a thing once. If it was already said this session, refer back to it rather than repeat it.

## Say what is important

Concise is not the same as short. A reply that omits the one thing we needed to know has failed, however few words it used.

- Rank before writing. Decide what the single most important fact is and put it first. A broken build, a changed behaviour, a decision we have to make: those go in the first line, before anything that went fine.
- Separate must-know from nice-to-know. "Pay attention to" holds things that change what we do: a risk, a side effect, a deviation from the task. It does not hold trivia to look thorough. Padding it trains us to skim it.
- Drop what does not matter. If a fact changes nothing for us, leave it out. If it is only worth a mention, it gets one clause, not a paragraph.
- Be specific. "The route order changed for `PROGRESS` too" is important; "there might be some side effects" is noise.
- When unsure whether something is important, include it in one line and say why it might matter.

## Every substantive reply carries four things

A reply that ends a step, finishes a task or reports a problem covers these, in this order:

1. **Context.** Where we are: branch, task, and what state the working tree is in. One line is usually enough. This is what lets either of us pick the session up cold.
2. **What happened.** What changed, in which files and sections, and whether it was verified and how. Reference code as `path:line` so it is clickable.
3. **Pay attention to.** Anything the reviewer must know: assumptions made, things touched outside the agreed scope, failures, behaviour that changed as a side effect, work left undone. If there is nothing, say "Nothing to flag."
4. **Next steps.** A concrete recommendation of what should happen now, whether that is a command to run, a decision to make, or a review to request. Recommend one thing; list alternatives only if the choice is genuinely ours.

Short acknowledgements ("Done, committed as `abc123`.") do not need all four. Anything longer does.

## Do not narrate the process

- Report outcomes, not activity. Never "Let me look at…", "Now I will…", "First I need to…".
- Do not explain reasoning unless the result is surprising or a judgement call was made. Then one sentence on why.
- Do not list the steps taken to get somewhere. The diff and the commit history are the record of the work; the reply is the summary.
- Do not repeat tool output. Quote only the line that matters, for example the failing assertion or the error message.

## Report faithfully

- A failure is reported as a failure, with the relevant output, in the first line, not buried after the successes.
- Skipped or partial work is stated explicitly: what was left out and why.
- Never claim verification that did not happen. For this project that mostly means: say whether the change was actually checked in the browser, and if not, say so.
- Assumptions are named as assumptions. "I assumed the deals list is sorted by price" is useful; silently building on it is not.
- If an earlier statement in the session was wrong and it matters, correct it in one sentence and move on.

## Ask only when it matters

- Make routine judgement calls and state them. Do not stop to ask about things that have an obvious default.
- Ask when different answers lead to materially different work, or when the action is hard to undo. Ask one question, offer the options, and put the recommended one first.
- Before touching a section of `index.html` or an area the other developer owns, say so and wait. This is the one case where asking is always right.

## Session start and end

- At the start of a session, confirm the branch, whether it is up to date with `origin/main`, and whether the tree is clean. One line.
- At the end of a session, or when handing off, give a summary the other developer can read without having seen the session: what landed, what is unfinished, what to review, what to do next.

## Commit messages and pull requests

The same rules apply in writing that outlives the session.

- Commit messages: one short line in the imperative describing the step. A body only when the why is not obvious from the diff.
- Pull request descriptions follow this shape:
  - **What**: the change, in two or three sentences.
  - **Why**: the reason, if it is not obvious.
  - **How to check**: what to open in the browser and what to look for.
  - **Pay attention to**: the same list as in a reply, so the reviewer knows where to look hardest.
  - **Out of scope**: anything noticed but deliberately not done.

## A reply that follows these rules

> On `aljaz/route-screen-fixes`, 2 commits ahead of `origin/main`, tree clean.
>
> Fixed the route ordering in the `ROUTE` section (`index.html:2140-2175`): stops are now sorted by distance from the chosen start store. Checked in the browser with the default sample data.
>
> Pay attention to: `sortStops()` is also called from `PROGRESS` when a stop is ticked off; I did not change that call but it now returns a different order, so the progress bar moves in a different sequence. Nothing else touched outside `ROUTE`.
>
> Next: rebase onto `origin/main` and open the PR, or first decide whether `PROGRESS` should keep its old order.
