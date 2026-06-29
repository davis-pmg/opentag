# In-house Slack AI bot — build plan

A plan for building **Prosper PM's own Slack AI agent** that connects into all of
our tools (Asana, Gmail, Google Calendar/Drive, QuickBooks, Ramp, Jotform,
Notion, Slack, Zapier) and acts on behalf of **each individual user**, using
OpenTag as the architectural starting point.

> **TL;DR.** OpenTag gives us ~80% of the skeleton for free: the Slack
> connection, the agent/runtime split, the MCP-based tool-wiring loop, and the
> human-in-the-loop approval gate. The ~20% we have to build ourselves is the
> part OpenTag deliberately doesn't have — a **per-user credential layer** so
> each person acts as themselves in Gmail, Drive, Ramp, and QuickBooks instead
> of through one shared service account. This plan is organized around closing
> that gap safely.

---

## 1. What we're keeping from OpenTag

OpenTag (MIT-licensed) is a thin reference bot built on CopilotKit's
`@copilotkit/bot` SDK. It runs as two processes:

```
Slack ──@mention──▶  bot (app/)  ──AG-UI/HTTP──▶  runtime (runtime.ts)
                                                    │  one LLM agent
                                                    ├── tool MCP  (bearer key)
                                                    └── tool MCP  (bearer key)
```

We keep these patterns essentially verbatim:

| Pattern | OpenTag source | Why we keep it |
|---|---|---|
| Bot ↔ runtime split over AG-UI | `app/index.ts` + `runtime.ts` | Swap models, scale the brain separately, reuse the agent for non-Slack surfaces later. |
| **MCP-as-tool-connector loop** | `runtime.ts:75-128` | Each integration is just a transport (`url` + auth header) in a list; failures are isolated (timeout race + `Promise.allSettled`) so one dead tool never kills a turn. This is exactly how we wire all our tools. |
| **Human-in-the-loop write gate** | `app/human-in-the-loop/confirm-write.tsx` | `awaitChoice()` blocks until Approve/Cancel before any write. Mandatory for Ramp (money), QuickBooks (books), Gmail (sending). |
| `read_thread` grounding | `app/tools/read-thread.ts` | Agent acts on the real conversation, not hallucinations. |
| Per-user **identity** context | `app/sender-context.ts` | Injects who is asking (name + email) into every turn. |
| Secret-gated multi-adapter bootstrap | `app/index.ts:74-180` | Set creds → that integration turns on. Clean config story. |
| Local rich rendering | `app/tools/render-*.tsx`, `app/render/` | Charts/tables/diagrams rendered locally (Playwright → PNG); data never leaves our infra. |
| Slash commands + modals | `app/commands/`, `app/modals/` | `/agent`, structured intake forms. |

What we drop: the Linear + Notion *integrations themselves* (we bring our own
tool list) and the triage-specific system prompt.

---

## 2. The gap we must close: per-user authorization

This is the heart of the project and the reason a plain OpenTag clone is not
enough.

**OpenTag is multi-user in *attribution* only, not *authorization*.** It knows
who is asking (passes their email as context) but every tool call uses **one
shared service-account credential** — a single `LINEAR_API_KEY`, a single
`NOTION_TOKEN` (`runtime.ts:77-98`). Its own prompt concedes this: *"issues are
still authored by the bot's API key… assignee is how you attribute work to the
requester"* (`runtime.ts:195-197`).

For our toolset that splits into two camps:

### Camp A — per-user identity is required (build the credential broker)
These tools expose data that is private *per person*, or take actions that must
be attributable to a real human. The bot must act **as the requesting user**,
with that user's own permissions:

- **Gmail** — reading/sending mail as the user. Never a shared mailbox.
- **Google Drive / Calendar** — a user sees only the docs/events they have access to.
- **Ramp** — spend, cards, reimbursements, approvals tied to the individual.
- **QuickBooks** — accounting actions must be attributable; scope by role.

### Camp B — a shared service account is acceptable
Org-wide resources where "the bot" acting as one integration identity is fine
(we still gate writes and log who triggered them):

- **Asana** — org workspace (use the requester's email to set assignee/attribution, OpenTag-style).
- **Notion** — shared workspace.
- **Jotform** — org forms.
- **Zapier** — org automation surface.

**Design implication:** `mcpTransports()` in `runtime.ts` must become
**dynamic per-turn** — instead of reading one static key from env, it looks up
the requesting Slack user's stored tokens and injects the right credential into
each MCP transport's `Authorization` header. Camp B tools keep using a static
org credential.

---

## 3. Target architecture

```
                         ┌─────────────────────────────────────────────┐
   Slack user @mentions  │                  bot (app/)                  │
   ───────────────────▶  │  - resolves Slack user id + email            │
                         │  - read_thread, render cards, HITL gate      │
                         │  - forwards Slack user id in AG-UI context   │
                         └───────────────┬─────────────────────────────┘
                                         │ AG-UI/HTTP  (+ Slack user id)
                                         ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │                          runtime (the brain)                        │
   │   per turn:                                                         │
   │     1. read Slack user id from context                              │
   │     2. credential broker → that user's OAuth tokens (Camp A)        │
   │     3. build MCP transports: per-user header (A) + org header (B)   │
   │     4. run LLM tool loop; gate every write through confirm_write    │
   └───────────────┬───────────────────────────────────────────────────┘
                   │
       ┌───────────┼───────────────┬───────────────┬──────────────┐
       ▼           ▼               ▼               ▼              ▼
   Gmail MCP   Drive/Cal MCP   Ramp MCP      QuickBooks MCP   Asana/Notion/…
   (per-user)  (per-user)      (per-user)    (per-user)       (org token)
```

### New component: the credential broker
The one piece OpenTag doesn't have. Responsibilities:

- **Store** per-user OAuth tokens, encrypted at rest (e.g. Postgres + envelope
  encryption, or a secrets manager). Key: Slack user id (+ tool).
- **Refresh** expired access tokens using stored refresh tokens.
- **Link flow**: first time a user invokes a Camp-A tool with no token on file,
  the bot replies with an ephemeral "Connect your Google account" / "Connect
  Ramp" OAuth link; the callback stores tokens against their Slack id.
- **Inject**: on each turn, hand the runtime the right `Authorization` header
  per Camp-A transport. Missing/expired token → that tool is simply absent for
  that turn (OpenTag's graceful-degradation pattern already handles a missing
  tool cleanly).

This is the bulk of the net-new engineering. Everything else is OpenTag glue.

---

## 4. Security & compliance (decide before coding)

Property management touches financials (Ramp/QuickBooks) and PII (Gmail/Drive),
so this is not optional:

- **Least privilege per tool.** Request the narrowest OAuth scopes that work
  (e.g. Gmail `gmail.send` + `gmail.readonly` rather than full mail).
- **Token encryption at rest** and short-lived access tokens; never log tokens.
- **Confirm-gate every write**, no exceptions, for Ramp / QuickBooks / Gmail-send.
  Reuse `confirm_write` and show the exact action + amount/recipient in the card.
- **Audit log**: every tool call → who (Slack id), what tool, what args summary,
  approved-by, timestamp. Independent of the LLM transcript.
- **Channel scoping**: decide whether the bot answers in any channel or only in
  approved ones; DMs by default carry the requester's own auth.
- **Data residency**: charts render locally (good); confirm the LLM provider
  choice (OpenAI vs Anthropic vs Google) against our data-handling policy. Model
  is a one-line env swap (`AGENT_MODEL`).
- **Prompt-injection posture**: thread/email/doc content is untrusted input.
  Keep write-gating human-in-the-loop so an injected "send money to X" can't
  execute without a person clicking Approve.

---

## 5. Phased rollout

### Phase 0 — Foundation (skeleton, no per-user auth yet)
- Fork OpenTag's `app/` + `runtime.ts` into a clean in-house repo.
- Strip Linear/Notion; rewrite the system prompt for our org.
- Stand up bot + runtime; confirm an @mention round-trips with the LLM.
- Wire **one Camp-B tool with a shared org token** (Asana or Notion) to prove
  the MCP loop end-to-end.
- **Exit criteria:** "@bot what are my open Asana tasks" returns a rendered card.

### Phase 1 — Credential broker + first per-user tool
- Build the token store + OAuth link flow + refresh.
- Make `mcpTransports()` per-turn dynamic.
- Wire **Gmail** as the first Camp-A tool (high value, clear per-user boundary).
- Confirm two different Slack users see *their own* mail.
- **Exit criteria:** two users, two mailboxes, correct isolation; missing-token
  users get a "connect your account" prompt.

### Phase 2 — Breadth
- Add Google Calendar + Drive (same Google OAuth, more scopes).
- Add Ramp and QuickBooks behind hard confirm-gates.
- Add remaining Camp-B tools (Jotform, Zapier).
- Audit logging live for every write.

### Phase 3 — Hardening & rollout
- Persistence (Redis store, per OpenTag's `demo-restart` pattern) so approval
  clicks survive restarts.
- Rate limits / per-user quotas; error budgets.
- Roll out to a pilot group, then the org.

---

## 6. Open decisions (need your input)

1. **Hosting**: where do bot + runtime + token DB run? (Our cloud / Railway /
   etc.) Drives the OAuth callback URL and secrets management.
2. **LLM provider**: OpenAI, Anthropic, or Google — per our data policy.
3. **Tool priority order** for Phase 2 (which of Ramp/QuickBooks/Calendar/Drive
   first).
4. **MCP sourcing**: use each vendor's official hosted MCP where it exists
   (lowest maintenance), self-host sidecars (OpenTag's Notion pattern), or route
   some via Zapier's MCP (9,000+ apps, one integration). Likely a mix.
5. **Channel scope**: org-wide, or allowlisted channels + DMs only.

---

## 7. Effort estimate (rough)

| Phase | Scope | Rough effort |
|---|---|---|
| 0 | Skeleton + 1 shared-token tool | ~few days |
| 1 | Credential broker + Gmail per-user | ~1–2 weeks (the hard part) |
| 2 | Remaining tools + audit log | ~1–2 weeks |
| 3 | Hardening, persistence, rollout | ~1 week |

The credential broker in Phase 1 is the critical path and the main risk; the
rest is largely OpenTag glue plus per-tool OAuth wiring.
