## What it does

`grill-reporter` runs `grill-evidence` against the reporter of a Jira ticket. You are the investigator; the reporter is the witness. Each round goes out as one Jira comment, and the evidence ledger lives in the ticket description, below a horizontal rule, with the reporter's original text above it untouched.

It is **stateful**: the ticket is the memory. A new session reads the description and comments, resumes from the ledger, and does not re-ask what the reporter already answered.

## When to reach for it

Type `/grill-reporter <ticket>`. The agent never reaches for it on its own, because it posts to a ticket other people read.

Use it when a bug report arrived raw and the person who can see the symptom is not you. If you can fetch the evidence yourself, use `grill-evidence` directly.

## Prerequisites

A Jira MCP that can read a ticket, post a comment, and update the description. The skill does not name a specific server. Check whether the MCP takes wiki markup or ADF for the description before the first run; the horizontal rule is written differently in each.

`grill-evidence` must be installed. This skill's `SKILL.md` delegates to it in its first line.

## The paper trail

| What | Where |
| --- | --- |
| The questions, per round | A Jira comment, starting with the AI disclaimer, ending with a Terms block |
| Established / Ruled out / Open / Waiting on reporter | The description, below the rule, rewritten in full each round |
| The reporter's original report | The description, above the rule, never edited |

Every post is shown to you first. Nothing goes to the ticket until you say so.

## It's working if

- The first comment asks about the reporter's environment and path, in language a non-engineer could act on.
- Every "how to get it" fits the reporter's access: a screenshot, a pasted log, one command.
- The description above the rule is byte-for-byte what the reporter wrote, every round.
- The ledger below the rule shrinks Open and grows Ruled out as comments come in.
- A reporter who goes quiet changes nothing: no conclusion, no nagging.
- The closing comment states confirmed facts; the root cause appears only if you said so.

## Common questions

**It edited the reporter's text.**
It must not. The skill reads the description back after every write and restores the part above the rule if it changed. If you see damage, check the ticket history and restore from there, then tell the agent which round broke it.

**The reporter answered in the wrong place.**
Reporters reply wherever they like. The skill reads all comments, not just replies to its own.

**The reporter stopped answering.**
The ticket stays where it is. Ask the agent for one reminder if you want one; it will not post more than that.

**Can I let it post without asking me?**
Not as written. The first comment a reporter reads decides whether they cooperate, and the description edit touches their words. Both are worth a look before they land.

## Where it fits

A front door on the `grill-evidence` primitive, in the same way `grill-me` sits on `grilling`. Upstream is `triage`, which decides a ticket needs more information; downstream, a ticket whose ledger has one surviving hypothesis is ready for `diagnosing-bugs` or `implement`.
