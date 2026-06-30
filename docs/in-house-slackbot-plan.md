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

### Roles (confirmed)
- **Admin** = **Davis + Clark only.** No one else. Can search **everything**,
  including their own and each other's private communications.
- **Standard** = **everyone else.** Can search everything **except** Davis's and
  Clark's private communications.

### Protected resources (admin-only) — confirmed list
- Davis's and Clark's **email** (Gmail).
- Davis's and Clark's **texts** (sent/received via **RingCentral API**).
- Davis's and Clark's **private/shared Google Drives**.

### Standard-user reach (confirmed) — and the catch
Standard users can search **broadly, including other employees' mailboxes** —
not just shared company tools and their own mail. The denylist is just two
people: Davis and Clark.

**This makes the boundary leaky, and it's the most important thing to resolve
before building.** Email is two-sided: a message Clark sends to an employee
lives in *both* Clark's mailbox *and* that employee's mailbox. If standard users
can search that employee's mailbox, they can read Clark's message through it —
"don't load Clark's mailbox" does **not** hide it. The only correspondence truly
hidden is mail purely between Davis and Clark (or with parties whose mailboxes
aren't searchable). See [§2a](#2a-the-mailbox-boundary-problem) — we need a call
here.

### How enforcement works (the design rule)
Authorization is enforced at the **tool-provisioning / tool-wrapper layer, never
by asking the LLM to behave.** Two mechanisms, depending on the resource:

- **Whole-resource gating** (Drives, SMS): the protected transport is **simply
  not loaded** on a standard turn. The model has no tool that can reach Davis's
  or Clark's Drive/texts, so it can't leak them even under prompt injection. This
  reuses OpenTag's graceful-degradation pattern (a missing tool just isn't there)
  and is airtight.
- **Mailbox filtering** (Gmail): because standard users *can* search other
  mailboxes, we can't just withhold Gmail. We need a **wrapper tool in front of
  Gmail** that injects the requester's role and, for standard users, hard-scopes
  the query to exclude Davis's and Clark's mailboxes. This is real net-new code
  and is only a *partial* fix because of the two-sided-email catch above.

**The LLM is never the security boundary; the transport/wrapper layer is.**

---

## 2a. The mailbox-boundary problem (needs a decision)

Goal: standard users **cannot read Davis's or Clark's email.** Reality: standard
users **can** search other employees' mailboxes. These two collide, because
every email exists in at least two mailboxes (sender + each recipient).

So if Clark emails an employee, that message sits in the employee's mailbox too.
A standard user searching that employee's mail finds it — even though we never
loaded Clark's mailbox. **Excluding leadership's mailboxes hides almost nothing
in practice.** Three ways to resolve it, cleanest first:

- **Option A — narrow standard reach (recommended).** Standard users get shared
  company tools + their **own** mailbox only, not all-staff mail. Then "hide
  Davis/Clark's mail" is a clean, airtight tool-gate. (This walks back part of
  answer #4 — flagging it because it's the only option that actually delivers the
  stated goal cheaply.)
- **Option B — message-level redaction.** Keep all-staff search, but the Gmail
  wrapper drops any message where Davis or Clark is a participant (from/to/cc),
  in *any* mailbox. Achievable but more code, slower searches, and easy to get
  subtly wrong (forwards, quoted replies, alias addresses).
- **Option C — accept the leak.** Keep all-staff search, only withhold the two
  mailboxes directly. Cheapest to build, but understand that leadership's
  correspondence with any employee remains discoverable. Probably **not** what
  "they can't read our emails" is meant to achieve.

There's also a **broader privacy question** worth a conscious decision: letting
all staff search each other's mailboxes (any option that keeps all-staff search)
is a significant surveillance capability with possible legal/HR implications.
Recommend confirming this is intended regardless of which option we pick.

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
   │     3. provision tools by role:                                     │
   │          • shared org tools          → ALWAYS loaded                │
   │          • leadership Drives / SMS    → loaded ONLY if admin         │
   │          • Gmail wrapper              → role passed in; standard     │
   │                                          turns exclude D&C mailboxes │
   │     4. run LLM tool loop; gate every write through confirm_write    │
   └───────────────┬───────────────────────────────────────────────────┘
                   │
       ┌───────────┼───────────────┬───────────────┬──────────────────┐
       ▼           ▼               ▼               ▼                  ▼
   Asana/Notion/  QuickBooks/     Gmail wrapper   Davis+Clark        Davis+Clark
   shared Drive   Ramp/Jotform    (role-scoped)   private Drives     SMS/RingCentral
   ── all users ──────────────┤   see §2a        (admin-only)       (admin-only)
                                  ── withheld from standard turns ──┤
```

### New component: the authorization layer
The one piece OpenTag doesn't have. Mostly a role check; the Gmail wrapper is the
one non-trivial part (see §2a):

- **Role map.** Static config: `{ admins: [<Davis Slack id>, <Clark Slack id>] }`.
  Everyone else is `standard`. (Start as a config file/env; move to a table if it
  ever grows — confirmed it won't for now.)
- **Protected-resource registry.** Transports tagged `adminOnly: true` — Davis's
  & Clark's **private/shared Drives** and their **RingCentral SMS**. These are
  whole-resource gated (not loaded for standard turns) → airtight.
- **Gmail wrapper.** Not whole-resource gated, because standard users *can* search
  other mailboxes. A wrapper tool receives the role and, for standard users,
  scopes the query to exclude Davis's & Clark's mailboxes — a partial control;
  see the §2a boundary problem and pick an option there.
- **Per-turn provisioning.** `provisionTools(role)` includes shared tools always,
  `adminOnly` transports only when `role === "admin"`, and passes role into the
  Gmail wrapper.
- **Credentials.** Likely **Google Workspace domain-wide delegation** for a single
  bot service account that can read any company mailbox/Drive (needed since
  standard users search all-staff mail), plus a **RingCentral API** app
  credential for SMS. Org-held bot credentials, not per-employee setup.

This is the bulk of the net-new engineering. Everything else is OpenTag glue.

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

### Phase 1 — The role gate, on whole-resource tools first (the core requirement)
- Add the **role map** (admins = Davis + Clark) and **protected-resource
  registry** (`adminOnly` transports). Forward resolved role bot → runtime.
- Make `provisionTools(role)` role-aware: shared tools always; `adminOnly`
  transports only on admin turns.
- Connect Davis's + Clark's **private/shared Drives** as the first protected
  resource — clean whole-resource gating, no boundary problem.
- **Exit criteria:** an admin can search D&C's private Drives; a standard user
  asking gets "I don't have access" because the tool was never loaded — verified
  to hold against a deliberate prompt-injection attempt.

### Phase 2 — Gmail (resolve §2a first), SMS, and breadth
- **Decide the §2a mailbox boundary** (Option A/B/C). Then connect Gmail via
  Workspace domain-wide delegation and build the role-scoped Gmail wrapper per
  that decision.
- Add Davis's + Clark's **texts via the RingCentral API** as an admin-only
  resource. No off-the-shelf RingCentral MCP is assumed — plan a thin wrapper
  tool around their REST API (auth, message search). This is the integration with
  the most unknowns.
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

## 6. Decisions

### Resolved
- ✅ **Role map**: admins are **Davis + Clark only**. No one else.
- ✅ **Protected (admin-only) set**: Davis's & Clark's email, their texts, and
  their private/shared Google Drives.
- ✅ **SMS source**: **RingCentral API** (no off-the-shelf MCP assumed — thin
  wrapper to build).
- ✅ **Standard-user reach**: broad, **includes other employees' mailboxes**
  (which is what surfaces the §2a boundary problem).

### Still open (need your input)
1. **§2a mailbox boundary — the big one.** Pick A (standard users get shared
   tools + own mailbox only — clean & cheap, recommended), B (all-staff search
   with message-level redaction of anything involving D&C), or C (accept that
   leadership's mail-with-employees stays discoverable). This decides whether the
   stated goal is actually achievable and how much Gmail code we write.
2. **All-staff mailbox search — intended?** Confirm letting all employees search
   each other's mail is a deliberate choice (privacy/HR/legal implications).
3. **Hosting**: where do bot + runtime + role store run? Drives the OAuth
   callback URL, the domain-wide-delegation setup, and secrets management.
4. **LLM provider**: OpenAI, Anthropic, or Google — per our data policy.
5. **MCP sourcing**: official hosted MCP per vendor (lowest maintenance),
   self-hosted sidecars (OpenTag's Notion pattern), or Zapier's MCP (9,000+ apps,
   one integration). Likely a mix; RingCentral is a custom wrapper regardless.
6. **Channel scope**: org-wide, or allowlisted channels + DMs only. (Role-gating
   keys off the *requester's* id, so it's correct even in a public channel an
   admin is watching.)

---

## 7. Effort estimate (rough)

| Phase | Scope | Rough effort |
|---|---|---|
| 0 | Skeleton + 1 shared-token tool | ~few days |
| 1 | Role gate + leadership Drives (whole-resource gating) | ~1 week |
| 2 | Gmail wrapper (§2a) + RingCentral SMS + remaining tools + audit log | ~2–3 weeks |
| 3 | Hardening, persistence, rollout | ~1 week |

The role gate itself is small (a role check + conditional transport loading) and
lands cleanly in Phase 1 on the Drives. The real cost moved to Phase 2: the
**Gmail wrapper** (size depends entirely on the §2a decision — Option A is
trivial, Option B is the most code) and the **RingCentral SMS** wrapper, which
has the most unknowns since there's no off-the-shelf MCP.
