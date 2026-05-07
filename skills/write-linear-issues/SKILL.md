---
name: write-linear-issues
description: Use when drafting a Linear engineering issue (a.k.a. issue) for a feature, refactor, schema change, migration, or infra task — anything where Context, Spec, Out of scope, and Acceptance Criteria need to hold up to review. Also trigger when rewriting or grading an existing issue whose sections blur together.
---

# Linear Issue — Context / Spec / Out of scope / Acceptance Criteria

## Overview

An issue is a contract across time and roles. **Each section answers exactly one question for exactly one reader.** Mixing readers and questions is the single failure mode this skill exists to prevent — every other rule below descends from that.

If a sentence doesn't serve its section's reader and answer its section's question, it belongs somewhere else — or nowhere.

## When to use

- Drafting a new engineering issue: feature, refactor, schema change, migration, or infra task.
- Rewriting an existing issue where sections have blurred (motivation in Spec, design in Context, scope in AC).
- Grading an issue draft another agent or human produced before posting it.

### When NOT to use

| Issue type | Use instead |
| --- | --- |
| Bug | Bug template — repro / current / expected / environment. |
| Incident retro | Postmortem template — timeline / impact / root cause / actions. |
| Spike or research | Research template — question / approach / timebox / deliverable. |
| Product / feature scoping (pre-engineering) | A scoping doc, not an issue. |

## The four sections

| Section | Question it answers | Primary reader |
| --- | --- | --- |
| Context | Why does this exist? Who is blocked? Why now? | Reviewer / future maintainer |
| Spec | What is being built? | Implementer |
| Out of scope | What is NOT in this issue, and where does it live instead? | PM / triage |
| Acceptance Criteria | How does a reviewer verify it's done? | Reviewer / QA |

The readers are not interchangeable:

- A reviewer reading **Context** next year does not need column names.
- An implementer reading **Spec** today does not need motivation.
- A PM reading **Out of scope** needs a destination, not a justification.
- QA reading **Acceptance Criteria** needs a binary outcome, not a restatement of intent.

## Process

1. **Classify.** Confirm the request fits the supported types above. If it's a bug, retro, spike, or scoping doc — stop and hand off.
2. **Ground the Spec only.** Read the codebase to confirm schema, RPC shape, neighboring migrations — anything the implementer needs. Do NOT research motivation; that comes from the user.
3. **Draft into the template.** Use only the sub-headings that apply. Mark every unknown inline as `[CONFIRM: <question>]` — never silently guess.
4. **Resolve `[CONFIRM]` items, one focused question at a time.** Do NOT probe Out of scope unprompted; soliciting scope questions invites scope creep. If the user volunteers scope risk, capture it.
5. **Walk the Quality checklist.** (See below.)
6. **Output.** Render the final markdown. If the user asks, create the issue via the Linear MCP `create-issue` tool with the rendered body.

## Template

```markdown
## Context

<2–4 sentences. Reader: reviewer / future maintainer. Question: why this exists, who is blocked, why now. No column names, no RPC names, no design decisions.>

## Spec

<Reader: implementer. Question: what to build. Use only the sub-headings that apply; omit the rest entirely.>

### Schema

<New tables, columns, edges, indexes.>

### API

<New or changed endpoints, RPCs, GraphQL fields, request / response shapes.>

### Migration

<Backfill steps, dual-write phase, deploy order, rollback plan.>

### Behavior

<New runtime behavior, validations, side effects, error handling.>

## Out of scope

<Reader: PM / triage. Each item names where the work lives instead — an issue, milestone, or follow-up.>

- <Item> — tracked in <issue / milestone / follow-up>.

## Acceptance Criteria

<Reader: reviewer / QA. Each item is a reviewer action plus a binary observable outcome. Not a restatement of Spec.>

- [ ] <Reviewer action> → <binary observable outcome>.
```

### When to use which Spec sub-heading

Drop sub-headings that don't apply. Empty headings invite the implementer to invent work or ignore the section entirely.

| Sub-heading | Include when |
| --- | --- |
| Schema | Touches table structure: new tables / columns / edges / indexes, type changes, soft-delete columns, FK additions. |
| API | Changes the wire contract: new or changed gRPC RPC, GraphQL field / mutation / query, REST endpoint, event payload. |
| Migration | Needs a data migration, dual-write / dual-read phase, backfill, or careful deploy order. Pure additive schema with no backfill: skip Migration. |
| Behavior | Changes runtime semantics: validation, authorization, side effects, error mapping, rate limits, logging contracts. |

## Worked example — Add `is_archived` to accounts

### ✗ Bad (sections blurred)

```markdown
## Context

Add an `is_archived` boolean column to the `accounts` table so we can filter archived accounts out of the admin list. We'll default it to false and the UI team will update the list query to filter on it. Bulk archive UI to come later.

## Acceptance Criteria

- [ ] `is_archived` column exists.
- [ ] Code is reviewed and merged.
```

What went wrong:

- **Context** names a column (`is_archived`), states a design choice (`DEFAULT false`), and references a future feature (bulk archive UI). It is doing the job of Spec, Out of scope, and motivation simultaneously.
- **AC** restates Spec ("column exists") and contains a non-binary, non-observable item ("code is reviewed").
- **No Out of scope section** despite mentioning future work inline.

### ✓ Good

```markdown
## Context

Operations cannot remove dormant accounts from their working list, which currently shows ~3k entries and is unscannable. We need a soft-archive bit so the existing admin list can hide archived accounts without deleting history.

## Spec

### Schema

- Add `is_archived BOOLEAN NOT NULL DEFAULT FALSE` to `accounts`.
- Add index `accounts_is_archived_idx (is_archived)` to keep list filters cheap.

### API

- Extend `ListAccountsRequest` with `include_archived bool` (default `false`).
- `ListAccountsResponse.accounts` excludes archived rows when `include_archived = false`.

### Behavior

- Existing callers continue to see only non-archived accounts.
- Setting `include_archived = true` returns the union; sort order unchanged.

## Out of scope

- Bulk archive UI — tracked in ENG-1234.
- Auto-archive policy (inactivity threshold) — tracked in PROD-456.
- Unarchive flow — tracked in ENG-1234 follow-up.

## Acceptance Criteria

- [ ] Run migration on staging → `DESCRIBE accounts;` shows `is_archived` and the new index.
- [ ] Call `ListAccounts` with default request on a tenant containing 1 archived + 1 non-archived account → response contains 1 account (the non-archived one).
- [ ] Call `ListAccounts` with `include_archived = true` on the same tenant → response contains 2 accounts.
- [ ] Existing admin-list integration test on the consumer side passes unchanged.
```

Why this works:

- **Context** explains *who is blocked* (operations) and *why now* (3k entries, unscannable). No column names, no defaults, no future features.
- **Spec** is implementer-ready: exact column type, exact request field, exact behavior delta.
- **Out of scope** items each name a destination issue — nothing rots into "TODO".
- **AC** items are reviewer actions producing binary outcomes — none of them restate Spec, none mention "code reviewed", none use Spec verbs ("Add", "Implement").

## Quality checklist

Walk this before handing off the draft.

- **Each section answers only its own question.** A sentence in Context that names a column, RPC, or component belongs in Spec. A sentence in Spec that says "because users…" or "to unblock…" belongs in Context.
- **Context has no design decisions.** Defaults, types, indexes, and rollout choices live in Spec.
- **Spec has no motivation.** No "because", no "to unblock", no "users want".
- **Spec sub-headings are pruned.** Drop sections that don't apply rather than littering with "N/A".
- **Out of scope items name a destination OR an explicit non-plan with reason.** Either `— tracked in <issue / milestone / follow-up>.` ("Bulk archive UI — see ENG-1234.") or `— not planned (<reason>).` ("Streamlit decommissioning — not planned (kept indefinitely as manual fallback).") Naked items with no follow-up rot into tribal lore.
- **Out of scope is a top-level section,** never nested under Context or Spec.
- **AC items are verifications, not restatements.**
  - ✗ "Adds `is_archived` column to `accounts`." (restates Spec)
  - ✗ "Code is reviewed and merged." (not observable)
  - ✗ "Ensure archived accounts are hidden." (not binary)
  - ✓ "Run `SELECT is_archived FROM accounts LIMIT 1;` on staging → returns without error." (action + binary outcome)
- **AC contains no `[CONFIRM]`.** Verification depends on a settled contract. If an AC is contingent on an unresolved decision, the issue isn't ready — hoist the question to Spec, resolve it, then write the AC.
- **Unknowns are marked, not guessed.** Every assumption not in the user's input is either confirmed in chat or left as `[CONFIRM: …]` in the draft.

## Common mistakes

| Mistake | Why it fails | Fix |
| --- | --- | --- |
| Putting motivation in Spec ("we need this because…") | Implementer skims for what to build; "because" sentences hide steps. | Move to Context. |
| Putting design choices in Context (`BOOLEAN DEFAULT FALSE`, index names) | Reviewer next year cannot tell *why*; the column reads as the goal. | Move to Spec → Schema. |
| Out of scope item without a destination or non-plan note | PM / triage cannot tell if the item is a future issue, an open question, or an explicit non-decision. | Append `— tracked in <issue / milestone / follow-up>` or `— not planned (<reason>)`; otherwise delete it. |
| AC that restates Spec | A reviewer cannot tell "done" from "started"; nothing is verified. | Rewrite each AC as `<reviewer action> → <binary observable outcome>`. |
| AC like "code is reviewed" or "tests pass" | Always true at merge time; verifies nothing specific. | Replace with the specific behavior the change adds. |
| `[CONFIRM:` inside an AC item | Reviewer cannot tell pass / fail when verification depends on an unresolved decision. | Resolve the decision in Spec first; AC follows the settled contract. |
| Guessing instead of `[CONFIRM]` | Quietly fabricated assumptions ship as the spec. | Mark every unknown inline; resolve in chat. |
| Asking the user about Out of scope unprompted | Soliciting scope worry invites scope creep. | Capture only scope items the user volunteers, plus obvious adjacent work the user named. |
| Embedding Out of scope under Context or Spec | PM / triage skim top-level structure; nested scope is invisible. | Promote to top-level `## Out of scope`. |
| Filling every Spec sub-heading because the template offers them | Empty sub-headings invite invented work. | Drop sub-headings that don't apply. |
| Writing `No <X> changes.` or `N/A` under a sub-heading instead of dropping it | The implementer reads the heading and reasons backward — "Schema is listed, but they say no change... do I need to verify nothing changes?". Defensive non-content is worse than no heading. | Delete the sub-heading entirely. |

## Red flags — stop and fix

If you catch yourself doing any of these, stop and rewrite the offending section:

- Writing a sentence in **Context** that contains a backticked symbol or a quoted column / RPC name.
- Writing a sentence in **Spec** that begins with "Because", "To", "Since", or "Users want".
- Writing an **AC** item that begins with "Add", "Implement", "Create", "Update" — those are Spec verbs, not verification verbs.
- Writing `[CONFIRM:` inside an **AC** item — AC is the verification contract; an unresolved decision means the contract isn't ready. Resolve the question in Spec first, then AC follows.
- AC items mapping 1-to-1 to Spec items (one AC per Spec line saying "X is added / created") — that's a Spec checklist, not a verification. Raw count is fine when each AC is a distinct observable.
- Filling every Spec sub-heading because the template offers them.
- Writing `No <X> changes.` or `N/A` under a Spec sub-heading instead of dropping the heading — the empty heading is the same anti-pattern, just with a defensive comment. If the section doesn't apply, delete it.
- Asking the user "is anything else out of scope?" — you are inviting scope creep.
- Resolving `[CONFIRM]` items by guessing rather than asking.

## Rationalizations to refuse

| Excuse | Reality |
| --- | --- |
| "It's all related, splitting feels artificial." | The issue is read by 4 different readers across months. They don't have your full context. The split is the point. |
| "AC restating Spec is clearer." | Restated AC verifies nothing; reviewers approve from feel rather than evidence. |
| "I'll skip `[CONFIRM]` and use a sensible default." | Sensible defaults become the spec by accident, then ship as the contract. Mark and ask. |
| "Out of scope without a destination is fine for now." | "For now" persists; the line becomes tribal lore that nobody owns. Always name a destination. |
| "More sub-headings make the issue look complete." | Empty sub-headings invite the implementer to invent work or skip the section entirely. |
| "The user is busy, I'll fill in the motivation myself." | You are now the source of record for *why* — and you don't have the business context. Leave it as `[CONFIRM]`. |
| "I'll put `[CONFIRM]` in the AC since the underlying decision isn't made yet." | The AC is the merge contract. If the contract has unresolved questions, the issue isn't ready. Hoist the question to Spec, settle it, then write the AC. |
