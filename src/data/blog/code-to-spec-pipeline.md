---
author: Andrew Axelrod
pubDatetime: 2026-03-19T00:00:00Z
title: "From Code to Spec: A Pipeline for Turning a Large Codebase into OpenSpec"
slug: code-to-spec-pipeline
featured: true
draft: false
tags:
  - openspec
  - documentation
  - software-architecture
  - developer-tools
  - api-design
description: A three-stage pipeline for systematically converting a large codebase with no existing specs into a fully populated OpenSpec baseline — without losing anything and at any scale.
---

Most teams have the same problem. The codebase has been running for two years. There are dozens of services, hundreds of endpoints, and a business logic layer that only two people fully understand. There are no specs. There's a Notion wiki from 2022 that's mostly wrong. There's a README that describes how to run the app but not what it does.

Now you want to use OpenSpec — a spec-driven workflow where `openspec/specs/` is the source of truth for how the system behaves. But OpenSpec assumes you already have a baseline. It's a change management tool, not a migration tool. It's designed for "here's what we're changing", not "here's everything we know about the system."

This article describes a three-stage pipeline for going from a large codebase with no existing specs to a fully populated OpenSpec baseline — systematically, without losing anything, and in a way that scales to codebases of any size.

---

## Why You Can't Go Direct

The tempting approach is to point Claude at your codebase and say "write me an OpenSpec spec for this." It will produce something. But three problems emerge immediately:

**The coverage problem.** A large codebase has dozens of domains. One prompt, one spec. You'll get the top-level behavior of the most obvious domain and miss everything else.

**The format problem.** OpenSpec specs use Given/When/Then scenarios, RFC 2119 keywords (MUST/SHALL/SHOULD), and a specific structure for data models and design rationale. Code doesn't look like that. Converting code directly to OpenSpec format in a single step produces specs that are structurally correct but behaviorally thin — they describe the happy path and miss invariants, edge cases, and the "why" behind decisions.

**The coordination problem.** OpenSpec is one change at a time. A large codebase has 20, 30, 50 domains. You need a way to route each domain to the right spec file, track what's been done, and ensure nothing gets dropped.

The solution is an **intermediate layer** — a format-neutral spec that sits between your code and OpenSpec. Both the code reading and the format conversion become smaller, more focused problems.

---

## The Pipeline

![Codebase to OpenSpec pipeline](/codebase_pipeline_diagram_fixed.svg)

Each phase has a defined input, a defined output, and a defined scope. You can run them in sequence, pause between phases, or hand off between team members. The manifest keeps track of where you are.

---

## Phase 0: Discovery

**Goal:** Understand what domains exist in the codebase before writing a single word of spec.

Discovery is a structural pass only. You are not reading for behavior — you are reading for shape. What modules exist? What are their rough responsibilities? What depends on what?

### What to do

1. Walk the top-level directory structure and identify candidate domains by directory, module, or layer
2. For each candidate domain, identify the 3–5 most important files (the ones where the real logic lives — not utils, not types)
3. Build a rough dependency graph: which domains call which
4. Assign a short name and a prefix to each domain

### Example

Say you have a Node.js API with this structure:

```
src/
├── auth/
│   ├── auth.service.ts
│   ├── auth.controller.ts
│   ├── jwt.strategy.ts
│   └── session.service.ts
├── users/
│   ├── users.service.ts
│   ├── users.controller.ts
│   └── users.repository.ts
├── orders/
│   ├── orders.service.ts
│   ├── orders.controller.ts
│   ├── fulfillment.service.ts
│   └── orders.repository.ts
├── notifications/
│   ├── notifications.service.ts
│   └── email.adapter.ts
└── shared/
    ├── database/
    ├── guards/
    └── middleware/
```

Your discovery output (`domain-map.md`) might look like this:

```markdown
# Domain Map

| ID     | Domain        | Directory       | Key Files                                      | Depends On        |
|--------|---------------|-----------------|------------------------------------------------|-------------------|
| AUTH   | Authentication | src/auth/       | auth.service.ts, jwt.strategy.ts, session.service.ts | SHARED            |
| USER   | User Management| src/users/      | users.service.ts, users.repository.ts         | AUTH, SHARED      |
| ORDER  | Order Processing| src/orders/    | orders.service.ts, fulfillment.service.ts     | USER, NOTIF, SHARED |
| NOTIF  | Notifications  | src/notifications/ | notifications.service.ts, email.adapter.ts | SHARED            |
| SHARED | Platform       | src/shared/     | database/, guards/, middleware/               | —                 |

## Phase Order
1. SHARED (no dependencies)
2. AUTH (depends on SHARED only)
3. USER, NOTIF (depend on AUTH + SHARED)
4. ORDER (depends on USER + NOTIF + SHARED)
```

The phase order is important. You write foundation specs before feature specs, because feature specs reference foundation specs. Getting this wrong means you'll be mid-spec on Orders and discover you need to reference an Auth concept that doesn't have a spec yet.

---

## Phase 1: The Manifest

**Goal:** Build the routing table that maps every domain to its destination spec file and tracks progress.

The manifest is your coordination document for the entire conversion. It tells you what exists, where it goes, and what's been done. It's based on the same structure as a conversion manifest — a master inventory and routing table with tracking IDs.

### What to do

For each domain from your discovery output:
1. Assign a spec destination path (`openspec/specs/<domain>/spec.md`)
2. Assign a tracking ID prefix (matches your domain prefix from discovery)
3. Note the phase (from your dependency order)
4. Add a status column to track progress

### Example

```markdown
# Codebase → OpenSpec Conversion Manifest

> Generated: 2026-03-19
> Source: src/
> Target: openspec/specs/

## Routing Table

| Phase | Domain | Prefix | Source Directory | Destination Spec | Status | Validated |
|-------|--------|--------|-----------------|-----------------|--------|-----------|
| 1 | Platform / Shared | SHARED | src/shared/ | openspec/specs/platform/spec.md | todo | [ ] |
| 2 | Authentication | AUTH | src/auth/ | openspec/specs/auth/spec.md | todo | [ ] |
| 3 | User Management | USER | src/users/ | openspec/specs/users/spec.md | todo | [ ] |
| 3 | Notifications | NOTIF | src/notifications/ | openspec/specs/notifications/spec.md | todo | [ ] |
| 4 | Order Processing | ORDER | src/orders/ | openspec/specs/orders/spec.md | todo | [ ] |

## Tracking ID Ranges

| Prefix | Domain | Sub-ID Pattern |
|--------|--------|----------------|
| SHARED | Platform | SHARED-001 to SHARED-NNN |
| AUTH | Authentication | AUTH-001 to AUTH-NNN |
| USER | User Management | USER-001 to USER-NNN |
| NOTIF | Notifications | NOTIF-001 to NOTIF-NNN |
| ORDER | Order Processing | ORDER-001 to ORDER-NNN |

## Post-Conversion Checklist
- [ ] All tracking IDs appear in exactly one output spec
- [ ] No cross-domain behavior described in the wrong spec
- [ ] All invariants preserved verbatim
- [ ] All edge cases documented
- [ ] Phase order respected (no forward references to unwritten specs)
```

Update the status column as you complete each domain. This is your source of truth for where you are in the conversion.

---

## Phase 2: Spec Synthesis

**Goal:** For each domain, read the code and write an intermediate spec in a format-neutral, behavior-first structure.

This is the hardest phase. You're translating implementation into behavior — taking code that describes *how* and writing specs that describe *what* and *why*. The intermediate format is designed to make this translation as systematic as possible.

### The intermediate format

```markdown
# Domain: [Name]  <!-- PREFIX-001 -->

**Source:** [directory]
**Phase:** [1/2/3/4]
**Depends on:** [list of domain names]

---

## Overview  <!-- PREFIX-001.01 -->

[2–3 sentences describing what this domain does and why it exists]

---

## Behaviors  <!-- PREFIX-001.02 -->

### [Behavior name]  <!-- PREFIX-001.02.01 -->

**Context:** [What must be true for this behavior to apply]
**Trigger:** [What initiates this behavior]
**Outcome:** [What happens as a result]
**Invariants:**
- [Constraint that always holds — things the system guarantees]
**Edge cases:**
- [What happens when X is missing / invalid / expired]

### [Another behavior]  <!-- PREFIX-001.02.02 -->
...

---

## Entities  <!-- PREFIX-001.03 -->

### [Entity name]  <!-- PREFIX-001.03.01 -->

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| [field] | [type] | yes/no | [constraint or description] |

**Relationships:**
- [Entity] has many [OtherEntity]
- [Entity] belongs to [OtherEntity]

---

## Decisions  <!-- PREFIX-001.04 -->

### [Decision name]  <!-- PREFIX-001.04.01 -->

**Decision:** [What was chosen]
**Rationale:** [Why — derived from code patterns, comments, or structure]
**Alternatives considered:** [What else could have been done, if apparent from code]

---

## Cross-Domain References  <!-- PREFIX-001.05 -->

- **[Domain]:** [How this domain uses or depends on it]

---

## Open Questions  <!-- PREFIX-001.06 -->

- [Anything unclear from reading the code that needs a human answer]
```

### Example: synthesizing the Auth domain

After reading `auth.service.ts`, `jwt.strategy.ts`, and `session.service.ts`, you'd write:

```markdown
# Domain: Authentication  <!-- AUTH-001 -->

**Source:** src/auth/
**Phase:** 2
**Depends on:** Platform (SHARED)

---

## Overview  <!-- AUTH-001.01 -->

Authentication manages how users prove their identity and how the system tracks active sessions.
It issues JWT access tokens and refresh tokens, validates credentials against stored hashes,
and enforces session limits per user.

---

## Behaviors  <!-- AUTH-001.02 -->

### Login  <!-- AUTH-001.02.01 -->

**Context:** User has a registered account with a verified email
**Trigger:** POST /auth/login with { email, password }
**Outcome:** Access token (15min TTL) and refresh token (7d TTL) returned; session record created
**Invariants:**
- Passwords are never stored in plaintext
- Failed attempts are counted; account locked after 5 consecutive failures
- Refresh tokens are single-use; each use rotates to a new token
**Edge cases:**
- Invalid credentials → 401, increment failure count, do not reveal which field is wrong
- Account locked → 403 with lock expiry time in response
- Unverified email → 403 with message to check email

### Token Refresh  <!-- AUTH-001.02.02 -->

**Context:** User holds a valid, unused refresh token
**Trigger:** POST /auth/refresh with { refreshToken }
**Outcome:** New access token and new refresh token issued; old refresh token invalidated
**Invariants:**
- Old refresh token must be invalidated atomically with new token issuance
- If a previously used refresh token is presented, all sessions for that user are revoked (token reuse detection)
**Edge cases:**
- Expired refresh token → 401, user must log in again
- Already-used refresh token → 401 + full session revocation

### Logout  <!-- AUTH-001.02.03 -->

**Context:** User has an active session
**Trigger:** POST /auth/logout with valid access token
**Outcome:** Current session's refresh token invalidated; access token remains valid until TTL expiry
**Invariants:**
- Access tokens cannot be revoked (stateless); rely on short TTL
- Logout only invalidates current session, not all sessions (use /auth/logout-all for that)

---

## Entities  <!-- AUTH-001.03 -->

### Session  <!-- AUTH-001.03.01 -->

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| id | uuid | yes | Primary key |
| userId | uuid | yes | FK → User |
| refreshToken | string (hashed) | yes | Stored as bcrypt hash, never plaintext |
| expiresAt | timestamp | yes | 7 days from issuance |
| createdAt | timestamp | yes | |
| revokedAt | timestamp | no | Null if active |

**Relationships:**
- Session belongs to User (many sessions per user, up to platform limit)

---

## Decisions  <!-- AUTH-001.04 -->

### JWT over session cookies  <!-- AUTH-001.04.01 -->

**Decision:** Stateless JWT access tokens with stateful refresh tokens
**Rationale:** Mobile clients need bearer token auth; cookies are browser-only. Access tokens are short-lived to limit exposure; refresh tokens are stateful so they can be revoked.
**Alternatives considered:** Full session cookies (rejected: mobile incompatible), fully stateless refresh (rejected: no revocation capability)

### Refresh token rotation  <!-- AUTH-001.04.02 -->

**Decision:** Every refresh rotates the refresh token
**Rationale:** Limits the window of a stolen token — if an attacker uses a token, the legitimate user's next refresh will trigger reuse detection and revoke all sessions.

---

## Cross-Domain References  <!-- AUTH-001.05 -->

- **Platform (SHARED):** Uses database connection pool; uses request logging middleware

---

## Open Questions  <!-- AUTH-001.06 -->

- What is the maximum number of concurrent sessions allowed per user? (Not found in code — may be unlimited)
- Is there a "remember me" behavior that extends refresh token TTL? (Not implemented but may be expected)
```

### Tips for reading code effectively

**Read services before controllers.** Controllers are routing and validation. Services are behavior. Start with the service file — that's where the invariants and decisions live.

**Look for guard clauses first.** The `if (!x) throw` patterns at the top of functions are your invariants and edge cases. They're the most important content in your spec and the easiest to miss when summarizing.

**Follow the data, not the call stack.** Trace what happens to the main entity (the user, the order, the session) from input to persistence. That sequence is your behavior description.

**Read the repository/data layer for entities.** The database schema is in migrations or ORM model definitions. That's where you get field types, constraints, and relationships — not from the service layer.

**Comments and TODOs are decisions.** A `// can't use X because Y` comment is a documented decision. A `// TODO: add rate limiting` is an open question. Capture both.

---

## Phase 3: Converting to OpenSpec

**Goal:** Transform each intermediate spec into `openspec/specs/<domain>/spec.md` using OpenSpec's format conventions.

This is a mechanical transformation. The content is already in the intermediate spec — you're restructuring it into OpenSpec's conventions: Given/When/Then scenarios, RFC 2119 keywords, and the accumulated source-of-truth format.

### Transformation rules

| Intermediate | OpenSpec |
|---|---|
| `## Behaviors / [Behavior]` | `## Scenarios` with Given/When/Then |
| `Invariants` | `## Invariants` with MUST/SHALL statements |
| `Edge cases` | Additional Given/When/Then scenarios per edge case |
| `## Entities` | `## Data Model` |
| `## Decisions` | `## Design Rationale` |
| `## Open Questions` | `## Open Questions` (unchanged) |
| Tracking IDs `<!-- AUTH-001 -->` | Preserved as HTML comments |

### Example: intermediate → OpenSpec

Taking the Auth Login behavior from Phase 2:

```markdown
# Authentication

> **Domain:** auth
> **Status:** baseline
> **Last updated:** 2026-03-19
> **Source:** Synthesized from src/auth/ — baseline conversion

---

## Overview  <!-- AUTH-001.01 -->

Authentication manages how users prove their identity and how the system tracks active sessions.
It issues JWT access tokens and refresh tokens, validates credentials against stored hashes,
and enforces session limits per user.

---

## Scenarios  <!-- AUTH-001.02 -->

### Login  <!-- AUTH-001.02.01 -->

**Given** a user has a registered account with a verified email,
**When** they POST to `/auth/login` with valid `{ email, password }`,
**Then** the system MUST return an access token (15-minute TTL) and a refresh token (7-day TTL),
AND the system MUST create a session record linked to the user.

**Given** a user provides incorrect credentials,
**When** they POST to `/auth/login`,
**Then** the system MUST return HTTP 401,
AND MUST NOT reveal which field (email or password) was incorrect,
AND MUST increment the failure count for that account.

**Given** an account has had 5 consecutive failed login attempts,
**When** any login attempt is made for that account,
**Then** the system MUST return HTTP 403 with the lock expiry time in the response body.

**Given** a user's email address has not been verified,
**When** they attempt to log in,
**Then** the system MUST return HTTP 403 with a message directing the user to check their email.

### Token Refresh  <!-- AUTH-001.02.02 -->

**Given** a user holds a valid, unused refresh token,
**When** they POST to `/auth/refresh` with that token,
**Then** the system MUST issue a new access token and a new refresh token,
AND MUST atomically invalidate the old refresh token in the same operation.

**Given** a previously used refresh token is presented,
**When** a refresh is attempted,
**Then** the system MUST return HTTP 401,
AND MUST revoke all active sessions for the associated user (token reuse detection).

---

## Invariants  <!-- AUTH-001.03 -->

- The system MUST NOT store passwords in plaintext. <!-- AUTH-001.03.01 -->
- Refresh tokens MUST be single-use; each use MUST rotate to a new token. <!-- AUTH-001.03.02 -->
- Access token revocation is not supported; access tokens remain valid until TTL expiry. <!-- AUTH-001.03.03 -->
- Logout MUST only invalidate the current session's refresh token, not all sessions. <!-- AUTH-001.03.04 -->

---

## Data Model  <!-- AUTH-001.04 -->

### Session  <!-- AUTH-001.04.01 -->

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| id | uuid | yes | Primary key |
| userId | uuid | yes | FK → users.id |
| refreshToken | string | yes | Stored as bcrypt hash — MUST NOT store plaintext |
| expiresAt | timestamp | yes | 7 days from issuance |
| createdAt | timestamp | yes | |
| revokedAt | timestamp | no | Null indicates active session |

**Relationships:**
- Session belongs to User (many sessions per user)

---

## Design Rationale  <!-- AUTH-001.05 -->

### JWT over session cookies  <!-- AUTH-001.05.01 -->

Stateless JWT access tokens with stateful refresh tokens. Mobile clients require bearer token auth; cookies are browser-only. Short access token TTL limits exposure window. Stateful refresh tokens allow revocation. Fully stateless refresh was rejected because it provides no revocation capability.

### Refresh token rotation  <!-- AUTH-001.05.02 -->

Every refresh rotates the token. This limits the window of a stolen token — if an attacker uses it, the legitimate user's next refresh triggers reuse detection and revokes all sessions.

---

## Cross-Domain References  <!-- AUTH-001.06 -->

- **Platform:** Uses shared database connection pool and request logging middleware.

---

## Open Questions  <!-- AUTH-001.07 -->

- What is the maximum number of concurrent sessions per user? (Not found in source code — may be unlimited)
- Is there a "remember me" behavior that extends refresh token TTL?
```

### Writing the spec to disk

Once you've written the spec, place it at the path from your manifest:

```
openspec/specs/auth/spec.md
```

Then update the manifest status:

```markdown
| 2 | Authentication | AUTH | src/auth/ | openspec/specs/auth/spec.md | done | [x] |
```

### Validation

After writing each spec, run a validation pass:

1. **Tracking ID check** — every `<!-- AUTH-NNN -->` in the intermediate spec should appear in the output spec. A quick grep confirms nothing was dropped:
   ```bash
   grep -o 'AUTH-[0-9]*\.[0-9]*\.[0-9]*\|AUTH-[0-9]*\.[0-9]*\|AUTH-[0-9]*' openspec/specs/auth/spec.md | sort | uniq
   ```

2. **Behavioral accuracy check** — re-read the key source files alongside the spec. Does every invariant in the code appear as a MUST in the spec? Does every guard clause appear as an edge case scenario?

3. **Cross-reference check** — does the spec reference other domains by name? Does each referenced domain already have a spec? (This is why phase order matters.)

---

## Putting It All Together

Here's the end-to-end flow for a five-domain codebase:

| Phase | Focus | Tasks | Output |
|-------|-------|-------|--------|
| **Phase 0** — Discovery | Map the codebase structure | Walk directory tree · identify key files per domain · build dependency graph · assign phase order | `domain-map.md` |
| **Phase 1** — Manifest | Build the routing table | Assign spec IDs · map source → destination · define phase sequence | `MANIFEST.md` |
| **Phase 2a** — Foundation synthesis | Read and synthesize foundation domains | Read `src/shared/` → intermediate · Read `src/auth/` → intermediate · mark SHARED, AUTH done in manifest | `intermediate/platform.md` · `intermediate/auth.md` |
| **Phase 2b** — Feature synthesis | Read and synthesize feature domains | Read `src/users/`, `src/notifications/`, `src/orders/` → intermediates · mark USER, NOTIF, ORDER done | `intermediate/users.md` · `intermediate/notifications.md` · `intermediate/orders.md` |
| **Phase 3** — Conversion | Transform to OpenSpec format | Convert each intermediate · run tracking ID validation grep per spec | `openspec/specs/*/spec.md` (5 specs) |
| **Review** | Validate and close | Collect open questions across specs · review with domain experts · update specs · mark baseline complete | Validated `openspec/specs/` baseline |

---

## When the Codebase Changes

Once `openspec/specs/` is populated, you stop using this pipeline for ongoing changes. That's what OpenSpec is built for — incremental delta changes via `/opsx:propose → /opsx:apply → /opsx:archive`.

The intermediate specs in `intermediate/` are not the source of truth. `openspec/specs/` is. The intermediates are working documents from the conversion and can be archived or deleted once the baseline is validated.

If a large refactor changes a domain significantly, re-run Phase 2 for that domain (synthesize a new intermediate from the updated code), then use `/opsx:propose` to propose the delta changes against the current spec. The manifest tracking system tells you which source files informed which spec, so you know exactly where to start.

---

## Summary

| Phase | Input | Output | Key tool |
|-------|-------|--------|---------|
| 0 — Discovery | Codebase file tree | `domain-map.md` | Directory reading, dependency analysis |
| 1 — Manifest | Domain map | `MANIFEST.md` | Routing table with tracking IDs |
| 2 — Synthesis | Key source files per domain | `intermediate/<domain>.md` | Code reading + behavioral interpretation |
| 3 — Conversion | Intermediate specs | `openspec/specs/<domain>/spec.md` | Format transformation + validation |

The intermediate is the asset. Discovery and the manifest are coordination. The conversion is mechanical. The only phase that requires real judgment is synthesis — reading code and understanding what it actually guarantees. Invest the most time there. A good intermediate spec makes everything else straightforward.
