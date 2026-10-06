# Design: Blossom walkthrough workflow on Hybro

> Status: Draft  
> Date: 2026-10-05 (rev: agents Hybro-agnostic)  
> Audience: Blossom team (design partner)  
> Companion: [a2a-mcp-walkthrough README](./README.md) · Hybro doc: `hybro/backend/docs/design-headless-agent-network.md`  
> Assumption: Blossom **self-hosts** Hybro; end users use **BlossomDoc UI** (not Hybro chat)

---

## 1. Purpose

Complete the real-estate multi-agent workflow described in the A2A/MCP walkthrough:

- **MCP (or HTTP tools)** for one-shot work (Karl, Extract, coherence).  
- **A2A** where a task must stay alive (human arbitration, multi-day email).  
- **Email** where a person is on the other end (Arsène ↔ third parties).  
- **Hybro** as the embedded interop/client layer — **invisible to domain agents**.

**Ideal:** Iris, Arsène, and any future Blossom agent are **plain A2A-compatible servers**. They do not import Hybro SDKs, call Hybro REST, or require `hybro_session_id`. Only **BlossomDoc (product)** and **ops** know Hybro exists.

---

## 2. Goals and non-goals

### Goals

1. Ship the eight-exchange choreography end-to-end with a real dossier.  
2. One Hybro **session per dossier** (product/Hybro concern).  
3. Human decisions (`resolution: human`, `authorised_by: user`) in **Blossom UI**.  
4. Iris and Arsène as **standard A2A agents**; Karl / Extract / analyze-* / coherence as **tools**.  
5. Keep `agent-gateway` as the only door into Supabase.  
6. App and mail remain **one engine, two doors**.  
7. **Agents stay Hybro-agnostic** — A2A + domain tools only.

### Non-goals

- Building a full A2A interop stack inside Blossom (Hybro owns that).  
- Registering Karl/Extract as A2A peers.  
- Putting Loi Alur types into Hybro schemas.  
- Requiring estate agents to use Hybro chat.  
- Teaching Iris/Arsène about Hybro APIs, sessions, or tokens.

---

## 3. Who knows what

```text
Knows Hybro                         Must NOT know Hybro
──────────────────────              ─────────────────────────────
BlossomDoc UI / BFF                 Iris (A2A server + tools)
Deploy/ops (register Card URLs)     Arsène (A2A server + email + tools)
Hybro instance                      Karl, Extract, analyze-*, coherence
```

| Layer | Speaks |
|---|---|
| BlossomDoc ↔ Hybro | Hybro headless REST (sessions, HITL, events) |
| Hybro ↔ Iris / Arsène | **A2A only** (Hybro is the A2A **client**) |
| Iris / Arsène ↔ tools | MCP or HTTP (`agent-gateway`, Extract, Karl, …) |
| Arsène ↔ humans/third parties | Email (Resend) |
| Arsène → client async updates | A2A **push** to the callback URL the **client** registered (happens to be Hybro ingress) |

Agents may know **domain peers** (optional): e.g. Iris configured with Arsène’s Agent Card URL and calling Arsène over A2A. That is still not “knowing Hybro.”

---

## 4. Target topology

```text
Blossom VPC
├── BlossomDoc UI ──REST──► Hybro (self-hosted) ──A2A──► Iris
│                              │                    └──A2A──► Arsène
│                              └── HITL/events back to UI
├── agent-gateway ←── tools used by Iris/Arsène/UI
├── Tools: karl-agent, blossom-extract-agent, analyze-*, detect-coherence
└── A2A agents: Iris, Arsène (publish Agent Cards; no Hybro client code)
```

| Component | Face | Knows Hybro? |
|---|---|---|
| BlossomDoc + EFs | Product + data | **Yes** (embedder) |
| Hybro | Interop client + HITL | — |
| Iris | A2A server | **No** |
| Arsène | A2A server + email | **No** |
| Extract / Karl | Tools | **No** |

---

## 5. Walkthrough → ownership matrix

| # | Exchange | Protocol | Who talks |
|---|---|---|---|
| 1 | Karl → file | Tool | UI or Iris tools → Karl (not A2A) |
| 2 | Extract → file | Tool | UI or agent tools → Extract / `process-document` |
| 3 | File → coherence | Tool | UI or agent tools → `detect-coherence` |
| 4 | Coherence → Iris → human | A2A + HITL | Hybro client → Iris `input-required`; UI answers via Hybro |
| 5 | Human → Iris | HITL → A2A continue | UI → Hybro → A2A Message to Iris (same Iris `taskId`) |
| 6 | Need mail → Arsène → third party | A2A + email | **Hybro client → Arsène** (default), or Iris→Arsène A2A (Iris knows Arsène Card, not Hybro) |
| 7 | Third party → Arsène → Extract | Email + tool + A2A push | Arsène updates **its** Task; push to client callback; tools for PDF |
| 8 | Close | Tool + A2A complete | Tools; Hybro/client drives Task completion / new Tasks |

Invariant:

- Discrepancy / HITL choices match dashboard (`resolution: human`).  
- Outbound mail requires `authorised_by: user` (domain rule on Arsène).  
- **MCP carries; A2A remembers** — tool `_meta` may carry **A2A** `taskId`/`contextId` and **domain** `dossier_id`, never a required `hybro_session_id`.

---

## 6. Orchestration patterns (agents stay Hybro-agnostic)

### Default — **P1: Hybro/product as sole A2A client (star)**

BlossomDoc uses Hybro to message **Iris** and **Arsène** separately.

1. UI/tools run extract + coherence.  
2. Hybro → Iris: “review dossier {dossier_id}”.  
3. Iris → `input-required` (conflict / missing doc / ask auth to mail).  
4. UI HITL via Hybro; Hybro continues Iris.  
5. Iris returns a **structured result/artifact** (e.g. “email syndic for PV, authorised”).  
6. **BlossomDoc/Hybro** (not Iris code) starts Arsène Task with that payload.  
7. Arsène sends mail; later email webhook advances **Arsène’s** Task; A2A push notifies Hybro.  
8. Hybro/UI continues or starts a follow-up Iris Task (`referenceTaskIds` if needed).

Iris never dials Arsène or Hybro. Wiring lives in the **product**.

### Optional — **P2: Iris orchestrates Arsène over A2A**

Iris is configured with Arsène’s Agent Card URL (domain config). Iris is A2A **client** to Arsène. Hybro only speaks to Iris. Arsène push goes to whatever callback **Iris** registered—or Iris polls Arsène. Hybro still does not appear in agent code; Iris knows a **peer agent**, not Hybro.

Prefer **P1** for clear HITL embedding and Session visibility of both Tasks.

---

## 7. Shared contracts

### 7.0 A2A identity

| Layer | Identity | Owner |
|---|---|---|
| Product case | `dossier_id` + Hybro `session_id` | BlossomDoc + Hybro only |
| Iris | A2A `contextId` + `taskId` | Iris server |
| Arsène | A2A `contextId` + `taskId` | Arsène server |
| Tools | Optional `_meta` with domain + A2A ids | Not Hybro ids |

Do not put `hybro_session_id` in agent prompts, tool required fields, or Agent Card extensions.

### 7.1 Session metadata (Hybro — embedder only)

```json
{
  "dossier_id": "uuid",
  "blossom_user_id": "uuid",
  "domain": "blossom"
}
```

BlossomDoc stores `dossiers.hybro_session_id`. Agents receive `dossier_id` in A2A message parts/metadata if they need domain context.

### 7.2 Thread ↔ Arsène task

| Blossom / Arsène | Meaning |
|---|---|
| `dossiers.external_ref` / Message-ID | Domain idempotency |
| Arsène internal map Message-ID → **its** `taskId` | Agent-local; not Hybro |
| A2A push to client callback | Notifies Hybro without naming it |

### 7.3 HITL

Blossom UI → Hybro HITL APIs → Hybro sends A2A continuation to Iris.  
Agents only see a normal A2A user Message resuming `input-required`.

### 7.4 Auth

| Hop | Auth |
|---|---|
| UI/BFF → Hybro | User or embedder service token |
| Hybro → Iris/Arsène | Per Agent Card security schemes |
| Iris/Arsène → agent-gateway | Existing `X-Agent-Secret` + `x-user-id` |
| Arsène → Resend | Resend keys |

**Iris/Arsène do not get Hybro API tokens.**

---

## 8. Runtime flows

### 8.0 Bootstrap

1. Deploy Hybro (self-host).  
2. Deploy Iris & Arsène as A2A servers (`/.well-known/agent-card.json`).  
3. **Ops/BlossomDoc** registers Card URLs in Hybro directory (agents don’t “register themselves into Hybro” as a product API).  
4. Point agent tool configs at gateway / Extract / Karl (domain).  
5. BlossomDoc Hybro client (session, HITL, events).

### 8.1 Open dossier → Hybro session (product only)

```http
POST /api/v1/network/sessions
{ "metadata": { "dossier_id", "blossom_user_id" }, "agents": ["iris"] }
```

### 8.2 Steps 1–3 — tools

UI and/or agents’ tools; no Hybro required for the PDF pipeline itself.

Then **BlossomDoc → Hybro → Iris** (A2A message with `dossier_id`).

### 8.3 Steps 4–5 — HITL

Iris Task → `input-required`. UI uses Hybro HITL; Hybro continues Iris via A2A.

### 8.4 Step 6 — Arsène (P1)

BlossomDoc/Hybro sends **new** A2A message to Arsène with domain payload + `authorised_by`.  
Arsène sends email; keeps **its** Task non-terminal. Push updates to Hybro ingress URL from push config.

### 8.5 Step 7 — email

Resend → Arsène (domain webhook). Arsène:

1. Maps thread → **its** `taskId`.  
2. Runs Extract / `process_document` tools.  
3. Updates Task status/artifacts; **A2A push** (or await GetTask from client).  

No `POST` to Hybro Network APIs from Arsène.

### 8.6 Step 8 — close

Product/Hybro drives coherence tools as needed and completes or refines Iris/Arsène Tasks per A2A terminal rules.

### 8.7 Mail-only door

Inbound mail without a prior Arsène Task: Arsène may still run domain pipeline via gateway. To involve Iris, **product/Hybro** starts or continues an Iris Task—Arsène does not call Hybro. Optional: Arsène exposes an A2A skill “ingest inbound email” that Hybro invokes when the product detects a new mail thread tied to a dossier.

---

## 9. Per-repo work

### 9.1 `flat-file-finder-v3` (knows Hybro)

| Work | Notes |
|---|---|
| `hybro_session_id` on dossiers | Product only |
| Hybro client | Session, message/dispatch, HITL, events |
| HITL UI | Embed Hybro pending/respond |
| Sequence P1 | After Iris auth-to-mail artifact → Hybro message to Arsène |
| Docs | Agents are A2A; product owns Hybro |

### 9.2 Iris (Hybro-agnostic)

| Work | Notes |
|---|---|
| A2A server + Agent Card | Standard card; streaming/push as needed |
| Tools | Karl, process_document, coherence, gateway |
| Policies | Human resolution; emit structured “mail request” artifacts |
| **No** Hybro client, session ids, or Hybro auth |

### 9.3 Arsène (Hybro-agnostic)

| Work | Notes |
|---|---|
| A2A server + Agent Card | Accept tasks; advertise push |
| Email + tools | Resend, process_document, signals |
| Map Message-ID → own `taskId` | Agent-local store |
| On inbound PDF | Tools + Task update + **A2A push** to client callback |
| **No** Hybro REST; refuse mail without `authorised_by` in task message |

### 9.4 Extract / Karl

Unchanged as tools; no Hybro; optional A2A ids in `_meta` for logging only.

---

## 10. Phased delivery

### Phase A
- A2A stubs for Iris/Arsène; Hybro registers Card URLs.  
- BlossomDoc session + one Iris round-trip via Hybro.

### Phase B
- Iris `input-required` + BlossomDoc HITL via Hybro.

### Phase C (P1)
- Hybro → Arsène after HITL; email; push back; tools on inbound.

### Phase D
- Full walkthrough + mail door (product starts Iris when needed).

### Phase E
- Harden; optional P2 (Iris→Arsène A2A) if desired.

---

## 11. Testing plan

| Case | Expect |
|---|---|
| Iris binary has no Hybro URL/token in config | Pass |
| Arsène never calls `/api/v1/network/*` | Pass |
| HITL round-trip | UI → Hybro → Iris A2A continue |
| Multi-day mail | Arsène Task + push; Hybro Session shows update |
| Replace Hybro with another A2A client in a lab | Agents still function |

---

## 12. Dependencies on Hybro (product only)

BlossomDoc needs Hybro headless: durable dispatch, GetTask/cancel/list, HITL, events, push ingress, embedder auth, metadata query.

Agents need only: A2A compliance + domain tools.

---

## 13. Success criteria

1. Walkthrough runnable with self-hosted Hybro.  
2. Estate agent never opens Hybro chat.  
3. App and mail share domain truth.  
4. Karl/Extract not on A2A directory.  
5. **Iris and Arsène run against any compliant A2A client** (Hybro is one embedder).  
6. Distinct A2A tasks per agent; Session correlates in Hybro/product only.  
7. No agent dependency on `hybro_session_id` or Hybro REST.

---

## 14. Summary

BlossomDoc embeds Hybro; **agents only speak A2A (+ tools + email)**. Hybro is the A2A client, directory, HITL bridge, and Session correlator. That keeps Blossom agents portable and keeps Hybro a true interop layer—not a proprietary agent SDK.

---

## 15. Design review vs “agents must not know Hybro”

| Earlier draft smell | Correction |
|---|---|
| Arsène `POST /network/tasks/...` | Arsène updates own Task + A2A push |
| “Register with Hybro” as agent duty | Ops/product registers Card URL |
| Hybro service tokens on Iris/Arsène | Only embedder holds Hybro tokens; Card auth inbound |
| `hybro_session_id` in agent/tool meta | Use `dossier_id` + A2A ids |
| Iris “via Hybro” as if Iris linked Hybro | P1: product sequences Hybro→Iris and Hybro→Arsène |
| Mail door “Arsène asks Iris via Hybro” | Product/Hybro starts Iris; or Arsène A2A skill invoked by client |
