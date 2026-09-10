---
name: grill-evidence
description: Interview the user to collect evidence for an incident or bug they can observe but you cannot reach. Use when the user reports a symptom ("I saw a 404 in the client log", "the job failed last night") and the facts live in logs, environments, or people you cannot query yourself.
---

Interview a witness to gather **evidence** until one explanation of the symptom survives. This is `grilling` pointed at a diagnosis: the same tree, frontier and rounds, but the nodes are **hypotheses** and the answers are **evidence**, not decisions.

Two roles take part, and they are not always the same person. The **investigator** owns the tree: scope, decisions, when to stop, and the root cause. The **witness** owns the evidence: they fetch it, confirm it, or deny it, and nothing more. A witness cannot end the session by saying the evidence is enough; only the investigator can. Unless the skill that called you says otherwise, the user is both.

The symptom is not the root of the tree; the **path** is. A symptom is emitted by one hop on the path from the observer to the system, and you cannot list who might have emitted it until you know the hops. So the first round maps the path, and nothing else: what the observer is (browser, mobile app, CLI, another service, a job) and who owns it; every hop between it and the target (proxy, CDN, gateway, mesh sidecar, load balancer); and where each hop's logs live and which of them the user can query. Ask only what you need to draw the path. Do not hypothesise, and do not tell the user how to fetch anything, before the path is drawn: instructions written for an assumed environment are wrong in a way the user may not notice.

"I don't know" is not an answer to a path question. A hop the user cannot name is a fact: fetch it yourself (cluster resources, mesh config, DNS, the repo), or give the user the exact command that reveals it. A path with a blank in it is a tree with a branch missing. Only decisions accept "I don't know".

When a blank in the path is one you tried to fill and could not, ask the user for it in a round of its own. Do not put it beside evidence questions, and do not draw the tree around it: every evidence question asked against a tree with a missing branch may have to be asked again. One extra round is cheaper than a round that was asked against the wrong tree.

With the path known, turn the symptom into a **hypothesis tree**: 3–5 ranked, falsifiable explanations, one per hop that could plausibly have emitted the symptom, each stating what evidence would confirm or rule it out. Show the tree before asking for evidence.

Work the tree in **rounds**. The **frontier** is every piece of evidence you can ask for _now_: its prerequisites (which host, which time window, which request) are already known. Ask the whole frontier in one round; wait for the user's findings before the next round. Order questions cheapest-and-most-discriminating first: prefer evidence that rules out several hypotheses over evidence that supports one.

Every question needs a fact. Route each fact by who can reach it:

- **You can fetch it** (repo, git log, deploy history, docs, tools you have): dispatch a sub-agent; never ask the user. Don't block on it: only the questions downstream of the running lookup wait.
- **Only the witness can fetch it** (production logs, the client device, a dashboard, another team): ask, and give the exact command, query, or place to look, and what to paste back. Never "please provide more info".
- **It is a decision** (scope, priority, what "fixed" means): put it to the user and wait, as in `grilling`.

The first bucket is a claim you test, not a label you assign. If the lookup fails (no tool, no context, no permission, no network), say so in one line: what you tried, why it failed. Then move the fact to the second bucket with an exact command, and record in the ledger what access would have saved the round trip. Never fill the gap with a guess: a guessed fact in the ledger poisons every branch built on it. With no tools at all, every fact is in the second bucket and the skill still holds; it becomes a questionnaire the user runs by hand.

Format each question like so:

```
❓ **Q1** - **<what we need>**: <question body>

🔍 **How to get it**: <exact command / query / dashboard path, and what to paste back>

➡️ **What it decides**: <which hypotheses this confirms or rules out, and what you expect to see under each>
```

Write every question in Simplified Technical English (ASD-STE100), whatever language the user speaks: one idea per sentence, no sentence longer than 20 words, active voice, the imperative for instructions, one meaning per word. Keep technical terms in English whatever language the rest of the sentence is in: `load balancer`, `sidecar`, `namespace`, never a translation. The user may be on call, tired, or junior: a question they misread produces evidence from the wrong place.

Do not define terms inside the question; that buries the question. End every round with a **Terms** block instead: one line per technical term the user has not used yet, plus any hint that helps them answer (where such a thing usually lives, what it looks like). A reader who knows the terms skips the block; a reader who does not is not left guessing.

STE limits the form of a sentence, not the depth of the question. Every distinction that changes the tree stays: same cluster or another cluster, inside the mesh or outside it, your code or a vendor's. Give each distinction its own sentence, or its own option in a list the user picks from. Plain words with fewer distinctions is a shallower interview, not a simpler one.

Keep an **evidence ledger** and restate it every round:

```
## Established
- <fact> — <source, timestamp>
## Ruled out
- <hypothesis> — <by which evidence>
## Open
- <hypothesis> — <waiting on Qn>
```

Redact every secret and token in anything you quote; write `<REDACTED>`. Quote only the lines that carry signal.

Each round's evidence reshapes the tree: rule out branches, add hypotheses the evidence suggests, recompute the frontier. Do not reason past the evidence: a hypothesis with no supporting fact in the ledger stays open, not assumed. Do not stop when the frontier is empty. Stop when **one hypothesis survives and is confirmed by evidence the others cannot explain**, or when the investigator says the evidence is sufficient. If several hypotheses still stand and the cheap questions are used up, say so and hand off to `diagnosing-bugs` for a reproduction loop. A witness who stops answering does not end the session: the ledger stays as it is, and nothing is concluded from silence. Do not name a root cause until the witness confirms the ledger matches what they saw, and the investigator agrees.
