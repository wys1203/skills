## What it does

`grill-evidence` interviews a witness to collect evidence for a symptom the agent cannot observe itself: "I saw a 404 in the client log", "the job failed last night". It is `grilling` pointed at a diagnosis. The same tree, frontier and rounds, but the nodes are **hypotheses** and the answers are **evidence**, not decisions.

The first round draws the **path**: what the client is, every hop between it and the target, where each hop's logs live. Only then does it build a **hypothesis tree**, one branch per hop that could have emitted the symptom, and start asking for evidence. Each question says how to fetch the evidence and what each possible answer would rule in or out. An **evidence ledger** (established, ruled out, open) is restated every round.

It is **stateless** on its own: the ledger lives in the conversation. `grill-reporter` is the stateful front door that keeps it in a Jira ticket.

## When to reach for it

Type `/grill-evidence` and the symptom, or let the agent reach for it when someone reports a symptom whose facts live in logs, environments, or people it cannot query.

| What you have | Reach for |
| --- | --- |
| A symptom, and you can fetch the evidence yourself | `grill-evidence` |
| A symptom reported by someone else on a Jira ticket | `grill-reporter` |
| Enough evidence to build a reproduction loop | `diagnosing-bugs` |
| A plan or decision to stress-test, not a symptom | `grilling` / `grill-me` |

It sits **before** `diagnosing-bugs`. That skill refuses to theorise without a tight feedback loop and assumes the agent can build one; `grill-evidence` is for when the agent cannot reach the environment and must get the evidence through a person first.

## Two roles

The **investigator** owns the tree: scope, decisions, when to stop, the root cause. The **witness** owns the evidence: fetching it, confirming it, denying it. A witness cannot end the session by saying the evidence is enough. When you run the skill on your own case, you are both.

## Facts, in three buckets

Every question needs a fact. Facts the agent can fetch (repo, cluster, deploy history) go to a sub-agent. Facts only the witness can fetch (production logs, the client device, another team) go to the witness with an exact command and what to paste back. Decisions go to the investigator. The first bucket is tested, not assumed: when a lookup fails for lack of tools or access, the agent says so, hands the fact to the witness, and records the access gap in the ledger. It never fills the gap with a guess.

## It's working if

- The first round asks only about the path: no hypotheses, no "how to get it" for evidence.
- A blank in the path is never accepted as "don't know": the agent fetches it or gives a command that reveals it.
- Every question has a 🔍 line and a ➡️ line, and the ➡️ line reads "if you see X it is H1, if Y it is H2", not a recommended answer.
- When the agent cannot fetch something, it says what it tried and why it failed, then hands the fact over with a command.
- The ledger is restated every round, every Established entry names a source, and Ruled out grows as rounds go by.
- Questions are short, one idea per sentence, technical terms in English, with a Terms block at the end of the round.
- It stops when one hypothesis survives, when the investigator says so, or when it hands off to `diagnosing-bugs`. Not when it runs out of questions.

## Common questions

**It skipped the path round and went straight to hypotheses.**
Weaker models and lower effort settings do this most. Say "draw the path first". If it keeps happening, point it at [EXAMPLE.md](EXAMPLE.md).

**It wrote something into Established that nobody told it.**
That is model knowledge leaking into the ledger. Ask where the entry came from; if the answer is not a message, a tool output, or a pasted log, it comes out. The rule is in the skill; some models need reminding.

**The command it gave me does not run.**
Paste the error back. Commands the agent has not run are marked untested for this reason; the next round fixes the command rather than re-asking the question.

**It asked everything at once with no 🔍 or ➡️ lines.**
The skill did not load, or a front door named it without loading it. Ask the agent which skills it has loaded.

**How long does it take?**
Two rounds at minimum: one for the path, one for evidence that narrows to a single hypothesis. There is no cap. If Ruled out has not grown for two rounds, the cheap evidence is exhausted; hand off to `diagnosing-bugs`.

## Where it fits

`grill-evidence` is a **primitive**. `grill-reporter` is its front door for Jira tickets; `triage` step 4 could call it instead of `grilling` for bug reports. Downstream, `diagnosing-bugs` takes over once the evidence supports a reproduction loop, and `to-runbook` can turn a solved case's ledger into an operations procedure. See [DESIGN.md](DESIGN.md) for why each rule exists and [VALIDATION.md](VALIDATION.md) for running it on other models and harnesses.
