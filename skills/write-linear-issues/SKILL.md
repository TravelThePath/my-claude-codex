---
name: write-linear-issues
description: Use when drafting a Linear engineering issue (feature, refactor, schema change, migration, infra task) at "In Preparation" — anything where Context, Changes, and Out of scope must express intent without freezing implementation. Also trigger when rewriting an issue whose sections have blurred into code-level detail.
---

# Linear Issue — Context / Changes / Out of scope

## Overview

An issue at **In Preparation** is an *intent contract*, not a merge contract. It tells reviewers *why*, implementers *what changes*, and PM/triage *what's not in*. Implementation details (SQL, proto signatures, code) belong in the PR — they freeze the issue prematurely if they show up here.

## When to use

- Drafting a new engineering issue: feature, refactor, schema change, migration, or infra task.
- Rewriting an issue whose sections have blurred (motivation in Changes, implementation in Changes, scope without destinations).

### When NOT to use

| Issue type | Use instead |
| --- | --- |
| Bug | Bug template — repro / current / expected / environment. |
| Incident retro | Postmortem template — timeline / impact / root cause / actions. |
| Spike or research | Research template — question / approach / timebox / deliverable. |
| Product / feature scoping (pre-engineering) | A scoping doc, not an issue. |

## The three sections

| Section | Reader | Question |
| --- | --- | --- |
| Context | reviewer / future maintainer | Why does this exist? Who is blocked? Why now? |
| Changes | implementer / reviewer | What is changing? (At "what" level — name fields, endpoints, pages.) |
| Out of scope | PM / triage | What is NOT in this issue, and where does it live instead? |

A sentence that doesn't answer its section's question for its section's reader belongs somewhere else — or nowhere.

## The what / how boundary

Changes names *what changes*. It does not specify *how to code it*.

| Domain | ✓ Goes in Changes | ✗ Stays out (lands in PR) |
| --- | --- | --- |
| DB schema | New / modified field name and meaning | SQL, column types, index names, ALTER statements |
| gRPC / GraphQL | New / modified proto / GraphQL field name and purpose | proto signatures, field numbers, struct or resolver code |
| UI | New / modified pages, components, actions | component code, props shape, CSS |
| Behavior | New runtime semantics, validations, authorization edges | the specific function or algorithm implementing them |

## Process

1. **Classify.** Confirm the request fits the supported types. Bug / retro / spike / scoping → hand off.
2. **Ground Changes.** Read the codebase to confirm the entity / endpoint / page names you're about to use. Do NOT research motivation — that comes from the user.
3. **Draft into the template.** Use only the sub-headings that apply; drop the rest. Mark every unknown inline as `[CONFIRM: <question>]` — never silently guess.
4. **Resolve `[CONFIRM]` items, one focused question at a time.** Do NOT probe Out of scope unprompted; soliciting scope questions invites scope creep. Capture only what the user volunteers.
5. **Estimate story point.** Apply the framework in [`story-point-estimation.md`](./story-point-estimation.md). Suggest a value from `0 / 1 / 2 / 3 / 5 / 8` with one sentence of rationale; let the user confirm or adjust.
6. **Deliver.** Render the markdown. If the user asks to create it, call Linear MCP `save_issue` with **`state=In Preparation`** by default and the agreed story point.

## Template

```markdown
## Context

<2–4 sentences. Reader: reviewer / future maintainer. Question: why this exists,
who is blocked, why now. No field names, endpoint names, table names.>

## Changes

<Reader: implementer / reviewer. Name what changes; do not specify how it's coded.
Use only the sub-headings that apply; drop the rest.>

### Schema

- <"Add Y field on entity X, meaning Z." No SQL, no column types, no index names.>

### API

- <"Add / modify field Y on RpcA / QueryB, used for Z." No proto signature, no resolver code.>

### UI

- <"Add / modify action Y on page X." No component code, no props shape.>

### Behavior

- <"X behavior now Y." No specific function or algorithm.>

## Out of scope

- <Item> — tracked in <issue / milestone / follow-up>.
- <Item> — not planned (<reason>).
```

Drop Changes sub-headings that don't apply. **Do not write `N/A` or "No changes."** — delete the heading. An empty heading invites the implementer to invent work or skip the section.

## Pre-delivery checklist

- [ ] Context contains no field names, endpoint names, or table names.
- [ ] Changes contains no SQL, no proto signatures, no struct definitions, no implementation code.
- [ ] Every Changes sub-heading present has content; unused sub-headings are deleted.
- [ ] Each Out of scope item names a destination (`tracked in <issue / milestone / follow-up>`) or carries `not planned (<reason>)`.
- [ ] No `[CONFIRM:` remains in the draft.
- [ ] Story point assigned (`0 / 1 / 2 / 3 / 5 / 8`) with a one-sentence rationale.
- [ ] Issue is created with `state=In Preparation`.

## Worked example — Add archive flag to accounts

### ✗ Bad — Context names columns, Changes specifies SQL

```markdown
## Context

Add an `is_archived BOOLEAN NOT NULL DEFAULT FALSE` column to `accounts` so
admin list can filter archived rows. Index on the column to keep filters cheap.

## Changes

### Schema
- `ALTER TABLE accounts ADD COLUMN is_archived BOOLEAN NOT NULL DEFAULT FALSE;`
- `CREATE INDEX accounts_is_archived_idx ON accounts(is_archived);`

### API
- Add `bool include_archived = 5` to `ListAccountsRequest`.
- Resolver appends `WHERE is_archived = false` when flag absent.

## Out of scope
- Bulk archive UI later.
```

What went wrong:

- **Context** is a snapshot of the implementation (`is_archived`, `BOOLEAN`, `DEFAULT FALSE`, index name) — a reviewer next year doesn't need any of it.
- **Changes** specifies SQL and a proto field number — it crosses into *how*, locking implementation choices the PR should be free to refine.
- **Out of scope** "later" has no destination — it rots into tribal knowledge.
- No story point, no status discipline.

### ✓ Good — intent only, what without how

```markdown
## Context

Operations cannot remove dormant accounts from their working list, which
currently shows ~3k entries and is unscannable. We need a soft-archive
mechanism so the existing admin list can hide archived accounts without
deleting history.

## Changes

### Schema
- Add an archive flag on the `accounts` entity. Archived accounts are hidden
  from default queries but kept in the database.

### API
- `ListAccounts` accepts an opt-in flag to include archived accounts. Default
  behavior excludes them; existing callers see no change.

### Behavior
- Archived accounts no longer appear in the admin list by default.
- Sort order and pagination semantics unchanged when the flag is absent.

## Out of scope
- Bulk archive UI — tracked in ENG-1234.
- Auto-archive policy (inactivity threshold) — tracked in PROD-456.
- Unarchive flow — tracked in ENG-1234 follow-up.
```

**Story point: 2** — single-entity schema addition plus one filter parameter on a list endpoint; well-trodden path, low risk.

**Status on create: `In Preparation`.**

Why this works:

- **Context** explains *who is blocked* (operations) and *why now* (3k entries, unscannable). No column names, no types, no index names.
- **Changes** names the archive flag and the opt-in filter — implementers know exactly where to look — but no SQL, no proto field number, no default-value choice. Those land in the PR.
- **Out of scope** items each have a destination.
- Story point follows the framework in `story-point-estimation.md` with a one-line rationale.
- Issue is created with `state=In Preparation`.
