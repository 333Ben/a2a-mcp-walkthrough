# A2A where it earns it. MCP everywhere else.

An interactive walkthrough of **eight exchanges** in a multi-agent system, showing the
protocol each one actually needs — and the two places where only A2A will do.

**→ [Open the walkthrough](https://333ben.github.io/a2a-mcp-walkthrough/)**

> 🇫🇷 La page est bilingue et suit la langue du navigateur. Un sélecteur FR/EN est en haut à droite.

---

## The argument

Most writing about the Agent2Agent protocol argues for adopting it. This argues for
**bounding** it.

The system below reads the documents involved in a French flat sale, cross-checks them
against each other, and flags what is missing or contradictory. It runs on eight
exchanges. **Five are plain MCP tool calls** — ask, answer, done. **Two need A2A.** One is
an email, because there is a person at the other end.

| # | Exchange | Protocol | Why this one |
|---|---|---|---|
| 1 | Karl → file | `MCP` | Three seconds, one answer, nothing to wait for. Karl is a tool, not an agent — no goal, no memory |
| 2 | Extract → file | `MCP` | A long call, but one that comes back. It can report progress; what it cannot do is stop and ask a question |
| 3 | file → Coherence | `MCP` | A data cross-check: no counterpart, no waiting, no decision |
| 4 | Coherence → Iris → human | **`A2A`** | **This is where MCP breaks.** See below |
| 5 | human → Iris | **`A2A`** | The answer lands on the *same task*, open since earlier |
| 6 | Iris → Arsène → third party | **`A2A`** + email | Arsène is going to wait for days. An MCP call cannot stay open that long |
| 7 | third party → Arsène → Extract | email + `MCP` | An ordinary tool call — carrying a task id that is three days old |
| 8 | Extract → file → Coherence | `MCP` + `A2A` | Back to normal. The last A2A message is the one that closes the task |

## Where MCP breaks

At step 4 the file contradicts itself: the listing says 71 m², the surveyor's certificate
says 68.4 m². And a document that should be there is missing.

A tool call answers or it fails. It cannot say *"I'm stopping, I need you, come back to
me."* The A2A task moves to `input-required` and **stays alive** while the human thinks.
That is the whole reason A2A is in this system.

## The moment worth the page

Step 7. The third party replies with the missing PDF three days later. Arsène pushes it
into the pipeline through the same door as the first twelve documents:

```jsonc
{
  "method": "tools/call",
  "params": {
    "name": "extraction_documents",
    "arguments": { "files": ["pv-ag-2023.pdf"] },
    "_meta": {
      "a2a/taskId": "task-9d02",     // open for 3 days
      "a2a/contextId": "ctx-4e77"
    }
  }
}
```

Same tool name, same arguments as step 2. Pure MCP. What changed is what it carries: a
task id that survived three days, a round trip to a third party, and a human decision.

**MCP carries. A2A remembers.** Each does exactly its job, and nothing is replayed.

## Two rules written into the engine

```jsonc
"resolution": "human"     // no agent picks the source of truth
"authorised_by": "user"   // without this field, Arsène does not send
```

Neither is a technical limit. When two documents disagree, the professional carries the
liability — so the arbitration is theirs, and it is recorded in the task with who decided
and when. And no message leaves the system on an agent's own initiative.

## Status — what this is, and what it is not

**This is a design study, not a deployed system.** The agents are real and the roles are
real; today they coordinate through a shared Supabase database. This page shows what the
same choreography would look like split between MCP and A2A, and argues where the split
should fall.

The file, the parties, the addresses and the correspondence are fictional. Every domain
used is under `.example`, reserved by [RFC 2606](https://www.rfc-editor.org/rfc/rfc2606)
for exactly this.

## Context

**Loi Alur** is the French statute requiring a seller to hand over a defined set of
co-ownership documents before a flat can be sold. The documents disagree with each other
more often than not — which is what makes the cross-check, and the arbitration, the
interesting part of the problem.

The two protocols:
[Model Context Protocol](https://modelcontextprotocol.io) ·
[Agent2Agent](https://a2a-protocol.org)

## Running it locally

```bash
open index.html
```

One file, no build, no dependencies, no network calls. Keyboard: `←` `→` to step,
`space` to play or pause. Honours `prefers-reduced-motion` and the system colour scheme.

---

© 2026 Garance Husson
