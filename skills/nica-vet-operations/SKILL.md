---
name: nica-vet-operations
description: Use when the user asks to query or change data in the running nica.vet system (patients, customers, sales, inventory, appointments, tasks), mentions a nica.vet operation, or calls a tool named mcp__nica-vet__*; covers tool lookup, scopes, the write-confirmation protocol, and argument rules.
---

# Operating nica.vet through MCP

The `nica-vet` MCP server exposes the nica.vet operation registry. Every tool is one registered operation; the server authorizes, audits, and scopes every call to one Org and one membership: the person who connected, by signing in on nica.vet's consent screen.

## 1. The tools you see

A connection that signed in (claude.ai, Claude Desktop, Claude Code) lists 22 tools. Tool name = operation name with each `.` replaced by `__`: `party.customer.list` → `party__customer__list`. In Claude Code the full name is `mcp__nica-vet__party__customer__list`. Each tool's `title` is the operation's summary; `readOnlyHint` marks the reads. A tool name the server does not list is refused as not found, never matched to a similar one.

Reads (no confirmation):

- Customers: `party__customer__list` (list Customers); `party__customer__note__list` (a Customer's append-only notes, newest first).
- Patients: `clinic__patient__list` (list/search patients by name, microchip, owner or status); `clinic__patient__read` (one patient with its species and breed names); `clinic__patient__alert__list` (patient alerts, per patient or org-wide by type and severity); `clinic__service_history__list` (completed service history of a patient); `clinic__visit__read` (a visit with notes and attachment links).
- Agenda: `scheduling__appointment__day_view` (all appointments of a date, with patient and resource context); `scheduling__appointment__upcoming` (upcoming appointments of one patient or of every patient of one customer); `scheduling__appointment__range` (appointments of the whole clinic between two dates, at most 31 days, paged); `scheduling__appointment__read` (one appointment with its cancellation record, arrival and recovery links); `scheduling__service__list` (service performances).
- Catalog: `catalog__product__list` (products with the price in force); `catalog__product__read` (one product with its category and today's prices); `catalog__service__list` (sellable services).
- Inventory: `inventory__replenishment_gaps__list` (products that sell or are used but have no replenishment policy at a Location).
- Confirmation: `platform__mcp__confirmation_status` (redacted state of a write confirmation; section 3 step 7).

Writes (confirmed per call, section 3):

- `party__customer__note__add` (append an immutable note to a Customer; IRREVERSIBLE).
- `scheduling__appointment__create` (book an appointment: patient, reason, assigned vet; COMPENSATABLE).

Discovery tools, for any operation that is not in the lists above:

1. `platform__mcp__search_operations` with a short `query` (a word of the entity or the action). It returns up to 8 matching operations the person's grant and role reach, each with its input schema.
2. `platform__mcp__call_read` for an operation that only reads, or `platform__mcp__call_write` for an operation that changes data. Both take `operation` (the dotted name from a search match) and `arguments` (exactly per that match's input schema). `call_read` refuses an operation that changes data and `call_write` refuses one that only reads. Operation catalogue: https://nica.vet/ai.
3. Use a named tool when one exists; use discovery only when no tool above covers the request.

Every description ends with `[module: X; reversibility: Y; HIGH-STAKES?; audited]`: `reversibility: REVERSIBLE` needs no confirmation step; `COMPENSATABLE`, `IRREVERSIBLE` or `HIGH-STAKES` follows section 3. Read the tool's input schema; the server rejects anything else.

## 2. Scopes decide what you can reach

- The connection fixes the areas. After a sign-in they are the areas («áreas», one per module: `party`, `clinic`, ...) the person checked on the consent screen. Scopes are dotted prefixes at segment boundaries (`party` covers `party.customer.create`; `clinic.patient` does not cover `clinic.visit.*`). A tool whose area was not granted is not listed, and a discovery call to an operation outside the granted areas answers an error; report it, do not retry.
- Every call runs as that person with their role in that Org: a listed tool can still answer `authz` when their role does not reach it.
- A missing tool or a search without a match means the scope was not granted or the role does not reach it. It does not mean the operation does not exist. Do not emulate a missing operation by chaining other tools.
- Tell the user which scope is missing (the operation name's prefix) and how to widen the connection. Signed in: disconnect in nica.vet (Administración › IA → «Conexiones con asistentes») or with `/mcp` → `nica-vet` → **Clear authentication**, then authenticate again and check more areas. `platform` and `identity` operations are never reachable through a named tool or through discovery, except the four `platform__mcp__*` tools of section 1.

## 3. Write confirmation protocol

Gated tools (`HIGH-STAKES`, or non-reversible while the connection confirms writes, which a sign-in always does) take two calls. Through `platform__mcp__call_write` the same two calls apply: the first answers `CONFIRMATION_REQUIRED`, and the second passes `confirmationId` and `confirmationFingerprint` inside `arguments` beside the identical business arguments.

1. Call the tool with the business arguments only.
2. If the text result parses to `{ "status": "CONFIRMATION_REQUIRED", ... }`, NOTHING happened yet. Show the user, verbatim:
   - `operation`
   - `summary`
   - the arguments you sent
   - `expiresAt`
3. Wait for an explicit yes in chat. Silence, "ok maybe", or a new request is not a yes.
4. Resend the IDENTICAL business arguments plus `confirmationId` and `confirmationFingerprint` (32 lowercase hex chars) copied from that response. Changing any argument invalidates the confirmation.
5. Never invent those two values, never reuse them for another call or another operation, never send them to a non-gated tool (the server rejects them as an input error).
6. Confirmations expire 10 minutes after issue; a stale one needs a fresh step 1.
7. On a timeout, a dropped connection, or an ambiguous result after step 4, do NOT resend. Call `platform__mcp__confirmation_status` with `{ "confirmationId": "..." }` and report its `state`:
   - `SUCCEEDED` → the stored receipt is the result.
   - `FAILED_BEFORE_EFFECT` → nothing changed; restart at step 1 if the user still wants it.
   - `PENDING` / `EXECUTING` → wait, then ask again.
   - `UNKNOWN_EFFECT` → tell the user the effect is unknown; do not retry.

Reversible operations need no confirmation, and every call is still audited under the connected person's identity and membership.

## 4. Argument rules

- Money: integer minor units plus an ISO currency code, always both; the input schema names the two fields. 400.00 NIO = `40000` with `"NIO"`. Never send a float, never guess a currency; ask the user.
- Timestamps: UTC ISO 8601 with offset (`2026-09-29T15:00:00.000Z`).
- Ids: UUIDv7 strings. Get them from list/search tools; never fabricate one.
- User-facing text you write into records (notes, names, messages): Spanish.
- Omit optional fields you have no value for; do not send `null` unless the schema allows it.

## 5. Errors

Errors arrive as `isError: true` with text:

- `<code>: <message>`, where code is one of `authn`, `authz`, `validation`, `not_found`, `conflict`, `invariant`.
- `internal error`: an unexpected server failure; report it, do not retry writes.
- Input errors: `<message>: <zod issues json>`. Fix the arguments at the listed paths and call again.

Rules:

- `authz` / `authn`: do not retry. Report the missing role, entitlement, or scope.
- `conflict`: read the message. It usually names a state precondition (already closed, stale version); re-read the entity before trying anything else.
- `not_found`: re-check the id with a read tool; do not guess another id.
- Never loop more than twice on the same error.

## 6. API keys

Members who prefer an API key connect it from the nica.vet app (Configuración › Claves de API); that path is documented there, not in this plugin.
