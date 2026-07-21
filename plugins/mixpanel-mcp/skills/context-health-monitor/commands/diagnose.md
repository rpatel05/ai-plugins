# Command: diagnose

Deep dive into a specific finding from the scan. Shows evidence, explains impact on AI output quality, and recommends the exact fix. Read-only — never writes.

**Session reads:** `drift_findings`, `current_context`, `schema_state`
**Session writes:** (none)

---

## Precondition

Requires a prior `scan` in this session (`drift_findings` must be populated). If missing, tell the user: "Run `scan` first so I have findings to diagnose." and hand off to scan.

## Step 1 — Select finding

If the user named a specific finding or dimension ("tell me about the schema drift", "why is freshness low"), match it. Otherwise, list the findings by number:

```
Which finding do you want to explore?
  1. [Critical] North star references stale event → accuracy
  2. [Warning] 4 active events not in context → schema coverage
  3. [Info] Schema Snapshot is 45 days old → freshness
```

One finding at a time. Don't dump all details for all findings.

## Step 2 — Show evidence

For the selected finding, present a side-by-side of what context says vs what the data shows.

**Format by finding type:**

### Schema drift — undocumented event
```
FINDING: Event `Feature X Activated` is active but not in context

  Live data:
    Volume (30 days):  12,450
    First seen:        ~[date range from volume data]
    Has description:   No
    Has tags:          No

  Context says:
    Not mentioned in org or project context.
    Not described in Lexicon.

  IMPACT: When a user asks about [Feature X], the AI won't know this
  event exists and may return incomplete results or use a wrong proxy.
```

### Schema drift — dead reference
```
FINDING: Context references `Legacy Signup` but it has zero volume

  Context says (project, Domain section):
    "The primary signup event is `Legacy Signup`..."

  Live data:
    Volume (30 days):  0
    Last volume:       [if detectable from schema]

  IMPACT: The AI will build funnels and analyses using an event that
  no longer fires, producing empty or misleading results.
```

### Accuracy — stale definition
```
FINDING: Qualified-user definition references events that changed

  Context says (project, Definition section):
    "Active user = performed `Old Event Name` in last 7 days"

  Live data:
    `Old Event Name` volume (30 days): 0
    Possible replacement: `New Event Name` (similar naming, 8,200 volume)

  IMPACT: Any analysis scoped to "active users" will use a dead event,
  returning zero users or silently falling back to an unscoped query.
```

### Freshness — stale snapshot
```
FINDING: Schema Snapshot is [N] days old

  Context snapshot (captured [date]):
    Top events: [list from snapshot]

  Live top events (today):
    [list from schema_state]

  Differences:
    + [new events not in snapshot]
    - [snapshot events no longer in top]

  IMPACT: The AI's understanding of which events matter is [N] days
  behind. It may prioritize deprecated events or miss new features.
```

## Step 3 — Recommend fix

Name the specific skill and command:

```
FIX: Update project business context to reflect the current schema.
  → prepare-ai-readiness / setup-context (to rewrite the affected section)
  → or prepare-ai-readiness / import-context (if the updated info exists elsewhere)

Want me to start the fix? (runs through prepare-ai-readiness)
```

For Lexicon gaps:
```
FIX: Add descriptions for undocumented events.
  → manage-lexicon / enrich-and-tag

Want me to start the fix? (runs through manage-lexicon)
```

## Follow-on

After presenting the diagnosis:
```
(a) Fix this finding    (b) Diagnose another finding    (c) Return to menu
```
