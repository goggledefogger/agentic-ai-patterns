---
type: pattern
date: "2026-10-17"
source: A 9B local agent model asked to port an end-to-end test harness across 13 major framework versions, graded against a stronger model's reference branch, measured 2026-10-16
tags:
  - local-models
  - migration
  - verification
  - grading
  - delegation
---

# A Small Model Moves the File and Drops the Behavior

A 9B agent model was asked to move a Cypress 3 test harness to Cypress 16. The task: write a new `cypress.config.js`, rename the specs into `cypress/e2e` as `.cy.js`, delete `plugins/index.js` "with anything it did moved into setupNodeEvents", fix 2 stale page URLs, and bump the cypress version.

The model read all 7 files first, made 22 tool calls with 0 errors in 3 minutes 39 seconds, and reported an accurate list of files created, changed and deleted. Every file move landed. The URL fixes matched the reference branch a stronger model had produced.

The behavior did not move. It deleted `plugins/index.js` and wrote a config with no `setupNodeEvents` at all, so the login and database-seed commands lost their node-side half. It then "replaced" the deleted node task with a browser-side Cypress command that calls Node's `fs` module, which cannot work in the browser, and left the specs calling the old task anyway. It bumped the one dependency the prompt named and left the 3 companion dependencies (`cypress-firebase`, `firebase`, `firebase-admin`) at 2019 versions. The prompt had said "if a step is beyond you, say which and why". The model said nothing.

## The Pattern

A small model does file surgery reliably: read, copy, rename, edit, delete. It ports behavior badly. "Delete X and move what it did into Y" executes as "delete X". It also does not know a framework's runtime split, here browser versus node, so it will put code where it cannot run and not notice.

- Name the behavior to carry, not only the files. "Move the login task and the seed task into setupNodeEvents" is a checklist the model can follow. "Move what it did" is not
- Name every dependency that moves together. A prompt that names one package gets one package
- Review at the runtime boundary. Check where each piece of code runs, not whether the file exists
- Diff the report against the tree. After the turn, compare the files the report names with `git status` and list the claims the tree does not back. Never grade from the report
- Do not expect "say if it is beyond you" to fire. A model that has confidently written the wrong port does not know it is beyond it
- Keep a stronger model's reference branch and diff against it. The diff is the grader

Naming helps but does not close the gap. Two more runs with the behaviors and all 4 dependencies spelled out, then a rule to carry every function before deleting the old file, moved the failure instead of removing it: run 2 landed the dependencies and dropped the behavior, run 3 landed the behavior and dropped the dependencies. The model lands about one hard step per run. Every report listed every step as done, and two reports listed a deleted file (`cypress/config.json`) that never existed. The file list is written from the plan, not from the tree.

## Why Reading the Output Doesn't Catch This

The transcript was clean: 0 tool errors, an accurate self-report, every listed file present. All of it was true. None of it says whether the port runs. The self-report described what was done to files, and the failure was in what the files no longer did. Only a comparison against a known-good result, or a run of the suite, shows the missing half. The false report makes this stricter: the account can be wrong about files too, so check it against `git status` before reading anything else into it. That check needs no model.

## When to Use

Any migration, framework upgrade or "move this logic over there" task handed to a small local model. Spell out the behavior and the dependency set in the prompt, and grade the result against a reference or a run, never against the model's own account of it.

## Related

- [[a-small-model-chooses-it-does-not-rewrite]], the same family: reshape the ask to fit what a small model reliably does, then grade the output against a source
- [[a-bare-prohibition-reads-as-no-tools]], from the same session: how wording alone can turn a small model's tool use off

## Source

Measured 2026-10-16 with a 9B agent model in a coding-agent harness. 22 tool calls, 0 errors, 3 minutes 39 seconds, accurate file report, file moves and URL fixes correct, node-side behavior dropped and 3 companion dependencies left stale. Graded by diffing against a stronger model's reference branch.

Follow-up 2026-10-17, same model, task and base, 2 more runs (23 and 32 tool calls). Run 2 (behaviors and dependencies named): dependencies landed, behavior dropped, report listed a nonexistent deleted file. Run 3 (plus a carry-everything lane line): behavior landed, dependencies dropped, same false file. Prompt tuning stopped; the move stays a cloud-model task on this hardware. The deterministic check shipped 2026-10-18 in the dashboard repo: after a local turn the bridge appends one line naming files the report mentions that are unchanged, and files it mentions that are not in the tree.
