# The shape of a runbook

The reader should know what they are about to do within the **first two screens**. Mechanism and version detail wait until they are done.

## Section order

| # | Section | Convergence condition |
|---:|---|---|
| 1 | 🎯 Goal · Not covered · Prerequisites | Three one-sentence callouts. On finishing them the reader knows whether this document is the one they want, and who has to prepare the environment for them first |
| 2 | The answer first | Three to five rows: which things get done, and why each is needed. No background |
| 3 | Shortest execution path | Phase, action, success condition, section link — nothing else. The reader sees the total step count at a glance |
| 4 | Values to prepare | Every variable listed in one place, each with the command to obtain it. After this section no undefined `<...>` appears anywhere |
| 5 | Execute `E0` → `En` | Written step by step on the skeleton in section 4. Conditional steps state outright when they apply |
| 6 | Validate `V1` → `Vn` | Each validation stands on its own, never relying on the reader remembering something seen earlier |
| 7 | When you're stuck | Ordered by the chain of evidence. Every branch states who owns it |
| 8 | Sign-off checklist | Every row provable by the output of a command that already appeared |
| 9 | Mechanism and references | Folded. Opens by saying "you do not need this section to follow the procedure" |

Sections 1 through 4 together stay under one screen of scrolling. Going over means background has crept back to the front.

## Anti-patterns

| Don't lay it out like this | Why it's bad | Instead |
|---|---|---|
| Architecture diagram and version matrix before the answer | The reader gets lost in background before knowing what they must do | Answer and shortest path first; background folded into section 9 |
| A block of validation and troubleshooting after every step | The main line gets chopped up and the total step count disappears | Disclose execution in full first, gather validation in section 6 |
| The fast path says Step 1, the detailed section is called E1 | Anchors don't match and the mental model breaks | One set of numbers document-wide |
| "It should succeed" in place of concrete output | Nothing to accept against | Write the Pod count, container name, IP type, HTTP status code, log line |
| The platform team's responsibilities written as the reader's preflight | The reader is asked to check something they have neither the rights nor the power to handle | Put it in section 1 under "prerequisites"; escalate when it does not hold |

## Variables and placeholders

Section 4 gives every variable a row, together with the command that obtains it:

| Variable | Meaning | How to get it |
|---|---|---|
| `$CALLER_POD` | The Pod that issues the request | `kubectl --context "$C1" -n "$CALLER_NS" get pod -l app=frontend -o name` |

Then give one block of `export`s that can be pasted straight into a terminal.

**Rule: no `<...>` survives inside a step's commands.** Copy and paste has to run. A line like `kubectl exec "$POD" -c <app-container>` gets pasted into the terminal verbatim — replace it with `$APP_CONTAINER` and give the retrieval command in section 4.

For a placeholder that must stay inside YAML, put one line directly above the code block naming which placeholders to replace and what to replace them with.

### Multi-day runbooks: the variable block must be re-runnable on its own

Canary traffic shifts, certificate rotation, batched migrations — the reader will not do these in one sitting. They do R1 today and R4 five days later, with the terminal closed several times in between.

Shell variables and shell functions live inside that one window and vanish with it. The "copy-paste and run" rule assumes a straight run to the end, and across days it fails silently.

**The test: the reader closes the terminal, opens a fresh one the next day, runs only the variable block, and can carry on with any later step.**

Three requirements:

1. **The variable block is one single re-runnable code block**, holding the retrieval commands for looked-up variables and every shell function the document uses. Do not scatter it across several fragments.
2. **A self-check at the end** that shouts loudly when any variable is empty.
3. **One line at the top of every cross-stage step**: in a new window, re-run the variable block first.

Point 2 is not idiot-proofing, it is disaster prevention. Many CLIs handle an empty value by **quietly falling back to a default** rather than erroring — `kubectl --context "" get ns` does not fail, it uses the current context, which may well be a different cluster. The reader executes a `write` step against the wrong target, with no error message at all.

The check must read values through `eval` to work in both bash and zsh (`${!v}` is bash-only):

```bash
for v in C1 C2 NS SVC CLIENT_CONTAINER; do
  eval "val=\$$v"
  [ -n "$val" ] || echo "🛑 $v is empty; continuing will hit the wrong place"
done
```

Where a variable's existence can be verified, verify it as well — whether the name resolves is worth more than whether it is non-empty.

## Tables, Mermaid and callouts

| Kind of information | What to use |
|---|---|
| Multi-field comparison, version differences, symptom lookup | Table |
| Data flow, sequence, branching decisions | Mermaid |
| A single warning or blocking condition | A one-line callout |
| One to three linear steps | Just write them; don't force a diagram |

Draw Mermaid only when a relationship or a decision flow really is easier to grasp as a picture than as text. Any diagram you draw carries a color key underneath it.

### Color semantics

Fixed within a document; meanings never get reassigned:

| Color | Meaning | Codes |
|---|---|---|
| Blue | Source side / starting point | fill `#E8F1FB` stroke `#2563EB` |
| Green | Target side / end point | fill `#E8F5E9` stroke `#2E7D32` |
| Gray | Neutral information, control plane | fill `#F3F4F6` stroke `#6B7280` |
| Red | An observed failure; must stop | fill `#FEE2E2` stroke `#DC2626` |
| Amber | A corrective action still to be taken | fill `#FEF3C7` stroke `#D97706` |

When a diagram carries no difference of role, state, or ownership, leave it in the default neutral color.

## How to order "When you're stuck"

Order by the **chain of evidence**: start from the last observation the reader knows to be correct and advance one link at a time.

For a cross-cluster call, that reads: does DNS resolve → does the sidecar see the remote endpoint → does TCP connect → is policy blocking it → is it readiness or TLS → and only then the application itself.

This section has three parts:

1. **A decision tree** (Mermaid): the reader walks from symptom to action.
2. **A symptom table**: symptom → most likely cause → what to check first → which direction to fix in.
3. **Ownership**: every row states whether the reader can fix it themselves or must escalate.

The escalation rows spell out **what to bring** — which output of which command. That lets the reader open a ticket carrying evidence, instead of throwing "cross-cluster is broken" over the wall.

Never tell the reader to change the architecture to "see if that helps". Temporarily switching to a LoadBalancer or NodePort, or bypassing the intended path, replaces the architecture under test and leaves the problem where it was.

## Bilingual

When an English and a Chinese version are maintained together, these must match: goal and scope, section order and numbering, YAML and shell commands, variable names, Mermaid nodes and edges and color semantics, fail-fast conditions, every row of the sign-off checklist.

Only these may be localized: sentence shape and word order, punctuation, the natural phrasing of callouts, and heading translations that do not change the meaning.

The two versions move in lockstep: any substantive change updates both in the same pass. When the user explicitly asks for a single-language change, say on delivery that the other version is not yet in sync.
