---
name: to-runbook
description: Rewrite a research report, technical note, or architecture doc into a runbook that a junior engineer, a DevOps on-caller, or a non-technical reader can follow to completion. Use when the user asks for a runbook, an SOP, an operations manual, a launch or migration procedure, or says "this is correct, but a new hire could never execute it".
---

A report succeeds when it is **correct**; a runbook succeeds when the **reader gets to the end**.

A document that is entirely correct and still strands a junior at step three is a failed runbook. Rewriting one is not reformatting — it is putting back everything the author left out because they knew it and the reader does not.

## Report vs. runbook

| A report writes | A runbook must write |
|---|---|
| Why the mechanism holds | Which line to type right now |
| "Confirm the settings are correct" | The exact text that will appear on the reader's screen, to compare against |
| Background first, to build context | The answer first, background folded away |
| Assumes the reader supplies the judgement | The judgement is already made for the reader; they only compare |
| Describes the correct path | Also describes how to get back when they stray off it |

## 1. Pin down the reader

A runbook has no "general reader". Start by writing a two-column table: **safe to assume they know**, **not safe to assume they know**.

That table is the sole basis for every later decision — which terms get explained on the spot, which judgements get rewritten into comparisons.

If the user has not said who the reader is, ask. Do not guess. "Junior DevOps", "engineer on an application team", and "a PM who never opens a terminal" produce three different documents.

Pin down the **language** at the same time: write in the reader's language. Source doc in English, readers a Chinese-speaking team — write Chinese.

## 2. Define what success looks like

Write, in one sentence, a result the reader can **see with their own eyes**.

- ✅ "The client on c1 calls `orders-internal.payments.svc.cluster.local:8080`, gets a 200, and that request shows up in the server log on c2."
- ❌ "Cross-cluster traffic works."

If you cannot write an observable sentence, the runbook is still missing a way to verify. Supply the verification first, then keep writing.

## 3. Lay out the skeleton

Execution and validation stay separate, and the numbering is fixed document-wide: Execute uses `E0`, `E1`, `E2`…, Validate uses `V1`, `V2`, `V3`…. Conditional steps take a letter (`EP` = policy). The numbers in the fast path and the numbers on the detailed sections must be one single set.

For the full section order, the convergence condition of each section, and the anti-patterns, read [`SHAPE.md`](SHAPE.md).

## 4. The fixed skeleton of every step

Every E/V section uses the same set of fields, so that by step three the reader already knows what step four will look like:

| Field | Content |
|---|---|
| What this step does | One sentence. First line under the heading |
| read / write | `read` = query only, getting it wrong breaks nothing; `write` = changes the cluster or live state |
| Command or YAML | Copy-paste runnable, with no unresolved `<...>` left in it |
| ✅ You should see | Paste the actual output. Not a description of the output |
| 💡 Why | Only when this step is counter-intuitive. One or two sentences |
| If you don't see it | Stop; which step to go back to, which table to read, or which output to carry to whom |
| ⚠️ How to undo | `write` steps only. One executable rollback command |

**Every `write` step needs a rollback.** A runbook without rollbacks is a one-way ticket — the reader who discovers halfway through that something is wrong can only call for help. This is the single most commonly dropped item when a report becomes a runbook.

"Copy-paste runnable" assumes the reader works straight through to the end. **When the runbook spans multiple days, that assumption fails silently** — shell variables and functions die with the window, and an empty variable usually raises no error; it just quietly picks a different target. For how to handle it, see "Multi-day runbooks" in [`SHAPE.md`](SHAPE.md).

## 5. Reader simulation: three questions

With the reader identity pinned in step 1, walk the draft from the first step to the last. At every step, ask three questions:

1. **Does this step contain a term they don't know?**
   Check it against the "not safe to assume they know" column. If yes, add one sentence **at the spot where the term first appears**, saying what role it plays in this step. Do not send them off to a background section to assemble it themselves.

2. **Can they tell that this step succeeded?**
   The judgement must be completable by **comparison**, never by **understanding**. "Confirm the output meets the requirement" does not count; pasting the text that will appear on their screen does.

3. **When it fails, do they know what to do next?**
   "Next" must be an action: go back to E1, read the symptom table, or paste this output to the platform team. "Check whether the configuration is correct" is not an action.

Any question that does not answer "yes" is a defect. Fix it in place before moving on.

**Completion criterion: every step passes all three questions.** Not a sample, not a spot check.

### Before and after the three questions

Before (report style — the reader has to judge for themselves):

> Confirm that the output allows these operations:
>
> | c1 / caller namespace | `get` and `list` Pods; `create` `pods/exec` |

After (runbook style — the reader only compares):

> ### E0-2 Confirm your permissions are sufficient · read
>
> Run all four; all four must return `yes`:
>
> ```bash
> kubectl --context "$C1" -n "$CALLER_NS" auth can-i get pods
> kubectl --context "$C1" -n "$CALLER_NS" auth can-i list pods
> kubectl --context "$C1" -n "$CALLER_NS" auth can-i create pods/exec
> kubectl --context "$C1" -n "$CALLER_NS" auth can-i create pods/portforward
> ```
>
> ✅ You should see:
>
> ```text
> yes
> yes
> yes
> yes
> ```
>
> 🙋 If any line returns `no`: your account lacks the permissions and the later steps cannot proceed. Paste the commands above together with their output to your cluster administrator and ask them to grant the ones that returned `no`.

Same information — the judgement moved off the reader and into the document.

## 6. Tone and emoji

Four rules on tone:

- One sentence, one thing.
- Use the imperative: "run this command", not "this command should be executed".
- One name per thing, document-wide. Pick `c1`, "source cluster", or "old cluster", then use it from start to finish.
- Say what to do before why. Mechanism goes after they have done it, or gets folded away.

Emoji follow the same rule as colors: **use one only when it carries a fixed meaning; otherwise use none.**

| Emoji | Meaning | Where it appears |
|---|---|---|
| 🎯 | What this document sets out to achieve | The top, once per document |
| ✅ | Seeing this means it worked | Expected results |
| 🛑 | Seeing this means stop, do not continue | Fail-fast |
| ⚠️ | This step changes live state | `write` steps, rollbacks |
| 💡 | Counter-intuitive mechanism | The "why" |
| 🙋 | You are stuck here; go find a human | Escalation points |
| ⬜ | An acceptance item still to be ticked | The sign-off checklist |

Emoji appear only in those seven situations, at most one per paragraph; three paragraphs in a row carrying one is too many. Bullets use `-`.

**Headings are text only.** Markdown generates anchors from heading text, so an emoji in a heading leaves invisible residue in the anchor (the variation selector on `⚠️` grows an extra segment), and your hand-written links stop matching — invisibly in the rendered output, discovered only when someone clicks. For markers like read/write that belong on the heading, write them as words: `## E1 Open the entry point on c2 · write`.

One emoji carries exactly one meaning throughout a document.

## 7. Before you deliver

| Status | Check |
|:---:|---|
| ⬜ | The reader table exists, and every term in its "not safe to assume they know" column is explained in one sentence where it first appears |
| ⬜ | Every step passes the three questions |
| ⬜ | Every `write` step is marked ⚠️ and carries a rollback command |
| ⬜ | Every `<...>` has a retrieval command in the variable section; the commands inside steps are copy-paste runnable |
| ⬜ | If the runbook spans multiple days: the variable block can be re-run on its own, and an empty variable is caught rather than silently aimed at the wrong target |
| ⬜ | The fast path's numbers and links line up exactly with the detailed sections' headings |
| ⬜ | Only the seven emoji above appear, none decorative, and none inside a heading |
| ⬜ | Every row of the sign-off checklist can be proven by the output of a command that already appeared in the document |
| ⬜ | Hand it to someone who did not help write it: can they say how many steps there are, what the evidence of success is, and which step means calling for help |

That last row is the only real acceptance test. Passing the first eight and failing that one means the runbook is not finished.
