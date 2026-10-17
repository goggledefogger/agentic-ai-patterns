---
type: pattern
date: "2026-10-16"
source: A 9B local agent model asked to list a folder, run through a coding-agent harness against a local llama.cpp server, measured 2026-10-16
tags:
  - local-models
  - prompting
  - tool-use
  - probes
---

# A Bare Prohibition Reads as No Tools

A 9B agent model was asked to list the files in a folder. The instruction ended with "Do not open any file." The model printed the folder path and stopped. 0 tool calls. The same thing happened on the second run. Nothing errored, the answer was well formed, and it was useless.

Reworded as "Use ls, and do not read the contents of any file", the same model on the same folder passed in 5 seconds: 1 tool call, 16 file names returned.

## The Pattern

To a small model, a prohibition with no instruction ahead of it reads as "do not use tools". The model has to hold two things at once: what it is allowed to do, and the one thing it must not. When only the second is stated, it resolves the ambiguity by doing the least, and the least is no tool call at all. A larger model infers the permitted action from the task. A small one often does not.

Say what to do first, then what not to do. Name the tool or the action ("Use ls"), then narrow it with the prohibition ("do not read the contents of any file"). The prohibition still earns its place, because it bounds the action. It just cannot be the only thing the instruction says.

## Where It Applies

- Probes written to test a small model. A failed probe may be measuring your wording, not the model. Before you score a 0-tool-call result as a capability failure, reword the instruction positively and run it again
- Task prompts for small local models
- System prompts for small local models, where a long list of "never" lines can add up to a model that does nothing

## Why Reading the Output Doesn't Catch This

A run with 0 tool calls and a clean, confident reply looks like a model that understood and declined. There is no error to find. The tell is the tool-call count, and only a harness that records it shows the gap. Check that number for any task that was supposed to touch something.

## Related

- [[a-small-model-chooses-it-does-not-rewrite]], another case where the shape of the ask, not the model's raw ability, decides whether a small model does the work
- [[a-small-model-moves-the-file-and-drops-the-behavior]], from the same session: what a small model does reliably once it is calling tools

## Source

Measured 2026-10-16 with a 9B agent model driven by a coding-agent harness against a local llama.cpp server. "Do not open any file." produced the folder path and 0 tool calls, twice. "Use ls, and do not read the contents of any file" produced 1 tool call and 16 names in 5 seconds.
