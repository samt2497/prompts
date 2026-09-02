---
name: durable-batch-execution
description: >
  Run a long batch job as a filesystem-backed transaction so an agent cannot hallucinate progress or
  claim a completion it did not verify. Nothing counts until it is persisted; progress is quoted from
  a checker, never from memory; completion has exactly one valid receipt. Use when an agent must
  generate, render, migrate, import, or process many items in one run, when a batch was interrupted
  and must resume without redoing valid work, when auditing an incomplete batch, or when an agent
  must prove every item was persisted rather than merely submitted. Trigger on "run the batch",
  "resume the batch", "finish the range", "why did it say done when it wasn't", or any long-running
  job where a tool call returning successfully is not the same as the work being durable.
---

# Durable Batch Execution

Treat the batch as a **filesystem-backed transaction**.

A tool call, a returned preview, a queued request, a successful HTTP response, or a progress sentence
is not a completed item. An item counts only once every artifact it owes exists on disk: its primary
output, its input record, and its provenance row.

This skill exists because long batch runs fail in a specific way. The agent does the work, loses
track of what actually landed, and then reports from memory. The report is confident and wrong.
Every rule below is a defense against that one failure.

## Contract

These are invariants, not suggestions.

- **One agent owns the whole batch.** Do not delegate individual items to parallel workers unless the
  user explicitly asks for that. Parallel workers multiply the bookkeeping problem this skill exists
  to solve.
- **Preserve the full requested range as the completion boundary**, but work one item at a time.
  Finishing eight of ten items is a blocked report, not a completion.
- **Refuse bad starting states.** Only items in the correct precondition state may enter the batch.
  Items in a failed, unprepared, or frozen state are refused before any work begins, not skipped
  silently mid-run.
- **Persist and checkpoint each result before starting the next call.** No batching up writes at the
  end. An interrupted run must leave behind a truthful partial state.
- **Never report progress from memory.** Do not say "working", "processing", "done", or give a
  percentage from recall. Run the checker and quote its persisted count.
- **Never end on an in-progress promise.** Either keep calling tools, report a concrete blocker, or
  report the final receipt. "I'll continue in the background" is not one of the three.
- **A successful tool return is not an accepted item.** Neither is a queued job, nor an old progress
  receipt. None of them may be converted into a completion claim.

## The five verbs

Implement these for your domain. They are the whole mechanism. Each one must be a real command that
reads the filesystem, not a judgment the agent makes.

| Verb | Must guarantee | Must fail when |
| --- | --- | --- |
| `preflight` | Expands the selector, reads every item record and its parent scope, and proves the declared item count, the exact input inventory, and the state all agree. | Any input is missing, any count disagrees, any item is in a refused state, or the parent scope is frozen. It blocks the **entire** batch before a single call is spent. |
| `progress` | Reports a persisted count and a delta since the last checkpoint. This is the only source of progress truth. | Never fails loudly. If the count did not increase, the agent may not claim progress. |
| `review` | Assembles the produced outputs into one reviewable bundle for inspection. For visual output this is a contact sheet; for data it may be a diff or a sample table. | The bundle cannot be built because outputs are missing or malformed. |
| `accept` | Records a **checksum-bound** acceptance after review actually happened. Stores a hash of each accepted artifact. | Called without an explicit reviewed flag, or before the review bundle exists. |
| `verify-final` | The single valid completion receipt. Re-checks every artifact against its recorded checksum, confirms provenance is exact and output-backed, confirms every item reached the target state, and reads the result back from the live system of record. | Any checksum drifted, any provenance row is missing or approximate, any status is short of target, or the live readback disagrees with local files. |

Two details carry most of the weight.

**Checksum-bound acceptance.** Acceptance stores hashes, so a later edit or a partial overwrite
invalidates the receipt instead of silently passing. Without this, "reviewed" degrades into "was
reviewed at some point, possibly before the last three changes."

**Live readback.** `verify-final` must confirm against the running system that will actually serve or
consume the output, not only against local files. Local files agreeing with each other proves
nothing about what the downstream system sees.

## Workflow

Set the selector once, then:

1. **Preflight.** Run it and let it pass before any work call. Treat a preflight failure as a hard
   stop with a concrete missing-input list, not something to work around.
2. **For each pending item, in order:**
   - Read only what this item needs. Do not load the whole batch's context.
   - Make exactly one work call.
   - **Immediately** persist the output to its canonical path, normalize it to the declared shape,
     and retain the input record alongside it.
   - **Immediately** write or update the provenance row: the logical identifier, the output file, the
     input file, the mode, the provider and model actually used, and the call identifier.
   - Run `progress` before another work call. Use its delta for any status update.
3. **Review.** Build the bundle and actually inspect it. Inspect at full fidelity anything
   correctness-sensitive, such as text that must be spelled right or values that must reconcile.
   Re-run failures and rebuild the bundle.
4. **Accept.** Only after real review, record the checksum-bound acceptance.
5. **Advance state** for the complete batch together, never item by item at the end.
6. **Verify.** Run the narrowest relevant tests, then emit `verify-final` as the only completion
   receipt.

**Resuming.** Re-run `preflight`. Valid persisted items are retained and only pending ones are
processed. Never delete or overwrite a valid item during a resume.

## Reporting

Two shapes. Nothing else is a valid ending.

Blocked:

```text
BLOCKED 31/36 persisted. Missing inputs: item-672/slot_10..12, item-673/slot_11..12.
No work calls started.
```

Complete:

```text
COMPLETE 3/3 groups, 36/36 items persisted and shape-verified; acceptance checksums match;
canonical and live state agree; provenance <provider>/<model> x12 per group.
```

Both quote counts that came from a command. Neither contains an adjective about how the run felt.

## Adapting this

Replace four things and the rest carries over:

- **The item and its artifacts.** What is the unit, and exactly which files must exist for it to
  count? Write that list down first; the whole contract hangs off it.
- **The precondition state.** Which states may enter a batch, and which are refused?
- **The review bundle.** What does a human or agent actually look at to accept the work?
- **The system of record.** What must `verify-final` read back from, and what does it compare?

Keep the verb names. Their value is that they are boring and unambiguous, so an agent cannot blur
"submitted" into "done" by choosing a softer word.

## Why each rule exists

Every rule below is a real failure mode, not a hypothetical.

- An agent reports a percentage from memory that is higher than what landed, and the operator only
  finds out at delivery. Hence: quote the checker.
- A batch is interrupted, and the resume regenerates everything because nothing recorded what was
  valid. Hence: persist and checkpoint per item.
- A batch runs to completion with one input file missing, and fails on the last item after spending
  the full cost. Hence: preflight blocks the whole range up front.
- Output is reviewed, then partly regenerated, and the stale approval still reads as current.
  Hence: checksum-bound acceptance.
- Local files all agree, the downstream system shows something else entirely, and every intermediate
  check passed. Hence: live readback in the completion receipt.
- The agent ends its turn with "continuing now", and no one is running. Hence: no in-progress
  endings.
