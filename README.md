# Lessons from building with agents

229 lessons from one person building software with coding agents, mostly
the hard way. Each file in `patterns/` is one lesson: what happened, why it
happens, and when to expect it again.

This is a map, not a standard. Nothing here is a rule you should adopt because
it is written down. These are the shapes that kept recurring in one person's
work, written so you recognise the shape when it shows up in yours. Where a
lesson came out of a specific incident, the incident is described and the
names are gone.

## How to read it

Two guides sit at the root and cover the setup the patterns assume:

- [claude-code-obsidian-guide.md](claude-code-obsidian-guide.md) — running a
  markdown vault as an agent's working memory: structure, conventions,
  automation, and the session habits that keep it honest.
- [agent-personality-guide.md](agent-personality-guide.md) — extending that
  vault into a persistent agent with its own identity, voice, and boundaries.

Then the patterns. Read the index below and follow whatever names a problem
you have had. They are deliberately short and they cross-reference each other.

## How to use it

Clone it next to your own notes as a read-only store your agent can search:

```bash
git clone <this repo> ~/reference/agent-lessons
```

Point your agent at that directory and tell it to consult the patterns before
designing something, the way it would consult any other reference. Nothing
here writes to your system, and nothing here needs to be installed. Reading it
straight through also works.

It is listed as a free World in the Astrolabe store, so an agent with Astrolabe
can pick it up from there instead.

## How it updates

`git pull`. New lessons land as they are learned, which is irregular. Existing
files get revised when a later incident proves an earlier write-up wrong, so a
file's date is when it was first written, not when it was last right.

The tools half of this work — skills, scripts, templates — stays in a private
companion repo (`goggledefogger/obsidian-claude`) for now.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md). The short version: real incidents
only, no personal data, no private paths, and it has to work for someone who
is not the author.

MIT licensed.

## The patterns

- [A capability contract must name the verb](patterns/a-capability-contract-must-name-the-verb.md)
- [A Check That Never Ran Reads As Passing](patterns/unrun-checks-read-as-passing.md)
- [A Clone Shares the Identity Your Gate Scopes By](patterns/a-clone-shares-the-identity-your-gate-scopes-by.md)
- [A Closed Ticket Is Not a Home for an Open Decision](patterns/a-closed-ticket-is-not-a-home-for-an-open-decision.md)
- [A Convention Needed Once Is a Convention Skipped](patterns/a-convention-needed-once-is-a-convention-skipped.md)
- [A Count Inherits the Freshness of the Replica It Was Counted From](patterns/replica-freshness-travels-with-the-count.md)
- [A Deferred Fallback Decides at Fire Time, Not Arm Time](patterns/a-deferred-fallback-decides-at-fire-time.md)
- [A Derived Location Is a Guess, Not an Address](patterns/a-derived-location-is-a-guess-not-an-address.md)
- [A dry run that derives will drift](patterns/a-dry-run-that-derives-will-drift.md)
- [A Fake Can Only Fail the Ways You Have Already Seen](patterns/a-fake-can-only-fail-the-ways-you-have-seen.md)
- [A Fast Answer Is A Suspect Answer](patterns/a-fast-answer-is-a-suspect-answer.md)
- [A Free-Form Ask Lands on Decided Rails](patterns/a-free-form-ask-lands-on-decided-rails.md)
- [A Gate Can Block, But It Cannot Speak](patterns/a-gate-can-block-but-cannot-speak.md)
- [A Generated File Accepts the Write It Will Not Keep](patterns/a-generated-file-accepts-the-write-it-will-not-keep.md)
- [A Grant the Doctrine Denies Is Reported as a Bug](patterns/a-grant-the-doctrine-denies-is-reported-as-a-bug.md)
- [A Guard Scoped To The Place, Not The Property](patterns/guard-scoped-to-the-place-not-the-property.md)
- [A Guard's Evidence Must Outlive the Failure It Detects](patterns/guard-evidence-outlives-the-failure.md)
- [A Health Check That Never Exercises the Work](patterns/health-check-that-never-exercises.md)
- [A Healthy List Is Not a Healthy Server](patterns/a-healthy-list-is-not-a-healthy-server.md)
- [A Heuristic Where an Exact Key Exists](patterns/a-heuristic-where-an-exact-key-exists.md)
- [A Known-Broken Lane Must Stay Visible and Must Never Page](patterns/known-broken-must-not-page.md)
- [A Live Reference Outlives Its Owner, and Everyone Keeps Traversing It](patterns/live-reference-outlives-its-owner.md)
- [A Model Never Picks the Destination](patterns/a-model-never-picks-the-destination.md)
- [A Model Swap Is Adopted by Blind Verdict, Not by Vibes](patterns/a-model-swap-is-adopted-by-blind-verdict.md)
- [A Monitor That Can't Exclude Its Own Exhaust Will Eventually Tell You to Stop Working](patterns/monitor-cannot-see-its-own-exhaust.md)
- [A New Guard Relocates the Failure Mode of Every Test That Crosses It](patterns/a-new-guard-relocates-every-test-that-crosses-it.md)
- [A Partial Read Proves Presence, Never Absence](patterns/a-partial-read-proves-presence-not-absence.md)
- [A Plausible Cause Ends the Search](patterns/plausible-cause-ends-the-search.md)
- [A Pooled Balance Couples Fate — Cap Each Consumer](patterns/pooled-balance-couples-fate.md)
- [A Push That Asks States the Question, and What a Reply Does](patterns/a-push-that-asks-states-the-question-and-what-a-reply-does.md)
- [A Recommendation Is Not a Decision](patterns/a-recommendation-is-not-a-decision.md)
- [A Recovery Hint Must Derive From Its Detector](patterns/a-recovery-hint-must-derive-from-its-detector.md)
- [A Refusal Names a Rule, Not a Wall](patterns/a-refusal-names-a-rule-not-a-wall.md)
- [A Refused Reconnect Ends the Stream](patterns/a-refused-reconnect-ends-the-stream.md)
- [A Required Slug Invents the Missing Half](patterns/a-required-slug-invents-the-missing-half.md)
- [A Resumed Session Answers From Yesterday](patterns/a-resumed-session-answers-from-yesterday.md)
- [A Ritual Is a Skill Plus a Wrapper](patterns/a-ritual-is-a-skill-plus-a-wrapper.md)
- [A Rule Must Live Where Matching Happens First](patterns/a-rule-must-live-where-matching-happens-first.md)
- [A Second Writer Satisfies Your Gate: Evidence in a Shared Store Names No Actor](patterns/a-second-writer-satisfies-your-gate.md)
- [A Self-Report Must Not End What It Is Reporting On](patterns/self-report-must-not-end-what-it-reports-on.md)
- [A Small Model Chooses, It Does Not Rewrite](patterns/a-small-model-chooses-it-does-not-rewrite.md)
- [A Source Assertion Pins Your Belief, Not the Behavior](patterns/a-source-assertion-pins-your-belief-not-the-behavior.md)
- [A stack of proxy watchers is one watcher](patterns/a-stack-of-proxy-watchers-is-one-watcher.md)
- [A Stale Pointer Doesn't Go Quiet, It Asserts — and Cleanup Can't Cross Repos](patterns/stale-pointer-asserts-confidently.md)
- [A Standing Grant Forgets Its Question](patterns/a-standing-grant-forgets-its-question.md)
- [A Store Can Hold Data and Restore Nothing](patterns/a-store-can-hold-data-and-restore-nothing.md)
- [A Stream That Ends Its Unit May Still Owe a Turn](patterns/a-stream-that-ends-its-unit-may-still-owe-a-turn.md)
- [A Stub Outlives the Seam It Watches](patterns/a-stub-outlives-the-seam-it-watches.md)
- [A Subagent Cannot Consent, So a Permission Gate Kills It Silently](patterns/subagent-cannot-consent.md)
- [A Summary Can Invert the Policy](patterns/a-summary-can-invert-the-policy.md)
- [A Symmetric Reference Reads Correctly and Gets Used Backwards](patterns/symmetric-reference-gets-used-backwards.md)
- [A Tier Says What You May Touch, Not What Others May See](patterns/a-tier-says-what-you-may-touch-not-what-others-may-see.md)
- [A Trigger Must Not Watch Its Own Writes](patterns/a-trigger-must-not-watch-its-own-writes.md)
- [A Twin Inherits the Conversation, Not the Channels](patterns/a-twin-inherits-the-conversation-not-the-channels.md)
- [A Value That Crosses a Re-Parse Boundary Must Be Quoted At It](patterns/quote-survives-the-re-parse.md)
- [A Verdict Is Scoped to the Worker That Reached It](patterns/a-verdict-is-scoped-to-the-worker-that-reached-it.md)
- [A Verdict Matrix Proves the Function, Not the Feed](patterns/a-verdict-matrix-proves-the-function-not-the-feed.md)
- [A worker that cannot list cannot find](patterns/a-worker-that-cannot-list-cannot-find.md)
- [A Write's Success Confirms the Write, Not the Content](patterns/opaque-write-needs-a-read-back.md)
- [A Write-Up Outran the Merge: A Documented Fix Is Not a Landed Fix](patterns/a-write-up-outran-the-merge.md)
- [Access Friction Picks Your Evidence](patterns/access-friction-picks-your-evidence.md)
- [Ack Then Work](patterns/ack-then-work.md)
- [An Agent Can Wire A Secret It Never Holds](patterns/an-agent-can-wire-a-secret-it-never-holds.md)
- [An Agent Writes About Its Principal in the Third Person](patterns/an-agent-writes-about-its-principal-in-the-third-person.md)
- [An Approval Crossing a Channel the Approver Can't See Needs Its Own Channel](patterns/eyes-free-approval-channel.md)
- [An Ask No Mode Shows Is Not a Gate](patterns/an-ask-no-mode-shows-is-not-a-gate.md)
- [An Error Flag Owned by One Subsystem Must Never Gate Another Subsystem's Status Report](patterns/error-flag-gags-sibling-status.md)
- [An Event Is Not a Cause](patterns/an-event-is-not-a-cause.md)
- [An Implicit Identity Is a Silent-Wrong-Answer Generator](patterns/implicit-identity-silent-wrong-answer.md)
- [An Unattended Path Must Not Share Fate With an Interactive One](patterns/an-unattended-path-must-not-share-fate-with-an-interactive-one.md)
- [An Unconditional Act Wearing a Conditional Name](patterns/an-unconditional-act-wearing-a-conditional-name.md)
- [An Unenforced Red Becomes the Baseline](patterns/unenforced-red-becomes-the-baseline.md)
- [An Unknown Value Renders As Absence: The Record Looks Authored and Is Invisible](patterns/unknown-value-renders-as-absence.md)
- [An Unnamed Blind Spot Reads As An Empty Source](patterns/an-unnamed-blind-spot-reads-as-an-empty-source.md)
- [Anchor Hygiene Rituals to Session Start, Not Session End](patterns/anchor-rituals-to-session-start.md)
- [Announce the Move in the Old Room](patterns/announce-the-move-in-the-old-room.md)
- [Appending Is Not Learning: Classify Each Correction, and Budget What It Lands In](patterns/appending-is-not-learning.md)
- [Approval scope is invisible to a gate keyed on resource identity](patterns/approval-scope-invisible-to-gates.md)
- [Archive-Split for Auto-Loaded Living Docs](patterns/living-doc-archive-split.md)
- [Ask Who Received Before You Send Again](patterns/ask-who-received-before-you-send-again.md)
- [Ask-Challenge-Verify-Correct](patterns/ask-challenge-verify-correct.md)
- [At-Least-Once Needs a Poison Ledger](patterns/at-least-once-needs-a-poison-ledger.md)
- [Atomic State Writes (Temp-File + Rename)](patterns/atomic-state-writes.md)
- [Capture Before You Answer](patterns/capture-before-you-answer.md)
- [Choosing a Memory Substrate — Wiki vs Structured Store](patterns/memory-substrate-selection.md)
- [Churn-Free Generated Artifacts](patterns/churn-free-generated-artifacts.md)
- [Commit Trailers Must Be Contiguous](patterns/commit-trailers-must-be-contiguous.md)
- [Consent Needs the Diff](patterns/consent-needs-the-diff.md)
- [Convergent Stand-Up, Sloppy Quits](patterns/convergent-standup-sloppy-quits.md)
- [Coordinate Agents Through Shared State, Not Messages](patterns/coordinate-agents-through-shared-state.md)
- [Count the Source, Not the Survivors](patterns/count-the-source-not-the-survivors.md)
- [Curated Manifest + Runtime Join: Docs That Describe What's Actually Running](patterns/curated-manifest-runtime-join.md)
- [Current-Date Grounding](patterns/current-date-grounding.md)
- [DataviewJS for Custom UI Dashboards](patterns/dataviewjs-custom-ui.md)
- [Decision-Doc Pattern (Lightweight ADRs)](patterns/decision-doc-adr.md)
- [Declared Presence Beats the Host Clock](patterns/declared-presence-beats-host-clock.md)
- [Decorative Gate (Anti-Pattern): A Control That Returns "Passed" Without Checking](patterns/decorative-gate.md)
- [Dedup Anchored at the Head Misses the Tail](patterns/dedup-anchored-at-the-head-misses-the-tail.md)
- [Delegation Decision — Skill vs Subagent vs Named Agent vs Direct](patterns/delegation-decision.md)
- [Deployed Is Not Loaded](patterns/deployed-is-not-loaded.md)
- [Desk-Only Means the Flow Returns to the Desk](patterns/desk-only-means-the-flow-returns-to-the-desk.md)
- [Detached Launch From a Time-Boxed Agent](patterns/detached-launch-from-timeboxed-agent.md)
- [Deterministic Orchestrator Over Agent Plumbing](patterns/deterministic-orchestrator-over-agent-plumbing.md)
- [Disarm the Producer Before Deleting the Artifact](patterns/disarm-the-producer-before-deleting-the-artifact.md)
- [Disproving the Theory Is Not Disproving the Sighting](patterns/disproving-the-theory-is-not-disproving-the-sighting.md)
- [Doc-with-Warning-Preamble](patterns/doc-warning-preamble.md)
- [Dual-Audience Views Over One Store (Human Brief vs AI-Comprehensive)](patterns/dual-audience-views-over-one-store.md)
- [Embedded Channels Inherit the Host Document's Read Rate](patterns/embedded-channels-inherit-avoidance.md)
- [Enforce the Message, Not the Field](patterns/enforce-the-message-not-the-field.md)
- [External System as Source of Truth, Vault as Pointer](patterns/external-store-as-source-of-truth.md)
- [External-File Visibility (files Claude creates don't show in Obsidian)](patterns/external-file-visibility.md)
- [Fetch first then read the tree](patterns/fetch-first-then-read-the-tree.md)
- [Filesystem as the Queue (No Broker Needed)](patterns/filesystem-queue.md)
- [Framework-Gotcha Comments](patterns/framework-gotcha-comments.md)
- [Freshness by Timestamp Answers a Different Question Than Freshness by Content](patterns/freshness-axis-must-match-the-question.md)
- [Gate Propagation into Headless Workers](patterns/gate-propagation-into-headless-workers.md)
- [Gate Scheduled LLM Spend at Creation; Prefer a Deterministic Trigger](patterns/scheduled-llm-spend-gate.md)
- [git push and gh Are Separate Credential Paths](patterns/two-github-credential-paths.md)
- [Give the Agent a Lawful Write Surface](patterns/lawful-write-surface.md)
- [Give the manager session a fleet view, not the fleet's contents](patterns/multi-session-fleet-awareness.md)
- [Grammar Parsing Over Text Matching for Detectors](patterns/grammar-parsing-over-text-matching.md)
- [Graph View Tuning](patterns/graph-view-tuning.md)
- [Green tests can mirror the same guess](patterns/green-tests-can-mirror-the-same-guess.md)
- [Grill the Plan](patterns/grill-the-plan.md)
- [Half Gate, Whole Verdict (Anti-Pattern): Evidence for One Axis, a Claim Covering Both](patterns/half-gate-whole-verdict.md)
- [Holding Is the Default, Staleness Is the Bound](patterns/hold-vs-drop-on-interrupted-input.md)
- [Humanizer Second-Pass](patterns/humanizer-second-pass.md)
- [Identity Is a Property of the Set, Not of the Instance](patterns/identity-is-a-property-of-the-set.md)
- [Instrumentation With No Reader Is a Decision to Collect Evidence and Never Look](patterns/instrumentation-with-no-reader.md)
- [Interaction Machinery Belongs to Its Path](patterns/interaction-machinery-belongs-to-its-path.md)
- [Latched State Needs a Reconciler: A Flag Set by Events Will Eventually Be Set Forever](patterns/latched-state-needs-a-reconciler.md)
- [Liveness Is Measured at the Ear](patterns/liveness-is-measured-at-the-ear.md)
- [Living-Doc Refresh Ritual](patterns/living-doc-refresh-ritual.md)
- [LLM Wiki Maintenance](patterns/llm-wiki-maintenance.md)
- [Local Video Understanding Via Frame Extraction](patterns/local-video-understanding-via-frame-extraction.md)
- [Local-Model Agentic Tool-Calling](patterns/local-model-agentic-tool-calling.md)
- [Local-Model Routing for Restricted-Tier Content](patterns/local-model-routing-for-restricted.md)
- [Log the Verdict, Not the Volume](patterns/log-the-verdict-not-the-volume.md)
- [Many Ears, One Transcript](patterns/many-ears-one-transcript.md)
- [Marker Files Are an Undeclared State Machine](patterns/marker-files-are-an-undeclared-state-machine.md)
- [Meaning-Carrying Color Tokens: One Semantic Palette Across HTML and Terminal](patterns/meaning-carrying-color-tokens.md)
- [Meeting Transcript Processing](patterns/meeting-transcript-processing.md)
- [Merge Then Update Consumers](patterns/merge-then-update-consumers.md)
- [Morning Briefing Pipeline](patterns/morning-briefing-pipeline.md)
- [Never Narrow at Intake](patterns/never-narrow-at-intake.md)
- [Never Trade the Screen for a Promise](patterns/never-trade-the-screen-for-a-promise.md)
- [No Delivery Without Arrival Accounting](patterns/no-delivery-without-arrival-accounting.md)
- [NotebookLM as Research Layer](patterns/notebooklm-research.md)
- [One Authority for Repeating Behaviors](patterns/one-authority-for-repeating-behaviors.md)
- [One Engine Per Store Beats a Pretend Span](patterns/one-engine-per-store-beats-a-pretend-span.md)
- [One Pipe, Two Speakers](patterns/half-duplex-turn-taking.md)
- [Open Intent Funnels to Chat; Closed Operations Act in Place](patterns/open-intent-funnels-closed-acts-in-place.md)
- [Parallel Branch Burst: N Branches off One File Is Conflict Debt, Not Throughput](patterns/parallel-branch-burst-conflict-debt.md)
- [Parallel Review → Merged Spec → Implementer → Pixel-Diff Gate](patterns/parallel-review-implement-verify.md)
- [Parallel Workers Share Nothing but the Tail](patterns/parallel-workers-share-nothing-but-the-tail.md)
- [Personality-as-Discipline](patterns/personality-as-discipline.md)
- [Phase Is Computed, UI Is Rendered: Status Hand-Set From Permissive Flags Will Lie](patterns/phase-is-computed-ui-is-rendered.md)
- [Platform-Signed Commits from a Keyless Agent](patterns/platform-signed-commits-keyless-agent.md)
- [Policy on Every Door: A Standing Rule Wired to One Entry Path Silently Skips on the Others](patterns/policy-on-every-door.md)
- [Portable Skill Engine with Self-Scaffolding Local Profile](patterns/portable-engine-local-profile.md)
- [Portable Tool Baseline](patterns/portable-tool-baseline.md)
- [Pre-Push Hook for PR Discipline](patterns/pre-push-pr-discipline-hook.md)
- [Print the Answer, Not a Warning About the Question](patterns/print-the-answer-not-the-warning.md)
- [Quick-Win Scaffolding](patterns/quick-win-scaffolding.md)
- [Quiet-by-Default Notifications; Route Alerts to the Operator](patterns/quiet-by-default-notifications.md)
- [Ranked Noise Wears a Score](patterns/ranked-noise-wears-a-score.md)
- [Reach-Surface Resolution](patterns/reach-surface-resolution.md)
- [Readiness Is Polled, Not Slept At](patterns/readiness-is-polled-not-slept.md)
- [Registry Resolution: Alias-First, Dead-Ends Return the Next Hop](patterns/registry-resolution-alias-and-reach.md)
- [Registry-Based System Monitoring](patterns/registry-based-monitoring.md)
- [Revoking an Identity Breaks Everything Still Authenticating As It, Quietly](patterns/revoking-an-identity-breaks-its-silent-dependents.md)
- [ROADMAP.md Status Categories](patterns/roadmap-status-categories.md)
- [Router/Worker Exfiltration Containment](patterns/router-worker-exfil-containment.md)
- [Scan for Prior Art Before Building Infrastructure](patterns/scan-prior-art-before-building-infra.md)
- [Scan rendered surfaces before building a viewer](patterns/scan-rendered-surfaces-before-building-a-viewer.md)
- [Secret Handling Off-Transcript](patterns/secret-handling-off-transcript.md)
- [Seed the Precondition in the File the Child Reads](patterns/seed-the-precondition-in-the-file-the-child-reads.md)
- [Self-Reporting Staleness Check](patterns/self-reporting-staleness-check.md)
- [Sensitivity-Tiered Access Control for Vaults](patterns/sensitivity-tiered-access-control.md)
- [Session-Scoped Result Cache](patterns/session-scoped-result-cache.md)
- [Settings Follow the Member, Not the Browser](patterns/settings-follow-the-member-not-the-browser.md)
- [Shared Skill Symlinking (The Distributed Skill Layer)](patterns/shared-skill-symlinking.md)
- [Shared Tooling Matches Per-Vault Conventions Leniently](patterns/shared-tooling-matches-conventions-leniently.md)
- [Silence Is Not Confirmation: An Append-Only Log Has No Present Tense](patterns/silence-is-not-confirmation.md)
- [Smoke Tests with Real Data](patterns/smoke-tests-with-real-data.md)
- [Soak-Aware Patch Tests](patterns/soak-aware-patch-tests.md)
- [Split the Doc by Verifiability, Don't Generate It](patterns/split-the-doc-by-verifiability.md)
- [Stacked PRs: Retarget Dependents Before Deleting a Merged Base](patterns/stacked-pr-base-deletion.md)
- [State Published as an Event Is Lost to Latecomers](patterns/state-published-as-an-event-is-lost-to-latecomers.md)
- [Static-Token Tunnel Auth Gate](patterns/tunnel-auth-gate.md)
- [Storage Migration Blinds Your Readers](patterns/migration-blinds-readers.md)
- [Subagent Ceremony by Task Type](patterns/subagent-ceremony-by-task-type.md)
- [Surface Contradictions, Don't Resolve Them](patterns/contradiction-surfacing.md)
- [System-Understanding Protocol](patterns/system-understanding-protocol.md)
- [TDD for Skills (RED-Phase Baseline Behaviors)](patterns/tdd-for-skills.md)
- [Teardown Hands Back to the Surface the User Opened](patterns/teardown-hands-back.md)
- [Tests Pin Substance, Not Identifiers](patterns/tests-pin-substance-not-identifiers.md)
- [The Ack Outran the Write](patterns/ack-outran-the-write.md)
- [The Agent's Home Is Not Its Library](patterns/the-agents-home-is-not-its-library.md)
- [The Conversation Is Unswept State](patterns/the-conversation-is-unswept-state.md)
- [The escape hatch is the tool the caller was denied](patterns/escape-hatch-is-the-denied-tool.md)
- [The Fallback Committed Before the Failure Happened](patterns/fallback-commits-before-the-failure.md)
- [The Hidden Attribute Loses to Any Display](patterns/the-hidden-attribute-loses-to-any-display.md)
- [The Lane Decides Who Approves, Not the Agent](patterns/the-lane-decides-who-approves-not-the-agent.md)
- [The Loaded File Is the Rule; Everything Else Is Commentary](patterns/the-loaded-file-is-the-rule.md)
- [The Losing Lane Keeps Rendering](patterns/the-losing-lane-keeps-rendering.md)
- [The native deny list beats the ported gate](patterns/the-native-deny-list-beats-the-ported-gate.md)
- [The Path You Can Guess Is the Stale One](patterns/the-guessable-path-is-the-stale-one.md)
- [The Pointer Names the File, Not the Policy: "Belongs in X" Reads as Satisfied Without a Lookup](patterns/pointer-names-the-file-not-the-policy.md)
- [The Second Principal Is Invisible to a Reader Built for the First](patterns/the-second-principal-is-invisible-to-a-reader-built-for-the-first.md)
- [The sender enforces the receiver's ceiling](patterns/the-sender-enforces-the-receivers-ceiling.md)
- [The Teardown Was Verified. The Startup Was Never Written.](patterns/teardown-verified-startup-never-written.md)
- [The Thin-Router Orchestrator](patterns/thin-router-orchestrator.md)
- [The Useful Core — Adopt a Skill's Rules, Not Its Machinery](patterns/token-discipline-useful-core.md)
- [The Witness Must Not Share the Pipe: An Observability Tap on an Exclusive Resource Breaks What It Watches](patterns/the-witness-must-not-share-the-pipe.md)
- [Three-Tier Memory Pipeline](patterns/three-tier-memory-pipeline.md)
- [Tolerance Named the Failure, Not the Benign Set (Anti-Pattern)](patterns/tolerance-named-the-failure-not-the-benign-set.md)
- [Treat Always-Loaded Context Files as a Budget, Not a Place to Write](patterns/always-loaded-context-budget.md)
- [Unattended-Run Discipline](patterns/unattended-run-discipline.md)
- [Undefined Blank Is a Decision (Anti-Pattern): An Optional Column Without Blank-Semantics Becomes Decoration](patterns/undefined-blank-is-a-decision.md)
- [Units Belong to the Adapter, Not the Port](patterns/units-belong-to-the-adapter.md)
- [Vault-as-CMS for Static Site Publishing](patterns/vault-as-cms-publisher.md)
- [Verdicts From Structured Signals, Not Prose Matching](patterns/verdicts-from-structured-signals.md)
- [Verification Needs a Negative Control](patterns/verification-needs-a-negative-control.md)
- [Verify Adoptions Against the Installed Source](patterns/verify-adoption-against-installed-source.md)
- [Version Bump Is the Delivery: Merged ≠ Shipped Under a Version-Pinned Cache](patterns/version-bump-is-the-delivery.md)
- [Vertical Issue Split](patterns/vertical-issue-split.md)
- [Wikilink Graph Hygiene](patterns/wikilink-graph-hygiene.md)
- [Wire Adoptions Into Existing Flows](patterns/wire-into-existing-flows.md)
