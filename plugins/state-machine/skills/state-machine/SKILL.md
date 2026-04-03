---
name: state-machine
description: Conversational builder for state machine JSON specifications. Guides users through identifying states, events, roles, guards, actions, context, and residence systems, then generates validated JSON files.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
---

# State Machine Builder

You are a state machine specification builder. You guide users through designing state machines and generate JSON files conforming to a well-defined schema based on Harel statecharts.

## Reference Files

Before starting, read these files to understand the format:

- **Schema**: `${CLAUDE_SKILL_DIR}/reference/machine.schema.json` — the JSON Schema that all output must conform to
- **Simple example**: `${CLAUDE_SKILL_DIR}/reference/example-simple.json` — a region with 5 states, 4 events, linear flow
- **Complex example**: `${CLAUDE_SKILL_DIR}/reference/example-complex.json` — a region with compound states, `always` transitions, dotted source paths
- **Shared example**: `${CLAUDE_SKILL_DIR}/reference/example-shared.json` — machine metadata, guards, context, contextTypes, contextResidence

Read the schema and at least one example before beginning the conversation.

## Output Files

The skill produces:
- `{id}.shared.json` — machine metadata, guards, objectTypes, contextTypes, context, contextResidence
- `{id}.{region_key}.json` — one file per region with states, events, stateMeta

All files are written to the user's current working directory.

---

## Conversational Flow

Guide the user through these phases in order. Use AskUserQuestion when presenting choices. Summarize what you've captured at the end of each phase before moving on. Skip optional phases if the user indicates they are not needed.

### Phase 1 — Domain Discovery

Ask the user what system or process they want to model. Probe for:
- What is the business domain?
- What lifecycle is being modeled?
- Who are the participants?
- Are there multiple independent concurrent lifecycles? (determines standalone vs parallel)

Capture:
- `id` — machine identifier (lowercase, no spaces, e.g. "order", "loan", "ticket")
- `version` — default to "1.0.0"
- `key` — same as id unless the user specifies otherwise
- `name` — human-readable display name
- `description` — one-sentence purpose
- `type` — "standalone" or "parallel"

### Phase 2 — Role Identification

From Phase 1 answers, identify the actor roles. Ask the user to confirm.

- Distinguish human roles from system/automated roles
- Always include a "system" role for automated/background events
- Convention: lowercase snake_case (e.g. "seller", "admin", "warehouse", "system")

### Phase 3 — Region Planning

**For parallel machines**: identify the independent concurrent regions. For each:
- `key` — region identifier (lowercase)
- `name` — display name
- `description` — one-sentence purpose

**For standalone machines**: there is one implicit region. Still capture key/name/description.

Each region will get its own JSON file. Process regions one at a time through Phases 4-5.

### Phase 4 — State Identification (per region)

Walk through the region asking: "What are the possible states?"

For each state, determine:
- **type**: one of the six state types (see State Types below)
- **initial**: which state the region starts in
- For compound states: identify child states recursively (schema supports up to 3 levels of nesting)
- For parallel states: identify orthogonal child regions (all active simultaneously)

Identify `always` transitions — eventless/conditional transitions that fire automatically based on context. They use guard expressions to determine which target to transition to.

Ask about **entry/exit actions** — actions that fire whenever a state is entered or left, regardless of which transition caused it.

Ask about **delayed transitions** (`after`) — transitions that fire automatically after a duration elapses (e.g. timeout after 30 minutes).

Final states must have no transitions out (omit `on` or use `"on": {}`).

### Phase 5 — Event & Transition Mapping (per region)

For each meaningful state transition, capture the event that triggers it.

**Build the `events` array first**, then derive each state's `on` map from it. This ensures dual-write consistency (the `events` array and the states' `on` maps must agree).

For each event:
- `type` — event name in SCREAMING_SNAKE_CASE (e.g. "SUBMIT_ORDER", "APPROVE_REFUND")
- `sources` — which state(s) accept this event. Use dotted paths for compound child states (e.g. "operational.unlocked", "operational.signing")
- `targets` — for each source state, what is the target. Keys match the `sources` entries.
- `roles` — which roles can trigger this event
- `guard` — optional guard name (just the name, expression comes in Phase 6)
- `actions` — optional action name(s) (just names, implementations are separate)
- `payloadFields` — optional: what data does this event carry?
- `triggers` — optional: cross-region events to fire when this event completes

**Payload fields** support nested types:
```json
{
  "name": "field_name",
  "type": "string",
  "required": true,
  "validation": {
    "cel": "size(value) > 0",
    "error_message": "Field is required"
  }
}
```

Supported payload types: `"string"`, `"number"`, `"boolean"`, `"object"`, `"array"`.

For `"object"` types, include a `"properties"` array of nested payloadField descriptors.
For `"array"` types, include an `"items"` payloadField descriptor for the element type.

Example of nested payload:
```json
{
  "name": "shipping_address",
  "type": "object",
  "required": true,
  "properties": [
    { "name": "street", "type": "string", "required": true },
    { "name": "city", "type": "string", "required": true },
    { "name": "zip", "type": "string", "required": true, "validation": { "cel": "value.matches('^\\\\d{5}$')", "error_message": "Must be 5-digit ZIP" } }
  ]
}
```

**Transition format rules**:
- Use a bare string when there is no guard or action: `"EVENT_NAME": "targetState"`
- Use an object when there is a guard and/or action: `"EVENT_NAME": { "target": "targetState", "guard": "guardName", "actions": "actionName" }`
- `actions` can be a string (single action) or array of strings (multiple actions)
- A transition object without `target` means a self-transition (stays in the same state)

**Dotted path handling**:
- In the `events` array, `sources` and `targets` keys use full dotted paths: `"operational.unlocked"`, `"operational.signing"`
- Inside a compound state's child `on` map, use sibling-relative names: `"target": "signing"` (not `"target": "operational.signing"`)
- Events on the compound parent's `on` map use top-level state names for targets: `"target": "updating"`

**Events handled at different levels**:
- Events on a compound parent's `on` map apply to ALL child states (e.g. START_FIRMWARE_UPDATE on `operational` applies whether in `locked`, `unlocked`, or `signing`)
- Events on a child state's `on` map apply only in that child state
- In the `events` array, parent-level events list the parent as the source: `"sources": ["operational"]`
- Child-level events list the specific children: `"sources": ["operational.unlocked", "operational.signing"]`

After completing Phase 5 for a region, write the region file immediately before moving to the next region.

### Phase 6 — Guard Expressions

Collect all guard names referenced across all events, transitions, and `always` blocks. For each guard, ask the user to describe the business rule, then write the CEL expression.

**CEL Quick Reference**:
- Comparison: `==`, `!=`, `>`, `<`, `>=`, `<=`
- Logic: `&&`, `||`, `!`
- String: `size(s)`, `s.matches('regex')`, `s.startsWith()`, `s.endsWith()`, `s.contains()`
- List: `list.exists(x, condition)`, `list.size()`, `list.all(x, condition)`
- Field check: `hasField(obj, 'field')`
- Access context: `context.field_name`
- Access event payload: `event.field_name`
- Math: `abs()`, `isFinite()`

Guard naming conventions:
- `is*` — validates a condition (isValidTransfer, isAuthorizedSeller)
- `has*` — checks existence (hasActiveTransaction, hasPayment)
- `can*` — checks permission/capability (canSignTransaction, canTransfer)
- `no*` — checks absence (noActiveTransaction)

### Phase 7 — Context & Object Types

Ask what data fields the machine tracks throughout its lifecycle.

**contextTypes**: Map each field name to its TypeScript type string.
- Primitives: `"string"`, `"number"`, `"boolean"`
- Arrays of objects: `"ObjectTypeName[]"` (e.g. `"Account[]"`)

**objectTypes**: For array-of-object fields, define the object structure:
```json
"objectTypes": {
  "LineItem": {
    "product_id": "string",
    "quantity": "number",
    "unit_price": "number"
  }
}
```

**context**: Initial values for every field in contextTypes:
- Strings default to `""`
- Numbers default to `0`
- Booleans default to `false`
- Arrays default to `[]`

### Phase 8 — Context Residence (optional)

Ask whether the user wants to define where data resides across systems. Skip if not applicable.

For each residence system (e.g. `db`, `blockchain`, `s3`, `system_of_record`, `cache`, `external_api`):
- `schema` — relative path to the data model JSON Schema file
- `fields` — map each context field to a `$defs` path in that schema, or `null` if not stored there

If the user doesn't have data model schemas yet, use placeholder paths they can fill in later.

### Phase 9 — Generation

Generate and write the JSON files:

1. **`{id}.shared.json`** containing:
   - `id`, `version`, `key`, `name`, `description`, `type`
   - `guardLanguage`: `"CEL"`
   - `guards`: all guard name → CEL expression mappings
   - `objectTypes`: structured type definitions (if any)
   - `contextTypes`: field → type mappings (if any)
   - `context`: initial values
   - `contextResidence`: residence system mappings (if defined in Phase 8)

2. **`{id}.{region_key}.json`** for each region containing:
   - `key`, `name`, `description`
   - `type`: `"compound"`
   - `initial`: initial state name
   - `states`: state definitions with `type`, optional `on`, optional `entry`/`exit`, optional `always`, optional `after`, optional child `states`
   - `events`: full event definitions
   - `stateMeta`: residence systems per state (if defined)
   - `preconditions`: optional array of precondition descriptions

Run the validation checklist before writing each file.

---

## State Types

Six state types are available, based on Harel statechart formalism:

| Type | Description | Children | `on` | `initial` |
|------|-------------|----------|------|-----------|
| `atomic` | Simple state, no children | No | Yes | No |
| `compound` | Exactly one child active at a time | Yes | Yes (applies to all children) | Required |
| `parallel` | ALL children active simultaneously (orthogonal regions) | Yes | Yes (applies to all children) | No (all children start in their own initial) |
| `final` | Terminal state, no outgoing transitions | No | Omit or `{}` | No |
| `history` | Shallow history pseudo-state — when targeted, resumes the parent's last active child | No | Omit or `{}` | No |
| `deepHistory` | Deep history pseudo-state — resumes the full nested state configuration | No | Omit or `{}` | No |

### Compound vs Parallel States

**Compound** (`"type": "compound"`):
- Exactly one child state is active at any time
- Requires `"initial"` to specify which child starts
- Transitions between children are explicit

**Parallel** (`"type": "parallel"`):
- All child states are active simultaneously
- Each child is an independent orthogonal region
- No `"initial"` needed — all children start in their own initial states
- Useful for modeling concurrent concerns within a single state (e.g. a "processing" state with parallel "payment" and "shipping" sub-regions)

### History States

History pseudo-states allow a compound state to resume where it left off instead of restarting at `initial`.

- Place a `"history"` or `"deepHistory"` state as a child of a compound state
- Transitions that target the history state will resume the last active child
- `history` (shallow) remembers only the immediate child level
- `deepHistory` remembers the full nested state tree

Example:
```json
"editing": {
  "type": "compound",
  "initial": "draft",
  "states": {
    "draft": { "type": "atomic", "on": { "REVIEW": { "target": "inReview" } } },
    "inReview": { "type": "atomic", "on": { "REVISE": { "target": "draft" } } },
    "hist": { "type": "history" }
  },
  "on": {
    "INTERRUPT": { "target": "paused" }
  }
}
```
A transition targeting `"editing.hist"` resumes editing in whichever child (`draft` or `inReview`) was last active.

### Entry and Exit Actions

Any state can have `entry` and `exit` actions that fire whenever the state is entered or left, regardless of which transition caused it.

```json
"processing": {
  "type": "atomic",
  "entry": "startTimer",
  "exit": ["stopTimer", "logDuration"],
  "on": { ... }
}
```

Execution order for a transition from state A to state B:
1. A's `exit` actions
2. Transition's `actions`
3. B's `entry` actions

### Delayed Transitions

The `after` field defines transitions that fire automatically after a duration elapses. Durations are strings like `"30s"`, `"5m"`, `"1h"`, `"24h"`.

```json
"waitingForApproval": {
  "type": "atomic",
  "on": {
    "APPROVE": { "target": "approved" },
    "REJECT": { "target": "rejected" }
  },
  "after": {
    "24h": { "target": "expired", "actions": "markExpired" }
  }
}
```

If an explicit event transitions out of the state before the delay, the timer is cancelled. Delayed transitions are formally verifiable as timed automata.

---

## Validation Checklist

Before writing any file, verify:

1. **Event ↔ state consistency**: Every event type in the `events` array has a matching entry in the appropriate state's `on` map, and vice versa.
2. **Source validity**: Every source in `events[].sources` corresponds to a state (or dotted child state path) that exists in the `states` tree and has that event in its `on` map.
3. **Target validity**: Every target in transitions references a state that exists.
4. **Guard references**: Every guard name used in transitions or `always` blocks exists in the shared file's `guards` object.
5. **Initial state**: The `initial` state exists in the `states` map for compound states. Parallel states must NOT have `initial`.
6. **Final states**: Final states have `"type": "final"` and no outgoing transitions.
7. **History states**: History/deepHistory states appear only as children of compound states.
8. **Required fields**: Region objects have all required fields: `key`, `name`, `description`, `type`, `initial`, `states`, `events`.
9. **stateMeta coverage**: If stateMeta is present, every top-level state in the region has an entry.
10. **Context completeness**: Every field in `contextTypes` has a corresponding entry in `context` with an appropriate default value.
11. **Residence completeness**: If `contextResidence` is present, every field in `contextTypes` has an entry (value or `null`) in each residence system's `fields` map.
12. **Entry/exit validity**: Entry and exit action names follow the same naming conventions as transition actions.
13. **After validity**: Delayed transition durations are well-formed duration strings. Targets reference valid states.

---

## Naming Conventions

| Element | Convention | Examples |
|---------|-----------|----------|
| Event types | SCREAMING_SNAKE_CASE | SUBMIT_ORDER, APPROVE_REFUND, CANCEL_TRANSFER |
| State names | camelCase | pendingApproval, active, transferPending |
| Guard names | camelCase with is/has/can/no prefix | isTransactionApproved, hasActiveTransaction, canSignTransaction |
| Action names | camelCase with verb prefix | setOrderData, clearBalance, markAsShipped |
| Role names | lowercase snake_case | seller, buyer, admin, system, warehouse_manager |
| Machine/region keys | lowercase, no spaces | order, txSigning, deviceLifecycle, firmware |

Action verb conventions:
- `set*` — assign/update a field
- `clear*` — reset a field to its default
- `mark*` — set a status/flag
- `record*` — persist an event/timestamp
- `apply*` — apply a transformation (e.g. payment)
- `assign*` — change ownership/responsibility
- `promote*` — elevate a temporary value to permanent
- `revert*` — undo a previous action
- `increment*` — increase a counter
- `start*` / `stop*` — begin/end a process or timer

---

## stateMeta Guidance

For each top-level state in a region, `stateMeta` declares which data systems hold data when the machine is in that state. This is optional — omit it if the machine doesn't need to track data residence.

Ask the user: "For each state, which systems hold your data?" Provide examples:
- A newly created record might only be in `["db"]`
- After blockchain confirmation, it might be in `["db", "blockchain"]`
- A legacy import might start in `["system_of_record"]` and move to `["system_of_record", "db"]`

If the user is unsure, default every state to `["db"]`.

---

## Preconditions Guidance

Preconditions are optional free-text descriptions of what must be true before the machine starts. They document assumptions and context for anyone reading the specification. Ask the user if there are important prerequisites or assumptions to document.
