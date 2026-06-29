# In-house Slack AI bot — build plan

A plan for building **Prosper PM's own Slack AI agent** that connects into all of
our tools (Asana, Gmail, Google Calendar/Drive, QuickBooks, Ramp, Jotform,
Notion, Slack, Zapier) and acts on behalf of **each individual user**, using
OpenTag as the architectural starting point.

> **TL;DR.** OpenTag gives us ~80% of the skeleton for free: the Slack
> connection, the agent/runtime split, the MCP-based tool-wiring loop, and the
> human-in-the-loop approval gate. The part we build ourselves is the part
> OpenTag deliberately doesn't have — **authorization**. Our model is a simple
> **two tiers**: admins (Davis + Clark) can search everything; everyone else can
> search everything *except* Davis's and Clark's private communications (email,
> texts, etc.). The trick is enforcing that at the tool-provisioning layer — the
> protected tools are simply not loaded for standard users — so the LLM is never
> the security boundary. This plan is organized around getting that right.

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

## 2. The gap we must close: a two-tier authorization model

This is the heart of the project and the reason a plain OpenTag clone is not
enough.

**OpenTag is multi-user in *attribution* only, not *authorization*.** It knows
who is asking (passes their email as context) but it applies **no access
control** — whatever the bot's credentials can reach, any user can ask for
(`runtime.ts:77-98`). We need real authorization on top.

Our model is **two roles + a protected-resource set** — not per-employee OAuth:

### Roles
- **Admin** = **Davis + Clark.** Can search **everything**, including their own
  and each other's private communications.
- **Standard** = **everyone else.** Can search everything **except** Davis's and
  Clark's private communications.

### Protected resources (admin-only)
The *only* thing gated is **leadership's private communications**:
- Davis's and Clark's **email** (Gmail).
- Davis's and Clark's **texts / SMS** (whatever SMS source we connect).
- "…etc." — to be enumerated: e.g. their private Drive files, DMs, personal
  calendars. **We need a precise list** (see open decisions).

Everything else — shared company tools (Asana, Notion, QuickBooks, Ramp,
Jotform, Drive shared with the org, etc.) — is searchable by **all** users.

### How enforcement works (the critical design rule)
Authorization is enforced at the **tool-provisioning layer, never by asking the
LLM to behave.** Concretely, in `mcpTransports()` (`runtime.ts:75`):

- **Admin turn** → load all transports, including the protected mailboxes/SMS.
- **Standard turn** → the protected transports are **simply not loaded**. The
  model literally has no tool that can reach Davis's or Clark's mail, so it
  cannot leak it even under a jailbreak or prompt-injection attempt. (This reuses
  OpenTag's graceful-degradation pattern: a missing tool just isn't there.)

This is far more robust than a shared credential + "please don't show this to
non-admins" instruction, which an injected message could defeat. **The LLM is
never the security boundary; tool provisioning is.**

> Note: this is *coarser and simpler* than the per-user-OAuth design in earlier
> drafts. We do **not** need every one of ~N employees to connect their own
> Google account. We need: (1) the bot connected to Davis's and Clark's
> mailboxes/SMS, and (2) a role check that withholds those from standard users.

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
   │     2. role lookup → admin (Davis/Clark) or standard               │
   │     3. build MCP transports:                                        │
   │          • shared org tools          → ALWAYS loaded                │
   │          • leadership mail / SMS      → loaded ONLY if admin        │
   │     4. run LLM tool loop; gate every write through confirm_write    │
   └───────────────┬───────────────────────────────────────────────────┘
                   │
       ┌───────────┼─────────────────┬──────────────────────────────┐
       ▼           ▼                 ▼                              ▼
   Asana/Notion/  QuickBooks/Ramp/   Davis+Clark Gmail            Davis+Clark
   Drive (shared) Jotform (shared)   (admin-only)                 SMS (admin-only)
   ── all users ────────────────────┤ withheld from standard turns ┤
```

### New component: the authorization layer
The one piece OpenTag doesn't have. It's small — a role check, not a per-user
OAuth system:

- **Role map.** Static config: `{ admins: [<Davis Slack id>, <Clark Slack id>] }`.
  Everyone else is `standard`. (Start as a config file/env; can move to a table
  later if roles grow.)
- **Protected-resource registry.** A list of MCP transports tagged
  `adminOnly: true` — Davis's mailbox, Clark's mailbox, their SMS source, plus
  whatever else lands in the "etc." list.
- **Per-turn provisioning.** `mcpTransports(role)` becomes role-aware: it always
  includes the shared tools and includes `adminOnly` transports **only when
  `role === "admin"`.** A standard turn never receives those clients, so the
  model cannot surface that data no matter what the prompt says.
- **Credentials.** The bot connects to Davis's and Clark's Gmail/SMS via their
  own OAuth grant (each grants once) or Google Workspace domain-wide delegation.
  These are org-held bot credentials, not something every employee sets up.

This is the bulk of the net-new engineering, and it's deliberately simple.
Everything else is OpenTag glue.

---

## 4. Security & compliance (decide before coding)

Property management touches financials (Ramp/QuickBooks) and PII (Gmail/Drive),
so this is not optional:

- **Authorization is provisioning, not prompting.** The single most important
  rule: standard-user turns must *never have the protected tools loaded*. Do not
  rely on a system-prompt instruction like "don't show leadership's mail to
  non-admins" — a crafted message could override it. Enforce in code, in
  `mcpTransports(role)`.
- **Resolve role from the true requester.** Use the Slack user id of whoever
  invoked the bot — not the channel, not who's watching. Confirm the id can't be
  spoofed (Slack events are signed; verify signatures).
- **Least privilege per tool.** Narrowest OAuth scopes that work (e.g. Gmail
  `gmail.readonly` for search-only; add `gmail.send` only if we let it send).
- **Token encryption at rest** for the leadership mailbox/SMS credentials; never
  log tokens.
- **Confirm-gate every write**, no exceptions, for Ramp / QuickBooks / Gmail-send.
  Reuse `confirm_write` and show the exact action + amount/recipient in the card.
- **Audit log**: every tool call → who (Slack id + role), what tool, what args
  summary, approved-by, timestamp. Independent of the LLM transcript. This is also
  how we'd catch a standard user *attempting* to reach protected data.
- **Data residency**: charts render locally (good); confirm the LLM provider
  choice against our data policy. Model is a one-line env swap (`AGENT_MODEL`).
- **Prompt-injection posture**: thread/email/doc content is untrusted input.
  Two defenses cover the worst cases — (1) protected tools absent for standard
  users, and (2) human-in-the-loop on writes — so neither "leak leadership's
  mail" nor "send money to X" can fire without the right role + a human click.

---

## 5. Phased rollout

### Phase 0 — Foundation (skeleton, all-access)
- Fork OpenTag's `app/` + `runtime.ts` into a clean in-house repo.
- Strip Linear/Notion; rewrite the system prompt for our org.
- Stand up bot + runtime; confirm an @mention round-trips with the LLM.
- Wire **one shared-token tool** (Asana or Notion) to prove the MCP loop
  end-to-end. No role gating yet — everyone can search it.
- **Exit criteria:** "@bot what are my open Asana tasks" returns a rendered card.

### Phase 1 — The two-tier role gate (the core requirement)
- Add the **role map** (admins = Davis + Clark) and **protected-resource
  registry** (`adminOnly` transports).
- Make `mcpTransports(role)` role-aware: shared tools always; `adminOnly` tools
  only on admin turns. Forward the resolved role from bot → runtime in context.
- Connect Davis's + Clark's **Gmail** as the first protected resource.
- **Exit criteria:** an admin can search Davis's/Clark's mail; a standard user
  asking the same gets "I don't have access to that" because the tool was never
  loaded — verified to hold even against a deliberate prompt-injection attempt.

### Phase 2 — Breadth
- Add Davis's + Clark's **SMS/texts** as a protected resource (pick the SMS
  source — see open decisions).
- Enumerate and add the rest of the "etc." protected set (private Drive,
  calendars, DMs) if in scope.
- Add the remaining **shared** tools: QuickBooks, Ramp, Jotform, shared Drive,
  Calendar, Zapier — all searchable by everyone.
- Hard **confirm-gate every write** (Ramp/QuickBooks/Gmail-send) and turn on
  audit logging.

### Phase 3 — Hardening & rollout
- Persistence (Redis store, per OpenTag's `demo-restart` pattern) so approval
  clicks survive restarts.
- Rate limits / per-user quotas; error budgets.
- Roll out to a pilot group, then the org.

---

## 6. Open decisions (need your input)

1. **Confirm the role map**: admins are exactly Davis + Clark, correct? Anyone
   else (e.g. an ops/finance lead) who should be admin?
2. **Enumerate "…etc."**: beyond Davis's & Clark's email and texts, what else is
   admin-only? Private Drive files? Personal calendars? Slack DMs? Be explicit —
   anything not on the protected list is searchable by everyone.
3. **SMS/texts source**: what carries your texts? (e.g. Google Voice, a business
   SMS platform, iMessage export, Twilio.) This determines whether there's an MCP
   / API to connect at all, and is the trickiest integration.
4. **Standard-user data scope**: when a standard user "searches everything,"
   does that include *other employees'* mailboxes, or only shared company tools
   plus their own? (i.e. is anyone's mail searchable besides leadership's?)
5. **Hosting**: where do bot + runtime + role/token store run? Drives the OAuth
   callback URL and secrets management.
6. **LLM provider**: OpenAI, Anthropic, or Google — per our data policy.
7. **MCP sourcing**: official hosted MCP per vendor (lowest maintenance),
   self-hosted sidecars (OpenTag's Notion pattern), or Zapier's MCP (9,000+ apps,
   one integration). Likely a mix.
8. **Channel scope**: org-wide, or allowlisted channels + DMs only. (Note:
   role-gating must work in *public* channels too — if a standard user @mentions
   the bot in a channel where an admin is present, the requester's role is what
   counts, not who's watching.)

---

## 7. Effort estimate (rough)

| Phase | Scope | Rough effort |
|---|---|---|
| 0 | Skeleton + 1 shared-token tool | ~few days |
| 1 | Two-tier role gate + leadership Gmail | ~1 week |
| 2 | SMS + remaining tools + audit log | ~1–2 weeks |
| 3 | Hardening, persistence, rollout | ~1 week |

The role gate in Phase 1 is the core requirement but is genuinely small (a role
check + conditional transport loading). The larger unknown is the **SMS/texts**
source in Phase 2 — that integration may or may not exist off the shelf.
